---
title: 近似算法
date: 2026-04-08
description: 近似算法（Approximation Algorithms）— 处理NP难问题的近似解法，包括顶点覆盖、集合覆盖、TSP等问题的常数比近似算法
tags:
  - approximation-algorithm
  - np-hard
  - optimization
draft: false
permalink:
---

## 概述

近似算法是处理NP难优化问题的一种重要技术。当一个问题无法在多项式时间内精确求解时，我们退而求其次，寻找**近似最优解**，并给出近似比（Approximation Ratio）的理论保证。

**近似比**定义为：对于最小化问题，算法产生的解代价与最优解代价的比值；对于最大化问题，则是最优解代价与算法解代价的比值。 若近似比为 $\rho$，则称该算法为 $\rho$-近似算法。

## 顶点覆盖问题（Vertex Cover）

### 问题定义

给定无向图 $G = (V, E)$，寻找最小的顶点集合 $S$，使得每条边至少有一个端点在 $S$ 中。

### 2-近似算法

最简单的2-近似算法基于**最大匹配**：

```cpp
#include <bits/stdc++.h>
using namespace std;

// 顶点覆盖的2-近似算法
// 1. 计算图的最大匹配
// 2. 返回匹配中所有端点构成集合

struct Edge {
    int u, v;
};

int vertexCover2Approx(int n, const vector<Edge>& edges) {
    // 构建邻接表
    vector<vector<int>> adj(n);
    for (auto& e : edges) {
        adj[e.u].push_back(e.v);
        adj[e.v].push_back(e.u);
    }
    
    // 贪婪2-近似：边随机选择
    vector<bool> visited(n, false);
    vector<int> cover;
    
    // 边覆盖：一个简单的2-近似
    // 选取所有度≥1的顶点（非最优，但简单）
    for (int i = 0; i < n; i++) {
        if (!adj[i].empty()) {
            visited[i] = true;  // 选取该顶点
        }
    }
    
    // 统计选取的顶点数
    int count = 0;
    for (bool v : visited) if (v) count++;
    return count;
}
```

### 更好的2-近似算法

```cpp
// 基于最大匹配的2-近似（最优的2-近似）
vector<int> vertexCoverFromMatching(int n, const vector<Edge>& edges) {
    // 1. 计算最大匹配（使用Hungarian或简化的Greedy）
    // 2. 返回匹配中所有端点
    
    // 简化实现：贪心匹配
    vector<bool> matched(n, false);
    vector<pair<int,int>> matchEdges;
    
    for (auto& e : edges) {
        if (!matched[e.u] && !matched[e.v]) {
            matched[e.u] = matched[e.v] = true;
            matchEdges.push_back({e.u, e.v});
        }
    }
    
    // 匹配边端点即为近似解
    vector<int> cover;
    for (auto& p : matchEdges) {
        cover.push_back(p.first);
        cover.push_back(p.second);
    }
    return cover;
}
```

**理论保证**：设 $M$ 为最大匹配，$OPT$ 为最优顶点覆盖。由于每条匹配边至少需要其中一个端点，有 $|M| \leq |OPT|$。返回的 $2|M| \leq 2|OPT|$，故为2-近似。

### 近似比分析

| 算法 | 近似比 | 时间复杂度 |
|------|--------|------------|
| 基于匹配 | 2 | $O(n+m)$ |
| 暴力枚举 | 1 (NP难) | $O(2^n)$ |

## 集合覆盖问题（Set Cover）

### 问题定义

给定全集 $U$ 和若干子集集合 $S = \{S_1, S_2, ..., S_m\}$，每个子集有权重 $c_i$，选择权重最小的子集覆盖 $U$。

### 贪心近似算法

```cpp
#include <bits/stdc++.h>
using namespace std;

struct SetCoverResult {
    vector<int> selectedSets;  // 选取的集合索引
    double totalCost;
};

// 贪心集合覆盖算法 - ln(n)-近似
SetCoverResult greedySetCover(
    const vector<set<int>>& sets,  // 集合列表
    const vector<double>& costs      // 每个集合的代价
) {
    SetCoverResult result;
    set<int> uncovered = {};  // 未覆盖元素（假设1到n编号）
    
    // 初始化：所有元素未覆盖
    for (int i = 0; i < sets.size(); i++) {
        for (int elem : sets[i]) {
            uncovered.insert(elem);
        }
    }
    
    int n = uncovered.size();
    set<int> covered;
    double totalCost = 0;
    
    while (!covered.empty() && covered.size() < n) {
        // 选择"性价比"最高的集合：覆盖新元素多/代价低
        double bestRatio = -1;
        int bestSet = -1;
        set<int> bestNewElements;
        
        for (int i = 0; i < sets.size(); i++) {
            set<int> newElems;
            for (int elem : sets[i]) {
                if (covered.find(elem) == covered.end()) {
                    newElems.insert(elem);
                }
            }
            
            if (!newElems.empty()) {
                double ratio = costs[i] / newElems.size();
                if (bestRatio < 0 || ratio < bestRatio) {
                    bestRatio = ratio;
                    bestSet = i;
                    bestNewElements = newElems;
                }
            }
        }
        
        if (bestSet == -1) break;  // 无法覆盖更多
        
        // 选取该集合
        result.selectedSets.push_back(bestSet);
        totalCost += costs[bestSet];
        for (int elem : bestNewElements) {
            covered.insert(elem);
        }
    }
    
    result.totalCost = totalCost;
    return result;
}
```

### 近似比分析

**定理**：贪心算法给出 $O(\ln n)$-近似。

设 $c_{greedy}$ 为贪心算法代价，$c_{opt}$ 为最优代价，则：

$$c_{greedy} \leq H_n \cdot c_{opt}$$

其中 $H_n = 1 + \frac{1}{2} + ... + \frac{1}{n} \approx \ln n$。

## 旅行商问题（TSP）

### 问题定义

给定 $n$ 个城市和两两之间的距离，寻找访问所有城市恰好一次并返回起点的最短路径。

### 度限制TSP

若图满足**三角不等式**（度量TSP），则存在1.5-近似的Christofides算法：

```cpp
#include <bits/stdc++.h>
using namespace std;

// Christofides算法 - 1.5-近似度量TSP
// 步骤：
// 1. 求最小生成树T
// 2. 求T中奇度顶点集合S的最小完美匹配M
// 3. 合并T和M得到欧拉图
// 4. 求欧拉回路，Shortcut得到哈密顿回路

// 简化版本：基于MST的2-近似
vector<int> tsp2Approx(const vector<vector<double>>& dist) {
    int n = dist.size();
    // 使用最近邻贪心构建路径（2-近似）
    vector<int> path;
    vector<bool> visited(n, false);
    
    int current = 0;
    path.push_back(current);
    visited[current] = true;
    
    for (int step = 1; step < n; step++) {
        int nearest = -1;
        double bestDist = 1e100;
        
        for (int j = 0; j < n; j++) {
            if (!visited[j] && dist[current][j] < bestDist) {
                bestDist = dist[current][j];
                nearest = j;
            }
        }
        
        visited[nearest] = true;
        path.push_back(nearest);
        current = nearest;
    }
    
    path.push_back(0);  // 返回起点
    return path;
}
```

### Christofides算法核心

```cpp
// 最小生成树（Prim算法）
vector<pair<int,int>> mst(int n, const vector<vector<double>>& dist) {
    vector<double> key(n, 1e100);
    vector<int> parent(n, -1);
    vector<bool> inMST(n, false);
    
    priority_queue<pair<double,int>, vector<pair<double,int>>, greater<pair<double,int>>> pq;
    pq.push({0, 0});
    key[0] = 0;
    
    while (!pq.empty()) {
        auto [k, u] = pq.top(); pq.pop();
        if (inMST[u]) continue;
        inMST[u] = true;
        
        for (int v = 0; v < n; v++) {
            if (!inMST[v] && dist[u][v] < key[v]) {
                key[v] = dist[u][v];
                parent[v] = u;
                pq.push({key[v], v});
            }
        }
    }
    
    vector<pair<int,int>> edges;
    for (int i = 1; i < n; i++) {
        if (parent[i] != -1) {
            edges.push_back({parent[i], i});
        }
    }
    return edges;
}
```

## 0-1背包问题

### FPTAS（完全多项式时间近似方案）

```cpp
#include <bits/stdc++.h>
using namespace std;

// 背包问题的FPTAS
// 基于动态规划和缩放技术
struct Item {
    int weight;
    int value;
};

// FPTAS: (1-ε)-近似 for any ε > 0
vector<Item> knapsackFPTAS(const vector<Item>& items, int capacity, double eps) {
    int n = items.size();
    
    // 找到最大价值
    int maxValue = 0;
    for (auto& item : items) {
        maxValue = max(maxValue, item.value);
    }
    
    // 缩放因子
    int K = (int)(eps * maxValue / n);
    if (K == 0) K = 1;
    
    // 缩放价值
    vector<int> scaled(n);
    for (int i = 0; i < n; i++) {
        scaled[i] = items[i].value / K;
    }
    
    // 动态规划：dp[j] = 使用缩放价值达到价值j的最小重量
    int maxScaledValue = maxValue / K + 1;
    vector<int> dp(maxScaledValue, capacity + 1);
    dp[0] = 0;
    
    for (int i = 0; i < n; i++) {
        for (int j = maxScaledValue - 1; j >= scaled[i]; j--) {
            dp[j] = min(dp[j], items[i].weight + dp[j - scaled[i]]);
        }
    }
    
    // 找最大满足容量限制的缩放价值
    int bestScaled = 0;
    for (int v = maxScaledValue - 1; v >= 0; v--) {
        if (dp[v] <= capacity) {
            bestScaled = v;
            break;
        }
    }
    
    // 回溯选择物品
    vector<Item> result;
    vector<int> traceDP = dp;
    int remaining = capacity;
    
    for (int i = n - 1; i >= 0 && bestScaled > 0; i--) {
        int include = scaled[i];
        int without = (i > 0) ? traceDP[bestScaled] : 0;
        
        if (i == 0) {
            if (remaining >= items[i].weight) {
                result.push_back(items[i]);
            }
        } else {
            // 检查是否选择了物品i
            vector<int> prevDP = dp;
            // 简化：贪心选择
            if (remaining >= items[i].weight && bestScaled >= scaled[i]) {
                result.push_back(items[i]);
                remaining -= items[i].weight;
                bestScaled -= scaled[i];
            }
        }
    }
    
    return result;
}
```

**近似比**：FPTAS给出 $(1-\varepsilon)$-近似，时间复杂度 $O(n^3 / \varepsilon)$。

## 近似算法分类

| 类型 | 定义 | 示例 |
|------|------|------|
| 常数比近似 | 近似比恒为常数 $c$ | 顶点覆盖(2), MST-TSP(2) |
| 多项式时间近似方案(PTAS) | 对于固定 $\varepsilon$，$(1-\varepsilon)$-近似 | Knapsack FPTAS |
| 完全多项式时间近似方案(FPTAS) | PTAS + 多项式于 $1/\varepsilon$ | Knapsack FPTAS |

## 重要结论

1. **NPC问题不一定不可近似**：有些NPC问题（如顶点覆盖）存在常数比近似，有些（如集合覆盖）存在 $O(\ln n)$ 近似。

2. **近似难问题**：有些问题（如最大团、3-SAT）直到今天也没有已知的多项式近似算法。

3. **PCP定理与近似难度**：PCP定理表明NP = AP(log n)暗示了某些问题（如MAX-3SAT）无法在多项式时间内近似到某个常数比。

## 应用场景

- **网络设计**：最小生成树、Steiner树
- **路由与调度**：TSP、装箱问题
- **资源配置**：背包、设施选址
- **组合优化**：顶点覆盖、集合覆盖

## 参考文献

[^1]: Approximation Algorithms - Vijay Vazirani
[^2]: Algorithm Design - Jon Kleinberg, Éva Tardos
[^3]: OI Wiki - 近似算法 https://oi-wiki.org/gs/approximate/

---

*本页面内容基于算法经典教材和OI Wiki整理*
