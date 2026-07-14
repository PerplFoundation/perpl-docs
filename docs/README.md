# Introduction

### What is Perpl?

Perpl is a high-performance, decentralized, perpetual futures (perps) exchange handcrafted for Monad EVM. Monad is an open, permissionless, geographically distributed blockchain (L1) that provides fast finality and 100% EVM-compatibility.&#x20;

Perps just need a price oracle and a stablecoin for margin. They allow traders and speculators to take positions in crypto assets with leverage. Unlike traditional derivatives, they don't have an expiration, which fosters deeper liquidity in markets. See our blog on [perpetual future contracts](https://blog.perpl.xyz/what-are-perps-perpetual-futures-contracts/).

### Why another Perp DEX?

Centralized exchanges (CEX) and decentralized exchanges (DEX) are the two types of exchanges that offer perps. CEXs have good UX, but have onerous KYC/AML requirements and custodial risk. DEXs are self-custodial and enable global, permissionless trading, but they have relied on slower, decentralized infrastructure. See our blog on the differences between [decentralized vs. centralized exchanges](https://blog.perpl.xyz/decentralized-vs-centralized-exchanges/).

The early DEX era was dominated by AMMs, not because they were better, but because L1 infrastructure couldn’t support performant order books. Teams eventually pivoted to “app-chains” to run central limit order books (CLOBs). However, they couldn’t enter new markets as quickly as CEXs because they relied on external MMs. See our blog on the differences between [AMMs vs. CLOBs](https://blog.perpl.xyz/order-books-vs-automated-market-makers-amms/).

<figure><img src=".gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

Pooled liquidity vaults, pioneered by GMX (GLP), provide liquidity to AMM-based markets with relatively simple strategies. However, CLOB-based perp DEXs require active liquidity management. This is where Hyperliquid's HLP changed the game for bootstrapping CLOB-based perp DEXs.

However, there are centralized points of failure and other risks when a single team operates the exchange, vault, and blockchain. We believe the optimal combination is a performant on-chain DEX built on a standalone high-throughput L1, and bootstrapped by independently operated vaults.&#x20;

### True Decentralization, Optimal UX

The holy grail of DeFi is **a fully on-chain perp DEX with CEX-like trading experience on EVM**. Perpl, built on the Monad L1, is positioned to achieve this ideal while honoring the boundaries between protocol, liquidity, and infrastructure.

Unlike other DEXs on L1s that use AMMs or off-chain matching, everything on Perpl happens on-chain. The exchange, matching, and settlement runs **entirely on-chain with no off‑chain or centralized points of failure**. See our blog on [on-chain perps](https://blog.perpl.xyz/onchain-perps-balancing-profit-vs-peril/).

<figure><img src=".gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

Building a DEX is hard; building a DEX on a blockchain you don't control/operate is even harder. However, we're building Perpl to be anti-fragile, scalable, and to leverage the benefits of composability. We have hyper-optimized the exchange on a specific vector: <100k gas for market-maker post+cancel. Every GWEI of gas matters because more gas costs = fewer market maker updates = wider quotes = worse fills for the end-user.

<figure><img src=".gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>
