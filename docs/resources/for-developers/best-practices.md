# Best Practices

This page covers how to quote efficiently on Perpl. The single most important optimization for a market maker (MM) is **how you move your quotes when the price changes**: prefer amending orders **in place** (`Change`) over cancelling and re-posting. On Monad's asynchronous execution this is the difference between quotes that stay continuously on the book and quotes that leave gaps every time you requote.

> **Interactive explainer:** a step-by-step visualization of the two regimes below is at [perplfoundation.github.io/explainers/change-order](https://perplfoundation.github.io/explainers/change-order/interactive.html).

## The golden rule: `Change`, don't cancel-and-repost

When an oracle moves and you need to reprice, you have three ways to do it. Ranked best to worst:

| Strategy                                          | How                                                              | Verdict                                                                                                        |
| ------------------------------------------------- | ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| **`Change` (amend-in-place)**                     | One `Change` op per order: same order ID, new price/size/expiry. | **Preferred** — lowest gas, order never leaves the book, keeps liquidity present.                              |
| **Batch cancel + post in one transaction**        | Multiple ops in a single `execOrders` call (cancel then post).   | Acceptable — safe against async lag because both ops settle in the same block, but \~2× the gas of a `Change`. |
| **Separate cancel, then post (two transactions)** | Cancel in one tx, post in another.                               | **Fragile — avoid.** Causes book gaps and margin-release rejections (see below).                               |

## Why this matters on Monad

Monad produces blocks every \~400 ms and executes transactions **asynchronously** — Monad's deferred-execution model, known as **kn3**: a transaction's state effects are settled a few blocks after it is included, and independent transactions in a block execute **in parallel**. Two consequences drive maker strategy:

**1. Cancel-and-repost drains the book mid-block.** A `Cancel` followed by a `Post` is two separate order-book mutations. Both touch shared book state — the price-level index (`Index16Bit`) and the per-market order count (`numOrders`) — so when several MMs requote the same levels at once, those writes contend and must be re-executed and serialized rather than run in parallel. During that window each MM's `Cancel` has already removed it from the book while its `Post` has not yet restored it. A taker arriving mid-block sees only a partial, stale book: fewer levels, wider spread, degraded depth. This is a _liquidity gap_.

**2. `Change` transitions the book atomically, in parallel.** A `Change` amends the existing order in place — the **order ID stays the same and the order never leaves the book**. The margin lock is updated atomically (the old lock is released and the new lock taken in a single step), so the account is never momentarily short of collateral. Because every MM is amending its _own_ order rather than fighting over the shared index, all the changes apply in parallel. The book transitions instantly from "full old prices" to "full new prices" — a taker arriving at any moment sees a complete book, never a drained one.

In short: `Change` is more gas-efficient (one book modification instead of a delete plus a write) and has **no cross-transaction ordering dependency**, so your liquidity stays present through every requote.

{% hint style="warning" %}
**Note — the failure mode to avoid.** If you cancel in one transaction and post in a _separate_ transaction, the cancel's margin release may not have settled when the post's balance check runs (the cancel settles a few blocks later under kn3). The post is then rejected with `AmountExceedsAvailableBalance`. Not knowing which posts failed, naive makers retry — producing cascading rejections. The fix is not to wait longer; it is to keep cancel-and-post **inside a single transaction** (or better, use `Change`). The problem is transaction boundaries, not raw latency.
{% endhint %}

## Using `Change`

A `Change` is an order operation that carries the **existing order ID** (`oid`) plus the fields you want to amend. In the WebSocket/REST API it is `OrderType` value **`7`**; the SDK/on-chain `RequestType` enum is 0-indexed, so the same operation is value **`6`** when you build it for `execOrders`. See [Types & Errors](/broken/pages/f3d5373018598765d9da98c708aefe0053551c2b) for the full order-type and flag tables, and [REST API](/broken/pages/81bef8fbf6300ac6ec7bd9f8e6c207c4f0869ddf) / [WebSocket API](/broken/pages/223abca72fc7209783a4ad6eb9ff8c6e0fade083) for the request shapes.

A single `Change` can amend any of:

* **Price level** — move the order to a different tick. This is the common requote when the oracle moves.
* **Size** — increase or decrease the resting quantity.
* **Expiry** — set a new expiry block. In the WebSocket API the operative field is `lb` (last execution block); the interface also defines `tif` (time-in-force), and the SDK exposes both `expiryBlock` and `lastExecutionBlock`.

### Queue priority (important)

Amending **size down keeps your queue priority**; amending size **up sends you to the back of the queue** at that level. If you need to grow a quote, be aware you forfeit your place in line. A **price change** always re-queues you at the back of the new level (you are a new arrival there).

Practical rule: shrink in place freely; grow deliberately.

## Batch your requotes

To reprice a whole ladder, send **multiple `Change` ops in one transaction** (`execOrders` on-chain, or the SDK's batch order call). This is the `batch change price` pattern:

* One transaction, one settlement, one gas overhead — all your levels move together.
* All levels transition in the same block, so the book is never half-updated.
* It composes with the parallel-execution benefit above: your batch does not contend with other MMs' batches.

If you genuinely must cancel and replace (e.g., changing an order's side), put the cancel **and** the post in the _same_ `execOrders` transaction so the cancel's margin release is visible to the post's balance check in-block.

## Other maker tips

* **`PostOnly` flag** — set the `PostOnly` flag (`1`) so an order that would cross the book is rejected instead of executing as a taker. This guarantees you pay maker, never taker, fees.
* **`IoC` / `FOK`** — use `ImmediateOrCancel` (`4`) or fill-or-kill semantics when you deliberately want to take, e.g. hedging.
* **Manage expiries by block** (the `lb` last-execution-block field) rather than cancelling — let stale quotes lapse, or refresh them with a `Change`.
* **Use `rq` (request ID) as a client order ID** and track the response sequence numbers for reliable, idempotent requoting — see the reliability guidance in [WebSocket API](/broken/pages/223abca72fc7209783a4ad6eb9ff8c6e0fade083).
* **Use the SDK's in-memory state cache** (`SnapshotBuilder` + `stream`) instead of polling — it keeps a live view of your orders and the book so you can compute the next `Change` locally without extra reads. See [SDK Concepts](/broken/pages/2771b2d599f49e7ea4cea4aa5d579dfc156838d8).

## Measuring yourself: the Change-to-Place ratio

The clearest health metric for a maker is the ratio of **`Change` operations to new `Post` operations**:

| Change : Place | What it means                                            |
| -------------- | -------------------------------------------------------- |
| **> 50 : 1**   | Amend-in-place strategy — minimal book absence.          |
| **< 5 : 1**    | Cancel/replace strategy — book absence on every requote. |

In current production this ratio spans a wide range — from below `1:1` for place-heavy accounts to well over `1000:1` for pure amend-in-place quoting, with cancel/replace-style makers clustering at low single digits (roughly `2–5:1`). The exact number matters less than the direction: if your ratio is low, converting requotes from cancel-and-post to `Change` (batched) is the highest-leverage change you can make — it improves your uptime on the book, tightens the effective spread takers see, and lowers your gas.
