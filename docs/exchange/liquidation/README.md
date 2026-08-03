# Liquidation

Liquidation occurs when the position value falls below the maintenance margin requirement. Liquidations are crucial for maintaining the protocol's functionality and protecting other traders.

## Example: a $100k BTC long at 10x

You post **$10,000** to open a **$100,000** position (10x leverage).

* A 10% drop would wipe out your full $10,000.
* But you are liquidated _before_ that — at a **6% drop, when BTC reaches $94,000**.

At $94,000 your remaining margin is **$4,000**: your $10,000 deposit minus the $6,000 the move cost you. That surviving **$4,000 — the residual — is split**: **$3,200 (80%) is returned to you**, and **$400 (10%) each to the insurance fund and the protocol**.

Because you are stopped _before_ your margin reaches zero, the amount you keep is 80% of whatever remains at that moment.

> **Why a 6% drop and not 10%?** BTC's **maintenance margin is 4%** of the position. Liquidation triggers when your remaining margin falls to that level — not when it reaches zero. Higher leverage posts less initial margin, so your liquidation sits closer to the current price; the exact distance for your position is shown live in the Adjust Leverage dialog (see [Margin](../margin.md)).

## Maintenance margin by market

Your liquidation point is set by each market's **maintenance margin** — the minimum margin a position must keep before it is liquidated. It is a fixed protocol parameter (you do not choose it) and it varies by market:

| Market | Maintenance margin | Maximum leverage |
| ------ | ------------------ | ---------------- |
| BTC    | 4%                 | 15x              |
| ETH    | 5%                 | 12x              |
| SOL    | 5%                 | 12x              |
| MON    | 5%                 | 10x              |
| HYPE   | 5%                 | 10x              |
| ZEC    | ~6.7%              | 8x               |

A 4% maintenance margin means a position is liquidated once its remaining margin falls to 4% of the position's value. Maximum leverage shown is the base maximum; initial margin requirements are dynamic and can rise — lowering the effective maximum — for larger positions or in volatile conditions (see [Margin](../margin.md)).

Here is how liquidation works on Perpl:

1. The liquidation engine identifies undercollateralized positions (positions where the value is at or beneath the maintenance margin)
2. The liquidation engine attempts to close the position on the order book.
3. The proceeds from the position closing due to liquidation are then distributed as follows:
   1. 80% to the original position holder
   2. 10% to the perpetual's insurance fund
   3. 10% to the protocol

All liquidation events will be recorded on-chain. To learn more about liquidation, read our blog on [understanding liquidation in perp markets.](https://blog.perpl.xyz/understanding-liquidations-in-perp-markets/)

> **Note on liquidation PnL:** The PnL shown for a liquidated position is the _would-be_ PnL had you closed at the exit price. Your actual balance change is larger, because liquidation proceeds are distributed (see below). You retain 80% of the remaining margin, not all of it. Always check your balance history for the realized impact.

<figure><img src="../../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

### Order Book Liquidations

Order book liquidations are initiated by the protocol; a position meeting the liquidation criteria is sold to the order book if possible, with the proceeds distributed to the original position holder, perpetual insurance fund, and protocol.

#### Invariant: Collateralization for liquidation:

<p align="center"><span class="math">0 &#x3C; FMV &#x3C;= MMR</span></p>

_FMV = Position fair market value._\
&#xNAN;_&#x4D;MR = Position Maintenance margin requirement._

#### Calculating Liquidation Price

A position’s liquidation price can be calculated as follows:

<p align="center"><span class="math">P_{Liquidation} = P_{Entry} + s \cdot \frac{C_{MMR} - C_{Deposit} - C_{Funding}}{L}</span></p>

Where:

* _P_<sub>_Entry_</sub> is the entry price of the position.
* _s_ is the side of the position or position type (if the position is long, then a mark price at or below the liquidation price indicates the position should be liquidated; the opposite is true for a short position).

<p align="center"><span class="math">s = \begin{cases} 1, &#x26; \text{long position}\\ -1, &#x26; \text {short position} \end{cases}</span></p>

* _C_<sub>_MMR_</sub> is the Maintenance Margin Requirement (MMR) of the position. (See note on calculating this value below.)
* _C_<sub>_Deposit_</sub> is the collateral deposited in the position.
* _C_<sub>_Funding_</sub> is the funding payment owed by the position (available in the smart contract as premium PNL).
* _L_ is the final liquidation lot size of the position.

The position’s MMR can be calculated as follows:

<p align="center"><span class="math">C_{MMR} = \frac{P_{Entry} * L}{C_{MMF}}</span></p>

Where:

* _C_<sub>_MMF_</sub> is the Maintenance Margin Factor (MMF) of the perpetual (`Perpetual::maintMarginFracHdths`).

The position’s bankruptcy price can be calculated similarly, except the Maintenance Margin Requirement value becomes zero (because the position is now bankrupt and has no value—it’s possible for a position to have a negative value at prices beneath the bankruptcy price, but this is clamped to zero in the auto-deleveraging process). The calculation of bankruptcy price is:

<p align="center"><span class="math">P_{Bankruptcy} = P_{Entry} - s \cdot \frac{C_{Deposit} + C_{Funding}}{L}</span></p>

#### Partial vs Full Liquidation

Depending on the position size, the crypto asset, and market conditions, the protocol may choose to perform a partial liquidation. This allows the trader to keep the trade open for a longer period. On the other hand, positions can be fully closed. In that scenario, the proceeds are split between the protocol, the insurance fund, and the trader. Both of these liquidations happen on the order book.

#### Liquidation Proceeds Distribution

When a position is liquidated, it is closed at the liquidation price; above the bankruptcy price, so residual margin typically remains. These proceeds are distributed between three parties:

| Recipient      | Share |
| -------------- | ----- |
| Trader (you)   | 80%   |
| Protocol       | 10%   |
| Insurance fund | 10%   |

The trader retains the majority of any margin left after the position is closed. The protocol and insurance fund shares cover liquidation costs and backstop solvency (see Insurance & ADL).

Distribution is configured per perpetual, so the exact split may vary by market. The values above are the current default.

#### Buy to Liquidate

In certain scenarios, when the position to be liquidated is too large to close out on the book, the PLP vault can buy the position to liquidate gradually.
