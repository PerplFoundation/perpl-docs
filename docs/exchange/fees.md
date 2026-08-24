# Fees

Perpl uses a maker-taker model. Maker fees are paid when adding liquidity to the order book, while taker fees are paid when removing liquidity from it. The user only pays fees to open a trade, not to close it. See the fee structure below in BPS.

<table><thead><tr><th width="76.8203125">Tier</th><th>14‑day Volume</th><th data-type="number">Maker Open</th><th data-type="number">Taker Open</th><th data-type="number">Maker Close</th><th data-type="number">Taker Close</th></tr></thead><tbody><tr><td>T1</td><td>&#x3C; $5M</td><td>0.9</td><td>6.9</td><td>0</td><td>0</td></tr><tr><td>T2</td><td>≥ $5M</td><td>0.5</td><td>6</td><td>0</td><td>0</td></tr><tr><td>T3</td><td>≥ $25M</td><td>0.3</td><td>5</td><td>0</td><td>0</td></tr><tr><td>T4</td><td>≥ $100M</td><td>0</td><td>4.2</td><td>0</td><td>0</td></tr><tr><td>T5</td><td>≥ $250M</td><td>-0.01</td><td>3.5</td><td>0</td><td>0</td></tr><tr><td>VIP1</td><td>≥ $500M</td><td>-0.1</td><td>3</td><td>0</td><td>0</td></tr><tr><td>VIP2</td><td>≥ $1B</td><td>-1</td><td>2.5</td><td>0</td><td>0</td></tr></tbody></table>

### Fee Types

| Fee               | Description                                                                                                                                                                       |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Trading fee**   | Charged per trade on the notional value. Configurable per perpetual, maximum 10%.                                                                                                 |
| **Recycle fee**   | Applied to orders with an expiry block. Refunded if the order is filled or self-canceled. Paid to whoever clears an expired order as an incentive. Currently set to 0 on mainnet. |
| **Insurance fee** | A portion of liquidation proceeds directed to the per-perpetual insurance fund (`liqInsAmtPer100K`).                                                                              |
| **Protocol fee**  | The remainder of liquidation proceeds after user and insurance shares.                                                                                                            |
