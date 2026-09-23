# Minimum Orders

Perpl keeps order minimums as low as the contract can represent. In short: **there is no dollar minimum to place an order today**, and you can **always close your entire position**, however small.

Three separate rules tend to get confused — the exchange checks them independently.

## Minimum order size (always on)

Every order must be for at least **one size unit** of the market — the smallest amount the market can represent. This never changes.

| Market | Minimum order size (1 unit) | Roughly worth  |
| ------ | --------------------------- | -------------- |
| BTC    | 0.00001 BTC                 | a few dollars  |
| ETH    | 0.001 ETH                   | a few dollars  |
| SOL    | 0.001 SOL                   | under a dollar |
| MON    | 1 MON                       | under a dollar |
| HYPE   | 0.01 HYPE                   | under a dollar |
| ZEC    | 0.0001 ZEC                  | under a dollar |

One size unit is 1 divided by the market's size scale. "Roughly worth" is indicative and moves with price.

{% hint style="info" %}
If a client or trading library blocks you at a larger size (for example "minimum 0.0005 BTC"), that limit is coming from that tool — not from Perpl. The exchange and the Perpl web app both use the one-unit floor above.
{% endhint %}

## Minimum order value (currently $0)

Separately from size, the exchange can require a minimum **dollar value** per order. **Today this is set to $0 on both mainnet and testnet, so it does not block anything.** It exists so the exchange can raise the bar during heavy congestion or an order-book spam attack, and can be set anywhere from $0 up to about $164 per order.

When a minimum order value is in effect, it is measured on how much collateral value the order moves — not raw notional:

* **Opening** (a new position, adding to one, or flipping sides): the initial margin for the new size (size × price ÷ leverage), plus any value freed by closing an opposite side in the same order.
* **Closing** (reducing): the pro-rata fair value removed — the share of the position's collateral plus its profit or loss that the closed portion represents.

{% hint style="info" %}
**Closing your whole position is always allowed.** A full close is exempt from the order-value minimum even if one is switched on — this is what lets you fully exit a tiny ("dust") position at any time.
{% endhint %}

## Minimum to open an account (a deposit rule, not an order rule)

To create a new trading account you make a first deposit of at least **$10 on mainnet** ($100 on testnet). This is a one-time check when the account is first funded — not a per-order limit.

## Current live settings

| Setting                            | Mainnet           | Testnet           |
| ---------------------------------- | ----------------- | ----------------- |
| Minimum order size                 | 1 unit per market | 1 unit per market |
| Minimum order value                | $0.00             | $0.00             |
| Minimum deposit to open an account | $10.00            | $100.00           |

These values are owner-adjustable. Integrators should read them live rather than hard-code them — see the developer note in [Networks & Configuration](../resources/for-developers/networks-and-configuration.md).
