# Order Types

Perpl supports the following order types:

* **Market Order**: Executes immediately for the best available price
* **Limit Order**: Executes at the specified price or better
* **Stop Market Order**: Becomes a market order when the trigger price is reached
* **Stop Limit Order**: Becomes a limit order when a trigger price is reached
* **Take Profit Order**: Closes a position when a profit target is reached
* **TWAP** (coming soon): Time weighted average price, executes large orders over some time, breaking them into smaller, more frequent trades, to minimize the impact on the market price.

{% hint style="info" %}
**On-chain vs SDK order types**: The smart contract supports 7 fundamental order types: OpenLong, OpenShort, CloseLong, CloseShort, Cancel, IncreasePositionCollateral, and Change (modify price/size/expiry of a resting order). Advanced order types like Stop Market, Stop Limit, Take Profit, and TWAP are abstractions built on top of these on-chain primitives via the SDK and keeper layer — they are not native contract operations.
{% endhint %}

### Order Options:

* **Good Til Cancel (GTC)**: An order that rests on the order book until it is filled or canceled
* **Immediate or Cancel (IOC)**: An order that will be canceled if it is not immediately filled
* **Fill or Kill (FOK)**: If the full quantity of the order cannot be filled instantly, the order is automatically canceled, preventing any partial or delayed execution
* **Reduce Only**: An order that reduces a current position as opposed to opening a new position in the opposite direction. A Reduce Only order can only be placed if there is an existing position.
* **Post Only**: An order that is added to the book only; it will revert if it would immediately cross the spread and match against an existing order.
* **Max Matches**: Caps the number of book matches per operation, helping control gas costs for large orders.
* **Threshold Price**: Maximum slippage protection, up to 65,535 bps from the reference price.
* **Expiry Block**: Time-in-force expressed as a block number. Expired orders can be cleared by anyone (with a recycle-fee incentive enabled when configured).
* **Minimum Size**: Every order must be for at least one size unit of the market (for example 0.00001 BTC, 0.001 ETH, 1 MON). There is no dollar minimum to place an order today, and closing your entire position is always allowed. See [Minimum Orders](minimum-orders.md).

{% hint style="info" %}
**Change Orders**: Resting limit orders on the book can be modified in-place (price, size, or expiry) using a single Change operation. This reuses the existing order's storage slot, saving approximately 15,000 gas compared to canceling and re-placing an order.
{% endhint %}

To learn more about order types, read our blog on [perp exchange order types](https://blog.perpl.xyz).
