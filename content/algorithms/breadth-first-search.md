---
title: BFS(Breadth-First Search)
date: 2026-03-13
description: 广度优先搜索（Breadth-First Search, BFS）
tags:
  - bfs
  - graph
draft: false
permalink:
---
# BFS算法
BFS和[[depth-first-search|DFS]]是图搜索中最基础的算法。
BFS每次都尝试访问同一层的节点。如果同一层都访问完了，再访问下一层。这样做的结果是，BFS 算法找到的路径是从起点开始的**最短**合法路径。换言之，这条路径所包含的边数最小。在 BFS 结束时，每个节点都是通过从起点到该点的最短路径访问的。[^1]
BFS的实现需要使用[[../data-structure/queue|队列(queue)]]。

[^1]: 本段来自[BFS（图论） - OI Wiki](https://oi-wiki.org/graph/bfs)
# 算法模板
## 网格搜索
网格上的搜索需要定义**后继关系**（相当于图的边）、**边界判断**（在网格内）。下面是在一个网格上的BFS搜索模板，把上下左右四个方向作为可行方向。
```cpp
#include <bits/stdc++.h>
using namespace std;

// 如果是网格，可以用 dx/dy 定义方向
int dx[4] = {1, -1, 0, 0};
int dy[4] = {0, 0, 1, -1};

int main() {
    int n, m; // n行 m列或节点数
    cin >> n >> m;
    vector<vector<int>> vis(n, vector<int>(m, 0));// 记录访问
    queue<pair<int,int>> q; // BFS队列
    int sx, sy;             // 起点
    cin >> sx >> sy;

    q.push({sx, sy});
    vis[sx][sy] = 1;

    while(!q.empty()) {
        auto [x, y] = q.front(); q.pop();
        // 遍历邻居(后继节点)
        for(int i = 0; i < 4; ++i) { // 网格4方向
            int nx = x + dx[i];
            int ny = y + dy[i];
            // 判断边界和访问
            if(nx >= 0 && nx < n && ny >= 0 && ny < m && !vis[nx][ny]) {
                vis[nx][ny] = 1;
                q.push({nx, ny});
            }
        }
    }
}
```
记录数组`visited`是必须的，否则将重复访问（和扩展）已经访问过的节点，这会让算法和遍历等价。
## 图搜索
# 例题
- [[../CCF-CSP/CSP202506B|机器人复建指南]]中使用的是分层的网格上的BFS。由于指定搜索了几层，所以使用BFS而不是DFS搜索网格。题目求的是可遍历的节点个数，所以需要一个变量保存访问的节点数，由于入队列(`q.push`)的操作是唯一的（任意节点最多入队一次），所以把它的自增和入队绑定即可。
