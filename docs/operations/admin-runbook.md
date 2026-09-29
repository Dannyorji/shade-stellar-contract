# Runbook: Admin operations and incident response

Day-to-day privileged operations on a live `shade` deployment — fee changes, token listing, merchant status, role management, platform account rotation — and the incident response procedure for when something goes wrong.

> **Warning:** Every operation in this runbook is gated by [`core::assert_admin`](../security/admin-and-ownership.md), meaning a single address controls all of it. Treat every command below as security-sensitive: confirm the target contract ID and network before running anything, and record what you did (see [Change log](#change-log)).

## The real admin surface

Before using this runbook, know what the code actually enforces — it differs from a naive reading of the function names:

- **Only three privileged operations check anything beyond plain admin:** `create_invoice_signed`, `restrict_merchant_account`, and `set_merchant_platform_fee`/`clear_merchant_platform_fee` accept **Admin or Manager** (`contracts/shade/src/components/invoice.rs`, `merchant.rs`, `platform_fee.rs`). Every other admin-gated function in the table below requires the admin specifically — `has_role` returns `true` for the admin on *any* role, but that's the admin's blanket authority showing through, not Manager/Operator gaining admin's powers.
- **`Role::Operator` gates nothing.** It exists in the `Role` enum (`contracts/shade/src/types.rs`) and can be granted/revoked/checked via `grant_role`/`revoke_role`/`has_role`, but no function in the current codebase checks for it. Granting `Operator` to an address today has no functional effect beyond `has_role` reporting `true` for it.
- **`assert_has_role` is defined but never called.** Don't assume a generic role-gate exists behind the scenes for functions not listed above.

| Function | Authorization | Notes |
|---|---|---|
| `add_accepted_token(s)` / `remove_accepted_token` | Admin only | |
| `set_fee` | Admin only | Immediate |
| `propose_fee` / `execute_fee` | Admin only | 48-hour time-lock (see [Fees](#routine-operation-fees)) |
| `set_merchant_platform_fee` / `clear_merchant_platform_fee` | Admin or Manager | Per-merchant fee override |
| `set_token_oracle` | Admin only | |
| `set_platform_account` | Admin only | |
| `set_account_wasm_hash` | Admin only | Stores a hash nothing currently reads — see [Deployment Runbook](deployment.md#what-actually-gets-deployed) |
| `set_merchant_status` / `verify_merchant` | Admin only | |
| `restrict_merchant_account` | Admin or Manager | Requires the merchant's `account` to be the `shade` contract's manager (see below) |
| `grant_role` / `revoke_role` | Admin only | |
| `propose_admin_transfer` | Admin only (current admin) | `accept_admin_transfer` requires the *proposed* admin's signature instead |
| `pause` / `unpause` | Admin only | |
| `upgrade` | Admin only | See [Upgrade Runbook](upgrade-runbook.md) |
| `register_bridge_listener` / `remove_bridge_listener` | Admin only | |
| `add_gov_member` / `remove_gov_member` / `set_governance_config` | Admin only | |
| `set_multisig_threshold` / `configure_multisig` | Admin only | |
| `create_campaign_category` / `update_campaign_category` | Admin only | |

## Routine operation: fees

### `set_fee` — immediate

```bash
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- set_fee --admin <admin> --token <token> --fee 500
```

Applies instantly. `fee` is in basis points (`500` = 5%; `10_000` = 100%). Fails with `TokenNotAccepted` if the token isn't whitelisted yet — run `add_accepted_token` first. **Treat this as a privileged financial operation**: an immediate fee change with no notice affects every payment from the moment it lands, and there is no on-chain cap — the admin can set a fee as high as `10_000` bps with nothing in the contract to stop it (see [Threat model § Fee configuration](../security/threat-model.md)).

### `propose_fee` → `execute_fee` — time-locked (recommended for routine changes)

```bash
# Propose (admin only)
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- propose_fee --admin <admin> --token <token> --fee 500

# Wait at least 172,800 seconds (48 hours) — the constant is FEE_UPDATE_DELAY in contracts/shade/src/components/admin.rs

# Execute (admin only) — fails with FeeUpdateTooEarly if called before the 48h delay elapses
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- execute_fee --admin <admin> --token <token>
```

- **Who can propose:** admin only.
- **Who can execute:** admin only (not automatic — someone must call `execute_fee` after the delay).
- **Timing:** exactly 48 hours (`172_800` seconds) from `proposed_at`. `execute_fee` succeeds at exactly 48h elapsed, not strictly after.
- **Token scope:** one token per `propose_fee` call; there is one pending slot per token (`DataKey::PendingTokenFee(token)`) — proposing again for the same token before executing overwrites the pending value.
- **State transition:** on execute, `DataKey::TokenFee(token)` is set to the proposed value and the pending entry is removed.
- **Verification:** `get_pending_fee --token <token>` while pending; `get_fee --token <token>` after execution.
- **Event verification:** `FeeProposedEvent` on propose, `FeeSetEvent` on both `set_fee` and `execute_fee` — an indexer watching only `FeeSetEvent` cannot distinguish an immediate change from a time-locked one landing; watch `FeeProposedEvent` too if that distinction matters to your monitoring.
- **Change-log entry:** record both the propose and execute transactions (see [Change log](#change-log)) — a fee change that was proposed but never executed, or executed by a different operator than who proposed it, should be visible in your log.

> **Note:** `set_fee` and `propose_fee`/`execute_fee` are independent — nothing stops an admin from using `set_fee` to bypass the time-lock on a token that also has a pending proposal. If your team's policy is "always use the time-locked path," that's a process discipline, not something the contract enforces.

## Routine operation: token listing

### Add an accepted token

```bash
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- add_accepted_token --admin <admin> --token <token>
# or, for several at once:
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- add_accepted_tokens --admin <admin> --tokens '["<token-1>","<token-2>"]'
```

Verify: `is_accepted_token --token <token>` → `true`.

### Configure fee and oracle for the token

Being "accepted" only whitelists the token for use — it does not set a fee or oracle. These are three genuinely distinct configuration states; do not conflate them:

| State | Set by | Check with |
|---|---|---|
| Accepted by the protocol at all | `add_accepted_token`/`add_accepted_tokens` | `is_accepted_token` |
| Configured for fees | `set_fee` or `propose_fee`/`execute_fee` | `get_fee` (returns `0` if never set — indistinguishable from an intentional zero fee) |
| Configured for fiat pricing/oracle usage | `set_token_oracle` | `get_token_oracle` (panics `OracleNotConfigured` if unset) |

A token can be accepted with no fee configured (fee defaults to `0`) and no oracle configured (fine for crypto-denominated invoices; required before anyone can use it with `create_fiat_invoice`).

### De-listing a token

```bash
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- remove_accepted_token --admin <admin> --token <token>
```

This removes the token from the accepted-tokens whitelist so new invoices/subscriptions/campaigns can no longer be created in it. It does **not** clear the token's fee (`get_fee` still returns whatever was last set) or its oracle config — re-adding the same token later restores its old fee/oracle configuration implicitly, which may or may not be what you want. There is no separate "clear fee" or "clear oracle" call for a de-listed token in the current interface; if you need those values reset, call `set_fee --fee 0` explicitly before or after de-listing.

## Merchant operations

Merchant state is not a simple active/inactive flag — `Merchant` (`contracts/shade/src/types.rs`) carries `active` and `verified` as two independent booleans, plus `account`, `webhook`, and auto-withdrawal configuration. Know which dimension you're changing.

### Activation / deactivation

```bash
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- set_merchant_status --admin <admin> --merchant_id <id> --status false
```

Admin-only. `status: false` deactivates; `status: true` reactivates. Verify: `is_merchant_active --merchant_id <id>`.

### Verification

```bash
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- verify_merchant --admin <admin> --merchant_id <id> --status true
```

Admin-only, independent of active/inactive. Verify: `is_merchant_verified --merchant_id <id>`.

### Account restriction

```bash
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- restrict_merchant_account --caller <admin-or-manager> --merchant_address <address> --status true
```

Admin or Manager. This calls `restrict_account` on the merchant's linked `account` contract (falling back to the merchant's own address if no separate account is set — `contracts/shade/src/components/merchant.rs`). For this call to succeed, the merchant's `account` contract's `manager` field must be the `shade` contract's own ID, since `shade` has to satisfy that account's `manager.require_auth()` internally — this is only relevant if the merchant has a real deployed `account` instance; see [Deployment Runbook](deployment.md#what-actually-gets-deployed) for why most merchants won't by default.

### State inspection

```bash
stellar contract invoke --id <shade-id> --network <network> -- get_merchant --merchant_id <id>
```

Returns the full `Merchant` record — `active`, `verified`, `account`, `webhook`, `auto_withdrawal_recipient`, `auto_withdrawal_thresholds`, `date_registered` — in one call.

### Audit requirement

Every status/verification/restriction change is admin- or Manager-attributable and emits an event (`MerchantStatusChangedEvent`, `MerchantVerifiedEvent`, `AccountRestrictedEvent`) — record the transaction ID and the reason in your [change log](#change-log) for each.

## Role management

```bash
# Grant
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- grant_role --admin <admin> --user <address> --role Manager

# Revoke
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- revoke_role --admin <admin> --user <address> --role Manager

# Verify
stellar contract invoke --id <shade-id> --network <network> -- has_role --user <address> --role Manager
```

Both `grant_role` and `revoke_role` are admin-only. As established in [The real admin surface](#the-real-admin-surface), only `Manager` currently gates anything — grant it deliberately to relayers/operators who need `create_invoice_signed`, `restrict_merchant_account`, or per-merchant fee override access, and treat every grant as expanding the set of addresses that can perform those specific privileged actions.

### Admin transfer

Two-step, not a direct write — see [Admin ownership](../security/admin-and-ownership.md#two-step-admin-transfer) for the full mechanism and why:

```bash
# Step 1 — current admin proposes
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- propose_admin_transfer --admin <current-admin> --new_admin <new-admin>

# Step 2 — proposed new admin accepts, signing as themselves
stellar contract invoke --id <shade-id> --network <network> --source-account new-admin -- accept_admin_transfer --new_admin <new-admin>
```

- **Authorized caller (step 1):** current admin.
- **Target account:** the proposed new admin — recorded under `DataKey::PendingAdmin`, overwritten by any subsequent `propose_admin_transfer` call before it's accepted.
- **Required confirmation:** the new admin must independently prove control by signing `accept_admin_transfer` themselves — this is what step 2 enforces; there is no way to complete the transfer with only the current admin's signature.
- **Verification:** `get_admin` returns the new address after step 2; test a simple admin-gated read (`is_paused` or any admin-only call) signed by the new admin before considering the transfer complete; confirm the old admin key can no longer perform admin operations.
- **Artifact/change-log entry:** record both transaction IDs, and rehearse this on testnet before doing it on a live deployment with real merchants (see [Admin ownership § Rehearsal](../security/admin-and-ownership.md#rehearsal)).

## Platform account rotation

```bash
# 1. Verify current
stellar contract invoke --id <shade-id> --network <network> -- get_platform_account

# 2. (out-of-band) confirm the target account is correct and controlled by the intended party

# 3. (out-of-band) obtain required internal approval before executing

# 4. Execute
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- set_platform_account --admin <admin> --account <new-platform-account>

# 5. Verify new
stellar contract invoke --id <shade-id> --network <network> -- get_platform_account

# 6. Verify dependent functionality — the next payment's platform-fee routing lands at the new account
stellar contract invoke --id <shade-id> --network <network> -- get_merchant_analytics --merchant <test-merchant> --token <token>
```

Admin-only, applies immediately — the very next `pay_invoice`/`pay_invoice_partial`/`charge_subscription`/ticket purchase that routes a platform fee (`PlatformFeeRoutedEvent`) sends it to the new account, with no transition period. This has no special impact on merchants, factories, or fees beyond where the platform's cut of each transaction is sent — merchant amounts, fee *rates*, and accepted-token configuration are all untouched by this call (`contracts/shade/src/components/admin.rs`). Record the transaction and the change per [Change log](#change-log).

## Admin key custody

This repository establishes no specific custody policy (no hardware-wallet or multisig requirement is enforced anywhere in the code — the admin is a single `Address`, full stop). The following are baseline operational requirements your team should confirm explicitly, not treat as already decided:

- **Never place a private key in source control**, in this repository or in any deployment tooling built around it. No secret key appears anywhere in this repo's history as of this writing, and none should ever be added.
- **Store the admin key in an environment/secret-manager system** appropriate to your infrastructure — this repo does not prescribe one.
- **Separate deployer and admin identities where practical.** The Makefile's `deploy-shade`/`init-shade` targets use one `--source-account default` identity for both deploying and initializing by convention, but nothing requires the deployer and the long-term admin to be the same key — consider transferring admin to a separate, better-secured identity immediately after `initialize` via the [two-step transfer](#admin-transfer) above.
- **Least privilege:** grant `Manager` only to the specific addresses that need `create_invoice_signed`, `restrict_merchant_account`, or fee-override access — see [The real admin surface](#the-real-admin-surface).
- **Require review before signing** any admin transaction, especially `set_fee`, `upgrade`, `set_account_wasm_hash`, and `set_platform_account` — these carry no on-chain safety rails (no caps, no timelock except `propose_fee`/`execute_fee`).
- **Key rotation** is the [two-step admin transfer](#admin-transfer) procedure above — there is no other rotation mechanism.
- **Emergency access:** if the admin key is lost and no transfer was ever proposed, the protocol has **no recovery path** — `propose_admin_transfer` and everything downstream of it requires the current admin's signature; a lost, un-transferred admin key permanently locks every admin-gated function, including `pause` and `upgrade` (see [Admin ownership § Compromised admin key](../security/admin-and-ownership.md#compromised-admin-key--blast-radius) for the parallel compromise scenario).
- **Record every privileged operation** — see [Change log](#change-log).

These are **operational requirements needing your team's explicit confirmation**, not established facts about this codebase — flag any gap here (e.g. "we don't yet have a secret manager wired up for the admin key") rather than assuming one exists.

## Change log

Record, at minimum, for every privileged action taken against a live deployment:

```text
Timestamp:            <UTC ISO 8601>
Operator role:        <who executed this, and in what capacity — e.g. "release operator">
Environment/network:  <testnet | mainnet | local>
Contract ID:          <C... address>
Action:               <function name, e.g. set_fee>
Parameters:           <arguments passed, excluding any secrets>
Transaction hash:     <tx hash>
Resulting ledger:      <ledger sequence>
Reviewer/approver:    <who signed off, if this action required approval>
Reason/change ticket: <why — link to an issue/ticket if one exists>
Verification result:  <the read-back command's result, confirming the change landed as intended>
```

On-chain transaction and event data (`FeeSetEvent`, `MerchantStatusChangedEvent`, `RoleGrantedEvent`, etc.) is the authoritative technical record of *what happened*; your change log's job is to capture *why* and *who approved it*, which the chain does not record.

## Incident response

`Detect → Assess → Contain → Diagnose → Remediate → Verify → Unpause → Communicate → Review`

### Detection signals

Signals that actually exist or can be obtained from this protocol, without assuming monitoring infrastructure this repo doesn't provide:

- Failed transactions against the `shade` contract ID (visible via any RPC/explorer watching that address).
- Unexpected events — an admin-gated event (`FeeSetEvent`, `RoleGrantedEvent`, `ContractUpgradedEvent`, `PlatformAccountSetEvent`) you did not initiate.
- Incorrect fee behavior — `calculate_fee`/`compute_platform_fee_split` returning a value inconsistent with the configured `get_fee`.
- Token/oracle anomalies — `get_token_oracle` returning a changed `contract` address, or fiat-priced invoice amounts resolving inconsistently.
- Merchant reports of missing/incorrect payments.
- Widespread `pay_invoice`/`charge_subscription` failures with no obvious single cause.
- Suspicious admin operations — any `set_*`, `grant_role`, or `upgrade` event you cannot attribute to a known, approved action in your change log.
- Unexpected state changes on inspection (a merchant's `active`/`verified` flags, accepted-tokens list, or role assignments differing from your last known-good record).
- Analytics inconsistencies — `get_merchant_volume`/`get_token_volume` diverging from your own independently-tracked totals.

This repository has no built-in monitoring/alerting — do not assume a dashboard or alert exists unless your team has separately built one; these are things to *check*, not things that will *page you* on their own.

### Pause decision

Objective criteria for pausing immediately:

- A `ContractUpgradedEvent`, `FeeSetEvent`, `RoleGrantedEvent`, `AdminTransferProposedEvent`, or `PlatformAccountSetEvent` you cannot attribute to an approved action in your change log (**unauthorized privileged change**).
- Confirmed incorrect fee charging on live payments.
- Confirmed incorrect token/oracle behavior affecting settlement amounts.
- A suspected path to fund loss (e.g. a reentrancy or authorization bug reported or discovered).
- A violated invariant you can demonstrate on-chain (e.g. `amount_refunded > amount_paid` on an invoice, which should be structurally impossible).
- Widespread payment failures with the root cause not yet identified.
- Anything unexpected during or immediately after an [upgrade](upgrade-runbook.md).

**Investigate without pausing** for: a single merchant's isolated complaint with a plausible non-protocol explanation (wrong token, insufficient balance, expired invoice), a cosmetic/read-path issue with no effect on funds, or a one-off transaction failure with an identifiable, non-recurring cause (e.g. a client submitted a malformed argument).

**Pause pending confirmation** for: a pattern that looks like it could be the above list but isn't yet confirmed — for example, one report of an incorrect fee that could be a client-side calculation error or could be a real contract-side issue. Escalate to [Diagnosis](#triage-and-diagnosis) first with a short, bounded investigation window before deciding.

Do not make pausing the default response to every report — see [Pausable § Risks of pausing with in-flight state](../security/pausable.md#risks-of-pausing-with-in-flight-state) for the real costs of an unnecessary pause (frozen subscriptions accruing overdue charges, invoices expiring while frozen, escrow deadlines passing).

### Pause procedure

```bash
# 1. Confirm target contract/network explicitly
echo "Target: <shade-id> on <network>"

# 2. Verify operator authority — confirm you're signing as the actual admin
stellar contract invoke --id <shade-id> --network <network> -- get_admin

# 3. Execute
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- pause --admin <admin-address>

# 4. Confirm paused state — never trust a UI indicator, read the contract directly
stellar contract invoke --id <shade-id> --network <network> -- is_paused
# true

# 5. Record the transaction in the change log immediately
# 6. Notify affected parties — see Communication plan in the Upgrade Runbook for the template shape
```

### Triage and diagnosis

Deterministic order of investigation, using only what's actually queryable from this contract:

1. **Confirm pause state and admin identity** — `is_paused`, `get_admin`. Rule out that the incident *is* an admin-key compromise before doing anything else that assumes you still control the admin key.
2. **Read the specific affected state** — the merchant, invoice, subscription, or campaign named in the report (`get_merchant`, `get_invoice`, `get_subscription`, `get_campaign`).
3. **Read the relevant configuration** — `get_fee`, `is_accepted_token`, `get_token_oracle` for the token(s) involved.
4. **Check role/admin state for the affected operation** — `has_role` for whoever executed the suspicious action, if known.
5. **Cross-check the transaction history** — pull the actual transaction and its emitted events from an RPC/explorer for the contract ID, for the time window around the report.
6. **Check the deployment/WASM identity** — confirm no unexpected `ContractUpgradedEvent` exists in the event log for this contract (see [Upgradeability § Establishing what is deployed](../architecture/upgradeability.md#establishing-what-is-deployed)); an upgrade you didn't expect is itself the incident.
7. **Compare against your last known-good change log entry** for anything in the affected area — the discrepancy between "what the log says should be true" and "what `get_*` actually returns" is usually where the incident is.

### Remediation

Based on actual protocol capabilities — do not promise a remediation this contract cannot perform:

| Remediation | How |
|---|---|
| Configuration correction | Re-run the correct `set_fee`/`add_accepted_token`/`set_token_oracle`/`set_platform_account` call (see the routine operations above). |
| Role correction | `revoke_role` the wrong grant, `grant_role` the correct one. |
| Token/oracle correction | `remove_accepted_token` + re-`add_accepted_token` with correct config, or `set_token_oracle` with corrected values. |
| Fee correction | `set_fee` (immediate) if the incident is ongoing and urgent; `propose_fee`/`execute_fee` if there's no urgency and you want the 48h notice period to apply going forward. |
| Account rotation | [Platform account rotation](#platform-account-rotation) or [admin transfer](#admin-transfer), as appropriate to what was compromised. |
| Upgrade | See [Upgrade Runbook](upgrade-runbook.md) — only if the root cause is a code defect, not a configuration error. |
| Rollback | Only meaningful for an upgrade-caused incident, and only if no incompatible state was written since — see [Upgrade Runbook § Rollback](upgrade-runbook.md#rollback). |

**Irreversible operations** — cannot be remediated after the fact, only mitigated going forward: funds already transferred via `pay_invoice`/`charge_subscription`/`refund_invoice` cannot be clawed back by the contract (no admin "force transfer back" exists); a fee already charged on a completed payment cannot be retroactively adjusted; an already-finalized DAO `upgrade` (via `finalize_upgrade`) that turns out to be wrong requires a forward-fix or the (conditional) rollback path above, not an "undo."

### Unpause criteria

All of the following, not just the first one that becomes true:

- [ ] Root cause identified, or risk fully contained (e.g. the compromised key rotated out via admin transfer, even if full root-cause analysis is still ongoing).
- [ ] Remediation from the table above has been applied.
- [ ] State verified correct — re-run the specific `get_*` checks from [Triage and diagnosis](#triage-and-diagnosis) step 2-4 and confirm they now show the expected values.
- [ ] `cargo test -p shade` (and any newly-added regression test for this specific incident) passes, if the remediation involved a code change.
- [ ] The [Deployment Runbook's smoke test](deployment.md#smoke-test) passes against the now-fixed contract.
- [ ] Security/operations approval obtained — a second person confirms the fix, independent of whoever applied it.
- [ ] Merchant communication sent, where the incident was visible to them (see the [Upgrade Runbook's communication plan](upgrade-runbook.md#communication-plan) for the template shape — reuse it here).

Unpause is a deliberate, checked decision — never "resume once the fix transaction succeeds." A remediation transaction succeeding tells you the transaction was valid; it does not by itself tell you the underlying problem is fixed.

```bash
stellar contract invoke --id <shade-id> --network <network> --source-account admin -- unpause --admin <admin-address>
stellar contract invoke --id <shade-id> --network <network> -- is_paused
# false
```

## Severity and escalation

Proposed operational targets — this repository has no existing SLA commitments; treat the response-time figures below as a starting point for your team to confirm or adjust, not an established fact:

### Critical

Potential fund loss, unauthorized privileged control (an admin action you didn't authorize), a compromised admin key, incorrect financial accounting at scale, or inability to establish which WASM is actually deployed.

- **Target response:** immediate (proposed: page within 15 minutes).
- **Who is paged:** incident/on-call owner and protocol engineer, immediately; security reviewer as soon as available.
- **Pause:** yes, immediately, per [Pause decision](#pause-decision).
- **Escalation path:** on-call owner → protocol engineer + security reviewer → product/operations owner.
- **Communication:** immediate internal notification; external (merchant-facing) notification within the window defined by your [Upgrade Runbook communication plan](upgrade-runbook.md#communication-plan) template, adapted for an unplanned incident.

### High

Major payment disruption, incorrect configuration affecting many merchants, or serious security degradation without confirmed fund loss.

- **Target response:** proposed within 1 hour.
- **Who is paged:** protocol engineer and incident/on-call owner.
- **Pause:** considered per [Pause decision](#pause-decision) — not automatic, but the default lean is toward pausing if the affected scope is genuinely "many merchants."
- **Escalation path:** on-call owner → protocol engineer; escalate to Critical handling if scope grows or fund loss is confirmed.

### Medium

Limited merchant/integration impact with a known workaround.

- **Target response:** proposed within 1 business day.
- **Who is paged:** protocol engineer, during normal hours.
- **Pause:** generally no — see [Pausable § Known limitations](../security/pausable.md#known-limitations) for why a global pause is a blunt instrument for a limited-scope issue.

### Low

Documentation, isolated operational, or non-critical issues.

- **Target response:** proposed within the normal development/review cycle.
- **Who is paged:** no page; tracked as a normal issue.
- **Pause:** no.

## Post-incident procedure

1. **Preserve evidence** — export the relevant transaction hashes, event payloads, and `get_*` reads taken during triage before they're needed for a writeup; don't rely on re-querying the same state later, since it may have changed.
2. **Reconstruct the timeline** from the change log and on-chain transaction/event history — what was configured/deployed when, in what order.
3. **Collect transaction/event/ledger records** for everything in scope of the incident.
4. **Identify root cause** — a specific function, configuration value, or key-management failure, not just a symptom.
5. **Determine affected merchants/transactions** — which specific invoice IDs, merchant IDs, or subscription IDs were touched.
6. **Communicate impact** to affected merchants specifically, not only a general "incident resolved" notice.
7. **Document remediation** — what was changed, by whom, with what verification (this should already exist in your change log; consolidate it into the incident record).
8. **Create follow-up tasks** — anything from "add a regression test" to "the team should adopt X monitoring" surfaced during the incident.
9. **Update tests/runbooks/threat model** — if this incident revealed a gap in [Threat model](../security/threat-model.md)'s residual-risk table or in this runbook, update the source document, not just an internal postmortem doc.
10. **Close the incident** only after the above are complete and verified — not once the immediate symptom stops.

Cross-reference on-chain transaction and event data as the authoritative record throughout — it's the one part of this process that cannot be misremembered or edited after the fact.

## Related pages

- [Pausable emergency-stop mechanism](../security/pausable.md) — full function-by-function pause behavior.
- [Admin ownership and two-step admin transfer](../security/admin-and-ownership.md) — the mechanism behind [Role management](#role-management) and key-compromise blast radius.
- [Upgrade Runbook](upgrade-runbook.md) — the procedure when remediation requires new contract code.
- [Deployment Runbook](deployment.md) — initial setup this runbook assumes is already done, and the smoke test reused above.
- [Threat model and security assumptions](../security/threat-model.md) — the trust assumptions and attack surfaces this runbook's detection signals are built from.

← [Back to Operations](README.md)
