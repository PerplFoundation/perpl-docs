# Price Indices

## Price Indices

Perpl works with three price quantities. Understanding how each is produced explains how your positions are valued, when they can be liquidated, and how funding is charged.

| Price                | What it is                                                      | Where it comes from                                                              |
| -------------------- | --------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| **Spot Index Price** | The fair spot price of the underlying asset                     | Chainlink Data Streams, pushed on-chain                                          |
| **Mark Price**       | A robust estimate of the fair _perpetual_ price                 | Computed off-chain from several sources, then anchored tightly to the Spot Index |
| **Funding Rate**     | The periodic payment that keeps the perpetual trading near spot | Computed off-chain from the order book relative to the Spot Index                |

***

### Spot Index Price

The **Spot Index Price** (also called the _oracle price_) is the reference spot price of the underlying asset. It is sourced from **Chainlink Data Streams** — low-latency, cryptographically signed price reports — with one feed per market.

The pricing service writes a fresh Spot Index on-chain whenever either of the following is true:

* the price has moved more than **0.1%** since the last on-chain value, or
* the on-chain value is within **10 seconds** of its maximum permitted age (so it never goes stale).

The Spot Index has three jobs:

1. **Anchor for the Mark Price.** The Mark Price is held within a tight band around the Spot Index (see below).
2. **Settlement of funding.** Funding payments are calculated against the Spot Index, not the Mark Price.
3. **Staleness guardrail.** Settlement and liquidation are rejected by the smart contract if the on-chain Spot Index is older than its maximum age, protecting users from trading against a frozen oracle.

***

### Mark Price

The **Mark Price** is an unbiased, robust estimate of the fair price of the perpetual. It is the price the protocol uses for:

* unrealized profit and loss (PnL),
* additional collateral requirements when a position is opened or increased if its PnL is negative,
* triggering liquidation and auto-deleveraging,
* the realized price in force-close and unwind operations.

#### How it is computed

The Mark Price is recomputed every block from up to **four independent inputs**, combined with a **median** so that no single source can dominate:

1. **External price** — the weighted median of mid prices from major venues (**Binance, Hyperliquid, OKX, Bybit**) for the corresponding perpetual.
2. **Basis-adjusted fair value** — the Spot Index scaled by the recent average _basis_ (the premium at which the perpetual trades over spot on those venues): `(1 + average_basis) × spot_index`. The basis is a smoothed (exponential moving average) measure and is itself clamped to a small range.
3. **Impact mid price** — the midpoint of the volume-weighted average prices (VWAP) obtained by walking Perpl's own order book to a set of notional depths (**$1,000 / $2,000 / $5,000**).
4. **Book price** — the median of the best bid, the best ask, and the last traded price (the last trade is dropped if it is too old).

In symbols, writing $$P_{\text{spot}}$$ for the Spot Index Price:

$$
P_{\text{median}} = \operatorname{median}\bigl(P_{\text{ext}},\; P_{\text{fair}},\; P_{\text{impact}},\; P_{\text{book}}\bigr)
$$

where the basis-adjusted fair value (input 2) is

$$
P_{\text{fair}} = \bigl(1 + \overline{b}\,\bigr)\,P_{\text{spot}},
\qquad
b_i = \operatorname{EMA}\!\left(\frac{P_i^{\text{mid}}}{P_{\text{spot}}} - 1\right)
$$

and $$\overline{b}$$ is the clamped, weighted average of the per-venue basis values $$b_i$$.

If fewer than four of these inputs are fresh, the median is backstopped first by a smoothed order-book price and, if necessary, by the raw Spot Index, so a Mark Price is always available even in thin or quiet conditions.

#### The Spot Index clamp (most important)

After the median is taken, the result is **currently clamped to within ±0.25% (25 basis points) of the Spot Index** before it is published on-chain. In production this band is **25 bps on every market**:

$$
P_{\text{mark}} = \operatorname{clamp}\bigl(P_{\text{median}},\; (1-\delta)\,P_{\text{spot}},\; (1+\delta)\,P_{\text{spot}}\bigr),
\qquad \delta = 25\ \text{bps} = 0.0025
$$

In other words: however the order book or external venues move, the on-chain Mark Price **never sits more than a quarter of one percent away from the Chainlink Spot Index.** This keeps the Mark Price an accurate, manipulation-resistant reflection of spot and bounds how far it can be pulled by a single noisy input.

The Mark Price is written on-chain whenever it moves more than **0.05%** from the last published value, or when the on-chain value is close to expiring. The smart contract independently rejects any proposed Mark Price that falls outside its configured tolerance of the Spot Index; the ±0.25% clamp keeps every published value comfortably inside that tolerance.

***

### Funding Rate

The **Funding Rate** is the mechanism that keeps the perpetual price aligned with the underlying spot price. When the perpetual trades above spot, longs pay shorts; when it trades below, shorts pay longs.

#### How it is computed

Perpl uses an **impact-premium** method. Throughout each funding interval, every few seconds the system measures how far the order book is from the Spot Index:

$$
f_t = \frac{\max\bigl(P^{\text{impact}}_{\text{bid}} - P_{\text{spot}},\; 0\bigr) - \max\bigl(P_{\text{spot}} - P^{\text{impact}}_{\text{ask}},\; 0\bigr)}{P_{\text{spot}}}
$$

where $$P^{\text{impact}}_{\text{bid}}$$ _and_ $$P^{\text{impact}}_{\text{ask}}$$ are the VWAP execution prices for trading a fixed notional (**$1,000**) into the bid and ask sides of the book. The interval's funding rate $$F$$ is the **average of those samples** over the interval, then clamped to a maximum magnitude $$F_{\max}$$:

$$
F = \operatorname{clamp}\!\left(\frac{1}{k+1}\sum_{t=0}^{k} f_t,\; -F_{\max},\; +F_{\max}\right)
$$

#### How it is charged

The funding payment on a position is:

$$
\text{funding payment} = Q \cdot F \cdot P_{\text{spot}}
$$

where $$Q$$ is the position's lot size. Funding is charged against the **Spot Index**, _not_ the Mark Price. The funding interval is set by the protocol (approximately one hour).
