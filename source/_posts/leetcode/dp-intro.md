---
title: LeetCode 动态规划入门：从斐波那契到背包问题
tags:
  - LeetCode
  - 动态规划
  - 算法
  - 题解
categories:
  - LeetCode
cover: https://images.unsplash.com/photo-1509228468518-180dd4864904?w=1280
description: 动态规划是 LeetCode 面试高频考点。本文从最简单的斐波那契数列开始，逐步过渡到经典的 0-1 背包问题。
katex: true
mermaid: false
mathjax: false
abbrlink: 3405
date: 2025-05-15 10:00:00
top_img:
---

# LeetCode 动态规划入门

动态规划（Dynamic Programming, DP）是算法面试中的高频考点。本文带你从零开始掌握 DP。

## 核心思想

1. **定义状态**：确定 `dp[i]` 代表什么
2. **状态转移方程**：`dp[i]` 和 `dp[j]` 之间的关系
3. **边界条件**：确定初始值
4. **计算顺序**：确保计算 `dp[i]` 时，所依赖的状态已经计算完成

## Level 1: 斐波那契数列

[LeetCode 509 - Fibonacci Number](https://leetcode.com/problems/fibonacci-number/)

$$F(n) = F(n-1) + F(n-2), \quad F(0)=0, F(1)=1$$

```java
class Solution {
    public int fib(int n) {
        if (n <= 1) return n;
        int prev = 0, curr = 1;
        for (int i = 2; i <= n; i++) {
            int next = prev + curr;
            prev = curr;
            curr = next;
        }
        return curr;
    }
}
```

**复杂度**：时间 $O(n)$，空间 $O(1)$

## Level 2: 爬楼梯

[LeetCode 70 - Climbing Stairs](https://leetcode.com/problems/climbing-stairs/)

```java
class Solution {
    public int climbStairs(int n) {
        if (n <= 2) return n;
        int prev1 = 1, prev2 = 2;
        for (int i = 3; i <= n; i++) {
            int curr = prev1 + prev2;
            prev1 = prev2;
            prev2 = curr;
        }
        return prev2;
    }
}
```

## Level 3: 零钱兑换

[LeetCode 322 - Coin Change](https://leetcode.com/problems/coin-change/)

```java
class Solution {
    public int coinChange(int[] coins, int amount) {
        int[] dp = new int[amount + 1];
        Arrays.fill(dp, amount + 1);
        dp[0] = 0;

        for (int i = 1; i <= amount; i++) {
            for (int coin : coins) {
                if (coin <= i) {
                    dp[i] = Math.min(dp[i], dp[i - coin] + 1);
                }
            }
        }

        return dp[amount] > amount ? -1 : dp[amount];
    }
}
```

**状态转移**：$dp[i] = \min_{coin \leq i}(dp[i - coin] + 1)$

## Level 4: 0-1 背包问题

经典背包问题：给定 $n$ 个物品，每个物品有重量 $w_i$ 和价值 $v_i$，背包容量为 $W$，求最大价值。

```java
public int knapsack(int[] weights, int[] values, int W) {
    int n = weights.length;
    int[][] dp = new int[n + 1][W + 1];

    for (int i = 1; i <= n; i++) {
        for (int w = 1; w <= W; w++) {
            if (weights[i - 1] <= w) {
                dp[i][w] = Math.max(
                    dp[i - 1][w],
                    dp[i - 1][w - weights[i - 1]] + values[i - 1]
                );
            } else {
                dp[i][w] = dp[i - 1][w];
            }
        }
    }

    return dp[n][W];
}
```

### 空间优化（一维 DP）

```java
public int knapsackOptimized(int[] weights, int[] values, int W) {
    int n = weights.length;
    int[] dp = new int[W + 1];

    for (int i = 0; i < n; i++) {
        // 逆序遍历，确保每个物品只选一次
        for (int w = W; w >= weights[i]; w--) {
            dp[w] = Math.max(dp[w], dp[w - weights[i]] + values[i]);
        }
    }

    return dp[W];
}
```

## DP 题型总结

| 题型 | 代表题目 | 状态定义 |
|------|----------|----------|
| 线性 DP | 最长递增子序列 | `dp[i]` = 以 i 结尾的 LIS 长度 |
| 区间 DP | 戳气球 | `dp[i][j]` = 区间 [i,j] 的最优解 |
| 背包 DP | 零钱兑换 | `dp[i]` = 容量 i 的最优解 |
| 树形 DP | 打家劫舍 III | `dp[node][0/1]` = 选/不选当前节点 |
| 状态压缩 DP | 旅行商问题 | `dp[mask]` = 二进制状态 |

## 刷题建议

1. **先理解再做题**：每道题先花 5 分钟想状态定义和转移方程
2. **从简单开始**：先做 Easy，再做 Medium，最后 Hard
3. **写题解**：做完后用自己的话写一遍思路
4. **定期复习**：一周后重新做一遍，看看是否还有印象

---

> 推荐资源：
> - [labuladong 的算法笔记](https://labuladong.online/algo/)
> - [代码随想录](https://programmercarl.com/)
> - [动态规划专题 - LeetCode](https://leetcode.com/tag/dynamic-programming/)
