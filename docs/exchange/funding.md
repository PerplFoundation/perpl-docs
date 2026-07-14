# Funding

Funding rate is a feedback mechanism to force the perpetual price to follow the underlying asset price more closely. Funding rate is computed over a funding interval and then applied to position holders during a funding event. Funding events occur periodically on perpetual exchanges and are often described as long position holders paying funding payments to short position holders (or vice versa).

The direction of payment depends on whether the funding rate is positive (payment flows from long positions to short positions) or negative (payment flows in the opposite direction). Premium PNL is the cumulative funding payments paid and received by a position.

The settlement of these payments through centralized exchanges is near or at the funding event. Unfortunately, this is prohibitive for decentralized perpetual exchanges that are not running on an app-chain, where gas is not a consideration, or for custom op-codes that can be crafted to enable low-gas-use implementation of such payments. The following sections outline the funding rate mechanism that affects a position’s premium PNL. Two major challenges involved in implementing the mechanism are:

1. Computing the funding rate over the funding interval.
2. Settling funding payments for all positions at every funding event.

Both challenges are difficult on-chain due to their implications for gas usage.

### Computing Funding Rate <a href="#funding-rate-calculation" id="funding-rate-calculation"></a>

The method used by contemporary exchanges to compute the funding rate involves frequently computing an impact price on both sides of the order book for a specified notional amount. This operation is performed frequently during the funding interval, implying significant gas use that would have to be absorbed by users interacting with the contract, an off-chain keeper system, or both.

Alternative methodologies are being explored that could provide a similarly robust funding rate calculation, while being more gas-efficient on-chain. Key considerations are gas efficiency and resistance to attack vectors. The initial release of this decentralized perpetual exchange enables funding rate values, computed from transparent on-chain data, to be set at a fixed time prior to the funding event using a permissioned method.

The funding rate can be applied approximately once per hour. _Approximately_ because it is applied after a constant number of Monad blocks:

* Every 8571 blocks (assumes 0.42 second average consensus time)
* Can be set up to 143 blocks in advance (1 minute, assuming 0.42-second average consensus time). _Cannot be set after the funding event block._

The funding rate is set with two parameters:

* _C_<sub>_FundingRate_</sub> is the funding rate calculated over the current funding interval, as detailed below.
  * The funding rate is clamped in the contract such that its absolute value does not exceed `absFundingClampPctPer100k` , configurable abs. magnitude from \[0%, 15%]
* _P_<sub>_FundingPrice_</sub> is the funding price, essentially the spot market price feed value at the time the funding rate is set.
  * This value is constrained to be within the reference tolerance % of the Chainlink Oracle price. The Chainlink Oracle price must not be stale (i.e., its timestamp’s distance from the current block timestamp cannot exceed the maximum reference price age parameter).

The funding rate is computed throughout the funding interval as follows:

<p align="center"><span class="math">C_{FundingRate} = \sum_{n=0}^{k}{ \frac{C_{ImpactPriceDifference}(nT)}{P_{Oracle}(nT)}}</span></p>

Where:

* _n_ is the sample number within the funding interval.
* _T_ is the sample period (for example, 5 seconds on Binance).
* _k_ is the last sample in the funding interval (if the funding rate is set \~1 minute before the funding event, then \~11 samples would be ignored for _T=5_.

<p align="center"><span class="math">k = \frac{3600}{T}-1</span></p>

* _C_<sub>_ImpactPriceDifference_</sub> is the impact price difference at a given sample point. It is calculated as follows for values at a given sample point:

<p align="center"><span class="math">\begin{align*}C_{ImpactPriceDifference} = &#x26;\operatorname{max}(P_{ImpactBid}-P_{Oracle}, 0) \quad- \newline&#x26;\operatorname{max}(P_{Oracle} - P_{ImpactAsk}, 0) \end{align*}</span></p>

* _P_<sub>_ImpactBid_</sub> and _P_<sub>_ImpactAsk_</sub> are the resulting execution prices of trading a specified notional amount in $USD on the bid and ask sides of the order book, respectively.
  * The notional amount should be configurable and would be converted to a lot amount to execute the order—i.e., if ETH was trading at $4,000 and the notional amount was $2,000, then the prices would be computed for trades of 0.5 lots.

This methodology for computing the funding rate was described by [Hyperliquid](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/funding) and is congruent with the method described by [Binance](https://www.binance.com/en/support/faq/detail/360033525031). _However, importantly, the method herein does not model the cost difference in borrowing USD versus spot crypto._

### Settling Funding Payments <a href="#h_0041d9bda9-2" id="h_0041d9bda9-2"></a>

Explicitly settling funding payments on-chain for every funding event incurs prohibitive gas costs. This is because perpetual contracts can have hundreds to thousands of positions concurrently. During a funding event, each position’s account must transfer funds to or from the other position’s accounts according to the position size, funding rate, and funding event price. This implies hundreds to thousands of transfers per transaction, requiring significant state changes, which is prohibitively expensive.

To address funding issues, payments must be made virtually. This means that the effect of the funding payment settlement is visible after a single transaction, permitting all perpetual contract users to continue as though the payments had settled, without having to update the individual state of each of their positions in the contract. Virtualizing payments is further complicated because funding payments have a cumulative effect on a position held through multiple funding events.

Perpl debuts a novel solution to efficiently settle funding payments for all positions in a perpetual contract virtually. It is based on adapting the staking algorithm and virtual orders idea to eliminate the need for explicit periodic funding payment settlement.

#### A Virtualized Implicit Funding Event Payment Settlement Solution

To derive a virtualized implicit funding payment settlement solution, consider the following equation for funding payment:

<p align="center"><span class="math">F_{payment}[j] = P_i[j] · F_{rate}[j] · L</span></p>

_Frate\[j] = Funding rate for funding event at block number j._\
&#xNAN;_&#x50;_<sub>_i_</sub>_&#x20;=_ Funding rate price, which is the spot index price at the time of the funding event.\
&#xNAN;_&#x4C; = Position lot size._

The position lot size L, and side (long or short), remain constant throughout each funding event during which a position is held (a smart contract constraint forces the realization of funding payments when the position lot size or side is changed).

This means that for all positions, cumulative funding payment information across multiple funding events can be determined using superposition. Instead of storing funding rate and index price for each funding event and looking up each funding event a position has been held through, a funding product sum can be stored to reduce read operations.

The funding product is the product of the funding rate and index price for a particular funding event–\
Importantly, the funding product is independent of information specific to an individual position:

<p align="center"><span class="math">F_{product}[j] = P_i[j] · F_{rate}[j]</span></p>

_F_<sub>_product_</sub>_\[j] = Funding product for funding event at block number j._\
&#xNAN;_&#x50;_<sub>_i_</sub>_&#x20;=_ Funding rate price, which is the spot index price at the time of the funding event.\
&#xNAN;_&#x46;_<sub>_rate_</sub>_\[j] = Funding rate for funding event at block number j._\
&#xNAN;_&#x6A; = funding event block number._

Substituting equation F<sub>product</sub> into equation F<sub>payment</sub> yields the following equation for the funding payment:

<p align="center"><span class="math">F_{payment}[j] = (F_{product}[j]) · L</span></p>

_F_<sub>_product_</sub>_\[j] = Funding product for funding event at block number j._\
&#xNAN;_&#x4C; = Position lot size._

The funding product sum is the sum of all previous funding products. The funding product sum is causal\
and thus is defined as zero prior to the creation of the perpetual contract.

<p align="center"><span class="math">F_{sum}[j] = \begin{cases} 0, &#x26; j \leq i \\ \sum_{j=i}^{N} F_{product}[j], &#x26; j > i \end{cases}</span></p>

_F_<sub>_product_</sub>_\[j] = Funding product for funding event at block number j._\
&#xNAN;_&#x6A; = funding event block number._\
&#xNAN;_&#x69; = Perpetual contract creation block number._

A funding product of a particular block can be determined by subtracting any two adjacent funding sums, as shown in the equation below:

<p align="center"><span class="math">F_{product}[j] = F_{sum}[j] − F_{sum}[j − k]</span></p>

_F_<sub>_sum_</sub>_\[j] = Funding product sum for funding event._\
&#xNAN;_&#x46;_<sub>_sum_</sub>_\[j − k] = Funding product sum for previous funding event._\
&#xNAN;_&#x6A; = funding event block number._\
&#xNAN;_&#x6B; = The number of blocks in a funding interval._

Substituting equation F<sub>product</sub> into equation F<sub>payment</sub>, the funding payment for a position that has been held through a single funding event at block j, can be computed:

<p align="center"><span class="math">F_{payment}[j] = (F_{sum}[j] − F_{sum}[j − k]) · L</span></p>

_F_<sub>_sum_</sub>_\[j] = Funding product sum for funding event._\
&#xNAN;_&#x46;_<sub>_sum_</sub>_\[j − k] = Funding product sum for previous funding event._\
&#xNAN;_&#x6A; = funding event block number._\
&#xNAN;_&#x6B; = The number of blocks in a funding interval._\
&#xNAN;_&#x4C; = Position lot size._

Observing that equation _F_<sub>_payment_</sub> only depends on the lot size and side (long or short) of a position, it is possible to extend the solution to compute funding payments cumulatively owed for a position held through multiple events. For example, consider a position held through 3 funding events, up to and including a funding event at block j. Its cumulative funding payment could be computed as follows:

<p align="center"><span class="math">F_{payment}[j] = (F_{sum}[j] − F_{sum}[j − 3k]) · L</span></p>

_F_<sub>_sum_</sub>_\[j] = Funding product sum for funding event._\
&#xNAN;_&#x46;_<sub>_sum_</sub>_\[j − 3k] = Funding product sum for the three funding events prior._\
&#xNAN;_&#x6A; = funding event block number._\
&#xNAN;_&#x6B; = The number of blocks in a funding interval._\
&#xNAN;_&#x4C; = Position lot size._

Importantly, notice that the cumulative funding payment owed can be determined in only two read operations. An important gas efficiency. Furthermore, the value of the funding payment need not be written to the chain until the entire position settles, bypassing the problem of immediate explicit settlement.

Thus, the generalized solution to compute any position’s cumulative funding payment, which can also be expressed as its premium PNL, θpnl, is:

<p align="center"><span class="math">θ_{pnl} = F_{payment}[j] = (F_{sum}[j] − F_{sum}[m]) · L</span></p>

_F_<sub>_sum_</sub>_\[j] = Funding product sum for funding event._\
&#xNAN;_&#x46;_<sub>_sum_</sub>_\[m] = Funding product sum for funding event prior to position creation._\
&#xNAN;_&#x6A; = funding event block number._\
&#xNAN;_&#x6D; = The funding event block number for the funding event prior to position creation._\
&#xNAN;_&#x4C; = Position lot size._
