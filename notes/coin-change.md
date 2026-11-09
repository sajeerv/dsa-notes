# Coin Change DP: Quick Reference

First identify what the problem asks: **minimum coins**, **number of combinations**, **number of ordered sequences**, or **whether an amount is reachable**. The state may look similar, but the transition and loop order matter.

Assume coin values are positive integers and the amount is nonnegative.

## The common variants

| Problem | DP meaning | Initialization | Transition |
|---|---|---|---|
| Minimum coins, unlimited supply | `dp[a]` = fewest coins to make `a` | `dp[0] = 0`, others unreachable | `dp[a] = min(dp[a], dp[a-c] + 1)` |
| Count combinations, unlimited supply | `dp[a]` = combinations to make `a` | `dp[0] = 1` | `dp[a] += dp[a-c]` |
| Count ordered sequences, unlimited supply | `dp[a]` = sequences to make `a` | `dp[0] = 1` | `dp[a] += dp[a-c]` |
| Reachability, unlimited supply | `dp[a]` = can make `a`? | `dp[0] = true` | `dp[a] |= dp[a-c]` |
| 0/1 use of each coin once | depends on objective | base case for amount zero | update amounts **downward** |

## 1. Minimum number of coins (unlimited supply)

Example: LeetCode 322, Coin Change. Return `-1` if the amount cannot be made.

```java
int[] dp = new int[amount + 1];
Arrays.fill(dp, amount + 1); // unreachable sentinel
dp[0] = 0;

for (int a = 1; a <= amount; a++) {
    for (int coin : coins) {
        if (coin <= a) {
            dp[a] = Math.min(dp[a], dp[a - coin] + 1);
        }
    }
}

return dp[amount] > amount ? -1 : dp[amount];
```

**Time:** `O(amount × numberOfCoins)` · **Space:** `O(amount)`

For a minimum, either amount-first or coin-first iteration works with unlimited coins. The recurrence looks back to a smaller amount, so each coin can be reused.

## 2. Count combinations (unlimited supply, order does not matter)

Example: LeetCode 518, Coin Change II. `[1, 2]` and `[2, 1]` are the same combination.

```java
int[] dp = new int[amount + 1];
dp[0] = 1; // one way to make zero: choose no coins

for (int coin : coins) {
    for (int a = coin; a <= amount; a++) { // upward
        dp[a] += dp[a - coin];
    }
}

return dp[amount];
```

**Time:** `O(amount × numberOfCoins)` · **Space:** `O(amount)`

Putting the **coin loop outside** ensures each combination is counted once, regardless of coin order. Going upward allows the current coin to be reused.

## 3. Count ordered sequences (unlimited supply, order matters)

Example: Combination Sum IV. `[1, 2]` and `[2, 1]` are different sequences.

For `nums = [1, 2]`, `target = 3`:

- **Coin Change II:** `{1+1+1, 1+2}` = 2 combinations. The orders `1+2` and `2+1` are equivalent.
- **Combination Sum IV:** `[1,1,1]`, `[1,2]`, `[2,1]` = 3 sequences. Different orders count separately.

```java
int[] dp = new int[target + 1];
dp[0] = 1;

for (int a = 1; a <= target; a++) {
    for (int coin : nums) {
        if (coin <= a) {
            dp[a] += dp[a - coin];
        }
    }
}

return dp[target];
```

**Time:** `O(target × numberOfCoins)` · **Space:** `O(target)`

Putting the **amount loop outside** counts ways to reach each amount by choosing the last coin, so different orders count separately. Watch for integer overflow if the number of sequences can be large.

**Memory aid:** coin loop outside → combinations; amount loop outside → ordered sequences.

## 4. Reachability (unlimited supply)

```java
boolean[] dp = new boolean[target + 1];
dp[0] = true;

for (int coin : coins) {
    for (int a = coin; a <= target; a++) { // upward: reuse allowed
        dp[a] |= dp[a - coin];
    }
}

return dp[target];
```

If each coin can be used **once**, iterate downward instead:

```java
for (int value : values) {
    for (int a = target; a >= value; a--) {
        dp[a] |= dp[a - value];
    }
}
```

Downward iteration ensures `dp[a - value]` still refers to the previous item, rather than reusing the current item. This is the 0/1 subset-sum pattern.

## Loop-order rule

| Goal | Outer loop | Amount direction | Meaning |
|---|---|---|---|
| Count unordered combinations, unlimited | Coins | Upward | Each set of coins counted once |
| Count ordered sequences, unlimited | Amounts | Upward via recurrence | Different orders count separately |
| Minimum coins, unlimited | Either | Recurrence from smaller amounts | Reuse is allowed |
| Use each item at most once | Items | Downward | Current item cannot feed itself |

## Common mistakes

- Confusing combinations with permutations: `[1, 2]` vs `[2, 1]` tells you whether order matters.
- Initializing count DP with `dp[0] = 0`. It should be `1`: there is one way to make zero by choosing nothing.
- Iterating upward in a 0/1 problem, accidentally reusing the same item.
- Using `int` when the number of ways may exceed `Integer.MAX_VALUE`; use `long` if allowed.
- Forgetting that the minimum-coins sentinel must be safely larger than any possible answer.
- Mixing up “unlimited coins” with “each listed coin can be used once.” Check the statement.

## Fast checklist

1. Is the goal minimum, count, or reachability?
2. Does order matter?
3. Is supply unlimited or 0/1/bounded?
4. Define `dp[amount]` and initialize the zero amount.
5. Choose loop nesting and direction to match the interpretation.

## Related LeetCode patterns

- **322 Coin Change:** minimum coins, unlimited supply.
- **518 Coin Change II:** unordered combinations, unlimited supply.
- **377 Combination Sum IV:** ordered sequences, unlimited supply.
- **416 Partition Equal Subset Sum:** reachability, each number used at most once.
