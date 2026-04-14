---
title: 树链剖分
date: 2026-04-05
description: 树链剖分（Heavy-Light Decomposition, HLD）一种将树拆分为多条链的树分解技术
tags:
  - hld
  - tree
  - decomposition
  - algorithm
draft: false
permalink:
---

## 定义

树链剖分（Heavy-Light Decomposition, HLD）是一种将树拆分为多条不相交的链（chain）的树分解技术。[^1]

### 核心性质

从树的任意节点出发，沿着轻边（light edge）向上走到根节点，最多经过 $O(\log n)$ 条重链（heavy path）。这一性质使得路径查询的时间复杂度从 $O(n)$ 降低到 $O(\log^2 n)$。

### 重边与轻边

设 $u$ 为节点 $v$ 的子节点，若：

$$
\text{size}(u) \geq \frac{\text{size}(v)}{2}
$$

则边 $(v, u)$ 称为**重边**（heavy edge），否则称为**轻边**（light edge）。

- **重边**：连接子节点中子树规模不小于父节点一半的边
- **轻边**：所有其他边

每个节点到其重子节点的边必为重边，而到其他子节点的边则为轻边。

## 核心算法

树链剖分分为两次深度优先搜索（DFS）：

### 第一次 DFS

计算每个节点的子树大小，并找出每个节点的重子节点（heavy child）。

```cpp
int dfs(int v, int p) {
    parent[v] = p;
    size[v] = 1;
    int maxSize = 0;
    for (int u : g[v]) {
        if (u != p) {
            depth[u] = depth[v] + 1;
            int sz = dfs(u, v);
            size[v] += sz;
            if (sz > maxSize) {
                maxSize = sz;
                heavy[v] = u;
            }
        }
    }
    return size[v];
}
```

### 第二次 DFS

对每条重链进行分解，分配链头（head of chain）和在线段树中的位置（pos）。

```cpp
int curPos = 0;
void decompose(int v, int h) {
    head[v] = h;
    pos[v] = ++curPos;  // 1-indexed for segment tree
    
    if (heavy[v] != 0) {
        // 优先遍历重子节点，保持重链连续
        decompose(heavy[v], h);
    }
    for (int u : g[v]) {
        if (u != parent[v] && u != heavy[v]) {
            // 轻边起点，形成新的链
            decompose(u, u);
        }
    }
}
```

## 代码实现

完整实现包含建图、两次 DFS、以及路径查询接口：

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MAXN = 1000005;

int n;
vector<int> g[MAXN];
int parent[MAXN], depth[MAXN], heavy[MAXN];
int head[MAXN], pos[MAXN], size[MAXN];
int curPos = 0;

// 第一次DFS：计算子树大小和重子节点
int dfs(int v, int p) {
    parent[v] = p;
    size[v] = 1;
    int maxSize = 0;
    for (int u : g[v]) {
        if (u != p) {
            depth[u] = depth[v] + 1;
            int sz = dfs(u, v);
            size[v] += sz;
            if (sz > maxSize) {
                maxSize = sz;
                heavy[v] = u;
            }
        }
    }
    return size[v];
}

// 第二次DFS：剖分重链
void decompose(int v, int h) {
    head[v] = h;
    pos[v] = ++curPos;
    
    if (heavy[v]) {
        decompose(heavy[v], h);
    }
    for (int u : g[v]) {
        if (u != parent[v] && u != heavy[v]) {
            decompose(u, u);
        }
    }
}

// 初始化
void init(int root = 1) {
    depth[root] = 0;
    dfs(root, 0);
    decompose(root, root);
}
```

### 路径查询（以最大值为例）

```cpp
int queryPath(int u, int v) {
    int res = 0;
    while (head[u] != head[v]) {
        if (depth[head[u]] < depth[head[v]]) swap(u, v);
        // head[u]更深，将该链段加入结果
        res = max(res, querySeg(pos[head[u]], pos[u]));
        u = parent[head[u]];
    }
    // 最后在同一链上
    if (depth[u] > depth[v]) swap(u, v);
    res = max(res, querySeg(pos[u], pos[v]));
    return res;
}
```

## 应用场景

树链剖分主要用于解决以下问题：

| 场景 | 操作 | 时间复杂度 |
|------|------|------------|
| 路径查询 | 最大值、最小值、和 | $O(\log^2 n)$ |
| 路径更新 | 单点更新、区间更新 | $O(\log^2 n)$ |
| 换根操作 | 任意节点为根的树 | $O(\log n)$ |

适用问题类型：

- 树上路径最值问题
- 树上路径求和问题
- 树上路径更新问题
- 需要高效处理多路径查询的场景

## 配合线段树

树链剖分将树形结构线性化为数组，每条重链对应数组中的一段连续区间。

### 线性化原理

- 每个节点被分配一个唯一的 `pos[v]`（在线段树中的位置）
- 同一条重链上的节点，`pos` 值连续
- 轻边连接的两端节点，不在同一段连续区间内

### 路径查询的分解过程

查询路径 $(u, v)$ 时，不断将 $u, v$ 向上移动至同一链：

1. 比较 `head[u]` 与 `head[v]` 的深度
2. 将较深链的 `[pos[head[u]], pos[u]]` 作为一段线段树区间处理
3. 将 $u$ 跳到 `parent[head[u]]`
4. 重复直到两端在同一链上

由于每次跳转都会跨越一条轻边，而任意路径上轻边数量不超过 $O(\log n)$，因此最多分解为 $O(\log n)$ 个线段树区间，总时间复杂度为 $O(\log^2 n)$。

如图所示：

$$
\text{路径查询} = \bigcup_{i=1}^{k} \text{链段}_i, \quad k \leq \log n
$$

## 例题

### 洛谷 P3384【模板】树链剖分

**题目描述**：给定一棵 $n$ 个节点的树，要求支持以下操作：

1. `1 u v x`：将 $u$ 到 $v$ 路径上所有节点的值加上 $x$
2. `2 u v`：查询 $u$ 到 $v$ 路径上所有节点的值之和
3. `3 u x`：将以 $u$ 为根的子树所有节点的值加上 $x$
4. `4 u`：查询以 $u$ 为根的子树所有节点的值之和

**解题思路**：使用树链剖分将树线性化，配合线段树（懒标记）维护区间和。路径操作分解为 $O(\log n)$ 个区间操作，子树操作对应线段树中的一段连续区间。

**参考实现**：

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MAXN = 100005;
int n, m, root, MOD;
vector<int> g[MAXN];
int parent[MAXN], depth[MAXN], heavy[MAXN];
int head[MAXN], pos[MAXN], size[MAXN];
int curPos = 0;
int a[MAXN], seg[MAXN * 4], lazy[MAXN * 4];

int dfs(int v, int p) {
    parent[v] = p;
    size[v] = 1;
    int maxSize = 0;
    for (int u : g[v]) {
        if (u != p) {
            depth[u] = depth[v] + 1;
            int sz = dfs(u, v);
            size[v] += sz;
            if (sz > maxSize) {
                maxSize = sz;
                heavy[v] = u;
            }
        }
    }
    return size[v];
}

void decompose(int v, int h) {
    head[v] = h;
    pos[v] = ++curPos;
    if (heavy[v]) decompose(heavy[v], h);
    for (int u : g[v]) {
        if (u != parent[v] && u != heavy[v]) {
            decompose(u, u);
        }
    }
}

// 线段树部分省略，可参考 [[segment-tree|线段树]] 模板
```

---

## 参考资料

[^1]: 树链剖分 - CP-Algorithms. https://cp-algorithms.com/graph/hld.html
