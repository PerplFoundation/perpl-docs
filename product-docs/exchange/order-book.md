# Order Book

The order book is a real-time on-chain record of all open buy and sell orders for any given market.

* **Limit Orders**: Rest on the book until matched or canceled. Specified with price and quantity.
* **Market Orders**: Execute immediately against the best available orders in the book
* **Priority**: Orders are matched based on best price first, then earliest submission time
* **Visibility**: the order book is entirely on-chain, meaning anyone can verify the order state at any time without relying on a central operator.

#### Architecture

Our order book is the most gas-performant primitive achievable on EVM. A post-and-cancel order costs 100k gas, including solvency checks. For context, a simple Uniswap V2 AMM swap costs 200k gas.

The key design objective was to minimize gas for market maker post + cancel, as that is the most common operation. Perpl achieves O(1) constant time for post, O(1) constant time for cancel, and O(N) linear time for matching.

<figure><img src="../.gitbook/assets/image (4).png" alt=""><figcaption><p>Figure 1: Order book design consisting of two bit-index trees and a partition map list</p></figcaption></figure>

#### Price Levels

The order book is extremely granular, allowing for more than 16M price levels, so large assets like Bitcoin can be potentially quoted in dimes ($0.10) increments. One bit-index tree is a three-level, 256-bit-word index of prices that contain orders in the partition map list. It allows for quick discovery of the next price above or below a specified price in a gas-efficient manner.

<figure><img src="../.gitbook/assets/image (10).png" alt=""><figcaption><p>Figure 2: Bit-Index tree with 4-Bit words showing 64 price-levels mapped to start at $100.0 in increments of $0.5.</p></figcaption></figure>

#### Order IDs

The other bit-index tree is a two-level, 256-bit-word index of order IDs, allowing more than 65k orders per contract. It also functions as an order ID counter for the perpetual contract’s CLOB, assigning unique order IDs to each new order.&#x20;

To mitigate DDoS attacks on the order book, we have designed a permissioned order cancellation system. When the number of available orders falls below a given threshold, the backend can cancel orders that are a configurable distance from the maximum bid or minimum ask prices.&#x20;

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption><p>Figure 3: Order book bid and ask levels showing region where permissioned cancel is possible when number of orders exceeds permissionless cancel threshold.</p></figcaption></figure>

#### Time in Force: Expiry Blocks and Recycle Fees

For non-immediate orders, the time in force can be specified with an expiry block, so the order automatically expires after the specified block. Orders with an expiry block require the issuer to pay an order recycling fee. This fee is refunded if the order is completely filled or canceled by the issuer.

If the order has expired and another market participant encounters the expired order in a traversal of the order book, the recycling fee is paid to this participant as compensation for clearing the order. Recycling fees can be configured at run time and changed during operation.&#x20;

The ability to change recycling fees allows tuning the incentive to match current market conditions — higher recycling fees when many orders are impeding market transactions or when gas costs are high.

#### Change Order: Efficient Cancel-Post

Market makers can quote orders with tighter spreads—the distance between the bid and ask prices—when the cost of canceling and reposting orders is reduced. Tighter spreads are attractive because they mean that the DEX is better able to compete with centralized solutions on price.&#x20;

As prices move, market makers cancel existing orders and post new ones to follow price movements. This is inefficient in an on-chain exchange because it implies deallocation of storage for the canceled order and allocation of new storage for the newly posted order. Figure 1 above shows how order storage and order IDs can be shared across different price levels, from m+4 down to m-3.&#x20;

Shared storage allows order storage to be moved to effect a price change, rather than being inefficiently deallocated and subsequently reallocated. Rather than submitting two transactions, a single transaction, “Change”, replaces Cancel and Post transactions.&#x20;

If the market maker is only changing the lot size or expiry block of an order, then the operation uses less gas as it does not need to move the order storage to a new partition. The change operation would move the order to the end of the current price partition's list if the order's lot size is increased or the expiry block is changed (the order is not moved to the end of the list if the lot size is decreased).
