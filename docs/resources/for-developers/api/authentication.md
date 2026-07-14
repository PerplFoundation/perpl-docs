# Authentication

Perpl authenticates programmatic clients — bots, trading terminals, and scripts — with **API keys**. An API key is an **Ed25519 key pair** (Ed25519 is a public-key signature algorithm). The server only ever stores the **public** key; the private key never leaves your machine. There is no bearer token and no session cookie to leak: **every request is signed with the key's private key**.

This page covers how to obtain a key (web UI or programmatic enrollment) and how to sign each REST (Representational State Transfer, i.e. HTTPS) and WebSocket request with it.

{% hint style="info" %}
The signing scheme includes the network's numeric **chain ID**, so you must sign with the value for the network you are calling. The two live networks are Mainnet (chain ID `143`, default) and Testnet (chain ID `10143`). Full endpoint, contract, and market details are on the [Networks](../networks-and-configuration.md) page — reuse those values rather than hard-coding your own.
{% endhint %}

| Network           | Chain ID | REST base URL                   | WebSocket URL             |
| ----------------- | -------- | ------------------------------- | ------------------------- |
| Mainnet (default) | `143`    | `https://app.perpl.xyz/api`     | `wss://app.perpl.xyz`     |
| Testnet           | `10143`  | `https://testnet.perpl.xyz/api` | `wss://testnet.perpl.xyz` |

{% hint style="info" %}
The REST base URL includes the `/api` suffix; the WebSocket URL does **not**.
{% endhint %}

The snippets below read the key material and network from environment variables:

| Variable               | Meaning                                          |
| ---------------------- | ------------------------------------------------ |
| `PERPL_API_URL`        | REST base URL (e.g. `https://app.perpl.xyz/api`) |
| `PERPL_WS_URL`         | WebSocket base URL (e.g. `wss://app.perpl.xyz`)  |
| `PERPL_CHAIN_ID`       | Chain ID (`143` mainnet, `10143` testnet)        |
| `PERPL_API_KEY`        | The opaque `X-API-Key` token from enrollment     |
| `PERPL_API_KEY_SECRET` | The Ed25519 private key, hex of the 32-byte key  |

## API keys and scopes

Every key carries a **scope**, chosen at enrollment. Scope is stored as a 32-bit bitmask (`scope_mask`). Trade implies read.

| `scope_mask` | Name               | Grants                                                          |
| ------------ | ------------------ | --------------------------------------------------------------- |
| `1`          | `read` (`1 << 0`)  | Read account, order, position, history, and points/rewards data |
| `2`          | `trade` (`1 << 1`) | Place / cancel / modify orders (**implies read**)               |
| `3`          | `read \| trade`    | Both                                                            |

{% hint style="info" %}
**Withdrawals and transfers-out are never permitted via an API key, regardless of scope.** Those actions require a wallet signature on the Exchange contract directly.
{% endhint %}

### API auth is not the same as an exchange account

**This is a common source of confusion.** Enrolling and authenticating an API key authorizes _API access_ for your wallet. It does **not** create an on-chain trading account.

| Concept                | What it means                                                       | Required for                                                                 |
| ---------------------- | ------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| **API authentication** | An enrolled API key can call authenticated API endpoints            | Reading order history, position history, connecting to the trading WebSocket |
| **Exchange account**   | An on-chain account exists on the Exchange contract with collateral | Placing orders, holding positions, trading                                   |

To trade you must additionally create an on-chain account with initial collateral by calling `createAccount(uint256 amountCNS)` on the Exchange contract. Until that account exists, some authenticated calls return **404**. See the [Networks](../networks-and-configuration.md) page for the Exchange contract address per network, and the account-creation walkthrough for the `cast` commands.

## Creating a key

### Web UI

Connect your wallet and create a key at:

* **Mainnet** — [app.perpl.xyz/apikeys](https://app.perpl.xyz/apikeys)
* **Testnet** — [testnet.perpl.xyz/apikeys](https://testnet.perpl.xyz/apikeys)

The UI hands you the opaque `X-API-Key` token and the Ed25519 private key. Store both — the token is not re-derivable.

{% hint style="info" %}
Listing and revoking keys is done from the web UI (`/apikeys`), **not** via the API. A revoked public key cannot be re-enrolled — generate a fresh key pair.
{% endhint %}

### Programmatic enrollment

Third-party integrations can enroll keys directly on behalf of a user's wallet. Enrollment is a one-time, wallet-authorized flow across two endpoints:

| Method | Path                      | Auth             | Purpose                           |
| ------ | ------------------------- | ---------------- | --------------------------------- |
| `POST` | `/api/v1/api-key/payload` | Wallet signature | Get the EIP-712 payload to sign   |
| `POST` | `/api/v1/api-key/enroll`  | Wallet signature | Enroll the key, receive the token |

EIP-712 is the Ethereum standard for signing typed structured data. The flow is:

{% stepper %}
{% step %}
## Generate an Ed25519 key pair

The public key is sent as raw 32 bytes, `0x`-hex encoded.

```typescript
import * as ed from '@noble/ed25519';

const privateKey = ed.utils.randomPrivateKey();            // 32 bytes, keep secret
const publicKey  = await ed.getPublicKeyAsync(privateKey); // 32 bytes
const publicKeyHex = '0x' + Buffer.from(publicKey).toString('hex');
```

Or with OpenSSL 3.x:

```bash
openssl genpkey -algorithm ed25519 -out apikey.pem
# raw 32-byte public key as 0x-hex:
PUBKEY_HEX=0x$(openssl pkey -in apikey.pem -pubout -outform DER | tail -c 32 | xxd -p -c 64)
```
{% endstep %}

{% step %}
## Request the enrollment payload

`POST /api/v1/api-key/payload` with an `ApiKeyPayloadRequest` body:

```typescript
interface ApiKeyPayloadRequest {
  chain_id: number;        // 143 mainnet, 10143 testnet
  address: string;         // signer wallet (owner or operator of the account)
  public_key: string;      // Ed25519 public key, 0x-hex (32 bytes)
  scope_mask: number;      // 1=read, 2=trade, 3=both
  label: string;           // human-readable key label (required)
  expires_at?: number;     // ms timestamp, 0 / omitted = never
  ip_cidrs?: string[];     // optional IP allow-list (max 4 CIDRs)
  target_profile?: string; // delegated account, if enrolling for one
}
```

It returns an `ApiKeyPayloadResponse`:

```typescript
interface ApiKeyPayloadResponse {
  typed_data: any;  // EIP-712 typed data — sign this exactly as returned
  mac: string;      // opaque; echo back unchanged in the enroll request
}
```

```typescript
const API_URL = process.env.PERPL_API_URL || 'https://app.perpl.xyz/api';
const ORIGIN = 'https://your-app.example';  // must be whitelisted by Perpl

const payloadRes = await fetch(`${API_URL}/v1/api-key/payload`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Origin': ORIGIN,  // set from a server; in a browser the Origin is set automatically
  },
  body: JSON.stringify({
    chain_id: 143,
    address: '0xUserWalletAddress',
    public_key: publicKeyHex,
    scope_mask: 3,
    label: 'my trading terminal',
  }),
});
const { typed_data, mac } = await payloadRes.json();
```

{% hint style="info" %}
To enroll a key for a **delegated account** (an operator acting for another profile), set `target_profile` to the delegated account address. `address` stays the signing wallet (owner or operator); the server resolves the principal and validates the delegation on-chain at enrollment.
{% endhint %}
{% endstep %}

{% step %}
## Sign and enroll

Enrollment requires **two** signatures over the returned `typed_data`:

1. **Wallet signature** — the wallet's secp256k1 EIP-712 signature (secp256k1 is the elliptic curve used by Ethereum wallets). Proves the user owns or operates the account.
2. **Proof-of-possession** — an Ed25519 signature by the API private key over the EIP-712 digest `keccak256(0x1901 ‖ domainSeparator ‖ hashStruct(message))`. Proves you hold the private key for the public key being enrolled.

```typescript
import { ethers } from 'ethers';
import * as ed from '@noble/ed25519';

// Illustrative only: an inline private key stands in for the signer here.
// Integrations are expected to sign with the user's CONNECTED wallet (e.g. a
// browser wallet, or a wagmi/viem/ethers signer) — the private key never
// touches your code. Any EIP-712 signer works.
const wallet = new ethers.Wallet('0xUserWalletPrivateKey');

// ethers wants the EIP-712 types WITHOUT the EIP712Domain entry.
const { EIP712Domain, ...types } = typed_data.types;

// 1. Wallet secp256k1 EIP-712 signature (from the user's connected wallet).
const signature = await wallet.signTypedData(typed_data.domain, types, typed_data.message);

// 2. Ed25519 proof-of-possession over the EIP-712 digest.
const digest = ethers.TypedDataEncoder.hash(typed_data.domain, types, typed_data.message);
const popSig = await ed.signAsync(ethers.getBytes(digest), privateKey);
const popSignature = '0x' + Buffer.from(popSig).toString('hex');
```

`POST /api/v1/api-key/enroll` with an `ApiKeyEnrollRequest` body:

```typescript
interface ApiKeyEnrollRequest {
  chain_id: number;
  address: string;
  typed_data: any;        // echoed from the payload response, unchanged
  mac: string;            // echoed from the payload response, unchanged
  signature: string;      // wallet EIP-712 signature, 0x-hex
  pop_signature: string;  // Ed25519 proof-of-possession, 0x-hex
  target_profile?: string;
}
```

The response is an `ApiKeyInfo`. Its `api_key` field is the opaque `X-API-Key` token — **store it, it is not re-derivable.**

```typescript
interface ApiKeyInfo {
  api_key: string;       // opaque X-API-Key token
  address: string;
  scope_mask: number;
  label: string;
  ip_cidrs: string[];
  origin: string;        // HTTP Origin the key was enrolled from
  expires_at: number;    // ms, 0 = never
  last_used_at: number;  // ms, 0 = never
  created_at: number;    // ms
}
```

```typescript
const enrollRes = await fetch(`${API_URL}/v1/api-key/enroll`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Origin': ORIGIN,  // same whitelisted Origin as the payload request
  },
  body: JSON.stringify({
    chain_id: 143,
    address: '0xUserWalletAddress',
    typed_data,
    mac,
    signature,
    pop_signature: popSignature,
  }),
});
const { api_key } = await enrollRes.json();
const API_KEY = api_key.api_key; // the X-API-Key token — hand this to the request signer
```

**Enroll status codes:**

| Code | Meaning                                                                                  |
| ---- | ---------------------------------------------------------------------------------------- |
| 404  | Target profile not found                                                                 |
| 409  | Public key already registered (revoked keys can't be re-enrolled — use a fresh key pair) |
| 423  | Per-profile key limit reached (max **16** active keys)                                   |
{% endstep %}
{% endstepper %}

## Signing REST requests

Sign a **canonical string** with the key's private key and send four headers. No cookies, no bearer token.

The canonical string is these **six fields joined by `\n`** (a single newline between each):

```
<chain_id>            e.g. 143 (mainnet) or 10143 (testnet)
<HTTP_METHOD>         e.g. GET, POST
<request-target>      path + query string exactly as sent, e.g. /v1/trading/fills?count=100
<timestamp_ms>        unix epoch milliseconds, decimal
<nonce>               client-random, base64url (no padding)
<sha256(body) hex>    hex of SHA-256 over the raw request body ("" body -> SHA-256 of the empty string)
```

The signature is `base64url(ed25519_sign(privateKey, canonical))` — base64url encoded, **no padding** — sent in the `X-API-Signature` header. (SHA-256 is a cryptographic hash; base64url is the URL-safe base64 alphabet.)

Four headers are required on every authenticated request:

| Header            | Value                                                |
| ----------------- | ---------------------------------------------------- |
| `X-API-Key`       | The opaque token from enrollment                     |
| `X-API-Timestamp` | The same `timestamp_ms` used in the canonical string |
| `X-API-Nonce`     | The same `nonce` used in the canonical string        |
| `X-API-Signature` | `base64url(ed25519 signature)`                       |

```typescript
import { createHash, randomBytes } from 'crypto';
import * as ed from '@noble/ed25519';

const API_URL = process.env.PERPL_API_URL || 'https://app.perpl.xyz/api';
const CHAIN_ID = Number(process.env.PERPL_CHAIN_ID) || 143;
const API_KEY = process.env.PERPL_API_KEY;
const privateKey = Buffer.from((process.env.PERPL_API_KEY_SECRET ?? '').replace(/^0x/, ''), 'hex');

async function signedRequest(method: string, target: string, body = '') {
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
const res = await signedRequest('GET', '/v1/trading/fills?count=1');
console.log(await res.json());
```

{% hint style="info" %}
The `request-target` must match byte-for-byte what the server receives — include the full query string (`?count=100&page=...`) exactly as sent, in the same order. Any mismatch changes the canonical string and the signature will be rejected.
{% endhint %}

### Worked example

Suppose you are calling Mainnet (`chain_id = 143`) with a `GET` on `/v1/trading/fills?count=1`, an empty body, at `timestamp_ms = 1751932800000`, with `nonce = 9Nq0Yp3kZ2c1aVb7` (base64url, no padding). The SHA-256 of the empty body is the well-known constant `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`.

The six fields assemble into this exact byte sequence (each line separated by a single `\n`):

```
143
GET
/v1/trading/fills?count=1
1751932800000
9Nq0Yp3kZ2c1aVb7
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

You then sign that exact string with the Ed25519 private key and base64url-encode (no padding) the result to produce `X-API-Signature`. The request carries the four headers:

```http
GET /api/v1/trading/fills?count=1 HTTP/1.1
Host: app.perpl.xyz
X-API-Key: <your opaque token>
X-API-Timestamp: 1751932800000
X-API-Nonce: 9Nq0Yp3kZ2c1aVb7
X-API-Signature: <base64url(ed25519 signature over the canonical string)>
```

{% hint style="info" %}
`X-API-Timestamp` and `X-API-Nonce` must be the **same** values you placed into the canonical string. Re-generate both (and re-sign) for every request.
{% endhint %}

## WebSocket authentication

The trading WebSocket lives at `/ws/v1/trading`. Authenticate by sending an `ApiKeySignIn` frame (message type `mt: 29`) as the **first** message after the socket opens.

The signature covers a WebSocket canonical string — **four fields joined by `\n`**:

```
<chain_id>
trading-ws-signin      literal action tag
<timestamp_ms>
<nonce>
```

```typescript
import { randomBytes } from 'crypto';
import * as ed from '@noble/ed25519';

const CHAIN_ID = Number(process.env.PERPL_CHAIN_ID) || 143;
const API_KEY = process.env.PERPL_API_KEY;
const privateKey = Buffer.from((process.env.PERPL_API_KEY_SECRET ?? '').replace(/^0x/, ''), 'hex');

const ts = Date.now().toString();
const nonce = randomBytes(16).toString('base64url');
const canonical = [CHAIN_ID, 'trading-ws-signin', ts, nonce].join('\n');
const sig = await ed.signAsync(Buffer.from(canonical), privateKey);

ws.onopen = () => {
  ws.send(JSON.stringify({
    mt: 29,                 // MsgTypeApiKeySignIn
    chain_id: CHAIN_ID,
    api_key: API_KEY,
    timestamp: ts,
    nonce,
    signature: Buffer.from(sig).toString('base64url'),
  }));
};
```

A `trade`-scoped key may place orders over the socket. A `read`-scoped key still receives snapshots and updates, but `OrderRequest` frames are rejected with status `403`.

{% hint style="info" %}
The market-data WebSocket (`/ws/v1/market-data`) requires no authentication — connect and subscribe directly.
{% endhint %}

## Signature validity

| Rule                 | Detail                                                                                                     |
| -------------------- | ---------------------------------------------------------------------------------------------------------- |
| **Timestamp window** | `X-API-Timestamp` must be within **±30 seconds** of server time. Keep the client clock in sync (NTP).      |
| **Nonce**            | Single-use within the validity window — generate a fresh random `nonce` per request. Replays are rejected. |
| **Expiry**           | Requests are rejected once the key is past its `expires_at`.                                               |
| **IP allow-list**    | When an `ip_cidrs` allow-list is set (max 4 CIDRs), requests from an IP outside it are rejected.           |

## Errors

### HTTP status codes

| Code | Meaning                                                                                                                | Recommended action                                                                 |
| ---- | ---------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| 200  | Success                                                                                                                | —                                                                                  |
| 400  | Bad Request                                                                                                            | Check the request shape / body                                                     |
| 401  | Unauthorized — missing/invalid headers, bad or stale signature, replayed nonce, revoked/expired key, or IP not allowed | Re-sign with a fresh timestamp + nonce; check the clock, key status, and source IP |
| 403  | Forbidden — scope insufficient (e.g. a `read` key attempting to trade)                                                 | Enroll a `trade`-scoped key                                                        |
| 404  | Not Found — including when no on-chain exchange account exists yet                                                     | Create an exchange account with `createAccount()`                                  |
| 429  | Too Many Requests                                                                                                      | Back off (see rate limits)                                                         |
| 500  | Internal Server Error                                                                                                  | Retry with backoff                                                                 |

### WebSocket close code

| Code | Meaning                | Action                                                               |
| ---- | ---------------------- | -------------------------------------------------------------------- |
| 3401 | Authentication failure | Re-send a fresh signed `ApiKeySignIn` (`mt: 29`) frame and reconnect |

## Next steps

* [Networks](../networks-and-configuration.md) — endpoints, chain IDs, contract and collateral addresses, market IDs.
* [REST Endpoints](rest.md) — the full list of HTTP endpoints and which require a signed request.
* [WebSocket](websocket.md) — real-time streams, subscription frames, and order placement over the trading socket.
