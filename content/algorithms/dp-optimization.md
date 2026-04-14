---
title: 动态规划优化
date: 2026-04-08
description: 动态规划优化技巧：单调队列优化、斜率优化、Knuth优化、四边形不等式
tags:
  - dynamic-programming
  - dp-optimization
  - monotonic-queue
  - slope-trick
draft: false
permalink:
---

## 概述

动态规划（DP）虽然思想简单，但状态转移的朴素实现往往时间复杂度较高。DP优化通过挖掘状态转移的性质，将$O(n^2)$或更高复杂度的DP优化到$O(n \log n)$甚至$O(n)$。[^1]

---

## 单调队列优化

### 适用场景

DP转移形如：

$$
dp[i] = \min_{j < i} \{ dp[j] + cost(j, i) \}
$$

其中 $cost(j, i)$ 满足**单调队列可分解**的性质，即：

$$
cost(j, i) = w_j + b_i \quad \text{或} \quad cost(j, i) = w_j \cdot c_i
$$

### 经典问题：滑动最值

```cpp
// 朴素O(nk)
vector<int> slidingMaximum(vector<int>& a, int k) {
    int n = a.size();
    vector<int> ans;
    for (int i = k - 1; i < n; i++) {
        int mx = a[i];
        for (int j = i - k + 1; j < i; j++) {
            mx = max(mx, a[j]);
        }
        ans.push_back(mx);
    }
    return ans;
}

// 单调队列优化O(n)
vector<int> slidingMaximum_opt(vector<int>& a, int k) {
    int n = a.size();
    vector<int> ans;
    deque<int> dq;  // 存下标，保持递减
    
    for (int i = 0; i < n; i++) {
        // 维护窗口：移除超出范围的下标
        while (!dq.empty() && dq.front() <= i - k) dq.pop_front();
        
        // 维护单调性：移除比当前元素小的
        while (!dq.empty() && a[dq.back()] <= a[i]) dq.pop_back();
        
        dq.push_back(i);
        
        if (i >= k - 1) ans.push_back(a[dq.front()]);
    }
    return ans;
}
```

### DP优化：状态转移

对于形如：

$$
dp[i] = \min_{j < i} \{ dp[j] + f(j) + g(i) \}
$$

可以使用单调队列优化（当 $f(j)$ 具有单调性时）。

```cpp
// 原始DP（假设f单调递增）
// dp[i] = min_{j < i} { dp[j] + f(j) } + g(i)

// 单调队列优化
int dp_optimized(int n) {
    deque<int> dq;  // 存可能成为最优解的j
    dq.push_back(0);  // j = 0
    
    for (int i = 1; i <= n; i++) {
        // dp[i] = dp[dq.front()] + f(dq.front()) + g(i)
        dp[i] = dp[dq.front()] + f(dq.front()) + g(i);
        
        // 插入新的j=i，维护队列
        // 移除不满足单调性的j
        while (!dq.empty() && 
               dp[dq.back()] + f(dq.back()) >= dp[i] + f(i)) {
            dq.pop_back();
        }
        dq.push_back(i);
    }
    return dp[n];
}
```

### 经典例题：股票买卖含冷冻期

```cpp
// dp[i][0] = max profit on day i with no stock
// dp[i][1] = max profit on day i with stock
// dp[i][2] = max profit on day i in cooldown

for (int i = 0; i < n; i++) {
    if (i == 0) {
        dp[0][0] = 0;
        dp[0][1] = -prices[0];
        dp[0][2] = 0;
    } else {
        dp[i][0] = max(dp[i-1][0], dp[i-1][2]);  // rest
        dp[i][1] = max(dp[i-1][1], dp[i-1][0] - prices[i]);  // buy
        dp[i][2] = dp[i-1][1] + prices[i];  // sell (cooldown)
    }
}
```

---

## 斜率优化

### 适用场景

DP转移形如：

$$
dp[i] = \min_{j < i} \{ dp[j] + (C_i - C_j)^2 \} \quad \text{或}
$$

$$
dp[i] = \min_{j < i} \{ dp[j] + (C_i \cdot K_j - K_j \cdot C_j) \}
$$

### 证明：化简为直线形式

设 $dp[j] + K_j^2 - 2 \cdot C_i \cdot K_j = (dp[j] + K_j^2) - 2K_j \cdot C_i$

令 $y = dp[j] + K_j^2$，$x = 2K_j$，$k = C_i$

则 $y - x \cdot k$ 是关于 $(x, y)$ 的直线在斜率 $k$ 处的取值。

我们需要在所有点 $(x_j, y_j)$ 中找到斜率为 $C_i$ 的切线斜率最小值。

### 凸壳维护

```cpp
// 点结构
struct Point {
    double x, y;  // x = 2*K_j, y = dp[j] + K_j^2
};

// 判断点p是否在点a,b组成的凸壳内部
bool bad(Point a, Point b, Point c) {
    // 交叉乘积判凸性
    return (c.y - a.y) * (c.x - b.x) <= (b.y - a.y) * (c.x - a.x);
}

class ConvexHull {
    vector<Point> hull;
    
    void add(Point p) {
        while (hull.size() >= 2 && bad(hull[hull.size()-2], hull.back(), p)) {
            hull.pop_back();
        }
        hull.push_back(p);
    }
    
    // 查询斜率为k的最小截距
    double query(double k) {
        // 二分查找切点（斜率递增）
        int l = 0, r = hull.size() - 1;
        while (l < r) {
            int m = (l + r) / 2;
            // 比较m和m+1哪个更优
            if (hull[m].y - hull[m].x * k < hull[m+1].y - hull[m+1].x * k) {
                r = m;
            } else {
                l = m + 1;
            }
        }
        return hull[l].y - hull[l].x * k;
    }
};
```

### 完整DP优化模板

```cpp
// 原始问题：dp[i] = min_{j < i} { dp[j] + (C_i - C_j)^2 }
// 假设 C_i = prefix_sum[i]

int n;
vector<long long> C(n+1), dp(n+1);

ConvexHull ch;
ch.add({2*C[0], dp[0] + C[0]*C[0]});  // j=0

for (int i = 1; i <= n; i++) {
    // 查询最优转移
    dp[i] = ch.query(C[i]) + C[i]*C[i];
    
    // 加入新的点j=i
    ch.add({2*C[i], dp[i] + C[i]*C[i]});
}
```

---

## Knuth优化

### 适用条件

DP形如：

$$
dp[i][j] = \min_{i < k < j} \{ dp[i][k] + dp[k][j] \} + C[i][j]
$$

且 $C$ 满足**四边形不等式**和**单调性**：

$$
C[a][c] + C[b][d] \leq C[a][d] + C[b][c] \quad \text{(四边形不等式)}
$$

$$
C[b][c] \leq C[a][d] \quad \text{(单调性)}, \quad a \leq b \leq c \leq d
$$

### 核心性质

最优决策点单调：

$$
opt[i][j-1] \leq opt[i][j] \leq opt[i+1][j]
$$

### 实现

```cpp
// 原始O(n^3)
for (int len = 2; len <= n; len++) {
    for (int i = 1; i + len - 1 <= n; i++) {
        int j = i + len - 1;
        dp[i][j] = INF;
        for (int k = i + 1; k < j; k++) {
            dp[i][j] = min(dp[i][j], dp[i][k] + dp[k][j] + C[i][j]);
        }
    }
}

// Knuth优化O(n^2)
vector<vector<int>> opt(n+1, vector<int>(n+1, 0));

for (int i = 1; i <= n; i++) {
    opt[i][i] = i;
}

for (int len = 2; len <= n; len++) {
    for (int i = 1; i + len - 1 <= n; i++) {
        int j = i + len - 1;
        dp[i][j] = INF;
        // 只需要搜索 opt[i][j-1] 到 opt[i+1][j]
        for (int k = opt[i][j-1]; k <= opt[i+1][j]; k++) {
            if (dp[i][j] > dp[i][k] + dp[k][j] + C[i][j]) {
                dp[i][j] = dp[i][k] + dp[k][j] + C[i][j];
                opt[i][j] = k;
            }
        }
    }
}
```

### 经典例题：矩阵链乘法

给定矩阵链 $A_1, A_2, ..., A_n$，找到最小乘法次数的括号方案。

```cpp
// 矩阵维度: A_i 是 p[i-1] x p[i]
// dp[i][j] = 最小乘法次数

vector<vector<int>> dp(n, vector<int>(n, 0));
vector<vector<int>> opt(n, vector<int>(n, 0));

for (int i = 0; i < n; i++) opt[i][i] = i;

for (int len = 2; len <= n; len++) {
    for (int i = 0; i + len - 1 < n; i++) {
        int j = i + len - 1;
        dp[i][j] = INT_MAX;
        for (int k = opt[i][j-1]; k <= opt[i+1][j]; k++) {
            int cost = dp[i][k] + dp[k+1][j] + p[i]*p[k+1]*p[j+1];
            if (cost < dp[i][j]) {
                dp[i][j] = cost;
                opt[i][j] = k;
            }
        }
    }
}
```

---

## 四边形不等式优化

### 定义

对于代价函数 $w(i, j)$，如果对任意 $a < b < c < d$ 有：

$$
w(a, c) + w(b, d) \leq w(a, d) + w(b, c)
$$

则称 $w$ 满足**四边形不等式**。

### 证明决策单调性

若 $C[i][j]$ 满足四边形不等式和单调性，则 $opt[i][j]$ 单调。

这意味着可以应用**Divide and Conquer Optimization**（CDQ分治优化）。

### CDQ分治优化

```cpp
void solve(int l, int r, int optL, int optR) {
    if (l > r) return;
    
    int mid = (l + r) >> 1;
    int bestK = -1;
    dp[l][mid] = INF;
    
    // 在optL到optR范围内找最优k
    for (int k = optL; k <= min(mid-1, optR); k++) {
        int cost = dp[k][mid] = dp[k][mid] + C[k][mid];
        if (cost < dp[l][mid]) {
            dp[l][mid] = cost;
            bestK = k;
        }
    }
    
    solve(l, mid-1, optL, bestK);
    solve(mid+1, r, bestK, optR);
}
```

---

## 复杂度对比

| 优化方法 | 适用条件 | 复杂度 |
|---------|---------|-------|
| 朴素DP | - | $O(n^2)$ ~ $O(n^3)$ |
| 单调队列 | 转移呈单调队列形式 | $O(n)$ |
| 斜率优化 | 转移可化为一次函数 | $O(n \log n)$ |
| Knuth优化 | 四边形不等式+单调性 | $O(n^2)$ |
| CDQ分治 | 决策单调 | $O(n \log n)$ |

---

## 参考资料

[^1]: 本词条参考[动态规划优化 - OI Wiki](https://oi-wiki.org/dp/opt/)

---

## 相关主题

- [[monotonic-queue|单调队列]] - 滑动窗口问题
- [[divide-and-conquer|递归与分治]] - CDQ分治基础
