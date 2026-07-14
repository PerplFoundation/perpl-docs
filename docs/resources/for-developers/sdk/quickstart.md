# Quickstart

The Perpl Rust SDK (`perpl-sdk`) gives you a convenient in-memory cache of on-chain exchange state. The pattern is always the same: **capture an initial snapshot**, then **stream on-chain events to keep that snapshot current**, and read prices, order books, positions, and trades straight out of the cache. When you want to trade, you build an `OrderRequest`, `prepare` it against the cached state, and submit it to the Exchange contract.

This page walks through a runnable example end to end. Every type, method, and value shown below comes from the SDK source and the shipped examples.

{% hint style="info" %}
SDK (software development kit); RPC (remote procedure call); L2/L3 refer to level-2 (aggregated price levels) and level-3 (individual resting orders) views of an order book; bps means basis points.
{% endhint %}

## Prerequisites

| Requirement              | Value                                                                                      |
| ------------------------ | ------------------------------------------------------------------------------------------ |
| Rust toolchain           | **1.85.0** or newer (the SDK uses Rust edition 2024)                                       |
| Local testing (optional) | Foundry `anvil` — required only if you use the SDK's `testing` module against a local node |
| API docs                 | `cargo doc -p perpl-sdk --no-deps --open`                                                  |

The SDK crate is `perpl-sdk` version `0.2.0`, part of the `perpl-sdk` workspace (see [perpl.xyz](https://perpl.xyz/)).

## Add the SDK to your project

The SDK is consumed as a **path dependency** — point your `Cargo.toml` at the `crates/sdk` directory of a local `dex-sdk` checkout. This is exactly how the shipped examples depend on it:

```toml
[package]
name = "my-perpl-app"
edition = "2024"
version = "0.1.0"

[dependencies]
# Adjust the relative path to wherever you checked out dex-sdk.
perpl-sdk = { path = "../dex-sdk/crates/sdk" }

alloy    = { version = "2.0.4", features = ["full"] }
fastnum  = "0.7.4"           # fixed-point decimals used for prices/sizes
futures  = "0.3.31"          # Stream combinators (StreamExt)
tokio    = { version = "1.49", features = ["full"] }
```

{% hint style="info" %}
The SDK exposes two Cargo features, both **on by default**: `display` (adds `std::fmt::Display` implementations for the state types, so you can `println!("{}", price)`) and `testing` (adds a local testing environment with the collateral token and Exchange contracts deployed).
{% endhint %}

## The core workflow

```
Chain  ──►  SnapshotBuilder  ──► Exchange (in-memory cache)
                                    │
                stream::raw  ───────┤  feed each block into
              (per-block events)    ▼  exchange.apply_events(...)
                                    │
                       read  ◄──────┤  perpetuals().get(id) -> l3_book(),
                                    │  mark_price(), last_price(), ...
                                    │
                       trade ◄──────┘  stream::trade(...) -> normalized Trades
```

1. Choose a `Chain` (mainnet, testnet, or a custom deployment).
2. Build an `Exchange` snapshot with `SnapshotBuilder`.
3. Open a `stream::raw` event stream and call `exchange.apply_events(...)` for each block to keep the cache current.
4. Read state (`mark_price`, `l3_book`, positions, …) from the cache, or run `stream::trade` for a normalized trade feed.
5. To trade, build an `OrderRequest`, `prepare` it, and submit via the Exchange contract.

## 1. Pick a chain

The `Chain` type carries the full per-network configuration — chain ID, collateral token, Exchange contract address, the block the Exchange was deployed at, and the list of listed perpetual (market) IDs. Use a built-in constructor:

```rust
use perpl_sdk::Chain;

let chain = Chain::testnet();   // recommended for development
// let chain = Chain::mainnet(); // live trading
```

| Constructor        | Chain ID | Exchange contract                            | Collateral token                             | Deploy block | Perpetual IDs             |
| ------------------ | -------- | -------------------------------------------- | -------------------------------------------- | ------------ | ------------------------- |
| `Chain::mainnet()` | `143`    | `0x34B6552d57a35a1D042CcAe1951BD1C370112a6F` | `0x00000000eFE302BEAA2b3e6e1b18d08D69a9012a` | `54773010`   | `[1, 10, 20, 31, 40, 50]` |
| `Chain::testnet()` | `10143`  | `0x1964C32f0bE608E7D29302AFF5E61268E72080cc` | `0xa9012a055bd4e0eDfF8Ce09f960291C09D5322dC` | `62953`      | `[16, 32, 48, 64, 256]`   |

{% hint style="info" %}
On mainnet, SOL is perpetual ID `31` (not `30`). Perpetual IDs are network-specific — the same asset has a different ID on testnet. See [Networks & Configuration](../networks-and-configuration.md) for the full market tables.
{% endhint %}

Read individual fields with the getters:

```rust
let id: u64           = chain.chain_id();
let exchange          = chain.exchange();           // Address
let collateral        = chain.collateral_token();   // Address
let markets           = chain.perpetuals();         // &[PerpetualId]
let deploy_block: u64 = chain.deployed_at_block();
```

For a non-standard deployment (for example a local `anvil` node), build one explicitly:

```rust
use perpl_sdk::Chain;
use alloy::primitives::address;

let chain = Chain::custom(
    /* chain_id          */ 20143,
    /* collateral_token  */ address!("0x0000000000000000000000000000000000000000"),
    /* deployed_at_block */ 0,
    /* exchange          */ address!("0x0000000000000000000000000000000000000000"),
    /* perpetuals        */ vec![],
);
```

## 2. Build a state snapshot

`SnapshotBuilder` fetches the current on-chain state for the markets you ask for and returns an `Exchange` — the in-memory cache. You give it a `&Chain` and an `alloy` provider:

```rust
use perpl_sdk::state::SnapshotBuilder;

let exchange = SnapshotBuilder::new(&chain, provider.clone())
    .with_perpetuals(vec![market_id])   // which markets to load
    .build()
    .await?;
```

Builder options:

| Method                                                       | Purpose                                                                               |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------- |
| `.at_block(BlockId)`                                         | Snapshot state as of a specific block instead of the latest.                          |
| `.with_perpetuals(Vec<PerpetualId>)`                         | Restrict the snapshot to these markets.                                               |
| `.with_accounts(Vec<AccountId>)`                             | Load only these accounts' positions. Mutually exclusive with `.with_all_positions()`. |
| `.with_all_positions()`                                      | Load positions for every account. Mutually exclusive with `.with_accounts(...)`.      |
| `.with_orders_per_batch(n)` / `.with_positions_per_batch(n)` | Tune multicall batch sizes (default `1000`).                                          |

Under the hood, `build()` normalizes the target block to a concrete number, probes the Exchange for `getPerpetualInfoV2` support (falling back to the V0 layout, defaulting the V2-only `fundingSumScalingExp` / `priceResiduePNSQ16` fields to `0`), fetches global and per-perpetual parameters, fees, and margins, then reads resting orders (order-ID bitmap → batched `getOrder` multicalls, preserving first-in-first-out (FIFO) order) and positions. The default batch size of `1000` is tuned for Monad's per-slot gas cost and the 30M-gas `eth_call` limit.

The returned `Exchange` exposes the cached state:

```rust
let instant = exchange.instant();
println!(
    "snapshot at block {} (ts {})",
    instant.block_number(),
    instant.block_timestamp(),
);

if let Some(perp) = exchange.perpetuals().get(&market_id) {
    println!("mark  = {}", perp.mark_price());
    println!("last  = {}", perp.last_price());
    println!("book  = {} orders", perp.l3_book().total_orders());
}
```

## 3. Stream events to stay current

A snapshot is a point-in-time view. To keep it live, open a `stream::raw` event stream starting just after the snapshot block and feed every block into `exchange.apply_events(...)`:

```rust
use futures::StreamExt;
use perpl_sdk::{stream, types::StateInstant};

let mut events = Box::pin(stream::raw(
    &chain,
    provider,
    // Start one block after the snapshot. The second argument is an
    // intra-block offset; use 0 to start at the beginning of the block.
    StateInstant::new(instant.block_number() + 1, 0),
    tokio::time::sleep,
));

while let Some(result) = events.next().await {
    let block_events = result?;
    match exchange.apply_events(&block_events)? {
        Some(_state_events) => { /* cache updated for this block */ }
        None => { /* block already applied; nothing to do */ }
    }
}
```

`stream::raw` polls `get_logs` per block from your starting `StateInstant`. On Monad it gates on the `safe` block tag, because the `latest` tag returns _proposed_ (non-final) blocks.

`apply_events` returns:

| Return                   | Meaning                                                                 |
| ------------------------ | ----------------------------------------------------------------------- |
| `Ok(Some(state_events))` | The block's events were applied; `state_events` describes what changed. |
| `Ok(None)`               | The block was already applied — safe to skip.                           |
| `Err(_)`                 | The block could not be applied.                                         |

{% hint style="warning" %}
`stream::raw` and `stream::trade` are **not cancellation-safe** — they do not use `select!` internally, so do not drop them across an `await` in a `select!` branch and expect to resume mid-block. Wrap your provider in `alloy`'s `RetryBackoffLayer` (and, if you have multiple RPC endpoints, a `Fallback` layer) so transient RPC failures are retried rather than surfaced as stream errors. The example below shows the retry layer.
{% endhint %}

## Full example: live order book

This is a complete, runnable `main.rs` modeled on the SDK's `print_book` utility. It builds a testnet snapshot for one market, prints the market info and the top of book, then streams events and reprints whenever the book changes.

```rust
use std::time::Duration;

use alloy::{
    providers::ProviderBuilder,
    rpc::client::RpcClient,
    transports::layers::RetryBackoffLayer,
};
use futures::StreamExt;
use perpl_sdk::{
    Chain,
    state::{OrderBook, Perpetual, SnapshotBuilder},
    stream,
    types::{PerpetualId, StateInstant},
};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let chain = Chain::testnet();
    let market: PerpetualId = 16; // testnet BTC
    let rpc_url = "https://testnet-rpc.monad.xyz";

    // Guard: make sure the market is listed on this chain.
    if !chain.perpetuals().contains(&market) {
        eprintln!(
            "market {} not on this chain; available: {:?}",
            market,
            chain.perpetuals(),
        );
        std::process::exit(1);
    }

    // Build an RPC client with a retry/backoff layer and a poll interval.
    let client = RpcClient::builder()
        .layer(RetryBackoffLayer::new(10, 100, 200))
        .connect(rpc_url)
        .await?;
    client.set_poll_interval(Duration::from_millis(500));
    let provider = ProviderBuilder::new().connect_client(client);

    // 1. Initial snapshot for the one market we care about.
    let mut exchange = SnapshotBuilder::new(&chain, provider.clone())
        .with_perpetuals(vec![market])
        .build()
        .await?;

    let instant = exchange.instant();
    println!(
        "snapshot at block {} (ts {})",
        instant.block_number(),
        instant.block_timestamp(),
    );

    // 2. Print initial state.
    if let Some(perp) = exchange.perpetuals().get(&market) {
        print_market_info(perp);
        print_top_of_book(perp.l3_book());
    }

    println!("\nlistening for updates (Ctrl+C to stop) ...");

    // 3. Stream events and keep the cache current.
    let mut events = Box::pin(stream::raw(
        &chain,
        provider,
        StateInstant::new(instant.block_number() + 1, 0),
        tokio::time::sleep,
    ));

    while let Some(result) = events.next().await {
        match result {
            Ok(block_events) => {
                let block_num = block_events.instant().block_number();
                match exchange.apply_events(&block_events) {
                    Ok(Some(_)) => {
                        if let Some(perp) = exchange.perpetuals().get(&market) {
                            println!(
                                "\nblock {} | last {} | mark {} | oracle {}",
                                block_num,
                                perp.last_price(),
                                perp.mark_price(),
                                perp.oracle_price(),
                            );
                            print_top_of_book(perp.l3_book());
                        }
                    }
                    Ok(None) => { /* already applied */ }
                    Err(e) => eprintln!("apply_events error: {e:?}"),
                }
            }
            Err(e) => eprintln!("stream error: {e:?}"),
        }
    }

    Ok(())
}

fn print_market_info(perp: &Perpetual) {
    println!("--- {} ({}) [perp {}] ---", perp.name(), perp.symbol(), perp.id());
    println!("last / mark / oracle : {} / {} / {}",
        perp.last_price(), perp.mark_price(), perp.oracle_price());
    println!("funding rate         : {}", perp.funding_rate());
    println!("open interest        : {}", perp.open_interest());
    println!("maker / taker fee    : {} / {}", perp.maker_fee(), perp.taker_fee());
    println!("init / maint margin  : {} / {}", perp.initial_margin(), perp.maintenance_margin());
    println!("paused               : {}", perp.is_paused());
}

fn print_top_of_book(book: &OrderBook) {
    match (book.best_bid(), book.best_ask()) {
        (Some((bid_px, bid_sz)), Some((ask_px, ask_sz))) => {
            println!("best bid {} ({}) | best ask {} ({})", bid_px, bid_sz, ask_px, ask_sz);
        }
        _ => println!("(one side of the book is empty)"),
    }
    println!("book: {} orders, {} bid levels, {} ask levels",
        book.total_orders(), book.bids().len(), book.asks().len());
}
```

The `OrderBook` type also supports full L2 and L3 rendering. Iterate aggregated levels with `book.bids()` / `book.asks()` (each maps a price to a level with `.size()` and `.num_orders()`), and individual resting orders with `book.bid_orders()` / `book.ask_orders()`. Each `BookOrder` exposes `order_id()`, `account_id()`, `size()`, `price()`, `leverage()`, `expiry_block()`, and `r#type()` (an `OrderType`).

## Reading recent trades

For a normalized trade feed, wrap a `stream::raw` stream with `stream::trade`. It aggregates the raw maker/taker fill events into one `Trade` per taker, with the maker side broken out into individual fills. This is the `print_trades` utility, in full:

```rust
use std::{pin::pin, time::Duration};

use alloy::{
    providers::{Provider, ProviderBuilder},
    rpc::client::RpcClient,
    transports::layers::RetryBackoffLayer,
};
use futures::StreamExt;
use perpl_sdk::{Chain, stream, types::StateInstant};

#[tokio::main(flavor = "current_thread")]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = RpcClient::builder()
        .layer(RetryBackoffLayer::new(10, 100, 200))
        .connect("https://testnet-rpc.monad.xyz")
        .await?;
    client.set_poll_interval(Duration::from_millis(500));
    let provider = ProviderBuilder::new().connect_client(client);

    let chain = Chain::testnet();

    // Start from the current block.
    let block_num = provider.get_block_number().await?;
    println!("starting from block {block_num}");

    let raw = stream::raw(
        &chain,
        provider.clone(),
        StateInstant::new(block_num, 0),
        tokio::time::sleep,
    );

    // stream::trade returns a stream; pin it before polling.
    let mut trades = pin!(stream::trade(&chain, provider, raw).await?);

    println!("listening for trades ...\n");

    while let Some(Ok(block_events)) = trades.next().await {
        for entry in block_events.events() {
            let trade = entry.event();
            println!(
                "taker {} {:?} {} @ {} on perp {} (fee {})",
                trade.taker_account_id,
                trade.taker_side,
                trade.total_size(),
                trade.avg_price().unwrap_or_default(),
                trade.perpetual_id,
                trade.taker_fee,
            );
            for fill in &trade.maker_fills {
                println!(
                    "  <- maker {} order {} filled {} @ {} (fee {})",
                    fill.maker_account_id,
                    fill.maker_order_id,
                    fill.size,
                    fill.price,
                    fill.fee,
                );
            }
        }
    }

    Ok(())
}
```

A `Trade` carries `taker_account_id`, `taker_side`, `total_size()`, `avg_price()` (an `Option`), `perpetual_id`, and `taker_fee`; its `maker_fills` is a `Vec<MakerFill>`, each with `maker_account_id`, `maker_order_id`, `size`, `price`, and `fee`.

## Posting an order (outline)

Reading state needs only a read-only provider. To **trade**, you need a provider configured with a wallet (signer), then you build an `OrderRequest`, `prepare` it against the cached `Exchange`, and submit it to the Exchange contract.

**Choose a request type.** `RequestType` is a `u8` enum:

| Value | Variant                      | Side | Notes                  |
| ----- | ---------------------------- | ---- | ---------------------- |
| 0     | `OpenLong`                   | Bid  |                        |
| 1     | `OpenShort`                  | Ask  |                        |
| 2     | `CloseLong`                  | Ask  | reduce-only            |
| 3     | `CloseShort`                 | Bid  | reduce-only            |
| 4     | `Cancel`                     | —    | cancel a resting order |
| 5     | `IncreasePositionCollateral` | —    |                        |
| 6     | `Change`                     | —    | amend a resting order  |

**Build and prepare the request.** `OrderRequest::new` takes the market, the request type, price/size as fixed-point decimals (`fastnum` `UD64`), leverage, and the order flags. `prepare(&Exchange)` scales the human-readable decimals to the on-chain fixed-point representation and returns an `OrderDesc` ready to send:

```rust
use perpl_sdk::types::{OrderRequest, RequestType};

let request = OrderRequest::new(
    /* request_id             */ 1,                      // becomes the on-chain client order ID
    /* perpetual_id           */ market,
    /* request_type           */ RequestType::OpenLong,
    /* order_id               */ None,                   // None for a new order
    /* price                  */ fastnum::udec64!(65000),
    /* size                   */ fastnum::udec64!(0.01),
    /* expiry_block           */ None,
    /* post_only              */ true,
    /* fill_or_kill           */ false,
    /* immediate_or_cancel    */ false,
    /* max_matches            */ None,
    /* leverage               */ fastnum::udec64!(1),
    /* last_exec_block        */ None,
    /* amount                 */ None,
    /* max_neg_pnl_collat_bps */ 0,
);

let desc = request.prepare(&exchange);
```

{% hint style="warning" %}
`OrderRequest::new` has a long positional signature. Confirm the exact argument order and types against the rustdoc (`cargo doc -p perpl-sdk --no-deps --open`) or `crates/sdk/src/types/request.rs` before relying on it. `immediate_or_cancel` (IoC) and `fill_or_kill` (FOK) are mutually-relevant execution flags; `post_only` makes the order maker-only.
{% endhint %}

**Submit to the Exchange contract.** The canonical entry point documented by the SDK is `Exchange::ExchangeInstance::execOrders`. The shipped examples submit through `execOrders`, passing the prepared order descriptors and a `revertOnFail` flag, then await the receipt. Your provider must be a wallet-enabled `alloy` provider (a `DynProvider` built with `.wallet(wallet)`):

```rust
// `instance` is an Exchange contract instance bound to a wallet-enabled provider.
let receipt = instance
    .execOrders(
        vec![desc],    // one or more prepared OrderDescs
        true,          // revertOnFail: all-or-nothing
    )
    .send()
    .await?
    .get_receipt()
    .await?;
```

{% hint style="info" %}
Full signer wiring — loading a private key, building the wallet-enabled `DynProvider`, and driving order submission in a loop — is shown in the `market-making` example (`dex-sdk-examples/market-making`), which implements best-bid-offer (BBO), spread, and taker strategies on top of exactly this flow.
{% endhint %}

## Inspecting state without writing code

For quick, one-off inspection you do not need to write a program — the SDK ships the `perpl-cli` binary, which uses the same snapshot/stream machinery:

```bash
# Print an order book for testnet BTC (perp 16), 10 levels deep.
perpl-cli --testnet show book --perp 16 --depth 10

# Print an account's recent trades.
perpl-cli --testnet show account --account <ACCOUNT_ID> --num-trades 10
```

`perpl-cli` defaults to mainnet; pass `--testnet` for testnet, or `--rpc <URL>` / `--exchange <ADDRESS>` for a custom deployment. See the CLI reference for the full command set (`snapshot`, `trace`, `show account`, `show book`, `show trades`, `block <n>`, `tx <hash>`).

## Limitations & follow-ups

The SDK documents these current limitations, worth knowing before you build on it:

* **Funding-event processing is not yet implemented** — funding data is a planned follow-up.
* **Event streaming uses log polling.** Future versions may lower indexing latency with WebSocket subscriptions and/or Monad execution events. Because it polls, always run behind a retry/backoff layer.
* **Test coverage is limited.** Treat the SDK as early-stage and validate against testnet before going to mainnet.

## Next steps

* [Networks & Configuration](../networks-and-configuration.md) — every endpoint, contract address, chain ID, and market ID for both networks.
* Generate the full API reference locally: `cargo doc -p perpl-sdk --no-deps --open`.
* Explore the `dex-sdk-examples` workspace: `utilities` (`print_book`, `print_trades`) and `market-making` (BBO, spread, and taker strategies).
