---
title: 差分约束（Difference Constraints）
date: 2026-04-06
description: 差分约束系统：将不等式组 x_j - x_i ≤ b_k 转化为最短路问题，利用SPFA或Bellman-Ford求解
tags:
  - difference-constraints
  - graph
  - shortest-path
  - inequality
draft: false
permalink:
---

## 概述

**差分约束（Difference Constraints）** 是一类特殊的线性不等式系统，形式为：

$$
x_j - x_i \leq b_k
$$

其中 $i, j$ 为变量下标，$b_k$ 为常数。[^1]

差分约束广泛应用于**调度问题、区间分配、资源分配**等场景。

## 转化为最短路问题

对于不等式 $x_j - x_i \leq b$，可以构造一条有向边 $i \rightarrow j$，权值为 $b$：

$$
x_j \leq x_i + b \Leftrightarrow x_j \leq x_i + w(i, j)
$$

这与最短路中的松弛操作 $d[j] \leq d[i] + w$ 形式完全一致。

### 构造方法

给定 $n$ 个变量 $x_1, x_2, \ldots, x_n$ 和 $m$ 个约束 $x_{v_i} - x_{u_i} \leq w_i$：

1. 建立源点 $0$，向所有点 $i$ 建立边 $0 \rightarrow i$，权值为 $0$
2. 对于每个约束 $x_j - x_i \leq b$，建立边 $i \rightarrow j$，权值为 $b$
3. 从源点运行 Bellman-Ford 或 SPFA

若存在负环，则系统无解。

## 求解算法

### Bellman-Ford

```cpp
struct Edge {
    int u, v, w;
};

const int INF = 1e9;

bool bellman_ford(int n, int s, const vector<Edge>& edges, vector<long long>& dist) {
    dist.assign(n + 1, INF);
    dist[s] = 0;
    
    // n 个点，松弛 n-1 次
    for (int i = 1; i <= n; i++) {
        bool updated = false;
        for (const auto& e : edges) {
            if (dist[e.u] != INF && dist[e.u] + e.w < dist[e.v]) {
                dist[e.v] = dist[e.u] + e.w;
                updated = true;
            }
        }
        if (!updated) break;
    }
    
    // 第 n 次松弛检查负环
    for (const auto& e : edges) {
        if (dist[e.u] != INF && dist[e.u] + e.w < dist[e.v]) {
            return false;  // 存在负环，无解
        }
    }
    return true;
}
```

### SPFA（队列优化）

```cpp
bool spfa(int n, int s, const vector<vector<pair<int,int>>>& g, vector<long long>& dist) {
    vector<int> cnt(n + 1, 0);  // 入队次数
    vector<bool> inq(n + 1, false);
    queue<int> q;
    
    dist.assign(n + 1, INF);
    dist[s] = 0;
    q.push(s);
    inq[s] = true;
    
    while (!q.empty()) {
        int u = q.front(); q.pop();
        inq[u] = false;
        
        for (auto [v, w] : g[u]) {
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                if (!inq[v]) {
                    q.push(v);
                    inq[v] = true;
                    if (++cnt[v] > n) return false;  // 负环
                }
            }
        }
    }
    return true;
}
```

## 典型应用

### 1. 区间约束

**问题**：给定 $n$ 个区间 $[l_i, r_i]$，每个区间至少包含 $c_i$ 个点，求满足条件的最小解。

**建模**：
- 令 $P(x)$ 为前 $x$ 个位置中放置的点数
- 约束：$P(r_i) - P(l_i-1) \geq c_i$
- 转化为 $P(r_i) \geq P(l_i-1) + c_i$

### 2.  Scheduling（工程调度）

**问题**：有 $n$ 个任务，任务 $j$ 必须在任务 $i$ 完成后才能开始，且需要 $t_i$ 时间。求最早完成时间。

**建模**：$start_j \geq start_i + t_i$

### 3. 差分数组验证

**问题**：给定数组 $a$，判断是否存在 $x$ 使得所有区间和约束满足。

**转化**：前缀和 $S_i = \sum_{k=1}^{i} x_k$，约束变为 $S_j - S_i \leq b$。

## 例题：某年到某年

**题目**（CSP真题简化）：有 $n$ 个学生，第 $i$ 个学生需要在 $[L_i, R_i]$ 时间内至少参加一个活动。安排活动使得总数最少。

**思路**：差分约束 + 二分答案

**建模**：
- 设 $P(t)$ 为截至时间 $t$ 已参加活动的学生数
- 约束：$P(R_i) - P(L_i-1) \geq 1$
- 即 $P(R_i) \geq P(L_i-1) + 1$

## 解的多样性

差分约束的解不唯一。若 $(x_1, x_2, \ldots, x_n)$ 是解，则 $(x_1+c, x_2+c, \ldots, x_n+c)$ 也是解（$c$ 为任意常数）。

通常取最小（或最大）的解，通过改变源点的边权实现。

## 与其他问题的关系

| 问题 | 约束形式 | 解法 |
|------|---------|------|
| 差分约束 | $x_j - x_i \leq b$ | SPFA/Bellman-Ford |
| 2-SAT | $x_i \rightarrow x_j$ | SCC缩点 |
| 0-1约束 | $x_j - x_i \in \{0, 1\}$ | 差分约束特殊形式 |

## 参考

[^1]: 本内容参考 [OI-Wiki 差分约束](https://oi-wiki.org/graph/diff-constraints/)，内容经过验证和扩展。

---

*Last updated: 2026-04-06*