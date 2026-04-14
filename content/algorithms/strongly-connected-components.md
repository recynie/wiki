---
title: 强连通分量
date: 2026-04-03
description: 强连通分量（Strongly Connected Components，SCC）算法详解
tags:
  - graph
  - scc
  - tarjan
draft: false
permalink:
---

## 定义

在有向图 G 中，如果任意两个结点都互相可达，则称该图是**强连通**的。

**强连通分量**（Strongly Connected Components，简称 SCC）是极大的强连通子图。换句话说，如果在一个子图中任意两个结点都互相可达，且再加入任何其他结点都会破坏这个性质，那么这个子图就是一个强连通分量。

## Tarjan 算法

Tarjan 算法基于深度优先搜索，在 $O(n + m)$ 时间复杂度内求出所有强连通分量。

### 核心思想

在 DFS 遍历过程中，为每个结点维护两个值：

- `dfn[u]`：结点 u 被首次访问的时间戳（顺序编号）
- `low[u]`：从 u 出发能回溯到的最早（dfn 最小）已在栈中的结点

### 状态转移

对于结点 u 及其相邻结点 v：

1. **v 未被访问**：递归 DFS v，然后 `low[u] = min(low[u], low[v])`
2. **v 已在栈中**：`low[u] = min(low[u], dfn[v])`

### 关键性质

当 `dfn[u] == low[u]` 时，栈中 u 及其上方的结点构成一个强连通分量。

### 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MAXN = 1000005;

struct Edge {
    int to, nxt;
} e[MAXN];

int head[MAXN], cnt;
int dfn[MAXN], low[MAXN], dfncnt;
int st[MAXN], top;
int in_stack[MAXN];
int sccno[MAXN], scccnt;
int siz[MAXN];

void add_edge(int u, int v) {
    e[++cnt] = {v, head[u]};
    head[u] = cnt;
}

void tarjan(int u) {
    low[u] = dfn[u] = ++dfncnt;
    st[++top] = u;
    in_stack[u] = 1;
    
    for (int i = head[u]; i; i = e[i].nxt) {
        int v = e[i].to;
        if (!dfn[v]) {
            tarjan(v);
            low[u] = min(low[u], low[v]);
        } else if (in_stack[v]) {
            low[u] = min(low[u], dfn[v]);
        }
    }
    
    if (dfn[u] == low[u]) {
        ++scccnt;
        while (true) {
            int v = st[top--];
            in_stack[v] = 0;
            sccno[v] = scccnt;
            siz[scccnt]++;
            if (v == u) break;
        }
    }
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n, m;
    cin >> n >> m;
    for (int i = 1; i <= m; i++) {
        int u, v;
        cin >> u >> v;
        add_edge(u, v);
    }
    
    for (int i = 1; i <= n; i++) {
        if (!dfn[i]) tarjan(i);
    }
    
    cout << scccnt << endl;  // 强连通分量个数
    for (int i = 1; i <= n; i++) {
        cout << i << " -> scc" << sccno[i] << endl;
    }
    
    return 0;
}
```

### 分量标号与拓扑序

Tarjan 算法发现的强连通分量顺序是**逆拓扑序**。如果将每个 SCC 缩成一个点，得到的 DAG 的拓扑序与 SCC 标号顺序相反。

## Kosaraju 算法

Kosaraju 算法需要两次 DFS：

1. **第一次 DFS**：在原图上按任意顺序遍历，记录结点的**后序遍历次序**
2. **第二次 DFS**：在反图上，按第一次记录次序的**逆序**开始 DFS

第二次 DFS 能访问到的所有结点恰好构成一个强连通分量。

### 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

int n, m;
vector<vector<int>> g, g2;
vector<int> order, comp;
vector<int> vis;
int comp_cnt;

void dfs1(int u) {
    vis[u] = 1;
    for (int v : g[u]) {
        if (!vis[v]) dfs1(v);
    }
    order.push_back(u);
}

void dfs2(int u) {
    comp[u] = comp_cnt;
    for (int v : g2[u]) {
        if (!comp[v]) dfs2(v);
    }
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    cin >> n >> m;
    g.resize(n + 1);
    g2.resize(n + 1);
    comp.assign(n + 1, 0);
    vis.assign(n + 1, 0);
    
    for (int i = 0; i < m; i++) {
        int u, v;
        cin >> u >> v;
        g[u].push_back(v);
        g2[v].push_back(u);  // 反图
    }
    
    for (int i = 1; i <= n; i++) {
        if (!vis[i]) dfs1(i);
    }
    
    reverse(order.begin(), order.end());
    for (int u : order) {
        if (!comp[u]) {
            ++comp_cnt;
            dfs2(u);
        }
    }
    
    cout << comp_cnt << endl;
    return 0;
}
```

## 算法对比

| 特性 | Tarjan | Kosaraju |
|------|--------|----------|
| 时间复杂度 | $O(n + m)$ | $O(n + m)$ |
| 实现难度 | 中等 | 简单 |
| 遍历次数 | 1 次 | 2 次 |
| 空间复杂度 | $O(n)$ | $O(n + m)$ |

两种算法在实际应用中都很常见，Tarjan 算法只需要一次遍历，在竞赛中更为常用。

## 缩点应用

将每个 SCC 缩成一个点后，原图会变成一个有向无环图（DAG）。这一转换使得许多问题变得简单：

- **判断图的连通性**
- **求必经点问题**
- **拓扑排序**

## 典型例题

- [[../CCF-CSP/CSP202506B|机器人复健指南]]（CSP202506B）
- [[../algorithms/topological-sort|拓扑排序]]的延伸应用

## 参考资料

[^1]: [强连通分量 - OI Wiki](https://oi-wiki.org/graph/scc/)
