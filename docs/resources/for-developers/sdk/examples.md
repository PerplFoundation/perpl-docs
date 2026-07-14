# Examples

The [`dex-sdk-examples`](https://github.com/PerplFoundation/dex-sdk-examples) repository is a small Cargo workspace of runnable programs built on top of the Perpl SDK (`perpl-sdk`). It shows how to build an exchange snapshot, stream on-chain events, keep a local cache current, and submit orders — the same building blocks you would use in a production trading bot.

There are two packages:

| Package                   | What it is                                                                                           | Binaries                                            |
| ------------------------- | ---------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| `perpl_market_making_bot` | A configurable trading bot with three interchangeable strategies (best bid and offer, spread, taker) | `perpl_market_making_bot`                           |
| `perpl_utilities`         | Read-only tools that stream and pretty-print live exchange state                                     | `print_book`, `print_trades`, `perpl_test_exchange` |

> **Note:** The examples workspace pins `alloy = "1.4.0"`, while the `perpl-sdk` crate itself uses `alloy = "2.0.4"`. If you copy code from the examples into a project that also depends on a newer SDK build, align the `alloy` version to avoid duplicate-crate type mismatches.

The examples depend on the SDK **by path** (`perpl-sdk = { path = "../dex-sdk/crates/sdk" }`), so they expect the [`dex-sdk`](https://github.com/PerplFoundation/dex-sdk) repository to be checked out as a sibling directory:

```
parent/
├── dex-sdk/            # the perpl-sdk crate
└── dex-sdk-examples/   # this repository
```

***

## Prerequisites

* **Rust 1.85.0 or newer** (the SDK uses edition 2024).
* **Foundry `anvil`** — only needed for the local test exchange (`perpl_test_exchange`); not required to run against testnet.
* A checkout of the `dex-sdk` repository as a sibling of `dex-sdk-examples` (see above).

## Clone and build

```bash
git clone https://github.com/PerplFoundation/dex-sdk-examples.git
cd dex-sdk-examples
cargo build
```

All commands below are run from the workspace root. `cargo run --bin <name>` builds and runs a single binary.

***

## The market-making bot

`perpl_market_making_bot` is one bot that can run any of three strategies. **Connection and account settings come from the environment** (loaded from a `.env` file); **the strategy and its parameters come from the command line.**

### Configuration

The bot reads a `PerplConfig` from environment variables via [`envy`](https://crates.io/crates/envy) + [`dotenvy`](https://crates.io/crates/dotenvy). Create a `.env` file in the workspace root:

```dotenv
# Chain / exchange the bot connects to
CHAIN_ID=10143
COLLATERAL_TOKEN_ADDRESS="0xa9012a055bd4e0eDfF8Ce09f960291C09D5322dC"
ADDRESS="0x1964C32f0bE608E7D29302AFF5E61268E72080cc"
DEPLOYED_AT_BLOCK=62953
PERPETUAL_ID=16
NODE_RPC_URL="https://testnet-rpc.monad.xyz"

# Optional: seconds between "run anyway" ticks when no events arrive (default 30)
# TIMEOUT_SECONDS=30

# The account the bot trades from. Use your own key.
PRIVATE_KEY="<your-private-key>"
```

The example above targets **testnet** (BTC is perpetual `16`). The values are:

| Variable                   | Meaning                                                   | Testnet value (from `Chain::testnet()`)      |
| -------------------------- | --------------------------------------------------------- | -------------------------------------------- |
| `CHAIN_ID`                 | Chain ID                                                  | `10143`                                      |
| `COLLATERAL_TOKEN_ADDRESS` | Collateral token (ERC-20) address                         | `0xa9012a055bd4e0eDfF8Ce09f960291C09D5322dC` |
| `ADDRESS`                  | Exchange contract address                                 | `0x1964C32f0bE608E7D29302AFF5E61268E72080cc` |
| `DEPLOYED_AT_BLOCK`        | Block the exchange was deployed at (snapshot lower bound) | `62953`                                      |
| `PERPETUAL_ID`             | Perpetual market to trade                                 | `16` (BTC)                                   |
| `NODE_RPC_URL`             | Monad JSON-RPC endpoint                                   | `https://testnet-rpc.monad.xyz`              |
| `TIMEOUT_SECONDS`          | Fallback strategy-run interval when no events arrive      | `30` (default)                               |
| `PRIVATE_KEY`              | Private key for the trading account                       | your key                                     |

> **Note:** The `.env` shipped in the repository points at a **local Anvil** instance (`CHAIN_ID=1337`, `NODE_RPC_URL="http://localhost:52778/"`) and ships a well-known Anvil development key. Use those defaults only against the local test exchange (`perpl_test_exchange`); replace every value with your own for testnet or mainnet, and never commit a real private key.

The bot builds a chain config from these values with `Chain::custom(...)`:

```rust
Chain::custom(
    perpl_config.chain_id,
    collateral_token_address,
    perpl_config.deployed_at_block,
    address,                       // exchange address
    vec![perpl_config.perpetual_id],
)
```

### Run a strategy

The bot takes a strategy **subcommand** plus that strategy's flags:

```bash
# Best bid and offer (BBO) — one bid + one ask that track the top of book
cargo run --bin perpl_market_making_bot -- bbo --order-size 0.01

# Spread — a ladder of quotes on each side of the mark price
cargo run --bin perpl_market_making_bot -- spread \
  --orders-per-side 5 \
  --order-size 0.01 \
  --leverage 2 \
  --max-matches 3

# Taker — repeatedly crosses the book to buy and sell
cargo run --bin perpl_market_making_bot -- taker --order-size 0.01 --leverage 2
```

Strategy flags:

| Subcommand | Flag                | Required | Meaning                                                   |
| ---------- | ------------------- | -------- | --------------------------------------------------------- |
| `bbo`      | `--order-size`      | yes      | Size of each quote                                        |
| `spread`   | `--orders-per-side` | yes      | Number of orders to place on each side                    |
| `spread`   | `--order-size`      | yes      | Size of each order                                        |
| `spread`   | `--max-matches`     | no       | Max matches per order (`max_matches`)                     |
| `spread`   | `--leverage`        | no       | Leverage for each order (default `1`)                     |
| `taker`    | `--order-size`      | yes      | Maximum order size (actual size is randomized up to this) |
| `taker`    | `--leverage`        | no       | Leverage for each order (default `1`)                     |

Set `RUST_LOG` to control log verbosity (the bot defaults to `info` if unset):

```bash
RUST_LOG=debug cargo run --bin perpl_market_making_bot -- bbo --order-size 0.01
```

### How the bot loop works

`PerplMarketMakingBot::try_new(...)` wires up an `alloy` provider with your wallet and an `ExchangeInstance`, then `run()` executes this loop (`market-making/src/lib.rs`):

{% stepper %}
{% step %}
## Snapshot

`SnapshotBuilder::new(&chain, provider).with_accounts(...).with_perpetuals(...).build()` fetches the current exchange state for your account and the target perpetual.
{% endstep %}

{% step %}
## Initialize

`strategy.initialize(&instance, &exchange)` runs one-time setup (BBO cancels any resting orders; all strategies resolve and store your account ID).
{% endstep %}

{% step %}
## Stream

`stream::raw(&chain, provider, exchange.instant(), tokio::time::sleep)` produces per-block raw contract events, pinned on the stack.
{% endstep %}

{% step %}
## Select

A `tokio::select!` reacts to whichever fires first:

* a new stream event → `exchange.apply_events(...)` updates the cache, then `strategy.execute(...)` runs on the resulting state events;
* an error reported back from a previous submission → re-run `execute`;
* a timeout tick (every `TIMEOUT_SECONDS`) → run `execute` anyway, in case the market is quiet.
{% endstep %}

{% step %}
## Concurrency guard

A `Semaphore(1)` permit gates `execute`. If a prior submission is still in flight, the current batch is skipped rather than double-submitting.
{% endstep %}

{% step %}
## Auto-restart

If the stream closes or errors, the outer loop rebuilds the snapshot and starts over.
{% endstep %}
{% endstepper %}

> **Note:** `stream::raw` and `stream::trade` are **not cancellation-safe** — do not drop them across an `await` in a `select!` arm without pinning, as the example does with `pin!(...)`.

### How orders are submitted

Every strategy expresses intent as an `OrderRequest`, calls `.prepare(&exchange)` to scale human-readable decimals into the contract's fixed-point `OrderDesc`, then submits a batch:

```rust
let builder = instance.execOrders(order_descs, /* revert_on_fail */ true);
let res = builder.send().await?;
let receipt = res.get_receipt().await?;
```

`execOrders(orderDescs, revertOnFail)` takes the prepared order descriptions and a `revertOnFail` flag — when `true`, the whole batch reverts together if any order fails.

An `OrderRequest` is constructed positionally. The BBO strategy's `place_order` shows the shape:

```rust
let request = OrderRequest::new(
    0,                    // request_id (becomes the on-chain client order id)
    self.perpetual_id,    // perpetual id
    order_type,           // RequestType: OpenLong / OpenShort / CloseLong / CloseShort / Cancel / Change
    None,                 // order_id (Some(id) when amending/cancelling)
    price,                // price (UD64)
    self.order_size,      // size (UD64)
    None,                 // expiry_block
    true,                 // post_only — provide liquidity, never cross
    false,                // fill_or_kill
    false,                // immediate_or_cancel
    None,                 // max_matches
    UD64::ONE,            // leverage
    None,                 // last_exec_block
    None,                 // amount
    0u16,                 // max_neg_pnl_collat_bps
);
let desc = request.prepare(exchange);
```

`RequestType` values used by the strategies:

| `RequestType` | Effect                                                   |
| ------------- | -------------------------------------------------------- |
| `OpenLong`    | Open/add to a long (a bid)                               |
| `OpenShort`   | Open/add to a short (an ask)                             |
| `CloseLong`   | Reduce/close a long (reduce-only ask)                    |
| `CloseShort`  | Reduce/close a short (reduce-only bid)                   |
| `Cancel`      | Cancel a resting order (`order_id` required)             |
| `Change`      | Amend a resting order's price/size (`order_id` required) |

***

### Strategy: BBO

**File:** `market-making/src/strategies/bbo.rs` — best bid and offer (BBO).

Keeps exactly one bid and one ask quoting at the current top of book.

* **Initialize:** requires exactly one account in the snapshot (errors otherwise), stores the account ID, then **cancels all existing orders** in one atomic batch.
* **Execute:** acts **only when a fill event is present** in the block's state events. On a fill it reads the current best bid and best ask from the level-3 (L3) book:
  * if there is no resting bid, place a `post_only` `OpenLong` at the best bid; if there is one and it is below the best bid, `Change` it up to the best bid;
  * symmetrically for the ask side with `OpenShort`.
* Orders use leverage `1` and are submitted **atomically**. The receipt is awaited on a spawned task so the loop keeps consuming events; errors flow back through the `error_tx` channel.

```bash
cargo run --bin perpl_market_making_bot -- bbo --order-size 0.01
```

### Strategy: Spread

**File:** `market-making/src/strategies/spread.rs`

Maintains a ladder of `orders_per_side` quotes on each side, stepped away from the mark price.

* **Target prices:** for `i` in `1..=orders_per_side`, the offset is `i / 500` (≈ 0.2% per step). Bid prices are `mark * (1 - offset)`, ask prices are `mark * (1 + offset)`.
* **Reconciliation** (`create_target_order_changes`): for each target price it keeps an existing order if the size already matches, `Change`s it if the size differs, reuses a spare order at a new price, or places a new `post_only` order for anything left over. This minimizes churn versus cancel-and-replace.
* **Ordering:** whether bids or asks are submitted first depends on the mark-price direction (`bids_first` when the new mark is at or below the previous mark), so the side moving _toward_ the market is refreshed first.
* Orders honor the optional `--leverage` and `--max-matches` flags and are submitted **non-atomically** (`atomic = false`).

```bash
cargo run --bin perpl_market_making_bot -- spread --orders-per-side 5 --order-size 0.01
```

### Strategy: Taker

**File:** `market-making/src/strategies/taker.rs`

A liquidity-taking stress/demo strategy that repeatedly crosses the book.

* **Side:** chosen randomly each run via a `Bernoulli(0.5)` distribution (long or short).
* **Size:** a random fraction in `(0, 1]` (`OpenClosed01`) times `--order-size`.
* **Position handling:** if a position exists on the opposite side, it is closed first (`CloseLong` / `CloseShort` for the full position size), then a new position is opened.
* **Crossing price:** opening orders are `immediate_or_cancel` (IoC) with a price of `UD64::MAX` for a buy and `UD64::ZERO` for a sell, so they always cross whatever is resting. Submitted **non-atomically**.

```bash
cargo run --bin perpl_market_making_bot -- taker --order-size 0.01
```

### Add your own strategy

All three implement the `Strategy` trait (`market-making/src/strategies/mod.rs`):

```rust
pub trait Strategy {
    fn name(&self) -> &'static str;
    fn perpetual_id(&self) -> PerpetualId;

    fn initialize(
        &mut self,
        instance: &ExchangeInstance<DynProvider>,
        exchange: &Exchange,
    ) -> impl Future<Output = Result<()>>;

    fn execute(
        &mut self,
        instance: &ExchangeInstance<DynProvider>,
        exchange: &Exchange,
        events: &[StateEvents],
        error_tx: &mpsc::Sender<DexError>,
        permit: OwnedSemaphorePermit,
    ) -> impl Future<Output = ()>;
}
```

To add a strategy: implement the trait for a new struct, add a variant to the `StrategyType` enum (which fans method calls out to each concrete strategy), and add a `clap` subcommand in `main.rs` that constructs it.

***

## The utilities

The `perpl_utilities` package holds read-only tools. They are the fastest way to confirm your RPC endpoint and market are live and to see the SDK's state types in action.

### `print_book`

Streams a single market's order book and reprints it whenever state changes. Exercises the `Perpetual` accessors and the `OrderBook` level-2 (L2), level-3 (L3), and compact renderers.

```bash
cargo run --bin print_book -- --market 16 --rpc-url https://testnet-rpc.monad.xyz
```

Flags (`utilities/src/print-book.rs`):

| Flag                    | Default      | Meaning                                                          |
| ----------------------- | ------------ | ---------------------------------------------------------------- |
| `-c`, `--chain`         | `testnet`    | Chain to connect to. **Only `testnet` is supported.**            |
| `-m`, `--market`        | — (required) | Perpetual market ID (e.g. `16` for BTC on testnet)               |
| `-r`, `--rpc-url`       | — (required) | Monad JSON-RPC URL                                               |
| `-d`, `--depth`         | `10`         | Price levels to display (`0` = all)                              |
| `-p`, `--poll-interval` | `500`        | RPC poll interval in milliseconds                                |
| `--mode`                | `l3`         | `l2` (aggregated levels), `l3` (individual orders), or `compact` |
| `--orders-per-level`    | `5`          | Max orders shown per level in L3/compact (`0` = all)             |

It first prints a market-info block (name, last/mark/oracle price, funding rate, open interest, fees, margins, paused flag) and the initial book, then uses a retry-backoff RPC client and `stream::raw` to reprint on every block that produces state events. The market ID must be one of the chain's perpetuals or the tool exits.

### `print_trades`

Streams and prints normalized trades from **testnet**. It has no command-line flags — the RPC endpoint (`https://testnet-rpc.monad.xyz`) and `Chain::testnet()` are hard-coded, and it starts from the current block.

```bash
cargo run --bin print_trades
```

It pipes `stream::raw` into `stream::trade`, which aggregates maker and taker fills into per-taker `Trade`s. For each block it prints the taker (account, side, total size, average price, perpetual, fee) and every maker fill (`maker_account_id`, `maker_order_id`, size, price, fee) — a good template for downstream trade analytics.

### `perpl_test_exchange`

Starts a local **Anvil** instance with the Perpl exchange deployed and seeded test accounts, using the SDK's `testing::TestExchange` helper (requires the SDK `testing` feature). It creates two maker accounts (IDs `0` and `1`) and one taker (ID `2`), each funded with 1,000,000 units of collateral, plus a BTC perpetual market, then logs the exchange address and RPC URL and stays running.

```bash
cargo run --bin perpl_test_exchange
```

Point the market-making bot's `.env` at the address, RPC URL, and chain ID this prints to drive strategies against a fully local exchange — no testnet funds or connectivity required.

***

## Next steps

* Chain configs, market IDs, and endpoints for every network: [Networks](../networks-and-configuration.md).
* Generate the full SDK API docs locally: `cargo doc -p perpl-sdk --no-deps --open`.
* For ad-hoc live inspection without writing code, use the `perpl-cli` tool shipped in the `dex-sdk` repository.
