# Generating and using contract bindings

How to generate a typed TypeScript client for a Shade Protocol contract, how the workspace's Soroban types surface in that client, and how to keep a generated client in sync with the WASM it targets. This page is for client/backend developers integrating with the contracts, not for contract authors — see [Building the Contracts](../getting-started/building.md) for the Rust side.

> **Note:** As of this writing, `cargo build -p shade --release` and `cargo build -p crowdfund --release` fail on `main` (duplicate `#[contracttype]` definitions and missing enum variants from unmerged feature branches — see [Deployment Runbook § Known-broken baseline](../operations/deployment.md#known-broken-baseline-verify-before-you-start)). Everything below was verified by actually building and generating bindings for the `account` contract, which compiles cleanly on `main`. The commands are identical for `shade`; you cannot run them against `shade` until that crate builds again. Confirm `cargo build -p shade --release` succeeds before generating `shade` bindings for anything beyond local experimentation.

## Prerequisites

- Toolchain from [Prerequisites](../getting-started/prerequisites.md): Rust ≥ 1.84.0, `wasm32-unknown-unknown` target, Stellar CLI 23.x.
- Node.js and npm, to build the generated TypeScript package.
- A WASM build of the contract you want bindings for (see [Building the Contracts](../getting-started/building.md)), or a contract already deployed to a network.

> **Note:** This page's commands were run and verified against Stellar CLI 25.2.0, one major version ahead of the 23.x this repository's `Cargo.lock` and CI target. The `bindings typescript` subcommand and its flags are unchanged between 23.x and 25.x. Two other subcommands used elsewhere in this doc set have been renamed since 23.x: `stellar contract install` → `stellar contract upload`, and `stellar contract optimize` → `stellar contract build --optimize` (the old names still work as deprecated aliases on 25.x). Check `stellar contract bindings typescript --help` against whatever CLI major version you have installed before relying on this page verbatim.

## Generating bindings from a local WASM build

Build and optimize the contract first (see [Building the Contracts](../getting-started/building.md#optimizing-the-wasm-for-deployment)):

```bash
cargo build --target wasm32-unknown-unknown --release -p shade
stellar contract optimize --wasm target/wasm32-unknown-unknown/release/shade.wasm
```

Then generate the TypeScript package from the optimized WASM:

```bash
stellar contract bindings typescript \
  --wasm target/wasm32-unknown-unknown/release/shade.optimized.wasm \
  --output-dir ./bindings/shade \
  --overwrite
```

- `--wasm` — path to the built WASM. Use the `.optimized.wasm` file, not the plain `.wasm` file, so the bindings match what you actually deploy.
- `--output-dir` — where the generated npm package is written. This example uses `./bindings/shade`, a convention, not a repository requirement — no `bindings/` directory exists in this repo yet, so pick a location and commit to it for your integration.
- `--overwrite` — required to regenerate into a directory that already has a previous version in it.

Build the generated package:

```bash
cd ./bindings/shade
npm install && npm run build
```

## Generating bindings from a deployed contract

If the contract is already deployed and you don't have (or don't trust) a local WASM build, generate straight from the network:

```bash
stellar contract bindings typescript \
  --contract-id CBQHNAXSI55GX2GN6D67GK7BHVPSLJUGZQEU7WJ5LKR5PNUCGLIMAO4E \
  --network testnet \
  --output-dir ./bindings/shade
```

`--contract-id` replaces `--wasm`; `--network` (or `--rpc-url` + `--network-passphrase`) tells the CLI which RPC endpoint to read the contract's stored spec from. `--contract-id` above is a placeholder — substitute your deployed contract's real ID. This path pulls the *currently installed* WASM's embedded spec (every Soroban contract embeds its own `ShadeTrait`/interface spec at compile time), so it always matches what's live on that network, but it tells you nothing about whether that's the interface version you expect — see [Detecting a stale or mismatched client](#detecting-a-stale-or-mismatched-client).

## Output layout

`stellar contract bindings typescript` writes an npm package:

```text
bindings/shade/
├── src/
│   └── index.ts        # generated client: types, Client class, ContractSpec
├── package.json         # depends on @stellar/stellar-sdk
├── tsconfig.json
├── README.md
└── .gitignore
```

`src/index.ts` contains everything: every `#[contracttype]` struct and enum from the contract as a TypeScript type, a `Client` class with one method per contract function, and the contract's XDR spec embedded as base64 strings (passed to `ContractSpec` in the constructor) — this is how the client validates and encodes/decodes calls without a second round-trip to fetch the spec. There is nothing else to wire up; importing `Client` from `src/index.ts` (or the built `dist/index.js`) is the complete generated API surface.

## Using the generated client in an application

```typescript
import { Client } from "./bindings/shade/src/index.js";

const client = new Client({
  contractId: "CBQHNAXSI55GX2GN6D67GK7BHVPSLJUGZQEU7WJ5LKR5PNUCGLIMAO4E", // placeholder — your deployed shade contract ID
  networkPassphrase: "Test SDF Network ; September 2015",                // testnet passphrase; use the mainnet passphrase in production
  rpcUrl: "https://soroban-testnet.stellar.org",
});

// Read: get_invoice(invoice_id: u64) -> Invoice
const { result: invoice } = await client.get_invoice({ invoice_id: 1n });
console.log(invoice.status, invoice.amount, invoice.token);

// Write: pay_invoice(payer: Address, invoice_id: u64)
const tx = await client.pay_invoice({
  payer: "GDPAYERPLACEHOLDER0000000000000000000000000000000000000",
  invoice_id: 1n,
});

// Sign and submit. `signTransaction` is supplied by your wallet integration
// (e.g. a browser wallet extension's signTransaction, or a server-side
// Keypair-based signer) — the generated client does not include one.
const sent = await tx.signAndSend({
  signTransaction: async (xdr) => {
    // Replace with your actual signing mechanism. Never hardcode a secret
    // key in application code — load it from a secret manager or wallet.
    throw new Error("wire up your signer here");
  },
});

console.log(sent.result);
```

Every generated write method (`add_token`, `pay_invoice`, `register_merchant`, etc.) follows this same three-step shape: call the method to get an `AssembledTransaction` (this simulates the call against the RPC and reports the would-be result before you sign anything), call `.signAndSend({ signTransaction })` to sign and submit it, then read `.result`. Read-only methods (`get_invoice`, `is_paused`, …) resolve immediately with `{ result }` from simulation and never need signing.

> **Note:** The generated client has no built-in wallet or signer. `signAndSend` takes a `signTransaction` callback you provide — from a browser wallet's signing API, a `Keypair.sign` call against a secret key held server-side in a secret manager, or a hardware-signing integration. Never inline a secret key in application source, and never commit one to a repository.

## Type mapping

Types below are exactly as `stellar contract bindings typescript` emits them — verified by generating and reading the output for the `account` contract, whose generated struct/enum shapes are structurally identical to `shade`'s (same SDK, same code generator).

### `i128` (and `i64`, `u128`, `u256`, `i256`)

Soroban's 128-bit integers do not fit in a JavaScript `number` (safe up to 2^53). The generated client imports a branded `i128` type from `@stellar/stellar-sdk/contract`, backed by `bigint` at runtime:

```typescript
export interface TokenBalance {
  balance: i128;
  token: string;
}
```

```typescript
const { result } = await client.get_balance({ token: usdcAddress });
// result is a bigint (branded as i128) — use bigint literals and arithmetic:
const doubled = result * 2n;
```

**Common mistake:** passing a plain JavaScript `number` into an `i128` field. Small values (e.g. `1`) coerce fine in practice, but a literal like `1_0000000` (1 token at 7 decimals) should be written `1_0000000n` — use `n`-suffixed `bigint` literals for every amount field, not just large ones, so you don't have a code path that silently breaks the first time a value exceeds `Number.MAX_SAFE_INTEGER`. Never use `parseInt`/`parseFloat` on a value bound for an `i128` parameter.

### `Address`

Every `Address` field or parameter (merchant, token, payer, admin, …) is represented as a plain TypeScript `string` in the generated interface — the Stellar `G...` (account) or `C...` (contract) strkey-encoded address:

```typescript
initialize: ({merchant, manager, merchant_id}: {merchant: string, manager: string, merchant_id: u64}, ...)
```

**Do not treat every `string` field as interchangeable.** `Address` strings, `BytesN<32>` hex/base64 strings, and human-readable `String` fields (descriptions, webhook URLs) are all typed `string` in the generated output — the type system will not stop you from passing a webhook URL where an address is expected. Read the field name and the source contract's `types.rs` (linked from [Data types reference](../reference/data-types.md)) to know which kind of string a given field actually is. An `Address` string is not the same thing as an `AccountId`/`MuxedAccount` from `@stellar/stellar-sdk`'s lower-level XDR types — the generated client's `string` is always the strkey form, and you generally never need to touch the XDR representation directly when using this client.

### `BytesN<32>` (and other fixed-size byte arrays)

`BytesN<32>` — used for WASM hashes, signed-invoice nonces, and bridge transaction IDs — is not a distinct branded type in the generated output; it appears wherever the Rust source uses it, typically alongside other bytes handling in `@stellar/stellar-sdk`. When constructing a call that takes a `BytesN<32>` (for example, `upgrade`'s `new_wasm_hash`, or `create_invoice_signed`'s `nonce`), pass a `Buffer` or hex string of exactly 32 bytes — a shorter or longer value fails at the XDR encoding step, not at the TypeScript type-check step, since the generated bindings do not enforce byte length in the type system. Generate a nonce with a cryptographically secure random source (e.g. Node's `crypto.randomBytes(32)`), never a predictable value — see [Signed invoices](../security/signatures.md) for why the nonce must be unpredictable and single-use.

### Enums: key enums vs. integer enums

This workspace has two different enum shapes on the Rust side (see [Upgradeability § Key enums vs. integer enums](../architecture/upgradeability.md#key-enums-the-variant-name-is-the-identity)), and they generate two different TypeScript shapes:

**Unit/tuple-variant enums** (like `DataKey`) become a discriminated union with a `tag` and `values` tuple:

```typescript
export type DataKey =
  | {tag: "Manager", values: void}
  | {tag: "WithdrawalRequest", values: readonly [u64]}
  | {tag: "WithdrawalAnalytics", values: readonly [string]};
```

You will not construct these directly as a client — `DataKey`-shaped types are Shade's internal storage keys, not part of `ShadeTrait`'s public parameters — but the same `{tag, values}` shape applies to any `#[contracttype]` enum with variant arguments that *is* part of the public interface, such as `FiatPricingData` (`None` / `Some(FiatPricing)`) or `PaymentRoute` (`Direct` / `Swap(SwapRoute)`):

```typescript
// Constructing a PaymentPayload with a direct route:
const payload = {
  input_token: tokenAddress,
  settlement_token: tokenAddress,
  route: { tag: "Direct", values: undefined },
  max_slippage_bps: undefined,
};
```

**`#[repr(u32)]` integer enums** (like `InvoiceStatus`, `EscrowStatus`, `Role`) become a plain TypeScript `enum` with numeric values matching the Rust discriminants exactly:

```typescript
export enum WithdrawalStatus {
  Pending = 0,
  Approved = 1,
  Executed = 2,
}
```

Compare an `Invoice.status` field against `InvoiceStatus.Paid`, not against the string `"Paid"` or the bare number `1` — using the generated enum keeps your code correct if a future contract version adds new variants at higher numbers (see [Upgradeability § Integer enums](../architecture/upgradeability.md#integer-enums-the-number-is-the-identity)).

### Custom structs

A representative example, `WithdrawalRequest` (from the `account` contract; `shade`'s `Invoice`, `Merchant`, and other stored structs follow the identical pattern — field names carried over verbatim, `i128`/`Address`/enum fields converted per the rules above):

```typescript
export interface WithdrawalRequest {
  amount: i128;
  approvals: Array<string>;
  id: u64;
  recipient: string;
  status: WithdrawalStatus;
  token: string;
}
```

Field order in the generated interface does not necessarily match declaration order in the Rust source (Soroban structs serialize as maps keyed by field name, not by position — see [Upgradeability § Structs](../architecture/upgradeability.md#structs-the-field-name-set-is-the-identity)), so match fields by name, not position, when reading a generated struct.

### Errors

Generated contract errors surface as a plain object keyed by numeric code:

```typescript
export const ContractError = {
  1: {message:"AlreadyInitialized"},
  2: {message:"NotInitialized"},
  // ...
};
```

An `AssembledTransaction`'s simulation failure surfaces the raw numeric code; look it up against the relevant contract's table in the [Errors reference](../reference/data-types.md) (or the contract's own `errors.rs`) to get the variant name. `shade` partitions its errors across nine numeric ranges in one contract (see the errors reference) — the same numeric code means different things on `shade` than on `account` or `escrow`, so always pair a code with the contract ID that raised it.

## Detecting a stale or mismatched client

Nothing in the generated client checks whether it matches the contract it's pointed at — a `Client` built from an old WASM's bindings will still compile and run against a newer (or different) deployed contract; it can silently pass the wrong argument order to a renamed parameter, or fail to decode a return type that's changed shape.

To detect a mismatch:

- **Wrong network / wrong contract ID entirely:** the simplest and most common failure. Before relying on a `Client` in an environment, call a cheap read function you know the expected answer to (`get_admin`, `is_paused`) and sanity-check the result against what you expect for that environment, not just that the call succeeded.
- **Interface mismatch (function renamed, removed, or resigned):** a call to a function that no longer exists, or whose parameters changed, fails at the RPC simulation step with an error naming the missing/mismatched function — the TypeScript compiler cannot catch this for you, because the generated types were frozen at generation time and know nothing about what's actually deployed now.
- **Stale generated types for a struct/enum that changed shape:** this is the dangerous case, because it does not fail loudly. If a stored struct gained, lost, or renamed a field in a way that's backward-compatible at the storage layer (see [Upgradeability § Migration patterns](../architecture/upgradeability.md#migration-patterns)) but the client wasn't regenerated, decoded values may be missing fields your code expects, or carry `undefined` where you assumed a value. Regenerate bindings after every deployed interface change — do not rely on the old client "still basically working."
- **Authoritative check available in this repo:** `stellar contract bindings typescript --contract-id <id> --network <net>` always pulls the spec embedded in whatever WASM is *currently* live at that contract ID. Diffing a fresh generation's `src/index.ts` against your committed one is a reliable way to detect drift — a byte-identical regeneration means the deployed interface hasn't changed since you last generated; any diff means it has.

There is no on-chain semantic version to check instead — this repository does not use one (see [Upgradeability § Versioning](../architecture/upgradeability.md#versioning-and-identifying-a-deployed-build)). The WASM hash is the only real identity; if you need certainty rather than a heuristic, record the WASM hash you generated bindings from (`sha256sum` the `.optimized.wasm`, or read it back from `stellar contract install`'s output) alongside the generated package, and compare it against the hash the [deployment runbook](../operations/deployment.md) recorded for that environment.

## Regenerating bindings after a contract change

1. Land the interface change in `contracts/<crate>/src/` (and update `docs/reference/shade-interface.md` / `docs/reference/data-types.md` in the same PR — see [CONTRIBUTING.md](../../CONTRIBUTING.md#keeping-reference-docs-in-sync)).
2. Rebuild: `cargo build --target wasm32-unknown-unknown --release -p <crate>`.
3. Optimize: `stellar contract optimize --wasm target/wasm32-unknown-unknown/release/<crate>.wasm`.
4. Regenerate: `stellar contract bindings typescript --wasm target/.../<crate>.optimized.wasm --output-dir ./bindings/<crate> --overwrite`.
5. Review the diff in `bindings/<crate>/src/index.ts` — a changed function signature, added/removed struct field, or renamed type is your signal that dependent application code needs updating too.
6. Rebuild and re-run your application's test suite against the new client (`npm run build` inside the bindings package, then whatever tests exercise it).
7. Deploy the new WASM (see [Deployment Runbook](../operations/deployment.md) or [Upgrade Runbook](../operations/upgrade-runbook.md)).
8. Verify the deployed contract's live spec matches by regenerating once more with `--contract-id` against the network and diffing — this confirms the deployed bytecode is the interface you just built, not a stale upload.
9. Publish/update the application's dependency on the `bindings/<crate>` package (however your project manages internal packages — this repository does not currently publish bindings to a registry).

## Network configuration

The generated `Client` takes `networkPassphrase` and `rpcUrl` directly; there is no separate network-config file to maintain in the generated package itself. Use the Stellar CLI's own named networks (`stellar network ls` shows `local`, `futurenet`, `testnet`, `mainnet` by default) when generating bindings with `--network <name>` instead of `--rpc-url`/`--network-passphrase`, and match the same network name across bindings generation, deployment (see [Deployment Runbook](../operations/deployment.md)), and the values you hardcode into your application client — a `Client` constructed with mainnet's passphrase against a testnet RPC URL (or vice versa) fails every call with a network mismatch, not a helpful error naming the mismatch itself.

## Related pages

- [Data types reference](../reference/data-types.md) — every type referenced above, with exact source line citations.
- [ShadeTrait function reference](../reference/shade-interface.md) — the full function list the generated `Client` wraps.
- [Upgradeability](../architecture/upgradeability.md) — storage-compatibility rules that determine whether an old client can safely talk to a newly upgraded contract.
- [Signed invoices](../security/signatures.md) — nonce and signature construction for `create_invoice_signed`.
- [Deployment Runbook](../operations/deployment.md) — deploying the WASM you generate bindings from.

← [Back to guides](README.md)
