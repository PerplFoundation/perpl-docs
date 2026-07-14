# Networks & Configuration

Perpl runs on two networks: **Mainnet** (the default, live trading) and **Testnet** (for development and testing). Both are deployed on [Monad](https://docs.monad.xyz/). This page is the single reference for every endpoint, contract address, chain ID, and market ID you need to point a client at either network.

Everything a client needs is configurable through environment variables, so switching networks is a matter of swapping values — no code changes required.

{% hint style="info" %}
All contract addresses below are shown in EIP-55 checksummed form (mixed-case). They are case-insensitive on-chain, so a lowercase copy refers to the same contract.
{% endhint %}

## Configuration via environment variables

The reference clients read all URLs and chain settings from environment variables. Copy the example file and fill in the values for the network you want:

```bash
cp .env.example .env
```

| Variable                 | Mainnet default                              | Description                                             |
| ------------------------ | -------------------------------------------- | ------------------------------------------------------- |
| `PERPL_API_URL`          | `https://app.perpl.xyz/api`                  | REST (Representational State Transfer) API base URL     |
| `PERPL_WS_URL`           | `wss://app.perpl.xyz`                        | WebSocket base URL                                      |
| `PERPL_CHAIN_ID`         | `143`                                        | Chain ID                                                |
| `PERPL_RPC_URL`          | `https://rpc.monad.xyz`                      | RPC (remote procedure call) URL for on-chain operations |
| `PERPL_EXCHANGE_ADDRESS` | `0x34B6552d57a35a1D042CcAe1951BD1C370112a6F` | Exchange contract                                       |
| `PERPL_COLLATERAL_TOKEN` | `0x00000000eFE302BEAA2b3e6e1b18d08D69a9012a` | AUSD collateral token                                   |

To target testnet, set the same variables to their testnet values (see the table in the next section).

## Network reference

{% tabs %}
{% tab title="Mainnet" %}
| Field               | Value                                                            |
| ------------------- | ---------------------------------------------------------------- |
| REST base URL       | `https://app.perpl.xyz/api`                                      |
| WebSocket URL       | `wss://app.perpl.xyz`                                            |
| Chain ID            | `143`                                                            |
| Exchange contract   | `0x34B6552d57a35a1D042CcAe1951BD1C370112a6F`                     |
| RPC URL             | `https://rpc.monad.xyz`                                          |
| Collateral token    | AUSD (Agora Dollar) `0x00000000eFE302BEAA2b3e6e1b18d08D69a9012a` |
| Collateral decimals | `6`                                                              |
{% endtab %}

{% tab title="Testnet" %}
| Field               | Value                                             |
| ------------------- | ------------------------------------------------- |
| REST base URL       | `https://testnet.perpl.xyz/api`                   |
| WebSocket URL       | `wss://testnet.perpl.xyz`                         |
| Chain ID            | `10143`                                           |
| Exchange contract   | `0x1964C32f0bE608E7D29302AFF5E61268E72080cc`      |
| RPC URL             | `https://testnet-rpc.monad.xyz`                   |
| Collateral token    | aUSD `0xa9012a055bd4e0eDfF8Ce09f960291C09D5322dC` |
| Collateral decimals | `6`                                               |
{% endtab %}
{% endtabs %}

### Ready-to-use `.env` files

{% tabs %}
{% tab title="Mainnet" %}
```bash
PERPL_API_URL=https://app.perpl.xyz/api
PERPL_WS_URL=wss://app.perpl.xyz
PERPL_CHAIN_ID=143
PERPL_RPC_URL=https://rpc.monad.xyz
PERPL_EXCHANGE_ADDRESS=0x34B6552d57a35a1D042CcAe1951BD1C370112a6F
PERPL_COLLATERAL_TOKEN=0x00000000eFE302BEAA2b3e6e1b18d08D69a9012a
```
{% endtab %}

{% tab title="Testnet" %}
```bash
PERPL_API_URL=https://testnet.perpl.xyz/api
PERPL_WS_URL=wss://testnet.perpl.xyz
PERPL_CHAIN_ID=10143
PERPL_RPC_URL=https://testnet-rpc.monad.xyz
PERPL_EXCHANGE_ADDRESS=0x1964C32f0bE608E7D29302AFF5E61268E72080cc
PERPL_COLLATERAL_TOKEN=0xa9012a055bd4e0eDfF8Ce09f960291C09D5322dC
```
{% endtab %}
{% endtabs %}

## Collateral token & decimals

Both networks settle in a 6-decimal AUSD collateral token. **Raw on-chain amounts are integers scaled by 10^6** — divide the raw value by `1_000_000` (`1e6`) to get a human-readable USD figure, and multiply a USD figure by `1e6` to build a raw amount for a contract call.

```typescript
const COLLATERAL_DECIMALS = 6;

// Raw on-chain integer -> display USD
function rawToUsd(raw: bigint): number {
  return Number(raw) / 10 ** COLLATERAL_DECIMALS; // 100_000_000n -> 100.0
}

// Display USD -> raw integer for a contract call
function usdToRaw(usd: number): bigint {
  return BigInt(Math.round(usd * 10 ** COLLATERAL_DECIMALS)); // 100.0 -> 100_000_000n
}
```

{% hint style="info" %}
The minimum deposit to open an exchange account is returned by the public API as a raw integer (for example `min_account_open_amount: 100000000`, which is `100.0` AUSD). Always read the current minimum from `GET /api/v1/pub/context` or `getAccountCreationInfo()` rather than hard-coding it.
{% endhint %}

## Markets

Each market is identified by a numeric `market_id` (also called a perpetual ID on-chain). **Market IDs are network-specific** — the same asset has a different ID on mainnet than on testnet — so never share a hard-coded ID across networks.

{% tabs %}
{% tab title="Mainnet markets" %}
| `market_id` | Symbol |
| ----------- | ------ |
| 1           | BTC    |
| 10          | MON    |
| 20          | ETH    |
| 31          | SOL    |
| 40          | HYPE   |
| 50          | ZEC    |

{% hint style="info" %}
SOL was relisted on 2026-07-06. The active SOL market is `market_id = 31`. The legacy SOL market (`market_id = 30`) only appears in historical / on-chain data from the migration window; use `31` for all new integrations.
{% endhint %}
{% endtab %}

{% tab title="Testnet markets" %}
| `market_id` | Symbol |
| ----------- | ------ |
| 16          | BTC    |
| 32          | ETH    |
| 48          | SOL    |
| 64          | MON    |
| 256         | ZEC    |

{% hint style="info" %}
The market list can change as markets are added or delisted. Fetch the authoritative, current list at runtime from `GET /api/v1/pub/context` (see below) rather than relying on this table alone.
{% endhint %}
{% endtab %}
{% endtabs %}

## Fetching configuration at runtime

The public `context` endpoint returns the live chain and market configuration and requires no authentication. Read it once at startup to discover the current market set instead of hard-coding IDs.

```typescript
const API_URL = process.env.PERPL_API_URL || 'https://app.perpl.xyz/api';

const context = await fetch(`${API_URL}/v1/pub/context`).then((r) => r.json());

console.log(context.markets); // Available markets for this network
console.log(context.chain);   // Chain configuration
```

The same base URLs drive the WebSocket streams — connect to `${PERPL_WS_URL}/ws/v1/market-data` for public market data and `${PERPL_WS_URL}/ws/v1/trading` for authenticated trading.

## Using the network config in the Rust SDK

The `dex-sdk` `Chain` type carries the full per-network configuration. Use the built-in constructors for the standard networks:

```rust
use perpl_sdk::Chain;

// Mainnet: chain_id 143, Exchange 0x34B6...12a6F, perpetuals [1, 10, 20, 31, 40, 50]
let chain = Chain::mainnet();

// Testnet: chain_id 10143, Exchange 0x1964...80cc, perpetuals [16, 32, 48, 64, 256]
let chain = Chain::testnet();

// Read individual fields
let id: u64 = chain.chain_id();
let exchange = chain.exchange();
let collateral = chain.collateral_token();
let markets = chain.perpetuals();          // &[PerpetualId]
let deploy_block = chain.deployed_at_block();
```

For a non-standard deployment (for example a local test node), build a chain explicitly:

```rust
use perpl_sdk::Chain;
use alloy::primitives::address;

let chain = Chain::custom(
    /* chain_id           */ 20143,
    /* collateral_token   */ address!("0x0000000000000000000000000000000000000000"),
    /* deployed_at_block  */ 0,
    /* exchange           */ address!("0x0000000000000000000000000000000000000000"),
    /* perpetuals         */ vec![],
);
```

{% hint style="info" %}
`Chain::mainnet()` and `Chain::testnet()` also record the block at which the Exchange contract was deployed — `deployed_at_block` is `54773010` on mainnet and `62953` on testnet. Use it as the lower bound when scanning on-chain event logs, so a full history scan does not start from block 0.
{% endhint %}

## API authentication is separate from network selection

Choosing a network only tells your client _where_ to connect. It does not authenticate you, and it does not create an exchange account. Those are separate steps covered elsewhere in the docs:

* **API authentication** — sign requests with an enrolled API key (an Ed25519 key pair). See [Authentication](file:///2362779/api/authentication.md).
* **Exchange account** — an on-chain account created with collateral by calling `createAccount(uint256)` on the Exchange contract. See [Authentication](file:///2362779/api/authentication.md).

{% hint style="info" %}
A successful signed API request does **not** mean you have an exchange account. Some calls return `404` until an on-chain account exists for your wallet.
{% endhint %}
