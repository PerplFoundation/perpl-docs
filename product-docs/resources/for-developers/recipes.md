# Recipes

Copy-pasteable, task-oriented snippets for the most common Perpl integration jobs: placing and cancelling an order, streaming your own fills, reading live positions and balances, subscribing to the order book, and attaching a take-profit / stop-loss (TP/SL) to a position.

Each recipe shows the direct-API approach first (REST over HTTPS and the WebSocket streams) and then notes the equivalent in the Rust SDK (software development kit, the `perpl-sdk` crate) where one exists. Acronyms used throughout: **REST** (Representational State Transfer), **WSS** (WebSocket Secure), **API** (application programming interface), **RPC** (remote procedure call), **IOC** (immediate-or-cancel), **FOK** (fill-or-kill), **GTC** (good-till-cancel), **bps** (basis points), **PnL** (profit and loss), **FIFO** (first-in-first-out).

> **Note:** These recipes assume you have already created an API key and set your environment variables. If not, start with the [Quickstart](quickstart.md) and the [Networks & Configuration](networks-and-configuration.md) reference.

***

## Before you start

Two things every recipe reuses: environment configuration and a request signer.

### Environment

API keys are **Ed25519 key pairs** (a modern elliptic-curve signature scheme). The server stores only your public key; the private key never leaves your machine, and **every request is signed** — there is no bearer token or session cookie.

```typescript
// Mainnet defaults; set the testnet values to target testnet.
const API_URL  = process.env.PERPL_API_URL || 'https://app.perpl.xyz/api'; // REST base URL — includes /api
const WS_URL   = process.env.PERPL_WS_URL  || 'wss://app.perpl.xyz';       // WebSocket base URL — NO /api prefix
const CHAIN_ID = Number(process.env.PERPL_CHAIN_ID) || 143;                // 143 mainnet, 10143 testnet

// Enrolled key (see the Quickstart):
const API_KEY    = process.env.PERPL_API_KEY!;                             // opaque X-API-Key token
const privateKey = Buffer.from(
  (process.env.PERPL_API_KEY_SECRET ?? '').replace(/^0x/, ''),
  'hex',
);                                                                          // 32-byte Ed25519 private key

// Market IDs are network-specific — never hard-code across networks.
// Mainnet: BTC=1, MON=10, ETH=20, SOL=31, HYPE=40, ZEC=50
// Testnet: BTC=16, ETH=32, SOL=48, MON=64, ZEC=256
const MARKETS = { BTC: 1, MON: 10, ETH: 20, SOL: 31, HYPE: 40, ZEC: 50 } as const;
```

### The `signedFetch` helper (REST)

Every REST call is signed over a **canonical string** of six fields joined by newlines: `<chain_id>`, `<HTTP_METHOD>`, `<request-target>` (path + query string exactly as sent), `<timestamp_ms>`, `<nonce>` (client-random, base64url, no padding), `<sha256(body) hex>`. The signature and three companion headers are sent as `X-API-*`.

```typescript
import { createHash, randomBytes } from 'crypto';
import * as ed from '@noble/ed25519';

// `target` is the path + query string exactly as sent, e.g. /v1/trading/fills?count=100
async function signedFetch(method: string, target: string, body = '') {
  const timestamp = Date.now().toString();
  const nonce = randomBytes(16).toString('base64url');
  const bodyHash = createHash('sha256').update(body).digest('hex');

  const canonical = [CHAIN_ID, method, target, timestamp, nonce, bodyHash].join('\n');
  const sig = await ed.signAsync(Buffer.from(canonical), privateKey);

  return fetch(`${API_URL}${target}`, {
    method,
    headers: {
      'X-API-Key': API_KEY,
      'X-API-Timestamp': timestamp,
      'X-API-Nonce': nonce,
      'X-API-Signature': Buffer.from(sig).toString('base64url'),
      ...(body ? { 'Content-Type': 'application/json' } : {}),
    },
    ...(body ? { body } : {}),
  });
}
```

{% hint style="info" %}
The `request-target` must match **byte-for-byte** what the server receives — include the query string exactly as sent, and sign that exact string. The timestamp must be within **±30 seconds** of server time, and each `nonce` is single-use. See [Authentication](api/authentication.md) for the full signing spec.
{% endhint %}

### Opening the trading WebSocket

Orders, fills, positions, and balances all flow over the authenticated trading WebSocket at `/ws/v1/trading`. The **first** frame after the socket opens must be a signed `ApiKeySignIn` frame (message type `mt: 29`). Placing orders requires a **`trade`-scoped** key; a `read`-scoped key still receives snapshots and updates but its order frames are rejected with `403`.

```typescript
import { randomBytes } from 'crypto';
import * as ed from '@noble/ed25519';

// Minimal trading-socket wrapper the recipes below build on. It authenticates,
// captures the account id + block height, and seeds the request-id counter.
function openTradingSocket(onMessage: (msg: any) => void) {
  const ws = new WebSocket(`${WS_URL}/ws/v1/trading`);
  const ctx = { accountId: 0, currentBlock: 0, nextRq: 0, lastSn: undefined as number | undefined };

  ws.onopen = async () => {
    // Canonical string for WS sign-in: 4 fields joined by "\n".
    const timestamp = Date.now().toString();
    const nonce = randomBytes(16).toString('base64url');
    const canonical = [CHAIN_ID, 'trading-ws-signin', timestamp, nonce].join('\n');
    const sig = await ed.signAsync(Buffer.from(canonical), privateKey);

    ws.send(JSON.stringify({
      mt: 29,               // ApiKeySignIn — must be the first frame
      chain_id: CHAIN_ID,
      api_key: API_KEY,
      timestamp,
      nonce,
      signature: Buffer.from(sig).toString('base64url'),
    }));
  };

  ws.onmessage = (event) => {
    const msg = JSON.parse(event.data);
    switch (msg.mt) {
      case 19: // WalletSnapshot — accounts, balances, and the sequence seed
        ctx.accountId = msg.as?.[0]?.id ?? ctx.accountId;
        // Seed the request-id counter from the account's last-forwarded id (lfr).
        ctx.nextRq = (msg.as?.[0]?.lfr ?? 0);
        ctx.lastSn = msg.sn;
        break;
      case 100: // Heartbeat — carries the head block number (h) and sequence (sn)
        if (ctx.lastSn != null && msg.sn !== ctx.lastSn + 1) {
          // Sequence gap → messages may have been lost; force a reconnect.
          ws.close();
          return;
        }
        ctx.lastSn = msg.sn;
        ctx.currentBlock = msg.h;
        break;
    }
    onMessage(msg);
  };

  // Keep-alive: send a Ping (mt: 1) about every 30 s.
  setInterval(() => {
    if (ws.readyState === WebSocket.OPEN) ws.send(JSON.stringify({ mt: 1, t: Date.now() }));
  }, 30_000);

  return { ws, ctx };
}
```

{% hint style="info" %}
On close code **3401** (authentication failure), reconnect and re-send a freshly signed `mt: 29` frame (new timestamp + nonce). Full connection, snapshot, sequence-tracking, and reconnection semantics are in the [WebSocket reference](api/websocket.md).
{% endhint %}

**SDK equivalent.** The Rust SDK does not use the WebSocket API. It maintains an in-memory cache of on-chain exchange state: build a snapshot with `state::SnapshotBuilder`, then keep it current from a per-block event stream (`stream::raw`) fed into `Exchange::apply_events`. See [SDK Concepts](sdk/concepts.md).

***

## Place and cancel an order

Orders are submitted as `OrderRequest` frames (`mt: 22`) on the trading WebSocket.

### Key fields

| Field         | Meaning                                                                                                                         |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `rq`          | Request ID — per-account idempotency key, **strictly increasing**. The server guarantees at-most-once execution per `rq`.       |
| `mkt` / `acc` | Market ID and your account ID.                                                                                                  |
| `t`           | Order type: `1` OpenLong, `2` OpenShort, `3` CloseLong, `4` CloseShort, `5` Cancel, `6` IncreasePositionCollateral, `7` Change. |
| `p`           | Limit price (scaled). **`0` = market order.**                                                                                   |
| `s`           | Size (scaled by the market's `size_decimals`).                                                                                  |
| `ms`          | Maximum market-order price slippage, in bps (recommended for market orders).                                                    |
| `fl`          | Flags: `0` GTC, `1` PostOnly, `2` FOK, `4` IOC.                                                                                 |
| `lv`          | Leverage in **hundredths** (`1000` = 10x).                                                                                      |
| `lb`          | Last execution block — the last Monad block at which the order is valid.                                                        |
| `oid`         | Order ID — required for Cancel / Change.                                                                                        |

{% hint style="info" %}
`rq` must be strictly increasing per account. Seed it from the account's last-forwarded request ID (`lfr`, delivered in the WalletSnapshot and AccountUpdate frames): `rq = max(localCounter, account.lfr) + 1`. Submitting `rq <= lfr` fails with reject reason `sr: 32` (OrderDescIdTooLow) — retry once with a fresh `rq`.
{% endhint %}

### Place a limit order (open long)

Prices and sizes are scaled integers. On mainnet BTC, `price_decimals = 1` (so `$95,000` → `950000`) and `size_decimals = 5` (so `0.1 BTC` → `10000`). Read the per-market decimals from `GET /api/v1/pub/context`.

```typescript
const { ws, ctx } = openTradingSocket((msg) => {
  if (msg.mt === 24) console.log('order update', msg.d);   // OrdersUpdate
});

// ...after the WalletSnapshot has populated ctx.accountId and ctx.nextRq:
function placeLimitLong(marketId: number, priceScaled: number, sizeScaled: number, leverageX: number) {
  const order = {
    mt: 22,
    rq: ++ctx.nextRq,               // strictly increasing per account
    mkt: marketId,
    acc: ctx.accountId,
    t: 1,                           // OpenLong
    p: priceScaled,                 // limit price (scaled); 0 would mean market
    s: sizeScaled,                  // size (scaled)
    fl: 0,                          // GTC
    lv: leverageX * 100,            // hundredths: 10x -> 1000
    lb: ctx.currentBlock + 100,     // valid for ~100 blocks
  };
  ws.send(JSON.stringify(order));
  return order.rq;                  // track this to correlate updates
}

// 0.1 BTC long at $95,000 with 10x leverage (mainnet scaling)
const rq = placeLimitLong(MARKETS.BTC, 95000 * 10, 10000, 10);
```

### Place a market order

Set `p: 0`, use the IOC flag, and bound your slippage with `ms`:

```typescript
function placeMarketLong(marketId: number, sizeScaled: number, leverageX: number, maxSlippageBps: number) {
  const order = {
    mt: 22,
    rq: ++ctx.nextRq,
    mkt: marketId,
    acc: ctx.accountId,
    t: 1,                           // OpenLong
    p: 0,                           // market
    s: sizeScaled,
    ms: maxSlippageBps,             // e.g. 50 = 0.5%
    fl: 4,                          // IOC — fill what crosses now, cancel the rest
    lv: leverageX * 100,
    lb: ctx.currentBlock + 100,
  };
  ws.send(JSON.stringify(order));
  return order.rq;
}
```

### Cancel an order

Cancel by order ID (`oid`) with order type `5`:

```typescript
function cancelOrder(marketId: number, orderId: number) {
  const order = {
    mt: 22,
    rq: ++ctx.nextRq,
    mkt: marketId,
    acc: ctx.accountId,
    oid: orderId,                   // the id of the order to cancel
    t: 5,                           // Cancel
    s: 0,
    fl: 0,
    lv: 0,
    lb: ctx.currentBlock + 100,
  };
  ws.send(JSON.stringify(order));
  return order.rq;
}
```

**Recommended validation** before sending (see the WebSocket reference): `size > 0`; `leverage` within the market's limits (`MarketConfig.initial_margin`); `marketId` present in `/api/v1/pub/context`; `price > 0` for limit orders and `price = 0` for market; `lb` no more than `head_block + market.order_ttl_blocks`; and the socket is open. Order-status updates arrive as OrdersUpdate (`mt: 24`) — orders carrying `r: true` should be removed from your open-orders view.

**SDK equivalent.** Build a `types::OrderRequest`, call `.prepare(&exchange)` to scale the decimal fields into the on-chain `OrderDesc`, and submit through the generated exchange binding:

```rust
use perpl_sdk::types::{OrderRequest, RequestType};
use fastnum::UD64;

// Open long: request_id becomes the on-chain client_order_id.
let req = OrderRequest::new(
    request_id, perp_id, RequestType::OpenLong,
    None,               // order_id (None -> 0)
    price, size,        // UD64 decimals — the SDK scales them for you
    None,               // expiry_block
    false, false, false,// post_only, fill_or_kill, immediate_or_cancel
    None,               // max_matches
    UD64::from(10u64),  // leverage (human units, e.g. 10x)
    None, None, 0u16,   // last_exec_block, amount, max_neg_pnl_collat_bps
);
let desc = req.prepare(&exchange);
// Cancel: RequestType::Cancel with the order_id set.

// Submit the prepared descriptor(s):
let receipt = instance
    .execOrders(vec![desc], /* revert_on_fail */ true)
    .send().await?.get_receipt().await?;
```

The SDK works in human-readable leverage (not hundredths) and scales prices/sizes via the perpetual's converters. Note the order-type numbering differs between the two layers: the WebSocket API's `t` field is **1-indexed** (`1` OpenLong … `7` Change, as in the table above), while the SDK's `RequestType` enum is **0-indexed** (`0` OpenLong … `6` Change) — the same operation, offset by one. See [SDK Concepts → Building and sending orders](sdk/concepts.md#building-and-sending-orders).

***

## Stream your own fills

After the trading socket authenticates, fills stream in as **FillsUpdate** (`mt: 25`). Each `Fill` carries the market (`mkt`), order (`oid`), order type (`t`), liquidity side (`l`: `1` Maker, `2` Taker), fill price (`p`, scaled), size (`s`, scaled), and fee/rebate (`f`).

```typescript
const { ws } = openTradingSocket((msg) => {
  if (msg.mt === 25) {              // FillsUpdate
    for (const fill of msg.d) {
      console.log({
        market: fill.mkt,
        orderId: fill.oid,
        side: fill.l === 1 ? 'maker' : 'taker',
        price: fill.p,              // scale by price_decimals
        size: fill.s,               // scale by size_decimals
        fee: fill.f,                // fees in micros (10^-6); negative = rebate
      });
    }
  }
});
```

For **historical** fills, page through the signed REST endpoint `GET /api/v1/trading/fills` (newest→oldest; `count` max 100; follow the `np` cursor):

```typescript
async function getAllFills() {
  const fills: any[] = [];
  let cursor: string | undefined;
  do {
    const params = new URLSearchParams({ count: '100' });
    if (cursor) params.set('page', cursor);
    const res = await signedFetch('GET', `/v1/trading/fills?${params.toString()}`);
    const data = await res.json();
    fills.push(...data.d);
    cursor = data.np;
  } while (cursor);
  return fills;
}
```

{% hint style="info" %}
The history endpoints do not support server-side filtering by market or date — filter client-side. See the [REST API reference](api/rest.md).
{% endhint %}

**SDK equivalent.** Layer the normalized trade stream `stream::trade` on top of `stream::raw`. It aggregates all maker fills belonging to one taker into a single `Trade` and normalizes the fixed-point values to decimals. Each `Trade` exposes `taker_account_id`, `taker_side`, `total_size()`, `avg_price()`, `perpetual_id`, `taker_fee`, and `maker_fills` (each with `maker_account_id`, `maker_order_id`, `size`, `price`, `fee`). See [SDK Concepts → the normalized trade stream](sdk/concepts.md#optional--the-normalized-trade-stream-streamtrade).

***

## Fetch positions and balances

Live positions and balances come from the trading WebSocket snapshots delivered right after authentication, then stay current via updates:

* **Balances** — WalletSnapshot (`mt: 19`) and AccountUpdate (`mt: 21`). The account object carries `b` (balance), `lb` (locked balance), and `lfr` (last forwarded request ID — also your `rq` seed).
* **Positions** — PositionsSnapshot (`mt: 26`) then PositionsUpdate (`mt: 27`).

```typescript
const positions = new Map<number, any>(); // key by position id

const { ws } = openTradingSocket((msg) => {
  switch (msg.mt) {
    case 19: // WalletSnapshot — balances live on each account entry
      for (const acc of (msg.as ?? [])) {
        console.log(`account ${acc.id}: balance=${acc.b} locked=${acc.lb} lfr=${acc.lfr}`);
      }
      break;
    case 21: // AccountUpdate — balance changed
      console.log(`account ${msg.id}: balance=${msg.b} locked=${msg.lb}`);
      break;
    case 26: // PositionsSnapshot — replace your view
    case 27: // PositionsUpdate — apply deltas
      for (const p of msg.d) positions.set(p.id, p);
      break;
  }
});
```

For **historical** positions and account events, use the signed REST endpoints `GET /api/v1/trading/position-history` and `GET /api/v1/trading/account-history` (same `{ d, np }` paginated shape as fills). Account events are typed by `et` (AccountEventType), e.g. `1` Deposit, `2` Withdrawal, `4` Settlement, `5` Liquidation, `8` Funding. Balances and amounts are decimal strings; fees are in micros (`10^-6`).

{% hint style="info" %}
There is no REST endpoint for _current_ positions — the live view is the WebSocket PositionsSnapshot/PositionsUpdate stream. REST `position-history` returns historical position records. The `Position` and `Account` type fields are documented in [Types & Errors](api/types-and-errors.md).
{% endhint %}

**SDK equivalent.** Snapshot the accounts you care about and read state directly off the cache:

```rust
let exchange = SnapshotBuilder::new(&chain, provider.clone())
    .with_accounts(vec![my_account.into()]) // fetch this account + its positions
    .build().await?;
```

`.with_accounts(...)` fetches the accounts' balances and positions; alternatively `.with_all_positions()` fetches every position (the two are mutually exclusive). See [SDK Concepts → the snapshot workflow](sdk/concepts.md#the-snapshot-then-stream-workflow).

***

## Subscribe to the order book

The order book is public — connect to the market-data WebSocket at `/ws/v1/market-data` (no authentication) and subscribe to `order-book@<market_id>` with a SubscriptionRequest (`mt: 5`). You receive an L2BookSnapshot (`mt: 15`) then incremental L2BookUpdate (`mt: 16`) messages. **L2** = level 2 (aggregated by price level). Each price level is `{ p, s, o }` (price, size, number of orders); a level with `o: 0` has been removed.

```typescript
class OrderBook {
  private ws!: WebSocket;
  private bids = new Map<number, { size: number; orders: number }>();
  private asks = new Map<number, { size: number; orders: number }>();

  constructor(private marketId: number) {}

  connect() {
    this.ws = new WebSocket(`${WS_URL}/ws/v1/market-data`);
    this.ws.onopen = () => {
      this.ws.send(JSON.stringify({
        mt: 5,  // SubscriptionRequest
        subs: [{ stream: `order-book@${this.marketId}`, subscribe: true }],
      }));
    };
    this.ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      if (msg.mt === 15) {            // snapshot — reset local state
        this.bids.clear(); this.asks.clear();
        this.apply(msg.bid, this.bids); this.apply(msg.ask, this.asks);
      } else if (msg.mt === 16) {     // update — apply deltas
        this.apply(msg.bid, this.bids); this.apply(msg.ask, this.asks);
      }
    };
  }

  private apply(levels: Array<{ p: number; s: number; o: number }>, book: Map<number, any>) {
    for (const lvl of levels ?? []) {
      if (lvl.o === 0) book.delete(lvl.p);              // o:0 -> remove level
      else book.set(lvl.p, { size: lvl.s, orders: lvl.o });
    }
  }

  bestBid() { return [...this.bids.keys()].sort((a, b) => b - a)[0]; }
  bestAsk() { return [...this.asks.keys()].sort((a, b) => a - b)[0]; }
}

const book = new OrderBook(MARKETS.BTC);
book.connect();
```

{% hint style="info" %}
Prices and sizes are scaled by the market's `price_decimals` / `size_decimals` (from `GET /api/v1/pub/context`). Other market-data streams use the same `mt: 5` subscribe shape: `trades@<market_id>`, `candles@<market_id>*<resolution>`, `market-state@<chain_id>`, `funding@<chain_id>`, `gas-stats@<chain_id>`, and `heartbeat@<chain_id>`. See the [WebSocket reference](api/websocket.md).
{% endhint %}

**SDK equivalent.** The SDK maintains a full **L3** (level 3, per-order) book: snapshot a perpetual, stream events, and read `perp.l3_book()` alongside `perp.mark_price()`, `perp.last_price()`, and `perp.oracle_price()`:

```rust
let mut exchange = SnapshotBuilder::new(&chain, provider.clone())
    .with_perpetuals(vec![16])
    .build().await?;

let from = exchange.instant();
let raw = stream::raw(&chain, provider.clone(), from, |d| tokio::time::sleep(d));
let mut raw = std::pin::pin!(raw);
while let Some(block) = raw.next().await {
    if exchange.apply_events(&block?)?.is_some() {
        if let Some(perp) = exchange.perpetuals().get(&16) {
            println!("mark = {}, book = {}", perp.mark_price(), perp.l3_book());
        }
    }
}
```

The command-line tool prints the same book without writing code: `perpl-cli show book --perp <id>`. See [SDK Concepts](sdk/concepts.md) and the [CLI reference](sdk/perpl-cli.md).

***

## Set a take-profit / stop-loss (trigger orders)

A **take-profit / stop-loss (TP/SL)** is a _trigger order_: a normal `OrderRequest` (`mt: 22`) that carries a trigger price and condition, and is not posted to the book until the market crosses that price. Trigger-specific fields:

| Field | Meaning                                                                                 |
| ----- | --------------------------------------------------------------------------------------- |
| `tp`  | Trigger price (scaled).                                                                 |
| `tpc` | Trigger condition: `1` GTELast, `2` LTELast, `3` GTEMark, `4` LTEMark.                  |
| `lp`  | Linked position ID — the trigger is cancelled when that position closes or inverts.     |
| `tr`  | Linked request ID — activate on the linked request's fill, cancel on its failure.       |
| `lb`  | **Must be `0` for trigger orders** (no expiry block; the server manages the lifecycle). |

For a **long** position you protect it with two reduce-only `CloseLong` (`t: 3`) triggers linked to the position via `lp`:

* **Stop-loss** — close when price falls to/below your stop → condition `LTELast` (`2`) or `LTEMark` (`4`).
* **Take-profit** — close when price rises to/above your target → condition `GTELast` (`1`) or `GTEMark` (`3`).

```typescript
// Stop-loss on a long position: close it if the MARK price falls to 90,000.
function stopLossLong(marketId: number, positionId: number, sizeScaled: number, stopScaled: number) {
  const order = {
    mt: 22,
    rq: ++ctx.nextRq,
    mkt: marketId,
    acc: ctx.accountId,
    t: 3,                 // CloseLong (reduce-only)
    p: 0,                 // fill at market once triggered
    s: sizeScaled,
    tp: stopScaled,       // trigger price (scaled)
    tpc: 4,               // LTEMark — fire when mark <= trigger
    lp: positionId,       // cancel automatically when the position closes
    fl: 4,                // IOC once triggered
    lv: 0,
    lb: 0,                // REQUIRED: trigger orders set lb = 0
  };
  ws.send(JSON.stringify(order));
  return order.rq;
}

// Take-profit on the same long: close it if the LAST price rises to 110,000.
function takeProfitLong(marketId: number, positionId: number, sizeScaled: number, targetScaled: number) {
  const order = {
    mt: 22,
    rq: ++ctx.nextRq,
    mkt: marketId,
    acc: ctx.accountId,
    t: 3,                 // CloseLong (reduce-only)
    p: 0,
    s: sizeScaled,
    tp: targetScaled,
    tpc: 1,               // GTELast — fire when last >= trigger
    lp: positionId,
    fl: 4,
    lv: 0,
    lb: 0,
  };
  ws.send(JSON.stringify(order));
  return order.rq;
}
```

For a **short** position, mirror the logic with `CloseShort` (`t: 4`): the stop-loss fires on a GTE condition (price rising against you) and the take-profit on an LTE condition (price falling in your favor).

{% hint style="info" %}
The number of resting trigger orders per account is capped by `max_account_trigger_orders` from the `ProtocolInstance` in `GET /api/v1/pub/context`. Trigger-order lifecycle, the `tr` request-linking behavior, and order-status transitions (`8` Untriggered → `9` Triggered) are described in the [WebSocket reference](api/websocket.md).
{% endhint %}

**SDK equivalent.** Not documented. The `OrderRequest::new` constructor in the current SDK sources exposes order-lifecycle fields (price, size, expiry, post-only / FOK / IOC, leverage, collateral amount) but **no** trigger-price / trigger-condition / linked-position fields, so trigger orders are placed over the WebSocket API shown above.

> **TODO(author):** Confirm whether `perpl-sdk` gained a trigger-order path (e.g. additional `OrderRequest` fields or a dedicated request type) in a version newer than the sources reviewed here; if so, document the SDK equivalent for TP/SL.

***

## Handling reconnects and rate limits

* **WebSocket sequence gaps** — track `sn`; seed it from the WalletSnapshot and require each Heartbeat (`mt: 100`) to be `sn + 1`. On a gap, reconnect and re-subscribe / re-authenticate to get a fresh snapshot.
* **WebSocket auth failure** — close code `3401`; reconnect and re-send a freshly signed `mt: 29` frame.
* **Rate limits** — approximate: REST public \~100 req/min, REST authenticated \~60 req/min, WS \~50 msg/sec per connection, \~5 connections per IP. On HTTP `429`, back off exponentially (1s / 2s / 4s).

Full reconnection and error-handling patterns are in the [WebSocket reference](api/websocket.md) and [Types & Errors](api/types-and-errors.md).

***

## See also

* [Quickstart](quickstart.md) — create a key and make your first signed call.
* [Networks & Configuration](networks-and-configuration.md) — endpoints, addresses, market IDs.
* [Authentication](api/authentication.md) — REST and WebSocket signing spec.
* [REST API reference](api/rest.md) — every endpoint, pagination, response shapes.
* [WebSocket reference](api/websocket.md) — message types, streams, order semantics.
* [Types & Errors](api/types-and-errors.md) — enums, `Position` / `Account` fields, reject reasons.
* [SDK Concepts](sdk/concepts.md) — the Rust `perpl-sdk` snapshot/stream/order model.

> **TODO(author):** Confirm the final GitBook navigation slugs for the cross-links above once the site structure is published (the API pages may live under a `direct-api/` section rather than `api/`); adjust the relative paths to match.
