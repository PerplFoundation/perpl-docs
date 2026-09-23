# Concepts

The Perpl Rust SDK (`perpl-sdk`) is a convenient, **in-memory cache of on-chain exchange state**. Rather than issuing an ad-hoc `eth_call` every time you need a price or an order book, you take one snapshot of the exchange at a block, then apply a continuous stream of on-chain events to keep that snapshot current. When you want to trade, you build strongly typed order requests and submit them through the exchange contract.

This page explains the model: the `Chain` config, the module map, the snapshot-then-stream workflow, and how an `OrderRequest` becomes an on-chain call. Throughout, "SDK" means `perpl-sdk`, "DEX" means decentralized exchange, and "RPC" means the JSON remote-procedure-call endpoint you point the SDK at.

{% hint style="info" %}
The SDK targets **Rust edition 2024** and a **minimum Rust version of 1.85.0**. The workspace version is **0.2.0**. Local testing additionally requires the `anvil` binary from Foundry.
{% endhint %}

***

## The mental model

Three moving parts, in order:

1. **`Chain`** — a small value object describing which deployment you are talking to (chain id, exchange address, collateral token, deploy block, and the list of perpetual markets).
2. **`state::SnapshotBuilder` → `state::Exchange`** — builds the initial in-memory snapshot of exchange state at a chosen block.
3. **`stream::raw`** — a per-block stream of raw contract events. You feed each block into `Exchange::apply_events` to keep the snapshot up to date.

From the crate's own overview:

> Use `state::SnapshotBuilder` to capture the initial state snapshot, then `stream::raw` to catch up with recent state and keep the snapshot up to date. Use `types::OrderRequest` to prepare order requests and send them with `abi::dex::Exchange::ExchangeInstance::execOrders`.

***

## Adding the dependency

The SDK is consumed **by path** in the reference examples (there is no published crates.io install instruction in the sources):

```toml
# Cargo.toml
[dependencies]
perpl-sdk = { path = "../dex-sdk/crates/sdk" }
```

Build the API docs locally with:

```bash
cargo doc -p perpl-sdk --no-deps --open
```

### Cargo features

Both features are enabled by default:

| Feature   | Default | Description                                                                                                         |
| --------- | ------- | ------------------------------------------------------------------------------------------------------------------- |
| `display` | yes     | Enables `std::fmt::Display` implementations for the state types.                                                    |
| `testing` | yes     | Enables the `testing` module — a local testing environment with a collateral token and exchange contracts deployed. |

{% hint style="info" %}
The crate documents three current limitations: funding-events processing is a follow-up (TODO); the event stream relies on **log polling** (future versions may use WebSocket subscriptions or Monad execution events); and test coverage is described as below reasonable. Design around the log-polling model for now.
{% endhint %}

***

## `Chain`: describing the deployment

`Chain` is a `Clone + Debug` struct with **private fields** and read-only getters. It carries everything the SDK needs to locate and interpret a deployment:

```rust
pub struct Chain {
    chain_id: u64,
    collateral_token: Address,
    deployed_at_block: u64,
    exchange: Address,
    perpetuals: Vec<PerpetualId>, // PerpetualId = u32
}
```

Getters: `chain_id()`, `collateral_token()`, `deployed_at_block()`, `exchange()`, and `perpetuals() -> &[PerpetualId]`.

### Built-in constructors

```rust
use perpl_sdk::Chain;

let mainnet = Chain::mainnet();
let testnet = Chain::testnet();
```

|                     | `Chain::mainnet()`                           | `Chain::testnet()`                           |
| ------------------- | -------------------------------------------- | -------------------------------------------- |
| `chain_id`          | `143`                                        | `10143`                                      |
| `collateral_token`  | `0x00000000eFE302BEAA2b3e6e1b18d08D69a9012a` | `0xa9012a055bd4e0eDfF8Ce09f960291C09D5322dC` |
| `deployed_at_block` | `54773010`                                   | `62953`                                      |
| `exchange`          | `0x34B6552d57a35a1D042CcAe1951BD1C370112a6F` | `0x1964C32f0bE608E7D29302AFF5E61268E72080cc` |
| `perpetuals`        | `[1, 10, 20, 31, 40, 50]`                    | `[16, 32, 48, 64, 256]`                      |

{% hint style="info" %}
On mainnet, SOL is perpetual **31** (SOL was relisted as perp 31), not 30. Always disambiguate the perpetual id from the market symbol.
{% endhint %}

### Custom deployments

For a local `anvil` node or any other deployment, build the `Chain` yourself:

```rust
use perpl_sdk::Chain;
use alloy::primitives::Address;

let chain = Chain::custom(
    chain_id,          // u64
    collateral_token,  // Address
    deployed_at_block, // u64
    exchange,          // Address
    perpetuals,        // Vec<PerpetualId>
);
```

***

## Module map

The SDK's public surface is a handful of modules declared in `lib.rs`:

| Module                      | Purpose                                                                                                                                                                                                                                                                           |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `abi`                       | `alloy::sol!`-generated bindings from the JSON application binary interface (ABI): `dex::Exchange`, `erc1967_proxy::ERC1967Proxy`, `errors::Exchange` (errors ABI), and `testing::TestToken`. Also exposes `pub const DEX_REVISION` (from the build-time `env!("DEX_REVISION")`). |
| `error`                     | Error types: the top-level `DexError`; `ProviderError<R>` for RPC/execution failures (`Fatal`, `InvalidRequest`, `NullResp`, `OutOfGas`, `Reverted`, `Transport`, `Timeout`); and `RevertReason<R>` (`Known` / `Generic` / `Unknown`) for decoded reverts.                        |
| `num`                       | Fixed-point ↔ decimal conversion. A `num::n` converter maps on-chain integers (`U256` / `I256` / `u64` / `i64`) to and from `fastnum` decimals using **floor rounding**.                                                                                                          |
| `state`                     | In-memory exchange-state tracking. `SnapshotBuilder` captures the snapshot; `Exchange` is the root object giving access to `Account`, `Perpetual`, `Position`, `Order`, the L3 (level-3, per-order) `OrderBook`, and derived market data.                                         |
| `stream`                    | Continuous per-block event streams: `stream::raw` (raw contract events) and `stream::trade` (normalized `Trade`s aggregated from the raw stream).                                                                                                                                 |
| `types`                     | Public data types and aliases used across the SDK (see next table).                                                                                                                                                                                                               |
| `testing` _(feature-gated)_ | Local testing environment with a collateral token and exchange contracts deployed.                                                                                                                                                                                                |

### Core type aliases (`types`)

| Type                                                       | Definition   | Notes                                                                                                  |
| ---------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------ |
| `PerpetualId`                                              | `u32`        | Perpetual market id.                                                                                   |
| `AccountId`                                                | `u32`        | Account id.                                                                                            |
| `OrderId`                                                  | `NonZeroU16` | `0` is the `NULL_ORDER_ID` sentinel, so a live order id is always non-zero.                            |
| `RequestId`                                                | `u64`        | Becomes the on-chain `client_order_id` once an order is placed.                                        |
| `AccountAddressOrID`                                       | —            | Identify an account either by address or by id.                                                        |
| `StateInstant`                                             | —            | A `(block_number, block_timestamp)` point in time.                                                     |
| `OrderSide` / `OrderType` / `RequestType` / `OrderRequest` | —            | Order-construction types (see [Building and sending orders](concepts.md#building-and-sending-orders)). |

### The `num` converter

Prices, sizes, leverage, and collateral are stored on-chain as scaled integers. The `num::n` converter translates between those integers and human-readable `fastnum` decimals, rounding **down** (floor):

```rust
// A converter has type `num::n` with private fields, so you obtain one from an
// accessor rather than constructing it — e.g. `perp.price_converter()` for a
// perpetual's prices, or `exchange.collateral_converter()` for collateral.
let conv = exchange.collateral_converter(); // num::n for the 6-decimal collateral token
// from_unsigned / from_signed / from_u64 / from_i64  -> decimal
// to_unsigned / to_signed                            -> on-chain integer
// scale(), decimals()                                -> inspect the converter
```

You rarely call this directly for orders — `OrderRequest::prepare` looks up the right per-perp converters for you (see below).

***

## The snapshot-then-stream workflow

### Step 1 — build the snapshot with `SnapshotBuilder`

`SnapshotBuilder::new(chain: &Chain, provider)` starts a chainable builder. The provider is any `alloy` provider that is `Provider + Clone`. Defaults: block = latest, perpetuals = all of `chain.perpetuals()`, no accounts, all-positions off, and batch sizes of **1000**.

```rust
use perpl_sdk::{Chain, state::SnapshotBuilder};

let chain = Chain::testnet();

let exchange = SnapshotBuilder::new(&chain, provider.clone())
    .with_perpetuals(vec![16])   // only the perps you care about
    // .at_block(block_id)       // pin to a specific block (optional)
    // .with_accounts(vec![...]) // fetch these accounts + their positions
    // .with_all_positions()     // OR fetch all positions (mutually exclusive)
    .build()
    .await?;                     // -> Result<Exchange, DexError>
```

Builder methods (each consumes and returns `self`):

| Method                                                               | Effect                                                                                                                                      |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `.at_block(BlockId)`                                                 | Pin the snapshot to a specific block. A block **tag** is normalized to a concrete block number first, so every fetch reads the same height. |
| `.with_perpetuals(Vec<PerpetualId>)`                                 | Restrict which perpetual markets to fetch.                                                                                                  |
| `.with_accounts(Vec<AccountAddressOrID>)`                            | Fetch these accounts' state and positions. Assumes the accounts exist. **Mutually exclusive** with `.with_all_positions()`.                 |
| `.with_all_positions()`                                              | Fetch all positions and their accounts (no per-account balance snapshot). **Mutually exclusive** with `.with_accounts()`.                   |
| `.with_orders_per_batch(usize)` / `.with_positions_per_batch(usize)` | Multicall batch sizes (default 1000 each).                                                                                                  |

What `.build()` does, in order:

1. **Normalize the block** — resolve any tag to a fixed block number, producing a `StateInstant { block_number, block_timestamp }`.
2. **Probe for V2 support** — call `getPerpetualInfoV2`; if the deployed contract predates the V2 getters it reverts on the unknown selector, and the builder falls back to the V0 getters (up-converting to the V2 shape and defaulting the missing V2 fields `fundingSumScalingExp` and `priceResiduePNSQ16` to `0`).
3. **Fetch global params** — exchange info, funding interval, minimum post / settle / recycle-fee amounts, halt flag, and account count.
4. **Fetch per-perp state and orders** — per-perp info, maker fee, taker fee, and margin fractions; active orders are read by walking the `getOrderIdIndex` bitmap and issuing batched `getOrder` multicalls, preserving first-in-first-out (FIFO) order.
5. **Fetch positions** according to the account / all-positions selection.

{% hint style="info" %}
The default batch size of 1000 is chosen against Monad's cost of 8100 gas per storage-slot access and the 30M gas limit on `eth_call`, with buffer. Lower it for very heavy perps if an `eth_call` runs out of gas.
{% endhint %}

### Step 2 — stream raw events with `stream::raw`

`stream::raw` produces a **strictly continuous, per-block** sequence of raw contract events by polling `get_logs` at the provider's poll interval, starting from `from.block_number()`.

```rust
use perpl_sdk::stream;
use std::time::Duration;

// Start streaming right after the snapshot block.
let from = exchange.instant();

let raw_events = stream::raw(
    &chain,
    provider.clone(),
    from,
    |d: Duration| tokio::time::sleep(d), // sleep function used between polls
);
```

The stream yields `Result<RawBlockEvents, DexError>`, one item per block.

{% hint style="info" %}
On Monad the `latest` block tag corresponds to a _Proposed_ block, which is not yet final. `stream::raw` therefore also reads the `safe` block tag and only yields a block once `safe.number >= block_num`, erroring with "block is not available yet" otherwise. This keeps the cache consistent with finalized state.
{% endhint %}

{% hint style="warning" %}
`stream::raw` is **not cancellation-safe** — do not drop it mid-poll inside a `select!` arm without understanding the consequences. The crate recommends wrapping your provider with `alloy`'s `FallbackLayer` and/or `RetryBackoffLayer` for resilience.
{% endhint %}

### Step 3 — keep the cache current with `apply_events`

Feed every streamed block into `Exchange::apply_events`:

```rust
use futures::StreamExt;

let mut raw_events = std::pin::pin!(raw_events);

while let Some(block) = raw_events.next().await {
    let block = block?;
    match exchange.apply_events(&block)? {
        Some(state_events) => {
            // Events were applied; `state_events` are the normalized
            // state-level changes — react to them (reprint the book,
            // run strategy logic, etc.).
        }
        None => {
            // Block already applied — skip.
        }
    }
}
```

`apply_events` returns `Result<Option<_>, DexError>`:

* `Ok(Some(state_events))` — events applied; the returned batch is the normalized set of state-level changes.
* `Ok(None)` — this block was already applied; nothing to do.
* `Err(e)` — application error.

Once the cache is current you read live market data straight off the `Exchange`:

```rust
let instant = exchange.instant();                 // snapshot StateInstant
if let Some(perp) = exchange.perpetuals().get(&16) {
    let book   = perp.l3_book();      // level-3 order book
    let mark   = perp.mark_price();
    let last   = perp.last_price();
    let oracle = perp.oracle_price();
}
```

### Optional — the normalized trade stream `stream::trade`

When you care about executions rather than raw events, layer `stream::trade` on top of `stream::raw`. It listens for `MakerOrderFilled` and `TakerOrderFilled`, batches all maker fills belonging to one taker into a single unified `Trade`, and normalizes the fixed-point values to decimals:

```rust
let raw_stream = stream::raw(&chain, provider.clone(), from, sleep);
let mut trades = stream::trade(&chain, provider.clone(), raw_stream).await?;
```

Each `Trade` exposes: `taker_account_id`, `taker_side`, `total_size()`, `avg_price()`, `perpetual_id`, `taker_fee`, and `maker_fills: Vec<MakerFill>` — where each `MakerFill` carries `maker_account_id`, `maker_order_id`, `size`, `price`, and `fee`. Like `stream::raw`, `stream::trade` is **not cancellation-safe**.

***

## Building and sending orders

Order construction is fully typed. You describe intent with an `OrderRequest`, call `.prepare(&exchange)` to scale the decimal fields into the on-chain fixed-point `OrderDesc`, and submit the descriptors through the exchange contract.

### `RequestType`

`RequestType` is a `u8`-repr enum that selects the operation:

| Value | Variant                      | Meaning                                                                | Side |
| ----- | ---------------------------- | ---------------------------------------------------------------------- | ---- |
| 0     | `OpenLong`                   | Open / decrease / close / invert a long (needs sufficient collateral). | Bid  |
| 1     | `OpenShort`                  | Open / decrease / close / invert a short.                              | Ask  |
| 2     | `CloseLong`                  | Reduce-only: close all or part of an existing long.                    | Ask  |
| 3     | `CloseShort`                 | Reduce-only: close all or part of an existing short.                   | Bid  |
| 4     | `Cancel`                     | Cancel an existing order.                                              | —    |
| 5     | `IncreasePositionCollateral` | Add collateral to a position (reduce leverage / fix margin).           | —    |
| 6     | `Change`                     | Gas-efficient change of parameters of an existing order.               | —    |

Helpers: `RequestType::try_side() -> Option<OrderSide>` (Bid for `OpenLong` / `CloseShort`, Ask for `OpenShort` / `CloseLong`, `None` otherwise), plus `From<u8>` and `From<RequestType> for OrderType` conversions.

### `OrderRequest`

`OrderRequest::new` takes 15 arguments, in this order:

```rust
use perpl_sdk::types::{OrderRequest, RequestType};
use fastnum::{UD64, UD128};

let req = OrderRequest::new(
    request_id,               // RequestId (u64) -> on-chain client_order_id
    perp_id,                  // PerpetualId (u32)
    RequestType::OpenLong,    // RequestType
    None,                     // order_id: Option<OrderId>  (None => 0 on-chain)
    price,                    // UD64
    size,                     // UD64
    None,                     // expiry_block: Option<u64>
    true,                     // post_only
    false,                    // fill_or_kill
    false,                    // immediate_or_cancel
    None,                     // max_matches: Option<u32>
    UD64::ONE,                // leverage: UD64
    None,                     // last_exec_block: Option<u64>
    None,                     // amount: Option<UD128>
    0u16,                     // max_neg_pnl_collat_bps: u16
);
```

Notes on the fields:

* `request_id` becomes the on-chain `client_order_id` once the order is placed.
* `price` and `size` are `fastnum` unsigned 64-bit decimals (`UD64`); `amount` (used for collateral operations) is a `UD128`.
* The three execution flags are `post_only`, `fill_or_kill` (FOK — fill entirely or reject), and `immediate_or_cancel` (IoC — fill what crosses now, cancel the rest).
* `max_neg_pnl_collat_bps` is expressed in basis points (bps).

### Prepare → `OrderDesc`

`.prepare(&Exchange)` looks up the perpetual's price / size / leverage converters (and the collateral converter) and scales the decimal fields into the on-chain `OrderDesc`:

```rust
let desc = req.prepare(&exchange); // -> OrderDesc
```

The produced `OrderDesc` carries the scaled fields: `orderDescId`, `perpId`, `orderType` (u8), `orderId` (`0` when `None`), `pricePNS`, `lotLNS`, `expiryBlock`, `postOnly`, `fillOrKill`, `immediateOrCancel`, `maxMatches`, `leverageHdths`, `lastExecutionBlock`, `amountCNS` (collateral-scaled, present only when `amount` and a collateral converter are available), and `maxNegPnlCollatBPS`.

### Submit through the exchange contract

The canonical send path documented on the crate is the generated binding `abi::dex::Exchange::ExchangeInstance::execOrders`. Construct the instance against your exchange address and a wallet-bearing provider:

```rust
use perpl_sdk::abi::dex::Exchange::ExchangeInstance;

let instance = ExchangeInstance::new(chain.exchange(), provider);
```

Build the provider as an `alloy` `DynProvider` carrying your wallet:

```rust
use alloy::providers::ProviderBuilder;
use alloy::rpc::client::RpcClient;

let provider = ProviderBuilder::new()
    .wallet(wallet) // EthereumWallet
    .connect_client(RpcClient::new_http(rpc_url));
```

The reference example programs submit their prepared descriptors via `execOrders`, passing the `Vec<OrderDesc>` and a revert-on-fail flag, then await the receipt:

```rust
let descs: Vec<_> = requests.iter().map(|r| r.prepare(&exchange)).collect();

let receipt = instance
    .execOrders(descs, /* revertOnFail */ true)
    .send()
    .await?
    .get_receipt()
    .await?;
```

The `revertOnFail` boolean is an all-or-nothing flag: pass `true` when every descriptor must succeed together (the best-bid/offer example uses `true`), or `false` for best-effort batch submission (the spread and taker examples use `false`).

***

## Putting it together

A minimal read-only loop that snapshots one perpetual and then keeps its order book current:

```rust
use perpl_sdk::{Chain, state::SnapshotBuilder, stream};
use futures::StreamExt;
use std::time::Duration;

// 1. Describe the deployment.
let chain = Chain::testnet();

// 2. Snapshot the exchange (single perpetual here).
let mut exchange = SnapshotBuilder::new(&chain, provider.clone())
    .with_perpetuals(vec![16])
    .build()
    .await?;

// 3. Stream from the snapshot instant and keep the cache current.
let from = exchange.instant();
let raw = stream::raw(&chain, provider.clone(), from, |d: Duration| tokio::time::sleep(d));
let mut raw = std::pin::pin!(raw);

while let Some(block) = raw.next().await {
    if exchange.apply_events(&block?)?.is_some() {
        if let Some(perp) = exchange.perpetuals().get(&16) {
            println!("mark = {}", perp.mark_price());
        }
    }
}
```

From here, add a wallet-bearing `ExchangeInstance`, build `OrderRequest`s, `.prepare()` them, and submit — turning the read-only cache into a trading loop.

## Where to go next

* **`perpl-cli`** — the same snapshot/stream engine wrapped as a command-line tool for reading and tracing exchange state (`snapshot`, `trace`, `show account`, `show book`, `show trades`, `block <n>`, `tx <hash>`).
* **Example programs** — a market-making bot (best-bid/offer, spread, and taker strategies) and utilities (`print_book`, `print_trades`) demonstrate the full snapshot → stream → apply → trade lifecycle end to end.
