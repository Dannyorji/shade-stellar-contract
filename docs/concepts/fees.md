# Fees, volume discounts, and time-locked fee changes

This page documents how the [protocol fee](../glossary.md#protocol-fee) is configured, how a merchant's payment volume discounts it, how `calculate_fee` resolves the final rate at payment time, and how a fee change can only take effect through a time-locked propose/execute flow.

## Why it exists

Fees are the protocol's revenue, and they're deducted from every invoice payment before the merchant sees the funds (see [Payment flows](./payments.md#walkthrough-pay_invoice)). Two design choices follow from that:

- **Volume discounts** reward merchants who process more through the protocol by lowering their effective rate automatically, with no manual intervention per merchant.
- **The timelock on fee changes** (`propose_fee` → wait → `execute_fee`) exists so a merchant integrating against a known fee rate has advance notice before it changes, rather than having the admin silently raise fees on their next payment.

## Setting the base fee: `set_fee` / `get_fee`

`set_fee(admin, token, fee)` ([`contracts/shade/src/components/admin.rs:109`](../../contracts/shade/src/components/admin.rs#L109)) stores a token's default fee rate. `fee` is in **basis points (bps)**, where `10_000` bps = 100%. It requires admin authorization and that `token` is already on the global accepted-token list; it does not validate that `fee` itself is within `0..=10_000` — see [Constraints and edge cases](#constraints-and-edge-cases).

`get_fee(token)` ([`contracts/shade/src/components/admin.rs:131`](../../contracts/shade/src/components/admin.rs#L131)) returns the stored value, or **`0` if the token has no fee configured at all** — an unset fee charges nothing, it does not fall back to some protocol-wide default percentage.

> **Note:** `set_fee` changes the active fee **immediately**. It bypasses the timelock entirely. The timelock only applies to fee changes made through `propose_fee` / `execute_fee` — see [Time-locked fee changes](#time-locked-fee-changes-propose_fee--execute_fee) below.

## The computation: `calculate_fee`

`calculate_fee(merchant, token, amount)` ([`contracts/shade/src/components/admin.rs:195`](../../contracts/shade/src/components/admin.rs#L195)) is the pure, non-panicking query a reader or integrator calls to find out what fee an amount would attract. It is also, internally, the same rate computation the payment path uses (via `platform_fee::effective_fee_bps`), so a value read from `calculate_fee` matches what `pay_invoice` will actually charge, *as of that ledger state* — it can still change between the query and the payment if the admin or the merchant's volume changes in between.

```rust
pub fn calculate_fee(env: &Env, merchant: &Address, token: &Address, amount: i128) -> i128 {
    if amount <= 0 {
        return 0;
    }
    let merchant_id = merchant::find_merchant_id(env, merchant).unwrap_or(0);
    let fee_bps = platform_fee::effective_fee_bps(env, merchant_id, merchant, token);
    if fee_bps == 0 {
        0
    } else {
        (amount * fee_bps) / 10_000i128
    }
}
```

Unlike the payment path's `compute_platform_fee_split` (below), `calculate_fee` never panics: a non-positive `amount` returns `0`, and an address that isn't a registered merchant is priced at the token's plain default rate (`merchant_id` falls back to `0`) instead of being rejected.

### Rate resolution order

`effective_fee_bps` ([`contracts/shade/src/components/platform_fee.rs:36`](../../contracts/shade/src/components/platform_fee.rs#L36)) resolves the rate in two ordered steps, applied every time a fee is computed (at invoice creation, at `calculate_fee`, and at payment time):

1. **Base rate** — a per-merchant, per-token override set via `set_merchant_platform_fee` (`DataKey::MerchantPlatformFee(merchant_id, token)`) if one exists; otherwise the token's default rate from `get_fee(token)`.
2. **Volume discount** — `apply_volume_discount` applies a percentage discount *to that base rate* based on the merchant's cumulative volume in that token (`get_merchant_volume`). The discount always applies on top of whichever base rate was resolved in step 1, including a merchant-specific override.

### Rounding rule

The final fee amount is `(amount * fee_bps) / 10_000` using Soroban's `i128` integer division, which truncates toward zero. Since `amount` and `fee_bps` are both non-negative in practice, this means the **platform fee always rounds down**, and the merchant's share (`amount - platform_fee`) absorbs the rounding remainder. The platform never collects more than the exact bps rate implies; the merchant never receives less.

## Volume discounts

`get_merchant_volume(merchant, token)` ([`contracts/shade/src/components/admin.rs:208`](../../contracts/shade/src/components/admin.rs#L208)) returns `total_volume` from that merchant/token pair's `MerchantAnalytics` record. This value accrues **automatically**: every completed payment calls `admin::record_merchant_payment` (from `platform_fee::finalize_route`, after the transfers for that payment have already executed) and adds the payment's gross amount to `total_volume`. There is no separate function to manually credit volume — a merchant's discount tier is purely a function of how much they've been paid through the protocol in that token, historically.

> **Note:** because volume is recorded *after* a payment's transfers complete, a payment's own amount is not counted toward the discount tier used to price that same payment — only volume from *prior* payments applies.

`apply_volume_discount` ([`contracts/shade/src/components/platform_fee.rs:18`](../../contracts/shade/src/components/platform_fee.rs#L18)) applies one of four fixed tiers, keyed on that cumulative volume:

| Cumulative volume in token | Discount off the base rate |
|---|---|
| `< 10,000` | 0% (no discount) |
| `>= 10,000` | 10% |
| `>= 50,000` | 25% |
| `>= 200,000` | 50% |

The discounted rate is `base_bps * (100 - discount_percentage) / 100`, again using integer division (floored).

> **Implementation note:** `contracts/shade/src/types.rs` defines a `VolumeDiscount` struct (`min_volume`, `discount_bps` — [`contracts/shade/src/types.rs:443-446`](../../contracts/shade/src/types.rs#L443-L446)), but nothing in the contract constructs, stores, or reads a `VolumeDiscount` value today. The tiers above are hardcoded constants inside `apply_volume_discount`, not driven by that type. Treat `VolumeDiscount` as reserved for a future configurable-tiers feature, not as the mechanism currently in effect.

## Time-locked fee changes: `propose_fee` / `execute_fee`

A fee change made through this flow cannot take effect until a fixed delay has passed since it was proposed.

1. **`propose_fee(admin, token, fee)`** ([`contracts/shade/src/components/admin.rs:391`](../../contracts/shade/src/components/admin.rs#L391)) — admin-only, requires `token` to be globally accepted. Stores a `PendingFee { token, fee, proposed_at: now }` at `DataKey::PendingTokenFee(token)` and emits `FeeProposedEvent`. The active fee (`get_fee`) is **unchanged** at this point.
2. **Wait** — `FEE_UPDATE_DELAY` is `172_800` seconds (48 hours) ([`contracts/shade/src/components/admin.rs:9`](../../contracts/shade/src/components/admin.rs#L9)).
3. **`execute_fee(admin, token)`** ([`contracts/shade/src/components/admin.rs:419`](../../contracts/shade/src/components/admin.rs#L419)) — admin-only. Loads the pending proposal (panics with `ContractError::NoPendingFeeUpdate` if none exists), checks `env.ledger().timestamp() - pending.proposed_at >= FEE_UPDATE_DELAY` (panics with `ContractError::FeeUpdateTooEarly` otherwise), then writes `pending.fee` to the active `TokenFee(token)` storage, deletes the pending entry, and emits `FeeSetEvent` — the same event `set_fee` emits, so the event log alone does not distinguish a direct `set_fee` call from an executed proposal.
4. **`get_pending_fee(token)`** ([`contracts/shade/src/components/admin.rs:452`](../../contracts/shade/src/components/admin.rs#L452)) — reads the current pending proposal, panicking with `NoPendingFeeUpdate` if there isn't one.

```mermaid
sequenceDiagram
    participant Admin
    participant Shade as Shade contract

    Admin->>Shade: propose_fee(admin, token, new_fee)
    Shade->>Shade: store PendingFee{token, fee, proposed_at}
    Note over Shade: get_fee(token) still returns the old fee
    Note over Admin,Shade: >= 48 hours pass
    Admin->>Shade: execute_fee(admin, token)
    Shade->>Shade: check elapsed >= FEE_UPDATE_DELAY
    Shade->>Shade: TokenFee(token) = pending.fee; clear PendingTokenFee
    Shade-->>Admin: FeeSetEvent
```

### Superseded and never-executed proposals

- **Superseding.** Calling `propose_fee` again for the same token before the first proposal is executed simply **overwrites** the stored `PendingFee` with the new `fee` and a new `proposed_at`, restarting the 48-hour clock. The superseded value is gone from storage; only the `FeeProposedEvent` history (off-chain, via an indexer) shows it ever existed.
- **Never executed.** A pending proposal has no expiry. It sits in storage indefinitely — `get_fee` keeps returning the old active fee — until an admin calls `execute_fee` (at any time at or after the 48-hour mark; there is no upper bound) or calls `propose_fee` again to replace it.

> **Warning:** `execute_fee` only checks that *a* proposal exists and is old enough — it does not re-validate that the proposed `fee` is still sensible (e.g. still `<= 10_000` bps). If a stale proposal with a bad value is executed, every subsequent payment in that token will fail at the `compute_split` guard described in [Constraints and edge cases](#constraints-and-edge-cases) below.

## The platform account: `set_platform_account` / `get_platform_account`

`get_platform_account(env)` ([`contracts/shade/src/components/admin.rs:153`](../../contracts/shade/src/components/admin.rs#L153)) is the destination of every platform-fee transfer (see [Payment flows](./payments.md#walkthrough-pay_invoice)). If it has never been set via `set_platform_account`, it **defaults to the current admin address** (`core::get_admin`) — platform fees silently flow into the admin's own wallet, not a dedicated treasury, until an admin explicitly configures one.

Because the platform-fee transfer happens in the same atomic call as the merchant-fee transfer (see [Payment flows](./payments.md#walkthrough-pay_invoice)), a misconfigured platform account — one that cannot receive the token (no trustline, a contract that rejects the transfer, etc.) — causes the *entire payment* to panic and roll back. A bad platform account doesn't just lose the platform's fee; it blocks the merchant from getting paid at all until the admin fixes it.

## Worked examples

All three examples use the rate-resolution order and rounding rule above. `calculate_fee`'s formula matches the contract's own test at [`contracts/shade/src/tests/test_time_locked_fees.rs`](../../contracts/shade/src/tests/test_time_locked_fees.rs) (`test_no_discount_below_threshold`: `fee == amount * fee_bps / 10_000`).

### Example 1 — default rate, no discount, with rounding

- Token default fee: `300` bps (3%), set via `set_fee`.
- Merchant has no per-merchant override and a cumulative volume of `2,000` in this token — below the `10,000` discount threshold, so the effective rate is the base `300` bps.
- Invoice payment amount: `4,999`.

```
platform_fee     = (4,999 * 300) / 10,000 = 1,499,700 / 10,000 = 149  (floored from 149.97)
merchant_amount  = 4,999 - 149             = 4,850
```

The platform receives `149`, the merchant receives `4,850`; the `0.03` of a unit that rounding would otherwise have cost the platform instead stays with the merchant.

### Example 2 — default rate with a volume discount

- Token default fee: `500` bps (5%).
- Merchant has no per-merchant override, but a cumulative volume of `75,000` in this token — at or above the `50,000` tier, which carries a 25% discount.
- Effective rate: `500 * (100 - 25) / 100 = 500 * 75 / 100 = 375` bps.
- Invoice payment amount: `12,345`.

```
platform_fee     = (12,345 * 375) / 10,000 = 4,629,375 / 10,000 = 462  (floored from 462.9375)
merchant_amount  = 12,345 - 462             = 11,883
```

### Example 3 — a merchant-specific fee override stacked with a volume discount

- Token default fee: `500` bps, but this merchant has a per-merchant override of `200` bps (2%) set via `set_merchant_platform_fee` — step 1 of rate resolution uses the override, not the token default.
- Merchant's cumulative volume in this token is `250,000` — at or above the `200,000` tier, a 50% discount.
- Effective rate: `200 * (100 - 50) / 100 = 200 * 50 / 100 = 100` bps (1%).
- Invoice payment amount: `1,000,000`.

```
platform_fee     = (1,000,000 * 100) / 10,000 = 10,000
merchant_amount  = 1,000,000 - 10,000          = 990,000
```

The merchant's per-merchant override (`200` bps) is what the volume discount is applied to, not the token's `500` bps default — the override fully replaces the base rate in step 1 of resolution, it does not combine with it.

## Relevant types and storage

| Type / key | Defined in | Purpose |
|---|---|---|
| `PendingFee` | [`contracts/shade/src/types.rs:479-485`](../../contracts/shade/src/types.rs#L479-L485) | A proposed fee awaiting its timelock: `token`, `fee`, `proposed_at`. |
| `PlatformFeeSplit` | [`contracts/shade/src/types.rs:656-663`](../../contracts/shade/src/types.rs#L656-L663) | The result of a fee computation: `gross_amount`, `platform_fee`, `merchant_amount`, `fee_bps_applied`. |
| `VolumeDiscount` | [`contracts/shade/src/types.rs:443-446`](../../contracts/shade/src/types.rs#L443-L446) | Defined but currently unused — see the implementation note above. |
| `DataKey::TokenFee(Address)` | [`contracts/shade/src/types.rs:49`](../../contracts/shade/src/types.rs#L49) | The active fee rate (bps) for a token. |
| `DataKey::PendingTokenFee(Address)` | [`contracts/shade/src/types.rs:51`](../../contracts/shade/src/types.rs#L51) | The pending, not-yet-executed fee proposal for a token. |
| `DataKey::MerchantPlatformFee(u64, Address)` | [`contracts/shade/src/types.rs:53`](../../contracts/shade/src/types.rs#L53) | Per-merchant, per-token fee override, if set. |
| `DataKey::PlatformAccount` | [`contracts/shade/src/types.rs:43`](../../contracts/shade/src/types.rs#L43) | The fee-destination address. |

## Relevant functions

| Function | Defined in | Purpose |
|---|---|---|
| `set_fee` / `get_fee` | [`contracts/shade/src/interface.rs:32-33`](../../contracts/shade/src/interface.rs#L32-L33) | Set/read a token's default fee immediately (no timelock). |
| `calculate_fee` | [`contracts/shade/src/interface.rs:108`](../../contracts/shade/src/interface.rs#L108) | Non-panicking fee preview for a merchant/token/amount. |
| `compute_platform_fee_split` | [`contracts/shade/src/interface.rs:109-114`](../../contracts/shade/src/interface.rs#L109-L114) | Exposes the exact `platform_fee::compute_split` used at payment time, including its guard against a fee `>=` the amount. |
| `set_merchant_platform_fee` / `get_merchant_platform_fee` / `clear_merchant_platform_fee` | [`contracts/shade/src/interface.rs:115-128`](../../contracts/shade/src/interface.rs#L115-L128) | Manage a per-merchant fee override for a token. |
| `get_merchant_volume` | [`contracts/shade/src/interface.rs:129`](../../contracts/shade/src/interface.rs#L129) | Cumulative paid volume feeding the discount tiers. |
| `propose_fee` / `execute_fee` / `get_pending_fee` | [`contracts/shade/src/interface.rs:38-40`](../../contracts/shade/src/interface.rs#L38-L40) | The time-locked fee-change flow. |
| `set_platform_account` / `get_platform_account` | [`contracts/shade/src/interface.rs:34-35`](../../contracts/shade/src/interface.rs#L34-L35) | Configure the fee destination. |

## Constraints and edge cases

- **No range validation on `set_fee` or `propose_fee`.** Neither checks that `fee` is within `0..=10_000` bps, unlike `set_merchant_platform_fee`, which does enforce that range. If an admin sets or executes a fee above `10_000` bps (over 100%), `platform_fee::compute_split`'s guard (`if platform_fee >= amount { panic }`) will cause **every payment in that token to fail** until the fee is corrected — this is a protocol-wide outage for that token, not a per-payment error.
- **`execute_fee` cannot be called early.** Attempting it before 48 hours have elapsed panics with `ContractError::FeeUpdateTooEarly` rather than silently waiting or queuing.
- **`propose_fee` / `execute_fee` are admin-only**, not Manager-accessible, unlike `set_merchant_platform_fee` / `clear_merchant_platform_fee`, which accept either an Admin or a Manager role (`assert_fee_operator`).
- **An unconfigured platform account routes fees to the admin's wallet**, not to a reverted or zero-value transfer — this can look like a feature (fees are never lost) but means fee income isn't segregated unless `set_platform_account` is called explicitly.
- Both the base-rate resolution and the volume-discount lookup happen fresh on every call — there is no caching or snapshot of a merchant's rate, so two payments seconds apart can be priced differently if the merchant's volume crosses a tier boundary between them, or if a fee change (direct or executed) lands in between.

## Related pages

- [Payment flows: full, partial, and batch](./payments.md)
- [Merchants](./merchants.md)
- [Merchant analytics and transaction history](./analytics-and-history.md)

← [Back to concepts](./README.md)
