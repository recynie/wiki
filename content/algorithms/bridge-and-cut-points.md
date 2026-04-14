---
title: 桥与割点
date: 2026-04-03
description: 图的桥（Bridge）和割点（Articulation Point）算法
tags:
  - graph
  - bridge
  - articulation-point
draft: false
permalink:
---

## 定义

### 桥（Bridge）

在无向图中，如果删除一条边 e 后，图变得不连通（即连通分量数增加），则称 e 为**桥**（或割边）。

形式化地说：边 $(u, v)$ 是桥，当且仅当不存在另一条从 u 到 v 的路径。

### 割点（Articulation Point）

在无向图中，如果删除一个顶点 v（及其关联的所有边）后，图变得不连通，则称 v 为**割点**（或关节点）。

## Tarjan 算法

与强连通分量类似，桥和割点的求解也基于 DFS 时间戳和 low 值。

### 核心概念

- `dfn[u]`：结点 u 被访问的时间戳
- `low[u]`：从 u 出发（经由树边和回边）能访问到的最早结点的时间戳

### 桥的判定

对于无向图的边 $(u, v)$（v 是 u 在 DFS 树中的子结点）：

- 如果 `low[v] > dfn[u]`，则 $(u, v)$ 是**桥**
  - 含义：从 v 出发无法绕过 u 到达更早的祖先

### 割点的判定

- **根结点**：如果根结点有超过一个子子树，则根是割点
- **非根结点**：如果存在子结点 v 使得 `low[v] >= dfn[u]`，则 u 是割点

## 代码实现

### 求桥

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MAXN = 1000005;

struct Edge {
    int to, nxt;
    int id;  // 边的编号
} e[MAXN * 2];

int head[MAXN], cnt;
int n, m;
int dfn[MAXN], low[MAXN], dfncnt;
vector<pair<int, int>> bridges;

void add_edge(int u, int v, int id) {
    e[++cnt] = {v, head[u], id};
    head[u] = cnt;
}

void tarjan(int u, int parent_edge) {
    dfn[u] = low[u] = ++dfncnt;
    
    for (int i = head[u]; i; i = e[i].nxt) {
        int v = e[i].to;
        if (e[i].id == parent_edge) continue;  // 跳过父边
        
        if (!dfn[v]) {
            tarjan(v, e[i].id);
            low[u] = min(low[u], low[v]);
            
            // 判定桥
            if (low[v] > dfn[u]) {
                bridges.emplace_back(min(u, v), max(u, v));
            }
        } else {
            low[u] = min(low[u], dfn[v]);
        }
    }
}
```

### 求割点（同时求桥）

```cpp
vector<int> cut_points;
bool is_cut[MAXN];  // 是否为割点

void tarjan_cut(int u, int parent) {
    dfn[u] = low[u] = ++dfncnt;
    int child_count = 0;
    bool is_cut_point = false;
    
    for (int i = head[u]; i; i = e[i].nxt) {
        int v = e[i].to;
        if (e[i].id == parent) continue;
        
        if (!dfn[v]) {
            child_count++;
            tarjan_cut(v, e[i].id);
            low[u] = min(low[u], low[v]);
            
            // 割点判定
            if (parent != -1 && low[v] >= dfn[u]) {
                is_cut_point = true;
            }
            
            // 桥判定
            if (low[v] > dfn[u]) {
                bridges.emplace_back(min(u, v), max(u, v));
            }
        } else {
            low[u] = min(low[u], dfn[v]);
        }
    }
    
    // 根结点特殊判定
    if (parent == -1 && child_count > 1) {
        is_cut_point = true;
    }
    
    if (is_cut_point) {
        cut_points.push_back(u);
        is_cut[u] = true;
    }
}
```

### 完整示例

```cpp
#include <bits/stdc++.h>
using namespace std;

const int MAXN = 100005;

struct Edge {
    int to, nxt;
    int id;
} e[MAXN * 2];

int head[MAXN], cnt;
int n, m;
int dfn[MAXN], low[MAXN], dfncnt;
vector<pair<int, int>> bridges;
vector<int> cut_points;

void add_edge(int u, int v, int id) {
    e[++cnt] = {v, head[u], id};
    head[u] = cnt;
}

void tarjan(int u, int parent_edge, bool is_root) {
    dfn[u] = low[u] = ++dfncnt;
    int child_count = 0;
    
    for (int i = head[u]; i; i = e[i].nxt) {
        int v = e[i].to;
        if (e[i].id == parent_edge) continue;
        
        if (!dfn[v]) {
            child_count++;
            tarjan(v, e[i].id, false);
            low[u] = min(low[u], low[v]);
            
            // 桥
            if (low[v] > dfn[u]) {
                bridges.emplace_back(min(u, v), max(u, v));
            }
            
            // 割点（非根）
            if (!is_root && low[v] >= dfn[u]) {
                cut_points.push_back(u);
            }
        } else {
            low[u] = min(low[u], dfn[v]);
        }
    }
    
    // 割点（根）
    if (is_root && child_count > 1) {
        cut_points.push_back(u);
    }
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    cin >> n >> m;
    for (int i = 1; i <= m; i++) {
        int u, v;
        cin >> u >> v;
        add_edge(u, v, i);
        add_edge(v, u, i);
    }
    
    for (int i = 1; i <= n; i++) {
        if (!dfn[i]) {
            tarjan(i, -1, true);
        }
    }
    
    sort(cut_points.begin(), cut_points.end());
    cut_points.erase(unique(cut_points.begin(), cut_points.end()), cut_points.end());
    
    cout << "Bridges: " << bridges.size() << endl;
    for (auto [u, v] : bridges) {
        cout << u << " - " << v << endl;
    }
    
    cout << "Cut points: " << cut_points.size() << endl;
    for (int v : cut_points) {
        cout << v << " ";
    }
    cout << endl;
    
    return 0;
}
```

## 复杂度分析

- **时间复杂度**：$O(n + m)$
- **空间复杂度**：$O(n + m)$

## 应用场景

- **图的连通性分析**：判断图的连通分量、桥、割点
- **网络可靠性**：通信网络中，去除哪些节点会导致网络断裂
- **双连通分量**：将图分解为双连通分量的过程

## 相关主题

- [[../algorithms/strongly-connected-components|强连通分量]]：有向图的连通性
- [[../algorithms/minimum-spanning-tree|最小生成树]]：桥在 MST 中的应用
- [[../algorithms/topological-sort|拓扑排序]]：DAG 中的应用

## 参考资料

[^1]: [割点和桥 - OI Wiki](https://oi-wiki.org/graph/cut/)
