---
title: 2-SAT
date: 2026-04-05
description: 2-SAT（2元布尔可满足性问题）算法详解
tags:
  - 2-sat
  - graph
  - algorithm
draft: false
permalink:
---

## 定义

2-SAT（2-Satisfiability）是布尔可满足性问题的一个子集，指的是在合取范式（CNF）中，每个子句恰好包含两个文字（literal）的布尔公式。[^1]

例如，以下就是一个典型的2-SAT实例：

$$
(a \lor \neg b) \land (\neg a \lor b) \land (\neg a \lor \neg b) \land (a \lor \neg c)
$$

其中每个括号内的表达式称为一个**子句**（clause），每个文字可以是变量（如 $a, b, c$）或其否定（如 $\neg a, \neg b$）。

## 核心思想

### 蕴含范式

2-SAT问题的核心技巧是将每个子句转化为**蕴含式**（implication）。对于任意子句 $a \lor b$，可以等价地表示为：

$$
a \lor b \equiv \neg a \Rightarrow b \land \neg b \Rightarrow a
$$

这个转化揭示了2-SAT的本质：**如果 $a$ 为假，则 $b$ 必须为真；如果 $b$ 为假，则 $a$ 必须为真**。

### 蕴含图

基于蕴含关系，我们可以构建一张**蕴含图**（implication graph）：

- 图中有 $2n$ 个顶点（$n$ 个变量，每个变量有正负两个状态）
- 每条边 $u \Rightarrow v$ 表示"若 $u$ 为真，则 $v$ 必须为真"
- 关键性质：若存在边 $a \Rightarrow b$，则必然存在边 $\neg b \Rightarrow \neg a$

## 算法步骤

2-SAT问题的求解算法如下：

1. **构建蕴含图**：根据所有子句，添加相应的有向边
2. **求强连通分量**：使用 Kosaraju 算法或 Tarjan 算法求出所有 SCC
3. **判定可行性**：若存在某个变量 $x$，使得 $x$ 和 $\neg x$ 位于同一个 SCC，则公式不可满足
4. **拓扑排序求赋值**：对 SCC 按拓扑序逆序进行赋值

## 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

struct TwoSatSolver {
    int n;                              // 变量个数
    vector<vector<int>> g, rg;          // 蕴含图和逆图
    vector<int> comp, order, assignment; // SCC组件、拓扑序、赋值
    
    TwoSatSolver(int n) : n(n) {
        g.assign(2 * n, {});
        rg.assign(2 * n, {});
        comp.assign(2 * n, -1);
        assignment.assign(n, 0);
    }
    
    // 变量x的两个状态: x和x^n (取反)
    inline int var(int x, bool val) { return x * 2 + (val ? 1 : 0); }
    inline int neg(int v) { return v ^ 1; }
    
    // 添加子句 (a OR b)，a和b是文字的索引
    // lit: 2*x表示x为假，2*x+1表示x为真
    void add_disjunction(int a, int na, int b, int nb) {
        // (a OR b) 等价于 (na => a) AND (nb => b)
        g[na].push_back(a);
        rg[a].push_back(na);
        g[nb].push_back(b);
        rg[b].push_back(nb);
    }
    
    // Kosaraju算法求SCC
    void dfs1(int v) {
        comp[v] = -2;
        for (int to : g[v]) {
            if (comp[to] == -1) dfs1(to);
        }
        order.push_back(v);
    }
    
    void dfs2(int v, int cl) {
        comp[v] = cl;
        for (int to : rg[v]) {
            if (comp[to] == -1) dfs2(to, cl);
        }
    }
    
    bool solve_2SAT() {
        // 第一遍DFS求拓扑序
        for (int i = 0; i < 2 * n; i++) {
            if (comp[i] == -1) dfs1(i);
        }
        reverse(order.begin(), order.end());
        
        // 第二遍DFS按逆拓扑序标记SCC
        fill(comp.begin(), comp.end(), -1);
        int j = 0;
        for (int v : order) {
            if (comp[v] == -1) dfs2(v, j++);
        }
        
        // 检查同一变量的两个状态是否在同一SCC
        assignment.assign(n, 0);
        for (int i = 0; i < n; i++) {
            if (comp[2 * i] == comp[2 * i + 1]) return false; // 矛盾
            assignment[i] = comp[2 * i] < comp[2 * i + 1] ? 0 : 1;
        }
        return true;
    }
};
```

### 使用示例

```cpp
int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n = 3;  // 3个变量: x0, x1, x2
    TwoSatSolver solver(n);
    
    // 添加子句: (a OR b)
    // 例如: (x0 OR x1)
    solver.add_disjunction(solver.var(0, true), solver.var(0, false),
                           solver.var(1, true), solver.var(1, false));
    
    if (solver.solve_2SAT()) {
        cout << "SATISFIABLE\n";
        for (int i = 0; i < n; i++) {
            cout << "x" << i << " = " << solver.assignment[i] << "\n";
        }
    } else {
        cout << "UNSATISFIABLE\n";
    }
    
    return 0;
}
```

## 典型应用场景

### 布尔方程组求解

给定一系列形如 $(x_i \lor x_j)$ 或 $(\neg x_i \lor x_j)$ 的约束，求解满足所有约束的布尔变量赋值。

### 条件约束问题

在许多实际问题中，约束可以转化为2-SAT形式：

- 选课问题：课程A和课程B至少选一个，但两门课时间冲突不能同时选
- 任务分配问题：每个任务只能分配给两个候选人中的一个
- 布尔电路问题：验证电路是否可满足

### 选择问题

当每个变量只能在两个选项中选择时（常见于二元选择场景），2-SAT提供了高效求解方法。

## 例题

### 洛谷 P4782 【模板】2-SAT 问题

**题目描述**：有 $n$ 个布尔变量，给定 $m$ 个条件，每个条件形如 $(x_i$ 为 $a$ 或 $x_j$ 为 $b)$，判断是否存在一组赋值使得所有条件同时满足。

**解题思路**：

1. 将每个变量 $x_i$ 拆分为两个状态：$x_i$ 为真和 $x_i$ 为假
2. 对于条件"($x_i$ 为 $a$ 或 $x_j$ 为 $b$)"，添加两条蕴含边：
   - $\neg (x_i$ 为 $a) \Rightarrow (x_j$ 为 $b)$
   - $\neg (x_j$ 为 $b) \Rightarrow (x_i$ 为 $a)$
3. 求SCC，检查矛盾
4. 按拓扑序逆序进行赋值

**参考实现**：

```cpp
#include <bits/stdc++.h>
using namespace std;

struct TwoSatSolver {
    int n;
    vector<vector<int>> g, rg;
    vector<int> comp, order, assignment;
    
    TwoSatSolver(int n) : n(n) {
        g.assign(2 * n, {});
        rg.assign(2 * n, {});
        comp.assign(2 * n, -1);
        assignment.assign(n, 0);
    }
    
    inline int var(int x, bool val) { return x * 2 + (val ? 1 : 0); }
    inline int neg(int v) { return v ^ 1; }
    
    void add_or(int x, bool xv, int y, bool yv) {
        // 添加 (x == xv) OR (y == yv)
        add_disjunction(var(x, xv), var(x, !xv), var(y, yv), var(y, !yv));
    }
    
    void add_disjunction(int a, int na, int b, int nb) {
        g[na].push_back(a);
        rg[a].push_back(na);
        g[nb].push_back(b);
        rg[b].push_back(nb);
    }
    
    void add_xor(int x, bool xv, int y, bool yv) {
        // (x == xv) XOR (y == yv) 等价于两个子句
        add_or(x, xv, y, !yv);
        add_or(x, !xv, y, yv);
    }
    
    void dfs1(int v) {
        comp[v] = -2;
        for (int to : g[v]) {
            if (comp[to] == -1) dfs1(to);
        }
        order.push_back(v);
    }
    
    void dfs2(int v, int cl) {
        comp[v] = cl;
        for (int to : rg[v]) {
            if (comp[to] == -1) dfs2(to, cl);
        }
    }
    
    bool solve() {
        for (int i = 0; i < 2 * n; i++) {
            if (comp[i] == -1) dfs1(i);
        }
        reverse(order.begin(), order.end());
        
        fill(comp.begin(), comp.end(), -1);
        int j = 0;
        for (int v : order) {
            if (comp[v] == -1) dfs2(v, j++);
        }
        
        for (int i = 0; i < n; i++) {
            if (comp[2 * i] == comp[2 * i + 1]) return false;
            assignment[i] = comp[2 * i] < comp[2 * i + 1] ? 0 : 1;
        }
        return true;
    }
};

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n, m;
    cin >> n >> m;
    TwoSatSolver solver(n);
    
    for (int i = 0; i < m; i++) {
        int xi, xj, ai, aj;
        cin >> xi >> ai >> xj >> aj;
        xi--; xj--;  // 转为0-indexed
        solver.add_or(xi, ai, xj, aj);
    }
    
    if (solver.solve()) {
        cout << "POSSIBLE\n";
        for (int i = 0; i < n; i++) {
            cout << solver.assignment[i] << " ";
        }
        cout << "\n";
    } else {
        cout << "IMPOSSIBLE\n";
    }
    
    return 0;
}
```

## 相关内容

- [[../graph/strongly-connected-components|强连通分量（SCC）]]：2-SAT算法的核心基础
- [[../graph/topological-sort|拓扑排序]]：用于求解赋值顺序
- [[breadth-first-search|广度优先搜索]]：图遍历基础
- [[depth-first-search|深度优先搜索]]：Kosaraju算法的关键步骤

## 参考资料

[^1]: 本篇内容参考 [cp-algorithms.com - 2-SAT](https://cp-algorithms.com/graph/2SAT.html)
