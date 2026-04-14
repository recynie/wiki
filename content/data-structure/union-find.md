---
title: 并查集
date: 2026-03-31
description: 并查集（Disjoint Set Union, Union-Find）
tags:
  - union-find
  - data-structure
  - graph
draft: false
permalink:
---

## 定义

并查集（Disjoint Set Union，简称 DSU 或 Union-Find）是一种用于处理**不相交集合**的合并与查询问题的数据结构。它支持两种基本操作：

- **Find（查找）**：确定某个元素属于哪个集合，通常用于判断两个元素是否在同一个集合中
- **Union（合并）**：将两个集合合并成一个集合[^1]

并查集最初用于处理连通性问题和等价关系，现在广泛用于图的连通分量维护、Kruskal 最小生成树算法等场景。

## 核心特性

### 时间复杂度

| 操作 | 朴素实现 | 路径压缩 | 路径压缩 + 按秩合并 |
|------|----------|----------|---------------------|
| Find | $O(n)$ | $O(\log n)$ | $O(\alpha(n))$ |
| Union | $O(1)$ | $O(\log n)$ | $O(\alpha(n))$ |

其中 $\alpha(n)$ 是阿克曼函数（Ackermann Function）的反函数，对于实际应用中可能出现的 $n$ 值，$\alpha(n) \leq 5$，可以认为是常数时间。[^2]

### 空间复杂度

- 空间复杂度：$O(n)$

## 基本实现

### 朴素并查集

```cpp
#include <bits/stdc++.h>
using namespace std;

class UnionFind {
private:
    vector<int> parent;  // 父节点数组
    
public:
    UnionFind(int n) {
        parent.resize(n);
        iota(parent.begin(), parent.end(), 0);  // 初始化：每个节点的父节点是自身
    }
    
    // 查找：返回元素 x 所属集合的根节点
    int find(int x) {
        if (parent[x] != x) {
            parent[x] = find(parent[x]);  // 路径压缩
        }
        return parent[x];
    }
    
    // 合并：将元素 x 和 y 所在的集合合并
    void unionSets(int x, int y) {
        int px = find(x);
        int py = find(y);
        if (px != py) {
            parent[px] = py;
        }
    }
    
    // 判断两个元素是否在同一集合
    bool connected(int x, int y) {
        return find(x) == find(y);
    }
};
```

### 路径压缩 + 按秩合并

```cpp
#include <bits/stdc++.h>
using namespace std;

class UnionFind {
private:
    vector<int> parent;
    vector<int> rank;  // 秩：树的深度近似
    
public:
    UnionFind(int n) {
        parent.resize(n);
        rank.assign(n, 0);
        iota(parent.begin(), parent.end(), 0);
    }
    
    int find(int x) {
        if (parent[x] != x) {
            parent[x] = find(parent[x]);  // 路径压缩
        }
        return parent[x];
    }
    
    void unionSets(int x, int y) {
        int px = find(x);
        int py = find(y);
        
        if (px == py) return;  // 已经在同一集合
        
        // 按秩合并：小秩合并到大秩
        if (rank[px] < rank[py]) {
            parent[px] = py;
        } else if (rank[px] > rank[py]) {
            parent[py] = px;
        } else {
            parent[py] = px;
            rank[px]++;
        }
    }
    
    bool connected(int x, int y) {
        return find(x) == find(y);
    }
    
    // 统计连通分量的个数
    int countComponents() {
        int cnt = 0;
        for (int i = 0; i < parent.size(); i++) {
            if (parent[i] == i) cnt++;
        }
        return cnt;
    }
};
```

## 代码模板

### 并查集模板（完整版）

```cpp
#include <bits/stdc++.h>
using namespace std;

struct DSU {
    vector<int> p, r;
    
    DSU(int n = 0) {
        init(n);
    }
    
    void init(int n) {
        p.resize(n);
        r.assign(n, 0);
        iota(p.begin(), p.end(), 0);
    }
    
    int find(int x) {
        if (p[x] == x) return x;
        return p[x] = find(p[x]);  // 路径压缩
    }
    
    void unite(int a, int b) {
        a = find(a);
        b = find(b);
        if (a == b) return;
        if (r[a] < r[b]) swap(a, b);
        p[b] = a;
        if (r[a] == r[b]) r[a]++;
    }
    
    bool same(int a, int b) {
        return find(a) == find(b);
    }
};

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n = 5;
    DSU dsu(n);
    
    dsu.unite(0, 1);
    dsu.unite(1, 2);
    
    cout << dsu.same(0, 2) << endl;  // 1 (true)
    cout << dsu.same(0, 3) << endl;  // 0 (false)
    
    return 0;
}
```

## 应用场景

### 岛屿数量问题

```cpp
#include <bits/stdc++.h>
using namespace std;

class UnionFind {
private:
    vector<int> parent, size;
public:
    UnionFind(int n) : parent(n), size(n, 1) {
        iota(parent.begin(), parent.end(), 0);
    }
    
    int find(int x) {
        if (parent[x] == x) return x;
        return parent[x] = find(parent[x]);
    }
    
    void unite(int a, int b) {
        a = find(a);
        b = find(b);
        if (a == b) return;
        if (size[a] < size[b]) swap(a, b);
        parent[b] = a;
        size[a] += size[b];
    }
    
    int count() {
        int cnt = 0;
        for (int i = 0; i < parent.size(); i++) {
            if (parent[i] == i) cnt++;
        }
        return cnt;
    }
};

int numIslands(vector<vector<char>>& grid) {
    if (grid.empty()) return 0;
    int m = grid.size(), n = grid[0].size();
    UnionFind dsu(m * n);
    
    auto idx = [&](int i, int j) { return i * n + j; };
    
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (grid[i][j] == '0') continue;
            if (i > 0 && grid[i-1][j] == '1') dsu.unite(idx(i, j), idx(i-1, j));
            if (j > 0 && grid[i][j-1] == '1') dsu.unite(idx(i, j), idx(i, j-1));
        }
    }
    
    int islands = 0;
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (grid[i][j] == '1' && dsu.find(idx(i, j)) == idx(i, j)) {
                islands++;
            }
        }
    }
    return islands;
}
```

### Kruskal 最小生成树

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Edge {
    int u, v, w;
    bool operator<(const Edge& other) const {
        return w < other.w;
    }
};

int kruskal(int n, vector<Edge>& edges) {
    sort(edges.begin(), edges.end());
    DSU dsu(n);
    int mstWeight = 0;
    int edgeCount = 0;
    
    for (auto& e : edges) {
        if (dsu.same(e.u, e.v)) continue;
        dsu.unite(e.u, e.v);
        mstWeight += e.w;
        edgeCount++;
        if (edgeCount == n - 1) break;
    }
    
    return mstWeight;
}
```

### 朋友圈问题

```cpp
#include <bits/stdc++.h>
using namespace std;

int findCircleNum(vector<vector<int>>& M) {
    int n = M.size();
    DSU dsu(n);
    
    for (int i = 0; i < n; i++) {
        for (int j = i + 1; j < n; j++) {
            if (M[i][j] == 1) {
                dsu.unite(i, j);
            }
        }
    }
    
    int provinces = 0;
    for (int i = 0; i < n; i++) {
        if (dsu.find(i) == i) provinces++;
    }
    return provinces;
}
```

## 扩展：带权并查集

```cpp
#include <bits/stdc++.h>
using namespace std;

class WeightedUnionFind {
private:
    vector<int> parent, rank, weight;  // weight[i] 表示 i 到 parent[i] 的距离
    
public:
    WeightedUnionFind(int n) {
        parent.resize(n);
        rank.assign(n, 0);
        weight.assign(n, 0);
        iota(parent.begin(), parent.end(), 0);
    }
    
    int find(int x) {
        if (parent[x] == x) return x;
        int px = parent[x];
        parent[x] = find(px);
        weight[x] += weight[px];  // 路径压缩时更新权重
        return parent[x];
    }
    
    double weight(int x) {
        find(x);  // 确保路径被压缩
        return weight[x];
    }
    
    void unite(int x, int y, int w) {
        // w = weight[y] - weight[x]
        int px = find(x);
        int py = find(y);
        if (px == py) return;
        
        // parent[py] = px
        // weight[py] = weight[x] + w - weight[y]
        parent[py] = px;
        weight[py] = weight[x] + w - weight[y];
    }
};
```

## 参考资料

[^1]: 并查集由伯纳德·查默斯（Bernard Chazelle）在2000年代系统化分析，其实际性能远超理论下界。

[^2]: $\alpha(n)$ 的增长极为缓慢，对于 $n = 2^{65536}$ 这样的宇宙级数字，$\alpha(n)$ 也只有 5。
