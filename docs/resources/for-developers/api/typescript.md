# TypeScript

This guide shows how to talk to the Perpl API directly from TypeScript — no SDK. You will configure a client from environment variables, sign requests with an Ed25519 (Edwards-curve Digital Signature Algorithm) key, make authenticated REST (Representational State Transfer) calls, subscribe to a WebSocket stream, and place and cancel an order.

Every snippet below is copy-pasteable. Fill in an enrolled API key and a target network, and the code runs as-is.

## What you need first

Before writing code, make sure you have:

* **An enrolled API key** — an Ed25519 key pair. The server stores only the public key; the private key never leaves your machine, and every request is signed with it. There is no bearer token or session cookie. Create a key by connecting your wallet in the web UI (mainnet `https://app.perpl.xyz/apikeys`, testnet `https://testnet.perpl.xyz/apikeys`); the UI hands you the `X-API-Key` token and the private key. See [Authentication](file:///2362779/api/authentication.md) for the full model.
* **A key scope that matches your task** — a key carries a scope of `read`, `trade`, or both (`trade` implies `read`). A `read` key can fetch data and subscribe to streams but cannot place orders. **Withdrawals and transfers-out are never permitted via an API key, at any scope.**
* **An on-chain exchange account** — API authentication is separate from having an account to trade with. Authenticating a key only authorizes API access; trading also requires an on-chain account created with collateral via `createAccount(uint256)` on the Exchange contract. A signed request will succeed at the API layer but some calls return `404` until that account exists.

{% hint style="info" %}
Prices, sizes, and collateral amounts are **scaled integers**, and leverage is expressed **in hundredths** (`1000` = 10x). See [Scaling helpers](typescript.md#scaling-helpers) below. Never send a human-readable float where the API expects a scaled integer.
{% endhint %}

## Install dependencies

The only third-party dependency is `@noble/ed25519` for signing. Everything else (`fetch`, `WebSocket`, and the Node `crypto` module) is provided by the runtime.

```bash
npm install @noble/ed25519
```

{% hint style="info" %}
`fetch` and `WebSocket` are global in browsers and in Node 22+. On older Node versions, provide a `WebSocket` implementation (for example the `ws` package) before running the WebSocket snippets. `createHash` and `randomBytes` come from Node's built-in `crypto` module.
{% endhint %}

## Configure the client

All endpoints and chain settings are read from environment variables, so switching between mainnet and testnet is a matter of swapping values — no code changes. Create a `.env` (or export the variables in your shell):

**Mainnet:**

```bash
PERPL_API_URL=https://app.perpl.xyz/api
PERPL_WS_URL=wss://app.perpl.xyz
PERPL_CHAIN_ID=143
PERPL_RPC_URL=https://rpc.monad.xyz
PERPL_EXCHANGE_ADDRESS=0x34B6552d57a35a1D042CcAe1951BD1C370112a6F
PERPL_COLLATERAL_TOKEN=0x00000000eFE302BEAA2b3e6e1b18d08D69a9012a

# The enrolled key:
PERPL_API_KEY=<the X-API-Key token>
PERPL_API_KEY_SECRET=<hex of the 32-byte Ed25519 private key>
```

**Testnet:**

```bash
PERPL_API_URL=https://testnet.perpl.xyz/api
PERPL_WS_URL=wss://testnet.perpl.xyz
PERPL_CHAIN_ID=10143
PERPL_RPC_URL=https://testnet-rpc.monad.xyz
PERPL_EXCHANGE_ADDRESS=0x1964C32f0bE608E7D29302AFF5E61268E72080cc
PERPL_COLLATERAL_TOKEN=0xa9012a055bd4e0eDfF8Ce09f960291C09D5322dC
```

{% hint style="info" %}
The REST base URL includes the `/api` suffix; the WebSocket base URL does **not**. Connect WebSockets to `${PERPL_WS_URL}/ws/v1/market-data` and `${PERPL_WS_URL}/ws/v1/trading`.
{% endhint %}

Load the configuration into a small module you can import everywhere:

```typescript
// config.ts
export const API_URL = process.env.PERPL_API_URL || 'https://app.perpl.xyz/api';
export const WS_URL  = process.env.PERPL_WS_URL  || 'wss://app.perpl.xyz'; // no /api prefix
export const CHAIN_ID = Number(process.env.PERPL_CHAIN_ID) || 143;

// The enrolled key (see Authentication):
//   PERPL_API_KEY        — the X-API-Key token
//   PERPL_API_KEY_SECRET — hex of the 32-byte Ed25519 private key
export const API_KEY = process.env.PERPL_API_KEY!;
export const privateKey = Buffer.from(
  (process.env.PERPL_API_KEY_SECRET ?? '').replace(/^0x/, ''),
  'hex',
);

// Market IDs for mainnet.
// Testnet uses different IDs (BTC=16, ETH=32, SOL=48, MON=64, ZEC=256).
export const MARKETS = {
  BTC: 1,
  MON: 10,
  ETH: 20,
  SOL: 31,
  HYPE: 40,
  ZEC: 50,
} as const;
```

{% hint style="info" %}
Market IDs are network-specific and can change as markets are added or delisted. Fetch the authoritative list at runtime from `GET /api/v1/pub/context` (see [Fetch market configuration](typescript.md#fetch-market-configuration)) rather than relying on the hard-coded table above.
{% endhint %}

## Sign REST requests

Every REST call is signed. The signature covers a **canonical string** — six fields joined by `\n` (newline):

| # | Field                | Value                                                                   |
| - | -------------------- | ----------------------------------------------------------------------- |
| 1 | `<chain_id>`         | e.g. `143`                                                              |
| 2 | `<HTTP_METHOD>`      | `GET`, `POST`, …                                                        |
| 3 | `<request-target>`   | path + query string exactly as sent, e.g. `/v1/trading/fills?count=100` |
| 4 | `<timestamp_ms>`     | Unix epoch milliseconds, decimal                                        |
| 5 | `<nonce>`            | client-random, base64url, no padding                                    |
| 6 | `<sha256(body) hex>` | hex SHA-256 of the raw body (`""` for an empty body)                    |

The signature is `base64url(ed25519_sign(privateKey, canonical))` (base64url, no padding), sent alongside three more headers:

| Header            | Value                                           |
| ----------------- | ----------------------------------------------- |
| `X-API-Key`       | the opaque token from enrollment                |
| `X-API-Timestamp` | the `timestamp_ms` used in the canonical string |
| `X-API-Nonce`     | the `nonce` used in the canonical string        |
| `X-API-Signature` | `base64url(ed25519 signature)`                  |

The `signedFetch` helper builds the canonical string, signs it, and attaches all four headers:

```typescript
// signedFetch.ts
import { createHash, randomBytes } from 'crypto';
import * as ed from '@noble/ed25519';
import { API_URL, CHAIN_ID, API_KEY, privateKey } from './config';

// `target` is the path + query string exactly as sent,
// e.g. /v1/trading/fills?count=100
export async function signedFetch(method: string, target: string, body = '') {
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

// Example: read your most recent fill
const res = await signedFetch('GET', '/v1/trading/fills?count=1');
console.log(await res.json());
```

{% hint style="info" %}
The `request-target` must match byte-for-byte what the server receives. Include the query string (`?count=100&page=...`) exactly as it appears in the URL you fetch — build it once and reuse the same string for both the signature and the request.
{% endhint %}

**Signature validity rules:**

* **Timestamp window** — `X-API-Timestamp` must be within **±30 seconds** of server time. Keep the client clock in sync.
* **Nonce** — single-use within the validity window. Generate a fresh random nonce per request; replays are rejected.
* **Expiry / IP** — requests are rejected once the key is past its `expires_at`, or when an `ip_cidrs` allow-list is set (max 4 CIDRs) and the caller's IP is not covered.

## Authenticated REST calls

### Fetch market configuration

The public `context` endpoint needs no authentication and returns the live chain, token, and market configuration. Read it once at startup to discover the current market set and the scaling decimals for each market.

```typescript
import { API_URL } from './config';

async function getContext() {
  const res = await fetch(`${API_URL}/v1/pub/context`);
  const context = await res.json();

  return {
    markets:   new Map(context.markets.map((m: any) => [m.id, m])),
    tokens:    new Map(context.tokens.map((t: any) => [t.id, t])),
    instances: new Map(context.instances.map((i: any) => [i.id, i])),
  };
}
```

{% hint style="info" %}
`getContext()` uses a plain `fetch` because `/v1/pub/context` is public. A signed request personalizes the response but is optional. Each `MarketConfig` carries `price_decimals` and `size_decimals` used for scaling (see [Scaling helpers](typescript.md#scaling-helpers)).
{% endhint %}

### Fetch trading history with pagination

Trading-history endpoints (`fills`, `order-history`, `position-history`, `account-history`) require a signed request. They page with a cursor: pass `count` (default 50, **max 100**) and `page` (the cursor returned as `np` in the previous response). The response shape is `{ d: T[], np: string }`, where `d` is newest-to-oldest and `np` is the next cursor (absent or empty when there are no more pages).

```typescript
import { signedFetch } from './signedFetch';

async function getAllFills(): Promise<any[]> {
  const fills: any[] = [];
  let cursor: string | undefined;

  do {
    // Build the query string; the signature binds the full request target.
    const params = new URLSearchParams();
    if (cursor) params.set('page', cursor);
    params.set('count', '100');
    const target = `/v1/trading/fills?${params.toString()}`;

    const res = await signedFetch('GET', target);
    const data = await res.json();

    fills.push(...data.d);
    cursor = data.np;
  } while (cursor);

  return fills;
}
```

The same pattern works for position history — only the path and page size change:

```typescript
async function getPositionHistory(): Promise<any[]> {
  const positions: any[] = [];
  let cursor: string | undefined;

  do {
    const params = new URLSearchParams();
    if (cursor) params.set('page', cursor);
    params.set('count', '50');
    const target = `/v1/trading/position-history?${params.toString()}`;

    const res = await signedFetch('GET', target);
    const data = await res.json();

    positions.push(...data.d);
    cursor = data.np;
  } while (cursor);

  return positions;
}
```

{% hint style="info" %}
History endpoints do **not** support server-side filtering by market or date — filter the returned rows client-side.
{% endhint %}

### Fetch candles

Candle (OHLCV — open, high, low, close, volume) data is public and returns raw scaled prices. Divide by the market's price scale to get human-readable values. A single request returns at most **1024** candles.

```typescript
import { API_URL } from './config';

async function getCandles(marketId: number, resolution: number, hours = 24) {
  const to = Date.now();
  const from = to - hours * 60 * 60 * 1000;

  const res = await fetch(
    `${API_URL}/v1/market-data/${marketId}/candles/${resolution}/${from}-${to}`,
  );
  const data = await res.json();

  // Scale prices with the market's price_decimals from /pub/context.
  const ctx = await getContext();
  const market = ctx.markets.get(marketId);
  const priceScale = Math.pow(10, market.config.price_decimals);

  return data.d.map((c: any) => ({
    time:   c.t,
    open:   c.o / priceScale,
    high:   c.h / priceScale,
    low:    c.l / priceScale,
    close:  c.c / priceScale,
    volume: parseFloat(c.v),
    trades: c.n,
  }));
}

// 1-hour candles for the last 24 hours
const btcCandles = await getCandles(MARKETS.BTC, 3600, 24);
```

{% hint style="info" %}
`resolution` is in seconds. Supported values: `60`, `300`, `900`, `1800`, `3600`, `7200`, `14400`, `28800`, `43200`, `86400`.
{% endhint %}

## Subscribe to a market-data WebSocket

The market-data WebSocket (`/ws/v1/market-data`) requires no authentication. Subscribe with a `SubscriptionRequest` frame (`mt: 5`) listing one or more streams. This example maintains a live L2 (Level 2, aggregated) order book: apply the snapshot (`mt: 15`), then apply each incremental update (`mt: 16`). A level with zero orders (`o === 0`) has been removed.

```typescript
import { WS_URL } from './config';

class OrderBookClient {
  private ws!: WebSocket;
  private bids = new Map<number, { size: number; orders: number }>();
  private asks = new Map<number, { size: number; orders: number }>();

  constructor(private marketId: number) {}

  connect() {
    this.ws = new WebSocket(`${WS_URL}/ws/v1/market-data`);

    this.ws.onopen = () => {
      this.ws.send(JSON.stringify({
        mt: 5,
        subs: [{ stream: `order-book@${this.marketId}`, subscribe: true }],
      }));
    };

    this.ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);

      if (msg.mt === 15) {
        // L2BookSnapshot — reset the book
        this.bids.clear();
        this.asks.clear();
        this.applyLevels(msg.bid, this.bids);
        this.applyLevels(msg.ask, this.asks);
      } else if (msg.mt === 16) {
        // L2BookUpdate — apply the delta
        this.applyLevels(msg.bid, this.bids);
        this.applyLevels(msg.ask, this.asks);
      }
    };
  }

  private applyLevels(
    levels: Array<{ p: number; s: number; o: number }>,
    book: Map<number, { size: number; orders: number }>,
  ) {
    for (const level of levels) {
      if (level.o === 0) book.delete(level.p);
      else book.set(level.p, { size: level.s, orders: level.o });
    }
  }

  getBestBid() {
    return [...this.bids.keys()].sort((a, b) => b - a)[0];
  }
  getBestAsk() {
    return [...this.asks.keys()].sort((a, b) => a - b)[0];
  }

  disconnect() {
    this.ws?.close();
  }
}

const book = new OrderBookClient(MARKETS.BTC);
book.connect();
```

Other public streams follow the same `mt: 5` subscription shape — swap the `stream` value:

| Stream                             | Purpose                       |
| ---------------------------------- | ----------------------------- |
| `heartbeat@<chain_id>`             | Block-sync heartbeat          |
| `gas-stats@<chain_id>`             | Gas price                     |
| `market-config@<chain_id>`         | Market configuration          |
| `market-state@<chain_id>`          | Prices, volume, open interest |
| `funding@<chain_id>`               | Funding rate                  |
| `candles@<market_id>*<resolution>` | OHLCV                         |
| `order-book@<market_id>`           | L2 book                       |
| `trades@<market_id>`               | Recent trades                 |

## Place and cancel orders over the trading WebSocket

The trading WebSocket (`/ws/v1/trading`) is authenticated.

{% stepper %}
{% step %}
## Open the socket and sign in

Open the socket and send a signed `ApiKeySignIn` frame (`mt: 29`) as the **first** message. Its signature covers a four-field canonical string — `<chain_id>`, the literal tag `trading-ws-signin`, `<timestamp_ms>`, `<nonce>` — joined by `\n`.
{% endstep %}

{% step %}
## Receive the initial snapshots

Receive the initial snapshots: `WalletSnapshot` (`mt: 19`), `OrdersSnapshot` (`mt: 23`), `PositionsSnapshot` (`mt: 26`). Read your account ID from the wallet snapshot.
{% endstep %}

{% step %}
## Seed sequence tracking

Seed sequence tracking from the `WalletSnapshot` `sn` field. Every `Heartbeat` (`mt: 100`) must carry `sn + 1`; a gap means you missed a message — force a reconnect.
{% endstep %}

{% step %}
## Keep the socket alive

Keep the socket alive by sending a `Ping` (`mt: 1`) roughly every 30 seconds.
{% endstep %}

{% step %}
## Place orders

Place orders with `OrderRequest` frames (`mt: 22`).
{% endstep %}
{% endstepper %}

```typescript
import { randomBytes } from 'crypto';
import * as ed from '@noble/ed25519';
import { WS_URL, CHAIN_ID } from './config';

class TradingClient {
  private ws!: WebSocket;
  private requestId = Date.now();
  private accountId!: number;
  private currentBlock = 0;
  private lastSn?: number;
  private pingInterval?: ReturnType<typeof setInterval>;

  constructor(
    private privateKey: Uint8Array,
    private apiKey: string,
    private onUpdate: (type: string, data: any) => void,
  ) {}

  connect() {
    this.ws = new WebSocket(`${WS_URL}/ws/v1/trading`);

    this.ws.onopen = async () => {
      // Authenticate: signed ApiKeySignIn frame as the FIRST message.
      const timestamp = Date.now().toString();
      const nonce = randomBytes(16).toString('base64url');
      const canonical = [CHAIN_ID, 'trading-ws-signin', timestamp, nonce].join('\n');
      const sig = await ed.signAsync(Buffer.from(canonical), this.privateKey);

      this.ws.send(JSON.stringify({
        mt: 29, // ApiKeySignIn
        chain_id: CHAIN_ID,
        api_key: this.apiKey,
        timestamp,
        nonce,
        signature: Buffer.from(sig).toString('base64url'),
      }));
    };

    this.ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);

      switch (msg.mt) {
        case 19: // WalletSnapshot
          this.accountId = msg.as?.[0]?.id;
          this.lastSn = msg.sn; // seed sequence tracking
          this.onUpdate('wallet', msg);
          break;
        case 23: // OrdersSnapshot
          this.onUpdate('orders', msg.d);
          break;
        case 24: // OrdersUpdate
          this.onUpdate('orderUpdate', msg.d);
          break;
        case 25: // FillsUpdate
          this.onUpdate('fills', msg.d);
          break;
        case 26: // PositionsSnapshot
          this.onUpdate('positions', msg.d);
          break;
        case 27: // PositionsUpdate
          this.onUpdate('positionUpdate', msg.d);
          break;
        case 100: // Heartbeat — must be sn + 1
          if (this.lastSn != null && msg.sn !== this.lastSn + 1) {
            console.warn('Sequence gap, reconnecting...');
            this.disconnect();
            this.connect();
            return;
          }
          this.lastSn = msg.sn;
          this.currentBlock = msg.h;
          break;
      }
    };

    // Keep-alive Ping ~every 30 s.
    this.pingInterval = setInterval(() => {
      if (this.ws.readyState === WebSocket.OPEN) {
        this.ws.send(JSON.stringify({ mt: 1, t: Date.now() }));
      }
    }, 30_000);
  }

  private nextRequestId(): number {
    return ++this.requestId;
  }

  // Open a long. Pass price = null for a market order (immediate-or-cancel, IOC), a scaled price for a limit order (good-till-canceled, GTC).
  async openLong(marketId: number, size: number, price: number | null, leverage: number) {
    const order = {
      mt: 22,
      rq: this.nextRequestId(),
      mkt: marketId,
      acc: this.accountId,
      t: 1,                       // OpenLong
      p: price ?? 0,              // 0 = market
      s: size,                    // scaled size
      fl: price ? 0 : 4,          // 0 = GTC (limit), 4 = ImmediateOrCancel (market)
      lv: leverage * 100,         // leverage in hundredths
      lb: this.currentBlock + 100 // last valid block
    };
    this.ws.send(JSON.stringify(order));
    return order.rq;
  }

  async openShort(marketId: number, size: number, price: number | null, leverage: number) {
    const order = {
      mt: 22,
      rq: this.nextRequestId(),
      mkt: marketId,
      acc: this.accountId,
      t: 2,                       // OpenShort
      p: price ?? 0,
      s: size,
      fl: price ? 0 : 4,
      lv: leverage * 100,
      lb: this.currentBlock + 100,
    };
    this.ws.send(JSON.stringify(order));
    return order.rq;
  }

  async closePosition(
    marketId: number,
    positionId: number,
    size: number,
    isLong: boolean,
    price: number | null,
  ) {
    const order = {
      mt: 22,
      rq: this.nextRequestId(),
      mkt: marketId,
      acc: this.accountId,
      t: isLong ? 3 : 4,          // CloseLong or CloseShort
      p: price ?? 0,
      s: size,
      fl: price ? 0 : 4,
      lp: positionId,             // position to close
      lv: 0,
      lb: this.currentBlock + 100,
    };
    this.ws.send(JSON.stringify(order));
    return order.rq;
  }

  async cancelOrder(marketId: number, orderId: number) {
    const order = {
      mt: 22,
      rq: this.nextRequestId(),
      mkt: marketId,
      acc: this.accountId,
      oid: orderId,               // order to cancel
      t: 5,                       // Cancel
      s: 0,
      fl: 0,
      lv: 0,
      lb: this.currentBlock + 100,
    };
    this.ws.send(JSON.stringify(order));
    return order.rq;
  }

  disconnect() {
    if (this.pingInterval) {
      clearInterval(this.pingInterval);
      this.pingInterval = undefined;
    }
    this.ws?.close();
  }
}
```

Wire it up and place an order once the snapshots have arrived:

```typescript
import { privateKey, API_KEY, MARKETS } from './config';

const client = new TradingClient(privateKey, API_KEY, (type, data) => {
  console.log(type, data);
});
client.connect();

// Wait for the socket to open and the account snapshot to load, then trade.
setTimeout(async () => {
  // Open 0.1 BTC long at market price with 10x leverage.
  // Size is scaled (BTC mainnet size_decimals = 5, so 0.1 BTC -> 10000).
  const requestId = await client.openLong(MARKETS.BTC, 10000, null, 10);
  console.log('Order submitted:', requestId);

  // Later: cancel a resting order by its order id.
  // await client.cancelOrder(MARKETS.BTC, orderId);
}, 2000);
```

### Order request fields

`OrderRequest` (`mt: 22`) key fields:

| Field | Meaning                                                                                          |
| ----- | ------------------------------------------------------------------------------------------------ |
| `rq`  | Request ID — idempotency key, at-most-once per account, strictly increasing (see the note below) |
| `mkt` | Market ID                                                                                        |
| `acc` | Account ID (from `WalletSnapshot`)                                                               |
| `oid` | Order ID — required for `Cancel`                                                                 |
| `t`   | Order type (see enum below)                                                                      |
| `p`   | Price, scaled (`0` = market order)                                                               |
| `s`   | Size, scaled                                                                                     |
| `ms`  | Max market slippage, basis points (optional)                                                     |
| `lb`  | Last valid block — the order expires after this block; triggers must set `lb: 0`                 |
| `fl`  | Order flags (see enum below)                                                                     |
| `lp`  | Position ID (for closing / trigger fields)                                                       |
| `lv`  | Leverage in hundredths (`1000` = 10x)                                                            |

{% hint style="info" %}
`rq` is an idempotency key and must be strictly increasing per account. The client above seeds it from `Date.now()`, which is simple but not restart-safe. The robust seed is the account's last-forwarded request ID (`lfr`): compute `rq = max(localCounter, lfr) + 1`. An `rq` at or below the account's last value fails with order-reject reason `sr: 32` (`OrderDescIdTooLow`).
{% endhint %}

### Order enums

| Enum                      | Values                                                                                                                   |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **OrderType** (`t`)       | 1 OpenLong, 2 OpenShort, 3 CloseLong, 4 CloseShort, 5 Cancel, 6 IncreasePositionCollateral, 7 Change                     |
| **OrderFlags** (`fl`)     | 0 GTC (good-till-canceled), 1 PostOnly, 2 FillOrKill, 4 ImmediateOrCancel                                                |
| **TriggerPriceCondition** | 1 GTELast, 2 LTELast, 3 GTEMark, 4 LTEMark                                                                               |
| **OrderStatus** (`st`)    | 1 Pending, 2 Open, 3 PartiallyFilled, 4 Filled, 5 Canceled, 6 Expired, 7 Failed, 8 Untriggered, 9 Triggered, 10 Executed |

{% hint style="info" %}
A `read`-scoped key may connect and receive snapshots and updates over the trading WebSocket, but `OrderRequest` frames are rejected — use a `trade`-scoped key to place orders.
{% endhint %}

## Scaling helpers

Prices and sizes are integers scaled by the market's `price_decimals` / `size_decimals` (read from `MarketConfig` via `/pub/context`). Leverage is stored in hundredths. Convert at the client boundary so the rest of your code works in human units.

```typescript
// Price: scale by 10 ** price_decimals
function createPriceConverter(priceDecimals: number) {
  const scale = Math.pow(10, priceDecimals);
  return {
    toScaled:   (price: number) => Math.round(price * scale),
    fromScaled: (scaled: number) => scaled / scale,
  };
}

// BTC mainnet has 1 price decimal
const btcPrice = createPriceConverter(1);
btcPrice.toScaled(95000);    // 950000
btcPrice.fromScaled(950000); // 95000

// Size: scale by 10 ** size_decimals
function createSizeConverter(sizeDecimals: number) {
  const scale = Math.pow(10, sizeDecimals);
  return {
    toScaled:   (size: number) => Math.round(size * scale),
    fromScaled: (scaled: number) => scaled / scale,
  };
}

// BTC mainnet has 5 size decimals
const btcSize = createSizeConverter(5);
btcSize.toScaled(0.1);    // 10000
btcSize.fromScaled(10000); // 0.1

// Leverage is stored in hundredths
const leverageToHundredths = (lev: number) => lev * 100; // 10 -> 1000
const hundredthsToLeverage = (h: number) => h / 100;     // 1000 -> 10
```

{% hint style="info" %}
Collateral amounts settle in a 6-decimal token — divide a raw on-chain integer by `1_000_000` for a USD figure. Fees are expressed in `Micros` (10⁻⁶; a negative value is a rebate) and monetary `Amount` fields are decimal strings. See [Networks & Configuration](file:///2362779/getting-started/networks.md).
{% endhint %}

## Error handling and rate limits

### HTTP status codes

| Status | Meaning                                                                                 | Action                                                                     |
| ------ | --------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| 200    | Success                                                                                 | —                                                                          |
| 400    | Bad Request                                                                             | Check request shape and parameters                                         |
| 401    | Unauthorized — bad/stale signature, replayed nonce, revoked/expired key, IP not allowed | Re-sign with a fresh timestamp + nonce; check clock, key status, source IP |
| 403    | Forbidden — scope insufficient (e.g. `read` key placing an order)                       | Use a `trade`-scoped key                                                   |
| 404    | Not Found — including "no on-chain account yet"                                         | Create an exchange account, or check the path                              |
| 429    | Too Many Requests                                                                       | Back off and retry (see below)                                             |
| 500    | Internal Server Error                                                                   | Retry with backoff                                                         |

### Rate limits

Limits are approximate; watch for `HTTP 429` and back off.

| Type                                          | Limit                       |
| --------------------------------------------- | --------------------------- |
| REST public (`/v1/pub/*`, market data)        | \~100 req/min               |
| REST authenticated (profile, trading history) | \~60 req/min                |
| WebSocket messages                            | \~50 msg/sec per connection |
| WebSocket connections                         | \~5 per IP                  |

### Retrying and reconnecting

Retry `429`s with exponential backoff (1s / 2s / 4s …), and reconnect the WebSocket with a backoff schedule. A close code of **`3401`** means WebSocket authentication failed — reconnect and re-send a fresh signed `mt: 29` frame with a new timestamp and nonce.

```typescript
async function safeApiCall<T>(fn: () => Promise<T>): Promise<T> {
  try {
    return await fn();
  } catch (error: any) {
    if (error.response?.status === 429) {
      await new Promise((r) => setTimeout(r, 1000)); // wait and retry
      return safeApiCall(fn);
    }
    throw error;
  }
}

function handleWebSocketError(ws: WebSocket, onReconnect: () => void) {
  const RETRY_DELAYS = [1000, 2000, 4000, 8000, 16000, 32000, 60000];
  let retries = 0;

  ws.onclose = (event) => {
    if (event.code === 3401) {
      // Auth failed: reconnect and re-send a fresh signed mt:29 frame.
      console.error('WebSocket auth failed, reconnecting to re-sign in');
      onReconnect();
      return;
    }
    const delay = RETRY_DELAYS[Math.min(retries++, RETRY_DELAYS.length - 1)];
    console.log(`Reconnecting in ${delay}ms...`);
    setTimeout(onReconnect, delay);
  };

  ws.onerror = (error) => console.error('WebSocket error:', error);
}
```

{% hint style="info" %}
Order rejections do not arrive as HTTP errors — they come back on order updates as an `sr` (OrderStatusReason) code. Common values include 1 `AmountExceedsAvailableBalance`, 13 `CrossesBook`, 14 `ExceedsLastExecutionBlock`, 15 `ForwardingReverted`, 32 `OrderDescIdTooLow`, 38 `OrderSizeExceedsAvailableSize`, 53 `PerpetualInsolvent`. Inspect the `sr` field on `OrdersUpdate` (`mt: 24`) frames to see why an order did not rest or fill.
{% endhint %}

## Reference

### REST endpoints

| Method | Path                                                           | Auth             | Purpose                                            |
| ------ | -------------------------------------------------------------- | ---------------- | -------------------------------------------------- |
| GET    | `/api/v1/pub/context`                                          | Optional         | Chain, instances, tokens, markets config           |
| GET    | `/api/v1/market-data/:market_id/candles/:resolution/:from-:to` | None             | OHLCV candles (max 1024/req)                       |
| GET    | `/api/v1/profile/announcements`                                | Optional         | Announcements                                      |
| GET    | `/api/v1/profile/ref-code`                                     | API key          | Your referral code                                 |
| GET    | `/api/v1/trading/account-history`                              | API key          | Account events (deposits, settlements, funding, …) |
| GET    | `/api/v1/trading/fills`                                        | API key          | Order fill history                                 |
| GET    | `/api/v1/trading/order-history`                                | API key          | Historical order events                            |
| GET    | `/api/v1/trading/position-history`                             | API key          | Position history                                   |
| POST   | `/api/v1/api-key/payload`                                      | Wallet signature | Get EIP-712 payload to sign for enrollment         |
| POST   | `/api/v1/api-key/enroll`                                       | Wallet signature | Enroll a key, receive the `X-API-Key` token        |

### WebSocket message types

| Endpoint             | Auth                       |
| -------------------- | -------------------------- |
| `/ws/v1/market-data` | None                       |
| `/ws/v1/trading`     | API key (`mt: 29` sign-in) |

Client-to-server frames: 1 Ping, 5 SubscriptionRequest, 22 OrderRequest, 29 ApiKeySignIn. Server-to-client frames include 3 StatusResponse, 6 SubscriptionResponse, 9 MarketStateUpdate, 15/16 L2Book snapshot/update, 17/18 Trades snapshot/update, 19/20 Wallet snapshot/update, 21 AccountUpdate, 23/24 Orders snapshot/update, 25 FillsUpdate, 26/27 Positions snapshot/update, 28 AccountStatsUpdate, 100 Heartbeat.

## Next steps

* [Authentication](file:///2362779/api/authentication.md) — the full API-key model and request-signing reference.
* [Networks & Configuration](file:///2362779/getting-started/networks.md) — every endpoint, contract address, and market ID for both networks.
