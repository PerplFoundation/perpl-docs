# Install

`perpl-sdk` is the Rust crate for building and maintaining an in-memory cache of Perpl exchange state and for constructing and posting orders. This page covers getting the crate into your Cargo project: the toolchain you need, how to add the dependency, which Cargo features to turn on or off, the extra prerequisite for running the local test environment, and how to build the API documentation.

SDK stands for software development kit. Perpl is a decentralized exchange (DEX) deployed on [Monad](https://docs.monad.xyz/); the SDK talks to the Exchange contract over a standard Ethereum-compatible RPC (remote procedure call) endpoint.

## Prerequisites

| Requirement       | Version       | Why                                                                                                                                      |
| ----------------- | ------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Rust toolchain    | **>= 1.85.0** | The crate uses Rust **edition 2024**, which requires this compiler.                                                                      |
| `anvil` (Foundry) | any recent    | Only for **local testing** — the `testing` feature spins up a local node via `anvil`. Not needed to build or run against a live network. |

Check your Rust version:

```bash
rustc --version   # must report 1.85.0 or newer
```

If it is older, update with [`rustup`](https://rustup.rs/):

```bash
rustup update stable
```

Install `anvil` (part of [Foundry](https://getfoundry.sh/)) only if you plan to run the SDK's local test environment:

```bash
curl -L https://foundry.paradigm.sh | bash
foundryup
anvil --version   # confirm it is on your PATH
```

{% hint style="info" %}
`anvil` is a **test-only** prerequisite. A production client that connects to mainnet or testnet does not need it.
{% endhint %}

## Add the crate to your project

The SDK lives in the `dex-sdk` workspace at `crates/sdk` (package name `perpl-sdk`, current version **0.2.0**). Add it to your project's `Cargo.toml` as a **path dependency** pointing at your local clone of `dex-sdk`:

```toml
[dependencies]
perpl-sdk = { path = "../dex-sdk/crates/sdk" }
```

Adjust the relative path to match where you cloned [`dex-sdk`](https://github.com/PerplFoundation) next to your own crate. This is the form used throughout the [`dex-sdk-examples`](https://github.com/PerplFoundation/dex-sdk-examples) repo.

> **TODO(author):** The source files document only the **path** dependency form. If the crate is (or will be) distributed via a Git URL or crates.io, document the `git = "…"` / version form here.

In Rust source, the crate is imported with an underscore:

```rust
use perpl_sdk::Chain;
```

### The SDK's own dependency stack

You do not add these yourself — Cargo pulls them in transitively — but it is useful to know the crate is built on:

| Dependency        | Version | Role                                                |
| ----------------- | ------- | --------------------------------------------------- |
| `alloy`           | 2.0.4   | Ethereum provider, contract bindings, RPC types     |
| `alloy-sol-types` | 1.5.7   | Solidity ABI types for the generated bindings       |
| `fastnum`         | 0.7.4   | Fixed-point decimal arithmetic for prices and sizes |
| `thiserror`       | 2       | Error types (`DexError`, `ProviderError`, …)        |

## Cargo features

The crate exposes the following features. `display` and `testing` are **on by default**.

| Feature      | Default | Enables                                                                                                                                                                                       | Pulls in              |
| ------------ | ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| `display`    | ✅ yes   | `Display` implementations for pretty-printing state (order books, accounts, …)                                                                                                                | `tabled`, `colored`   |
| `testing`    | ✅ yes   | The local test environment (spins up a node)                                                                                                                                                  | `alloy/node-bindings` |
| `test-utils` | ❌ no    | Test builders (`Perpetual::for_test`, `with_bid`, `with_ask`, …) for downstream crates to use from their **dev-dependencies** without exposing internal mutation methods in production builds | —                     |

The default set is convenient for exploration and examples. For a lean production build you will typically drop `testing` (which drags in `alloy/node-bindings` and requires `anvil`) and keep only what you use.

**Keep the defaults** (nothing to write — `perpl-sdk = { path = "…" }` already enables `display` + `testing`).

**Disable defaults and select explicitly** — for example, keep pretty-printing but drop the local-test machinery:

```toml
[dependencies]
perpl-sdk = { path = "../dex-sdk/crates/sdk", default-features = false, features = ["display"] }
```

**Opt into the test builders** from your own test code — add `test-utils` under `dev-dependencies` so it never reaches your release binary:

```toml
[dev-dependencies]
perpl-sdk = { path = "../dex-sdk/crates/sdk", features = ["test-utils"] }
```

{% hint style="info" %}
`test-utils` is deliberately separate from `testing`. `testing` enables the `anvil`-backed local environment; `test-utils` only exposes in-memory state builders. Enabling `test-utils` does **not** require `anvil`.
{% endhint %}

## Build the API documentation

Generate and open the crate's full API reference locally:

```bash
cargo doc -p perpl-sdk --no-deps --open
```

* `-p perpl-sdk` — document just this crate.
* `--no-deps` — skip documenting every transitive dependency (faster, focused output).
* `--open` — open the generated HTML in your browser when it finishes.

This is the authoritative reference for every type mentioned across the SDK docs (`Chain`, `SnapshotBuilder`, `Exchange`, `OrderRequest`, the `stream` module, and so on).

## Verify your setup

A minimal build check that the crate resolves and compiles in your project:

```bash
cargo build
```

Then confirm the network config constructors are reachable — this compiles and prints the mainnet chain ID and Exchange address without touching the network:

```rust
use perpl_sdk::Chain;

fn main() {
    let chain = Chain::mainnet();
    println!("chain_id = {}", chain.chain_id());   // 143
    println!("exchange = {}", chain.exchange());    // 0x34B6552d57a35a1D042CcAe1951BD1C370112a6F
}
```

For the full per-network reference (RPC URLs, contract and collateral-token addresses, market IDs), see [Networks & Configuration](file:///2362779/getting-started/networks.md).

## Next steps

* **Read exchange state.** Build a snapshot and keep it current from the event stream — see the [SDK Quickstart](/broken/pages/fa98ed8f00683553133d4226d3f9b39bd0235fc3).
* **Use the CLI.** The workspace also ships `perpl-cli`, a command-line tool (CLI) for reading and tracing exchange state and events — see the [CLI reference](/broken/pages/5268aa41b0d895ff11b9b51dcbc6fc179e54aadb).
* **Browse examples.** Working market-making and utility bots live in [`dex-sdk-examples`](https://github.com/PerplFoundation/dex-sdk-examples).
