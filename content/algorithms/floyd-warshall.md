---
title: Floyd-Warshall算法
date: 2026-04-03
description: 全源最短路（Floyd-Warshall）算法
tags:
  - shortest-path
  - graph
  - algorithm
draft: false
permalink:
---

## 定义

**全源最短路径问题（All-Pairs Shortest Path）**：给定有向带权图，求任意两点对之间的最短路径长度。

Floyd-Warshall 算法（也称 Floyd 算法）是解决该问题的经典动态规划算法，时间复杂度为 $O(V^3)$，空间复杂度为 $O(V^2)$。

## 动态规划思路

设 $dp[k][i][j]$ 表示在使用顶点集合 $\{1, 2, \dots, k\}$ 作为中间顶点的情况下，从 $i$ 到 $j$ 的最短路径长度。

**转移方程**：

$$
dp[k][i][j] = \min(dp[k-1][i][j],\ dp[k-1][i][k] + dp[k-1][k][j])
$$

即：要么不经过 $k$，要么经过 $k$（此时路径被分为 $i \to k$ 和 $k \to j$ 两段）。

**边界条件**：$dp[0][i][j]$ 为直接边权值，若无边则为 $\infty$。

## 状态压缩至二维

观察到 $dp[k][i][j]$ 只与 $dp[k-1][i][j]$ 相关，因此可以压缩为一维数组原地更新：

$$
dp[i][j] = \min(dp[i][j],\ dp[i][k] + dp[k][j])
$$

**注意**：需要先记录 $dp[i][k]$ 和 $dp[k][j]$ 的旧值（因为 $dp[i][j]$ 的更新可能影响同一轮中 $dp[i][k]$ 或 $dp[k][j]$ 的后续使用），或者按 $k$ 外层循环时直接使用上一轮的结果。

更稳妥的做法是**三重循环顺序**为：先 $k$，再 $i$，最后 $j$。

## 代码模板

```cpp
#include <bits/stdc++.h>
using namespace std;
const long long INF = LLONG_MAX / 4;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n, m;
    cin >> n >> m;
    // 邻接矩阵初始化
    vector<vector<long long>> dist(n, vector<long long>(n, INF));
    vector<vector<int>> pre(n, vector<int>(n, -1));
    
    for (int i = 0; i < n; i++) dist[i][i] = 0;
    
    for (int i = 0; i < m; i++) {
        int u, v;
        long long w;
        cin >> u >> v >> w;
        u--; v--;
        if (w < dist[u][v]) {  // 处理重边
            dist[u][v] = w;
            pre[u][v] = u;
        }
    }
    
    // Floyd-Warshall
    for (int k = 0; k < n; k++) {
        for (int i = 0; i < n; i++) {
            if (dist[i][k] == INF) continue;
            for (int j = 0; j < n; j++) {
                if (dist[k][j] == INF) continue;
                if (dist[i][j] > dist[i][k] + dist[k][j]) {
                    dist[i][j] = dist[i][k] + dist[k][j];
                    pre[i][j] = pre[k][j];  // 记录前驱
                }
            }
        }
    }
    
    // 检测负环：若存在负环，则 dp[i][i] < 0
    for (int i = 0; i < n; i++) {
        if (dist[i][i] < 0) {
            cout << "Negative cycle detected!" << endl;
        }
    }
    
    return 0;
}
```

## 路径重建

使用**前驱矩阵** `pre[i][j]` 记录最短路径上 $j$ 的前一个顶点。初始化时，若存在边 $i \to j$，则 `pre[i][j] = i`。

更新时：`pre[i][j] = pre[k][j]`（当经过 $k$ 更优时）。

**重建函数**：

```cpp
vector<int> reconstructPath(int u, int v, const vector<vector<int>>& pre) {
    vector<int> path;
    if (pre[u][v] == -1) return path;  // 无路径
    path.push_back(v);
    while (v != u) {
        v = pre[u][v];
        path.push_back(v);
    }
    reverse(path.begin(), path.end());
    return path;
}
```

## 负环检测

在 Floyd-Warshall 算法完成后：

- 若图中有**负环**，则至少有一个顶点 $i$ 满足 $dist[i][i] < 0$。
- 这是因为负环上的任意顶点到自身的距离可以通过不断绕环而不断减小。

**时间戳法求负环**（更常用）：对每个顶点执行 Bellman-Ford，但 Floyd 的负环检测更简洁。

## 与 Dijkstra 算法对比

| 特性 | Floyd-Warshall | Dijkstra |
|------|---------------|----------|
| **时间复杂度** | $O(V^3)$ | $O(E \log V)$ 或 $O(V^2)$ |
| **适用场景** | 稠密图，负权边 | 稀疏图，无负权边 |
| **负权边** | 支持（无负环） | 不支持 |
| **单源/全源** | 全源 | 单源（可多次调用变通） |
| **空间复杂度** | $O(V^2)$ | $O(V)$ 或 $O(E)$ |

**选择策略**：

- 图**稠密**（$E \approx V^2$）且可能包含负权边时，用 Floyd。
- 图**稀疏**（$E \ll V^2$）且无负权边时，用 Dijkstra 更优。
- 若只需求单源最短路且无负权边，Dijkstra 通常更快。

## 应用场景

### 传递闭包（Transitive Closure）

将边权值视为布尔值（连通/不连通），或令每条边权值为 1，求传递闭包：

```cpp
// 传递闭包：可达性矩阵
vector<vector<bool>> reach(n, vector<bool>(n, false));
for (int i = 0; i < n; i++) reach[i][i] = true;

for (int k = 0; k < n; k++)
    for (int i = 0; i < n; i++)
        for (int j = 0; j < n; j++)
            reach[i][j] = reach[i][j] || (reach[i][k] && reach[k][j]);
```

### 最小环（Minimum Cycle）

在 Floyd 过程中，固定 $k$ 为环中编号最大的顶点，$i \to k \to j$ 与已存在的 $i \to j$ 路径构成一个环：

$$
\text{min\_cycle} = \min_{i,j} (dist[i][j] + w[j][i])
$$

其中 $dist[i][j]$ 是当前已求得的仅使用 $< k$ 顶点作为中间点的最短路。

```cpp
long long minCycle = INF;
for (int k = 0; k < n; k++) {
    for (int i = 0; i < n; i++) {
        if (dist[i][k] == INF || dist[k][i] == INF) continue;
        minCycle = min(minCycle, dist[i][k] + dist[k][i]);
    }
    // 再做标准 Floyd 更新
    for (int i = 0; i < n; i++)
        for (int j = 0; j < n; j++)
            if (dist[i][j] > dist[i][k] + dist[k][j])
                dist[i][j] = dist[i][k] + dist[k][j];
}
```

## 参考

[^1]: [Floyd-Warshall 算法 - OI Wiki](https://oi-wiki.org/graph/shortest-path/#floyd-warshall)
