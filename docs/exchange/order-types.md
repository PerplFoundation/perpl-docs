# Order Types

Perpl supports the following order types:

* **Market Order**: Executes immediately against the order book at the best available prices (see [How market orders execute](#how-market-orders-execute))
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

## How market orders execute

A market order fills against the resting liquidity in the order book, consuming price levels from the best price outward until the full size is filled. The price you receive is the **volume-weighted average** of every level consumed — not a single price. On a deep book that average sits at or near the best price; on a thin or fast-moving market it can be several levels away.

{% hint style="info" %}
**Mark price vs. fill price — the key distinction.** Your unrealized PnL, a stop order's trigger, and liquidation are all measured against the [**Mark Price**](price-indices.md#mark-price). The price you actually receive — and therefore your **realized** PnL — comes from the **order book**. These are different numbers. A stop can trigger exactly at your mark-based level and still fill at a worse average price when the book is thin at that moment: the order behaved correctly; the gap is the book, not the trigger.
{% endhint %}

**Example.** A stop-loss triggers when the Mark Price falls to 63,583.0. At that instant the best bid is 63,534.9, and the size needed is spread across 8 resting orders down to 63,515.6 — so the position closes at an average of 63,519.8. The trigger fired on mark; the fill came from the book.

### Slippage protection

Market orders carry a **maximum slippage** limit, set in the app's trade Settings (expressed on-chain as the order's [Threshold Price](#order-options)). The order fills across the book only up to that limit; any size that would fill beyond it is **not** executed. This is why a market order — including a one-click **Close** — can fill only partially, or not at all, on a thin book: the protocol will not execute it at a price worse than your slippage setting allows. Widening the limit trades a worse possible price for a higher chance of a complete fill.

### Stop-Loss and Take-Profit: market vs. limit

When a stop order triggers (evaluated against the Mark Price), it is submitted as one of two things, depending on whether you set a limit price:

* **No limit price → market order** (default). Fills immediately at the best available price; slippage is possible, but the order is very likely to execute.
* **Limit price set → limit order.** Rests at your chosen price; you get that price or better, but the order may **never execute** if the book does not reach it.

{% hint style="warning" %}
**On thin markets you are choosing between price and certainty.** A market stop guarantees execution but not price; a limit stop guarantees price but not execution. If you use a limit stop, set its limit price _past_ the trigger (further into the loss for a stop-loss) so there is book depth to fill against — otherwise the order can trigger and then sit unfilled.
{% endhint %}

To learn more about order types, read our blog on [perp exchange order types](https://blog.perpl.xyz).
