---
title: 深度优先搜索 (depth-first-search)
date: 2026-03-13
description: 深度优先搜索（Depth-First Search, DFS）是一种图的遍历算法
tags:
  - dfs
  - graph
draft: false
permalink:
---
# 深度优先搜索（DFS）

深度优先搜索（Depth-First Search, DFS）和[[breadth-first-search|BFS]]是图搜索中最基础的两种算法。DFS 的基本思想是尽可能「深」地搜索一个分支，直到该分支遍历完毕，再回溯到上一个节点继续搜索其他分支。这种「不撞南墙不回头」的特性使得 DFS 在许多场景下非常有用。[^1]

[^1]: 本段来自[DFS（图论） - OI Wiki](https://oi-wiki.org/graph/dfs)

DFS 的核心特点：
- **优先深度**：沿着一条路径一直向下，知道无法继续才回溯
- **栈（递归）驱动**：可以用递归实现（系统栈），也可以用显式栈模拟
- **回溯**：搜索过程中需要「撤销」选择，回到之前的状态

# 算法模板

## 递归实现

递归实现是 DFS 最自然的方式，充分利用了系统调用栈。

```cpp
#include <bits/stdc++.h>
using namespace std;

vector<vector<int>> g;      // 图的邻接表
vector<int> visited;        // 访问标记

void dfs(int u) {
    visited[u] = 1;
    cout << u << " ";  // 访问节点（可替换为具体操作）

    for (int v : g[u]) {
        if (!visited[v]) {
            dfs(v);  // 递归深入
        }
    }
    // 无需显式回溯，递归返回自动完成
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    int n, m;  // n: 节点数, m: 边数
    cin >> n >> m;

    g.assign(n, {});
    visited.assign(n, 0);

    for (int i = 0; i < m; ++i) {
        int u, v;
        cin >> u >> v;
        g[u].push_back(v);
        g[v].push_back(u);  // 无向图
    }

    // 遍历所有连通分量
    for (int i = 0; i < n; ++i) {
        if (!visited[i]) {
            dfs(i);
        }
    }
}
```

## 显式栈实现（迭代）

有时候递归深度过深会导致栈溢出，此时可以使用显式栈模拟递归过程。

```cpp
#include <bits/stdc++.h>
using namespace std;

void dfs_iterative(int start, const vector<vector<int>>& g) {
    vector<int> visited(g.size(), 0);
    stack<int> st;
    st.push(start);
    visited[start] = 1;

    while (!st.empty()) {
        int u = st.top();
        st.pop();
        cout << u << " ";

        for (int v : g[u]) {
            if (!visited[v]) {
                visited[v] = 1;
                st.push(v);
            }
        }
    }
}
```

注意：上述迭代实现**不等同于**递归版本的遍历顺序。栈的 LIFO 特性导致节点的访问顺序与递归版本相反。如果需要保持与递归相同的顺序，可以在压栈时调整：

```cpp
void dfs_iterative_correct_order(int start, const vector<vector<int>>& g) {
    vector<int> visited(g.size(), 0);
    stack<pair<int, int>> st;  // {节点, 下一个要访问的邻居索引}
    st.push({start, 0});
    visited[start] = 1;

    while (!st.empty()) {
        auto& [u, idx] = st.top();
        if (idx < (int)g[u].size()) {
            int v = g[u][idx++];
            if (!visited[v]) {
                visited[v] = 1;
                st.push({v, 0});
            }
        } else {
            st.pop();
        }
    }
}
```

# 网格搜索

网格上的 DFS 与图类似，需要定义**后继关系**（相邻格子）和**边界判断**。

```cpp
#include <bits/stdc++.h>
using namespace std;

int n, m;
vector<vector<int>> grid;
vector<vector<int>> vis;

int dx[4] = {1, -1, 0, 0};
int dy[4] = {0, 0, 1, -1};

bool in_grid(int x, int y) {
    return x >= 0 && x < n && y >= 0 && y < m;
}

void dfs_grid(int x, int y) {
    vis[x][y] = 1;
    cout << "(" << x << ", " << y << ") ";

    for (int dir = 0; dir < 4; ++dir) {
        int nx = x + dx[dir];
        int ny = y + dy[dir];
        if (in_grid(nx, ny) && !vis[nx][ny] && grid[nx][ny] != '#') {
            dfs_grid(nx, ny);
        }
    }
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    cin >> n >> m;
    grid.assign(n, vector<int>(m));
    vis.assign(n, vector<int>(m, 0));

    for (int i = 0; i < n; ++i) {
        for (int j = 0; j < m; ++j) {
            cin >> grid[i][j];
        }
    }

    int sx, sy;
    cin >> sx >> sy;
    dfs_grid(sx, sy);
}
```

## 网格 DFS 的应用

### 连通块数量

统计网格中所有「岛屿」（相连的 1）的数量：

```cpp
void dfs_count(int x, int y, vector<vector<char>>& grid, vector<vector<int>>& vis) {
    int n = grid.size(), m = grid[0].size();
    vis[x][y] = 1;

    for (int dir = 0; dir < 4; ++dir) {
        int nx = x + dx[dir];
        int ny = y + dy[dir];
        if (nx >= 0 && nx < n && ny >= 0 && ny < m 
            && !vis[nx][ny] && grid[nx][ny] == '1') {
            dfs_count(nx, ny, grid, vis);
        }
    }
}

int num_islands(vector<vector<char>>& grid) {
    if (grid.empty()) return 0;
    int n = grid.size(), m = grid[0].size();
    vector<vector<int>> vis(n, vector<int>(m, 0));
    int count = 0;

    for (int i = 0; i < n; ++i) {
        for (int j = 0; j < m; ++j) {
            if (!vis[i][j] && grid[i][j] == '1') {
                ++count;
                dfs_count(i, j, grid, vis);
            }
        }
    }
    return count;
}
```

# 图的 DFS

## 有向图的 DFS

有向图的 DFS 与无向图类似，但只沿有向边前进：

```cpp
vector<vector<int>> g;      // 邻接表
vector<int> visited;

void dfs_directed(int u) {
    visited[u] = 1;
    cout << u << " ";

    for (int v : g[u]) {
        if (!visited[v]) {
            dfs_directed(v);
        }
    }
}
```

## 连通分量

利用 DFS 可以找出图的所有连通分量：

```cpp
int count_connected_components(int n, const vector<vector<int>>& g) {
    vector<int> visited(n, 0);
    int components = 0;

    function<void(int)> dfs = [&](int u) {
        visited[u] = 1;
        for (int v : g[u]) {
            if (!visited[v]) {
                dfs(v);
            }
        }
    };

    for (int i = 0; i < n; ++i) {
        if (!visited[i]) {
            ++components;
            dfs(i);
        }
    }
    return components;
}
```

# 遍历顺序

DFS 有两种主要的遍历顺序：

## 先序（Preorder）

**在递归调用之前**访问节点：先处理当前节点，再递归处理子节点。

```cpp
void preorder(int u) {
    if (u == -1) return;
    cout << u << " ";      // 先访问（先序）
    preorder(u->left);
    preorder(u->right);
}
```

## 后序（Postorder）

**在递归调用之后**访问节点：先递归处理子节点，再处理当前节点。

```cpp
void postorder(int u) {
    if (u == -1) return;
    postorder(u->left);
    postorder(u->right);
    cout << u << " ";      // 后访问（后序）
}
```

树的遍历示例：

```
       1
      / \
     2   3
    / \
   4   5

先序遍历: 1 2 4 5 3
后序遍历: 4 5 2 3 1
```

# 回溯法

回溯是 DFS 的高级应用，在搜索过程中**尝试**选择，如果到达死胡同就**撤销**选择（回溯）。

## 全排列问题

```cpp
vector<vector<int>> result;
vector<int> path;
vector<int> used;

void backtrack(vector<int>& nums) {
    if (path.size() == nums.size()) {
        result.push_back(path);
        return;
    }

    for (int i = 0; i < (int)nums.size(); ++i) {
        if (used[i]) continue;

        path.push_back(nums[i]);
        used[i] = 1;
        backtrack(nums);  // 递归深入

        path.pop_back();  // 撤销选择
        used[i] = 0;
    }
}

vector<vector<int>> permute(vector<int>& nums) {
    used.assign(nums.size(), 0);
    backtrack(nums);
    return result;
}
```

## 子集问题

```cpp
vector<vector<int>> result;
vector<int> path;

void backtrack_subsets(const vector<int>& nums, int start) {
    result.push_back(path);  // 每个节点都是一个子集

    for (int i = start; i < (int)nums.size(); ++i) {
        path.push_back(nums[i]);
        backtrack_subsets(nums, i + 1);  // 递归
        path.pop_back();                  // 撤销
    }
}

vector<vector<int>> subsets(vector<int>& nums) {
    backtrack_subsets(nums, 0);
    return result;
}
```

# 常见应用

## 1. 连通分量检测

利用 DFS/BFS 可以检测图中连通分量的数量，用于判断图是否连通。

## 2. 环检测

在有向图中检测环是 DFS 的经典应用：

```cpp
// DFS 颜色标记法：0=未访问, 1=访问中, 2=已完成
bool has_cycle_directed(int u, vector<vector<int>>& g, 
                        vector<int>& color) {
    color[u] = 1;  // 标记为访问中
    for (int v : g[u]) {
        if (color[v] == 1) return true;   // 发现环
        if (color[v] == 0 && has_cycle_directed(v, g, color)) {
            return true;
        }
    }
    color[u] = 2;  // 标记为已完成
    return false;
}
```

## 3. 拓扑排序（前置知识）

拓扑排序是许多问题的前置知识，如任务调度、课程依赖等。DFS 可以用来判断有向无环图（DAG）并生成拓扑序列。

### 关键概念

- **有向无环图（DAG）**：存在至少一个拓扑序列的图
- **拓扑序**：对有向无环图的每条边 $(u, v)$，都有 $u$ 在序列中出现在 $v$ 之前

### DFS 拓扑排序实现

```cpp
vector<int> topo_sort_dfs(int n, vector<vector<int>>& g) {
    vector<int> color(n, 0);     // 0=未访问, 1=访问中, 2=已完成
    vector<int> order;           // 存储拓扑序（逆序）
    bool cycle = false;

    function<void(int)> dfs = [&](int u) {
        color[u] = 1;
        for (int v : g[u]) {
            if (color[v] == 1) {
                cycle = true;  // 发现环
                return;
            }
            if (color[v] == 0) {
                dfs(v);
            }
        }
        color[u] = 2;
        order.push_back(u);     // 后序时加入
    };

    for (int i = 0; i < n; ++i) {
        if (color[i] == 0) {
            dfs(i);
        }
    }

    if (cycle) return {};  // 有环，无拓扑序
    reverse(order.begin(), order.end());  // 逆序得到正序
    return order;
}
```

### 拓扑排序的性质

- **DAG 判定**：有向图存在拓扑序 $\iff$ 图是无环的
- **唯一性**：如果拓扑序唯一，则图中每条边 $(u, v)$ 都满足 $order[u] + 1 = order[v]$
- **应用场景**：课程表、任务调度、依赖安装、源码编译顺序

> 详细应用见：[[../data-structure/graph-topological-sort|拓扑排序]]

## 4. 二分图判定

利用 DFS（也可用 BFS）可以判定一个图是否为二分图：

```cpp
bool is_bipartite(int start, const vector<vector<int>>& g) {
    vector<int> color(g.size(), -1);  // -1=未染色, 0/1=两种颜色
    queue<int> q;
    q.push(start);
    color[start] = 0;

    while (!q.empty()) {
        int u = q.front(); q.pop();
        for (int v : g[u]) {
            if (color[v] == -1) {
                color[v] = 1 - color[u];
                q.push(v);
            } else if (color[v] == color[u]) {
                return false;  // 相邻节点同色，不是二分图
            }
        }
    }
    return true;
}
```

## 5. 欧拉路径/回路

在图论中，DFS 也是求解欧拉路径和回路的重要工具。

# DFS 与递归的关系

## 核心联系

DFS 和递归本质上是**同一个概念**的两种表达：

| 递归 | DFS |
|------|-----|
| 函数调用 | 图遍历 |
| 系统栈 | 显式栈（迭代版本） |
| 递归终止条件 | 访问边界条件 |
| 子问题 | 邻接节点 |

## 递归的本质

递归的核心要素：
1. **基准情况（Base Case）**：最小子问题的解
2. **递归情况（Recursive Case）**：将问题分解为更小的子问题
3. **状态保存**：在递归调用前后保持程序状态一致

## 递归的内存模型

每次递归调用都会在栈上分配新的局部变量，形成「深度优先」的调用顺序：

```cpp
void f(int n) {
    if (n <= 0) return;
    cout << "进入 f(" << n << ")\n";
    f(n - 1);  // 递归调用
    cout << "退出 f(" << n << ")\n";
}

int main() {
    f(3);
}
```

输出：
```
进入 f(3)
进入 f(2)
进入 f(1)
退出 f(1)
退出 f(2)
退出 f(3)
```

这正是**后序遍历**的体现。

## 递归的复杂度分析

- **时间复杂度**：$O(V + E)$，每个节点和边最多访问一次
- **空间复杂度**：$O(H)$，$H$ 为递归深度（系统栈深度）
  - 最好情况：平衡树，$H = O(\log n)$
  - 最坏情况：链表，$H = O(n)$，可能导致栈溢出

# 递归 vs 迭代

| 特性 | 递归 | 迭代 |
|------|------|------|
| 代码简洁性 | ✅ 更简洁 | ❌ 需要显式栈 |
| 内存效率 | ❌ 系统栈有限 | ✅ 自己管理 |
| 栈溢出风险 | ❌ 深递归可能 | ✅ 无此问题 |
| 调试难度 | ❌ 难以跟踪 | ✅ 容易单步 |
| 适用场景 | 树/图遍历 | 搜索路径、状态压缩 |

# 总结

DFS 是图论中最基本的搜索算法之一，其核心思想是「不撞南墙不回头」。通过递归或显式栈实现，DFS 可用于：

- 图/树的遍历（先序、后序）
- 连通分量检测
- 环检测
- 拓扑排序
- 回溯法（组合、排列、子集）
- 路径搜索

理解 DFS 的递归本质对于掌握更高级的算法（如记忆化搜索、动态规划的递归写法）至关重要。

# 参考

- [DFS（图论） - OI Wiki](https://oi-wiki.org/graph/dfs)
- [BFS（图论） - OI Wiki](https://oi-wiki.org/graph/bfs)
- [拓扑排序 - OI Wiki](https://oi-wiki.org/graph/topo/)
