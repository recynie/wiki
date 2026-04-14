---
title: 背包问题
date: 2026-04-01
description: 背包问题（Knapsack）动态规划详解
tags:
  - dynamic-planning
  - knapsack
draft: false
permalink:
---

## 定义

背包问题（Knapsack Problem）是动态规划中最经典的入门问题之一。给定 $n$ 件物品，每件物品 $i$ 有重量 $w_i$ 和价值 $v_i$，以及一个容量为 $W$ 的背包。求解如何选择物品装入背包，在不超过背包容量的前提下，使背包中物品的总价值最大。[^1]

## 核心概念

### 状态定义

设 $f[i][j]$ 表示考虑前 $i$ 件物品，背包容量为 $j$ 时能获得的最大价值。答案为 $f[n][W]$。

状态转移方程：

$$
f[i][j] = \max(f[i-1][j], \; f[i-1][j-w_i] + v_i)
$$

其中：
- $f[i-1][j]$ 表示不选第 $i$ 件物品
- $f[i-1][j-w_i] + v_i$ 表示选第 $i$ 件物品（前提是 $j \geq w_i$）

### 空间优化

观察到状态转移只与 $i-1$ 有关，可以将二维数组压缩为一维数组。但需要注意遍历顺序：

**0-1 背包**：容量需要**逆序**遍历，保证每个物品只被使用一次。

**完全背包**：容量需要**正序**遍历，允许物品重复使用。

时间复杂度为 $O(n \times W)$，空间复杂度为 $O(W)$。

---

## 0-1 背包

### 问题描述

每件物品只能选择**一次**（选或不选），求解最大价值。

### 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    int n, W;
    cin >> n >> W;
    vector<int> w(n+1), v(n+1);
    for (int i = 1; i <= n; i++) cin >> w[i] >> v[i];
    vector<int> f(W+1, 0);
    for (int i = 1; i <= n; i++)
        for (int j = W; j >= w[i]; j--)
            f[j] = max(f[j], f[j-w[i]] + v[i]);
    cout << f[W] << endl;
    return 0;
}
```

### 为什么逆序遍历？

假设物品重量 $w = 2$，价值 $v = 3$，背包容量 $W = 5$。

如果正序遍历（错误）：
- $j = 2$ 时：$f[2] = \max(f[2], f[0] + 3) = 3$
- $j = 3$ 时：$f[3]$ 不变
- $j = 4$ 时：$f[4] = \max(f[4], f[2] + 3) = \max(0, 3+3) = 6$

此时 $f[4]$ 使用了两次物品，违反了 0-1 背包的限制。

逆序遍历则保证先更新大的容量，再更新小的容量，$f[2]$ 在 $j = 4$ 时仍是上一轮的值：

- $j = 5$ 时：$f[5] = \max(f[5], f[3] + 3)$
- $j = 4$ 时：$f[4] = \max(f[4], f[2] + 3)$，$f[2]$ 是**旧值**
- $j = 3$ 时：$f[3] = \max(f[3], f[1] + 3)$
- $j = 2$ 时：$f[2] = \max(f[2], f[0] + 3) = 3$

### 模板

```cpp
vector<int> f(W+1, 0);
for (int i = 1; i <= n; i++)
    for (int j = W; j >= w[i]; j--)
        f[j] = max(f[j], f[j-w[i]] + v[i]);
```

---

## 完全背包

### 问题描述

每件物品可以选择**无限次**，求解最大价值。

### 与 0-1 背包的区别

完全背包的状态转移方程为：

$$
f[j] = \max(f[j], \; f[j-w_i] + v_i) \quad (j \geq w_i)
$$

与 0-1 背包的区别在于：**正序遍历**，使得在更新 $f[j]$ 时，$f[j-w_i]$ 已经是本轮更新后的值（即考虑了当前物品多次使用）。

### 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    int n, W;
    cin >> n >> W;
    vector<int> w(n+1), v(n+1);
    for (int i = 1; i <= n; i++) cin >> w[i] >> v[i];
    vector<int> f(W+1, 0);
    // 完全背包 - 正序遍历
    for (int i = 1; i <= n; i++)
        for (int j = w[i]; j <= W; j++)
            f[j] = max(f[j], f[j-w[i]] + v[i]);
    cout << f[W] << endl;
    return 0;
}
```

### 模板

```cpp
// Complete knapsack - items can be taken unlimited times
// Loop direction is FORWARD (opposite of 0-1 knapsack)
for (int i = 1; i <= n; i++)
    for (int j = w[i]; j <= W; j++)
        f[j] = max(f[j], f[j-w[i]] + v[i]);
```

### 对比总结

| 类型 | 遍历方向 | 含义 |
|------|----------|------|
| 0-1 背包 | 逆序 | 每个物品只用一次 |
| 完全背包 | 正序 | 每个物品可用无限次 |

---

## 多重背包

### 问题描述

第 $i$ 件物品最多只能选 $k_i$ 次，求解最大价值。

### 朴素解法

```cpp
for (int i = 1; i <= n; i++)
    for (int j = W; j >= w[i]; j--)
        for (int t = 1; t <= k[i] && j >= t * w[i]; t++)
            f[j] = max(f[j], f[j-t*w[i]] + t*v[i]);
```

时间复杂度为 $O(n \times W \times k)$，当 $k$ 较大时效率低下。

### 二进制分组优化

将物品数量 $k$ 拆分成若干个"二进制分组"：

$$
k = 1 + 2 + 4 + \dots + 2^{p} + remainder
$$

每个分组视为一个 0-1 背包的物品。例如 $k = 13$，可以拆分为 $\{1, 2, 4, 6\}$ 四个分组，总组合数仍然可以表示 $0 \sim 13$ 的任意数量。

优化后时间复杂度降为 $O(W \times \sum \log k_i)$。

### 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Item {
    int w, v;
};

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    int m, W;
    cin >> m >> W;  // m: 物品种类数，W: 背包容量
    vector<Item> list;  // 拆分后的物品列表
    list.reserve(1000);
    
    for (int i = 1; i <= m; i++) {
        int p, h, k;
        cin >> p >> h >> k;  // p: 重量，h: 价值，k: 数量
        int c = 1;
        while (k - c > 0) {
            k -= c;
            list.push_back({c * p, c * h});
            c *= 2;
        }
        // 剩余部分
        list.push_back({p * k, h * k});
    }
    
    // 转换为 0-1 背包
    vector<int> f(W+1, 0);
    for (auto& item : list) {
        for (int j = W; j >= item.w; j--)
            f[j] = max(f[j], f[j-item.w] + item.v);
    }
    
    cout << f[W] << endl;
    return 0;
}
```

### 二进制分组模板

```cpp
// Binary grouping optimization for multi-knapsack
int index = 0;
for (int i = 1; i <= m; i++) {
    int c = 1, p, h, k;
    cin >> p >> h >> k;  // price, value, count
    while (k - c > 0) {
        k -= c;
        list[++index].w = c * p;
        list[index].v = c * h;
        c *= 2;
    }
    list[++index].w = p * k;
    list[index].v = h * k;
}
```

---

## 三种背包对比

| 类型 | 物品限制 | 遍历顺序 | 时间复杂度 |
|------|----------|----------|------------|
| 0-1 背包 | 选 0 或 1 次 | 逆序 | $O(n \times W)$ |
| 完全背包 | 无限次 | 正序 | $O(n \times W)$ |
| 多重背包 | 有限次 $k$ | 逆序 + 二进制优化 | $O(W \times \sum \log k_i)$ |

---

## 典型应用

### 1. 恰好装满 vs 可以不装满

上述代码默认**可以不装满**（$f[0] = 0$）。

若要求**恰好装满**，需初始化为负无穷：

```cpp
vector<int> f(W+1, -INF);
f[0] = 0;
```

### 2. 求方案数

将 $\max$ 换成求和：

```cpp
vector<int> f(W+1, 0);
f[0] = 1;
for (int i = 1; i <= n; i++)
    for (int j = W; j >= w[i]; j--)
        f[j] += f[j-w[i]];
```

### 3. 依赖背包

物品之间存在依赖关系（如选了物品 $i$ 才能选物品 $j$），可以将依赖关系转化为"01 背包 + 分组背包"的组合形式。

---

## 总结

背包问题是动态规划的基石。核心在于：

1. **状态定义**：$f[i][j]$ 表示前 $i$ 件物品、容量 $j$ 下的最优价值
2. **状态转移**：$\max$ 决策，选或不选
3. **空间优化**：一维数组 + 遍历顺序控制
4. **多重背包**：二进制分组转化为 0-1 背包

掌握这三种背包问题后，可以进一步学习[[../data-structure/segment-tree|单调栈]]等更复杂的 DP 优化技巧。

---

[^1]: 背包问题九讲 - 崔添翼 https://github.com/tianyicui/pack
