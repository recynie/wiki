---
title: 分数规划
date: 2026-04-06
description: 分数规划（Fractional Programming）——求解形如 max/min (f(x)/g(x)) 的优化问题
tags:
  - optimization
  - algorithm
draft: false
permalink:
---

## 定义

分数规划是一类优化问题，目标是求解形如：

$$
\max_{x \in S} \frac{f(x)}{g(x)} \quad \text{或} \quad \min_{x \in S} \frac{f(x)}{g(x)}
$$

其中 $f(x)$ 和 $g(x)$ 是线性或非线性函数，且 $g(x) > 0$（或 $< 0$ 根据问题而定）。[^1]

---

## 核心思想

### 转化为参数化问题

设原问题是：

$$
\max \frac{f(x)}{g(x)}
$$

令 $r = \frac{f(x)}{g(x)}$，则 $f(x) - r \cdot g(x) \geq 0$。

定义辅助函数：

$$
F(r) = \max_{x \in S} [f(x) - r \cdot g(x)]
$$

### 关键性质

- 如果 $F(r) > 0$，则存在 $x$ 使得 $\frac{f(x)}{g(x)} > r$
- 如果 $F(r) = 0$，则 $r$ 是最优解
- 如果 $F(r) < 0$，则 $r$ 太大，最优解 $< r$

因此，可以通过 **二分搜索** $r$ 的值，找到 $F(r) = 0$ 的点。

---

## 0-1 分数规划

最常见的是 0-1 分数规划，即 $x_i \in \{0, 1\}$：

$$
\max \frac{\sum_{i} a_i x_i}{\sum_{i} b_i x_i}
$$

约束：$x_i \in \{0, 1\}$，通常还有额外约束（如选择恰好 $k$ 个）

### Dinkelbach 算法

比二分更高效的方法是 **Dinkelbach 算法**：[^2]

```cpp
double fractionalKnapsack(vector<int> a, vector<int> b, int k) {
    double r = 0;  // 初始猜测
    const double EPS = 1e-7;
    
    while (true) {
        // 计算当前 r 下的最大价值
        vector<double> values;
        for (int i = 0; i < a.size(); i++) {
            values.push_back(a[i] - r * b[i]);
        }
        
        // 按价值降序排序，选择前 k 个
        sort(values.begin(), values.end(), greater<double>());
        double current = 0;
        for (int i = 0; i < k; i++) {
            current += values[i];
        }
        
        // 检查收敛
        if (fabs(current) < EPS) break;
        
        // 更新 r（牛顿法思想）
        double newR = computeRatio(a, b, k);  // 需要额外计算
        if (fabs(newR - r) < EPS) break;
        r = newR;
    }
    
    return r;
}
```

### 二分搜索实现

```cpp
bool check(double r, vector<int>& a, vector<int>& b, int k) {
    // 判断 ratio >= r 是否可行
    vector<double> diff(a.size());
    for (int i = 0; i < a.size(); i++) {
        diff[i] = a[i] - r * b[i];
    }
    sort(diff.begin(), diff.end(), greater<double>());
    
    double sum = 0;
    for (int i = 0; i < k; i++) {
        sum += diff[i];
    }
    return sum >= 0;
}

double binarySearch(vector<int>& a, vector<int>& b, int k) {
    double lo = 0, hi = 1e9;
    const int ITER = 60;
    
    for (int it = 0; it < ITER; it++) {
        double mid = (lo + hi) / 2;
        if (check(mid, a, b, k)) {
            lo = mid;
        } else {
            hi = mid;
        }
    }
    
    return lo;
}
```

---

## 典型应用

### 1. 最优比率生成树（Optimal Ratio Spanning Tree）

给定边权 $a_i$（如延迟）和 $b_i$（如费用），求一棵生成树使得：

$$
\min \frac{\sum_{e \in T} a_e}{\sum_{e \in T} b_e}
$$

**解法**：二分 + 最小生成树

```cpp
struct Edge {
    int u, v;
    double a, b;
};

double check(double r, vector<Edge>& edges) {
    // 使用 (a - r*b) 作为边权求 MST
    // 如果总权 < 0，说明 r 可以更大
}

double optimalRatioST(int n, vector<Edge>& edges) {
    double lo = 0, hi = 1e6;
    for (int it = 0; it < 60; it++) {
        double mid = (lo + hi) / 2;
        if (check(mid, edges) < 0) hi = mid;
        else lo = mid;
    }
    return lo;
}
```

### 2. 最优比率环

在图中寻找一个环，使得 $\frac{\sum a_i}{\sum b_i}$ 最小/最大。

### 3. 投资组合优化

给定预期收益 $a_i$ 和风险 $b_i$，选择投资组合使收益/风险比最大。

---

## 算法框架总结

### 二分法（Binary Search）

```
1. 确定答案的上下界 [L, R]
2. while (R - L > EPS):
     mid = (L + R) / 2
     if (可行(mid)):
         L = mid  // 答案可以更大
     else:
         R = mid  // 答案必须更小
3. return L
```

### Dinkelbach 算法

```
1. r0 = 任意可行初始值
2. while true:
     x* = argmax [f(x) - r*g(x)]
     if f(x*) - r*g(x*) == 0: break
     r_{k+1} = f(x*) / g(x*)
3. return r
```

---

## 例题

### 例题：最优比率背包

#### 问题

有 $n$ 件物品，每件物品有价值 $a_i$ 和重量 $b_i$。选择恰好 $k$ 件物品，使得价值/重量比最大。

#### 分析

典型的 0-1 分数规划问题。使用二分搜索转化为判断问题。

#### 代码

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Item {
    int a, b;
};

bool check(double r, const vector<Item>& items, int k) {
    vector<double> diff;
    for (const auto& item : items) {
        diff.push_back(item.a - r * item.b);
    }
    sort(diff.begin(), diff.end(), greater<double>());
    
    double sum = 0;
    for (int i = 0; i < k; i++) {
        sum += diff[i];
    }
    return sum >= 0;
}

double fractionalKnapsack(const vector<Item>& items, int k) {
    double lo = 0, hi = 1e9;
    
    for (int it = 0; it < 60; it++) {
        double mid = (lo + hi) / 2;
        if (check(mid, items, k)) {
            lo = mid;
        } else {
            hi = mid;
        }
    }
    
    return lo;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n, k;
    cin >> n >> k;
    vector<Item> items(n);
    for (int i = 0; i < n; i++) {
        cin >> items[i].a >> items[i].b;
    }
    
    cout << fixed << setprecision(6) << fractionalKnapsack(items, k) << endl;
    return 0;
}
```

---

## 复杂度分析

| 方法 | 时间复杂度 | 适用场景 |
|------|-----------|---------|
| 二分搜索 | $O(\log(\text{precision})) \times O(\text{子问题})$ | 子问题易求解 |
| Dinkelbach | $O(\text{迭代次数}) \times O(\text{子问题})$ | 通常收敛更快 |

---

## 参考资料

[^1]: Fractional Programming - Wikipedia. https://en.wikipedia.org/wiki/Fractional_programming

[^2]: Dinkelbach, W. (1969). On nonlinear fractional programming. Management Science, 13(7), 492-498.

[^3]: 分数规划 - OI Wiki. https://oi-wiki.org/misc/frac-programming/
