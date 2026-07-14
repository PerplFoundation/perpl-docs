# REST

The Perpl REST (Representational State Transfer) API is served over HTTPS and covers public market data, account and trading history, and API-key enrollment. Streaming data (order books, live prices, order/position updates, order placement) is handled by the [WebSocket API](/broken/pages/e71e088c834b9b77da57aec97f4819cfcfb4fab6), not REST.

All response types are generated from Go structs and shipped as TypeScript interfaces (via `tygo`), so the field names shown below match the wire format exactly.

## Base URL

The REST base URL **includes the `/api` suffix**. Choose the base URL for your target network:

| Network           | REST base URL                   | Chain ID |
| ----------------- | ------------------------------- | -------- |
| Mainnet (default) | `https://app.perpl.xyz/api`     | `143`    |
| Testnet           | `https://testnet.perpl.xyz/api` | `10143`  |

The examples on this page read the base URL from the `PERPL_API_URL` environment variable, falling back to the mainnet default:

```bash
export PERPL_API_URL="https://app.perpl.xyz/api"    # mainnet
# export PERPL_API_URL="https://testnet.perpl.xyz/api"  # testnet
```

{% hint style="info" %}
Market IDs differ per network. Mainnet: BTC=1, MON=10, ETH=20, SOL=31, HYPE=40, ZEC=50. Testnet: BTC=16, ETH=32, SOL=48, MON=64, ZEC=256. For the full list of network values (RPC URLs, contract addresses, collateral token) see [Networks](file:///2362779/getting-started/networks.md).
{% endhint %}

## Authentication overview

Authentication uses **API keys**, which are Ed25519 key pairs enrolled once with a wallet signature. The server stores only your public key; the private key never leaves your client. There is no bearer token and no session cookie — **every authenticated request is signed** with four headers:

| Header            | Value                                                                         |
| ----------------- | ----------------------------------------------------------------------------- |
| `X-API-Key`       | The opaque token returned at enrollment                                       |
| `X-API-Timestamp` | Request timestamp in milliseconds (must be within ±30 seconds of server time) |
| `X-API-Nonce`     | Client-random, single-use base64url value (no padding)                        |
| `X-API-Signature` | `base64url(ed25519_sign(privateKey, canonicalString))`, no padding            |

Each endpoint below is labelled with one of three authentication requirements:

* **None** — no signature needed.
* **Optional** — works unauthenticated; providing an API-key signature personalizes the response.
* **API key** — an API-key signature is required.

A key also carries a **scope** (`read`, `trade`, or both). REST history and profile reads require `read`; `trade` is used for order placement over WebSocket. **Withdrawals and transfers-out are never permitted via API key, under any scope.**

{% hint style="info" %}
Enrolling an API key only authorizes API access — it does not create an exchange account. Trading additionally requires an on-chain account created via `createAccount(uint256 amountCNS)` on the Exchange contract. Endpoints that need an on-chain account may return `404` if none exists.
{% endhint %}

For the full canonical-string format and a signing helper, see [Authentication](/broken/pages/95f3238273f946046f1da0b3b3d345d40c81ddb4). To obtain a key, see [Authentication → Creating a key](/broken/pages/95f3238273f946046f1da0b3b3d345d40c81ddb4#creating-a-key).

***

## Public endpoints

### GET /api/v1/pub/context

Returns global protocol configuration: chain, protocol instances, tokens, and markets.

* **Authentication**: Optional (a signature personalizes the response)

**Response**:

```typescript
interface Context {
  chain: Chain;
  instances: ProtocolInstance[];
  tokens: Token[];
  markets: Market[];
}
```

Each `ProtocolInstance` carries operational limits such as `min_account_open_amount`, `min_deposit_amount`, `min_withdraw_amount`, and `max_account_trigger_orders`. Each `Market` carries `price_decimals` and `size_decimals`, which you use to scale integer prices and sizes into human-readable values (see [Types](/broken/pages/ddd73ec926cff4ce86a45002c3091dc0c9a5e2e1)).

**Example**:

```bash
# Using the default live URL
curl https://app.perpl.xyz/api/v1/pub/context

# Or using the environment variable
curl "${PERPL_API_URL:-https://app.perpl.xyz/api}/v1/pub/context"
```

***

### GET /api/v1/market-data/:market\_id/candles/:resolution/:from-:to

Returns OHLCV (open-high-low-close-volume) candlestick data.

* **Authentication**: None

**URL parameters**:

| Parameter    | Type   | Description                                         |
| ------------ | ------ | --------------------------------------------------- |
| `market_id`  | number | Market ID (e.g. `1` for BTC on mainnet)             |
| `resolution` | number | Candle resolution in seconds (see supported values) |
| `from`       | number | Start timestamp (ms)                                |
| `to`         | number | End timestamp (ms)                                  |

**Limits**: A maximum of **1024 candles** per request.

**Supported resolutions** (seconds): `60` (1m), `300` (5m), `900` (15m), `1800` (30m), `3600` (1h), `7200` (2h), `14400` (4h), `28800` (8h), `43200` (12h), `86400` (1d).

**Response**:

```typescript
interface CandleSeries {
  mt: number;           // Message type
  at: BlockTimestamp;   // Timestamp
  r: number;            // Resolution (seconds)
  d: Candle[];          // Candle data
}

interface Candle {
  t: number;    // Open timestamp (ms)
  o: number;    // Open price (scaled)
  c: number;    // Close price (scaled)
  h: number;    // High price (scaled)
  l: number;    // Low price (scaled)
  v: string;    // Volume (collateral token)
  n: number;    // Number of trades
}
```

**Example**:

```bash
# Get 1-hour BTC candles for the last 24 hours (mainnet, market_id=1)
API_URL=${PERPL_API_URL:-https://app.perpl.xyz/api}
FROM=$(($(date +%s) * 1000 - 86400000))
TO=$(($(date +%s) * 1000))
curl "${API_URL}/v1/market-data/1/candles/3600/${FROM}-${TO}"
```

***

### GET /api/v1/profile/announcements

Returns active announcements.

* **Authentication**: Optional (works unauthenticated for the public audience; a signature personalizes the returned announcements)

**Response**:

```typescript
interface AnnouncementsResponse {
  ver: number;
  active: Announcement[];
}

interface Announcement {
  id: number;
  title: string;
  content: string;
}
```

**Example**:

```bash
curl "${PERPL_API_URL:-https://app.perpl.xyz/api}/v1/profile/announcements"
```

***

## API-key enrollment endpoints

Enrollment is a one-time, wallet-authorized flow that turns a locally generated Ed25519 key pair into an `X-API-Key` token. Both endpoints below are authorized by a **wallet signature**, not an API-key signature, and are CORS (cross-origin resource sharing) enabled — the request `Origin` must be pre-whitelisted by Perpl.

For the full step-by-step flow (keypair generation, EIP-712 signing, proof-of-possession), see [Authentication → Programmatic enrollment](/broken/pages/95f3238273f946046f1da0b3b3d345d40c81ddb4#programmatic-enrollment).

### POST /api/v1/api-key/payload

Returns the EIP-712 typed data to sign for enrollment, plus an opaque `mac` that you echo back on enroll.

* **Authentication**: Wallet signature

**Purpose**: Obtain the `typed_data` + `mac` used in the next step.

### POST /api/v1/api-key/enroll

Enrolls the public key and returns `ApiKeyInfo`. The `api_key.api_key` field is the opaque `X-API-Key` token.

* **Authentication**: Wallet signature

**Request** (echo `typed_data` + `mac` from the payload step, plus two signatures):

* `signature` — wallet secp256k1 EIP-712 signature (proves account ownership)
* `pop_signature` — Ed25519 proof-of-possession over the enrollment digest

{% hint style="info" %}
Store the returned `X-API-Key` token immediately — it is not re-derivable. Listing and revoking keys is done in the web UI (`/apikeys`), not via the API. A revoked public key cannot be re-enrolled; use a fresh keypair.
{% endhint %}

**Enroll status codes**:

| Code  | Meaning                                                            |
| ----- | ------------------------------------------------------------------ |
| `404` | Target profile not found                                           |
| `409` | Public key already registered (revoked keys are not re-enrollable) |
| `423` | Per-profile key limit reached (maximum 16 active keys)             |

***

## Profile endpoints

### GET /api/v1/profile/ref-code

Returns your current referral code.

* **Authentication**: API key

**Response**:

```typescript
interface RefCode {
  code: string;
  limit?: number;      // Max profiles that can be created with this code
  used?: number;       // Profiles already created with this code
  volume?: Amount;     // Total volume generated by referred profiles (tier-1 only, all time)
  created_at: number;  // Ref code creation timestamp (ms)
}
```

Returns `404` with an empty `code` if no referral code is assigned.

**Example** (`$SIG`, `$TS`, `$NONCE` are the signed values — see [Authentication](/broken/pages/95f3238273f946046f1da0b3b3d345d40c81ddb4)):

```bash
curl "${PERPL_API_URL:-https://app.perpl.xyz/api}/v1/profile/ref-code" \
  -H "X-API-Key: ${PERPL_API_KEY}" \
  -H "X-API-Timestamp: ${TS}" \
  -H "X-API-Nonce: ${NONCE}" \
  -H "X-API-Signature: ${SIG}"
```

***

## Trading history endpoints

All trading history endpoints require an API-key signature and support pagination. The response is always a page object:

```typescript
interface HistoryPage<T> {
  d: T[];      // Data array, newest to oldest
  np: string;  // Next-page cursor
}
```

**Pagination query parameters**:

| Parameter | Type   | Default | Description                              |
| --------- | ------ | ------- | ---------------------------------------- |
| `page`    | string | –       | Cursor from the previous response's `np` |
| `count`   | number | 50      | Items per page (maximum 100)             |

{% hint style="info" %}
Server-side filtering by market ID or date range is not currently supported. Filter results client-side if needed.
{% endhint %}

### GET /api/v1/trading/account-history

Returns account events (deposits, withdrawals, settlements, funding, and more).

* **Authentication**: API key

**Response**:

```typescript
interface AccountHistoryPage {
  d: AccountEvent[];
  np: string;
}

interface AccountEvent {
  at: BlockTxLogTimestamp;  // Timestamp
  in: number;               // Instance ID
  id: number;               // Account ID
  et: AccountEventType;     // Event type
  m?: number;               // Market ID
  r?: number;               // Request ID
  o?: number;               // Order ID
  p?: number;               // Position ID
  a: string;                // Amount change
  b: string;                // Updated balance
  lb: string;               // Locked balance
  f: string;                // Fee
}
```

**Account event types** (`et`):

| Value | Name                        |
| ----- | --------------------------- |
| 0     | Unspecified                 |
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

***

### GET /api/v1/trading/fills

Returns order fill history.

* **Authentication**: API key

**Response**:

```typescript
interface FillHistoryPage {
  d: Fill[];
  np: string;
}

interface Fill {
  at: BlockTxLogTimestamp;
  mkt: number;      // Market ID
  acc: number;      // Account ID
  oid: number;      // Order ID
  t: OrderType;     // Order type
  l: LiquiditySide; // Maker=1, Taker=2
  p?: number;       // Fill price (scaled)
  s: number;        // Filled size (scaled)
  f: string;        // Fee/rebate
}
```

***

### GET /api/v1/trading/order-history

Returns historical order events.

* **Authentication**: API key

**Response**:

```typescript
interface OrderHistoryPage {
  d: Order[];
  np: string;
}
```

See [Types](/broken/pages/ddd73ec926cff4ce86a45002c3091dc0c9a5e2e1) for the `Order` structure.

***

### GET /api/v1/trading/position-history

Returns position history.

* **Authentication**: API key

**Response**:

```typescript
interface PositionHistoryPage {
  d: Position[];
  np: string;
}
```

See [Types](/broken/pages/ddd73ec926cff4ce86a45002c3091dc0c9a5e2e1) for the `Position` structure.

***

## Pagination example

Each request is signed with the API-key headers. `signedRequest(method, target, body)` is the helper defined in [Authentication](/broken/pages/95f3238273f946046f1da0b3b3d345d40c81ddb4#signing-rest-requests) — note that the `request-target` (path + query string) must be signed exactly as sent.

```typescript
async function fetchAllFills() {
  const fills: Fill[] = [];
  let page: string | undefined;

  do {
    const params = new URLSearchParams({ count: '100' });
    if (page) params.set('page', page);
    const target = `/v1/trading/fills?${params.toString()}`;

    // signed with X-API-* headers, see authentication.md
    const response = await signedRequest('GET', target);

    const data: FillHistoryPage = await response.json();
    fills.push(...data.d);
    page = data.np;
  } while (page);

  return fills;
}
```

***

## Rate limits

Rate limits are approximate. Monitor for HTTP `429` responses and back off.

| Type               | Limit         | Applies to                   |
| ------------------ | ------------- | ---------------------------- |
| REST public        | \~100 req/min | `/api/v1/pub/*`, market data |
| REST authenticated | \~60 req/min  | profile, trading history     |

On a `429 Too Many Requests`, retry with exponential backoff (for example 1s, then 2s, then 4s).

## Errors

**HTTP status codes**:

| Code  | Meaning                                                                             |
| ----- | ----------------------------------------------------------------------------------- |
| `200` | Success                                                                             |
| `400` | Bad Request                                                                         |
| `401` | Unauthorized — bad or stale signature, replayed nonce, or a revoked/expired key     |
| `403` | Forbidden — insufficient scope (for example a `read` key attempting a trade action) |
| `404` | Not Found — including no on-chain account for the caller                            |
| `429` | Too Many Requests                                                                   |
| `500` | Internal Server Error                                                               |

A request is rejected with `401` if the timestamp is outside the ±30-second window, the nonce has already been used within the validity window, the key is past its `expires_at`, or the caller IP is not in the key's `ip_cidrs` allow-list (CIDR = Classless Inter-Domain Routing; maximum 4 CIDRs) when one is set.

{% hint style="info" %}
No JSON error-envelope schema is defined in the source documentation. Inspect the HTTP status code to classify failures.
{% endhint %}
