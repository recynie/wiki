---
title: 斯特林数与贝尔数
date: 2026-04-06
description: 第一类斯特林数、第二类斯特林数与贝尔数的定义、性质与计算方法
tags:
  - stirling-numbers
  - combinatorics
  - bell-numbers
draft: false
permalink:
---

## 概述

**斯特林数（Stirling Numbers）** 是一类重要的组合数，用于描述集合的划分和排列的结构关系。[^1]

主要有两类：
- **第一类斯特林数**：将 $n$ 个元素排成 $k$ 个非空轮换
- **第二类斯特林数**：将 $n$ 个元素划分成 $k$ 个非空集合

## 第一类斯特林数（Stirling Numbers of the First Kind）

### 定义

记作 $s(n, k)$ 或 $\left[\begin{n}\\k\end{array}\right]$，表示将 $n$ 个不同元素排成 $k$ 个**非空轮换（Cycle）**的方案数。

**轮换**：环形排列，如 $(1, 2, 3)$ 和 $(2, 3, 1)$ 是同一个轮换。

### 递推关系

$$
s(n, k) = s(n-1, k-1) + (n-1) \cdot s(n-1, k)
$$

含义：
- 新元素单独成一个轮换：$s(n-1, k-1)$
- 新元素插入已有轮换的任意位置：$(n-1) \cdot s(n-1, k)$

### 性质

- $s(n, 1) = (n-1)!$
- $s(n, n-1) = \binom{n}{2}$
- $s(n, n) = 1$
- $s(n, 0) = 0$（当 $n > 0$）

## 第二类斯特林数（Stirling Numbers of the Second Kind）

### 定义

记作 $S(n, k)$ 或 $\left\{\begin{n}\\k\end{array}\right\}$，表示将 $n$ 个不同元素划分成 $k$ 个**非空集合（Subset）**的方案数。

### 递推关系

$$
S(n, k) = S(n-1, k-1) + k \cdot S(n-1, k)
$$

含义：
- 新元素单独成一个集合：$S(n-1, k-1)$
- 新元素加入已有 $k$ 个集合中的任意一个：$k \cdot S(n-1, k)$

### 封闭形式

$$
S(n, k) = \frac{1}{k!} \sum_{i=0}^{k} (-1)^i \binom{k}{i} (k-i)^n
$$

这是容斥原理的应用。

## 贝尔数（Bell Numbers）

### 定义

**贝尔数** $B_n$ 表示将 $n$ 个元素划分成**任意数量**非空集合的方案数：

$$
B_n = \sum_{k=0}^{n} S(n, k)
$$

### 递推关系（Dobinski公式）

$$
B_{n+1} = \sum_{k=0}^{n} \binom{n}{k} B_k
$$

### 性质

- $B_0 = 1$
- $B_1 = 1$
- $B_2 = 2$
- $B_3 = 5$
- $B_4 = 15$
- $B_5 = 52$

## 计算方法

### O(nk) 动态规划

```cpp
// 第一类斯特林数
vector<vector<long long>> stirling1(int n) {
    vector<vector<long long>> s(n+1, vector<long long>(n+1, 0));
    s[1][1] = 1;
    for (int i = 2; i <= n; i++) {
        for (int k = 1; k <= i; k++) {
            s[i][k] = (s[i-1][k-1] + (i-1) * s[i-1][k]) % MOD;
        }
    }
    return s;
}

// 第二类斯特林数
vector<vector<long long>> stirling2(int n) {
    vector<vector<long long>> S(n+1, vector<long long>(n+1, 0));
    S[1][1] = 1;
    for (int i = 2; i <= n; i++) {
        for (int k = 1; k <= i; k++) {
            S[i][k] = (S[i-1][k-1] + k * S[i-1][k]) % MOD;
        }
    }
    return S;
}
```

### O(n log n) 使用多项式

第二类斯特林数可以通过多项式求幂在 $O(n \log n)$ 内计算一行：

$$
S(n, k) = \frac{1}{k!} \sum_{i=0}^{k} (-1)^{k-i} \binom{k}{i} i^n
$$

## 应用场景

### 1. 排列分解

将 $n$ 个元素的排列分解成 $k$ 个轮换，第一类斯特林数给出方案数。

**例**：$n=3$ 的排列可以分解为：
- 1个轮换：$(123)$ → $s(3,1) = 2$ 种
- 2个轮换：$(12)(3), (13)(2), (23)(1)$ → $s(3,2) = 3$ 种
- 3个轮换：$(1)(2)(3)$ → $s(3,3) = 1$ 种

### 2. 集合划分计数

将 $n$ 个任务分配给 $k$ 个工人（工人不可区分），方案数为 $S(n, k)$。

### 3. 与其他数列的关系

- **阶乘**：$n! = \sum_{k=0}^{n} s(n, k)$
- **贝尔多项式**：指数生成函数与贝尔多项式相关

## 对照表

| $n$ | $s(n,k)$ (第一类) | $S(n,k)$ (第二类) |
|-----|-------------------|-------------------|
| 1 | $s(1,1)=1$ | $S(1,1)=1$ |
| 2 | $s(2,1)=1, s(2,2)=1$ | $S(2,1)=1, S(2,2)=1$ |
| 3 | $s(3,1)=2, s(3,2)=3, s(3,3)=1$ | $S(3,1)=1, S(3,2)=3, S(3,3)=1$ |
| 4 | $s(4,1)=6, s(4,2)=11, s(4,3)=6, s(4,4)=1$ | $S(4,1)=1, S(4,2)=7, S(4,3)=6, S(4,4)=1$ |

## 参考

[^1]: 本内容参考 [OI-Wiki 斯特林数](https://oi-wiki.org/math/combinatorics/stirling/)，内容经过验证和扩展。

---

*Last updated: 2026-04-06*