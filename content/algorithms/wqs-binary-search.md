---
title: WQS二分与凸优化（Aliens Trick）
date: 2026-04-06
description: WQS二分（Aliens Trick）是一种将带限制的DP优化问题转化为无限制凸代价函数优化的技巧
tags:
  - wqs-binary-search
  - dp-optimization
  - convex-optimization
  - aliens-trick
draft: false
permalink:
---

## 概述

**WQS二分**（Wasserstrom-Spjudovich Binary Search），又称 **Aliens Trick**，是解决如下问题的高效技巧：

> 在DP中，要求恰好选择 $k$ 个物品，使得总价值最大。如果对"选择物品数"没有限制时可以直接贪心，但加上恰好 $k$ 个的限制时如何处理？

核心思想：将"恰好选 $k$ 个"的约束转化为"每选一个物品惩罚 $\lambda$"，通过二分 $\lambda$ 找到最优解的 $k$ 值。

## 算法思想

设原问题的解为 $f(k)$，即选恰好 $k$ 个物品的最大价值。

定义 $g(\lambda) = \max\{\text{价值} - \lambda \times \text{选中的物品数}\}$（无限制选物品数，但每选一个扣减 $\lambda$）。

### 关键性质

- $g(\lambda)$ 是关于 $\lambda$ 的**凸函数**（或凹函数，取决于问题）
- $f(k)$ 是 $g(\lambda)$ 的**对偶函数**
- 通过二分 $\lambda$，可以找到对应恰好 $k$ 个物品的解

### 典型形式

$$
\text{最大化} \quad \sum v_i - \lambda \cdot \text{count}
$$

当 $\lambda$ 增加时，选择的物品数减少。通过二分找到使得选择数为 $k$ 的 $\lambda^*$。

## 代码框架

```cpp
// 假设 dp(state) 返回 (maxValue, count) 的最大值贪心解
pair<long long, int> solveWithPenalty(double lambda) {
    // 返回：最大价值（减去惩罚后），选择的物品数
    long long val = 0;
    int cnt = 0;
    // 根据问题，贪心地选择或DP
    for (each option) {
        if (benefit - lambda * cost > 0) {
            val += benefit - lambda * cost;
            cnt += cost;
        }
    }
    return {val, cnt};
}

double wqsBinarySearch(int targetK) {
    double lo = MIN_LAMBDA, hi = MAX_LAMBDA;
    for (int iter = 0; iter < 60; iter++) {  // 二分迭代确保精度
        double mid = (lo + hi) / 2;
        auto [val, cnt] = solveWithPenalty(mid);
        if (cnt > targetK) {
            lo = mid;  // 选多了，需要增大惩罚
        } else {
            hi = mid;  // 选少了，需要减小惩罚
        }
    }
    return (lo + hi) / 2;
}
```

## 经典应用

### 1. 序列分组（Partition Array）

**问题**：将数组分成 $k$ 段，每段和的最大值最小。

**转化**：惩罚每增加一段。

```cpp
// 最大化 sum - λ * segments
// 二分 λ，找到恰好 k 段的解

int n, k;
vector<long long> a(n+1);

// check: 在惩罚 λ 下，最多能分多少段
int check(double λ) {
    int segments = 0;
    long long sum = 0;
    for (int i = 1; i <= n; i++) {
        if (sum + a[i] > λ) {
            segments++;
            sum = a[i];
        } else {
            sum += a[i];
        }
    }
    return segments;
}
```

### 2. 编辑距离变种

**问题**：给定字符串，每次操作可以删除一个字符，要求恰好删除 $k$ 个字符使得剩余字符串字典序最小。

**转化**：每删除一个字符惩罚 $\lambda$，二分找到最优。

### 3. 背包DP优化

**问题**：选恰好 $k$ 件物品的背包 DP，要求价值最大。

```cpp
// dp[i] = 选择 i 件物品的最大价值
// 惩罚 λ 后变成：dp'[i] = max(dp[i] - λ * i)
// 这是一个单调队列优化的DP
```

## 凸性证明

设 $S$ 是所有可行解的集合，$val(x)$ 是解 $x$ 的价值，$cnt(x)$ 是解 $x$ 选择的物品数。

定义 $g(\lambda) = \max_{x \in S} \{val(x) - \lambda \cdot cnt(x)\}$。

**引理**：$g(\lambda)$ 是凸函数。

**证明**：对于任意 $\lambda_1, \lambda_2$ 和 $t \in [0, 1]$：

$$
\begin{aligned}
g(t\lambda_1 + (1-t)\lambda_2)
&= \max_x \{val(x) - (t\lambda_1 + (1-t)\lambda_2) \cdot cnt(x)\} \\
&= \max_x \{t(val(x) - \lambda_1 \cdot cnt(x)) + (1-t)(val(x) - \lambda_2 \cdot cnt(x))\} \\
&\leq t \max_x \{val(x) - \lambda_1 \cdot cnt(x)\} + (1-t) \max_x \{val(x) - \lambda_2 \cdot cnt(x)\} \\
&= t g(\lambda_1) + (1-t) g(\lambda_2)
\end{aligned}
$$

因此 $g(\lambda)$ 是凸函数。

## 例题：K 件物品的价值最大化

**题目**（CF 319C 简化）：有 $n$ 件物品，每件有价值 $v_i$。选恰好 $k$ 件物品，总价值最大。

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

const int MAXN = 100005;
int n, k;
ll v[MAXN];
ll dp[MAXN];  // dp[i] = 选 i 件物品的最大价值

// 带惩罚 λ 的贪心选择
pair<ll, int> greedyWithPenalty(ll λ) {
    ll val = 0;
    int cnt = 0;
    // 按单位价值排序，选择 v_i - λ > 0 的物品
    vector<pair<ll, int>> idx;
    for (int i = 1; i <= n; i++) {
        idx.push_back({v[i], i});
    }
    sort(idx.begin(), idx.end(), [](auto& a, auto& b) {
        return a.first > b.first;
    });
    
    for (auto& p : idx) {
        ll benefit = p.first;
        if (benefit > λ && cnt < k) {
            val += benefit - λ;
            cnt++;
        }
    }
    return {val, cnt};
}
```

## 注意事项

1. **精度问题**：使用 long long 时要注意精度损失，二分迭代60次通常足够
2. **边界处理**：$\lambda$ 的上下界需要根据问题确定
3. **不唯一解**：如果存在多个最优解，需要额外处理

## 与其他优化方法的关系

| 方法 | 适用场景 | 复杂度 |
|------|---------|--------|
| WQS二分 | 限制选择数量的凸优化 | $O(n \log C)$ |
| 单调队列 | 满足四边形不等式的DP | $O(n)$ |
| 斜率优化 | DP代价函数是线性函数 | $O(n)$ |
| 凸包维护 | 凸代价函数的DP | $O(n \log n)$ |

## 参考

[^1]: 本内容参考 [CF WQS Binary Search Explanation](https://codeforces.com/blog/entry/8219) 及竞赛常用技巧，内容经过验证和扩展。

---

*Last updated: 2026-04-06*