# Payment flows: full, partial, and batch

This page documents how value actually moves when a [payer](../glossary.md#payer) settles an [invoice](../glossary.md#invoice): the checks that run before any token transfer, the order transfers happen in, and the guarantees a payer and [merchant](../glossary.md#merchant) get from `pay_invoice`, `pay_invoice_partial`, and `pay_invoices_batch`.

> **Implementation note:** the payment logic described here lives in [`contracts/shade/src/components/invoice.rs`](../../contracts/shade/src/components/invoice.rs) (the `pay_invoice*` family) and [`contracts/shade/src/components/platform_fee.rs`](../../contracts/shade/src/components/platform_fee.rs) (the fee split). `contracts/shade/src/components/payment.rs` is a separate, smaller component that only validates swap-routing payloads (`validate_payment_payload`) — see [Payment payloads, swap routing, and the cross-chain bridge placeholder](./payment-payloads-and-routing.md). It is not where invoice settlement happens.

## Why it exists

A payer calling `pay_invoice` needs a single, atomic action that moves funds to the merchant, carves out the platform's cut, and updates the invoice's state — with no window in which funds have moved but the invoice still looks unpaid, or vice versa. Shade gets this from Soroban's own transaction atomicity (any panic unwinds every storage write and token transfer made during the call) rather than from custom rollback code, so the preconditions below exist to make the panics happen *before* any transfer, not after.

## Preconditions checked before any transfer

Three layers of checks run before a token ever moves. In call order:

1. **Contract not paused** — the dispatch functions in [`contracts/shade/src/shade.rs`](../../contracts/shade/src/shade.rs) (`pay_invoice`, `pay_invoices_batch`, `pay_invoice_partial`) call `pausable_component::assert_not_paused(&env)` before delegating to `invoice_component`. A paused contract rejects every payment call outright.
2. **Payer authorization** — `payer.require_auth()` runs once per call. `pay_invoices_batch` authorizes the payer **once for the whole batch**, then pays each invoice without re-authorizing, because Soroban only allows `require_auth` to be invoked once per address per call frame (see the per-invoice helper's doc comment at [`contracts/shade/src/components/invoice.rs:676-678`](../../contracts/shade/src/components/invoice.rs#L676-L678)).
3. **Invoice-level checks**, all inside `pay_invoice_partial_inner` ([`contracts/shade/src/components/invoice.rs:697`](../../contracts/shade/src/components/invoice.rs#L697)):
   - `amount > 0` → `ContractError::InvalidAmount`.
   - The invoice exists (`get_invoice` panics with `InvoiceNotFound` otherwise).
   - If the invoice has an `expires_at` and the ledger timestamp has reached it → `ContractError::InvoiceExpired`.
   - The invoice status is `Pending` or `PartiallyPaid` → anything else (`Draft`, `Paid`, `Cancelled`, `Refunded`, `PartiallyRefunded`) raises `ContractError::InvalidInvoiceStatus`.
   - `amount_paid + amount` does not exceed `invoice.amount` → overpayment raises `ContractError::InvalidAmount` (see [Overpayment and underpayment](#overpayment-underpayment-and-rounding) below).
   - The invoice's token is globally accepted (`admin::is_accepted_token`) → otherwise `ContractError::TokenNotAccepted`.
   - If the invoice already has a recorded `payer` and the caller is a different address → `ContractError::NotAuthorized`. The first successful payer "claims" the invoice; later partial payments must come from that same address.
4. **Fee-split guard**, inside `platform_fee::compute_split` ([`contracts/shade/src/components/platform_fee.rs:42`](../../contracts/shade/src/components/platform_fee.rs#L42)): if the computed platform fee would be greater than or equal to the payment amount, the call panics with `ContractError::InvalidAmount` rather than letting the merchant receive zero or a negative amount. This can only happen if a token's configured fee is misconfigured above 100% — see [Fees, volume discounts, and time-locked fee changes](./fees.md#constraints-and-edge-cases).

> **Note:** merchant-active status is enforced at invoice *creation* time (`validate_invoice_creation` checks `merchant::is_merchant_active`), not at payment time. A merchant deactivated after issuing a pending invoice does not block that invoice from still being paid — `pay_invoice_partial_inner` never re-checks merchant activation. Likewise, per-merchant token acceptance (`is_token_accepted_for_merchant`) is only enforced when the invoice is created; at payment time only the *global* accepted-token list is rechecked.

## Walkthrough: `pay_invoice`

`pay_invoice(payer, invoice_id)` pays an invoice in full in one call:

1. `payer.require_auth()`.
2. `pay_invoice_inner` loads the invoice and confirms its status is `Pending` or `PartiallyPaid`.
3. It computes `remaining_amount = resolve_invoice_amount(invoice_id) - invoice.amount_paid`. For a fiat-priced invoice that has not yet received a first payment, `resolve_invoice_amount` re-quotes the oracle price live rather than using the stored `amount` — see [Fiat pricing](../glossary.md#fiat-pricing).
4. If `remaining_amount <= 0`, it panics with `InvalidInvoiceStatus` (defensive — in practice an invoice in `Pending`/`PartiallyPaid` always has a positive remainder).
5. It calls `pay_invoice_partial_inner(payer, invoice_id, remaining_amount)`, which does the actual work:
   1. Re-runs the preconditions above.
   2. Refreshes the fiat quote and persists it if the invoice is `FixedFiat` and unpaid (`refresh_fiat_invoice_quote`), emitting a `FiatInvoicePricedEvent`.
   3. Resolves the merchant's address and [merchant account](../glossary.md#merchant-account) from the invoice's `merchant_id`.
   4. Calls `platform_fee::route_from_payer`, which:
      - Computes the fee split (`compute_split` — see [Fees](./fees.md#the-computation)).
      - **Transfers the merchant's share directly from the payer to the merchant account** via `token::TokenClient::transfer`.
      - **Transfers the platform's share directly from the payer to the platform account**, as a *second*, separate `transfer` call, only if the fee is greater than zero.
      - Records the payment against the merchant's volume/fee analytics (`admin::record_merchant_payment`) and publishes a `PlatformFeeRoutedEvent`.
   5. Sets `invoice.amount_paid += amount` and records `payer` on the invoice (or checks it matches the existing payer).
   6. Sets `invoice.status = Paid` and `invoice.date_paid = now` if `amount_paid == amount`, otherwise `PartiallyPaid`.
   7. Persists the invoice, emits `InvoicePaidEvent` and `PaymentSplitRoutedEvent`, and records a `Transaction` in the merchant's history.
   8. Calls `auto_withdrawal::check_and_trigger_auto_withdrawal`, which may immediately sweep the merchant account's balance in that token to a configured recipient if the merchant's auto-withdrawal threshold has been reached — see [Auto-withdrawal and merchant settlement](./auto-withdrawal.md).

> **Note:** there are exactly two token transfers per payment — payer → merchant account, and payer → platform account — not one transfer followed by an internal split. The merchant account never custodies the platform's cut, even momentarily.

> **Security:** neither `pay_invoice_partial_inner` nor `platform_fee::route_from_payer` takes the [reentrancy guard](../glossary.md#reentrancy-guard) (`reentrancy::enter`/`exit`). The guard is used by the *admin-configuration* functions in `admin.rs` and `platform_fee.rs` (`set_fee`, `set_merchant_platform_fee`, `propose_fee`, `execute_fee`, …), not by the payment path itself. Reentrancy protection for the payment path currently relies on Soroban's own call-frame authorization model and the token contract's behavior, not an explicit flag.

## Walkthrough: `pay_invoice_partial`

`pay_invoice_partial(payer, invoice_id, amount)` is the same `pay_invoice_partial_inner` logic described above, called directly with a caller-chosen `amount` instead of the full remainder:

- **Accumulation.** Each call adds `amount` to `invoice.amount_paid`. There is no separate "partial payment" ledger — `amount_paid` is a running total across every full and partial payment the invoice has received.
- **Resulting status.** If the new `amount_paid` equals `invoice.amount` exactly, the invoice becomes `Paid` (with `date_paid` set). Otherwise it becomes `PartiallyPaid`, whether it was previously `Pending` or already `PartiallyPaid`.
- **Fees on partial amounts.** The fee split is computed on *each individual payment's* `amount`, not on the invoice total — `compute_split` is called fresh for every `pay_invoice_partial` call, using whatever fee rate and volume discount apply *at that moment*. Two partial payments on the same invoice can be charged at different effective fee rates if the merchant's volume crosses a discount tier, or a token's fee changes, between them. There is no invoice-level fee reconciliation step.
- A partial payment cannot overshoot: `amount_paid + amount > invoice.amount` is rejected before any transfer happens.

## Walkthrough: `pay_invoices_batch`

`pay_invoices_batch(payer, invoice_ids)`:

1. Authorizes `payer` once for the entire call.
2. Iterates `invoice_ids` **in the order given** and calls `pay_invoice_inner` (the full-payment path) for each one, without re-authorizing.

**Atomicity.** The loop has no internal error handling — if any invoice in the batch fails a precondition (wrong status, expired, insufficient balance for the transfer, etc.), that call panics, and Soroban unwinds the *entire* transaction: every transfer and storage write made for every invoice processed earlier in the same batch is rolled back too. A batch is all-or-nothing; there is no "pay what you can" partial-success mode and no per-invoice try/catch.

> **Warning:** because a single failing invoice aborts the whole batch, put the invoices most likely to fail (close to expiry, uncertain status) at the end of the list, or validate their state with `get_invoice` off-chain before submitting the batch.

**Practical size limits.** There is no explicit `MAX_BATCH_SIZE` constant in the contract. The real limit is Soroban's per-transaction resource metering: CPU instructions, memory, and the number of ledger entries read/written in one invocation. Each invoice paid in a batch performs multiple storage reads/writes (invoice, merchant, merchant account, analytics, history) and at least two token transfers, so batches are bounded by how many of those a single transaction's resource budget can afford — not by any check in this contract.

## Overpayment, underpayment, and rounding

- **Overpayment** is rejected outright: `pay_invoice_partial_inner` panics with `ContractError::InvalidAmount` if `amount_paid + amount` would exceed `invoice.amount`. A payer cannot send more than the invoice is worth in a single call; there's no automatic refund-the-difference behavior because the excess transfer never happens.
- **Underpayment** is allowed and expected — any positive `amount` less than the remainder moves the invoice to `PartiallyPaid` and leaves the rest payable later, by the same payer, via further `pay_invoice_partial` or `pay_invoice` calls.
- **Rounding in the fee split.** `compute_split` computes `platform_fee = (amount * fee_bps) / 10_000` using integer (floor) division, and `merchant_amount = amount - platform_fee`. Division always truncates toward zero for these non-negative values, so the platform fee rounds **down** and the merchant receives any fractional remainder — the platform never over-collects by rounding up. See [Fees, volume discounts, and time-locked fee changes](./fees.md#the-computation) for worked numeric examples of this rounding.

## Sequence diagram: a full payment

The diagram shows `pay_invoice` for a `Pending` invoice with a non-zero fee.

```mermaid
sequenceDiagram
    participant Payer
    participant Shade as Shade contract
    participant Token as Token contract
    participant Merchant as Merchant account
    participant Platform as Platform account

    Payer->>Shade: pay_invoice(payer, invoice_id)
    Shade->>Shade: assert_not_paused()
    Shade->>Payer: require_auth()
    Shade->>Shade: load invoice, check status/expiry
    Shade->>Shade: compute_split(merchant, token, amount)
    Shade->>Token: transfer(payer, merchant_account, merchant_amount)
    Token->>Merchant: credit merchant_amount
    alt platform_fee > 0
        Shade->>Token: transfer(payer, platform_account, platform_fee)
        Token->>Platform: credit platform_fee
    end
    Shade->>Shade: record_merchant_payment (volume/fee analytics)
    Shade->>Shade: invoice.amount_paid += amount; status = Paid
    Shade->>Shade: emit InvoicePaidEvent, PaymentSplitRoutedEvent
    Shade->>Shade: record Transaction; check auto-withdrawal
    Shade-->>Payer: success
```

## Relevant types and storage

| Type / key | Defined in | Purpose |
|---|---|---|
| `Invoice` | [`contracts/shade/src/types.rs:396-413`](../../contracts/shade/src/types.rs#L396-L413) | The payable record: amount, token, status, `amount_paid`, `payer`. |
| `InvoiceStatus` | [`contracts/shade/src/types.rs:355-366`](../../contracts/shade/src/types.rs#L355-L366) | `Pending`, `Paid`, `Cancelled`, `Refunded`, `PartiallyRefunded`, `PartiallyPaid`, `Draft`. |
| `PlatformFeeSplit` | [`contracts/shade/src/types.rs:656-663`](../../contracts/shade/src/types.rs#L656-L663) | The result of a fee computation: `gross_amount`, `platform_fee`, `merchant_amount`, `fee_bps_applied`. |
| `DataKey::Invoice(u64)` | [`contracts/shade/src/types.rs:64`](../../contracts/shade/src/types.rs#L64) | Persistent storage of each invoice, keyed by invoice ID. |

## Relevant functions

| Function | Defined in | Purpose |
|---|---|---|
| `pay_invoice` | [`contracts/shade/src/interface.rs:139`](../../contracts/shade/src/interface.rs#L139), impl at [`contracts/shade/src/components/invoice.rs:671`](../../contracts/shade/src/components/invoice.rs#L671) | Pay an invoice's full remaining balance. |
| `pay_invoice_partial` | [`contracts/shade/src/interface.rs:141`](../../contracts/shade/src/interface.rs#L141), impl at [`contracts/shade/src/components/invoice.rs:690`](../../contracts/shade/src/components/invoice.rs#L690) | Pay an arbitrary amount toward an invoice. |
| `pay_invoices_batch` | [`contracts/shade/src/interface.rs:140`](../../contracts/shade/src/interface.rs#L140), impl at [`contracts/shade/src/components/invoice.rs:661`](../../contracts/shade/src/components/invoice.rs#L661) | Pay several invoices in full, atomically, under one authorization. |
| `platform_fee::compute_split` | [`contracts/shade/src/components/platform_fee.rs:42`](../../contracts/shade/src/components/platform_fee.rs#L42) | Computes the merchant/platform split for one payment amount. |
| `platform_fee::route_from_payer` | [`contracts/shade/src/components/platform_fee.rs:72`](../../contracts/shade/src/components/platform_fee.rs#L72) | Computes the split, executes both transfers, and records analytics/events. |

## Constraints and edge cases

- A `pay_invoices_batch` call that includes the same invoice ID twice will pay it twice in sequence within the same transaction (second call sees the updated `amount_paid`); there is no de-duplication of `invoice_ids`.
- A partial payment can change the effective fee rate applied to a later partial payment on the *same* invoice, because fees are recomputed per payment, not locked in at invoice creation. See [Fees, volume discounts, and time-locked fee changes](./fees.md).
- A fiat-priced invoice (`InvoicePricingMode::FixedFiat`) only re-quotes its oracle price before the *first* payment (`amount_paid == 0`); once any payment has landed, `invoice.amount` is frozen for the remaining partial payments.

## Related pages

- [Fees, volume discounts, and time-locked fee changes](./fees.md)
- [Invoice lifecycle and statuses](./invoice-lifecycle.md)
- [Auto-withdrawal and merchant settlement](./auto-withdrawal.md)
- [Refunds and voids](./refunds-and-voids.md)
- [Payment payloads, swap routing, and the cross-chain bridge placeholder](./payment-payloads-and-routing.md)

← [Back to concepts](./README.md)
