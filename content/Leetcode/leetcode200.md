---
title: LeetCode 200 - 岛屿数量
date: 2026-04-06
description: 给定一个由'1'（陆）和'0'（水）组成的二维网格图，返回岛屿的数量
tags:
  - dfs
  - bfs
  - graph
  - leetcode
draft: false
permalink:
---

# LeetCode 200 - 岛屿数量

## 题目描述

给定一个 `m × n` 的二维网格图 `grid`，地图上 `'1'` 表示陆地，`'0'` 表示水域。返回**岛屿**的数量。

岛屿的定义：
- 由水平或垂直方向的 `'1'` 连接而成
- 四连通（上下左右）
- 被水域环绕

### 示例

```
输入：grid = [
  ["1","1","1","1","0"],
  ["1","1","0","1","0"],
  ["1","1","0","0","0"],
  ["0","0","0","0","0"]
]
输出：1

输入：grid = [
  ["1","1","0","0","0"],
  ["1","1","0","0","0"],
  ["0","0","1","0","0"],
  ["0","0","0","1","1"]
]
输出：3
```

## 解法一：DFS（深度优先搜索）

### 思路分析

遍历整个网格，当遇到未被访问的 `'1'` 时：
1. 这是一个新岛屿，计数器 +1
2. 从该位置开始 DFS，将所有相邻的 `'1'` 标记为已访问

### 代码实现

```cpp
class Solution {
public:
    int numIslands(vector<vector<char>>& grid) {
        if (grid.empty() || grid[0].empty()) return 0;
        
        int n = grid.size(), m = grid[0].size();
        int count = 0;
        
        function<void(int, int)> dfs = [&](int x, int y) {
            // 越界或已是水域，返回
            if (x < 0 || x >= n || y < 0 || y >= m || grid[x][y] == '0') {
                return;
            }
            
            // 标记为已访问
            grid[x][y] = '0';
            
            // 递归访问四个方向
            dfs(x + 1, y);
            dfs(x - 1, y);
            dfs(x, y + 1);
            dfs(x, y - 1);
        };
        
        for (int i = 0; i < n; ++i) {
            for (int j = 0; j < m; ++j) {
                if (grid[i][j] == '1') {
                    ++count;
                    dfs(i, j);
                }
            }
        }
        
        return count;
    }
};
```

### 复杂度分析

| 复杂度 | 值 |
|--------|-----|
| 时间复杂度 | $O(m \times n)$ |
| 空间复杂度 | $O(m \times n)$（最坏情况为全是岛屿时递归深度） |

## 解法二：BFS（广度优先搜索）

```cpp
class Solution {
public:
    int numIslands(vector<vector<char>>& grid) {
        if (grid.empty() || grid[0].empty()) return 0;
        
        int n = grid.size(), m = grid[0].size();
        int count = 0;
        queue<pair<int, int>> q;
        int dx[4] = {1, -1, 0, 0};
        int dy[4] = {0, 0, 1, -1};
        
        for (int i = 0; i < n; ++i) {
            for (int j = 0; j < m; ++j) {
                if (grid[i][j] == '1') {
                    ++count;
                    grid[i][j] = '0';
                    q.push({i, j});
                    
                    while (!q.empty()) {
                        auto [x, y] = q.front();
                        q.pop();
                        
                        for (int dir = 0; dir < 4; ++dir) {
                            int nx = x + dx[dir];
                            int ny = y + dy[dir];
                            
                            if (nx >= 0 && nx < n && ny >= 0 && ny < m 
                                && grid[nx][ny] == '1') {
                                grid[nx][ny] = '0';
                                q.push({nx, ny});
                            }
                        }
                    }
                }
            }
        }
        
        return count;
    }
};
```

## 解法三：并查集

```cpp
class DSU {
    vector<int> parent, rank;
public:
    DSU(int n) : parent(n), rank(n, 0) {
        iota(parent.begin(), parent.end(), 0);
    }
    
    int find(int x) {
        return parent[x] == x ? x : parent[x] = find(parent[x]);
    }
    
    void unite(int x, int y) {
        x = find(x);
        y = find(y);
        if (x == y) return;
        if (rank[x] < rank[y]) swap(x, y);
        parent[y] = x;
        if (rank[x] == rank[y]) ++rank[x];
    }
};

class Solution {
public:
    int numIslands(vector<vector<char>>& grid) {
        if (grid.empty() || grid[0].empty()) return 0;
        
        int n = grid.size(), m = grid[0].size();
        DSU dsu(n * m);
        int dx[4] = {1, -1, 0, 0};
        int dy[4] = {0, 0, 1, -1};
        
        for (int i = 0; i < n; ++i) {
            for (int j = 0; j < m; ++j) {
                if (grid[i][j] == '1') {
                    for (int dir = 0; dir < 4; ++dir) {
                        int ni = i + dx[dir];
                        int nj = j + dy[dir];
                        if (ni >= 0 && ni < n && nj >= 0 && nj < m 
                            && grid[ni][nj] == '1') {
                            dsu.unite(i * m + j, ni * m + nj);
                        }
                    }
                }
            }
        }
        
        unordered_set<int> roots;
        for (int i = 0; i < n; ++i) {
            for (int j = 0; j < m; ++j) {
                if (grid[i][j] == '1') {
                    roots.insert(dsu.find(i * m + j));
                }
            }
        }
        
        return roots.size();
    }
};
```

## 变形题目

| 题目 | 变形 |
|------|------|
| [[leetcode463|463. 岛屿的周长]] | 计算岛屿周长 |
| [[leetcode695|695. 岛屿的最大面积]] | 求最大岛屿面积 |
| [[leetcode694|694. 不同岛屿的数量]] | 区分不同形状的岛屿 |

## 关键点

1. **网格搜索**：二维矩阵的 DFS/BFS 需注意边界条件
2. **标记访问**：在 DFS 中直接将 `grid[x][y]` 设为 `'0'` 是常用的剪枝技巧
3. **连通分量**：本题本质上是求「1」形成的连通分量数量

## 参考

- [LeetCode 200 - 岛屿数量](https://leetcode.cn/problems/number-of-islands/)
- [[../algorithms/depth-first-search|DFS]]
- [[../algorithms/breadth-first-search|BFS]]
- [[../data-structure/union-find|并查集]]
