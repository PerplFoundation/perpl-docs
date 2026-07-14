# Margin

The margin engine ensures that users who place trades always have sufficient capital to open and maintain their intended positions.

* **Initial Margin**: The amount required to open a position; calculated as a percentage of notional value based on the asset's risk profile.
* **Maintenance Margin**: Minimum collateral required to keep a position open; if breached, liquidation is triggered

Initial margin requirements on Perpl are dynamic and may vary by market, depending on volatility and position size.

#### **Liquidation Distance**

Your liquidation distance is how far the price can move against you before your position is liquidated. It's set by two things: the leverage you choose (which determines your initial margin) and the market's maintenance margin.

For example, on a market with a 5% maintenance margin, opening at 5x means posting 20% initial margin. The position is liquidated once your equity falls to the 5% maintenance level — roughly a 16% adverse move:

<p align="center">(20% − 5%) ÷ (1 − 5%) ≈ 16%</p>

Lower leverage widens this distance; higher leverage narrows it. Maintenance margin varies by market, so the exact figure for your position is shown live in the Adjust Leverage dialog. You can add collateral to an open position at any time to increase it.

<figure><img src="../.gitbook/assets/Screenshot 2026-06-16 at 10.56.26.png" alt="" width="375"><figcaption></figcaption></figure>

#### Margin Mode

The difference between a cross and an isolated margin is:

* **Isolated Margin**: Each position has its own margin
* **Cross Margin**: All collateral in the account is shared across positions
* **Hybrid Margin**: Manually manage margin between positions

{% hint style="info" %}
To begin with, Perpl will be restricted to isolated margin.
{% endhint %}

<figure><img src="https://809468657-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FXVKhDpsjDZo7VFt2Qfew%2Fuploads%2FT0Wu8pWV98wymIzdfmLk%2Fimage.png?alt=media&#x26;token=3394098b-112d-466d-9014-73f294aa976f" alt=""><figcaption></figcaption></figure>

#### Position Equations

Perpetual contracts can be implemented using the following representation of lot sizes, L:

* {L : L > 0, L ∈ Z}

**Position Notional Value,&#x20;**_**N**_**:**

<p align="center"><span class="math">N = P · L</span></p>

_P = The mark, entry, or realized price, depending on whether or not the value being calculated is unrealized, position, or realized notional value, respectively._\
&#xNAN;_&#x4C; = The position lot size (the number of contracts of the position)._

**Position Margin Requirement,&#x20;**_**MR**_**:**

<p align="center"><span class="math">MR = N/MF</span></p>

_N = The position notional value._\
&#xNAN;_&#x4D;F = The margin fraction (analogous to leverage)._

**Position Initial Margin Requirement,&#x20;**_**IMR**_**:**

<p align="center"><span class="math">IMR = N/IMF</span></p>

_N = The position notional value._\
&#xNAN;_&#x49;MF = Initial margin fraction (maximum leverage allowed to open a position)._

**Position Maintenance Margin Requirement,&#x20;**_**MMR**_**:**

<p align="center"><span class="math">MMR = N/MMF</span></p>

_N = The position notional value._\
&#xNAN;_&#x4D;MF = Maintenance margin fraction (minimum collateralization permitted before a position can be_\
&#xNAN;_&#x6C;iquidated)._

#### Collateral Management

**Increase Collateral**

Traders can add margin to an existing position at any time to reduce the risk of liquidation. This increases the position's `depositCNS` without affecting the entry price or lot size.

**Decrease Collateral (DCP)**

Traders can remove excess margin from a position, subject to the following constraints:

* **120-second expiry**: A DCP request must be executed within 120 seconds of submission. After this window, the request expires and must be resubmitted.
* **Margin tolerance check**: The remaining collateral after the decrease must still satisfy the initial margin requirement for the position.
* **OI threshold**: DCP is blocked when the perpetual's open interest exceeds the `dcpBorrowThreshHdths` threshold (default 85% of capacity), as the market is under stress.

{% hint style="info" %}
The 120-second expiry on DCP prevents stale withdrawal requests from being executed after market conditions have changed significantly.
{% endhint %}

#### Margining Criteria

The price used to determine the notional value in the invariants presented in this section is the most accurate price available. For example, the best price to use for the notional value of a new position is the price at which the position is entered (realized).

In some situations, for example, auto-deleveraging a position, a realization price is not available, and the failed position's bankruptcy price is used. In other situations, the mark or synthetic perpetual price may be used to calculate the position's fair market value (for the value of an existing position).

**Collateralization for establishing or changing a position:**

<p align="center"><span class="math">FMV >= IMR</span></p>

_FMV = Position fair market value._\
&#xNAN;_&#x49;MR = Position initial margin requirement._

{% hint style="info" %}
**Reduce-only exemption**: Closing or decreasing a position does not require FMV >= IMR. This allows traders to close underwater positions that would otherwise be trapped.
{% endhint %}

**Collateralization for liquidation:**

<p align="center"><span class="math">0 &#x3C; FMV &#x3C;= MMR</span></p>

_FMV = Position fair market value._\
&#xNAN;_&#x4D;MR = Position Maintenance margin requirement._

**Collateralization for Auto Deleverage (ADL):**

<p align="center"><span class="math">FMV &#x3C;= 0</span></p>

_FMV = Position fair market value._
