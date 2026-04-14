---
title: Hamilton路径
date: 2026-04-07
description: 哈密顿路径与哈密顿回路（Hamilton Path）
tags:
  - hamilton
  - graph
draft: false
permalink:
---

## 定义

### 哈密顿路径（Hamiltonian Path）

在图中找到一条路径，使得该路径恰好经过**每个顶点一次**。若路径的起点和终点相同，则称为**哈密顿回路（Hamiltonian Cycle）**；若不同，则称为**哈密顿通路**。

### 与欧拉路径的区别

| 特征 | 欧拉路径 | Hamilton 路径 |
|------|----------|----------------|
| 遍历对象 | **边**（每条边恰好一次） | **顶点**（每个顶点恰好一次） |
| 判定难度 | 多项式时间 $O(E)$ | NP 完全 |
| 存在条件 | 度数奇偶性 + 连通性 | 无简单充要条件 |

---

## NP 完全性

哈密顿路径问题是 **NP 完全** 的。这意味着：

1. 目前没有已知的多项式时间算法来判定一般图中哈密顿路径的存在性
2. 若存在多项式时间算法，则 $P = NP$（千禧年七大难题之一）
3. 实际竞赛中，通常在**特殊图**或**小规模**情况下求解

---

## 判定条件

### 竞赛图（Tournament Graph）

**定理**：任意竞赛图必定存在哈密顿路径。[^1]

**证明（数学归纳法）**：

设 $n$ 为竞赛图的顶点数。

- **基例**：当 $n = 1, 2$ 时，显然存在哈密顿路径。

- **归纳假设**：假设对于 $n = k$ 的竞赛图，命题成立。

- **归纳步骤**：对于 $n = k + 1$ 的竞赛图：
  1. 任意删除一个顶点 $v$，得到 $k$ 个顶点的子图 $G'$
  2. 由归纳假设，$G'$ 存在哈密顿路径 $P' = (v_1, v_2, \ldots, v_k)$
  3. 将顶点 $v$ 插入路径 $P'$ 中：
     - 找到第一个满足存在边 $v_i \to v$ 的顶点 $v_i$（即 $v$ 可以到达 $v_i$）
     - 若不存在，则将 $v$ 放在路径末尾
     - 否则，将 $v$ 放在 $v_{i-1}$ 和 $v_i$ 之间

  由此得到的路径正好经过所有 $k+1$ 个顶点一次。$\square$

### 哈密顿回路的必要条件

**Dirac 定理**：设 $G$ 为 $n \geq 3$ 的简单无向图，若对于任意顶点 $v$，都有 $\deg(v) \geq n/2$，则 $G$ 存在哈密顿回路。[^2]

### 哈密顿回路的充分条件

**Ore 定理**：设 $G$ 为 $n \geq 3$ 的简单无向图，若对于任意**不相邻**的顶点对 $(u, v)$，都有 $\deg(u) + \deg(v) \geq n$，则 $G$ 存在哈密顿回路。

---

## 求解方法

### 状压 DP

对于**带权完全图**上的最短哈密顿路径问题（即经典 TSP），可以使用状压 DP 求解。

**状态定义**：

$$
dp[mask][i] = \text{从顶点 } 0 \text{ 出发，经过 } mask \text{ 中的顶点（包含 } i \text{），终点为 } i \text{ 的最短路径长度}
$$

**状态转移**：

$$
dp[mask][i] = \min_{j \in mask, j \neq i} \left( dp[mask \setminus \{i\}][j] + w[j][i] \right)
$$

**复杂度**：$O(n^2 \cdot 2^n)$

### 竞赛图哈密顿路径构造

竞赛图中构造哈密顿路径存在 $O(n^2)$ 的算法，其核心思想即上述归纳证明的**构造性算法**：

1. 维护一条有序路径 $P = (v_1, v_2, \ldots, v_k)$
2. 依次插入新顶点 $v$：在路径中找到第一个 $v_i$ 使得存在边 $v_i \to v$
3. 若找到，将 $v$ 插入 $v_{i-1}$ 和 $v_i$ 之间；否则放在末尾

---

## 代码模板

### 状压 DP（最短 Hamilton 路径）

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n;
    cin >> n;
    vector<vector<int>> w(n, vector<int>(n));
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            cin >> w[i][j];
        }
    }
    
    const int INF = 1e9;
    // dp[mask][i] = 经过mask中的点，终点为i的最短路径长度
    vector<vector<int>> dp(1 << n, vector<int>(n, INF));
    dp[1 << 0][0] = 0;  // 起点为0
    
    for (int mask = 1; mask < (1 << n); mask++) {
        for (int i = 0; i < n; i++) {
            if (!(mask & (1 << i))) continue;      // i不在mask中
            if (dp[mask][i] == INF) continue;       // 不可达
            
            for (int j = 0; j < n; j++) {
                if (mask & (1 << j)) continue;      // j已在路径中
                int nmask = mask | (1 << j);
                dp[nmask][j] = min(dp[nmask][j], dp[mask][i] + w[i][j]);
            }
        }
    }
    
    int ans = INF;
    int fullMask = (1 << n) - 1;
    for (int i = 1; i < n; i++) {
        ans = min(ans, dp[fullMask][i] + w[i][0]);  // 回到起点0
    }
    
    cout << ans << endl;
    return 0;
}
```

### 竞赛图 Hamilton 路径构造

```cpp
#include <bits/stdc++.h>
using namespace std;

// 竞赛图：任意两点之间恰有一条有向边
// 返回哈密顿路径（顶点序列）
vector<int> tournamentHamiltonPath(const vector<vector<int>>& g) {
    int n = g.size();
    vector<int> path = {0};  // 初始路径只有一个顶点
    
    for (int v = 1; v < n; v++) {
        // 在路径中找到插入位置
        int pos = path.size();  // 默认插到最后
        for (int i = 0; i < (int)path.size(); i++) {
            // g[v][path[i]] == 1 表示存在边 v -> path[i]
            if (g[v][path[i]]) {
                pos = i;
                break;
            }
        }
        path.insert(path.begin() + pos, v);
    }
    
    return path;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n;
    cin >> n;
    vector<vector<int>> g(n, vector<int>(n, 0));
    
    // 读入竞赛图（每对顶点之间有一条有向边）
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            if (i == j) continue;
            int x;
            cin >> x;
            g[i][j] = x;  // x为1表示 i -> j
        }
    }
    
    vector<int> path = tournamentHamiltonPath(g);
    
    cout << "Hamilton路径: ";
    for (int v : path) cout << v << " ";
    cout << endl;
    
    return 0;
}
```

---

## 例题：最短 Hamilton 路径

### 题目描述

给定 $n$ 个城市的坐标 $(x_i, y_i)$，求从城市 $0$ 出发，经过所有城市恰好一次后回到城市 $0$ 的最短路径长度。[^3]

### 解题思路

这是经典的 TSP 问题，使用状压 DP 求解。

设 $dp[mask][i]$ 表示从城市 $0$ 出发，经过 $mask$ 中的城市（包含 $i$），当前在城市 $i$ 的最短路径长度。转移时枚举下一个要访问的城市 $j$。

答案为 $\min_{i \neq 0} (dp[fullMask][i] + dist(i, 0))$。

### 参考代码

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n;
    cin >> n;
    vector<pair<int,int>> p(n);
    for (int i = 0; i < n; i++) {
        cin >> p[i].first >> p[i].second;
    }
    
    // 预处理距离矩阵
    vector<vector<int>> w(n, vector<int>(n));
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            long long dx = p[i].first - p[j].first;
            long long dy = p[i].second - p[j].second;
            w[i][j] = (int)(sqrt(dx*dx + dy*dy) + 0.5);
        }
    }
    
    const int INF = 1e9;
    vector<vector<int>> dp(1 << n, vector<int>(n, INF));
    dp[1 << 0][0] = 0;
    
    for (int mask = 1; mask < (1 << n); mask++) {
        for (int i = 0; i < n; i++) {
            if (!(mask & (1 << i))) continue;
            if (dp[mask][i] == INF) continue;
            
            for (int j = 0; j < n; j++) {
                if (mask & (1 << j)) continue;
                int nmask = mask | (1 << j);
                dp[nmask][j] = min(dp[nmask][j], dp[mask][i] + w[i][j]);
            }
        }
    }
    
    int fullMask = (1 << n) - 1;
    int ans = INF;
    for (int i = 1; i < n; i++) {
        ans = min(ans, dp[fullMask][i] + w[i][0]);
    }
    
    cout << ans << endl;
    return 0;
}
```

---

## 总结

| 方法 | 适用场景 | 时间复杂度 |
|------|----------|------------|
| 状压 DP | 带权完全图（$n \leq 20$） | $O(n^2 \cdot 2^n)$ |
| 竞赛图构造 | 竞赛图（任意两点间恰有一条有向边） | $O(n^2)$ |
| Dirac / Ore 定理 | 判定无向图哈密顿回路存在性 | $O(n)$ 判定 |

哈密顿路径问题虽然本质上是 NP 难的，但在竞赛中往往通过挖掘题目图的特殊性质（如完全图、竞赛图、二分图等）来设计多项式算法。

---

## 参考资料

[^1]: 竞赛图必有哈密顿路径这一结论最早由 Redei 于 1934 年证明。

[^2]: Dirac G. (1952). "A theorem of R. L. Brooks on the colouring of graphs"

[^3]: 本题来自 LeetCode 847，是经典的状态压缩 DP 例题。

[^4]: 本段参考了 [哈密顿路径 - OI Wiki](https://oi-wiki.org/graph/hamilton/)
