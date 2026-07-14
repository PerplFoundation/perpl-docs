# 🏧 Withdrawal Limits

### Overview

The withdrawal rate limit is a safety mechanism that limits the global rate of fund withdrawals from the exchange. It is designed to slow mass withdrawals during a security incident, giving operators time to detect and respond before protocol funds are drained.

This rate limit applies to **all accounts**, including the Owner (unless explicitly bypassed). It does **not** affect internal protocol transfers or accounting operations that do not reduce TVL.

The system operates using Monad block approximations, not timestamps. Assuming an average block consensus time of **0.42 seconds**, an "hour" is represented by **8,571 blocks**.

### Parameters

| Parameter             | Default               | Range            | Description                                          |
| --------------------- | --------------------- | ---------------- | ---------------------------------------------------- |
| `wrlsThousandthsTvl`  | 100 (10% of TVL/hour) | 1–250 (0.1%–25%) | Fraction of TVL withdrawable per hour                |
| `minWithdrawLimitCNS` | $1,000,000            | $1M–$100M        | Floor on hourly withdrawal limit for low-TVL periods |
| Blocks per hour       | 8,571                 | Fixed            | Based on 0.42s average Monad block time              |

Both parameters are configurable by the Owner via `setThousandthsTvlWRLS` and `setMinWithdrawLimit`.

### Rate-Limit Calculation

When the first withdrawal of a new hour-period occurs, the contract:

1. Reads the current Total Value Locked (TVL)
2. Computes the maximum allowed withdrawal for the upcoming hour

#### Hourly Withdrawal Limit

$$
\textbf{hourly limit} = \max\left( \frac{\textit{wrlsThousandthsTvl} \cdot \text{TVL}}{1000},\ \textit{minWithdrawLimitCNS} \right)
$$

This ensures:

* The limit scales with protocol size
* There is a minimum cap to ensure usability when TVL is small

#### Collateral Per Block (Release Rate)

$$
\textbf{collateralPerBlock} = \frac{\text{hourly limit}}{\text{8,571 blocks}}
$$

This controls how quickly additional withdrawal capacity becomes available as blocks are mined.

### Burst Window (First \~15 Minutes)

At the start of each hour period:

* A burst amount equal to \~25% of the hourly limit is immediately available
* This includes any rounding remainder from integer division
* After this initial window, additional capacity unlocks block-by-block at the `collateralPerBlock` rate

The burst ensures:

* Regular user withdrawals remain smooth
* Emergency withdrawals remain possible
* Attackers cannot instantly drain the protocol

### Withdrawal Availability Curve

```
           ^
           |
           |
  hourly --+ - - - - - - - - - - - - - - - - - - - - - - - - - - - -*+*
  limit    |                                                       ***
           |                                                    ***    |
           |                                                 ***
  3/4      |                                              ***          |
  hourly - + - - - - - - - - - - - - - - - - - - - - - -*+*
  limit    |                                        ***  |             |
           |                                     ***
           |                                  ***        |             |
  1/2      |                               ***
  hourly - + - - - - - - - - - - - - - -*+*              |             |
  limit    |                         ***
           |                      ***    |               |             |
           |                   ***
  1/4      |     Burst      ***          |               |             |
  hourly --+****************
  limit    |               |             |               |             |
           |
           |               |             |               |             |
           |
           +---------------+-------------+---------------+-------------+------------>
           |
           |               |             |               |             |
           |
                       ~15 Minutes   ~30 Minutes     ~45 Minutes    ~1 Hour
       Start of
       Limit Period

NOTE: Burst amount includes remainder from downward rounding division.
```

#### Key Behaviors

* **First \~15 minutes**: Burst amount available immediately (\~25% of hourly limit)
* **15–60 minutes**: Linear increase at `collateralPerBlock`
* **At \~60 minutes**: Limit fully consumed; next withdrawal resets the cycle
* **Next withdrawal after hour boundary**: Recomputes limit using new TVL

### Cycle Reset

At the end of the hour (approximate block count reached):

* The next withdrawal call resets the cycle
* The current TVL is used to recompute the new hourly limit
* The cycle repeats: burst → linear release → reset

### Bypass Addresses

The Owner can designate addresses that bypass the withdrawal rate limit entirely via `setWithdrawBypass`. This is used for:

* Protocol-operated addresses that need unrestricted withdrawal access
* Emergency response — ensuring critical operations are not blocked during high-withdrawal periods

Bypass status is toggled per address and emits a `WithdrawRateLimitBypassSet` event for transparency.

### Owner Force Reset

The Owner can manually reset the withdrawal rate limit cycle at any time via `forceResetWithdrawRateLimit`. This immediately:

* Recomputes the hourly limit from the current TVL
* Restarts the burst + ramp cycle
* Restores withdrawal availability for all users

This is the primary incident response tool when legitimate withdrawals are being blocked.

### Known Limitation

A user can temporarily exhaust the withdrawal rate limit by depositing a large amount and immediately withdrawing up to the available allowance. This blocks other users from withdrawing until the limit replenishes or the cycle resets.

**Mitigations:**

* **Bypass address list**: Critical addresses (e.g., market makers, protocol vaults) can be whitelisted to bypass the rate limit
* **Owner force reset**: The Owner can manually reset the cycle to immediately restore withdrawal capacity
* **TVL-proportional scaling**: Larger protocol TVL means larger absolute withdrawal limits, making this attack more capital-intensive
