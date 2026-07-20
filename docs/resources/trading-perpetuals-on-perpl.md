# Trading Perpetuals on Perpl

Assuming you have a funded wallet with AUSD on Monad, you’re ready to start trading. If you do not have a funded wallet with AUSD on Monad, go back a few documents to get set up.&#x20;

To start trading, visit Perpl.xyz, click the “Connect” button, and choose a wallet to connect. A pop-up will appear in your wallet extension asking you to connect to Perpl. Press “Connect.”

Click the “Enable Trading” button. A pop-up will appear in your wallet extension asking you to sign a gasless transaction. Press "Sign." &#x20;

Given your assets are now on Monad, you simply deposit them onto the exchange. Click “Deposit” to get your funds onto the exchange. Confirm the transaction in your wallet, and your collateral should show up in your account near instantly.&#x20;

Now that your Perpl account is funded, it’s time to start trading perps. Perps allows you to long or short a token using AUSD as collateral, without having to own the token directly, as in spot trading.

Placing your first trade is easy:

1. Select a Token: Use the token selector or search function to find the market you want to trade.
2. Choose Long or Short
3. Long if you expect the price to rise.
4. Short if you expect the price to fall.
5. Set Position Size: Use the slider or enter a value.
6. Position Size = Leverage × Collateral
7. Example: 10x leverage X $100 collateral = $1,000 position size
8. Place Your Order: Click Place Order, then confirm in the pop-up.&#x20;

There’s no dollar minimum to open a trade — the only floor is one size unit of the market (for example 0.00001 BTC), worth well under a few dollars — and you can always close your entire position, however small. See [Minimum Orders](../exchange/minimum-orders.md).

Congratulations, you’ve now placed your first trade on 11/11. Now comes the fun part, watching your trade play out and timing the markets right. There are a few considerations, like funds and liquidations, you should know about before sitting back. Read the rest of the architecture documents for a better understanding of some of the nuances that come with trading perps.
