# Runbook: Deploying the Shade Protocol contracts

Deploy the `shade` hub contract (and, where your integration needs them, the standalone `account`, `escrow_factory`, `ticketing_factory`, and `crowdfund_factory` contracts) to local, testnet, or mainnet, and bring the deployment to a usable state.

> **Warning:** Once merchants have registered and invoices exist against a `shade` contract ID, that ID is permanent for that deployment's data — see [Upgradeability](../architecture/upgradeability.md) for what an upgrade can and cannot change. Mainnet deployment is effectively a one-way door for the identity of the contract your integrators build against.

## Known-broken baseline — verify before you start

As of this writing (`main` at `027ef11`), the repository does **not** build cleanly:

- `cargo build -p shade --release` fails with ~120 compiler errors (duplicate `#[contracttype]` implementations for `VestingSchedule`, missing `DataKey`/`ContractError` variants referenced by an in-progress KYC feature, and related issues from overlapping feature-branch merges).
- `cargo build -p crowdfund --release` fails separately (non-exhaustive `DataKey` match in `contracts/crowdfund/src/lib.rs`).
- GitHub Actions CI (`ci.yml`) has been failing on `main` across the last several merges.
- `account`, `escrow`, `escrow_factory`, `ticketing`, `ticketing_factory`, and `subscription` build and produce WASM cleanly in isolation (`cargo build -p <crate> --target wasm32-unknown-unknown --release`).

**Do not deploy `shade` or `crowdfund` from `main` at this commit.** Before running any command in this runbook:

```bash
cargo build --workspace --release
cargo test -p shade
```

If either fails, either wait for the breakage to be fixed upstream, or pin your deployment to the last commit where both passed (check `gh run list --branch main` or the Actions tab for the most recent green `CI` run and deploy from that commit's SHA, not from `main`'s tip). Do not "fix" the compile errors yourself as part of a deployment — that is a code change with its own review, not an operations task.

## Pre-deployment checklist

- [ ] `cargo build --workspace --release` succeeds (see above).
- [ ] `cargo test -p shade` and `cargo test --workspace` pass.
- [ ] `cargo fmt --all -- --check` and `cargo clippy --workspace --all-features -- -D warnings` pass (the same gates CI runs).
- [ ] Working tree is clean and you know the exact commit SHA you are deploying (`git rev-parse HEAD`).
- [ ] WASM built with the plain `release` profile via `make optimize` (or the manual `cargo build --release` + `stellar contract optimize` sequence in [Building the Contracts](../getting-started/building.md)) — never `release-with-logs` for anything beyond local scratch testing.
- [ ] Toolchain matches [Prerequisites](../getting-started/prerequisites.md): Rust ≥ 1.84.0, Stellar CLI 23.x, `soroban-sdk` 23.5.3 (pinned in `Cargo.lock`).
- [ ] Deployer identity funded on the target network (`stellar keys generate`/`stellar keys fund` for testnet/local; a real funded mainnet account for mainnet — see the [Mainnet](#mainnet) section).
- [ ] Admin address decided (may be the same as the deployer, or a separate address/multisig — see [Admin ownership](../security/admin-and-ownership.md)).
- [ ] Accepted tokens and their fee rates decided for this environment.
- [ ] Fiat-priced tokens' oracle contracts identified, if you're using [fiat pricing](../concepts/fiat-pricing-and-oracles.md).
- [ ] A place to record contract IDs and WASM hashes for this deployment (see [Artifact recording](#artifact-recording)).

## What actually gets deployed

Read this before following the issue tracker's or anyone else's mental model of "the deployment": **the four standalone factories (`escrow_factory`, `ticketing_factory`, `crowdfund_factory`, and the `account` contract itself) are not wired into `shade` at all.** There is no `set_escrow_factory`, `set_ticketing_factory`, or equivalent call anywhere in `contracts/shade/src/`, and `shade`'s own `account_factory` component (`contracts/shade/src/components/account_factory.rs`) is dead code — `register_merchant` (`contracts/shade/src/components/merchant.rs`) does not call it. `set_account_wasm_hash` therefore stores a hash that nothing in the current code path reads.

This means there is no cross-contract "deployment ordering" enforced by the protocol the way earlier issue drafts assumed. What you actually have is:

1. **`shade`** — the payment hub. Fully self-contained once initialized and configured; does not need any other contract deployed to accept invoice payments.
2. **`account`** (optional per merchant) — a separate contract you deploy once per merchant who wants an on-chain vault instead of receiving funds directly at their wallet address. Independent of `shade`'s deployment.
3. **`escrow_factory`**, **`ticketing_factory`**, **`crowdfund_factory`** — fully independent contracts. Deploy any of them, in any order, only if your integration uses on-chain escrow, ticketing, or crowdfunding *outside* `shade`'s own built-in escrow/ticketing/campaign components (`shade` has its own internal escrow, event, and campaign logic — see [ShadeTrait reference](../reference/shade-interface.md) — that does not go through these factories at all).

If your integration only needs invoices, subscriptions, `shade`'s built-in escrow/tickets/campaigns, and merchants paid directly to their wallets, **you only need to deploy `shade`.** Deploy the factories and `account` only if you specifically need the standalone contracts they produce.

## Local

### Requirements

- `stellar network container start local` (or an equivalent local Soroban RPC/Horizon setup) — see the [Stellar quickstart docs](https://developers.stellar.org/docs/build/smart-contracts/getting-started/setup) for the current local-network container command, since this repository has no local-network script of its own.
- A funded local identity: `stellar keys generate deployer --network local` then `stellar keys fund deployer --network local`.

### Build and deploy

```bash
make build ADMIN=<admin-address>
make optimize
NETWORK=local make deploy-shade ADMIN=<admin-address>
NETWORK=local make init-shade ADMIN=<admin-address>
```

`make deploy-shade` runs `stellar contract deploy --wasm target/wasm32-unknown-unknown/release/shade.optimized.wasm --network local --source-account default` and writes the resulting contract ID to `.stellar/shade_contract_id.txt`. `make init-shade` reads that file and runs `stellar contract invoke --id $(cat .stellar/shade_contract_id.txt) --network local --source-account default -- initialize --admin $(ADMIN)`.

> **Note:** The Makefile's `deploy-shade`/`deploy-account`/`init-shade` targets use `--source-account default`, a CLI identity named `default` — set one up with `stellar keys generate default --network local` (or override by invoking the underlying `stellar contract deploy`/`invoke` commands directly with your own `--source-account`) before running `make`.

### Contract IDs

Read back what was deployed:

```bash
cat .stellar/shade_contract_id.txt
```

### Initialization

`make init-shade` (above) is the entire initialization step for a fresh `shade` deployment — `initialize(admin)` sets the admin, sets the platform account to that same admin address, and records `ContractInfo`. See [Post-deployment configuration](#post-deployment-configuration) for what to run next before the deployment is actually usable.

### Smoke test

No scripted smoke test exists in this repository (see [Smoke test](#smoke-test) below for what to run manually). On local, the fastest check is:

```bash
stellar contract invoke --id $(cat .stellar/shade_contract_id.txt) --network local -- get_admin
# should print the admin address you passed to init-shade
```

## Testnet

### Requirements

- A funded testnet identity: `stellar keys generate deployer --network testnet` then `stellar keys fund deployer --network testnet` (this calls Friendbot).
- Decide accepted tokens for testnet — typically a test SAC (Stellar Asset Contract) token you also control, or one of the well-known testnet token contracts your team already uses for integration testing.

### Deploy and initialize

```bash
make build ADMIN=<admin-address>
make optimize
NETWORK=testnet make deploy-shade ADMIN=<admin-address>
NETWORK=testnet make init-shade ADMIN=<admin-address>
```

If you also need a merchant `account` instance for testing:

```bash
NETWORK=testnet make deploy-account
```

`deploy-account` deploys a standalone `account` WASM instance but does **not** call its `initialize` — there is no `init-account` Makefile target. Call it directly:

```bash
stellar contract invoke \
  --id $(cat .stellar/account_contract_id.txt) \
  --network testnet \
  --source-account default \
  -- \
  initialize \
  --merchant <merchant-address> \
  --manager <shade-contract-id> \
  --merchant_id <merchant-id>
```

`manager` must be the `shade` contract's ID — every manager-gated call on the account (`add_token`, `refund`, `verify_account`, `restrict_account`, `set_withdrawal_threshold`, `approve_withdrawal`) requires `manager.require_auth()`, and `shade`'s `restrict_merchant_account` is what's meant to satisfy that. `initialize` itself has no auth check beyond a one-time guard (`contracts/account/src/account.rs`) — anything that gets to call it first wins, so run it immediately after deploying, before publishing the contract ID anywhere.

### Post-deployment configuration

Run these against the deployed `shade` contract ID before treating the deployment as usable. All require the admin's authorization (`--source-account` must be able to sign as the admin, or matching `--source-account`/`--admin` if admin and deployer are the same identity):

| Step | Command | Purpose | Verify |
|---|---|---|---|
| 1. Whitelist tokens | `stellar contract invoke --id <shade-id> --network testnet --source-account admin -- add_accepted_tokens --admin <admin> --tokens '["<token-1>","<token-2>"]'` | Invoices, subscriptions, and campaigns can only settle in accepted tokens. | `stellar contract invoke --id <shade-id> --network testnet -- is_accepted_token --token <token-1>` → `true` |
| 2. Set the platform account | `stellar contract invoke --id <shade-id> --network testnet --source-account admin -- set_platform_account --admin <admin> --account <fee-recipient>` | `initialize` defaults the platform account to the admin address; change it here if fees should go elsewhere. | `stellar contract invoke --id <shade-id> --network testnet -- get_platform_account` |
| 3. Set fees per token | `stellar contract invoke --id <shade-id> --network testnet --source-account admin -- set_fee --admin <admin> --token <token-1> --fee 500` | Fee in basis points (`500` = 5%). `set_fee` applies immediately; use `propose_fee`/`execute_fee` instead for a 48-hour time-locked change (see [Admin Runbook § Fees](admin-runbook.md#routine-operation-fees)). | `stellar contract invoke --id <shade-id> --network testnet -- get_fee --token <token-1>` |
| 4. Configure oracles for fiat-priced tokens | `stellar contract invoke --id <shade-id> --network testnet --source-account admin -- set_token_oracle --admin <admin> --token <token-1> --oracle '{"contract":"<oracle-id>","price_decimals":14,"token_decimals":7}'` | Only needed for tokens used with `create_fiat_invoice`. | `stellar contract invoke --id <shade-id> --network testnet -- get_token_oracle --token <token-1>` |
| 5. Grant initial roles (optional) | `stellar contract invoke --id <shade-id> --network testnet --source-account admin -- grant_role --admin <admin> --user <address> --role Manager` | Only `Manager` gates anything in the current code (`create_invoice_signed`, `restrict_merchant_account`, per-merchant fee overrides) — `Operator` gates nothing yet. Skip this step if you have no signed-invoice relayer or fee-operator to authorize. | `stellar contract invoke --id <shade-id> --network testnet -- has_role --user <address> --role Manager` |

`set_account_wasm_hash` is **not** in this list. It stores a hash that is never read by any code path reachable from `register_merchant` (see [What actually gets deployed](#what-actually-gets-deployed)) — do not spend a deployment step on it unless you have your own tooling that reads `DataKey::AccountWasmHash` directly.

### Smoke test

No automated smoke-test script exists in `scripts/` (only `scripts/setup-hooks.sh` is present, despite `docs/architecture/workspace-layout.md` describing deployment/testing scripts that do not currently exist — flagged there as stale). Run this sequence manually, substituting a real test token and addresses:

```bash
# 1. Register a merchant
stellar contract invoke --id <shade-id> --network testnet --source-account merchant -- register_merchant --merchant <merchant-address>

# 2. Create an invoice
stellar contract invoke --id <shade-id> --network testnet --source-account merchant -- create_invoice \
  --merchant <merchant-address> --description "smoke test" --amount 1000000 --token <token-1>

# 3. Pay the invoice
stellar contract invoke --id <shade-id> --network testnet --source-account payer -- pay_invoice \
  --payer <payer-address> --invoice_id 1

# 4. Verify
stellar contract invoke --id <shade-id> --network testnet -- get_invoice --invoice_id 1
# status should be Paid (1); amount_paid should equal the invoice amount
```

For a broader pre-merge regression check (not a lightweight post-deploy smoke test), `contracts/shade/src/tests/test_payment.rs::test_successful_payment_with_fee` and `contracts/shade/src/tests/test_auto_withdrawal.rs` are the most complete register → invoice → pay flows in the test suite, including a real `account` contract instance in the latter. Both run against the native build via `cargo test`, not against a deployed network — they verify the logic, not a specific deployment.

> **Note:** `pay_invoice` requires the merchant to have a resolvable account. If you skip `set_merchant_account`, the merchant's `account` field defaults to their own wallet address at registration (`register_merchant` in `contracts/shade/src/components/merchant.rs`), which is enough for the smoke test above to succeed without deploying a separate `account` contract. `test_payment_merchant_account_not_set` in `test_payment.rs` documents the one case where an unset account causes a failure.

## Mainnet

> **Warning:** Do not shortcut any step below to "save time." A mainnet deployment with the wrong admin address, an unreviewed WASM, or a skipped hash verification is not something you can quietly redo — merchants and integrators will build against the contract ID you publish.

### Before you deploy

- [ ] Deploying from a tagged release commit, not an arbitrary `main` HEAD. Confirm `git rev-parse HEAD` matches the tag and that CI was green on that exact commit (see [Known-broken baseline](#known-broken-baseline-verify-before-you-start) — do not deploy from a commit where CI failed).
- [ ] `cargo test --workspace --all-features` passes locally against that commit, in addition to CI.
- [ ] The optimized WASM was built with the plain `release` profile (never `release-with-logs`).
- [ ] The WASM hash has been computed and independently confirmed by at least one other team member building from the same tagged commit (see [Upgradeability § Versioning](../architecture/upgradeability.md#versioning-and-identifying-a-deployed-build) for why reproducible builds are the only real verification here).
- [ ] The deployer identity is explicitly confirmed to be a mainnet-funded account, and the person running the command has read back `stellar keys address <deployer>` and cross-checked it against the intended funding source before deploying — a testnet identity reused against `--network mainnet` fails safely (insufficient funds), but a *different, unintended, but funded* mainnet identity would not fail at all.
- [ ] The intended admin address is confirmed by a second person, out of band from the deployment command itself. If the admin is a multisig or a contract, its ability to call `require_auth` successfully has been tested independently before this deployment, not assumed.
- [ ] Accepted tokens, fee rates, and oracle configuration for mainnet are written down and reviewed **before** running any `set_*` command, not decided live during the deployment.
- [ ] Someone is designated to record the deployment artifact record (below) as each command completes, in real time, not reconstructed afterward.
- [ ] A rollback/pause plan (see [Rollback](#rollback) below) is understood by whoever is running the deployment, before they start.

### Deploy

```bash
NETWORK=mainnet make deploy-shade ADMIN=<confirmed-admin-address>
```

Read the printed contract ID back and cross-check it against `.stellar/shade_contract_id.txt` before proceeding — do not trust only the terminal scrollback.

### Initialize

```bash
NETWORK=mainnet make init-shade ADMIN=<confirmed-admin-address>
```

Immediately after, verify independently:

```bash
stellar contract invoke --id <shade-id> --network mainnet -- get_admin
```

Confirm the returned address is the exact one you intended, byte for byte, before running any further configuration command. `initialize` can only be called once (`AlreadyInitialized`, error code 2) — there is no "redo" on this step; a wrong admin here requires the full [two-step admin transfer](../security/admin-and-ownership.md#two-step-admin-transfer) to fix, which itself requires the wrong admin's cooperation or key if it was a typo'd-but-controllable address, and is unrecoverable if it was not.

### Post-deployment configuration and smoke test

Follow the same sequence as [Testnet](#testnet) above, against `--network mainnet`, with real accepted tokens and real oracle contract addresses. Run the smoke test with **small, disposable amounts** — the "payer" and "merchant" in a mainnet smoke test are moving real funds.

### Artifact recording

Record every mainnet deployment. This repository has no established external system for this (no deployment log directory, no dashboard) — until your team adopts one, use this template and store it wherever your team keeps operational records, flagging that as a decision the team should make explicit rather than leaving implicit:

```text
Release / commit:        <git tag or SHA>
Toolchain:                Rust <version>, Stellar CLI <version>, soroban-sdk <version>
Network:                  mainnet
Deployment timestamp:     <UTC ISO 8601>
Deployer identity:        <G... address>
Contract name:            shade
Contract ID:              <C... address>
WASM hash:                <sha256 hex, from `stellar contract install` output>
Deployment ledger:        <ledger sequence number>
Initialization ledger/tx: <ledger sequence / tx hash>
Configuration tx IDs:     add_accepted_tokens=<tx>, set_fee(<token>)=<tx>, set_token_oracle(<token>)=<tx>, ...
Smoke-test tx IDs:        register_merchant=<tx>, create_invoice=<tx>, pay_invoice=<tx>
Operator:                 <name/role>
Reviewer:                 <name/role>
Notes:                    <anything that deviated from this runbook and why>
```

## Rollback

What can and cannot be undone, by stage:

| Stage | Reversible? | How |
|---|---|---|
| Deploy failed before `initialize` | Yes | The contract instance exists at a contract ID but holds no state. Abandon that ID; redeploy fresh. Nothing to clean up on-chain — an unused, uninitialized contract instance is not a liability, just an unused address. |
| Factory deployment failed (`escrow_factory`/`ticketing_factory`/`crowdfund_factory`) | Yes | Same as above — these are independent of `shade`; a failed factory deploy has no effect on anything else. Redeploy. |
| `shade` deployed but `initialize` failed partway (should not partially apply — Soroban transactions are atomic) | Yes | If `initialize` genuinely never committed, `get_admin` will panic with `NotInitialized`. Re-run `initialize`. If it *did* commit with the wrong admin, this is not a "failed" state — see [Initialize](#initialize) above; you're now in a live-with-wrong-config scenario, not a rollback-eligible one. |
| Post-deployment configuration step failed or was wrong (wrong fee, wrong token, wrong oracle) | Yes, if caught before payments settle against it | Re-run the same `set_*` call with the corrected value. `set_fee` applies immediately; a wrong fee that already collected payments cannot retroactively refund the difference — see [Admin Runbook § Fees](admin-runbook.md#routine-operation-fees). |
| Protocol is live, merchants have registered, invoices exist, and configuration needs to change | Partially | Fees, accepted tokens, oracles, and roles can all be changed going forward via their respective `set_*`/`grant_role`/`revoke_role` calls (see [Admin Runbook](admin-runbook.md)). Past invoices, payments, and analytics already recorded are not retroactively affected by a later config change — this is by design, not a gap. |
| Irreversible on-chain state already created (invoices paid, subscriptions charged, tickets sold, funds distributed) | **No** | Soroban has no contract-deletion or state-rollback mechanism. A materially wrong deployment discovered after real payment activity is a [pause](admin-runbook.md#pause-procedure) situation, not a rollback — see the [Admin Runbook's incident response section](admin-runbook.md#incident-response) and the [Upgrade Runbook](upgrade-runbook.md) if the fix requires new contract code. |

**There is no "undeploy" operation.** A Soroban contract instance, once created, exists at its contract ID permanently; you cannot delete it or reclaim the address. Treat every deployment — especially mainnet — as something you're choosing to make permanent at the moment you run `stellar contract deploy`, not something you can quietly walk back if the smoke test surfaces a problem five minutes later.

## Related pages

- [Upgradeability](../architecture/upgradeability.md) — what changing the deployed WASM later can and cannot do to this deployment's state.
- [Upgrade Runbook](upgrade-runbook.md) — the procedure for changing a deployment after this one, once it's live.
- [Admin Runbook](admin-runbook.md) — routine configuration changes and incident response for a deployment that's already up.
- [Contract bindings guide](../guides/contract-bindings.md) — generating a client against the contract ID this runbook produces.
- [Pausable emergency-stop mechanism](../security/pausable.md) — what you'd reach for if a deployment goes live with a serious problem.
- [Admin ownership](../security/admin-and-ownership.md) — the admin address this runbook's `initialize` step sets.

← [Back to Operations](README.md)
