---
title: Burnside引理与Polya计数
date: 2026-04-06
description: Burnside引理（Burnside's Lemma）用于处理组合计数中的对称性，Polya计数定理是其有力推广
tags:
  - burnside
  - combinatorics
  - counting
  - group-action
draft: false
permalink:
---

## 概述

**Burnside引理**（又称Burnside's Lemma或Cauchy-Frobenius引理）是组合计数中处理**对称性**的重要工具。[^1]

核心思想：在计数有限集合的轨道数时，可以按"保持元素不动"的方式统计，而非枚举轨道。

## 基本概念

### 群作用（Group Action）

设 $G$ 是一个有限群，$X$ 是一个集合。群 $G$ 在 $X$ 上的**作用**是一个映射：

$$
G \times X \rightarrow X, \quad (g, x) \mapsto g \cdot x
$$

满足：
- $e \cdot x = x$（单位元）
- $(gh) \cdot x = g \cdot (h \cdot x)$（结合律）

### 轨道与稳定子群

- **轨道（Orbit）**：$x$ 的轨道是 $G$ 作用下所有能到达的元素集合
- **稳定子群（Stabilizer）**：使 $x$ 不动的元素集合 $G_x = \{g \in G \mid g \cdot x = x\}$

### 不动点

对于 $g \in G$，记 $X^g = \{x \in X \mid g \cdot x = x\}$ 为 $g$ 的**不动点**集合。

## Burnside引理

设有限群 $G$ 作用在有限集合 $X$ 上，则 $X$ 的轨道数为：

$$
|X/G| = \frac{1}{|G|} \sum_{g \in G} |X^g|
$$

即：**轨道数等于各元素不动点数目的平均值**。

### 证明思路

由轨道-稳定子群定理：$|G| = |Orbit(x)| \cdot |G_x|$，对每个轨道求和可得。

## Polya计数定理

Polya定理是Burnside引理的推广，用**置换群**的**共轭类**来简化计算。

### 置换的轮换表示

将 $1 \sim n$ 的置换写成轮换分解形式，如 $(1)(2 \ 3)(4 \ 5 \ 6)$ 包含：
- 1个1-轮换
- 1个2-轮换
- 1个3-轮换

记 $c_k(g)$ 为置换 $g$ 中 $k$-轮换的个数。

### Polya定理

设置换群 $G$ 作用于 $n$ 个对象，用 $m$ 种颜色染色，则染色方案数为：

$$
Z(G, m) = \frac{1}{|G|} \sum_{g \in G} m^{c_1(g) + c_2(g) + \cdots + c_n(g)}
= \frac{1}{|G|} \sum_{g \in G} m^{c(g)}
$$

其中 $c(g)$ 是置换 $g$ 的轮换总数。

## 应用示例

### 例1：项链旋转计数

**问题**：用 $m$ 种颜色给 $n$ 颗珠子的项链染色，考虑旋转但不考虑翻转，有多少种方案？

**分析**：旋转群 $G = \mathbb{Z}_n = \{0, 1, \ldots, n-1\}$，其中旋转 $k$ 将第 $i$ 颗珠子移到第 $i+k \pmod{n}$ 颗。

旋转 $k$ 的不动点要求所有珠子颜色相同，因此：
- 当 $k = 0$（恒等旋转）：$m^n$ 个不动点
- 当 $k \neq 0$：若 $n$ 和 $k$ 互质，轮换数为 $n$（每个珠子单独一轮换）；否则轮换数为 $\gcd(n, k)$

由Burnside引理：

$$
\text{答案} = \frac{1}{n} \sum_{k=0}^{n-1} m^{\gcd(n, k)}
$$

### 例2：正方形旋转群染色

**问题**：用 $m$ 种颜色给正方形的四个顶点染色，考虑旋转但不考虑翻转，有多少种方案？

**分析**：旋转群 $G = \{0°, 90°, 180°, 270°\}$：

| 旋转角度 | 轮换结构 | 不动点个数 |
|---------|---------|-----------|
| $0°$（恒等） | 4个1-轮换 | $m^4$ |
| $90°$ | 1个4-轮换 | $m$ |
| $180°$ | 2个2-轮换 | $m^2$ |
| $270°$ | 1个4-轮换 | $m$ |

答案：

$$
\frac{m^4 + m^2 + 2m}{4}
$$

### 例3：立方体染色（考虑翻转）

立方体的12个面转动，包括：
- 恒等：1个，$m^6$ 不动点
- 90°面中心旋转：6个，各 $m^3$ 不动点
- 180°面中心旋转：3个，各 $m^4$ 不动点
- 120°顶点旋转：8个，各 $m^2$ 不动点

由Polya定理可得总方案数。

## 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

long long mod_pow(long long a, long long e) {
    long long r = 1;
    while (e) {
        if (e & 1) r = r * a % MOD;
        a = a * a % MOD;
        e >>= 1;
    }
    return r;
}

// 计算置换的轮换数
int count_cycles(int n, const vector<int>& perm) {
    vector<int> vis(n, 0);
    int cycles = 0;
    for (int i = 0; i < n; i++) {
        if (!vis[i]) {
            cycles++;
            int j = i;
            while (!vis[j]) {
                vis[j] = 1;
                j = perm[j];
            }
        }
    }
    return cycles;
}

// Burnside引理：计算轨道数
long long burnside(int n, const vector<vector<int>>& group) {
    long long sum = 0;
    for (const auto& perm : group) {
        sum += mod_pow(m, count_cycles(n, perm));
    }
    return sum / group.size() % MOD;
}
```

## 常见置换群

| 群 | 对象数 | 置换类型 | 轮换结构 |
|----|--------|---------|---------|
| 项链旋转 | $n$ | 旋转 $k$ 位 | $\gcd(n, k)$ 个轮换 |
| 项链翻转 | $n$ | 翻转 | $n$ 为奇数：$(n+1)/2$ 个2-轮换；$n$ 为偶数：$n/2+1$ 个2-轮换 |
| 正方形翻转 | 4 | 沿对角线/中线 | 2个1-轮换+1个2-轮换，或2个2-轮换 |
| 立方体旋转 | 6面 | 面中心/顶点旋转 | 多种结构 |

## 参考

[^1]: 本内容参考 [OI-Wiki Burnside引理与Polya计数](https://oi-wiki.org/math/combinatorics/burnside/)，内容经过验证和扩展。

---

*Last updated: 2026-04-06*