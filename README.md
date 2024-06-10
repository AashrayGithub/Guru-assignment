





## Addressing Price Discrepancies Between Chainlink and Uniswap V3

### Overview

I am presenting a solution to the issue of price discrepancies between Chainlink price feeds and Uniswap V3 pool exchange rates that we have observed in our protocol. This discrepancy can lead to suboptimal swaps, resulting in fewer tokens received than anticipated. Below, I explain the differences between the two rates, the practical need for synchronization, and propose a theoretically sound solution for aligning the Uniswap V3 pool exchange rates with Chainlink price feeds.


### Understanding the Discrepancy

* **Chainlink:** Provides decentralized, tamper-resistant price feeds by aggregating data from multiple sources, reflecting a reliable representation of asset values.
* **Uniswap V3:** Determines exchange rates based on current liquidity and the constant product formula (x * y = k). This formula ensures a constant product of reserves (x and y) but can deviate from the true market price, especially during large trades or imbalances.

### Why Perfect Synchronization May Not Be Necessary

I believe it is important to understand that Chainlink price feeds and Uniswap V3 exchange rates are inherently different and represent distinct mechanisms. Chainlink aggregates prices from various sources to provide a broader market view, while Uniswap V3 exchange rates are a direct result of the current liquidity in the pool and recent trades. Therefore, while these prices tend to be similar, expecting them to be equal at all times is unrealistic. They reflect different aspects of the market: one is an aggregated, broad market price, and the other is a localized pool-specific price. Forcing them to be equal could overlook these fundamental differences and lead to unnecessary complexities.


### Proposed Solution for Synchronization

If aligning exchange rates with Chainlink price feeds is essential, here's a proposed solution:

**Step-by-Step Process:**

1. **Initial Setup:**
    * Retrieve the current price of ARB in USD from Chainlink.
    * Get the exchange rate from the ARB/USDC Uniswap V3 pool.
2. **Define Variables:**
    * `x`: Current reserve of ARB in the pool.
    * `y`: Current reserve of USDC in the pool.
    * `k`: Initial exchange rate from Uniswap V3 pool (k = y / x).
    * `oracle_ratio`: Desired exchange rate from Chainlink (1 ARB = oracle_ratio USDC).
    * `p`: Small transaction amount for ARB, minimizing impact on the pool's balance.
3. **Iterative Adjustment:**
    * Swap `p` ARB for USDC in the Uniswap V3 pool.
    * Calculate new reserves:
        * `new_x = x - p`
        * `new_y = y + k * p`
    * Determine the new exchange rate `k_new`:
        * `k_new = new_y / new_x`
    * Repeat the swap with updated reserves and exchange rates until:
        * `| k_latest - oracle_ratio | <= 1e-5`

By adjusting the value of `p`, the process ensures precise and controlled modifications to the pool, aligning the exchange rate with Chainlink's price feed.



## Further Optimizing Above Approach by Reducing Number of Transactions

This document describes an approach to optimize the value of `p` during Uniswap V3 swaps, minimizing the number of transactions required to achieve the desired exchange rate.

### Problem

Fixed-value `p` can lead to a high number of transactions, especially when dealing with large discrepancies between the Uniswap V3 exchange rate and the target price feed.

### Solution: Dynamic `p`

This solution dynamically adjusts `p` based on the current discrepancy:

* Larger adjustments for larger discrepancies.
* Smaller adjustments as we get closer to the target rate.

This minimizes the total number of transactions needed.

### Steps

1. **Base Transaction Amount (base_p)**

* Define the minimum possible value for a Uniswap trade.

2. **Calculate Discrepancy**

* Determine the absolute difference between the current and target rates.
* Calculate discrepancy percentage: `abs(current_rate - target_rate) / target_rate`

3. **Adjustment Factor**

* Calculate the minimum of the discrepancy percentage and 10%.
* This ensures a maximum adjustment of 10% per transaction to avoid market impacts.

4. **Dynamic Transaction Amount (dynamic_p)**

* `dynamic_p = base_p + (base_p * adjustment_factor)`
* This scales the transaction size based on the discrepancy.

### Example Calculation

**Initial Values:**

* `base_p = 0.001`
* Current Rate: 1 ARB = 1.5 USDC
* Target Rate: 1 ARB = 2 USDC

**Discrepancy Calculation:**

* Discrepancy: |1.5 - 2| = 0.5
* Discrepancy Percentage: 0.5 / 2 = 0.25 (25%)

**Adjustment Factor:**

* Minimum of 25% and 10% = 10% (capped at 10%)

**Dynamic p Calculation:**

* `dynamic_p = 0.001 + (0.001 * 0.1) = 0.0011`

### Benefits


This approach ensures that instead of making numerous small transactions at a base `p` value, the bot performs fewer, larger transactions by adjusting `p` based on the current discrepancy. If the gap percentage is more than 10%, the bot will reduce the gap by 10% in one transaction. If the gap is less than 10%, the bot adjusts `p` to close the gap in a single transaction. This strategy significantly reduces the total number of transactions needed to achieve the desired alignment, thus optimizing performance and reducing gas fees.




### Pseudo Code:


```



// Algorithm to dynamically adjust Uniswap V3 pool based on Chainlink price feed

// Function to get Chainlink price for asset
function get_chainlink_price(asset):
  // Simulate fetching price from Chainlink oracle
  return target_price  // Replace with actual price retrieval logic

// Function to adjust Uniswap V3 pool rate
function adjust_pool_rate(pool, target_price):
  // Get current reserves of tokens in the pool
  current_reserve_x = get_pool_reserve(pool, token_x)
  current_reserve_y = get_pool_reserve(pool, token_y)

  // Calculate current exchange rate
  current_rate = current_reserve_y / current_reserve_x

  // Set minimum swap amount
  base_p = minimum_feasible_swap_amount

  // Loop until target rate is achieved within a tolerance
  while (abs(current_rate - target_price) > tolerance):
    // Calculate discrepancy between current and target rate
    discrepancy = abs(current_rate - target_price)

    // Calculate discrepancy percentage
    discrepancy_perc = discrepancy / target_price

    // Set adjustment factor (capped at 10%)
    adjustment_factor = min(discrepancy_perc, 0.1)

    // Calculate dynamic swap amount
    dynamic_p = base_p + (base_p * adjustment_factor)

    // Simulate swap: exchange dynamic_p of token_x for token_y
    swap(pool, token_x, dynamic_p, token_y, 0)  // 0 = minimum amount received of token_y

    // Update reserves and exchange rate after swap
    current_reserve_x = current_reserve_x - dynamic_p
    current_reserve_y = current_reserve_y + dynamic_p * current_rate
    current_rate = current_reserve_y / current_reserve_x

  // End loop (target rate achieved)

// Helper functions (replace with actual implementations)
function get_pool_reserve(pool, token):
  // Simulate retrieving reserve for a specific token in the pool

function swap(pool, token_sell, amount, token_buy, min_amount_buy):
  // Simulate swap functionality (exchange tokens in the pool)

function abs(value):
  // Implement absolute value function

function min(a, b):
  // Implement minimum value function

function min(a, b):
  // Implement minimum value function
```
## Conclusion

By understanding the inherent differences between Chainlink price feeds and Uniswap V3 pool exchange rates, I have developed a method to align these rates effectively. The proposed iterative adjustment process, combined with dynamic transaction optimization, ensures minimal impact on the liquidity pool while achieving the desired exchange rate with fewer transactions. This approach not only improves the accuracy of swaps but also enhances the efficiency and cost-effectiveness of the protocol.



