---
title: 欧拉路径
date: 2026-04-03
description: 欧拉路径与欧拉回路（Eulerian Path）
tags:
  - euler
  - graph
  - traversal
draft: false
permalink:
---

## 定义

### 欧拉路径（Eulerian Path）

在图中找到一条路径，使得该路径恰好经过每条边**一次**。若路径的起点和终点相同，则称为**欧拉回路（Eulerian Circuit）**；若不同，则称为**欧拉通路**。

### 欧拉图（Eulerian Graph）

- **欧拉回路图**：具有欧拉回路的图
- **欧拉通路图**：仅有欧拉通路而无欧拉回路的图

---

## 判定条件

### 欧拉回路存在条件

| 条件 | 说明 |
|------|------|
| 连通性 | 图必须连通（忽略孤立点） |
| 度数条件 | **所有**顶点的度数均为**偶数** |

### 欧拉路径存在条件

| 条件 | 说明 |
|------|------|
| 连通性 | 图必须连通（忽略孤立点） |
| 度数条件 | 恰好 **0** 或 **2** 个奇度顶点 |

- 0 个奇度顶点 → 存在欧拉回路
- 2 个奇度顶点 → 存在欧拉通路（起点和终点为这两个奇度点）

### 记忆口诀

> **有奇无回路，多奇多通路，零奇两偶是回路**

---

## Hierholzer 算法

Fleury 算法复杂度为 $O(E^2)$，实际应用中通常使用 **Hierholzer 算法**，复杂度为 $O(E)$。

### 算法思想

1. **选择起点**：若有欧拉回路则任意选一点；若有欧拉通路则选奇度点
2. **DFS 遍历**：沿着未访问的边深入，标记已访问
3. **栈维护**：将遍历过的边入栈
4. **回溯输出**：递归返回时依次弹出栈顶，输出边的顺序即为欧拉回路/路径

### 算法流程

```
Hierholzer(start):
    path = empty stack
    circuit = empty list
    path.push(start)
    
    while path is not empty:
        v = path.top()
        if v has unvisited edges:
            e = any unvisited edge from v
            remove e from graph
            path.push(e.to)
        else:
            circuit.append(path.pop())
    
    reverse circuit
    return circuit
```

### 图示说明

以无向图为例，Hierholzer 算法的关键在于**逆序输出**。递归深入直到无法继续，然后将顶点依次弹出，形成的序列正好是欧拉回路的逆序。

---

## 代码模板

### 无向图（Undirected Graph）

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Edge {
    int to;
    int id;
};

vector<vector<Edge>> g;
vector<int> usedEdge;
vector<int> ans;

// 无向图 Hierholzer
void hierholzer_undirected(int start) {
    stack<int> st;
    st.push(start);
    
    while (!st.empty()) {
        int v = st.top();
        bool hasUnvisited = false;
        
        for (auto &e : g[v]) {
            int eid = e.id;
            if (!usedEdge[eid]) {
                usedEdge[eid] = 1;
                st.push(e.to);
                hasUnvisited = true;
                // 移除这条边（防止重复访问）
                break;
            }
        }
        
        if (!hasUnvisited) {
            ans.push_back(st.top());
            st.pop();
        }
    }
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n, m;
    cin >> n >> m;
    g.resize(n);
    usedEdge.assign(m, 0);
    
    // 读取无向边
    for (int i = 0; i < m; i++) {
        int u, v;
        cin >> u >> v;
        g[u].push_back({v, i});
        g[v].push_back({u, i});
    }
    
    // 检查是否存在欧拉路径/回路
    int start = 0;
    int oddCount = 0;
    vector<int> degree(n, 0);
    
    for (int i = 0; i < n; i++) {
        if (!g[i].empty()) start = i;
        oddCount += (g[i].size() % 2);
    }
    
    if (oddCount == 0) {
        cout << "存在欧拉回路" << endl;
    } else if (oddCount == 2) {
        cout << "存在欧拉通路" << endl;
    } else {
        cout << "不存在欧拉路径或回路" << endl;
        return 0;
    }
    
    hierholzer_undirected(start);
    reverse(ans.begin(), ans.end());
    
    cout << "欧拉路径/回路: ";
    for (int v : ans) cout << v << " ";
    cout << endl;
    
    return 0;
}
```

### 有向图（Directed Graph）

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Edge {
    int to;
    int id;
};

vector<vector<Edge>> g;
vector<int> usedEdge;
vector<int> ans;

// 有向图 Hierholzer
void hierholzer_directed(int start) {
    stack<int> st;
    st.push(start);
    
    while (!st.empty()) {
        int v = st.top();
        bool hasUnvisited = false;
        
        for (auto &e : g[v]) {
            int eid = e.id;
            if (!usedEdge[eid]) {
                usedEdge[eid] = 1;
                st.push(e.to);
                hasUnvisited = true;
                break;
            }
        }
        
        if (!hasUnvisited) {
            ans.push_back(st.top());
            st.pop();
        }
    }
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n, m;
    cin >> n >> m;
    g.resize(n);
    usedEdge.assign(m, 0);
    
    // 读取有向边
    for (int i = 0; i < m; i++) {
        int u, v;
        cin >> u >> v;
        g[u].push_back({v, i});
    }
    
    // 有向图欧拉路径判定：出度等于入度（回路）或恰好一个起点一个终点
    vector<int> indeg(n, 0), outdeg(n, 0);
    for (int u = 0; u < n; u++) {
        for (auto &e : g[u]) {
            outdeg[u]++;
            indeg[e.to]++;
        }
    }
    
    int start = 0;
    int oddIn = 0, oddOut = 0;
    for (int i = 0; i < n; i++) {
        if (!g[i].empty()) start = i;
        oddIn += (indeg[i] - outdeg[i] == 1);
        oddOut += (outdeg[i] - indeg[i] == 1);
    }
    
    if ((oddIn == 0 && oddOut == 0) || (oddIn == 1 && oddOut == 1)) {
        cout << "存在欧拉回路/通路" << endl;
    } else {
        cout << "不存在欧拉路径或回路" << endl;
        return 0;
    }
    
    hierholzer_directed(start);
    reverse(ans.begin(), ans.end());
    
    cout << "欧拉路径/回路: ";
    for (int v : ans) cout << v << " ";
    cout << endl;
    
    return 0;
}
```

---

## 应用场景

### 一笔画问题

著名的「一笔画」问题本质就是判断是否存在欧拉路径或欧拉回路。

- **能一笔画**：欧拉回路（0 个奇点）或欧拉通路（2 个奇点）
- **不能一笔画**：奇点数量大于 2

著名的七桥问题（哥尼斯堡桥问题）正是欧拉证明不存在一笔画走法的经典案例。

### 回路打印

在电路板布线、邮递员送信（ chinese POSTMAN Problem）等场景中，需要找到经过每条边至少一次的最小路径。欧拉路径是解决此类问题的基础。

### 邮递员问题（Postman Problem）

- **中国邮递员问题**：在带权无向图中找到经过每条边至少一次并返回起点的最短路径
- 若图是欧拉图，则欧拉回路即为最优解
- 若不是欧拉图，需要引入额外的「重复边」来消除奇点影响

---

## 与 Hamilton 路径的区别和联系

| 特征 | 欧拉路径 | Hamilton 路径 |
|------|----------|----------------|
| 遍历对象 | **边**（每条边恰好一次） | **顶点**（每个顶点恰好一次） |
| 判定难度 | 多项式时间（$O(E)$） | NP 完全 |
| 存在条件 | 度数奇偶性 + 连通性 | 无简单充要条件 |
| 算法 | Hierholzer $O(E)$ | 无多项式算法 |

### 联系

- 两者都是路径问题，目标是找到一条满足特定条件的遍历路径
- 在某些特殊图（如完全图）中，Hamilton 路径容易构造，但欧拉路径仍需按度数判定
- 两者结合的「欧拉-哈密顿图」是图论研究的重要方向

---

## 参考资料

[^1]: 本段来自[欧拉路径 - OI Wiki](https://oi-wiki.org/graph/euler/)

[^2]: 图论基础与算法实现，清华大学出版社
