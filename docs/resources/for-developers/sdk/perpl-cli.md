# Perpl-CLI

`perpl-cli` is a command-line tool for **reading and tracing Perpl exchange state and events** directly from the chain. Use it to take a point-in-time snapshot of the exchange, follow the live event stream, or inspect a single account, order book, block, or transaction — all without writing any code.

It is a thin wrapper over the [Perpl Rust SDK](https://github.com/PerplFoundation/dex-sdk). Under the hood it builds a state snapshot with the SDK's `SnapshotBuilder` and then follows the chain with the SDK's raw event stream, so anything the CLI prints, you can reproduce programmatically with the SDK.

{% hint style="info" %}
`perpl-cli` is read-only. It never signs or sends transactions — it only queries an RPC (remote procedure call) endpoint. To place or cancel orders, use the SDK's order-posting helpers.
{% endhint %}

## Installing

`perpl-cli` ships as the `crates/cli` member of the `perpl-sdk` workspace and is built from source with Cargo. You need **Rust `>= 1.85.0`** (the workspace uses edition 2024).

From the `dex-sdk` workspace root:

```bash
# Build an optimized binary at target/release/perpl-cli
cargo build --release -p perpl-cli

# ...or run it directly through Cargo (arguments after `--` go to the CLI)
cargo run -p perpl-cli -- snapshot
```

For the examples on this page, assume `perpl-cli` is on your `PATH` (for example by copying `target/release/perpl-cli` into a directory on your `PATH`). If you prefer to run through Cargo, replace `perpl-cli` with `cargo run -p perpl-cli --` in any example below.

## Usage

```
perpl-cli [OPTIONS] <COMMAND>
```

All options are **global** — they can appear before the command or after it. Command-specific options (such as `--depth` for `show book`) must follow their subcommand.

By default `perpl-cli` targets **Mainnet** (chain ID `143`). Pass `--testnet` to target **Testnet** (chain ID `10143`). Each network has its own default RPC endpoint and Exchange contract address; see [Networks & Configuration](../networks-and-configuration.md) for the full per-network reference.

## Commands

| Command                | Purpose                                                                                                       |
| ---------------------- | ------------------------------------------------------------------------------------------------------------- |
| `snapshot`             | Fetch the exchange state at a single block height and print it.                                               |
| `trace`                | Take an initial snapshot, then follow (trace) events block by block, and print the final state when it stops. |
| `show account`         | Print live account state (balances, positions, orders, recent trades).                                        |
| `show book`            | Print a perpetual's live order book.                                                                          |
| `show trades`          | Print recent trades.                                                                                          |
| `block <BLOCK_NUMBER>` | Trace the raw events emitted in one specific block.                                                           |
| `tx <TX_HASH>`         | Trace the raw events emitted by one specific transaction.                                                     |

### `snapshot`

Captures the full exchange state at one block and prints it. With no filters it snapshots every perpetual and account the exchange knows about; narrow it with `--perp` and/or `--account`. Pin the block with `--block` (defaults to the latest block).

```bash
# Full exchange snapshot at the latest mainnet block
perpl-cli snapshot

# Snapshot of BTC (perpetual 1) only, at a specific historical block
perpl-cli --perp 1 --block 55000000 snapshot

# Snapshot of a single account (by account ID) across all perpetuals
perpl-cli --account 42 snapshot
```

### `trace`

Takes an initial snapshot, then follows the event stream forward from that point, applying each block's events to keep the state current. It prints the final state when it stops. Without `--num-blocks` it runs until you interrupt it with `Ctrl+C`.

```bash
# Follow all mainnet events indefinitely (Ctrl+C to stop and print final state)
perpl-cli trace

# Trace BTC on testnet, starting at a given block, for 100 blocks then stop
perpl-cli --testnet --perp 16 --block 62953 --num-blocks 100 trace
```

{% hint style="info" %}
Tracing follows finalized blocks. On Monad the `latest` tag is a _proposed_ (not-yet-final) block, so the stream waits for a block to reach the `safe` tag before emitting its events. This keeps the traced state consistent at the cost of a short lag behind the chain head.
{% endhint %}

### `show account`

Prints the live state of one account: balances, open positions, resting orders, and (by default) its most recent trades. `--account` is **required** for this command.

**Command option:**

| Option             | Default | Meaning                                                        |
| ------------------ | ------- | -------------------------------------------------------------- |
| `--num-trades <N>` | `10`    | Number of recent trades to show. `0` hides the trades section. |

```bash
# Show account 42 with the default 10 recent trades
perpl-cli --account 42 show account

# Show account 42 with 50 recent trades
perpl-cli --account 42 show account --num-trades 50

# Show account state without any trade history
perpl-cli --account 42 show account --num-trades 0
```

You can also identify the account by its on-chain address instead of its numeric ID:

```bash
perpl-cli --account 0xYourAccountAddress show account
```

### `show book`

Prints a perpetual's live order book. `--perp` is **required** for this command.

**Command options:**

| Option                   | Default | Meaning                                                                                                                   |
| ------------------------ | ------- | ------------------------------------------------------------------------------------------------------------------------- |
| `-d`, `--depth <N>`      | `10`    | Number of price levels per side to show. `0` shows all levels.                                                            |
| `--orders-per-level <N>` | `10`    | Number of individual orders to show at each price level (the book is level-3, i.e. order-by-order). `0` shows all orders. |
| `--show-expired`         | `false` | Include expired orders in the output.                                                                                     |

```bash
# Top 10 levels of the BTC (perpetual 1) book on mainnet
perpl-cli --perp 1 show book

# Deeper view: 25 levels, all orders at each level
perpl-cli --perp 1 show book --depth 25 --orders-per-level 0

# Full book on testnet BTC (perpetual 16), including expired orders
perpl-cli --testnet --perp 16 show book --depth 0 --show-expired
```

### `show trades`

Prints recent trades. With no `--perp` it shows trades across all perpetuals; pass `--perp` (repeatable) to filter.

```bash
# Recent trades across all mainnet perpetuals
perpl-cli show trades

# Recent trades for ETH (perpetual 20) only
perpl-cli --perp 20 show trades

# Recent trades for BTC and ETH
perpl-cli --perp 1 --perp 20 show trades
```

### `block` and `tx`

Inspect the raw exchange events emitted by a single block or a single transaction. These are handy for debugging a specific on-chain action.

```bash
# All exchange events emitted in one block
perpl-cli block 55123456

# All exchange events emitted by one transaction
perpl-cli tx 0xabc123...def
```

## Options

Every option below is global and applies to all commands.

| Option                              | Default                                                                                                      | Meaning                                                                                                                                                                             |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--testnet`                         | off (Mainnet)                                                                                                | Target Testnet instead of Mainnet. Switches the default RPC endpoint and Exchange address to their testnet values.                                                                  |
| `--rpc <RPC>`                       | Mainnet: `https://rpc.monad.xyz`; Testnet: `https://testnet-rpc.monad.xyz`                                   | RPC endpoint to connect to. Supply your own node URL to override the default.                                                                                                       |
| `--rpc-throttle <REQ_PER_SEC>`      | `15` for the built-in RPC endpoints; none for a custom `--rpc`                                               | Client-side rate limit in requests per second.                                                                                                                                      |
| `--exchange <ADDRESS>`              | Mainnet: `0x34B6552d57a35a1D042CcAe1951BD1C370112a6F`; Testnet: `0x1964C32f0bE608E7D29302AFF5E61268E72080cc` | Exchange contract address. Override for a non-standard deployment.                                                                                                                  |
| `--block <BLOCK>`                   | latest block                                                                                                 | Block number to fetch state at (`snapshot`) or to start tracing from (`trace`).                                                                                                     |
| `--num-blocks <NUM_BLOCKS>`         | unlimited (until `Ctrl+C`)                                                                                   | Number of blocks to trace or show.                                                                                                                                                  |
| `--account <ADDRESS or ACCOUNT_ID>` | all accounts for `snapshot`/`trace`; **required** for `show account`                                         | Account to snapshot, trace, or show. Repeatable — pass the flag multiple times to select several accounts. Accepts either the numeric account ID or the account's on-chain address. |
| `--perp <PERPETUAL_ID>`             | all perpetuals for `snapshot`/`trace`/`show trades`; **required** for `show book`                            | Perpetual (market) to operate on. Repeatable.                                                                                                                                       |

{% hint style="info" %}
Perpetual IDs are network-specific — for example BTC is perpetual `1` on Mainnet but `16` on Testnet. See the market tables in [Networks & Configuration](../networks-and-configuration.md), or run a `snapshot` with no `--perp` filter to list every market on the target network.
{% endhint %}

### Using a custom RPC endpoint

Point the CLI at your own node — for example a private archival node or a local test chain — with `--rpc`. When you supply a custom endpoint the built-in 15 req/sec throttle is disabled; set `--rpc-throttle` yourself if your provider enforces a rate limit.

```bash
# Snapshot against a private node, throttled to 30 requests/second
perpl-cli --rpc https://my-node.example.com --rpc-throttle 30 snapshot

# Point at a non-standard Exchange deployment on a custom RPC
perpl-cli --rpc http://127.0.0.1:8545 --exchange 0xYourExchangeAddress snapshot
```

## Common tasks

| I want to…                                 | Command                                        |
| ------------------------------------------ | ---------------------------------------------- |
| See the whole exchange right now (mainnet) | `perpl-cli snapshot`                           |
| See one market's state at a past block     | `perpl-cli --perp 1 --block 55000000 snapshot` |
| Watch the order book for BTC live          | `perpl-cli --perp 1 show book`                 |
| Watch one account's positions and orders   | `perpl-cli --account 42 show account`          |
| Stream recent trades for a market          | `perpl-cli --perp 20 show trades`              |
| Follow all events on testnet for a while   | `perpl-cli --testnet --num-blocks 500 trace`   |
| Debug what one transaction did             | `perpl-cli tx 0xabc123...def`                  |
| Debug what happened in one block           | `perpl-cli block 55123456`                     |

## See also

* [Networks & Configuration](../networks-and-configuration.md) — chain IDs, RPC URLs, Exchange addresses, and market IDs for both networks.
* [Perpl Rust SDK](https://github.com/PerplFoundation/dex-sdk) — the library `perpl-cli` is built on, for programmatic state snapshots, event streams, and order posting.
