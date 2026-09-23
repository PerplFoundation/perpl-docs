# Rescue UI

## Rescue Mode

[Rescue Mode](https://recovery.perpl.xyz/) is a stripped-down version of Perpl that lets you close positions and withdraw funds even when the main app is unavailable. It is hosted as a standalone single-page app, and depends on nothing from our servers or frontend infrastructure.

If the main site is down, your funds are still safe and still yours to move.

### Why it exists

A perpetual futures DEX is only as trustworthy as your ability to exit. Most of the time, you'll never see Rescue Mode – but if our frontend goes offline, a node provider has an outage, or anything else takes the main app down, you should never be locked out of your own money.

Because Rescue Mode talks directly to the chain, it keeps working when the rest of the stack doesn't.

### What it can do

Rescue Mode is **reduce-only**. It is intentionally limited to the actions you need to protect your account:

* View your open positions and orders
* Close positions, fully or partially
* Close all positions at once
* Cancel open orders
* Withdraw your funds

You cannot open new positions, increase existing ones, or place new entry orders. This is by design – Rescue Mode is an exit, not a trading terminal.

### What you'll see

**Account panel**

* **Maintenance Margin** — the minimum margin required to keep your positions open
* **Available Margin** — margin not currently committed to positions
* **Balance** — your total account balance
* **Unrealized PnL** — open profit or loss across your positions
* **Close ALL positions** — a single action to flatten your entire account

**Positions and Orders**

Each position shows its market, side and leverage (e.g. BTC Long 10x), size, mark price, and liquidation price, along with partial-close controls. The Orders tab lists any resting orders, which you can cancel.

### How to use it

If the main app is down, Perpl will direct you to Rescue Mode automatically

1. Open the Rescue Mode site (automatically or via [https://recovery.perpl.xyz/](https://recovery.perpl.xyz/).)
2. Connect the same wallet you use to trade on Perpl.
3. Review your positions and orders.
4. Close positions partially or fully, cancel orders, and withdraw as needed.

### How it works

Rescue Mode reads your account state directly from the chain and submits reduce-only and withdrawal transactions straight to the smart contracts — no API, no backend, no dependency on Perpl's servers. As long as the network is running and you can reach an IPFS gateway, you can act on your account.

### Good to know

* **You'll usually be sent here automatically.** If the main app is down, Perpl points you to Rescue Mode.
* **Prices may look different.** Rescue Mode is built for safety and resilience, not for precise execution – expect a simpler view than the main trading app.
* **It's reduce-only.** If you want to open or add to positions, use the main Perpl app once it's back.
* **Your funds are always yours.** Rescue Mode exists so that your ability to exit never depends on us being online.
