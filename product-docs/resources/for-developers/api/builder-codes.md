# Builder Codes

A **builder code** identifies your integration — a trading terminal, bot, or app — to Perpl. It lets you **charge your own fee** on the orders you route (on top of the protocol fee, attributed to your code and settled to you), and it gives you **volume attribution** even when you charge nothing.

Builder codes are optional. Everything in [Authentication](authentication.md) and the [WebSocket API](websocket.md) works without one — a builder code only adds fee attribution on top.

Three things have to line up:

1. **Perpl registers your builder code** — see [Registering as a builder](#registering-as-a-builder).
2. **Each user's API key is enrolled bound to that code**, with a fee ceiling the user signs for — see [Enrolling a builder-bound key](#enrolling-a-builder-bound-key).
3. **Each order states the fee it wants to charge**, within that ceiling — see [Charging a builder fee](#charging-a-builder-fee).

> **Note:** A builder code is bound to an API key **at enrollment** and is frozen there — it cannot be attached to an already-enrolled key, and an order never names its own code. This is what stops one integration from attributing another's flow to itself.

## Fee units

Builder fees are expressed in **hundred-thousandths** (`per_100k`), the unit the on-chain fee schedule uses. `1` = 0.1 basis points (bps; one bps is one hundredth of one percent) = 0.001%.

| `per_100k` | bps | percent |
|---|---|---|
| `1` | 0.1 | 0.001% |
| `10` | 1 | 0.01% |
| `100` | 10 | 0.1% |

The maximum is `100` (0.1%).

> **Note:** This is **not** the same unit as the market fee rates elsewhere in the API, which are in **micros** (`10^-6`) — see [Numeric scaling](types-and-errors.md#numeric-scaling). `1 per_100k` = `10 micros`.

## Registering as a builder

**Contact Perpl to register.** Builder codes are issued by the operator; there is no self-service endpoint. You provide:

| | |
|---|---|
| **Display name** | Shown to *your users* in the wallet prompt when they authorize a key (see [What the user signs](#what-the-user-signs)). |
| **Perpl account address** | The Perpl account your accrued builder fees are paid out to. |

You receive a **builder id** in the range `1..255` (the id is a `uint8` on-chain, so the registry is deliberately small).

## Enrolling a builder-bound key

Enrollment is the same two-step, wallet-authorized flow as an ordinary key (see [Programmatic enrollment](authentication.md#programmatic-enrollment)) — the only difference is **two extra fields on the payload request**:

```typescript
interface ApiKeyPayloadRequest {
  // ...the standard fields (chain_id, address, public_key, scope_mask, label, ...)

  builder_id?: number;               // your registered builder code, 1..255
  max_builder_fee_per_100k?: number; // fee ceiling the user authorizes, 1 = 0.1 bps
}
```

Request the payload with those fields set:

```typescript
const API_URL = process.env.PERPL_API_URL || 'https://app.perpl.xyz/api';
const ORIGIN = 'https://your-app.example';   // must be whitelisted by Perpl
const CHAIN_ID = Number(process.env.PERPL_CHAIN_ID) || 143;

const BUILDER_ID = Number(process.env.PERPL_BUILDER_ID) || 0;
const MAX_BUILDER_FEE = Number(process.env.PERPL_MAX_BUILDER_FEE_PER_100K) || 0;

const payloadRes = await fetch(`${API_URL}/v1/api-key/payload`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'Origin': ORIGIN },
  body: JSON.stringify({
    chain_id: CHAIN_ID,
    address: '0xUserWalletAddress',
    public_key: publicKeyHex,
    scope_mask: 3,                              // read | trade
    label: 'my trading terminal',
    builder_id: BUILDER_ID,                     // your registered code
    max_builder_fee_per_100k: MAX_BUILDER_FEE,  // ceiling: at most 100 (0.1%) per order
  }),
});
```

Sign and submit exactly as in the [programmatic enrollment flow](authentication.md#programmatic-enrollment). The enroll response echoes the terms back so you can confirm you registered what you intended:

```typescript
interface ApiKeyInfo {
  // ...the standard fields (api_key, address, scope_mask, label, ...)

  // Builder terms, present only on a builder-bound key:
  builder_id?: number;                // the code the key submits under
  builder_name?: string;              // registered display name; empty if the code is no longer registered — show builder_id instead
  max_builder_fee_per_100k?: number;  // enrolled ceiling, 1 = 0.1 bps
  max_builder_fee_pct?: string;       // the same ceiling formatted, e.g. "0.100%"
}
```

All failures are `400`:

| Condition | Message |
|---|---|
| `builder_id` outside `1..255` | `builder_id must be in 1..255` |
| `max_builder_fee_per_100k` above the environment ceiling | `max_builder_fee_per_100k must be at most <N>` |
| `max_builder_fee_per_100k` without a `builder_id` | `max_builder_fee_per_100k requires a builder_id` |
| Non-zero fee ceiling on a `read`-only key | `max_builder_fee_per_100k requires the trade scope` |
| Code not registered / not enabled | `builder code <N> is not registered` |
| Builder enrollment not enabled in that environment | `builder-bound api keys are not enabled` |

> **Note:** `max_builder_fee_per_100k: 0` with a `builder_id` is valid and useful — it gives you **attribution without a fee**. It is also the only shape a `read`-scoped builder key can take (such a key can never place an order).

### What the user signs

**The signer is the end user, not you.** You generate the Ed25519 key pair and supply the proof-of-possession; the user's wallet signs the EIP-712 payload. That signature **is** the fee authorization — there is no separate builder-side approval step — so the payload is written to be legible in the wallet prompt:

- the machine-enforced terms are EIP-712 fields (`builderId`, `maxBuilderFeePer100K`), and
- the same terms in prose are in the payload's `statement` field, naming your registered builder name and the ceiling as a percentage. This is what the user actually reads before approving.

`/api-key/enroll` **re-derives that statement** from the registry and rejects the enrollment if it does not match the one signed. The practical consequence: if your builder name changes between the payload and the enroll call, the enrollment fails — request a fresh payload and have the user sign again.

Users see the same builder terms for every key on their `/apikeys` page and can revoke any key there at any time — consent is granted once, visibility and revocation are continuous.

## Charging a builder fee

Orders are placed over the trading WebSocket as an `OrderRequest` (`mt: 22`, see [Placing Orders](websocket.md#placing-orders-mt-22)). A builder-bound key adds one field, `bf`:

```typescript
ws.send(JSON.stringify({
  mt: 22,
  sn: 1,                       // unique, non-zero — echoed as `cid` on the mt: 3 status (correlates this order)
  rq: nextRequestId(),         // idempotency key, strictly increasing per account
  mkt: 1, acc: accountId,      // BTC (mainnet)
  t: 1,                        // OpenLong
  p: 0, s: 10000,              // market order, 0.1 BTC
  fl: 0, lv: 1000,             // GTC, 10x leverage
  lb: currentBlock + 100,
  bf: 50,                      // builder fee: 5 bps (per_100k), must be <= the key's ceiling
}));
```

Rules:

- **There is no `builder_id` on the request.** The code comes from the authenticating key — a client that could name its own code could attribute another builder's flow to itself.
- **The enrolled ceiling is a maximum, not a default.** Set `bf` on every order you want to charge for. Omitting it is *not* an error: the order executes, attributed to your code, at zero fee — you simply earn nothing on it.
- **A fee above the ceiling is rejected, not clamped.** Silently reducing it would make your accounting disagree with the chain.
- The fee applies to the size that **opens or increases** a position. Closing or reducing fills carry no builder fee.
- Builder fees only exist on orders routed through the API. Orders a user sends directly on-chain cannot be attributed to a builder.

A builder-fee rejection arrives as a [`StatusResponse` (`mt: 3`)](websocket.md#command-status-mt-3), and no order update (`mt: 24`) follows:

| `error` | Condition | `code` |
|---|---|---|
| `builder fee not permitted for this api key` | `bf` above the key's ceiling, or any `bf` on a non-builder key | 400 |
| `api key lacks trade scope` | `read`-scoped key | 403 |

## Reconciling what was charged

Fees are reported **gross**: the `f` (fee) amount on orders, fills, and account events is the total the user paid — protocol fee **plus** builder fee. The builder portion is broken out alongside it, so **never add the two together**.

| Field | On | Meaning |
|---|---|---|
| `bfa` | `Fill` ([`mt: 25`](websocket.md#fill-updates-mt-25)), `Order` ([`mt: 24`](websocket.md#order-updates-mt-24)), `AccountEvent` | Builder-fee portion of that event's `f` (same unit as `f`) |
| `tbf` | `AccountStats` ([`mt: 28`](websocket.md#account-stats-mt-28)) | Lifetime builder fees, already included in `tf` (total fees) |

Both are omitted when zero, so an ordinary (non-builder) account sees no change. A mis-integration is visible within one fill: flow that reaches you with `bf` unset shows up as volume with `bfa: 0`.

## Getting paid

Accrued builder fees are collected per builder code and settled to the Perpl account registered with your code, on a periodic epoch schedule.

## Related pages

- [Authentication](authentication.md) — the full API-key enrollment flow that builder-bound enrollment extends.
- [WebSocket API](websocket.md) — the `OrderRequest` (`mt: 22`) frame the `bf` field is set on, and the fill/order/stats updates that carry `bfa` / `tbf`.
- [Types & Errors](types-and-errors.md) — numeric scaling (micros vs `per_100k`) and the shared enums.
