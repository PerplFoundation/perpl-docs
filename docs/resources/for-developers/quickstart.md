# Quickstart

Get from zero to your first authenticated Perpl API call in a few minutes.

Perpl authenticates programmatic clients (bots, terminals, scripts) with an **API key** — an **Ed25519 key pair** (a modern elliptic-curve digital-signature scheme). The server stores only the public key; the private key never leaves your machine. There is no bearer token or session cookie: **every request is signed** with the key's private key.

This page covers the fastest path: create a key, configure your environment, then sign and send a REST (Representational State Transfer, over HTTPS) request. When you are ready to trade, jump to [placing an order](quickstart.md#next-steps).

## Prerequisites

* **Node.js 18 or newer** — for the built-in `fetch` and `Buffer` `base64url` support used in the snippets below.
* The **`@noble/ed25519`** package for signing:

```bash
npm install @noble/ed25519
```

{% stepper %}
{% step %}
## Create an API key

Create a key in the web UI. Connect your wallet, then open the API-keys page:

| Network | API-keys page                     |
| ------- | --------------------------------- |
| Mainnet | https://app.perpl.xyz/apikeys     |
| Testnet | https://testnet.perpl.xyz/apikeys |

The UI walks you through the wallet-signed enrollment and hands you two values:

* **`X-API-Key` token** — the opaque token sent on every request.
* **Ed25519 private key** — used to sign every request; **store it now, it is not re-derivable.**

A key carries a **scope** (`read`, `trade`, or both; `trade` implies `read`). Reading your account data needs only `read`; placing orders needs `trade`. Withdrawals and transfers-out are **never** permitted via an API key, at any scope.

> **Note:** An API key only authorizes API access. Trading also requires an on-chain exchange account (created with initial collateral on the Exchange contract). Until that account exists, some authenticated endpoints return **404**.

Third-party integrations can also enroll keys programmatically with a wallet-signed flow — see [Integrations](file:///2362779/api/authentication.md).
{% endstep %}

{% step %}
## Configure your environment

The snippets read configuration from environment variables. Set the ones for your target network:

```bash
# Mainnet (defaults shown; omit to use them)
export PERPL_API_URL="https://app.perpl.xyz/api"
export PERPL_WS_URL="wss://app.perpl.xyz"        # WebSocket URL has NO /api prefix
export PERPL_CHAIN_ID="143"

# Testnet
# export PERPL_API_URL="https://testnet.perpl.xyz/api"
# export PERPL_WS_URL="wss://testnet.perpl.xyz"
# export PERPL_CHAIN_ID="10143"

# Your enrolled key (from Step 1)
export PERPL_API_KEY="<your X-API-Key token>"
export PERPL_API_KEY_SECRET="<hex of your 32-byte Ed25519 private key>"
```

| Variable               | Purpose                                   | Mainnet default             |
| ---------------------- | ----------------------------------------- | --------------------------- |
| `PERPL_API_URL`        | REST base URL (**includes** `/api`)       | `https://app.perpl.xyz/api` |
| `PERPL_WS_URL`         | WebSocket base URL (**no** `/api` prefix) | `wss://app.perpl.xyz`       |
| `PERPL_CHAIN_ID`       | Chain ID — part of every signature        | `143` (testnet: `10143`)    |
| `PERPL_API_KEY`        | The opaque `X-API-Key` token              | —                           |
| `PERPL_API_KEY_SECRET` | Hex of the 32-byte Ed25519 private key    | —                           |

For the full network reference (RPC URLs, contract and token addresses, market IDs), see [Networks](networks-and-configuration.md).
{% endstep %}

{% step %}
## Sign and send your first request

Every REST call is signed. The signature covers a **canonical string** of six fields joined by newlines (`\n`):

```
<chain_id>
<HTTP_METHOD>          e.g. GET, POST
<request-target>       path + query string exactly as sent, e.g. /v1/trading/fills?count=1
<timestamp_ms>         unix epoch milliseconds, decimal
<nonce>                client-random, base64url (no padding)
<sha256(body) hex>     hex of SHA-256 over the raw body ("" body → sha256 of empty string)
```

The signature is `base64url(ed25519_sign(privateKey, canonical))` (no padding), sent alongside three more headers:

| Header            | Value                                           |
| ----------------- | ----------------------------------------------- |
| `X-API-Key`       | the opaque token from Step 1                    |
| `X-API-Timestamp` | the `timestamp_ms` used in the canonical string |
| `X-API-Nonce`     | the `nonce` used in the canonical string        |
| `X-API-Signature` | `base64url(ed25519 signature)`                  |

Save the following as `first-request.ts` and run it. It reads your most recent fill — a small, safe read that works with a `read`-scoped key:

```typescript
import { createHash, randomBytes } from 'crypto';
import * as ed from '@noble/ed25519';

const API_URL = process.env.PERPL_API_URL || 'https://app.perpl.xyz/api';
const CHAIN_ID = Number(process.env.PERPL_CHAIN_ID) || 143;
const API_KEY = process.env.PERPL_API_KEY!;
const privateKey = Buffer.from(
  (process.env.PERPL_API_KEY_SECRET ?? '').replace(/^0x/, ''),
  'hex',
);

// `target` is the path + query string exactly as sent.
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

// Example: read your most recent fill.
const res = await signedFetch('GET', '/v1/trading/fills?count=1');
console.log(res.status, await res.json());
```

> **Note:** The `request-target` must match **byte-for-byte** what the server receives — include the query string (`?count=1`) exactly as sent, and sign that exact string.

### Signature validity

* **Timestamp window** — `X-API-Timestamp` must be within **30 seconds** of server time. Keep the client clock in sync.
* **Nonce** — single-use within the validity window. Generate a fresh random `nonce` per request; replays are rejected.

### If it fails

| Status | Meaning                                                                               | Fix                                                                                |
| ------ | ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `401`  | Bad/stale signature, replayed nonce, revoked or expired key, or source IP not allowed | Re-sign with a fresh timestamp + nonce; check the clock, key status, and source IP |
| `403`  | Scope insufficient (e.g. a `read` key trying to trade)                                | Enroll a `trade`-scoped key                                                        |
| `404`  | No matching resource — often no on-chain exchange account yet                         | Create an on-chain account with collateral, then retry                             |
| `429`  | Rate limited                                                                          | Back off (1s / 2s / 4s) and retry                                                  |
{% endstep %}
{% endstepper %}

## Next steps

* **Place an order.** Orders are submitted over the trading WebSocket (`/ws/v1/trading`): authenticate with a signed `ApiKeySignIn` frame, then send `OrderRequest` frames. See [Placing orders over WebSocket](file:///2362779/api/websocket.md).
* **Use the SDK.** For a higher-level Rust client that tracks exchange state and builds orders for you, see the [SDK Quickstart](file:///2362779/sdk/quickstart.md).
* **Explore the endpoints.** Browse the full [REST API reference](file:///2362779/api/rest.md) and [WebSocket reference](file:///2362779/api/websocket.md).
