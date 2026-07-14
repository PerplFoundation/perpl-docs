# 🛣️ Limiting Open Interest

### Overview

Open interest (OI) caps protect the exchange from taking on more aggregate market exposure than it can safely support. As total OI grows, the protocol's risk of insolvency, liquidity stress, and liquidation cascades increases. Perpl enforces a per perpetual OI cap (`maxOpenInterestLNS`) and dynamically scales margin requirements as open interest approaches that cap.

There are four thresholds that activate progressively as OI grows:

| Threshold                   | Default    | Effect                                                                            |
| --------------------------- | ---------- | --------------------------------------------------------------------------------- |
| `dcpBorrowThreshHdths`      | 85%        | Decrease Collateral (DCP) blocked — traders cannot withdraw margin from positions |
| `unityDescentThreshHdths`   | 90%        | Dynamic IMF begins descending from the perpetual's default toward 1x leverage     |
| `overColDescentThreshHdths` | 95%        | Dynamic IMF descends further, approaching full collateralization                  |
| `maxOpenInterestLNS`        | 100% (cap) | **Hard cap** — only reduce-only orders allowed; open orders are auto-canceled     |

These thresholds are configurable per perpetual by the Owner and must satisfy: `dcpBorrowThresh < unityDescentThresh < overColDescentThresh`.

### Hard OI Cap

When total open interest reaches `maxOpenInterestLNS`:

* **Only reduce-only orders are accepted** — no new positions or position increases
* **Resting open orders on the book are auto-canceled** with the recycle fee remitted to the canceller
* **Normal operation resumes** when OI drops back below the cap

This serves as the ultimate backstop to prevent the exchange from becoming overexposed in any single market.

### Dynamic Initial Margin (OIMF)

Instead of using a constant Initial Margin Fraction (IMF), the protocol dynamically adjusts the margin requirement as open interest approaches its maximum. This makes it progressively more expensive to open leveraged positions as the market becomes crowded.

The dynamic IMF is computed as a piecewise-linear function of the perpetual's forward-looking open interest:

#### Segment 1 — Default IMF (OI below 90%)

When open interest is below `unityDescentThreshHdths`:

The perpetual's default IMF applies unchanged. For example, if the perpetual allows 50x leverage, traders can use up to 50x.

#### Segment 2 — Unity Descent (OI between 90%–95%)

Between `unityDescentThreshHdths` and `overColDescentThreshHdths`:

IMF interpolates linearly from the perpetual's default down toward 1x leverage (unity). As OI grows through this range, the maximum allowed leverage decreases. A perpetual that normally allows 50x might only allow 25x at 92.5% OI.

#### Segment 3 — Over-Collateralization Descent (OI above 95%)

After `overColDescentThreshHdths`, up to maximum open interest:

IMF continues descending below unity toward a minimum value, effectively requiring full or over-collateralization. At this point, opening new leveraged positions becomes impractical — the margin requirement approaches or exceeds the full notional value.

```
Max Leverage
(IMF)    ^
         |
  50x    |_______________
         |               \
         |                \   (linear descent)
         |                 \
   1x    |                  \________
         |                           \
         +----------------------------\---------> Open Interest (%)
         0%      85%    90%     95%   100%
                  ^      ^       ^      ^
                  |      |       |      |
                 DCP   Unity  OverCol  Hard
                Block  Descent Descent  Cap
```

#### DCP Borrow Threshold (OI above 85%)

Before the IMF curve activates, the first protective measure kicks in at 85% OI: Decrease Collateral Position (DCP) requests are blocked. This prevents traders from withdrawing margin from existing positions when the market is under stress, preserving collateral buffers across the system.

### Reference

For concrete visualizations and finite-precision behavior of the OIMF curve, refer to the Desmos example: [https://www.desmos.com/calculator/x0jeddkmp8](https://www.desmos.com/calculator/x0jeddkmp8)
