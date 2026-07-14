# Types & Errors

This page is the reference for the core data types you will decode off the wire and the error model you will handle in a client. It covers the primitive type aliases, numeric scaling rules, market IDs, and the full set of enumerations, followed by HTTP (Hypertext Transfer Protocol) status codes, WebSocket close codes, order-reject reasons, and rate limits.

All types are generated from the backend Go structs and shipped as TypeScript interfaces (via `tygo`), so the field names shown below match the wire format byte-for-byte. The full object shapes (`Order`, `Position`, `Wallet`, `Account`, `Market`, …) are documented alongside the endpoints that return them — see [REST API](/broken/pages/ae7aa55bf35629d758b5d2d04960b35978d996f2) and [WebSocket API](/broken/pages/e71e088c834b9b77da57aec97f4819cfcfb4fab6). This page focuses on the shared primitives, scaling, and enums those objects are built from, plus the error model common to both channels.

## Primitive types

Most numeric fields are type aliases over `number` or `string`. The alias names carry the semantics — for example, a `Price` is always a scaled integer and a `Micros` is always a `10^-6` fraction.

```typescript
type ChainID = number;       // uint64 - EIP-155 chain ID
type InstanceID = number;    // uint32 - Protocol instance ID
type TokenID = number;       // uint32 - Token ID
type MarketID = number;      // uint32 - Market ID
type PerpetualID = number;   // uint32 - Perpetual ID (smart contract)
type FeeLevelID = number;    // uint32 - Fee level ID
type AccountID = number;     // uint64 - Trading account ID
type OrderID = number;       // uint64 - Order ID
type RequestID = number;     // uint64 - Request ID (idempotency key)
type PositionID = number;    // uint64 - Position ID
type Decimals = number;      // uint8  - Decimal places
type Fraction = number;      // uint32 - Fraction in hundredths
type Micros = number;        // int64  - Value in 10^-6 fractions
type Amount = string;        // Decimal string for large numbers
type Price = number;         // uint64 - Scaled price
type SPrice = number;        // int64  - Signed scaled price
type Size = number;          // uint64 - Scaled size
```

{% hint style="info" %}
`Amount` is a **decimal string**, not a number, so large collateral values do not lose precision when passing through JSON (JavaScript Object Notation). Parse it with a big-decimal library rather than `Number()`.
{% endhint %}

## Numeric scaling

Prices and sizes are transmitted as **scaled integers**. To convert to and from human-readable values, use the per-market decimal counts from [`MarketConfig`](/broken/pages/ae7aa55bf35629d758b5d2d04960b35978d996f2) (`price_decimals`, `size_decimals`). Fees use `Micros` (`10^-6` fractions), margins and other ratios use `Fraction` (hundredths), and leverage is expressed in hundredths.

| Concept           | Type               | Rule                                              |
| ----------------- | ------------------ | ------------------------------------------------- |
| Price             | `Price` / `SPrice` | Divide by `10^price_decimals` for the human value |
| Size              | `Size`             | Divide by `10^size_decimals` for the human value  |
| Fee               | `Micros`           | `10^-6` fraction; **negative = rebate**           |
| Margin / ratio    | `Fraction`         | Hundredths — divide by 100; `1000` = 10.00%       |
| Leverage          | `number`           | Hundredths, e.g. `1000` = 10x                     |
| Collateral amount | `Amount`           | Decimal string; collateral token has 6 decimals   |

Helper functions for price scaling:

```typescript
// Convert a scaled price to a human-readable value
function scalePrice(scaled: number, priceDecimals: number): number {
  return scaled / Math.pow(10, priceDecimals);
}

// Convert a human price to the scaled integer sent on the wire
function unscalePrice(price: number, priceDecimals: number): number {
  return Math.round(price * Math.pow(10, priceDecimals));
}
```

Worked example — BTC on mainnet has `price_decimals = 1` and `size_decimals = 5`:

```typescript
// Price: scaled 950000 with price_decimals=1 → $95,000.0
scalePrice(950000, 1);      // => 95000
unscalePrice(95000, 1);     // => 950000

// Size: 0.1 BTC with size_decimals=5 → scaled 10000
Math.round(0.1 * Math.pow(10, 5));  // => 10000
```

{% hint style="info" %}
Leverage in an `OrderRequest` and on `Order` / `Position` objects is in hundredths — send `1000` for 10x, `250` for 2.5x.
{% endhint %}

## Timestamps

Three timestamp shapes appear across the types, each adding one more level of on-chain locality. All time fields (`t`) are Unix epoch **milliseconds**.

```typescript
interface BlockTimestamp {
  b?: number;  // Block number
  t?: number;  // Timestamp (ms)
}

interface BlockTxTimestamp {
  b?: number;    // Block number
  t?: number;    // Timestamp (ms)
  tx?: number;   // Transaction index in block
  txid?: string; // Transaction hash
}

interface BlockTxLogTimestamp {
  b?: number;    // Block number
  t?: number;    // Timestamp (ms)
  tx?: number;   // Transaction index in block
  txid?: string; // Transaction hash
  l?: number;    // Log index in transaction
}
```

## Market IDs

Market IDs differ per network. Use `GET /api/v1/pub/context` to fetch the live list for the network you are connected to; the values below are the current assignments.

| Market | Mainnet ID | Testnet ID |
| ------ | ---------- | ---------- |
| BTC    | `1`        | `16`       |
| MON    | `10`       | `64`       |
| ETH    | `20`       | `32`       |
| SOL    | `31`       | `48`       |
| HYPE   | `40`       | —          |
| ZEC    | `50`       | `256`      |

{% hint style="info" %}
For the full network reference (RPC URLs, chain IDs, contract addresses, collateral token) see [Networks](file:///2362779/getting-started/networks.md).
{% endhint %}

## Order enums

### OrderType

The `t` field on orders and fills. Sides are encoded directly in the type: open vs. close and long vs. short.

| Value | Name                       | Description              |
| ----- | -------------------------- | ------------------------ |
| 0     | Unspecified                |                          |
| 1     | OpenLong                   | Open a long position     |
| 2     | OpenShort                  | Open a short position    |
| 3     | CloseLong                  | Close a long position    |
| 4     | CloseShort                 | Close a short position   |
| 5     | Cancel                     | Cancel an order          |
| 6     | IncreasePositionCollateral | Add margin to a position |
| 7     | Change                     | Modify an existing order |

### OrderFlags

The `fl` field. Controls time-in-force / execution behavior.

| Value | Name              | Description                                            |
| ----- | ----------------- | ------------------------------------------------------ |
| 0     | GoodTillCancel    | Default — rests until filled or canceled (GTC)         |
| 1     | PostOnly          | Maker-only; rejected if it would cross the book        |
| 2     | FillOrKill        | Fill the entire order or cancel it (FOK)               |
| 4     | ImmediateOrCancel | Fill what is available now, cancel the remainder (IoC) |

### TriggerPriceCondition

The `tpc` field on trigger orders. `Last` conditions compare against the last trade price; `Mark` conditions compare against the mark price.

| Value | Name        | Description                             |
| ----- | ----------- | --------------------------------------- |
| 0     | Unspecified |                                         |
| 1     | GTELast     | Trigger when last price ≥ trigger price |
| 2     | LTELast     | Trigger when last price ≤ trigger price |
| 3     | GTEMark     | Trigger when mark price ≥ trigger price |
| 4     | LTEMark     | Trigger when mark price ≤ trigger price |

### OrderStatus

The `st` field — the current lifecycle state of an order.

| Value | Name            |
| ----- | --------------- |
| 0     | Unspecified     |
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

### OrderStatusReason

The `sr` field — the reason an order reached its current status. This is the primary source of order-reject detail; see [Error model](types-and-errors.md#error-model) below for how to consume it. The full enumeration:

| Value | Name                                    |
| ----- | --------------------------------------- |
| 0     | Unspecified                             |
| 1     | AmountExceedsAvailableBalance           |
| 2     | AccountFrozen                           |
| 3     | CancelExistingInvalidCloseOrders        |
| 4     | CantChangeCloseOrder                    |
| 5     | ChangeExpiredOrderNeedsNewExpiry        |
| 6     | ClearingExpiredOrder                    |
| 7     | ClearingFrozenAccountOrder              |
| 8     | ClearingInvalidCloseOrder               |
| 9     | ClearingSelfMatchingOrder               |
| 10    | CloseOrderExceedsPosition               |
| 11    | CloseOrderPositionMismatch              |
| 12    | ContractNotOperational                  |
| 13    | CrossesBook                             |
| 14    | ExceedsLastExecutionBlock               |
| 15    | ForwardingReverted                      |
| 16    | ImmediateOrCancelExecuted               |
| 17    | ImmediateOrderUnderMinimum              |
| 18    | InsuficientFundsForRecycleFee           |
| 19    | InvalidAccountFrozenOrder               |
| 20    | InvalidExpiryBlock                      |
| 21    | InvalidOrderId                          |
| 22    | MakerOrderFilled                        |
| 23    | MakerOrderSettlementFailed              |
| 24    | MaximumAccountOrders                    |
| 25    | MaxMatchesReached                       |
| 26    | NoOp                                    |
| 27    | OrderBookFull                           |
| 28    | OrderCancelled                          |
| 29    | OrderCancelledByAdmin                   |
| 30    | OrderCancelledByLiquidator              |
| 31    | OrderChanged                            |
| 32    | OrderDescIdTooLow                       |
| 33    | OrderDoesNotExist                       |
| 34    | OrderForwardingNotAllowed               |
| 35    | OrderPlaced                             |
| 36    | OrderPostFailed                         |
| 37    | OrderSettlementImpliesInsolvent         |
| 38    | OrderSizeExceedsAvailableSize           |
| 39    | PostOrderUnderMinimum                   |
| 40    | PriceOutOfRange                         |
| 41    | RecycleBalanceInsufficientSevere        |
| 42    | SizeOutOfRange                          |
| 43    | TakerOrderFilled                        |
| 44    | TakerOrderSettlementFailed              |
| 45    | UnableToCancelOrder                     |
| 46    | UnmatchedLotRemainsInFillOrKill         |
| 47    | UnspecifiedCollateral                   |
| 48    | UnspecifiedPrice                        |
| 49    | UnspecifiedSize                         |
| 50    | WrongAccountForOrder                    |
| 51    | WrongChainForOrder                      |
| 52    | WrongMarketForOrder                     |
| 53    | PerpetualInsolvent                      |
| 54    | Triggered                               |
| 55    | InvalidAmount                           |
| 56    | InvalidFlags                            |
| 57    | InvalidTriggerOrder                     |
| 58    | WrongTriggerPosition                    |
| 59    | TriggerDescIdTooLow                     |
| 60    | TriggerOrderRequest                     |
| 61    | ValueExceedsMaximum                     |
| 62    | ClearingRemainingOrderLockBeyondBalance |
| 63    | PriceSetDuringTriggerExec               |
| 64    | TriggeredExecutionAttemptsExhausted     |
| 65    | TriggeredOrderExecuted                  |
| 66    | TriggeredOrderPartiallyFilled           |
| 67    | TriggeredOrderExpired                   |
| 68    | TriggeredOrderRecoverableFailure        |

### LiquiditySide

The `l` field on fills — whether your order provided liquidity (maker) or removed it (taker).

| Value | Name        |
| ----- | ----------- |
| 0     | Unspecified |
| 1     | Maker       |
| 2     | Taker       |

## Position enums

### PositionType

The `sd` field on positions — the direction of the position.

| Value | Name        |
| ----- | ----------- |
| 0     | Unspecified |
| 1     | Long        |
| 2     | Short       |

### PositionStatus

The `st` field on positions.

| Value | Name        |
| ----- | ----------- |
| 0     | Unspecified |
| 1     | Open        |
| 2     | Closed      |
| 3     | Liquidated  |
| 4     | Deleveraged |
| 5     | Unwound     |
| 6     | Failed      |

### PositionStatusReason

The `sr` field on positions. The documented common values:

| Value | Name                |
| ----- | ------------------- |
| 13    | PositionClosed      |
| 14    | PositionDecreased   |
| 15    | PositionDeleveraged |
| 17    | PositionIncreased   |
| 18    | PositionInverted    |
| 19    | PositionLiquidated  |
| 21    | PositionOpened      |
| 22    | PositionUnwound     |

## Trade & account enums

### TradeSide

The `sd` field on public trades.

| Value | Name |
| ----- | ---- |
| 1     | Buy  |
| 2     | Sell |

### AccountEventType

The `et` field on account events (returned by `GET /api/v1/trading/account-history` and the `AccountUpdate` WebSocket message).

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

## API-key scope

The `scope_mask` field on an API key is a `uint32` bitmask. `trade` implies `read`. **Withdrawals and transfers-out are never permitted via an API key, under any scope.**

```typescript
type ScopeMask = number;                     // uint32 bitmask

const ScopeRead: ScopeMask = 1 << 0;         // 1 - read account/order/position data
const ScopeTrade: ScopeMask = 1 << 1;        // 2 - place/cancel/modify orders (implies read)
const ScopeAll = ScopeRead | ScopeTrade;     // 3 - full scope
```

| `scope_mask` | Grants               |
| ------------ | -------------------- |
| `1`          | Read only            |
| `2`          | Trade (implies read) |
| `3`          | Read + trade         |

See [Authentication](/broken/pages/95f3238273f946046f1da0b3b3d345d40c81ddb4) for how scope is enforced on each channel.

## Error model

The API conveys errors through **HTTP status codes** on the REST channel, a **close code** on the WebSocket channel, and **status-reason enums** (`sr`) on order and position updates. There is no separate JSON error-envelope schema — inspect the status code (or close code, or `sr`) to determine the failure.

### HTTP status codes (REST)

| Code | Meaning               | Common cause                                                                                    |
| ---- | --------------------- | ----------------------------------------------------------------------------------------------- |
| 200  | Success               |                                                                                                 |
| 400  | Bad Request           | Malformed request                                                                               |
| 401  | Unauthorized          | Bad or stale signature, replayed nonce, revoked or expired key, caller IP not in the allow-list |
| 403  | Forbidden             | Scope insufficient (e.g. a `read`-scoped key attempting to place an order)                      |
| 404  | Not Found             | Resource missing, or no on-chain account exists yet                                             |
| 429  | Too Many Requests     | Rate limit exceeded — see [Rate limits](types-and-errors.md#rate-limits)                        |
| 500  | Internal Server Error | Server-side failure                                                                             |

### Enrollment status codes

The two API-key enrollment endpoints (`POST /api/v1/api-key/payload`, `POST /api/v1/api-key/enroll`) return these additional codes:

| Code | Meaning                                                                                           |
| ---- | ------------------------------------------------------------------------------------------------- |
| 404  | Target profile not found (invalid `target_profile`)                                               |
| 409  | Public key already registered — revoked keys are **not** re-enrollable; generate a fresh key pair |
| 423  | Per-profile key limit reached (**maximum 16 active keys**)                                        |

### WebSocket close code

| Code | Meaning                | Handling                                                             |
| ---- | ---------------------- | -------------------------------------------------------------------- |
| 3401 | Authentication failure | Re-send a fresh signed `ApiKeySignIn` (`mt: 29`) frame and reconnect |

{% hint style="warning" %}
A `3401` close means the signed sign-in frame was rejected (bad signature, stale timestamp, replayed nonce, or a revoked/expired key). Do not retry with the same frame — recompute the signature with a current timestamp and a new nonce.
{% endhint %}

### Order-reject reasons

When an order is rejected or transitions to a terminal state, the `sr` field on the order update carries an [`OrderStatusReason`](types-and-errors.md#orderstatusreason). Read `sr` together with `st` (the [`OrderStatus`](types-and-errors.md#orderstatus)) — an `st` of `Failed` (7) or `Canceled` (5) paired with `sr` tells you exactly why. Frequently seen reasons:

| `sr` | Name                          | Typical cause                                                     |
| ---- | ----------------------------- | ----------------------------------------------------------------- |
| 1    | AmountExceedsAvailableBalance | Order would exceed available collateral                           |
| 13   | CrossesBook                   | A `PostOnly` order would have crossed the book                    |
| 14   | ExceedsLastExecutionBlock     | Order not executed before its last valid block                    |
| 15   | ForwardingReverted            | On-chain forwarding transaction reverted                          |
| 32   | OrderDescIdTooLow             | `rq` (RequestID) is not strictly greater than the account's `lfr` |
| 38   | OrderSizeExceedsAvailableSize | Requested size exceeds available book/position size               |
| 53   | PerpetualInsolvent            | The perpetual is insolvent                                        |

{% hint style="info" %}
`sr = 32` (`OrderDescIdTooLow`) means the idempotency key `rq` was not strictly increasing. Seed `rq` from the account's `lfr` (last forwarded request ID) as `rq = max(local, lfr) + 1`. See [WebSocket API](/broken/pages/e71e088c834b9b77da57aec97f4819cfcfb4fab6) for order placement.
{% endhint %}

### Rate limits

Limits are approximate; treat an HTTP `429` (or the equivalent throttling on WebSocket) as the definitive signal and back off.

| Channel                 | Limit              | Scope                                 |
| ----------------------- | ------------------ | ------------------------------------- |
| REST — public           | \~100 requests/min | `/api/v1/pub/*`, market data          |
| REST — authenticated    | \~60 requests/min  | Profile and trading-history endpoints |
| WebSocket — messages    | \~50 messages/sec  | Per connection                        |
| WebSocket — connections | \~5                | Per IP address                        |

**Recommended handling:** on a `429`, retry with exponential backoff — 1 s, then 2 s, then 4 s.

```typescript
async function withBackoff<T>(fn: () => Promise<Response>): Promise<Response> {
  const delays = [1000, 2000, 4000]; // ms
  for (let attempt = 0; ; attempt++) {
    const res = await fn();
    if (res.status !== 429 || attempt >= delays.length) return res;
    await new Promise((r) => setTimeout(r, delays[attempt]));
  }
}
```

## Related pages

* [REST API](/broken/pages/ae7aa55bf35629d758b5d2d04960b35978d996f2) — endpoints and the full response object shapes.
* [WebSocket API](/broken/pages/e71e088c834b9b77da57aec97f4819cfcfb4fab6) — message types, streams, and order placement.
* [Authentication](/broken/pages/95f3238273f946046f1da0b3b3d345d40c81ddb4) — canonical string format and request signing.
* [Networks](file:///2362779/getting-started/networks.md) — chain IDs, RPC URLs, contract addresses, market IDs.
