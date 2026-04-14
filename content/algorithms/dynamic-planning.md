---
title: 动态规划
date: 2026-04-01
description: 动态规划（Dynamic Programming，DP）算法详解
tags:
  - dynamic-planning
  - dp
draft: false
permalink:
---

## 定义

动态规划（Dynamic Programming，简称 DP）是运筹学的一个分支，是求解决策过程最优化的数学方法。[^1]

动态规划的核心思想是：把原问题分解为若干个子问题，通过求解子问题并存储其答案，避免重复计算，从而获得原问题的最优解。这种"记忆化"的思想使得指数级别的时间复杂度降低为多项式级别。

### 适用条件

动态规划通常适用于满足以下条件的问题：

1. **最优子结构**：问题的最优解可以由其子问题的最优解组合而成
2. **无后效性**：某阶段的状态一旦确定，之后的决策不再受之前状态的影响
3. **子问题重叠**：子问题之间不独立，即存在大量重复计算

### 经典应用场景

- 最长上升子序列
- 背包问题（详见[[../algorithms/knapsack|背包问题]]）
- 最短路劲问题
- 字符串编辑距离
- 矩阵链乘法

---

## 核心概念

动态规划的四大核心要素是：**状态定义**、**状态转移**、**初始化**和**答案提取**。[^2]

### 1. 状态定义

状态是对问题在某个时刻的数学描述。状态定义是 DP 的第一步，也是最关键的一步。

设 $f[i]$ 表示以第 $i$ 个元素结尾的某个最优解，或 $f[i][j]$ 表示考虑前 $i$ 个物品、背包容量为 $j$ 时的最优价值。

状态定义的原则：
- 状态应该包含解决问题所需的所有信息
- 状态的数量应该尽可能少（降低维度）
- 状态转移应该容易计算

### 2. 状态转移

状态转移方程描述了状态之间的关系，即如何从已知状态推导未知状态。

以最长上升子序列（LIS）为例：

$$
f[i] = \max_{j < i, a_j < a_i}(f[j] + 1)
$$

这表示：对于以第 $i$ 个元素结尾的 LIS，其长度等于所有前面比它小的元素中，LIS 长度最大的那个加一。

### 3. 初始化

初始状态是 DP 的边界条件，决定了整个 DP 的起点。

常见的初始化方式：
- $f[0] = 0$ 或 $f[0] = 1$
- $f[i] = 0$ 或 $f[i] = -\infty$（根据问题而定）
- 多维状态需要初始化多个边界

### 4. 答案提取

根据状态定义，答案可能是：
- $f[n]$：最后一个状态
- $\max(f[i])$：所有状态中的最大值
- $f[n][m]$：特定位置的状态值

---

## 线性 DP

线性 DP 是指状态为一维或二维，状态转移呈线性顺序的 DP 类型。

### 最长上升子序列（LIS）

#### 问题描述

给定一个长度为 $n$ 的序列 $a_1, a_2, \dots, a_n$，求最长上升子序列的长度。

上升子序列：$i_1 < i_2 < \dots < i_k$ 且 $a_{i_1} < a_{i_2} < \dots < a_{i_k}$。

#### 解法一：$O(n^2)$ 朴素 DP

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    int n;
    cin >> n;
    vector<int> a(n+1);
    for (int i = 1; i <= n; i++) cin >> a[i];
    
    vector<int> f(n+1, 1);  // f[i] 表示以 a[i] 结尾的 LIS 长度
    int ans = 1;
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j < i; j++) {
            if (a[j] < a[i])
                f[i] = max(f[i], f[j] + 1);
        }
        ans = max(ans, f[i]);
    }
    cout << ans << endl;
    return 0;
}
```

#### 解法二：$O(n \log n)$ 二分优化

利用一个辅助数组 $d$，$d[len]$ 表示长度为 $len$ 的上升子序列的最小结尾元素值。

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    int n;
    cin >> n;
    vector<int> a(n);
    for (int i = 0; i < n; i++) cin >> a[i];
    
    vector<int> d;  // d[len] = 长度为 len 的 LIS 的最小结尾元素
    for (int x : a) {
        auto it = lower_bound(d.begin(), d.end(), x);
        if (it == d.end())
            d.push_back(x);
        else
            *it = x;
    }
    cout << d.size() << endl;
    return 0;
}
```

#### 模板

```cpp
vector<int> d;
for (int x : a) {
    auto it = lower_bound(d.begin(), d.end(), x);  // 改为 upper_bound 则为最长非下降子序列
    if (it == d.end()) d.push_back(x);
    else *it = x;
}
```

---

### 数的划分

#### 问题描述

将整数 $n$ 划分成 $k$ 个正整数之和，求不同的划分方案数。

#### 解法

设 $f[i][j]$ 表示将整数 $i$ 划分成 $j$ 个正整数之和的方案数。

状态转移方程：

$$
f[i][j] = f[i-1][j-1] + f[i-j][j]
$$

- $f[i-1][j-1]$：新划分出的第一个数为 1 的情况
- $f[i-j][j]$：新划分出的第一个数大于 1 的情况（将 $j$ 个数都减 1）

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    int n, k;
    cin >> n >> k;
    vector<vector<int>> f(n+1, vector<int>(k+1, 0));
    
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= min(i, k); j++) {
            if (j == 1 || j == i) {
                f[i][j] = 1;  // 只有一种划分
            } else {
                f[i][j] = f[i-1][j-1] + f[i-j][j];
            }
        }
    }
    cout << f[n][k] << endl;
    return 0;
}
```

---

### 最长公共子序列（LCS）

#### 问题描述

给定两个字符串 $s$ 和 $t$，求它们的最长公共子序列长度。

子序列：不改变相对顺序，在原序列中删除若干元素后得到的序列。

#### 解法

设 $f[i][j]$ 表示 $s[1..i]$ 和 $t[1..j]$ 的最长公共子序列长度。

$$
f[i][j] = \begin{cases} 
f[i-1][j-1] + 1 & s[i] = t[j] \\
\max(f[i-1][j], f[i][j-1]) & s[i] \neq t[j]
\end{cases}
$$

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    string s, t;
    cin >> s >> t;
    int n = s.size(), m = t.size();
    s = " " + s;
    t = " " + t;
    
    vector<vector<int>> f(n+1, vector<int>(m+1, 0));
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= m; j++) {
            if (s[i] == t[j])
                f[i][j] = f[i-1][j-1] + 1;
            else
                f[i][j] = max(f[i-1][j], f[i][j-1]);
        }
    }
    cout << f[n][m] << endl;
    return 0;
}
```

---

## 区间 DP

区间 DP 是指状态由区间定义，状态转移涉及区间的合并。

### 石子合并

#### 问题描述

有 $n$ 堆石子排成一排，每次合并相邻的两堆石子，合并的代价为两堆石子重量之和。求合并所有石子堆的最小代价。

#### 解法

设 $f[l][r]$ 表示合并第 $l$ 堆到第 $r$ 堆石子的最小代价。

状态转移方程：

$$
f[l][r] = \min_{k=l}^{r-1}(f[l][k] + f[k+1][r] + sum[l..r])
$$

其中 $sum[l..r]$ 表示第 $l$ 堆到第 $r$ 堆的石子重量之和。

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    int n;
    cin >> n;
    vector<int> a(n+1);
    for (int i = 1; i <= n; i++) cin >> a[i];
    
    // 前缀和
    vector<int> prefix(n+1, 0);
    for (int i = 1; i <= n; i++) prefix[i] = prefix[i-1] + a[i];
    
    auto sum = [&](int l, int r) {
        return prefix[r] - prefix[l-1];
    };
    
    const int INF = 1e9;
    vector<vector<int>> f(n+2, vector<int>(n+2, 0));
    
    // 区间长度从 2 到 n
    for (int len = 2; len <= n; len++) {
        for (int l = 1; l + len - 1 <= n; l++) {
            int r = l + len - 1;
            f[l][r] = INF;
            for (int k = l; k < r; k++) {
                f[l][r] = min(f[l][r], f[l][k] + f[k+1][r] + sum(l, r));
            }
        }
    }
    cout << f[1][n] << endl;
    return 0;
}
```

#### 模板

```cpp
// 区间 DP 模板
for (int len = 2; len <= n; len++) {           // 枚举区间长度
    for (int l = 1; l + len - 1 <= n; l++) {    // 枚举左端点
        int r = l + len - 1;                     // 右端点
        f[l][r] = INF;
        for (int k = l; k < r; k++) {           // 枚举分割点
            f[l][r] = min(f[l][r], f[l][k] + f[k+1][r] + cost(l, r));
        }
    }
}
```

---

## 树形 DP

树形 DP 是指在树形结构上进行动态规划，通常采用深度优先搜索的方式计算状态。

### 树的直径

#### 问题描述

给定一棵无根树，求树的直径（任意两点间距离的最大值）。

#### 解法

两次 DFS 或树形 DP。

设 $f[u]$ 表示从节点 $u$ 向下走的最长路径长度。则：

$$
f[u] = \max_{v \in son(u)}(f[v] + w(u,v))
$$

树的直径等于 $\max_{u}(f[u] + \text{次长路径})$。

```cpp
#include <bits/stdc++.h>
using namespace std;

int n;
struct Edge {
    int to, w;
};
vector<vector<Edge>> g;

pair<int, int> dfs(int u, int parent) {
    // 返回 {最长路径长度, 对应节点}
    int max1 = 0, max2 = 0;
    int farNode = u;
    
    for (auto &e : g[u]) {
        if (e.to == parent) continue;
        auto [childLen, childNode] = dfs(e.to, u);
        childLen += e.w;
        if (childLen > max1) {
            max2 = max1;
            max1 = childLen;
            farNode = childNode;
        } else if (childLen > max2) {
            max2 = childLen;
        }
    }
    
    // 树的直径 = max1 + max2（如果 max2 存在）
    return {max1, farNode};
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    cin >> n;
    g.resize(n+1);
    for (int i = 1; i < n; i++) {
        int u, v, w;
        cin >> u >> v >> w;
        g[u].push_back({v, w});
        g[v].push_back({u, w});
    }
    
    // 两次 DFS 求直径
    auto [far, node] = dfs(1, 0);
    auto [diameter, tmp] = dfs(node, 0);
    cout << diameter << endl;
    
    return 0;
}
```

---

### 没有上司的舞会

#### 问题描述

某公司有 $n$ 名职员，构成了一个树形结构。每个职员有一个快乐指数。现在要邀请一部分职员参加聚会，但邀请的对象不能包括直接上司和直接下属。求邀请职员的快乐指数之和的最大值。

#### 解法

设 $f[u][0]$ 表示不邀请 $u$ 时，以 $u$ 为根的子树的最大快乐值；$f[u][1]$ 表示邀请 $u$ 时的最大快乐值。

$$
f[u][0] = \sum_{v \in son(u)} \max(f[v][0], f[v][1]) \quad (\text{不邀请 } u)
$$

$$
f[u][1] = \text{happy}[u] + \sum_{v \in son(u)} f[v][0] \quad (\text{邀请 } u)
$$

```cpp
#include <bits/stdc++.h>
using namespace std;

int n;
vector<vector<int>> g;
vector<int> happy;
vector<vector<int64_t>> f;

void dfs(int u, int parent) {
    f[u][1] = happy[u];
    f[u][0] = 0;
    
    for (int v : g[u]) {
        if (v == parent) continue;
        dfs(v, u);
        f[u][0] += max(f[v][0], f[v][1]);
        f[u][1] += f[v][0];
    }
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    cin >> n;
    g.resize(n+1);
    happy.resize(n+1);
    f.assign(n+1, vector<int64_t>(2, 0));
    
    for (int i = 1; i <= n; i++) cin >> happy[i];
    
    vector<int> isRoot(n+1, 1);
    for (int i = 0; i < n-1; i++) {
        int u, v;
        cin >> u >> v;
        g[u].push_back(v);
        g[v].push_back(u);
        isRoot[u] = 0;
    }
    
    int root = 1;
    while (!isRoot[root]) root++;
    dfs(root, 0);
    
    cout << max(f[root][0], f[root][1]) << endl;
    return 0;
}
```

---

## 状态压缩 DP

状态压缩 DP（状压 DP）利用二进制表示状态，将"集合"转化为整数进行 DP。

### 蒙德里安的梦想

#### 问题描述

求用 $1 \times 2$ 的多米诺骨牌填满 $N \times M$ 棋盘的方法数。

#### 解法

设 $f[i][state]$ 表示第 $i$ 行，状态为 $state$ 时的方案数。

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    int n, m;
    while (cin >> n >> m) {
        if (n == 0 && m == 0) break;
        
        int maxState = 1 << m;
        vector<vector<long long>> f(n+1, vector<long long>(maxState, 0));
        vector<vector<int>> valid(maxState);
        
        // 预处理所有合法状态
        for (int state = 0; state < maxState; state++) {
            for (int k = 0; k < maxState; k++) {
                // 检查 state 和 k 是否兼容
                if ((state & k) != 0) continue;  // 不能有冲突
                int rest = state | k;
                bool ok = true;
                // 检查连续的 0 必须是偶数（可以放竖着的骨牌）
                for (int j = 0; j < m; ) {
                    if (rest & (1 << j)) {
                        j++;
                    } else {
                        int cnt = 0;
                        while (j < m && (rest & (1 << j)) == 0) {
                            cnt++;
                            j++;
                        }
                        if (cnt % 2 == 1) {
                            ok = false;
                            break;
                        }
                    }
                }
                if (ok) valid[state].push_back(k);
            }
        }
        
        f[0][0] = 1;
        for (int i = 1; i <= n; i++) {
            for (int state = 0; state < maxState; state++) {
                for (int preState : valid[state]) {
                    f[i][state] += f[i-1][preState];
                }
            }
        }
        
        cout << f[n][0] << endl;
    }
    return 0;
}
```

### 最短 Hamilton 路径

#### 问题描述

给定一张 $n$ 个点的无向图，求从点 $0$ 到点 $n-1$ 的最短 Hamilton 路径。

Hamilton 路径：经过每个点恰好一次。

#### 解法

设 $f[state][i]$ 表示经过 $state$ 中表示的点的集合，当前位于点 $i$ 的最短路径长度。

$$
f[state][i] = \min_{j \in state, j \neq i}(f[state - \{i\}][j] + dist[j][i])
$$

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    int n;
    cin >> n;
    vector<vector<int>> dist(n, vector<int>(n));
    for (int i = 0; i < n; i++)
        for (int j = 0; j < n; j++)
            cin >> dist[i][j];
    
    int maxState = 1 << n;
    vector<vector<int>> f(maxState, vector<int>(n, 1e9));
    f[1][0] = 0;  // 起点：经过点集 {0}，位于点 0
    
    for (int state = 1; state < maxState; state++) {
        for (int i = 0; i < n; i++) {
            if (!(state & (1 << i))) continue;  // i 不在集合中
            if (f[state][i] == 1e9) continue;
            
            for (int j = 0; j < n; j++) {
                if (state & (1 << j)) continue;  // j 已经在集合中
                int nextState = state | (1 << j);
                f[nextState][j] = min(f[nextState][j], f[state][i] + dist[i][j]);
            }
        }
    }
    
    int fullState = (1 << n) - 1;
    cout << f[fullState][n-1] << endl;
    return 0;
}
```

---

## DP 优化技术

### 单调队列优化

当 DP 状态转移方程形如：

$$
f[i] = \min_{j < i}(f[j] + cost(j+1, i))
$$

且 $cost$ 函数具有某种单调性时，可以使用单调队列优化。

#### 经典问题：滑动窗口最小值

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    int n, k;
    cin >> n >> k;
    vector<int> a(n);
    for (int i = 0; i < n; i++) cin >> a[i];
    
    deque<int> q;  // 存储下标
    for (int i = 0; i < n; i++) {
        // 移除超出窗口的元素
        if (!q.empty() && q.front() <= i - k) q.pop_front();
        
        // 维护单调递增队列
        while (!q.empty() && a[q.back()] >= a[i]) q.pop_back();
        q.push_back(i);
        
        // 输出窗口最小值
        if (i >= k - 1) cout << a[q.front()] << " ";
    }
    cout << endl;
    return 0;
}
```

#### 背包优化：多重背包的单调队列优化

多重背包的状态转移：

$$
f[i][j] = \max_{0 \leq t \leq k_i}(f[i-1][j - t \cdot w_i] + t \cdot v_i)
$$

其中 $k_i = \lfloor j / w_i \rfloor$。

将 $j$ 按 $w_i$ 取模分类，对每类使用单调队列优化，时间复杂度降为 $O(n \times W)$。[^3]

---

### 斜率优化

当 DP 状态转移方程形如：

$$
f[i] = \min_{j < i}(f[j] + (X_i - X_j) \cdot Y_j)
$$

可以利用凸包维护所有决策点 $(X_j, Y_j)$，在 $O(n)$ 或 $O(n \log n)$ 时间内求解。

设 $dp_j = f[j] + X_j \cdot Y_j - X_j \cdot X_i$，将其改写为截距形式：

$$
f[i] = X_i \cdot Y_j + (dp_j - X_j \cdot Y_j)
$$

维护一个下凸包即可。

#### 玩具装箱（TOYS）

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    int n;
    long long L;
    cin >> n >> L;
    vector<long long> x(n+1), dp(n+1);
    for (int i = 1; i <= n; i++) cin >> x[i];
    
    for (int i = 1; i <= n; i++) x[i] += x[i-1];
    
    auto cross = [&](int i, int j, int k) {
        // 判断 (j,i,k) 是否构成上凸
        long long yj = dp[j] - x[j] * x[j] + L * x[j];
        long long yi = dp[i] - x[i] * x[i] + L * x[i];
        long long yk = dp[k] - x[k] * x[k] + L * x[k];
        return (yi - yj) * (xk - xj) >= (yk - yj) * (xi - xj);
    };
    
    deque<int> q;
    q.push_back(0);
    
    for (int i = 1; i <= n; i++) {
        while (q.size() >= 2) {
            int j = q[0], k = q[1];
            long long val_j = dp[j] + (x[i] - x[j] + L) * (x[i] - x[j] + L);
            long long val_k = dp[k] + (x[i] - x[k] + L) * (x[i] - x[k] + L);
            if (val_j >= val_k) q.pop_front();
            else break;
        }
        
        int j = q.front();
        dp[i] = dp[j] + (x[i] - x[j] + L) * (x[i] - x[j] + L);
        
        while (q.size() >= 2 && cross(q[q.size()-2], q[q.size()-1], i))
            q.pop_back();
        q.push_back(i);
    }
    
    cout << dp[n] << endl;
    return 0;
}
```

---

### 矩阵快速幂优化

当 DP 状态转移是线性齐次递推式时，可以使用矩阵快速幂在 $O(n^3 \log k)$ 时间内求解。

#### 斐波那契数列

$$
\begin{pmatrix} f_n \\ f_{n-1} \end{pmatrix} = 
\begin{pmatrix} 1 & 1 \\ 1 & 0 \end{pmatrix}^{n-1} \cdot 
\begin{pmatrix} f_1 \\ f_0 \end{pmatrix}
$$

```cpp
#include <bits/stdc++.h>
using namespace std;

using Matrix = vector<vector<long long>>;

Matrix mul(const Matrix &A, const Matrix &B) {
    int n = A.size();
    Matrix C(n, vector<long long>(n, 0));
    for (int i = 0; i < n; i++)
        for (int k = 0; k < n; k++)
            if (A[i][k])
                for (int j = 0; j < n; j++)
                    C[i][j] = (C[i][j] + A[i][k] * B[k][j]) % MOD;
    return C;
}

Matrix mpow(Matrix A, long long k) {
    int n = A.size();
    Matrix R(n, vector<long long>(n, 0));
    for (int i = 0; i < n; i++) R[i][i] = 1;
    while (k) {
        if (k & 1) R = mul(R, A);
        A = mul(A, A);
        k >>= 1;
    }
    return R;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    long long n;
    const long long MOD = 1e9+7;
    cin >> n;
    
    if (n <= 1) {
        cout << n << endl;
        return 0;
    }
    
    Matrix M = {{1, 1}, {1, 0}};
    Matrix R = mpow(M, n - 1);
    cout << (R[0][0] + R[0][1]) % MOD << endl;
    return 0;
}
```

---

## 问题解决框架

### DP 解题步骤

1. **分析问题**：判断是否适合 DP（最优子结构、无后效性、子问题重叠）
2. **状态定义**：确定状态的维度和含义
3. **状态转移**：推导状态转移方程
4. **初始化**：确定边界条件
5. **答案提取**：确定输出格式
6. **复杂度分析**：分析时间和空间复杂度
7. **代码实现**：按照正确顺序实现

### 状态设计技巧

| 技巧 | 适用场景 |
|------|----------|
| 升维 | 需要记录更多信息时 |
| 降维 | 空间优化，与前一个状态有关 |
| 坐标压缩 | 状态范围大但实际用到的不多 |
| 记忆化搜索 | 状态转移顺序不明确时 |

### 常见 DP 类型

| 类型 | 特征 | 经典问题 |
|------|------|----------|
| 线性 DP | 一维/二维状态，顺序转移 | LIS、LCS、数的划分 |
| 区间 DP | 状态为区间，常用合并操作 | 石子合并、括号序列 |
| 树形 DP | 在树上做 DP，DFS 计算 | 树的直径、树上最大独立集 |
| 状压 DP | 状态用二进制表示 | 旅行商、棋盘覆盖 |
| 数位 DP | 按数位进行 DP | 数字统计 |
| 概率 DP | 期望值计算 | 随机游走 |

---

## 经典例题

### 例题一：最大子段和

#### 问题

给定数组 $a_1, a_2, \dots, a_n$，求最大连续子段和。

#### 分析

设 $f[i]$ 表示以第 $i$ 个元素结尾的最大子段和。

$$
f[i] = \max(a[i], f[i-1] + a[i])
$$

答案：$\max(f[i])$

#### 代码

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    int n;
    cin >> n;
    vector<long long> a(n+1);
    for (int i = 1; i <= n; i++) cin >> a[i];
    
    vector<long long> f(n+1, 0);
    f[1] = a[1];
    long long ans = f[1];
    for (int i = 2; i <= n; i++) {
        f[i] = max(a[i], f[i-1] + a[i]);
        ans = max(ans, f[i]);
    }
    cout << ans << endl;
    return 0;
}
```

---

### 例题二：编辑距离

#### 问题

给定两个字符串 $s$ 和 $t$，可以对 $s$ 进行插入、删除、替换操作，求将 $s$ 变成 $t$ 的最小操作数。

#### 分析

设 $f[i][j]$ 表示 $s[1..i]$ 变成 $t[1..j]$ 的最小操作数。

$$
f[i][j] = \min\begin{cases}
f[i-1][j] + 1 & \text{（删除 } s[i]）\\
f[i][j-1] + 1 & \text{（插入 } t[j]）\\
f[i-1][j-1] + (s[i] \neq t[j]) & \text{（替换或相同）}
\end{cases}
$$

#### 代码

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    string s, t;
    cin >> s >> t;
    int n = s.size(), m = t.size();
    s = " " + s;
    t = " " + t;
    
    vector<vector<int>> f(n+1, vector<int>(m+1, 0));
    for (int i = 0; i <= n; i++) f[i][0] = i;
    for (int j = 0; j <= m; j++) f[0][j] = j;
    
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= m; j++) {
            f[i][j] = min({f[i-1][j] + 1, f[i][j-1] + 1, 
                          f[i-1][j-1] + (s[i] != t[j])});
        }
    }
    cout << f[n][m] << endl;
    return 0;
}
```

---

### 例题三：传纸条

#### 问题

在一个 $m \times n$ 的网格中，从 $(1,1)$ 出发到 $(m,n)$，只能向右或向下走，求两条路径使所经格子的数字之和最大，且两条路径不能经过同一个格子。

#### 分析

从 $(1,1)$ 到 $(m,n)$ 的两条路径，等价于从两端同时出发的两条路径。

设 $f[i][j][k]$ 表示第一条路径走到 $(i,j)$，第二条路径走到 $(k, i+j-k)$ 时的最大和。

状态转移考虑四种情况（两条路径的上一步）。

#### 代码

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    int m, n;
    cin >> m >> n;
    vector<vector<int>> a(m+1, vector<int>(n+1));
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            cin >> a[i][j];
    
    vector<vector<vector<int>>> f(m+1, vector<vector<int>>(n+1, vector<int>(m+1, 0)));
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            for (int k = 1; k <= m; k++) {
                int l = i + j - k;  // 第二条路径的列
                if (l < 1 || l > n) continue;
                
                int &cur = f[i][j][k];
                int val = a[i][j] + a[k][l];
                if (i == k && j == l) val -= a[i][j];  // 重复格子只加一次
                
                cur = max(cur, f[i-1][j][k-1]);  // 两条都从上方
                cur = max(cur, f[i-1][j][k]);    // 第一条上方，第二条左方
                cur = max(cur, f[i][j-1][k-1]);  // 第一条左方，第二条上方
                cur = max(cur, f[i][j-1][k]);    // 两条都从左方
                cur += val;
            }
        }
    }
    
    cout << f[m][n][m] << endl;
    return 0;
}
```

---

## 总结

动态规划是算法竞赛中最重要的内容之一，掌握动态规划需要：

1. **理解核心思想**：记忆化搜索、最优子结构、无后效性
2. **熟练四大要素**：状态定义、状态转移、初始化、答案提取
3. **掌握经典模型**：线性 DP、区间 DP、树形 DP、状压 DP
4. **学会优化技巧**：单调队列、斜率优化、矩阵快速幂
5. **多做练习**：从经典问题入手，逐步进阶

在学习过程中，建议先掌握背包问题（详见[[../algorithms/knapsack|背包问题]]）和线性 DP，再学习更复杂的 DP 类型。

---

## 参考资料

[^1]: 动态规划 - OI Wiki. https://oi-wiki.org/dp/

[^2]: 算法竞赛入门经典（罗永军）—— 动态规划部分

[^3]: 背包问题九讲 - 崔添翼. https://github.com/tianyicui/pack
