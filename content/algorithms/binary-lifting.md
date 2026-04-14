---
title: 倍增 (binary-lifting)
date: 2026-04-01
description: 倍增算法详解——通过预处理加速树上查询的经典技巧
tags:
  - binary-lifting
  - lca
  - algorithm
draft: false
permalink:
---

## 1. 定义

**倍增**（Binary Lifting）是一种常用的算法优化技巧，其核心思想是：**将问题分解为「二進制拆分」的子问题，通过预处理实现 $O(\log n)$ 的查询效率**。

倍增法广泛应用于树上的路径查询、区间最值查询等问题。例如，在求最近公共祖先（[[depth-first-search|LCA]]）、K 步祖先、固定区间最值（RMQ）等场景中，倍增都能提供简洁而高效的解决方案。[^1]

## 2. 核心思想

### 2.1 预处理 $2^j$ 的力量

对于树中的每个节点 $i$，我们预处理一张「跳跃表」：

$$
f[i][j] = \text{节点 } i \text{ 的第 } 2^j \text{ 个祖先}
$$

其中：
- $j = 0$ 时，$f[i][0]$ 就是节点 $i$ 的父节点
- $j = 1$ 时，$f[i][1]$ 是节点 $i$ 的 $2^1 = 2$ 个祖先
- $j = 2$ 时，$f[i][2]$ 是节点 $i$ 的 $2^2 = 4$ 个祖先
- 以此类推……

### 2.2 状态转移方程

利用倍增的递推关系：

$$
f[i][j] = f[f[i][j-1]][j-1]
$$

这表示「跳 $2^j$ 步」等价于「先跳 $2^{j-1}$ 步，再跳 $2^{j-1}$ 步」。

### 2.3 预处理代码

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MAXN = 1000005;
const int LOG = 20; // 根据节点数量，LOG = ceil(log2(N))

int n, lgn;
int depth[MAXN];
int f[MAXN][LOG];

void dfs(int u, int parent) {
    f[u][0] = parent;
    for (int j = 1; j <= lgn; j++) {
        if (f[u][j-1] != -1) {
            f[u][j] = f[f[u][j-1]][j-1];
        }
    }
    for (int v : g[u]) {
        if (v != parent) {
            depth[v] = depth[u] + 1;
            dfs(v, u);
        }
    }
}
```

## 3. 基本应用

### 3.1 求 K 步祖先

给定节点 $u$ 和整数 $k$，求 $u$ 的第 $k$ 个祖先。

**思路**：将 $k$ 分解为二进制表示，例如 $k = 13 = 1101_2 = 8 + 4 + 1$。从高位到低位遍历，若当前二进制位为 $1$，则从 $u$ 向上跳对应的 $2^j$ 步。

```cpp
int kth_ancestor(int u, int k) {
    for (int j = 0; j <= lgn; j++) {
        if (k & (1 << j)) {
            u = f[u][j];
            if (u == -1) break;
        }
    }
    return u;
}
```

**复杂度**：预处理 $O(N \log N)$，单次查询 $O(\log N)$。

---

### 3.2 最近公共祖先（LCA）

**LCA**（Lowest Common Ancestor）是树上两个节点深度最大的公共祖先。倍增法是求解 LCA 的经典算法之一。

**算法步骤**：

1. **对齐深度**：将较深的节点 $u$ 向上跳，使其与 $v$ 深度相同
2. **同时上跳**：从最高位 $j = \log_2(N)$ 枚举到 $0$，若 $u$ 和 $v$ 在 $2^j$ 步处不相等，则同时上跳
3. **返回父节点**：此时 $u$ 和 $v$ 的父节点即为 LCA

```cpp
int lca(int u, int v) {
    if (depth[u] < depth[v]) swap(u, v);
    
    // 1. 对齐深度
    int diff = depth[u] - depth[v];
    for (int j = 0; j <= lgn; j++) {
        if (diff & (1 << j)) {
            u = f[u][j];
        }
    }
    
    if (u == v) return u;
    
    // 2. 同时上跳
    for (int j = lgn; j >= 0; j--) {
        if (f[u][j] != f[v][j]) {
            u = f[u][j];
            v = f[v][j];
        }
    }
    
    // 3. 返回父节点
    return f[u][0];
}
```

---

### 3.3 RMQ（区间最值查询）

RMQ 用于快速查询区间 $[l, r]$ 内的最小值或最大值。结合 Sparse Table，倍增思想可扩展为 **ST 表** 算法。

**预处理**：$st[i][j]$ 表示区间 $[i, i + 2^j - 1]$ 的最值，递推公式为：

$$
st[i][j] = \min(st[i][j-1], st[i + 2^{j-1}][j-1])
$$

**查询**：对于区间 $[l, r]$，设 $len = r - l + 1$，取 $k = \lfloor \log_2 len \rfloor$，则：

$$
\text{rmq}(l, r) = \min(st[l][k], st[r - 2^k + 1][k])
$$

```cpp
int st[MAXN][LOG];
int lg2[MAXN];

void build_rmq() {
    for (int j = 1; j <= lgn; j++) {
        for (int i = 1; i + (1 << j) - 1 <= n; i++) {
            st[i][j] = min(st[i][j-1], st[i + (1 << (j-1))][j-1]);
        }
    }
    for (int i = 2; i <= n; i++) lg2[i] = lg2[i/2] + 1;
}

int rmq(int l, int r) {
    int k = lg2[r - l + 1];
    return min(st[l][k], st[r - (1 << k) + 1][k]);
}
```

## 4. 与线段树的对比

| 特性 | 倍增（ST 表） | 线段树 |
|------|---------------|--------|
| 预处理复杂度 | $O(N \log N)$ | $O(N)$ |
| 单次查询复杂度 | $O(1)$ | $O(\log N)$ |
| 空间复杂度 | $O(N \log N)$ | $O(N)$ |
| 适用范围 | 静态区间（不可修改） | 动态区间（支持更新） |
| 可合并操作 | 满足结合律的操作 | 任意操作 |

**选择建议**：
- 若数据**静态不变**，且查询次数极多，优先选择 ST 表（$O(1)$ 查询）
- 若数据**频繁修改**，选择线段树或树状数组

## 5. 扩展应用

倍增思想可以推广到更多场景：

### 5.1 树上路径最值
在求两点间路径的最大边权、最小边权时，可以在倍增表中同时维护最值信息。

### 5.2 字符串匹配（KMP）
KMP 的前缀函数本身就是一种「倍增」思想——利用已匹配的最长前缀来加速后续匹配。

### 5.3 图论中的跳表
在非负权图中，Dijkstra 的堆优化可用「倍增」思想进行「扩容」优化。

### 5.4 二分答案验证
在二分搜索中，若需 $O(\log N)$ 次验证，每次验证使用倍增可在 $O(\log N)$ 内完成跳转。

## 6. 总结

倍增是一种「以空间换时间」或「以预处理换查询」的经典技巧。其核心在于：

1. **预处理 $2^j$ 跳转表**，建立 $f[i][j]$ 关系
2. **利用二进制拆分**，将复杂跳转分解为 $O(\log N)$ 次简单跳跃
3. **结合问题特性**，将状态转移方程扩展到更多维度

熟练掌握倍增后，可在面试和竞赛中快速解决一系列「查询跳转让」的问题。

---

## 参考资料

[^1]: 本篇参考 [OI Wiki - 倍增](https://oi-wiki.org/topic/binary-lifting/)
