# 🦢 Insurance & ADL

In the event of a Black Swan scenario, there are measures in place to ensure the exchange remains solvent while minimizing negative consequences for users.

### Insurance Fund

An insurance fund is maintained individually for each perpetual contract. It serves as a backstop when the perpetual's main balance is insufficient to cover obligations.

**Funding sources:**

* Liquidation proceeds — a configurable share (`liqInsAmtPer100K`) of each order book liquidation is directed to the insurance fund

**Usage:**

* Covers shortfalls when a perpetual's balance goes negative during settlement
* Distributed pro-rata to profitable positions during contract unwinding

<figure><img src="https://809468657-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FXVKhDpsjDZo7VFt2Qfew%2Fuploads%2FNxz15q22KOByWiKz0tu8%2Fimage.png?alt=media&#x26;token=b01a7002-9031-4f56-9bc0-7196741d17c8" alt=""><figcaption></figcaption></figure>

### Force Close

If there is no resting liquidity in the order book and the price remains stagnant or sideways for an extended period, traders can request that the protocol force-close their position at the mark price and exit the perpetual contract.

### Auto-Deleveraging (ADL)

This usually happens when there's not enough new liquidity entering, or price gaps up/down drastically. ADL ensures the protocol stays solvent, as was widely experienced during the October 10th, 2025, crash. Rapid price changes can push liquidatable positions into bankruptcy before the system can liquidate them.

In this situation, it's important for the system to remove positions (deleverage) quickly to ensure that other users with profitable positions are paid. Failure to do so in a timely manner, along with continued price fluctuations, could result in the perpetual's insolvency.

**How ADL works:**

* The caller provides a sorted list of opposing position IDs (most profitable first)
* Positions are force-closed at the mark price against these opposing profitable positions
* The **perp is not paused** and can continue operating

{% hint style="info" %}
ADL position selection is performed off-chain. The caller submits the sorted list of opposing positions to be deleveraged against.
{% endhint %}

### Contract Unwind

Unlike ADL, the **perp is paused** due to an extreme trading scenario (price manipulation, black swan events, thinly traded markets, etc.). The perpetual contract may be delisted under some conditions.

The unwind is a controlled 3-stage process:

#### Stage 1: Prepare (`UnwindPrepared`)

* All position changes are frozen — no new orders, no position modifications
* This stage is **reversible** — the Owner can cancel the unwind and return the perpetual to normal operation
* Existing resting orders remain on the book, but cannot be matched

#### Stage 2: Initialize (`UnwindInitialized`)

* The protocol commits the sum of all positive fair market values (FMVs) across all positions
* This establishes the total payout obligation for the perpetual
* This stage is also **reversible**

#### Stage 3: Trigger (`UnwindStarted`)

* Positions with positive value are paid out **pro rata** from the perpetual's remaining balance plus the insurance fund
* This stage is **irreversible** — once triggered, the perpetual is effectively delisted
* If the perpetual does not have sufficient funds to pay all positions at full value, payouts are reduced proportionally

In an unwind, the perpetual may not have sufficient funds to pay existing positions at the value implied by the mark price. Regardless, it will compensate positions with positive values on a pro-rata basis from the remaining funds. The price used to determine position value is the mark price at the time of pausing.

{% hint style="info" %}
The price may be adjusted by the protocol under extreme circumstances. All such changes are logged on the chain for transparency.
{% endhint %}
