# Fees

Perpl uses a maker-taker model. Maker fees are paid when adding liquidity to the order book, while taker fees are paid when removing liquidity from it. The user only pays fees to open a trade, not to close it. See the fee structure below in BPS.

<table><thead><tr><th>Tier</th><th>14D Volume</th><th data-type="number">Maker (bps)</th><th data-type="number">Taker (bps)</th></tr></thead><tbody><tr><td>1</td><td>&#x3C; $5M</td><td>0.45</td><td>3.45</td></tr><tr><td>2</td><td>≥ $5M</td><td>0.25</td><td>3</td></tr><tr><td>3</td><td>≥ $25M</td><td>0.15</td><td>2.5</td></tr><tr><td>4</td><td>≥ $100M</td><td>0</td><td>2.1</td></tr><tr><td>5</td><td>≥ $250M</td><td>-0.01</td><td>1.75</td></tr><tr><td>VIP 1</td><td>≥ $500M</td><td>-0.1</td><td>1.5</td></tr><tr><td>VIP 2</td><td>≥ $1B</td><td>-1</td><td>1.25</td></tr></tbody></table>

### Maker rebates

Negative maker rates (on Tier 5, VIP 1 and VIP 2) are rebates. You receive the stated amount instead of paying a fee.

Rebates are calculated off-chain and paid separately. They do not arrive with each fill. Payouts usually occur every two weeks.

### Fee Types

| Fee               | Description                                                                                                                                                                       |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Trading fee**   | Charged per trade on the notional value. Configurable per perpetual, maximum 10%.                                                                                                 |
| **Recycle fee**   | Applied to orders with an expiry block. Refunded if the order is filled or self-canceled. Paid to whoever clears an expired order as an incentive. Currently set to 0 on mainnet. |
| **Insurance fee** | A portion of liquidation proceeds directed to the per-perpetual insurance fund (`liqInsAmtPer100K`).                                                                              |
| **Protocol fee**  | The remainder of liquidation proceeds after user and insurance shares.                                                                                                            |
