---
title: Dijkstra算法
date: 2026-03-31
description: Dijkstra单源最短路算法
tags:
  - graph
  - shortest-path
  - dijkstra
draft: false
permalink:
---

## 定义

Dijkstra算法由荷兰计算机科学家艾兹格·迪杰斯特拉（Edsger W. Dijkstra）于1959年提出，用于计算**单源最短路径**——即从一个固定源点到图中所有其他顶点的最短路径长度。

该算法适用于**边权非负**的有向图或无向图。[^1]

## 核心特性

### 时间复杂度

使用不同的数据结构，实现的时间复杂度也不同：

| 实现方式 | 时间复杂度 |
|----------|------------|
| 朴素实现（数组） | $O(V^2)$ |
| 二叉堆（优先队列） | $O((V + E) \log V)$ |
| 斐波那契堆 | $O(E + V \log V)$ |

其中 $V$ 是顶点数，$E$ 是边数。

### 适用条件

- **边权非负**：这是 Dijkstra 算法正确性的前提条件
- **单源最短路**：从一个源点出发到其他所有点的最短路

如果存在负权边，需要使用 [[../algorithms/bellman-ford|Bellman-Ford]] 算法。

## 算法思想

Dijkstra 算法采用**贪心策略**。它的核心思想是：

1. 维护一个集合 $S$，表示已经找到最短路径的顶点
2. 每次从**不在** $S$ 中的顶点中，选择距离源点最近的顶点 $u$，加入 $S$
3. 使用顶点 $u$ **松弛**其所有邻接边，更新最短距离
4. 重复步骤 2-3，直到所有顶点都在 $S$ 中

为什么贪心有效？因为当所有边权非负时，一旦某个顶点的最短距离被确定，这个值就不会再改变。[^2]

## 代码实现

### 朴素实现 $O(V^2)$

```cpp
#include <bits/stdc++.h>
using namespace std;

const int INF = 1e9;

vector<int> dijkstraNaive(int n, int src, vector<vector<pair<int, int>>>& adj) {
    vector<int> dist(n, INF);
    vector<bool> visited(n, false);
    
    dist[src] = 0;
    
    for (int i = 0; i < n; i++) {
        // 找到未访问的最小距离顶点
        int u = -1;
        for (int j = 0; j < n; j++) {
            if (!visited[j] && (u == -1 || dist[j] < dist[u])) {
                u = j;
            }
        }
        
        if (dist[u] == INF) break;  // 无法到达
        visited[u] = true;
        
        // 松弛邻接边
        for (auto [v, w] : adj[u]) {
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
            }
        }
    }
    
    return dist;
}
```

### 优先队列优化 $O((V+E) \log V)$

```cpp
#include <bits/stdc++.h>
using namespace std;

const int INF = 1e9;

vector<int> dijkstra(int n, int src, vector<vector<pair<int, int>>>& adj) {
    vector<int> dist(n, INF);
    vector<int> parent(n, -1);  // 用于记录最短路径
    priority_queue<pair<int, int>, vector<pair<int, int>>, greater<pair<int, int>>> pq;
    
    dist[src] = 0;
    pq.push({0, src});
    
    while (!pq.empty()) {
        auto [d, u] = pq.top();
        pq.pop();
        
        if (d > dist[u]) continue;  // 跳过过期条目
        
        for (auto [v, w] : adj[u]) {
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                parent[v] = u;
                pq.push({dist[v], v});
            }
        }
    }
    
    return dist;
}

// 重建最短路径
vector<int> getPath(int src, int dst, const vector<int>& parent) {
    vector<int> path;
    for (int v = dst; v != -1; v = parent[v]) {
        path.push_back(v);
    }
    reverse(path.begin(), path.end());
    if (path.empty() || path[0] != src) return {};  // 无路径
    return path;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n = 5;  // 顶点数
    vector<vector<pair<int, int>>> adj(n);
    
    // 添加边：adj[u].push_back({v, w})
    adj[0].push_back({1, 10});
    adj[0].push_back({2, 5});
    adj[1].push_back({2, 2});
    adj[1].push_back({3, 1});
    adj[2].push_back({1, 3});
    adj[2].push_back({3, 9});
    adj[2].push_back({4, 2});
    adj[3].push_back({4, 4});
    adj[4].push_back({3, 6});
    
    vector<int> dist = dijkstra(n, 0, adj);
    
    cout << "从顶点 0 出发的最短距离:" << endl;
    for (int i = 0; i < n; i++) {
        cout << "到 " << i << ": " << dist[i] << endl;
    }
    
    return 0;
}
```

### 处理大规模图

```cpp
#include <bits/stdc++.h>
using namespace std;

const long long INF = 4e18;

struct Edge {
    int to;
    long long w;
};

vector<long long> dijkstraLarge(int n, int src, const vector<vector<Edge>>& adj) {
    vector<long long> dist(n, INF);
    priority_queue<pair<long long, int>, vector<pair<long long, int>>, greater<pair<long long, int>>> pq;
    
    dist[src] = 0;
    pq.push({0, src});
    
    while (!pq.empty()) {
        auto [d, u] = pq.top();
        pq.pop();
        
        if (d != dist[u]) continue;
        
        for (const auto& e : adj[u]) {
            if (dist[u] + e.w < dist[e.to]) {
                dist[e.to] = dist[u] + e.w;
                pq.push({dist[e.to], e.to});
            }
        }
    }
    
    return dist;
}
```

## 算法正确性证明简述

Dijkstra 算法的正确性基于以下引理：

**引理**：当顶点 $u$ 被加入集合 $S$ 时，$dist[u]$ 是源点到 $u$ 的最短路径长度。

**证明**：假设存在一条更短的路径 $src \rightarrow ... \rightarrow x \rightarrow u$，其中 $x$ 不在 $S$ 中。由于所有边权非负，在 $src$ 到 $u$ 的最短路径上，$x$ 的距离一定小于 $u$ 的距离。但这与 $u$ 是 $S$ 外最小距离顶点的选择矛盾。因此 $dist[u]$ 是最短的。

## 应用场景

### 最短路径导航

地图导航、路线规划等。

### 网络路由

OSPF 等协议使用 Dijkstra 算法计算最优路由。

### 其他优化问题

- **最小花费最大流**：每次增广使用最短路
- **Top-K 最短路**：修改算法记录多条最短路径

## 参考资料

[^1]: Dijkstra 算法是图论中最经典的算法之一，最初发表在《Numerische Mathematik》期刊上。

[^2]: 如果存在负权边，贪心策略可能失败——某个顶点的最短距离可能在后续发现更短的路径。因此负权边必须使用 Bellman-Ford 算法。
