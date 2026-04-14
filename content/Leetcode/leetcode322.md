---
title: 零钱兑换 (LeetCode 322)
date: 2026-04-11
description: 使用动态规划解决完全背包问题
tags:
  - dynamic-programming
  - knapsack
draft: false
permalink:
---

## 零钱兑换 (LeetCode 322)

给定 `n` 种硬币，面值分别为 `coins[i]`，每种硬币可以无限次使用。给定一个总金额 `amount`，求凑成该金额所需的**最少硬币数**。若无法凑成，返回 `-1`。

### 算法思路：动态规划（完全背包）

将问题转化为完全背包问题：

- **状态**：`dp[i]` 表示凑成金额 `i` 所需的最少硬币数
- **初始状态**：`dp[0] = 0`，金额为 0 需要 0 枚硬币；其余为 `INF`（无法凑成）
- **转移方程**：遍历每种硬币，对于每个金额 `j`，尝试使用当前硬币：
  $$dp[j] = \min(dp[j], dp[j - coin] + 1)$$

### 完全背包 vs 0-1 背包

- **0-1 背包**：每件物品只能选一次，内层循环倒序遍历
- **完全背包**：每种物品可以选无限次，内层循环正序遍历

本题为完全背包问题，内层循环正序。

### C++ 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

class Solution {
public:
    int coinChange(vector<int>& coins, int amount) {
        const int INF = amount + 1;  // 硬币数不可能超过amount
        vector<int> dp(amount + 1, INF);
        dp[0] = 0;
        
        for (int coin : coins) {
            for (int j = coin; j <= amount; ++j) {
                dp[j] = min(dp[j], dp[j - coin] + 1);
            }
        }
        
        return dp[amount] == INF ? -1 : dp[amount];
    }
};
```

### 优化版本：按金额外层循环

```cpp
class Solution {
public:
    int coinChange(vector<int>& coins, int amount) {
        vector<int> dp(amount + 1, amount + 1);
        dp[0] = 0;
        
        for (int i = 1; i <= amount; ++i) {
            for (int coin : coins) {
                if (i - coin >= 0) {
                    dp[i] = min(dp[i], dp[i - coin] + 1);
                }
            }
        }
        
        return dp[amount] > amount ? -1 : dp[amount];
    }
};
```

### 复杂度分析

- **时间复杂度**：$O(n \times amount)$，n 为硬币种类数
- **空间复杂度**：$O(amount)$

### 参考

- [[dynamic-programming|动态规划]]
- [[knapsack|背包问题]]
