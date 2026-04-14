---
title: 爬楼梯 (LeetCode 70)
date: 2026-04-11
description: 使用动态规划求解爬楼梯问题
tags:
  - dynamic-programming
  - fibonacci
draft: false
permalink:
---

## 爬楼梯 (LeetCode 70)

假设你正在爬楼梯。每次你可以爬 `1` 个或 `2` 个台阶。到达楼顶前，你有 `n` 个台阶。计算有多少种不同的方法可以爬到楼顶。

### 问题分析

到达第 `n` 级台阶的方法数取决于：

- 先爬 `1` 级，再爬 `n-1` 级的方法数
- 先爬 `2` 级，再爬 `n-2` 级的方法数

因此：$dp[n] = dp[n-1] + dp[n-2]$

这与**斐波那契数列**完全相同。

### 动态规划解法

**状态定义**：`dp[i]` 表示爬到第 `i` 级台阶的方法数

**初始状态**：
- `dp[0] = 1`（0 级台阶：一种方法，什么都不做）
- `dp[1] = 1`（1 级台阶：只能爬 1 级）

**转移方程**：`dp[i] = dp[i-1] + dp[i-2]`

### C++ 代码实现

#### 标准 DP（空间 O(n)）

```cpp
#include <bits/stdc++.h>
using namespace std;

class Solution {
public:
    int climbStairs(int n) {
        if (n <= 1) return 1;
        vector<int> dp(n + 1, 0);
        dp[0] = 1;
        dp[1] = 1;
        
        for (int i = 2; i <= n; ++i) {
            dp[i] = dp[i - 1] + dp[i - 2];
        }
        return dp[n];
    }
};
```

#### 空间优化（空间 O(1)）

```cpp
class Solution {
public:
    int climbStairs(int n) {
        if (n <= 1) return 1;
        int prev1 = 1, prev2 = 1;  // dp[1], dp[0]
        int curr = 0;
        
        for (int i = 2; i <= n; ++i) {
            curr = prev1 + prev2;
            prev2 = prev1;
            prev1 = curr;
        }
        return prev1;
    }
};
```

#### 矩阵快速幂（时间 O(log n)）

利用斐波那契矩阵公式：

$$
\begin{pmatrix} dp[n] \\ dp[n-1] \end{pmatrix} = 
\begin{pmatrix} 1 & 1 \\ 1 & 0 \end{pmatrix}^{n-1} \begin{pmatrix} 1 \\ 1 \end{pmatrix}
$$

```cpp
class Solution {
public:
    vector<vector<long long>> matrixMul(vector<vector<long long>>& A, 
                                          vector<vector<long long>>& B) {
        int n = A.size();
        vector<vector<long long>> C(n, vector<long long>(n, 0));
        for (int i = 0; i < n; ++i) {
            for (int k = 0; k < n; ++k) {
                if (A[i][k] == 0) continue;
                for (int j = 0; j < n; ++j) {
                    C[i][j] += A[i][k] * B[k][j];
                }
            }
        }
        return C;
    }
    
    vector<vector<long long>> matrixPow(vector<vector<long long>> A, int n) {
        vector<vector<long long>> result = {{1, 0}, {0, 1}};
        while (n > 0) {
            if (n & 1) result = matrixMul(result, A);
            A = matrixMul(A, A);
            n >>= 1;
        }
        return result;
    }
    
    int climbStairs(int n) {
        if (n <= 1) return 1;
        vector<vector<long long>> base = {{1, 1}, {1, 0}};
        vector<vector<long long>> M = matrixPow(base, n - 1);
        return M[0][0] + M[0][1];  // dp[n] = M[0][0]*dp[1] + M[0][1]*dp[0]
    }
};
```

### 复杂度分析

| 方法 | 时间复杂度 | 空间复杂度 |
|------|-----------|-----------|
| 标准 DP | $O(n)$ | $O(n)$ |
| 空间优化 | $O(n)$ | $O(1)$ |
| 矩阵快速幂 | $O(\log n)$ | $O(1)$ |

### 参考

- [[dynamic-programming|动态规划]]
- [[fibonacci|斐波那契数列]]
