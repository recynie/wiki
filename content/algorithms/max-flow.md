---
title: 最大流
date: 2026-04-03
description: 最大流算法（Maximum Flow）
tags:
  - network-flow
  - graph
  - algorithm
draft: false
permalink:
---

## 定义（Definition）

**流网络**（Flow Network）是一个有向图 $G = (V, E)$，其中：

- 每条边 $(u, v) \in E$ 有一条**容量**（Capacity）$c(u, v) \geq 0$
- 图中有一个特殊的源点（Source）$s$
- 图中有一个特殊的汇点（Sink）$t$

**流**（Flow）是一个函数 $f: V \times V \rightarrow \mathbb{R}$，满足：

1. **容量约束**：$0 \leq f(u, v) \leq c(u, v)$，对所有边
2. **流量守恒**：$\sum_{v \in V} f(v, u) = \sum_{v \in V} f(u, v)$，对所有中间顶点 $u$（即 $u \neq s$ 且 $u \neq t$）

**最大流问题**：找到从 $s$ 到 $t$ 的最大可行流量值 $|f| = \sum_{v \in V} f(s, v) = \sum_{v \in V} f(v, t)$。

## 残量网络（Residual Network）

给定一个流网络 $G$ 和流 $f$，**残量网络** $G_f$ 定义为：

- 对于每条原始边 $(u, v)$，定义**残量容量**：
  - 正向边残量：$c_f(u, v) = c(u, v) - f(u, v)$
  - 反向边残量：$c_f(v, u) = f(u, v)$

残量网络中的每条边表示还能发送多少流量。**注意**：即使原始图中没有边 $(v, u)$，残量网络中也会创建一条反向边，这是 Ford-Fulkerson 方法的关键。

## 增广路（Augmenting Path）

**增广路**是残量网络中从源点 $s$ 到汇点 $t$ 的一条简单路径 $P$。

路径上所有边的最小残量容量称为**瓶颈容量**（Bottleneck）：

$$
b = \min_{(u,v) \in P} c_f(u, v)
$$

沿增广路增加 $b$ 单位流量后，流的值增加 $b$。

**最大流的充要条件**：当残量网络中不存在任何从 $s$ 到 $t$ 的增广路时，当前的流就是最大流（**最大流最小割定理**）。

## Edmond-Karp 算法

Edmond-Karp 算法是 Ford-Fulkerson 方法的 BFS 实现，保证 $O(VE^2)$ 时间复杂度。

**核心思想**：每次用 BFS 在残量网络中找到一条最短增广路（按边数计）。

### C++ 实现

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Edge {
    int to, rev;      // 目标顶点，反向边索引
    long long cap;    // 容量
};

class EdmondKarp {
    int n, s, t;
    vector<vector<Edge>> g;
    vector<int> parent;        // 记录路径上的父边
    vector<int> parent_idx;    // 记录父边在邻接表中的索引
    
public:
    EdmondKarp(int n, int s, int t) : n(n), s(s), t(t) {
        g.assign(n, {});
        parent.resize(n);
        parent_idx.resize(n);
    }
    
    void addEdge(int u, int v, long long cap) {
        Edge a{v, (int)g[v].size(), cap};
        Edge b{u, (int)g[u].size(), 0};  // 反向边初始容量为0
        g[u].push_back(a);
        g[v].push_back(b);
    }
    
    long long bfs() {
        fill(parent.begin(), parent.end(), -1);
        queue<int> q;
        q.push(s);
        parent[s] = s;
        
        while (!q.empty() && parent[t] == -1) {
            int u = q.front(); q.pop();
            for (int i = 0; i < (int)g[u].size(); i++) {
                Edge &e = g[u][i];
                if (parent[e.to] == -1 && e.cap > 0) {
                    parent[e.to] = u;
                    parent_idx[e.to] = i;
                    q.push(e.to);
                }
            }
        }
        
        if (parent[t] == -1) return 0;  // 无法增广
        
        // 计算瓶颈容量
        long long flow = LLONG_MAX;
        int v = t;
        while (v != s) {
            int u = parent[v];
            Edge &e = g[u][parent_idx[v]];
            flow = min(flow, e.cap);
            v = u;
        }
        return flow;
    }
    
    void augment(long long flow) {
        int v = t;
        while (v != s) {
            int u = parent[v];
            Edge &e = g[u][parent_idx[v]];
            e.cap -= flow;           // 正向边减少
            g[v][e.rev].cap += flow; // 反向边增加
            v = u;
        }
    }
    
    long long maxFlow() {
        long long flow = 0, augment_flow;
        while ((augment_flow = bfs()) > 0) {
            augment(augment_flow);
            flow += augment_flow;
        }
        return flow;
    }
};
```

### 时间复杂度分析

- 每次 BFS 找增广路：$O(E)$
- 每次增广后最短路径长度至少增加 1（边数）
- 从 $s$ 到 $t$ 最短路径最长为 $V-1$ 条边
- 最多 $VE$ 次增广（每条边可能成为瓶颈 $V$ 次）
- 总复杂度：$O(VE^2)$

## Dinic 算法

Dinic 算法在 Edmond-Karp 基础上增加了**分层图**和**阻塞流**概念，达到 $O(V^2E)$ 时间复杂度。

**核心思想**：
1. BFS 构建分层图（level graph）
2. DFS 在分层图上寻找阻塞流
3. 重复直到没有增广路

### 关键优化

1. **分层图**：只保留从 $s$ 出发、沿着残量边能到达且容量 > 0 的边
2. **当前弧优化**：记录每条边当前遍历到的位置，避免重复遍历
3. **多路增广**：一条 DFS 可以同时找到多条增广路

### C++ 实现

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Edge {
    int to, rev;
    long long cap;
};

class Dinic {
    int n, s, t;
    vector<vector<Edge>> g;
    vector<int> level;      // 分层图中的层级
    vector<int> it;        // 当前弧优化：每条边的当前遍历位置
    
public:
    Dinic(int n, int s, int t) : n(n), s(s), t(t) {
        g.assign(n, {});
        level.resize(n);
        it.resize(n);
    }
    
    void addEdge(int u, int v, long long cap) {
        Edge a{v, (int)g[v].size(), cap};
        Edge b{u, (int)g[u].size(), 0};
        g[u].push_back(a);
        g[v].push_back(b);
    }
    
    // BFS 构建分层图
    bool bfs() {
        fill(level.begin(), level.end(), -1);
        queue<int> q;
        level[s] = 0;
        q.push(s);
        
        while (!q.empty()) {
            int u = q.front(); q.pop();
            for (const Edge &e : g[u]) {
                if (e.cap > 0 && level[e.to] < 0) {
                    level[e.to] = level[u] + 1;
                    q.push(e.to);
                }
            }
        }
        return level[t] >= 0;  // 能到达汇点
    }
    
    // DFS 寻找阻塞流
    long long dfs(int u, long long f) {
        if (u == t) return f;
        
        for (int &i = it[u]; i < (int)g[u].size(); i++) {
            Edge &e = g[u][i];
            if (e.cap > 0 && level[u] < level[e.to]) {
                long long d = dfs(e.to, min(f, e.cap));
                if (d > 0) {
                    e.cap -= d;
                    g[e.to][e.rev].cap += d;
                    return d;
                }
            }
        }
        return 0;
    }
    
    long long maxFlow() {
        long long flow = 0, INF = LLONG_MAX;
        
        while (bfs()) {
            fill(it.begin(), it.end(), 0);
            // 反复 DFS 找阻塞流
            while (long long f = dfs(s, INF)) {
                flow += f;
            }
        }
        return flow;
    }
};
```

### 时间复杂度分析

- 每次 BFS：$O(E)$
- 每次 DFS 沿一条路径增广后，至少有一条边满载被删除
- 每层图的构建最多产生 $O(E)$ 次增广
- 最多 $O(V)$ 层（BFS 每次至少增加一层）
- 总复杂度：$O(V^2E)$，在实际中通常远优于理论界

### 当前弧优化详解

`it[u]` 记录从 $u$ 出发的边已经遍历到哪个位置。当一条边 `g[u][i]` 被榨干后（`cap == 0`），直接跳过而不从头开始遍历，避免了 $O(E^2)$ 的最坏情况。

## 应用（Applications）

### 二分图最大匹配

最大流可以解决**二分图最大匹配**问题。

**规约方法**：
1. 创建源点 $s$ 和汇点 $t$
2. 左半部分顶点连接源点，容量 1
3. 右半部分顶点连接汇点，容量 1
4. 原图的边作为二分图边，容量 1
5. 求最大流即为最大匹配数

```cpp
// 二分图最大匹配（霍普克罗夫特-卡普也可）
int bipartiteMatch(int nLeft, int nRight, vector<pair<int,int>>& edges) {
    int n = nLeft + nRight + 2;
    int s = n - 2, t = n - 1;
    Dinic dinic(n, s, t);
    
    for (int i = 0; i < nLeft; i++)
        dinic.addEdge(s, i, 1);
    for (int i = 0; i < nRight; i++)
        dinic.addEdge(nLeft + i, t, 1);
    for (auto [u, v] : edges)
        dinic.addEdge(u, nLeft + v, 1);
    
    return dinic.maxFlow();
}
```

### 最大割与最小割的关系

**最大流最小割定理**：最大流的值为最小割的容量。

**割**（Cut）：将顶点集 $V$ 划分为两部分 $S$ 和 $T=V\setminus S$，满足 $s \in S$ 且 $t \in T$。

**割的容量**：所有从 $S$ 到 $T$ 的边的容量之和。

**应用**：在网络设计中，最小割可以找到最脆弱的瓶颈；在图像分割中，可用于前景背景分离。

## 算法选择

| 算法 | 时间复杂度 | 适用场景 |
|------|-----------|---------|
| Edmond-Karp | $O(VE^2)$ | 简单实现，小规模图 |
| Dinic | $O(V^2E)$ | 一般场景，推荐使用 |
| HLPP（最高标签预流推进） | $O(V^3)$ | 大规模密集图 |

Dinic 是竞赛和工程中最常用的选择，其当前弧优化和多路增广使其在实践中非常高效。[^1]

[^1]: 本篇内容参考自 [网络流 - OI Wiki](https://oi-wiki.org/graph/flow)
