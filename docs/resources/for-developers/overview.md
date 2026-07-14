# Overview

Perpl is an **isolated-margin perpetual-futures decentralized exchange (DEX)** that runs on the [Monad](https://monad.xyz/) blockchain. A _perpetual future_ ("perp") is a derivatives contract that tracks the price of an underlying asset (for example BTC or ETH) with no expiry date. You post collateral, open a long or short position with leverage, and the position stays open until you close it or it is liquidated.

This page introduces the core concepts you need before you write any code, then walks through the two ways to integrate with Perpl:

* [**Direct REST + WebSocket API**](#two-ways-to-integrate) — call the HTTP and WebSocket endpoints from any language.
* [**Rust SDK**](#two-ways-to-integrate) — use the `perpl-sdk` crate and the `perpl-cli` command-line tool.

## Isolated margin

Perpl uses **isolated margin**, not cross margin. This is the single most important concept to understand before trading:

* **Each position has its own dedicated collateral deposit.**
* **Your account balance does&#x20;**_**not**_**&#x20;back your open positions.** Free balance sitting in your account is never automatically pulled in to save a position that is moving against you.
* If a position's own margin is exhausted, that position is liquidated on its own — it cannot cascade into your other positions or your account balance.

> **Note:** If you are coming from an exchange that uses cross margin, this behavior will surprise you. A position can be liquidated even while your account holds ample free balance. To add margin to a position, you must explicitly increase that position's collateral — the account balance will not do it for you.

This model prevents one losing position from cascading into the rest of your account.

## Markets

Each tradable perp is identified by a numeric `market_id`. **Market IDs differ between mainnet and testnet**, so always confirm which network you are targeting.

**Mainnet** (Chain ID `143`):

| `market_id` | Symbol |
| ----------- | ------ |
| 1           | BTC    |
| 10          | MON    |
| 20          | ETH    |
| 31          | SOL    |
| 40          | HYPE   |
| 50          | ZEC    |

**Testnet** (Chain ID `10143`):

| `market_id` | Symbol |
| ----------- | ------ |
| 16          | BTC    |
| 32          | ETH    |
| 48          | SOL    |
| 64          | MON    |
| 256         | ZEC    |

> **Note:** On mainnet, SOL was relisted as perp **31** on 2026-07-06. The legacy SOL perp **30** only appears in historical / on-chain data during migration — use **31** for the active SOL market.

Per-market parameters (price and size scaling, fees, leverage limits) are served live by the API. Fetch them from the public context endpoint (`GET /v1/pub/context`) rather than hard-coding them — see the [API Guide](overview.md#where-to-go-next).

## Collateral (AUSD)

Positions and account balances on Perpl are denominated in **AUSD (Agora Dollar)**, an ERC-20 stablecoin with **6 decimals**. All collateral amounts are integers scaled by `10^6` (so `100000000` = 100.0 AUSD).

| Network | Collateral token    | Address                                      | Decimals |
| ------- | ------------------- | -------------------------------------------- | -------- |
| Mainnet | AUSD (Agora Dollar) | `0x00000000eFE302BEAA2b3e6e1b18d08D69a9012a` | 6        |
| Testnet | aUSD                | `0xa9012a055bd4e0eDfF8Ce09f960291C09D5322dC` | 6        |

To trade you must hold an on-chain **exchange account**, created by depositing collateral into the Exchange contract. Authenticating with the API is _not_ the same as having an exchange account — the two are separate steps, covered in the [Quickstart](overview.md#where-to-go-next).

| Network | Exchange contract                            | RPC URL                         |
| ------- | -------------------------------------------- | ------------------------------- |
| Mainnet | `0x34B6552d57a35a1D042CcAe1951BD1C370112a6F` | `https://rpc.monad.xyz`         |
| Testnet | `0x1964C32f0bE608E7D29302AFF5E61268E72080cc` | `https://testnet-rpc.monad.xyz` |

The full network reference lives in [Networks](networks-and-configuration.md).

## Two ways to integrate

{% tabs %}
{% tab title="Path A: REST + WebSocket API" %}
The API has two channels:

| Channel        | Protocol | Purpose                                  | Auth   |
| -------------- | -------- | ---------------------------------------- | ------ |
| REST           | HTTPS    | History queries, authentication, profile | Varies |
| WebSocket (WS) | WSS      | Real-time data and trading               | Varies |

Base URLs:

| Network | REST base URL                   | WebSocket URL             |
| ------- | ------------------------------- | ------------------------- |
| Mainnet | `https://app.perpl.xyz/api`     | `wss://app.perpl.xyz`     |
| Testnet | `https://testnet.perpl.xyz/api` | `wss://testnet.perpl.xyz` |

**Public market data requires no authentication.** For example, fetch the chain and market configuration:

```typescript
const API_URL = process.env.PERPL_API_URL || 'https://app.perpl.xyz/api';

// Fetch context (markets, tokens, chain config)
const context = await fetch(`${API_URL}/v1/pub/context`)
  .then(r => r.json());

console.log(context.markets); // Available markets
console.log(context.chain);   // Chain configuration
```

Subscribe to a real-time stream over the market-data WebSocket:

```typescript
const WS_URL = process.env.PERPL_WS_URL || 'wss://app.perpl.xyz';

const ws = new WebSocket(`${WS_URL}/ws/v1/market-data`);

ws.onopen = () => {
  // Subscribe to the BTC order book (market_id=1 on mainnet)
  ws.send(JSON.stringify({
    mt: 5, // MsgTypeSubscriptionRequest
    subs: [{ stream: 'order-book@1', subscribe: true }]
  }));
};

ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  console.log('Message type:', msg.mt);
};
```

**Authenticated calls** (order history, trading) use an **API key**: an Ed25519 key pair that you enroll once with a one-time wallet signature, then use to sign every request. There is no session cookie or bearer token. The signing scheme and the full endpoint list are covered in the [API Guide](overview.md#where-to-go-next).

This path works from **any language** — JavaScript, TypeScript, Python, Rust, or anything that can make HTTPS and WebSocket calls.
{% endtab %}

{% tab title="Path B: Rust SDK" %}
If you build in Rust, the `perpl-sdk` workspace gives you typed access to exchange state and events, plus `perpl-cli` for reading and tracing the exchange from the command line.

| Crate       | Purpose                                                                                                  |
| ----------- | -------------------------------------------------------------------------------------------------------- |
| `perpl-sdk` | SDK types for building and maintaining an in-memory cache of exchange state, plus order-posting helpers. |
| `perpl-cli` | Command-line tool for reading and tracing exchange state and events.                                     |

Requirements: **Rust `>= 1.85.0`** (edition 2024). Add the SDK as a path dependency in your `Cargo.toml`:

```toml
[dependencies]
perpl-sdk = { path = "../dex-sdk/crates/sdk" }
```

The `Chain` type carries the per-network configuration (chain ID, Exchange address, collateral token, and the active perpetual list). Built-in constructors cover both networks:

```rust
use perpl_sdk::Chain;

let chain = Chain::mainnet(); // or Chain::testnet()

println!("chain_id      = {}", chain.chain_id());       // 143
println!("exchange      = {}", chain.exchange());       // 0x34B6…12a6F
println!("collateral    = {}", chain.collateral_token());
println!("perpetuals    = {:?}", chain.perpetuals());   // [1, 10, 20, 31, 40, 50]
```

To browse the generated API reference:

```bash
cargo doc -p perpl-sdk --no-deps --open
```

> **Note:** The SDK currently sources events via log polling and does not yet process funding events. See the [SDK Guide](overview.md#where-to-go-next) for the current module map, streaming model, and limitations.
{% endtab %}
{% endtabs %}

## Where to go next

| Guide                                     | What it covers                                                                    |
| ----------------------------------------- | --------------------------------------------------------------------------------- |
| [Quickstart](quickstart.md)               | Create an exchange account, deposit collateral, and place your first order.       |
| [Networks](networks-and-configuration.md) | Full mainnet and testnet reference: URLs, chain IDs, contract addresses, tokens.  |
| [API Guide](api/)                         | REST endpoints, WebSocket streams, and API-key authentication (any language).     |
| [SDK Guide](sdk/)                         | The `perpl-sdk` crate and `perpl-cli`: state cache, event streams, order posting. |
