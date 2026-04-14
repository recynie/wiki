---
title: Bellman-Ford算法
date: 2026-04-06
description: Bellman-Ford算法用于计算单源最短路径，可以处理负权边
tags:
  - shortest-path
  - graph
  - bellman-ford
draft: false
permalink:
---

# Bellman-Ford算法

Bellman-Ford算法用于计算单源最短路径，其核心思想是对所有边进行 $N-1$ 次松弛操作。该算法可以处理**负权边**，并能检测负环。[^1]

[^1]: 本段参考[Bellman-Ford算法 - OI Wiki](https://oi-wiki.org/graph/shortest-path/)

## 算法思想

给定带权有向图 $G = (V, E)$ 和源点 $s$，Bellman-Ford算法通过以下步骤求解最短路：

1. 初始化：$dist[s] = 0$，其余 $dist[v] = \infty$
2. 对每条边 $(u, v, w)$ 进行 $N-1$ 次松弛操作
3. 检查是否存在负环

### 松弛操作

对于每条边 $(u, v, w)$，如果 $dist[u] + w < dist[v]$，则更新 $dist[v]$：

$$
dist[v] = min(dist[v], dist[u] + w)
$$

## 代码实现

```cpp
struct Edge {
    int u, v, w;
};

const int INF = 1e9;

vector<int> bellmanFord(int n, int s, const vector<Edge>& edges) {
    vector<int> dist(n, INF);
    dist[s] = 0;
    
    // N-1轮松弛操作
    for (int i = 0; i < n - 1; ++i) {
        bool updated = false;
        for (const auto& e : edges) {
            if (dist[e.u] != INF && dist[e.u] + e.w < dist[e.v]) {
                dist[e.v] = dist[e.u] + e.w;
                updated = true;
            }
        }
        if (!updated) break;  // 已经收敛，提前结束
    }
    
    return dist;
}
```

### 检测负环

如果进行第 $N$ 轮松弛操作时仍有更新，说明存在负环：

```cpp
bool hasNegativeCycle(int n, const vector<Edge>& edges, const vector<int>& dist) {
    for (const auto& e : edges) {
        if (dist[e.u] != INF && dist[e.u] + e.w < dist[e.v]) {
            return true;  // 存在负环
        }
    }
    return false;
}
```

## SPFA算法

SPFA（Shortest Path Faster Algorithm）是Bellman-Ford的队列优化版本：

```cpp
vector<int> spfa(int n, int s, const vector<vector<pair<int,int>>>& g) {
    vector<int> dist(n, INF);
    vector<int> inqueue(n, 0);
    queue<int> q;
    
    dist[s] = 0;
    q.push(s);
    inqueue[s] = 1;
    
    while (!q.empty()) {
        int u = q.front();
        q.pop();
        inqueue[u] = 0;
        
        for (const auto& [v, w] : g[u]) {
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                if (!inqueue[v]) {
                    q.push(v);
                    inqueue[v] = 1;
                }
            }
        }
    }
    
    return dist;
}
```

## 与Dijkstra算法的比较

| 特征 | Bellman-Ford | Dijkstra |
|------|-------------|----------|
| 时间复杂度 | $O(NE)$ | $O(N \log N + M)$（堆优化）|
| 负权边 | 支持 | 不支持 |
| 负环检测 | 支持 | 不支持 |
| 适用场景 | 负权边、负环检测 | 非负权图 |

## 复杂度分析

- **时间复杂度**：$O(N \cdot E)$
- **空间复杂度**：$O(N)$

## 应用场景

1. **含有负权的最短路问题**
2. **负环检测**：判断图中是否存在负环
3. **差分约束系统**：转化为最短路问题

## 参考

- [Bellman-Ford算法 - OI Wiki](https://oi-wiki.org/graph/shortest-path/)
- [SPFA - OI Wiki](https://oi-wiki.org/graph/spfa/)
