---
title: 二分图最大匹配
date: 2026-04-03
description: 二分图最大匹配算法：匈牙利算法和 KM 算法
tags:
  - graph
  - bipartite-matching
  - hungarian
draft: false
permalink:
---

## 定义

### 二分图

二分图是一种特殊的图，其顶点集合可以划分为两个互不相交的子集 $U$ 和 $V$，使得图中的每条边都连接 $U$ 中的顶点和 $V$ 中的顶点。

### 匹配

**匹配**是指图中选取一组边，使得任意两条边都不共享顶点（即没有公共顶点）。

### 最大匹配

**最大匹配**是指包含边数最多的匹配。在二分图中，最大匹配的边数记为 $|M_{max}|$。

### 完美匹配

如果 $|M| = \min(|U|, |V|)$，即所有顶点都被匹配，则称为**完美匹配**。

## 匈牙利算法（Hungarian Algorithm）

匈牙利算法由 Harold Kuhn 于 1955 年提出，用于求解二分图的最大匹配。

### 核心思想

不断寻找增广路径来增加匹配数。**增广路径**是指连接两个未匹配顶点的路径，其中未匹配的边和匹配的边交替出现。

### 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MAXN = 1005;

int n, m, e;  // n: 左部点数, m: 右部点数, e: 边数
vector<int> g[MAXN];  // 邻接表
int matchR[MAXN];     // 右部点匹配的左部点
int d[MAXN];          // BFS 距离
bool visited[MAXN];

// 尝试从左部点 u 寻找增广路径
bool dfs(int u) {
    for (int v : g[u]) {
        if (!visited[v]) {
            visited[v] = true;
            if (matchR[v] == 0 || dfs(matchR[v])) {
                matchR[v] = u;
                return true;
            }
        }
    }
    return false;
}

// BFS 层次图构建（用于 Hopcroft-Karp）
bool bfs() {
    queue<int> q;
    for (int u = 1; u <= n; u++) {
        if (matchL[u] == 0) {  // 未匹配
            d[u] = 0;
            q.push(u);
        } else {
            d[u] = -1;  // INF
        }
    }
    bool found = false;
    while (!q.empty()) {
        int u = q.front();
        q.pop();
        for (int v : g[u]) {
            if (matchR[v] && d[matchR[v]] == -1) {
                d[matchR[v]] = d[u] + 1;
                q.push(matchR[v]);
            }
            if (!matchR[v]) {
                found = true;  // 找到未匹配的右部点
            }
        }
    }
    return found;
}

int matchL[MAXN];  // 左部点匹配的右部点（可选，用于记录）

// Hopcroft-Karp 算法 - O(E sqrt(V))
int hopcroft_karp() {
    int matching = 0;
    while (bfs()) {
        for (int u = 1; u <= n; u++) {
            if (matchL[u] == 0) {
                fill(visited, visited + m + 1, false);
                if (dfs(u)) {
                    matching++;
                }
            }
        }
    }
    return matching;
}

// 朴素匈牙利算法 - O(VE)
int hungarian() {
    int ans = 0;
    fill(matchR, matchR + m + 1, 0);
    for (int u = 1; u <= n; u++) {
        fill(visited, visited + m + 1, false);
        if (dfs(u)) ans++;
    }
    return ans;
}
```

### 朴素版本详解

```cpp
// 朴素匈牙利算法（邻接矩阵版本）
int n, m;  // n 左部顶点数，m 右部顶点数
int a[MAXN][MAXN];  // 邻接矩阵，a[u][v] = 1 表示有边
int match[MAXN];    // 右部点 v 匹配的左部点
bool vis[MAXN];

bool dfs(int u) {
    for (int v = 1; v <= m; v++) {
        if (a[u][v] && !vis[v]) {
            vis[v] = true;
            if (match[v] == 0 || dfs(match[v])) {
                match[v] = u;
                return true;
            }
        }
    }
    return false;
}

int hungarian() {
    int ans = 0;
    fill(match, match + m + 1, 0);
    for (int u = 1; u <= n; u++) {
        fill(vis, vis + m + 1, false);
        if (dfs(u)) ans++;
    }
    return ans;
}
```

## KM 算法（ Kuhn-Munkres Algorithm）

KM 算法用于求解**二分图最大权匹配**，即边权之和最大的匹配。

### 顶标思想

为左部点集合 $U$ 中的每个顶点 $u$ 维护顶标 $lx[u]$，为右部点集合 $V$ 中的每个顶点 $v$ 维护顶标 $ly[v]$，满足：

$$
lx[u] + ly[v] \geq w(u, v) \quad \text{（对于所有边）}
$$

且相等子图（$lx[u] + ly[v] = w(u, v)$ 的边构成的子图）中存在完美匹配。

### 代码实现

```cpp
const int INF = 1e9;
int n;  // 左部和右部顶点数相同
vector<vector<int>> w;  // 边权矩阵 (1-indexed)
vector<int> lx, ly, match, slack;
vector<int> px, py;  // 增广路记录
vector<bool> visx, visy;

bool dfs(int u) {
    visx[u] = true;
    for (int v = 1; v <= n; v++) {
        if (visy[v]) continue;
        int t = lx[u] + ly[v] - w[u][v];
        if (t == 0) {
            visy[v] = true;
            if (match[v] == 0 || dfs(match[v])) {
                match[v] = u;
                return true;
            }
        } else {
            slack[v] = min(slack[v], t);
        }
    }
    return false;
}

int km() {
    lx.assign(n + 1, 0);
    ly.assign(n + 1, 0);
    match.assign(n + 1, 0);
    slack.assign(n + 1, INF);
    
    // 初始化左部顶标为最大边权
    for (int u = 1; u <= n; u++) {
        lx[u] = *max_element(w[u].begin() + 1, w[u].end());
    }
    
    for (int u = 1; u <= n; u++) {
        fill(slack.begin(), slack.end(), INF);
        while (true) {
            visx.assign(n + 1, false);
            visy.assign(n + 1, false);
            if (dfs(u)) break;
            
            // 未能增广，需要修改顶标
            int d = INF;
            for (int v = 1; v <= n; v++) {
                if (!visy[v]) d = min(d, slack[v]);
            }
            
            for (int i = 1; i <= n; i++) {
                if (visx[i]) lx[i] -= d;
                if (visy[i]) ly[i] += d;
                else slack[i] -= d;
            }
        }
    }
    
    int ans = 0;
    for (int v = 1; v <= n; v++) {
        if (match[v]) ans += w[match[v]][v];
    }
    return ans;
}
```

## 算法对比

| 算法 | 时间复杂度 | 适用场景 |
|------|-----------|----------|
| 朴素匈牙利 | $O(VE)$ | 小规模数据，最大匹配 |
| Hopcroft-Karp | $O(E\sqrt{V})$ | 大规模最大匹配 |
| KM 算法 | $O(n^3)$ | 最大权匹配 |

## 应用场景

- **任务分配**：将 n 个任务分配给 n 个工人，最小化总成本
- **运动员配对**：双人比赛选手配对
- **婚礼座次**：男女宾客配对
- **课程选择**：学生与课程的匹配

## 相关主题

- [[../algorithms/strongly-connected-components|强连通分量]]：在 2-SAT 问题中的应用
- [[../algorithms/dijkstra|最短路]]：在费用流中的应用
- [[../algorithms/minimum-spanning-tree|最小生成树]]：图的连通性

## 参考资料

[^1]: [二分图最大匹配 - OI Wiki](https://oi-wiki.org/graph/graph-matching/bigraph-match/)
[^2]: [二分图最大权匹配 - OI Wiki](https://oi-wiki.org/graph/graph-matching/bigraph-weight-match/)
