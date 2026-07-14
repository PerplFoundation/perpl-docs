# WebSocket

Perpl streams real-time market data and account/trading data over two WebSocket (WSS) endpoints. Market data is public; trading and account data require authentication with an API key.

All frames are JSON. Prices and sizes are transmitted as **scaled integers** — divide by the market's `price_decimals` / `size_decimals` (from `MarketConfig`) to recover human-readable values. Leverage is in hundredths (`1000` = 10x).

## Endpoints

| Endpoint             | Purpose                  | Authentication     |
| -------------------- | ------------------------ | ------------------ |
| `/ws/v1/market-data` | Public market data       | None               |
| `/ws/v1/trading`     | Trading and account data | Required (API key) |

Base URL comes from the `PERPL_WS_URL` environment variable. Unlike the REST base URL, the **WebSocket URL has no `/api` prefix**.

| Network           | WebSocket base URL        | Chain ID |
| ----------------- | ------------------------- | -------- |
| Mainnet (default) | `wss://app.perpl.xyz`     | `143`    |
| Testnet           | `wss://testnet.perpl.xyz` | `10143`  |

Full URLs:

* Market Data: `${PERPL_WS_URL}/ws/v1/market-data`
* Trading: `${PERPL_WS_URL}/ws/v1/trading`

{% hint style="info" %}
Approximate rate limits are \~50 messages/second per connection and \~5 connections per IP (market-data and trading combined). Monitor for disconnects and back off if you are throttled.
{% endhint %}

## Message Format

Every frame carries a common header. The message type (`mt`) determines the rest of the frame's shape.

```typescript
interface MessageHeader {
  mt: number;       // Message type (see table below)
  sid?: number;     // Subscription ID
  sn?: number;      // Sequence number
  cid?: number;     // Correlation ID
  ses?: string;     // Session ID
}
```

### Message Types

| Value | Name                 | Direction       |
| ----- | -------------------- | --------------- |
| 1     | Ping                 | Client → Server |
| 2     | Pong                 | Server → Client |
| 3     | StatusResponse       | Server → Client |
| 5     | SubscriptionRequest  | Client → Server |
| 6     | SubscriptionResponse | Server → Client |
| 7     | GasPriceUpdate       | Server → Client |
| 8     | MarketConfigUpdate   | Server → Client |
| 9     | MarketStateUpdate    | Server → Client |
| 10    | MarketFundingUpdate  | Server → Client |
| 11    | CandlesSnapshot      | Server → Client |
| 12    | CandlesUpdate        | Server → Client |
| 15    | L2BookSnapshot       | Server → Client |
| 16    | L2BookUpdate         | Server → Client |
| 17    | TradesSnapshot       | Server → Client |
| 18    | TradesUpdate         | Server → Client |
| 19    | WalletSnapshot       | Server → Client |
| 20    | WalletUpdate         | Server → Client |
| 21    | AccountUpdate        | Server → Client |
| 22    | OrderRequest         | Client → Server |
| 23    | OrdersSnapshot       | Server → Client |
| 24    | OrdersUpdate         | Server → Client |
| 25    | FillsUpdate          | Server → Client |
| 26    | PositionsSnapshot    | Server → Client |
| 27    | PositionsUpdate      | Server → Client |
| 28    | AccountStatsUpdate   | Server → Client |
| 29    | ApiKeySignIn         | Client → Server |
| 100   | Heartbeat            | Server → Client |

> **Note:** Message types 4, 13, and 14 are reserved and not currently defined.

***

## Market Data WebSocket

The market-data endpoint requires no authentication. Open the connection, then subscribe to one or more streams.

### Connecting

```typescript
const WS_URL = process.env.PERPL_WS_URL || 'wss://app.perpl.xyz';
const ws = new WebSocket(`${WS_URL}/ws/v1/market-data`);
```

### Available Streams

Streams are identified by a string of the form `<name>@<key>`. Chain-scoped streams take the chain ID as the key; market-scoped streams take a market ID.

| Stream        | Format                             | Description                                |
| ------------- | ---------------------------------- | ------------------------------------------ |
| heartbeat     | `heartbeat@<chain_id>`             | Block sync heartbeat                       |
| gas-stats     | `gas-stats@<chain_id>`             | Gas price updates                          |
| market-config | `market-config@<chain_id>`         | Market configuration                       |
| market-state  | `market-state@<chain_id>`          | Prices, volume, open interest (OI)         |
| funding       | `funding@<chain_id>`               | Funding rate updates                       |
| candles       | `candles@<market_id>*<resolution>` | OHLCV (open/high/low/close/volume) candles |
| order-book    | `order-book@<market_id>`           | L2 order book                              |
| trades        | `trades@<market_id>`               | Recent trades                              |

**Chain ID**: from `PERPL_CHAIN_ID` (default `143`, Monad Mainnet; `10143` on testnet).

**Market IDs** (mainnet): BTC=`1`, MON=`10`, ETH=`20`, SOL=`31`, HYPE=`40`, ZEC=`50`. (testnet): BTC=`16`, ETH=`32`, SOL=`48`, MON=`64`, ZEC=`256`.

**Candle resolutions** (seconds): `60`, `300`, `900`, `1800`, `3600`, `7200`, `14400`, `28800`, `43200`, `86400`.

### Subscribing

Send a `SubscriptionRequest` (`mt: 5`) with a `subs` array. Each entry names a `stream` and sets `subscribe: true` to subscribe or `subscribe: false` to unsubscribe.

```typescript
// Subscribe to streams (mainnet, chain 143)
ws.send(JSON.stringify({
  mt: 5,  // SubscriptionRequest
  subs: [
    { stream: 'heartbeat@143', subscribe: true },
    { stream: 'order-book@1', subscribe: true },     // BTC order book
    { stream: 'trades@1', subscribe: true },         // BTC trades
    { stream: 'candles@1*3600', subscribe: true }    // BTC 1h candles
  ]
}));
```

### Unsubscribing

Send the same `SubscriptionRequest` frame with `subscribe: false`:

```typescript
ws.send(JSON.stringify({
  mt: 5,
  subs: [
    { stream: 'trades@1', subscribe: false }
  ]
}));
```

### Subscription Response

The server replies with a `SubscriptionResponse` (`mt: 6`). Match the returned `sid` (subscription ID) against the `sid` on later update frames to route them to the right handler. `status.code === 0` means the subscription succeeded.

```typescript
interface SubscriptionResponse {
  mt: 6;
  subs: Array<{
    stream: string;
    sid?: number;      // Subscription ID (use to match updates)
    status?: {
      code: number;    // 0 = success
      error?: string;
    };
  }>;
}
```

### Order Book

**Snapshot** (`mt: 15`) — the full L2 (aggregated-by-price) book at a block:

```typescript
interface L2Book {
  mt: 15;
  sid: number;
  at: BlockTimestamp;
  bid: L2PriceLevel[];  // Bids (best to worst)
  ask: L2PriceLevel[];  // Asks (best to worst)
}

interface L2PriceLevel {
  p: number;  // Price (scaled by price_decimals)
  s: number;  // Size (scaled by size_decimals)
  o: number;  // Number of orders at this level
}
```

**Update** (`mt: 16`) — same structure, carrying only changed levels. A level with `o: 0` should be removed from your local book.

### Trades

**Snapshot** (`mt: 17`):

```typescript
interface TradeSeries {
  mt: 17;
  sid: number;
  d: Trade[];
}

interface Trade {
  at: BlockTxLogTimestamp;
  p: number;       // Price (scaled)
  s: number;       // Size (scaled)
  sd: TradeSide;   // 1=Buy, 2=Sell
}
```

**Update** (`mt: 18`) — same structure, containing new trades.

### Candles

**Snapshot** (`mt: 11`):

```typescript
interface CandleSeries {
  mt: 11;
  sid: number;
  at: BlockTimestamp;
  r: number;     // Resolution (seconds)
  d: Candle[];   // Candles (oldest to newest)
}
```

**Update** (`mt: 12`) — contains up to 2 candles: the previous (now closed) candle and the current (still-updating) candle.

> **TODO(author):** the `Candle` object's field layout is not defined in the source. Document its fields (e.g. open/high/low/close/volume) once confirmed.

### Market State (`mt: 9`)

Delivered on the `market-state@<chain_id>` stream. `d` maps each market ID to its current state.

```typescript
interface MarketStateUpdate {
  mt: 9;
  d: Record<MarketID, MarketState | undefined>;
}

interface MarketState {
  at: BlockTimestamp;
  orl: number;   // Oracle price
  mrk: number;   // Mark price
  lst: number;   // Last price
  mid: number;   // Mid price
  bid: number;   // Best bid
  ask: number;   // Best ask
  prv: number;   // Price 24h ago
  dv: number;    // Daily volume (size)
  dva: string;   // Daily volume (amount)
  oi: number;    // Open interest
  tvl: string;   // Total value locked
}
```

### Heartbeat (`mt: 100`)

The `heartbeat@<chain_id>` stream emits a continuously-increasing sequence number and the latest head block. Track `sn` to detect dropped messages.

```typescript
interface Heartbeat {
  mt: 100;
  sn: number;  // Sequence number (strictly +1 from previous)
  h: number;   // Latest head block number
}
```

> **TODO(author):** the `GasPriceUpdate` (`mt: 7`), `MarketConfigUpdate` (`mt: 8`), and `MarketFundingUpdate` (`mt: 10`) payload shapes are not defined in the source. Document their fields once confirmed.

***

## Trading WebSocket

The trading endpoint delivers your wallet, order, position, and account data and accepts order requests. It requires authentication with an API key.

### Authenticating (`mt: 29`)

API keys are Ed25519 (Edwards-curve Digital Signature Algorithm) key pairs. Create one in the web UI (`app.perpl.xyz/apikeys` for mainnet, `testnet.perpl.xyz/apikeys` for testnet) or programmatically (see [Authentication](authentication.md)). Placing orders requires a `trade`-scoped key — a `read`-scoped key still receives snapshots and updates, but its `OrderRequest` frames are rejected with `403`.

Send an `ApiKeySignIn` frame as the **first** message after the socket opens. The Ed25519 signature covers the WS canonical string — four fields joined by `\n` (newline):

```
<chain_id>
trading-ws-signin      literal action tag
<timestamp_ms>         unix epoch milliseconds, decimal string
<nonce>                client-random, base64url (no padding)
```

Frame shape:

```typescript
{
  mt: 29,               // ApiKeySignIn
  chain_id: number,
  api_key: string,      // X-API-Key token from enrollment
  timestamp: string,    // unix ms, decimal
  nonce: string,        // client-random, base64url (no padding)
  signature: string,    // base64url(ed25519 signature over the canonical string)
}
```

```typescript
import { randomBytes } from 'crypto';
import * as ed from '@noble/ed25519';

const WS_URL = process.env.PERPL_WS_URL || 'wss://app.perpl.xyz';
const CHAIN_ID = Number(process.env.PERPL_CHAIN_ID) || 143;

const ws = new WebSocket(`${WS_URL}/ws/v1/trading`);

ws.onopen = async () => {
  const timestamp = Date.now().toString();
  const nonce = randomBytes(16).toString('base64url');
  const canonical = [CHAIN_ID, 'trading-ws-signin', timestamp, nonce].join('\n');
  const sig = await ed.signAsync(Buffer.from(canonical), privateKey);

  // Must authenticate immediately, as the first frame
  ws.send(JSON.stringify({
    mt: 29,             // ApiKeySignIn
    chain_id: CHAIN_ID,
    api_key: API_KEY,   // X-API-Key token from enrollment
    timestamp,
    nonce,
    signature: Buffer.from(sig).toString('base64url'),
  }));
};
```

{% hint style="info" %}
The signature timestamp must be within ±30 seconds of server time, and each nonce is single-use within the validity window. Generate a fresh `timestamp` and `nonce` for every sign-in — including on every reconnect.
{% endhint %}

### Initial Snapshots

After successful authentication, the server pushes three snapshots:

1. **WalletSnapshot** (`mt: 19`) — wallet and account balances.
2. **OrdersSnapshot** (`mt: 23`) — open orders.
3. **PositionsSnapshot** (`mt: 26`) — open positions.

The **WalletSnapshot** carries a sequence number (`sn` in the message header) that seeds sequence tracking. Store it and validate every subsequent heartbeat against it (see [Heartbeat](websocket.md#heartbeat-trading)).

### Placing Orders (`mt: 22`)

```typescript
interface OrderRequest {
  mt: 22;
  rq: number;          // Request ID (idempotency key; strictly increasing)
  mkt: number;         // Market ID
  acc: number;         // Account ID
  oid?: number;        // Order ID (for modify/cancel)
  t: OrderType;        // Order type (see table)
  p?: number;          // Limit price, scaled (0 for market)
  s: number;           // Size (scaled)
  a?: string;          // Amount (for collateral increase; decimal string)
  ms?: number;         // Maximum market order price slippage, bps
  tif?: number;        // Time-in-force (also defined); order expiry/validity is governed by the lb field below
  fl: OrderFlags;      // Flags: GoodTillCancel (GTC), PostOnly, FillOrKill (FOK), ImmediateOrCancel (IOC)
  tp?: number;         // Trigger price (stop / take-profit orders)
  tpc?: number;        // Trigger condition (1=GTLast, 2=LTELast, 3=GTEMark, 4=LTEMark)
  tr?: number;         // Linked trigger request ID
  lp?: number;         // Linked position ID
  lv: number;          // Leverage (hundredths, e.g. 1000 = 10x)
  lb: number;          // Last execution block
}
```

#### Idempotency and Request IDs (`rq`)

`rq` is an idempotency key scoped per account. The server guarantees **at-most-once** execution per `rq` — sending the same `rq` more than once yields a single execution. It is the API equivalent of a client order ID on centralized exchanges (it applies only to orders sent via the API, not to direct on-chain transactions).

`rq` must be **strictly increasing**. The server tracks the last processed value as `lfr` on the Account object (present in WalletSnapshot `mt: 19` and AccountUpdate `mt: 21`).

1. On connect, seed a local counter from `account.lfr`.
2. For each order: `rq = max(localCounter, account.lfr) + 1`.

Submitting `rq <= lfr` fails with `sr: 32` (`OrderDescIdTooLow`).

> **Note:** For smart-contract / SDK users placing non-API orders, `rq` may be set to any value to identify the order and need not be unique.

#### Retries and Deduplication

The client is responsible for retries. Multiple status updates can arrive for a single `rq`; deduplicate them.

| Scenario                                                              | Action                                                             |
| --------------------------------------------------------------------- | ------------------------------------------------------------------ |
| No status received yet, `lb` not expired                              | Retry with the **same** `rq`                                       |
| `sr: 32` (`OrderDescIdTooLow`) received                               | Retry **once** with a new `rq` (common with multiple clients/tabs) |
| Head block ≥ `lb`, no status received, no reconnections since posting | Retry with a **new** `rq`                                          |

Deduplication rules:

* The first non-failure status (`st` in `2, 3, 4, 5, 8, 9, 10`) is definitive — ignore everything after it, including later failures.
* If only failures (`st: 7`) arrive, process the first one only.
* After retrying with a new `rq`, ignore late failures from the old `rq`.

#### Trigger Orders

* Trigger orders must set `lb: 0` (no expiry block). The server manages their lifecycle from the trigger condition.
* `tp` + `tpc`: the order is not posted until the market last price crosses the trigger price per the condition (GTE = greater-than-or-equal, LTE = less-than-or-equal).
* `tr`: links this trigger to another request. When the linked request trades, the trigger activates; when it fails, the trigger is cancelled. If the linked request places an order, the trigger activates when that order fills and cancels when it is cancelled.
* `lp`: links the trigger to a position. The trigger is cancelled when the position is closed or inverted.

#### Order Types (`t`)

| Value | Name                       |
| ----- | -------------------------- |
| 1     | OpenLong                   |
| 2     | OpenShort                  |
| 3     | CloseLong                  |
| 4     | CloseShort                 |
| 5     | Cancel                     |
| 6     | IncreasePositionCollateral |
| 7     | Change                     |

#### Order Flags (`fl`)

| Value | Name                    |
| ----- | ----------------------- |
| 0     | GoodTillCancel (GTC)    |
| 1     | PostOnly                |
| 2     | FillOrKill (FOK)        |
| 4     | ImmediateOrCancel (IOC) |

#### Example — Open Long

```typescript
ws.send(JSON.stringify({
  mt: 22,
  rq: nextRequestId(),         // Strictly increasing; seeded from account.lfr
  mkt: 1,                      // BTC market (mainnet)
  acc: accountId,              // Your account ID
  t: 1,                        // OpenLong
  p: 95000 * 10,               // Price $95,000 (BTC mainnet price_decimals = 1)
  s: 10000,                    // 0.1 BTC (size_decimals = 5)
  fl: 0,                       // GTC
  lv: 1000,                    // 10x leverage
  lb: currentBlock + 100       // Valid for 100 blocks
}));
```

#### Example — Cancel Order

```typescript
ws.send(JSON.stringify({
  mt: 22,
  rq: nextRequestId(),
  mkt: 1,
  acc: accountId,
  oid: orderIdToCancel,
  t: 5,  // Cancel
  s: 0,
  fl: 0,
  lv: 0,
  lb: currentBlock + 100
}));
```

#### Input Validation (recommended for production)

* `size > 0` — reject zero or negative sizes.
* `leverage` within market limits — check `MarketConfig.initial_margin` (e.g. `1000` = 10% = max 10x).
* `marketId` is valid — verify against `/api/v1/pub/context` markets.
* `price > 0` for limit orders; `price = 0` for market (IOC) orders.
* `lb` should not exceed `head_block_number + market.order_ttl_blocks`.
* The WebSocket is connected — check `ws.readyState === WebSocket.OPEN`.

### Order Updates (`mt: 24`)

```typescript
interface WalletOrders {
  mt: 24;
  at: BlockTimestamp;
  d: Order[];
}
```

Orders with `r: true` should be removed from your open-orders view. Order-level status is carried in `st` (OrderStatus) and reject reasons in `sr` (OrderStatusReason — see [Order Reject Reasons](websocket.md#order-reject-reasons)).

**OrderStatus (`st`)**:

| Value | Name            |
| ----- | --------------- |
| 1     | Pending         |
| 2     | Open            |
| 3     | PartiallyFilled |
| 4     | Filled          |
| 5     | Canceled        |
| 6     | Expired         |
| 7     | Failed          |
| 8     | Untriggered     |
| 9     | Triggered       |
| 10    | Executed        |

### Fill Updates (`mt: 25`)

```typescript
interface WalletFills {
  mt: 25;
  at: BlockTimestamp;
  d: Fill[];
}
```

Each fill carries a `LiquiditySide` (1 = Maker, 2 = Taker).

### Position Updates (`mt: 27`)

```typescript
interface WalletPositions {
  mt: 27;
  at: BlockTimestamp;
  d: Position[];
}
```

Positions carry a `PositionType` (1 = Long, 2 = Short).

### Account Updates (`mt: 21`)

```typescript
interface Account {
  mt: 21;
  in: number;       // Instance ID
  id: number;       // Account ID
  fr: boolean;      // Is frozen
  fw: boolean;      // Allows forwarding
  lfr: number;      // Last forwarded request ID (use to seed `rq` generation)
  b: string;        // Balance (decimal string)
  lb: string;       // Locked balance (decimal string)
  h?: AccountEvent[];  // Recent events
}
```

`AccountEvent` entries carry an `AccountEventType`:

| Value | Name                        |
| ----- | --------------------------- |
| 1     | Deposit                     |
| 2     | Withdrawal                  |
| 3     | IncreasePositionCollateral  |
| 4     | Settlement                  |
| 5     | Liquidation                 |
| 6     | TransferToProtocol          |
| 7     | TransferFromProtocol        |
| 8     | Funding                     |
| 9     | Deleveraging                |
| 10    | Unwinding                   |
| 11    | PositionCollateralDecreased |
| 12    | LastForwardedDescIdReset    |

### Account Stats (`mt: 28`)

`AccountStatsUpdate` (`mt: 28`) carries per-account trading statistics. The same stats are also delivered inside the **WalletSnapshot** (`mt: 19`) via the wallet's `sts?` field.

```typescript
interface AccountStatsUpdate {
  mt: 28;
  // AccountStats fields
}
```

> **TODO(author):** the `AccountStats` field layout is defined in the shared types reference (`types-and-errors.md#accountstats`), not the WebSocket source. Cross-link or inline once available.

### Heartbeat (Trading) <a href="#heartbeat-trading" id="heartbeat-trading"></a>

On the trading WebSocket, sequence tracking is initialized from the WalletSnapshot rather than the heartbeat stream:

1. Initialize `lastSn` from the `sn` field of the **WalletSnapshot** (`mt: 19`) received after authentication.
2. Each subsequent heartbeat must satisfy `sn === previousSn + 1`.
3. On a sequence gap (missed heartbeat), **force reconnect** — the gap means messages may have been lost.

```typescript
let lastSn: number | undefined;

// On WalletSnapshot (mt: 19)
lastSn = walletMessage.sn;

// On Heartbeat (mt: 100)
if (lastSn != null && heartbeat.sn !== lastSn + 1) {
  // Sequence gap detected — reconnect to get fresh state
  ws.close();
  reconnect();
  return;
}
lastSn = heartbeat.sn;
```

### Keep-Alive

Send a Ping (`mt: 1`) about every 30 seconds to keep the connection open:

```typescript
setInterval(() => {
  ws.send(JSON.stringify({
    mt: 1,  // Ping
    t: Date.now()
  }));
}, 30000);
```

The server replies with a Pong (`mt: 2`).

***

## Error Handling and Reconnection

### Close Code 3401 — Authentication Failure

Close code **3401** means authentication failed. Reconnect and send a fresh, freshly-signed `ApiKeySignIn` frame (new `timestamp` and `nonce`) as the first message.

```typescript
function handleClose(event) {
  if (event.code === 3401) {
    // Auth failed — reconnect; the onopen handler re-sends a freshly
    // signed ApiKeySignIn frame (new timestamp + nonce) as the first message.
    reconnect();
  } else {
    // Normal close — reconnect with backoff (applied by reconnect()).
    reconnect();
  }
}
```

### Reconnection Strategy

Reconnect with exponential backoff. On every reconnect, re-authenticate as the first frame and re-seed sequence tracking from the new WalletSnapshot.

```typescript
const RETRY_DELAYS = [1000, 2000, 4000, 8000, 16000, 32000, 60000];
let retryCount = 0;
let ws: WebSocket;

// Open the socket and wire up the handlers. Called again by reconnect().
function connect() {
  ws = new WebSocket(`${WS_URL}/ws/v1/trading`);

  ws.onopen = async () => {
    // Authenticate immediately: the first frame is a signed ApiKeySignIn (mt: 29).
    const timestamp = Date.now().toString();
    const nonce = randomBytes(16).toString('base64url');
    const canonical = [CHAIN_ID, 'trading-ws-signin', timestamp, nonce].join('\n');
    const sig = await ed.signAsync(Buffer.from(canonical), privateKey);

    ws.send(JSON.stringify({
      mt: 29,             // ApiKeySignIn
      chain_id: CHAIN_ID,
      api_key: API_KEY,   // X-API-Key token from enrollment
      timestamp,
      nonce,
      signature: Buffer.from(sig).toString('base64url'),
    }));

    onConnectSuccess();
  };

  ws.onmessage = (event) => { /* handle snapshots/updates */ };
  ws.onclose = handleClose;  // the onclose handler shown above
}

function reconnect() {
  const delay = RETRY_DELAYS[Math.min(retryCount, RETRY_DELAYS.length - 1)];
  setTimeout(() => {
    retryCount++;
    connect();
  }, delay);
}

function onConnectSuccess() {
  retryCount = 0;
}

connect();
```

### Order Reject Reasons

Order rejects and position status changes are delivered as `sr` (OrderStatusReason) on order updates. Common values:

| `sr` | Meaning                       |
| ---- | ----------------------------- |
| 1    | AmountExceedsAvailableBalance |
| 13   | CrossesBook                   |
| 14   | ExceedsLastExecutionBlock     |
| 15   | ForwardingReverted            |
| 32   | OrderDescIdTooLow             |
| 38   | OrderSizeExceedsAvailableSize |
| 53   | PerpetualInsolvent            |

> **Note:** The full `OrderStatusReason` enum spans values 0–68, and a `PositionStatusReason` enum (subset 13–22) is also defined. See the shared types reference for the complete lists. No JSON error-envelope schema is specified for WebSocket frames.

***

## Sequence Numbers

* The `heartbeat` and `gas-stats` streams have **continuous** sequence numbers.
* Other streams may have gaps (for example, when there is no activity).
* Track `sn` to detect missed messages.
* On a gap, resubscribe (market data) or force reconnect (trading) to obtain a fresh snapshot.
