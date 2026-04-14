---
title: 课程表 (LeetCode 207)
date: 2026-04-11
description: 使用拓扑排序判断课程学习顺序是否存在
tags:
  - topological-sort
  - graph
  - bfs
draft: false
permalink:
---

## 课程表 (LeetCode 207)

你是个大学生，需要选修 `numCourses` 门课程，课程编号从 `0` 到 `numCourses - 1`。有些课程有先修要求：给定一个数组 `prerequisites`，其中 `prerequisites[i] = [ai, bi]` 表示若要选修课程 `ai`，必须先完成课程 `bi`。

判断是否可能完成所有课程的学习（即是否存在一个可行的课程学习顺序）。

### 算法思路：拓扑排序（BFS）

将课程和先修关系建模为有向图：

- **节点**：每门课程
- **有向边**：`bi -> ai`（必须先学 bi 才能学 ai）
- **入度**：每门课程有多少门先修课程

#### Kahn 算法（BFS）

1. 计算每门课程的入度
2. 将所有入度为 0 的课程加入队列（这些课程可以立即学习）
3. 不断取出队首课程，学习它，并将其后继课程的入度减 1
4. 若后继课程入度变为 0，则加入队列
5. 结束时，若学习的课程数量等于总课程数，则存在可行顺序

### C++ 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

class Solution {
public:
    bool canFinish(int numCourses, vector<vector<int>>& prerequisites) {
        vector<vector<int>> adj(numCourses);  // 邻接表
        vector<int> indegree(numCourses, 0);   // 入度
        
        // 构建图和入度
        for (auto& p : prerequisites) {
            int a = p[0], b = p[1];
            adj[b].push_back(a);
            ++indegree[a];
        }
        
        queue<int> q;
        // 将所有入度为0的课程加入队列
        for (int i = 0; i < numCourses; ++i) {
            if (indegree[i] == 0) q.push(i);
        }
        
        int learned = 0;
        while (!q.empty()) {
            int course = q.front();
            q.pop();
            ++learned;
            
            for (int next : adj[course]) {
                if (--indegree[next] == 0) {
                    q.push(next);
                }
            }
        }
        
        return learned == numCourses;
    }
};
```

### 复杂度分析

- **时间复杂度**：$O(V + E)$，V 为课程数，E为先修关系数
- **空间复杂度**：$O(V + E)$，存储图和入度数组

### 检测环的原理

若图中存在环，则环内的课程入度始终大于 0，无法加入队列，最终学习的课程数小于总课程数。

### 参考

- [[topological-sort|拓扑排序]]
- [[../data-structure/bfs|BFS 广度优先搜索]]
