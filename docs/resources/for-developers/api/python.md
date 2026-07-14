# Python

This guide ports the Perpl API — the REST (Representational State Transfer, i.e. HTTPS) endpoints and the WebSocket (WSS) streams — to idiomatic Python. Every request is signed with an API key, so the bulk of this page is a small, reusable signing layer plus copy-pasteable examples for the public data, authenticated history, and trading flows.

The signing scheme, endpoints, and message types are identical across languages; only the tooling differs. Where a detail is specific to Python (a library choice, a request-encoding gotcha) it is called out explicitly.

> **Note:** Perpl authenticates programmatic clients with **API keys**. An API key is an **Ed25519 key pair** (Ed25519 is a public-key signature algorithm). The server stores only the **public** key; the private key never leaves your machine. There is no bearer token or session cookie — **every request is signed with the private key**. Withdrawals and transfers-out are never permitted via an API key, regardless of scope.

## Prerequisites

Install three third-party packages:

| Package            | Used for                                    |
| ------------------ | ------------------------------------------- |
| `requests`         | REST calls over HTTPS                       |
| `websocket-client` | WebSocket streams (imported as `websocket`) |
| `pynacl`           | Ed25519 signing                             |

```bash
pip install requests websocket-client pynacl
```

> **Note:** `pynacl` is used throughout this page for Ed25519. If you already depend on the `cryptography` package you can swap it in — both accept a 32-byte private-key seed. See [Loading the key](python.md#loading-the-key) for the one-line difference.

You also need an **enrolled API key**. Create one in the web UI (connect your wallet at `https://app.perpl.xyz/apikeys` for mainnet or `https://testnet.perpl.xyz/apikeys` for testnet); the UI hands you the `X-API-Key` token and the Ed25519 private key. Programmatic enrollment is also possible — see [Enrolling a key programmatically](python.md#enrolling-a-key-programmatically).

> **Note:** An API key authorizes _API access_. It does **not** create an on-chain trading account. Trading also requires an exchange account created on-chain with initial collateral; some authenticated calls return `404` until that account exists.

## Configuration

The examples read the network and key material from environment variables, matching the reference clients. Full endpoint, contract, and market values for both networks are on the [Networks](file:///2362779/getting-started/networks.md) page — reuse those rather than hard-coding your own.

| Variable               | Meaning                                         | Mainnet default             |
| ---------------------- | ----------------------------------------------- | --------------------------- |
| `PERPL_API_URL`        | REST base URL (**includes** `/api`)             | `https://app.perpl.xyz/api` |
| `PERPL_WS_URL`         | WebSocket base URL (**no** `/api`)              | `wss://app.perpl.xyz`       |
| `PERPL_CHAIN_ID`       | Chain ID (`143` mainnet, `10143` testnet)       | `143`                       |
| `PERPL_API_KEY`        | The opaque `X-API-Key` token from enrollment    | —                           |
| `PERPL_API_KEY_SECRET` | The Ed25519 private key, hex of the 32-byte key | —                           |

```python
import os

API_URL = os.environ.get("PERPL_API_URL", "https://app.perpl.xyz/api")
WS_URL = os.environ.get("PERPL_WS_URL", "wss://app.perpl.xyz")
CHAIN_ID = int(os.environ.get("PERPL_CHAIN_ID", "143"))
API_KEY = os.environ["PERPL_API_KEY"]
API_KEY_SECRET = os.environ["PERPL_API_KEY_SECRET"]
```

> **Note:** The chain ID is part of the signed canonical string, so you must sign with the value for the network you are calling. A signature built for chain `143` is rejected on testnet (chain `10143`).

## The signing primitive

Both REST and WebSocket authentication sign a **canonical string** — a fixed set of fields joined by newline (`\n`) — with the Ed25519 private key, then base64url-encode the 64-byte signature **without padding**. These helpers are shared by every signed call.

### Loading the key

The private key arrives as hex of the 32-byte seed (with or without a `0x` prefix):

```python
from nacl.signing import SigningKey

def load_signing_key(secret_hex: str) -> SigningKey:
    seed = bytes.fromhex(secret_hex.removeprefix("0x"))
    if len(seed) != 32:
        raise ValueError(f"expected a 32-byte Ed25519 seed, got {len(seed)} bytes")
    return SigningKey(seed)

signing_key = load_signing_key(API_KEY_SECRET)
```

> **Note:** With the `cryptography` package instead of `pynacl`, this becomes `Ed25519PrivateKey.from_private_bytes(seed)`, and signing is `key.sign(message)` (which already returns the 64-byte signature directly).

### Encoding helpers

```python
import base64
import hashlib
import os
import time

def b64url_nopad(data: bytes) -> str:
    """base64url, no padding — the encoding Perpl expects for signatures and nonces."""
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode("ascii")

def new_nonce() -> str:
    """Fresh, single-use, client-random nonce per request."""
    return b64url_nopad(os.urandom(16))

def timestamp_ms() -> str:
    """Unix epoch milliseconds, decimal string."""
    return str(int(time.time() * 1000))

def sign_canonical(signing_key: SigningKey, canonical: str) -> str:
    signature = signing_key.sign(canonical.encode("utf-8")).signature  # 64 bytes
    return b64url_nopad(signature)
```

## Signing REST requests

A REST canonical string is **six fields joined by `\n`**:

```
<chain_id>
<HTTP_METHOD>          e.g. GET, POST
<request-target>       path + query string, byte-for-byte as sent, e.g. /v1/trading/fills?count=100
<timestamp_ms>         unix epoch milliseconds, decimal
<nonce>                client-random, base64url (no padding)
<sha256(body) hex>     hex of SHA-256 over the raw body ("" body -> sha256 of the empty string)
```

Send four headers alongside the request:

| Header            | Value                                           |
| ----------------- | ----------------------------------------------- |
| `X-API-Key`       | the opaque token from enrollment                |
| `X-API-Timestamp` | the `timestamp_ms` used in the canonical string |
| `X-API-Nonce`     | the `nonce` used in the canonical string        |
| `X-API-Signature` | `base64url(ed25519 signature)`                  |

```python
import requests

def signed_request(method: str, target: str, body: str = "") -> requests.Response:
    """
    `target` is the path + query string exactly as it will appear in the URL,
    e.g. "/v1/trading/fills?count=100". It is signed byte-for-byte, so build it
    yourself and do NOT pass a separate `params=` dict (see the note below).
    """
    ts = timestamp_ms()
    nonce = new_nonce()
    body_bytes = body.encode("utf-8")
    body_hash = hashlib.sha256(body_bytes).hexdigest()

    canonical = "\n".join([str(CHAIN_ID), method, target, ts, nonce, body_hash])
    signature = sign_canonical(signing_key, canonical)

    headers = {
        "X-API-Key": API_KEY,
        "X-API-Timestamp": ts,
        "X-API-Nonce": nonce,
        "X-API-Signature": signature,
    }
    if body:
        headers["Content-Type"] = "application/json"

    return requests.request(
        method,
        API_URL + target,
        headers=headers,
        data=body_bytes if body else None,
    )

# Example: read your most recent fill
resp = signed_request("GET", "/v1/trading/fills?count=1")
print(resp.status_code, resp.json())
```

> **Note:** **The `request-target` must match byte-for-byte what the server receives.** In Python this means you must build the query string yourself and sign that exact string — do **not** hand `requests` a `params={...}` dict, because `requests` may re-encode or reorder parameters, breaking the signature. Build the target with `urllib.parse.urlencode` once, sign it, and request that same URL. The pagination helper below shows the pattern.

## Public data (no auth)

Public endpoints require no signature. Fetch the global context (chain, instances, tokens, market configs) and candles with plain `requests` calls.

### Context and market scaling

Prices and sizes on the wire are **scaled integers**. Divide by `10 ** price_decimals` / `10 ** size_decimals` (read from each market's config in the context) to get human-readable values; leverage is in hundredths.

```python
def get_context() -> dict:
    resp = requests.get(f"{API_URL}/v1/pub/context")
    resp.raise_for_status()
    return resp.json()  # {chain, instances, tokens, markets}

def market_scales(context: dict) -> dict[int, dict]:
    """market_id -> {'price_decimals', 'size_decimals'} from MarketConfig."""
    out = {}
    for m in context["markets"]:
        cfg = m["config"]
        out[m["id"]] = {
            "price_decimals": cfg["price_decimals"],
            "size_decimals": cfg["size_decimals"],
        }
    return out
```

Scaling round-trips, using BTC on mainnet (`price_decimals = 1`, `size_decimals = 5`):

```python
def to_scaled(value: float, decimals: int) -> int:
    return round(value * (10 ** decimals))

def from_scaled(scaled: int, decimals: int) -> float:
    return scaled / (10 ** decimals)

to_scaled(95000, 1)    # 950000  ($95,000 price)
to_scaled(0.1, 5)      # 10000   (0.1 BTC size)

# Leverage is stored in hundredths
lev_hundredths = 10 * 100   # 1000 == 10x
```

### Candles

The candles endpoint takes a `market_id`, a resolution in seconds, and a `from-to` millisecond range. A maximum of **1024 candles** are returned per request. Supported resolutions (seconds): `60, 300, 900, 1800, 3600, 7200, 14400, 28800, 43200, 86400`.

```python
def get_candles(market_id: int, resolution: int, hours: int = 24) -> list[dict]:
    to_ms = int(time.time() * 1000)
    from_ms = to_ms - hours * 60 * 60 * 1000
    url = f"{API_URL}/v1/market-data/{market_id}/candles/{resolution}/{from_ms}-{to_ms}"
    resp = requests.get(url)
    resp.raise_for_status()
    series = resp.json()

    scales = market_scales(get_context())
    pscale = 10 ** scales[market_id]["price_decimals"]
    return [
        {
            "time": c["t"],
            "open": c["o"] / pscale,
            "high": c["h"] / pscale,
            "low": c["l"] / pscale,
            "close": c["c"] / pscale,
            "volume": float(c["v"]),
            "trades": c["n"],
        }
        for c in series["d"]
    ]

# 1-hour BTC candles for the last 24 hours (mainnet market_id = 1)
btc_candles = get_candles(1, 3600, 24)
```

## Authenticated history

The trading-history endpoints are all `GET`, signed with the four `X-API-*` headers, and paginated. Response shape is `{"d": [...newest to oldest...], "np": "<next cursor>"}`. Pass `count` (default 50, **max 100**) and `page` (the `np` cursor from the previous response). Server-side filtering by market or date is **not** supported — filter client-side.

| Endpoint                           | Returns                                            |
| ---------------------------------- | -------------------------------------------------- |
| `GET /v1/trading/fills`            | Order fill history                                 |
| `GET /v1/trading/order-history`    | Historical order events                            |
| `GET /v1/trading/position-history` | Position history                                   |
| `GET /v1/trading/account-history`  | Account events (deposits, settlements, funding, …) |
| `GET /v1/profile/ref-code`         | Your referral code (`404` + empty if none)         |

```python
from urllib.parse import urlencode

def fetch_all(path: str, count: int = 100) -> list[dict]:
    """Page through a signed trading-history endpoint, newest to oldest."""
    rows: list[dict] = []
    cursor: str | None = None
    while True:
        # Build the exact query string, then sign that exact target.
        params = {"count": str(count)}
        if cursor:
            params["page"] = cursor
        target = f"{path}?{urlencode(params)}"

        resp = signed_request("GET", target)
        resp.raise_for_status()
        page = resp.json()

        rows.extend(page["d"])
        cursor = page.get("np")
        if not cursor:
            break
    return rows

all_fills = fetch_all("/v1/trading/fills")
positions = fetch_all("/v1/trading/position-history", count=50)
```

`AccountEvent.et` (event type) values on `account-history`: `1` Deposit, `2` Withdrawal, `3` IncreasePositionCollateral, `4` Settlement, `5` Liquidation, `6` TransferToProtocol, `7` TransferFromProtocol, `8` Funding, `9` Deleveraging, `10` Unwinding, `11` PositionCollateralDecreased, `12` LastForwardedDescIdReset.

## Rate limits and error handling

Limits are approximate; back off on HTTP `429`.

| Type                                          | Limit                       |
| --------------------------------------------- | --------------------------- |
| REST public (`/v1/pub/*`, market data)        | \~100 req/min               |
| REST authenticated (profile, trading history) | \~60 req/min                |
| WebSocket messages                            | \~50 msg/sec per connection |
| WebSocket connections                         | \~5 per IP                  |

```python
def signed_request_retrying(method: str, target: str, body: str = "", max_retries: int = 3):
    delay = 1.0  # seconds; exponential backoff 1s / 2s / 4s
    for attempt in range(max_retries + 1):
        resp = signed_request(method, target, body)
        if resp.status_code != 429 or attempt == max_retries:
            return resp
        time.sleep(delay)
        delay *= 2
    return resp
```

REST status codes and what to do:

| Status | Meaning                                                                               | Action                                                                                    |
| ------ | ------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `400`  | Bad request                                                                           | Fix the request shape                                                                     |
| `401`  | Bad/stale signature, replayed nonce, revoked/expired key, or IP not in the allow-list | Re-sign with a fresh timestamp + nonce; check the system clock, key status, and source IP |
| `403`  | Scope insufficient (e.g. a `read` key placing an order)                               | Use a `trade`-scoped key                                                                  |
| `404`  | Not found (e.g. no exchange account, or no referral code)                             | Verify the on-chain account exists                                                        |
| `429`  | Too many requests                                                                     | Exponential backoff (1s / 2s / 4s)                                                        |
| `500`  | Internal server error                                                                 | Retry with backoff                                                                        |

> **Note:** The signature timestamp must be within **±30 seconds** of server time, and each `nonce` is single-use within that window. A clock that drifts more than 30 seconds produces `401` on every request — keep the client clock in sync (NTP).

## WebSocket: market data

The `/ws/v1/market-data` endpoint needs no authentication. Subscribe with a `SubscriptionRequest` (`mt: 5`) carrying a `subs` list of `{stream, subscribe}`. Available streams: `heartbeat@<chain_id>`, `gas-stats@<chain_id>`, `market-config@<chain_id>`, `market-state@<chain_id>`, `funding@<chain_id>`, `candles@<market_id>*<resolution>`, `order-book@<market_id>`, `trades@<market_id>`.

Every message carries a header: `mt` (message type), optional `sid` (subscription id), `sn` (sequence), `cid` (correlation id), `ses` (session id).

```python
import json
import websocket  # from the websocket-client package

def stream_order_book(market_id: int):
    def on_open(ws):
        ws.send(json.dumps({
            "mt": 5,  # SubscriptionRequest
            "subs": [{"stream": f"order-book@{market_id}", "subscribe": True}],
        }))

    def on_message(ws, raw):
        msg = json.loads(raw)
        if msg["mt"] in (15, 16):  # 15 L2BookSnapshot, 16 L2BookUpdate
            # Each price level is {p: price, s: size, o: order_count}.
            # On updates, a level with o == 0 has been removed.
            for lvl in msg.get("bid", []):
                print("BID", lvl["p"], lvl["s"], lvl["o"])
            for lvl in msg.get("ask", []):
                print("ASK", lvl["p"], lvl["s"], lvl["o"])

    ws = websocket.WebSocketApp(
        f"{WS_URL}/ws/v1/market-data",
        on_open=on_open,
        on_message=on_message,
    )
    ws.run_forever()

stream_order_book(1)  # BTC on mainnet
```

Relevant server → client message types: `9` MarketStateUpdate, `10` MarketFundingUpdate, `11`/`12` Candles snapshot/update, `15`/`16` L2 (level-2) book snapshot/update, `17`/`18` Trades snapshot/update, `100` Heartbeat (carries `sn` and `h` = latest head block).

## WebSocket: trading

The `/ws/v1/trading` endpoint requires authentication.

{% stepper %}
{% step %}
## Open the socket

Open the trading socket.
{% endstep %}

{% step %}
## Send the sign-in frame first

Send a signed `ApiKeySignIn` (`mt: 29`) frame **as the very first message**.
{% endstep %}

{% step %}
## Receive the initial snapshots

Receive snapshots: `WalletSnapshot` (`mt: 19`), `OrdersSnapshot` (`mt: 23`), `PositionsSnapshot` (`mt: 26`).
{% endstep %}

{% step %}
## Track sequence numbers

Seed sequence tracking from the `WalletSnapshot` `sn`; every `Heartbeat` (`mt: 100`) must be `sn + 1` (a gap forces a reconnect).
{% endstep %}

{% step %}
## Keep the connection alive

Send an application-level `Ping` (`mt: 1`) roughly every 30 seconds to keep the connection alive.
{% endstep %}
{% endstepper %}

The `mt: 29` signature covers a **four-field** canonical string joined by `\n`: `<chain_id>`, the literal tag `trading-ws-signin`, `<timestamp_ms>`, `<nonce>`.

> **Note:** A `read`-scoped key connects and receives all snapshots and updates, but its `OrderRequest` frames are rejected. Placing orders needs a `trade`-scoped key. WebSocket close code **3401** means authentication failed — reconnect and re-send a fresh signed `mt: 29` frame.

The client below authenticates, tracks the head block and the order sequence, keeps the connection alive on a background thread, and exposes order-placement methods.

```python
import json
import threading
import time
import websocket

class TradingClient:
    def __init__(self, api_key: str, signing_key, on_update=None):
        self.api_key = api_key
        self.signing_key = signing_key
        self.on_update = on_update or (lambda kind, data: None)
        self.ws: websocket.WebSocketApp | None = None
        self.account_id: int | None = None
        self.current_block = 0
        self._last_sn: int | None = None
        self._rq = 0            # last request id we have used locally
        self._stop = False

    # --- connection -------------------------------------------------------
    def connect(self):
        self._stop = False
        self.ws = websocket.WebSocketApp(
            f"{WS_URL}/ws/v1/trading",
            on_open=self._on_open,
            on_message=self._on_message,
            on_close=self._on_close,
            on_error=self._on_error,
        )
        threading.Thread(target=self.ws.run_forever, daemon=True).start()

    def _on_open(self, ws):
        ts = timestamp_ms()
        nonce = new_nonce()
        canonical = "\n".join([str(CHAIN_ID), "trading-ws-signin", ts, nonce])
        signature = sign_canonical(self.signing_key, canonical)
        ws.send(json.dumps({
            "mt": 29,  # ApiKeySignIn — must be the first frame
            "chain_id": CHAIN_ID,
            "api_key": self.api_key,
            "timestamp": ts,
            "nonce": nonce,
            "signature": signature,
        }))
        threading.Thread(target=self._keep_alive, daemon=True).start()

    def _keep_alive(self):
        while not self._stop and self.ws and self.ws.sock and self.ws.sock.connected:
            try:
                self.ws.send(json.dumps({"mt": 1, "t": int(time.time() * 1000)}))
            except Exception:
                break
            time.sleep(30)

    def _on_close(self, ws, code, msg):
        self._stop = True
        if code == 3401:
            # Auth failure: reconnect and re-send a fresh signed mt:29 frame.
            print("trading WS auth failed (3401), reconnecting")
        # Add your own backoff/reconnect policy here.

    def _on_error(self, ws, err):
        print("trading WS error:", err)

    # --- inbound messages -------------------------------------------------
    def _on_message(self, ws, raw):
        msg = json.loads(raw)
        mt = msg["mt"]
        if mt == 19:    # WalletSnapshot
            accounts = msg.get("as") or []
            if accounts:
                self.account_id = accounts[0]["id"]
                # Seed the request-id counter from the account's lfr.
                self._rq = max(self._rq, accounts[0].get("lfr", 0))
            self._last_sn = msg.get("sn")
            self.on_update("wallet", msg)
        elif mt == 21:  # AccountUpdate — lfr may advance here too
            self._rq = max(self._rq, msg.get("lfr", 0))
            self.on_update("account", msg)
        elif mt == 23:  # OrdersSnapshot
            self.on_update("orders", msg.get("d"))
        elif mt == 24:  # OrdersUpdate
            self.on_update("order_update", msg.get("d"))
        elif mt == 25:  # FillsUpdate
            self.on_update("fills", msg.get("d"))
        elif mt == 26:  # PositionsSnapshot
            self.on_update("positions", msg.get("d"))
        elif mt == 27:  # PositionsUpdate
            self.on_update("position_update", msg.get("d"))
        elif mt == 100:  # Heartbeat
            sn = msg.get("sn")
            if self._last_sn is not None and sn != self._last_sn + 1:
                print("sequence gap, reconnecting")
                self.ws.close()   # your _on_close should trigger a reconnect
                return
            self._last_sn = sn
            self.current_block = msg.get("h", self.current_block)

    # --- outbound orders --------------------------------------------------
    def _next_rq(self) -> int:
        # rq must be strictly increasing; the server rejects rq <= lfr (sr: 32).
        self._rq += 1
        return self._rq

    def open_long(self, market_id: int, size: int, price: int | None, leverage: int) -> int:
        order = {
            "mt": 22,                       # OrderRequest
            "rq": self._next_rq(),
            "mkt": market_id,
            "acc": self.account_id,
            "t": 1,                         # OpenLong
            "p": price or 0,                # 0 == market
            "s": size,                      # size, scaled by size_decimals
            "fl": 0 if price else 4,        # GoodTillCancel (GTC) for limit, ImmediateOrCancel (IOC) for market
            "lv": leverage * 100,           # hundredths (10x -> 1000)
            "lb": self.current_block + 100, # last valid block
        }
        self.ws.send(json.dumps(order))
        return order["rq"]

    def cancel_order(self, market_id: int, order_id: int) -> int:
        order = {
            "mt": 22,
            "rq": self._next_rq(),
            "mkt": market_id,
            "acc": self.account_id,
            "oid": order_id,
            "t": 5,                         # Cancel
            "s": 0,
            "fl": 0,
            "lv": 0,
            "lb": self.current_block + 100,
        }
        self.ws.send(json.dumps(order))
        return order["rq"]


# Usage
client = TradingClient(API_KEY, signing_key, on_update=lambda kind, data: print(kind, data))
client.connect()

# After the snapshots have arrived (account_id + current_block populated):
time.sleep(2)
rq = client.open_long(market_id=1, size=10000, price=None, leverage=10)  # 0.1 BTC market long, 10x
print("submitted order request", rq)
```

### Request IDs, idempotency, and retries

`rq` (Request ID) is an idempotency key scoped per account; the server guarantees **at-most-once** execution per `rq`. It must be **strictly increasing**. The server tracks the last processed value as `lfr` (last forwarded request id), delivered on `WalletSnapshot` (`mt: 19`) and `AccountUpdate` (`mt: 21`).

* On connect, seed a local counter from `account.lfr`.
* For each order, `rq = max(local_counter, account.lfr) + 1`.
* Submitting `rq <= lfr` fails with `sr: 32` (`OrderDescIdTooLow`).

| Scenario                                                                  | Action                                                             |
| ------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| No status received yet, `lb` (last valid block) not expired               | Retry with the **same** `rq`                                       |
| `sr: 32` (`OrderDescIdTooLow`)                                            | Retry **once** with a new `rq` (common with multiple clients/tabs) |
| Head block ≥ `lb`, no status received, and no reconnections since posting | Retry with a **new** `rq`                                          |

Multiple updates can arrive for one `rq`. Deduplicate on the client: take the first non-failure status (`st` in `2,3,4,5,8,9,10`) as definitive and ignore everything after it; if only failures (`st: 7`) arrive, process the first one only.

### Order fields and enums

`OrderRequest` (`mt: 22`) fields:

| Field        | Meaning                                                                             |
| ------------ | ----------------------------------------------------------------------------------- |
| `rq`         | Request ID (idempotency key, strictly increasing)                                   |
| `mkt`        | Market ID                                                                           |
| `acc`        | Account ID                                                                          |
| `oid`        | Order ID (for modify / cancel)                                                      |
| `t`          | Order type (see below)                                                              |
| `p`          | Limit price, scaled (`0` = market)                                                  |
| `s`          | Size, scaled by `size_decimals`                                                     |
| `a`          | Amount (for collateral increase)                                                    |
| `ms`         | Max market-order slippage, basis points (bps)                                       |
| `tif`        | Time-in-force (also defined; `lb` is the operative field for order validity/expiry) |
| `fl`         | Flags (see below)                                                                   |
| `tp` / `tpc` | Trigger price and condition                                                         |
| `tr`         | Linked trigger request ID                                                           |
| `lp`         | Linked position ID                                                                  |
| `lv`         | Leverage in hundredths (`1000` = 10x)                                               |
| `lb`         | Last execution block                                                                |

| Order type (`t`) |                            | Flag (`fl`) |                         | Trigger condition (`tpc`) |         |
| ---------------- | -------------------------- | ----------- | ----------------------- | ------------------------- | ------- |
| `1`              | OpenLong                   | `0`         | GoodTillCancel (GTC)    | `1`                       | GTELast |
| `2`              | OpenShort                  | `1`         | PostOnly                | `2`                       | LTELast |
| `3`              | CloseLong                  | `2`         | FillOrKill (FOK)        | `3`                       | GTEMark |
| `4`              | CloseShort                 | `4`         | ImmediateOrCancel (IOC) | `4`                       | LTEMark |
| `5`              | Cancel                     |             |                         |                           |         |
| `6`              | IncreasePositionCollateral |             |                         |                           |         |
| `7`              | Change                     |             |                         |                           |         |

> **Note:** Trigger orders (a take-profit / stop-loss, TP/SL) must set `lb: 0` — they have no expiry block, and the server manages their lifecycle. `tp` + `tpc` fire when the market last/mark price crosses the trigger; `tr` links the trigger to another request; `lp` links it to a position (cancelled when the position closes or inverts).

Order-reject reasons are delivered as `sr` (OrderStatusReason) on order updates — a `0`–`68` enum. Common values: `1` AmountExceedsAvailableBalance, `13` CrossesBook, `14` ExceedsLastExecutionBlock, `15` ForwardingReverted, `32` OrderDescIdTooLow, `38` OrderSizeExceedsAvailableSize, `53` PerpetualInsolvent.

## Enrolling a key programmatically

Most integrations create keys in the web UI. Programmatic enrollment is a one-time, wallet-authorized flow over two endpoints:

{% stepper %}
{% step %}
## Generate an Ed25519 keypair locally

Send the public key as raw 32 bytes, `0x`-hex.
{% endstep %}

{% step %}
## Request an enrollment payload

`POST /api/v1/api-key/payload` → returns an EIP-712 `typed_data` payload plus an opaque `mac`.
{% endstep %}

{% step %}
## Sign the payload twice

Produce **two** signatures over `typed_data`: (a) a wallet secp256k1 EIP-712 signature proving account ownership, and (b) an Ed25519 **proof-of-possession** over the digest `keccak256(0x1901 ‖ domainSeparator ‖ hashStruct(message))`.
{% endstep %}

{% step %}
## Submit the enrollment request

`POST /api/v1/api-key/enroll` (echo `typed_data` + `mac`, send `signature` + `pop_signature`) → returns `ApiKeyInfo`, whose `api_key.api_key` is the opaque `X-API-Key` token. **Store it — it is not re-derivable.**
{% endstep %}
{% endstepper %}

Key facts for the flow:

* **Scope bitmask** (`scope_mask`, uint32): `1` = read (`1 << 0`), `2` = trade (`1 << 1`, implies read), `3` = both.
* **Origin whitelisting**: both enroll endpoints are CORS-enabled and the request `Origin` must be pre-whitelisted by Perpl; a non-browser client must set `Origin` explicitly.
* **Delegated accounts**: set `target_profile` to the delegated account address; `address` stays the signing wallet (owner/operator). Delegation is validated on-chain at enrollment.
* **Enroll error codes**: `404` target profile not found; `409` public key already registered (revoked keys cannot be re-enrolled — use a fresh keypair); `423` per-profile key limit reached (**max 16 active keys**).
* Listing and revoking keys is done in the web UI (`/apikeys`), not via the API.

> **TODO(author):** verify a full Python enrollment example end-to-end. The source docs provide only a JavaScript reference (`examples/js/enroll_api_key.js`); a Python port needs a secp256k1 EIP-712 signer (e.g. `eth-account`) plus a hand-built proof-of-possession digest (`keccak256(0x1901 ‖ domainSeparator ‖ hashStruct(message))`). The exact request/response JSON field names for `POST /api/v1/api-key/payload` and `POST /api/v1/api-key/enroll` are not enumerated in the API reference and should be confirmed against `examples/js/enroll_api_key.js` before publishing runnable code.

## Next steps

* [Networks & Configuration](file:///2362779/getting-started/networks.md) — every endpoint, contract address, chain ID, and market ID for both networks.
* [REST API](file:///2362779/api/rest.md) — full endpoint reference and response types.
* [WebSocket API](file:///2362779/api/websocket.md) — all message types, streams, and trading-flow semantics.
* [Authentication](file:///2362779/api/authentication.md) — the signing scheme in full, including scopes and validity rules.
