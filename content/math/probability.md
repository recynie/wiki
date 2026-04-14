---
title: 概率与期望
date: 2026-04-05
description: 概率动态规划、期望计算及相关算法
tags:
  - probability
  - expectation
  - dynamic-programming
  - math
draft: false
permalink:
---

## 基础概念

### 概率定义与基本公式

设样本空间为 $\Omega$，事件 $A$ 的概率记为 $P(A)$，满足：

- $0 \leqslant P(A) \leqslant 1$
- $P(\Omega) = 1$
- 若 $A_1, A_2, \ldots, A_n$ 互不相容，则 $P\left(\bigcup_{i=1}^n A_i\right) = \sum_{i=1}^n P(A_i)$

**独立事件**：若 $P(AB) = P(A)P(B)$，则称事件 $A$ 与 $B$ 独立。

### 期望的线性性质

期望满足线性性质，这是概率 DP 的核心基础：

$$
E(X + Y) = E(X) + E(Y)
$$

注意：对于独立事件的乘积有 $E(XY) = E(X) \cdot E(Y)$，但这一性质在大多数概率 DP 问题中不直接使用。[^1]

### 条件概率与贝叶斯公式

**条件概率**：在事件 $B$ 已发生的条件下，事件 $A$ 发生的概率

$$
P(A \mid B) = \frac{P(AB)}{P(B)}
$$

**贝叶斯公式**：

$$
P(B \mid A) = \frac{P(A \mid B) \cdot P(B)}{P(A)}
$$

更多内容参见 [[../mathematical-statistics/bayesian-inference|贝叶斯推断]]。

## 概率 DP

概率 DP 是指在动态规划中引入概率和期望，用于求解在随机过程中达到某种状态的概率或期望值。

### 顺推法（求概率）

顺推法从初始状态出发，按照状态转移逐步计算到达目标状态的概率。

**适用场景**：已知初始状态概率分布，求解到达目标状态的概率。

设 $dp[i]$ 表示经过 $i$ 步后到达目标状态的概率，转移方程为：

$$
dp[i+1][j] = \sum_{k} dp[i][k] \cdot P(k \to j)
$$

### 逆推法（求期望）

逆推法从目标状态反推回初始状态，利用期望的线性性质构建方程。

**适用场景**：求从初始状态到达目标状态的期望步数或期望代价。

设 $E[i]$ 表示从状态 $i$ 到达目标状态的期望步数，则：

$$
E[i] = 1 + \sum_{j} P(i \to j) \cdot E[j]
$$

其中 $1$ 表示当前这一步。

### 高斯消元法

当状态转移存在环（即后效性）时，逆推法构建的方程组可能包含多个未知数，需要使用高斯消元求解。

**适用场景**：状态转移图存在环，无法直接用 DAG 上的 DP 求解。

设期望向量为 $\mathbf{E}$，转移矩阵为 $\mathbf{P}$，则：

$$
\mathbf{E} = \mathbf{1} + \mathbf{P} \cdot \mathbf{E}
$$

整理得：

$$
(\mathbf{I} - \mathbf{P}) \cdot \mathbf{E} = \mathbf{1}
$$

## 典型问题类型

### 到达目标的期望步数

常见于游走类问题，如掷骰子、移动棋子等。

**问题模式**：从起点出发，每次随机移动，求首次到达目标的期望步数。

**解法**：逆推法 + 方程组建立 + 高斯消元

### 期望收益最大化

在随机环境中做决策，使得期望收益最大化。

**问题模式**：每一步有多种选择，每种选择有对应的概率分布和收益，求最大期望收益。

**解法**：概率 DP 求得所有状态的期望收益后取最大值

### 概率最大化决策

在不确定环境下做出决策，使得达到目标的概率最大。

**问题模式**：与期望收益最大化类似，但目标是概率而非收益。

## 代码实现

### 基础概率 DP 模板

以下模板展示顺推法求解概率问题：

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n, m;
    cin >> n >> m;  // n: 状态数, m: 转移方式数
    vector<vector<pair<int, double>>> g(n);
    
    // 构建转移图：g[i] = {(next_state, probability), ...}
    for (int i = 0; i < m; i++) {
        int u, v;
        double p;
        cin >> u >> v >> p;
        g[u].push_back({v, p});
    }
    
    vector<double> dp(n, 0.0);
    dp[0] = 1.0;  // 初始状态概率为 1
    
    for (int i = 0; i < n; i++) {
        for (auto [v, p] : g[i]) {
            dp[v] += dp[i] * p;
        }
    }
    
    // 输出到达各状态的概率
    for (int i = 0; i < n; i++) {
        cout << "State " << i << ": " << dp[i] << "\n";
    }
    
    return 0;
}
```

### 期望 DP 与方程组

以下模板展示使用高斯消元求解期望 DP：

```cpp
#include <bits/stdc++.h>
using namespace std;

const double EPS = 1e-8;

int gauss(vector<vector<double>>& a, vector<double>& ans) {
    int n = a.size();
    int m = a[0].size() - 1;
    
    for (int col = 0, row = 0; col < m && row < n; col++, row++) {
        // 选主元
        int max_row = row;
        for (int i = row; i < n; i++) {
            if (fabs(a[i][col]) > fabs(a[max_row][col]))
                max_row = i;
        }
        if (fabs(a[max_row][col]) < EPS) return 0;  // 无解
        
        swap(a[row], a[max_row]);
        
        // 消元
        for (int i = 0; i < n; i++) {
            if (i != row && fabs(a[i][col]) > EPS) {
                double factor = a[i][col] / a[row][col];
                for (int j = col; j <= m; j++) {
                    a[i][j] -= factor * a[row][j];
                }
            }
        }
    }
    
    // 回代
    for (int i = 0; i < m; i++) {
        ans[i] = a[i][m] / a[i][i];
    }
    return 1;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n;  // 状态数
    cin >> n;
    vector<vector<pair<int, double>>> g(n);
    vector<int> out_degree(n, 0);
    
    // 构建转移图
    for (int i = 0; i < n; i++) {
        int k;
        cin >> k;
        out_degree[i] = k;
        for (int j = 0; j < k; j++) {
            int v;
            double p;
            cin >> v >> p;
            g[i].push_back({v, p});
        }
    }
    
    // 建立方程组: E[i] = 1 + sum(P[i->j] * E[j])
    // 移项得: E[i] - sum(P[i->j] * E[j]) = 1
    vector<vector<double>> a(n, vector<double>(n + 1, 0.0));
    
    for (int i = 0; i < n; i++) {
        a[i][i] = 1.0;
        a[i][n] = 1.0;  // 常数项
        for (auto [v, p] : g[i]) {
            a[i][v] -= p;
        }
    }
    
    vector<double> ans(n);
    if (gauss(a, ans)) {
        cout << "期望值:\n";
        for (int i = 0; i < n; i++) {
            cout << "State " << i << ": " << ans[i] << "\n";
        }
    }
    
    return 0;
}
```

## 例题

### 骰子期望问题

**问题**：掷一枚均匀六面骰子，求首次掷出 6 的期望掷骰次数。

**分析**：设期望为 $E$。第一次掷骰：
- 若掷出 6（概率 $1/6$），结束，步数为 1
- 若未掷出 6（概率 $5/6$），步数为 $1 + E$

列方程：
$$
E = \frac{1}{6} \cdot 1 + \frac{5}{6} \cdot (1 + E)
$$
解得 $E = 6$。

### 游走类问题

**问题**：在一维数轴上，从位置 0 出发，每次等概率向左或向右走一步，求首次到达位置 $n$ 的期望步数。

**分析**：设从位置 $i$ 出发的期望为 $E_i$，有：

$$
E_i = 1 + \frac{1}{2}E_{i-1} + \frac{1}{2}E_{i+1}
$$

边界条件：$E_0 = 0$，$E_n = 0$（假设 $n$ 为目标）。

解此方程组得 $E_i = i(n-i)$。

### 状态压缩概率 DP

**问题**：给定 $n$ 枚硬币，每枚硬币正面朝上的概率为 $p_i$，求恰好有 $k$ 枚正面朝上的概率。

**分析**：设 $dp[mask]$ 表示状态 $mask$ 下正面朝上的概率，状态转移：

$$
dp[mask \cup \{i\}] += dp[mask] \cdot p_i
$$
$$
dp[mask] *= (1 - p_i)  \text{（不选第 } i \text{ 枚）}
$$

复杂度 $O(n \cdot 2^n)$。

## 参考资料

[^1]: 本段内容来自 [概率期望 DP - OI Wiki](https://oi-wiki.org/dp/probability/)
