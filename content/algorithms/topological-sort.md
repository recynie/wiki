---
title: 拓扑排序
date: 2026-03-31
description: 拓扑排序（Topological Sort）
tags:
  - graph
  - topological-sort
  - dag
draft: false
permalink:
---

## 定义

拓扑排序（Topological Sort）是对**有向无环图**（DAG, Directed Acyclic Graph）的所有顶点进行线性排序的算法，使得对于每一条有向边 $(u, v)$，顶点 $u$ 都出现在顶点 $v$ 之前。

换句话说，拓扑排序找到一种满足所有依赖关系的任务执行顺序。[^1]

## 核心特性

### 前提条件

- 图必须是**有向无环图**（DAG）
- 如果图中有环，则不存在拓扑排序

### 时间复杂度

| 算法 | 时间复杂度 |
|------|------------|
| Kahn 算法（BFS） | $O(V + E)$ |
| DFS 方法 | $O(V + E)$ |

### 结果不唯一

拓扑排序的结果可能不唯一，只要满足约束条件即可。

## Kahn 算法（BFS 方法）

Kahn 算法的核心思想是：**不断删除入度为 0 的顶点**。

1. 计算所有顶点的入度
2. 将所有入度为 0 的顶点加入队列
3. 从队列取出顶点 $u$，将其加入拓扑序列
4. 将 $u$ 的所有邻接边 $(u, v)$ 删除，相当于将 $v$ 的入度减 1
5. 如果 $v$ 的入度变为 0，则将 $v$ 加入队列
6. 重复步骤 3-5，直到队列为空
7. 如果输出的顶点数小于 $V$，则图中存在环[^2]

```cpp
#include <bits/stdc++.h>
using namespace std;

vector<int> topologicalSortKahn(int n, vector<vector<int>>& adj) {
    vector<int> indegree(n, 0);
    queue<int> q;
    vector<int> result;
    
    // 计算入度
    for (int u = 0; u < n; u++) {
        for (int v : adj[u]) {
            indegree[v]++;
        }
    }
    
    // 入度为 0 的顶点入队
    for (int i = 0; i < n; i++) {
        if (indegree[i] == 0) {
            q.push(i);
        }
    }
    
    // BFS 过程
    while (!q.empty()) {
        int u = q.front();
        q.pop();
        result.push_back(u);
        
        for (int v : adj[u]) {
            indegree[v]--;
            if (indegree[v] == 0) {
                q.push(v);
            }
        }
    }
    
    // 检查是否有环
    if (result.size() != n) {
        return {};  // 存在环
    }
    
    return result;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n = 6;  // 顶点数
    vector<vector<int>> adj(n);
    
    // 示例：课程安排
    // 课程 0 -> 课程 1 -> 课程 2
    // 课程 0 -> 课程 2
    // 课程 3 -> 课程 4
    adj[0] = {1, 2};
    adj[1] = {2};
    adj[3] = {4};
    
    vector<int> topo = topologicalSortKahn(n, adj);
    
    if (topo.empty()) {
        cout << "图中存在环，无法拓扑排序" << endl;
    } else {
        cout << "拓扑排序结果: ";
        for (int v : topo) {
            cout << v << " ";
        }
        cout << endl;
    }
    
    return 0;
}
```

## DFS 方法

DFS 方法基于以下观察：在 DFS 中，**后序遍历的逆序**就是一个拓扑排序。

```cpp
#include <bits/stdc++.h>
using namespace std;

bool dfs(int u, vector<vector<int>>& adj, vector<int>& visited, vector<int>& result) {
    visited[u] = 1;  // 正在访问
    
    for (int v : adj[u]) {
        if (visited[v] == 1) {  // 发现环
            return false;
        }
        if (visited[v] == 0) {  // 未访问
            if (!dfs(v, adj, visited, result)) {
                return false;
            }
        }
    }
    
    visited[u] = 2;  // 访问完成
    result.push_back(u);  // 后序加入
    return true;
}

vector<int> topologicalSortDFS(int n, vector<vector<int>>& adj) {
    vector<int> visited(n, 0);  // 0=未访问, 1=正在访问, 2=已完成
    vector<int> result;
    
    for (int i = 0; i < n; i++) {
        if (visited[i] == 0) {
            if (!dfs(i, adj, visited, result)) {
                return {};  // 存在环
            }
        }
    }
    
    reverse(result.begin(), result.end());  // 逆序得到拓扑排序
    return result;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n = 6;
    vector<vector<int>> adj(n);
    
    adj[0] = {1, 2};
    adj[1] = {2};
    adj[3] = {4};
    
    vector<int> topo = topologicalSortDFS(n, adj);
    
    if (topo.empty()) {
        cout << "图中存在环" << endl;
    } else {
        cout << "拓扑排序结果: ";
        for (int v : topo) {
            cout << v << " ";
        }
        cout << endl;
    }
    
    return 0;
}
```

## 实战模板

### 课程表问题

```cpp
#include <bits/stdc++.h>
using namespace std;

bool canFinish(int numCourses, vector<pair<int, int>>& prerequisites) {
    vector<vector<int>> adj(numCourses);
    vector<int> indegree(numCourses, 0);
    
    for (auto& p : prerequisites) {
        adj[p.second].push_back(p.first);
        indegree[p.first]++;
    }
    
    queue<int> q;
    for (int i = 0; i < numCourses; i++) {
        if (indegree[i] == 0) q.push(i);
    }
    
    int count = 0;
    while (!q.empty()) {
        int u = q.front();
        q.pop();
        count++;
        
        for (int v : adj[u]) {
            if (--indegree[v] == 0) {
                q.push(v);
            }
        }
    }
    
    return count == numCourses;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int numCourses = 2;
    vector<pair<int, int>> prerequisites = {{1, 0}};
    
    cout << (canFinish(numCourses, prerequisites) ? "Yes" : "No") << endl;
    
    return 0;
}
```

### 字典序最小的拓扑排序

```cpp
#include <bits/stdc++.h>
using namespace std;

vector<int> smallestTopoSort(int n, vector<vector<int>>& adj) {
    vector<int> indegree(n, 0);
    priority_queue<int, vector<int>, greater<int>> pq;  // 小顶堆
    
    for (int u = 0; u < n; u++) {
        for (int v : adj[u]) {
            indegree[v]++;
        }
    }
    
    for (int i = 0; i < n; i++) {
        if (indegree[i] == 0) {
            pq.push(i);
        }
    }
    
    vector<int> result;
    while (!pq.empty()) {
        int u = pq.top();
        pq.pop();
        result.push_back(u);
        
        for (int v : adj[u]) {
            if (--indegree[v] == 0) {
                pq.push(v);
            }
        }
    }
    
    if (result.size() != n) return {};  // 有环
    return result;
}
```

## 应用场景

### 任务调度

根据任务间的依赖关系确定执行顺序，如 Makefile、编译顺序等。

### 课程安排

大学课程的前置课程要求。

### 病毒检测

判断文件依赖图是否有循环依赖。

### [[../algorithms/dijkstra|Dijkstra]] 算法的依赖

在 [[../algorithms/dijkstra|Dijkstra]] 算法中，如果需要记录路径，需要按拓扑顺序处理。

### [[../data-structure/union-find|Kruskal]] 最小生成树

Kruskal 算法需要按边权重排序，但也可以用拓扑排序的思路理解。

## 参考资料

[^1]: 拓扑排序的概念最早由卡塞尔（Kahn）在1962年提出，也称为"先后序关系"或"关键路径分析"。

[^2]: Kahn 算法的优点是可以通过队列中顶点的顺序选择实现不同的拓扑排序，DFS 方法的优点是实现简洁且容易并行化。
