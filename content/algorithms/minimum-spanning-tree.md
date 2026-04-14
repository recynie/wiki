---
title: 最小生成树
date: 2026-04-01
description: 最小生成树（Minimum Spanning Tree, MST）算法详解
tags:
  - graph
  - mst
  - kruskal
  - prim
draft: false
permalink:
---

## 定义

### 什么是生成树

给定一个无向连通图 $G = (V, E)$，其中 $V$ 为顶点集合，$E$ 为边集合。**生成树**（Spanning Tree）是图 $G$ 的一个子图，它满足以下条件：

1. 是连通的无环图
2. 包含图中的所有顶点
3. 边的数量恰好为 $|V| - 1$

换句话说，生成树是在保持图连通的前提下，删去尽可能多的边后得到的树形结构。

### 最小生成树

在一个带权无向连通图中，**最小生成树**（Minimum Spanning Tree, MST）是指所有生成树中，边权之和最小的那个。

设原图 $G = (V, E)$，每条边 $e \in E$ 的权重为 $w(e)$，则最小生成树 $T_{MST}$ 满足：

$$
w(T_{MST}) = \min_{T \text{ 是 } G \text{ 的生成树}} \sum_{e \in T} w(e)
$$

### 前置知识

理解最小生成树算法需要掌握以下基础知识：

- [[../data-structure/union-find|并查集]]：Kruskal 算法中用于判断环和合并集合
- [[heap-sort|堆]]：Prim 算法中用于高效获取最小边
- [[../data-structure/graph-basic|图论基础]]：邻接表、邻接矩阵等图表示方法

---

## Kruskal 算法

### 算法思想

Kruskal 算法的核心思想是**贪心**：将所有边按权重从小到大排序，依次遍历每条边，如果当前边连接的两个顶点不在同一个连通分量中，则将这条边加入最小生成树。

具体步骤：

1. 将图中的所有边按权重升序排列
2. 初始化并查集，每个顶点各自成为一个集合
3. 依次处理每条边 $(u, v, w)$：
   - 如果 $u$ 和 $v$ 不在同一集合，则将它们合并，并将该边加入 MST
   - 如果已在同一集合，则跳过（会形成环）
4. 当已选边数达到 $n - 1$ 时算法结束

### 算法正确性证明

**定理**：Kruskal 算法生成的树是最小生成树。

**证明**（Cut Property 归纳法）：

设经过若干次迭代后，已选的边集合为 $S$，剩余的边构成集合 $E \setminus S$。

考虑权重最小的边 $e = (u, v, w)$，该边连接两个不同的连通分量 $C_1$ 和 $C_2$（否则会被跳过）。

将图划分为两部分：$C_1$ 和 $V \setminus C_1$，这构成一个 **cut**（割）。根据 **Cut Property**：

> 在一个带权图中，任意一个割中权重最小的边必定属于某个最小生成树。

由于 $e$ 是该割中权重最小的边，它必然属于某个最小生成树。将 $e$ 加入 $S$ 不会影响算法的正确性。

通过数学归纳法，当算法终止时，$S$ 中的 $n-1$ 条边构成一棵最小生成树。$\square$

### 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Edge {
    int u, v, w;
    bool operator<(const Edge& other) const {
        return w < other.w;
    }
};

struct DSU {
    vector<int> fa;
    DSU(int n) : fa(n+1) {
        iota(fa.begin(), fa.end(), 0);
    }
    int find(int x) {
        return fa[x] == x ? x : fa[x] = find(fa[x]);
    }
    bool unite(int x, int y) {
        x = find(x); y = find(y);
        if (x == y) return false;
        fa[x] = y;
        return true;
    }
};

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    int n, m;
    cin >> n >> m;
    vector<Edge> edges(m);
    for (int i = 0; i < m; i++)
        cin >> edges[i].u >> edges[i].v >> edges[i].w;
    sort(edges.begin(), edges.end());
    
    DSU dsu(n);
    long long ans = 0;
    int cnt = 0;
    for (auto& e : edges) {
        if (dsu.unite(e.u, e.v)) {
            ans += e.w;
            cnt++;
            if (cnt == n - 1) break;
        }
    }
    cout << ans << endl;
    return 0;
}
```

### 时间复杂度

| 操作 | 复杂度 |
|------|--------|
| 边排序 | $O(m \log m)$ |
| 并查集操作 | $O(m \alpha(n))$ |
| **总时间** | $O(m \log m)$ |

其中 $\alpha(n)$ 是 Ackermann 函数的反函数，近似为常数。

### 适用场景

Krusal 算法适用于：
- 稀疏图（边数 $m$ 较少）
- 需要离线处理所有边
- 图以边列表形式给出

---

## Prim 算法

### 算法思想

Prim 算法是另一种经典 MST 算法，它从一个顶点开始，**逐步扩展**生成树的边界，直到所有顶点都被包含。

具体步骤：

1. 任选一个起始顶点 $s$
2. 维护一个最小堆，存储候选边
3. 初始时，将 $s$ 加入生成树
4. 重复以下步骤直到所有顶点都被访问：
   - 取出当前连接已访问集和未访问集的最小边 $(u, v, w)$
   - 将 $v$ 标记为已访问，将 $w$ 累加到答案
   - 更新与 $v$ 相邻的所有未访问顶点的距离

### 算法正确性证明

**定理**：Prim 算法生成的树是最小生成树。

**证明**（Cut Property 直接证明）：

在算法的每一步，已访问顶点集合为 $S$，未访问集合为 $V \setminus S$。这构成了一个割 $(S, V \setminus S)$。

算法每次选择的边是连接 $S$ 和 $V \setminus S$ 的边中权重最小的。直接由 Cut Property 可知，这条边必然属于某个最小生成树。

通过反证法可证：若最小生成树不包含算法选择的边，则可以将该边替换为 MST 中的对应边，得到的生成树权重不会增加。

当所有顶点都被加入时，得到的生成树就是 MST。$\square$

### 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    int n, m;
    cin >> n >> m;
    vector<vector<pair<int,int>>> adj(n+1);
    for (int i = 0; i < m; i++) {
        int u, v, w;
        cin >> u >> v >> w;
        adj[u].push_back({v, w});
        adj[v].push_back({u, w});
    }
    
    vector<int> dist(n+1, INT_MAX);
    vector<int> visited(n+1, false);
    priority_queue<pair<int,int>, vector<pair<int,int>>, greater<pair<int,int>>> pq;
    
    dist[1] = 0;
    pq.push({0, 1});
    long long ans = 0;
    
    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        if (visited[u]) continue;
        visited[u] = true;
        ans += d;
        
        for (auto [v, w] : adj[u]) {
            if (!visited[v] && w < dist[v]) {
                dist[v] = w;
                pq.push({w, v});
            }
        }
    }
    cout << ans << endl;
    return 0;
}
```

### 时间复杂度

| 实现方式 | 时间复杂度 |
|----------|-----------|
| 朴素实现（遍历所有边） | $O(n^2)$ |
| 堆优化 | $O(m \log n)$ |
| Fibonacci 堆 | $O(m + n \log n)$ |

堆优化版本中，每次更新操作 $O(\log n)$，共 $m$ 次更新。

### 适用场景

Prim 算法适用于：
- 稠密图（边数 $m$ 接近 $n^2$）
- 图以邻接表形式给出
- 需要在线增量计算

---

## Borůvka 算法

Borůvka 算法是第三种经典的 MST 算法，由 Otakar Borůvka 于 1926 年提出。该算法采用**并行化**的思想，每轮同时选择多个安全边。

### 算法步骤

1. 初始时每个顶点各自为一个连通分量
2. 对于每个连通分量，找出连接该分量与 **不同** 分量的权重最小的边
3. 将所有找到的最小边加入 MST，并合并对应的连通分量
4. 重复直到只剩一个连通分量

### 特点

- **优势**：可以实现并行化，适合分布式计算
- **缺点**：实现相对复杂
- **时间复杂度**：$O(m \log n)$

在竞赛中较少直接使用 Borůvka 算法，但它在某些特殊场景（如完全图并行计算）中很有价值。

---

## 最小生成树的唯一性

### 什么时候 MST 是唯一的？

一个带权图的最小生成树**唯一**当且仅当：**所有边的权重都不相同**（即没有权重相等的边）。

更形式化地说：
- 如果任意两条不同边的权重都不相等，则 MST 唯一
- 如果存在权重相等的边，则可能存在多个不同的 MST

### 唯一性判定方法

给定图 $G$，判断 MST 是否唯一：

1. 按权重对所有边排序
2. 对于每条权重相同的边，使用并查集检查它们是否会形成环
3. 如果某条边连接的两个顶点在加入它之前已经连通（且该边的权重与当前处理的权重相同），则 MST 不唯一

### 次小生成边问题

如果要求所有 MST（而非仅仅权重和最小），可以枚举所有权重相同的边，递归计算。

---

## 次小生成树

### 问题定义

**次小生成树**（Second Best MST）是指权重仅次于最小生成树的生成树。可能存在多个，次小生成树的权重是所有非 MST 生成树中的最小值。

### 朴素算法

1. 先求出 MST，设为 $T$，其权重为 $W_{MST}$
2. 遍历每条 **不在** $T$ 中的边 $(u, v, w)$
3. 在 $T$ 中找到 $u$ 到 $v$ 路径上的最大边权 $maxEdge(u, v)$
4. 计算以 $(u, v, w)$ 替换 $maxEdge(u, v)$ 后的生成树权重：$W' = W_{MST} - maxEdge(u, v) + w$
5. 所有 $W'$ 中的最小值即为次小生成树权重

### 时间复杂度

- 求 MST：$O(m \log n)$ 或 $O(n^2)$
- 对每条非树边，用 LCA 或树上倍增求路径最大边权：$O(m \log n)$
- **总时间**：$O(m \log n)$

### 代码框架

```cpp
// 预处理：LCA + 树上最大边权
int n, m;
vector<vector<pair<int,int>>> adj(n+1);
vector<vector<int>> maxEdge(n+1, vector<int>(LOGN, 0));
vector<vector<int>> parent(n+1, vector<int>(LOGN, -1));
vector<int> depth(n+1);

// 1. 求 MST
// 2. 在 MST 上预处理 parent 和 maxEdge（DFS/BFS）
// 3. 对每条非 MST 边计算替换代价，取最小值
```

---

## 应用场景

最小生成树在实际中有广泛的应用：

### 1. 网络设计

- 电路板布线：连接所有元件的最小线缆长度
- 通信网络：构建最低成本的通信网络
- 公路/铁路网：连接所有城市的最小道路长度

### 2. 聚类分析

- 层次聚类：Kruskal 算法天然适合构建层次聚类树
- 图像分割：将像素视为顶点，根据相似度构建 MST

### 3. 近似算法

-旅行商问题（TSP）的启发式解法

### 4. 其他应用

- 随机生成树：MST 是图论研究的重要基础
- 坐标几何中的最近邻问题

---

## 算法对比

| 特性 | Kruskal | Prim |
|------|---------|------|
| 算法思想 | 选边 | 选点 |
| 数据结构 | 并查集 | 优先队列 |
| 适合图类型 | 稀疏图 | 稠密图 |
| 时间复杂度 | $O(m \log m)$ | $O(m \log n)$ |
| 实现难度 | 简单 | 中等 |

实际选择时，通常根据图的稀疏程度选择：
- 稀疏图：优先选择 Kruskal
- 稠密图：优先选择 Prim

---

## 参考资料

[^1]: 本段参考[最小生成树 - OI Wiki](https://oi-wiki.org/graph/mst/)
