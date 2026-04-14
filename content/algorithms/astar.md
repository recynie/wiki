---
title: A*搜索算法
date: 2026-04-06
description: A*（A-Star）算法是一种启发式搜索算法，结合了Dijkstra和贪心最佳优先搜索的优点
tags:
  - search
  - algorithm
  - heuristic
draft: false
permalink:
---

# A*搜索算法

A*（A-Star）算法是一种**启发式搜索**算法，用于在图中找到从起点到终点的最短路径。它结合了Dijkstra算法的完备性和贪心最佳优先搜索的效率。[^1]

[^1]: 本段参考[A*搜索算法 - OI Wiki](https://oi-wiki.org/search/)

## 核心公式

A*算法使用评估函数 $f(n)$ 来选择下一个扩展的节点：

$$
f(n) = g(n) + h(n)
$$

其中：
- $g(n)$：从起点到当前节点 $n$ 的实际代价
- $h(n)$：**启发函数**，从节点 $n$ 到终点的估计代价
- $f(n)$：通过节点 $n$ 到达终点的总估计代价

## 启发函数的要求

### 1. Admissible（可采纳性）

启发函数 $h(n)$ 必须是**可采纳的**，即它永远不会高估从 $n$ 到终点的实际代价：

$$
h(n) \leq h^*(n)
$$

其中 $h^*(n)$ 是实际最小代价。

### 2. Consistency（一致性）或 monotonicity（单调性）

对于任意节点 $n$ 和其后继节点 $n'$：

$$
h(n) \leq c(n, n') + h(n')
$$

其中 $c(n, n')$ 是边 $(n, n')$ 的代价。

## 算法步骤

```cpp
struct Node {
    int x, y;
    int g, h, f;
    bool operator<(const Node& other) const {
        return f > other.f;  // 小顶堆
    }
};

int astar(const vector<vector<int>>& grid, pair<int,int> start, pair<int,int> end) {
    int n = grid.size(), m = grid[0].size();
    vector<vector<int>> dist(n, vector<int>(m, INT_MAX));
    priority_queue<Node> pq;
    
    auto heuristic = [&](int x, int y) {
        return abs(x - end.first) + abs(y - end.second);  // Manhattan距离
    };
    
    dist[start.first][start.second] = 0;
    pq.push({start.first, start.second, 0, heuristic(start.first, start.second), 
             heuristic(start.first, start.second)});
    
    int dx[] = {1, -1, 0, 0};
    int dy[] = {0, 0, 1, -1};
    
    while (!pq.empty()) {
        Node cur = pq.top();
        pq.pop();
        
        if (cur.x == end.first && cur.y == end.second) {
            return cur.g;
        }
        
        for (int dir = 0; dir < 4; ++dir) {
            int nx = cur.x + dx[dir];
            int ny = cur.y + dy[dir];
            
            if (nx >= 0 && nx < n && ny >= 0 && ny < m && grid[nx][ny] == 0) {
                int newG = cur.g + 1;
                if (newG < dist[nx][ny]) {
                    dist[nx][ny] = newG;
                    int h = heuristic(nx, ny);
                    pq.push({nx, ny, newG, h, newG + h});
                }
            }
        }
    }
    
    return -1;  // 无路径
}
```

## 常见启发函数

| 问题类型 | 启发函数 | 公式 |
|---------|---------|------|
| 网格移动（4方向） | Manhattan距离 | $\|x_1 - x_2\| + \|y_1 - y$ |
| 网格移动（8方向） | Chebyshev距离 | $\max(\|x_1 - x_2\|, \|y_1 - y_2\|)$ |
| 任意方向移动 | Euclidean距离 | $\sqrt{(x_1 - x_2)^2 + (y_1 - y_2)^2}$ |
| 滑块 puzzle | Hamming距离 | 不在正确位置的棋子数 |

## A*与Dijkstra、BFS的关系

| 算法 | 评估函数 | 特点 |
|------|---------|------|
| BFS | $f(n) = g(n)$（层数） | 找到最短路径，无启发 |
| Dijkstra | $f(n) = g(n)$ | 找到最短路径，均匀扩展 |
| 贪心最佳优先 | $f(n) = h(n)$ | 最快找到解，不保证最优 |
| A* | $f(n) = g(n) + h(n)$ | 综合两者，保证最优 |

## 正确性证明思路

A*算法保证找到最短路径当且仅当启发函数是可采纳的：

1. **最优路径代价** $f^*(s)$：从起点 $s$ 到目标的最优路径代价
2. **扩展条件**：只有 $f(n) < f^*(s)$ 的节点才会被扩展
3. **终止证明**：当目标节点从优先队列弹出时，其 $g$ 值等于 $f^*(s)$

## 应用场景

- **路径规划**：游戏中的寻路、机器人导航
- **八皇后/八数码问题**：经典启发式搜索问题
- **迷宫求解**：带障碍的最短路径
- **调度问题**：任务分配、路由选择

## 参考

- [A*搜索算法 - OI Wiki](https://oi-wiki.org/search/)
- [Understanding A* - Red Blob Games](https://www.redblobgames.com/pathfinding/a-star/)
