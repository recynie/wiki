---
title: 费用流
date: 2026-04-05
description: 最小费用最大流（Min-Cost Max-Flow）算法详解
tags:
  - min-cost-flow
  - network-flow
  - graph
draft: false
permalink:
---

## 定义

给定一个网络 $G=(V,E)$，每条边除了有容量限制 $c(u,v)$，还有一个单位流量的费用 $w(u,v)$。

当 $(u,v)$ 的流量为 $f(u,v)$ 时，需要花费 $f(u,v) \times w(u,v)$ 的费用。

$w$ 满足斜对称性，即 $w(u,v) = -w(v,u)$。

则该网络中总花费最小的最大流称为 **最小费用最大流**，即在最大化 $\sum_{(s,v)\in E}f(s,v)$ 的前提下最小化 $\sum_{(u,v)\in E}f(u,v) \times w(u,v)$。

## SSP 算法

SSP（Successive Shortest Path）算法是一个贪心算法。它的思路是每次寻找单位费用最小的增广路进行增广，直到图上不存在增广路为止。

如果图上存在单位费用为负的圈，SSP 算法无法正确求出该网络的最小费用最大流。此时需要先使用消圈算法消去图上的负圈。

### 证明

我们考虑使用数学归纳法和反证法来证明 SSP 算法的正确性。

设流量为 $i$ 的时候最小费用为 $f_i$。我们假设最初的网络上 **没有负圈**，这种情况下 $f_0 = 0$。

假设用 SSP 算法求出的 $f_i$ 是最小费用，我们在 $f_i$ 的基础上，找到一条最短的增广路，从而求出 $f_{i+1}$。这时 $f_{i+1} - f_i$ 是这条最短增广路的长度。

假设存在更小的 $f_{i+1}$，设它为 $f'_{i+1}$。因为 $f_{i+1} - f_i$ 已经是最短增广路了，所以 $f'_{i+1} - f_i$ 一定对应一个经过 **至少一个负圈** 的增广路。

这时候矛盾就出现了：既然存在一条经过至少一个负圈的增广路，那么 $f_i$ 就不是最小费用了。因为只要给这个负圈添加流量，就可以在不增加 $s$ 流出的流量的前提下，使 $f_i$ 对应的费用更小。

综上，SSP 算法可以正确求出无负圈网络的最小费用最大流。

### 时间复杂度

如果使用 Bellman-Ford 算法求解最短路，每次找增广路的时间复杂度为 $O(nm)$。设该网络的最大流为 $f$，则最坏时间复杂度为 $O(nmf)$。事实上，SSP 算法是伪多项式时间的。

## 实现

只需将 EK 算法或 Dinic 算法中找增广路的过程，替换为用最短路算法寻找单位费用最小的增广路即可。

### 基于 EK 算法的实现

```cpp
struct qxx {
    int nex, t, v, c;  // v: 容量, c: 费用
};

vector<qxx> e;
vector<int> h;
int cnt = 1;

void add_path(int f, int t, int v, int c) {
    e[++cnt] = {h[f], t, v, c};
    h[f] = cnt;
}

void add_flow(int f, int t, int v, int c) {
    add_path(f, t, v, c);
    add_path(t, f, 0, -c);
}

vector<long long> dis;
vector<int> pre, incf;
vector<bool> vis;

bool spfa(int s, int t) {
    fill(dis.begin(), dis.end(), INF);
    queue<int> q;
    q.push(s), dis[s] = 0, incf[s] = INF, incf[t] = 0;
    while (!q.empty()) {
        int u = q.front();
        q.pop();
        vis[u] = false;
        for (int i = h[u]; i; i = e[i].nex) {
            int v = e[i].t, w = e[i].v, c = e[i].c;
            if (!w || dis[v] <= dis[u] + c) continue;
            dis[v] = dis[u] + c;
            incf[v] = min(w, incf[u]);
            pre[v] = i;
            if (!vis[v]) q.push(v), vis[v] = true;
        }
    }
    return incf[t] > 0;
}

int maxflow;
long long mincost;

void update(int s, int t) {
    maxflow += incf[t];
    for (int u = t; u != s; u = e[pre[u] ^ 1].t) {
        e[pre[u]].v -= incf[t];
        e[pre[u] ^ 1].v += incf[t];
        mincost += 1LL * incf[t] * e[pre[u]].c;
    }
}

// 调用: while(spfa(s, t)) update(s, t);
```

### 基于 Dinic 算法的实现

```cpp
struct Edge {
    int v, cap, cost, nxt;
};

vector<Edge> E;
vector<int> lnk, cur, dis;
vector<bool> vis;

void add_edge(int u, int v, int w, int c) {
    E.push_back({v, w, c, lnk[u]});
    lnk[u] = E.size() - 1;
    E.push_back({u, 0, -c, lnk[v]});
    lnk[v] = E.size() - 1;
}

bool spfa(int s, int t) {
    fill(dis.begin(), dis.end(), INF);
    memcpy(cur.data(), lnk.data(), lnk.size() * sizeof(int));
    queue<int> q;
    q.push(s), dis[s] = 0, vis[s] = true;
    while (!q.empty()) {
        int u = q.front();
        q.pop(), vis[u] = false;
        for (int i = lnk[u]; i; i = E[i].nxt) {
            int v = E[i].v;
            if (E[i].cap && dis[v] > dis[u] + E[i].cost) {
                dis[v] = dis[u] + E[i].cost;
                if (!vis[v]) q.push(v), vis[v] = true;
            }
        }
    }
    return dis[t] != INF;
}

int dfs(int u, int t, int flow) {
    if (u == t) return flow;
    vis[u] = true;
    int ans = 0;
    for (int &i = cur[u]; i && ans < flow; i = E[i].nxt) {
        int v = E[i].v;
        if (!vis[v] && E[i].cap && dis[v] == dis[u] + E[i].cost) {
            int x = dfs(v, t, min(E[i].cap, flow - ans));
            if (x) {
                ret += 1LL * x * E[i].cost;
                E[i].cap -= x;
                E[i ^ 1].cap += x;
                ans += x;
            }
        }
    }
    vis[u] = false;
    return ans;
}

int mcmf(int s, int t) {
    int ans = 0;
    while (spfa(s, t)) {
        int x;
        while ((x = dfs(s, t, INF))) ans += x;
    }
    return ans;
}
```

## Primal-Dual 原始对偶算法

用 Bellman-Ford 求解最短路的时间复杂度为 $O(nm)$，无论在稀疏图上还是稠密图上都不及 Dijkstra 算法。但网络上存在单位费用为负的边，因此无法直接使用 Dijkstra 算法。

Primal-Dual 原始对偶算法的思路与 Johnson 全源最短路径算法类似，通过为每个点设置一个势能，将网络上所有边的费用全部变为非负值，从而可以应用 Dijkstra 算法找出网络上单位费用最小的增广路。

首先跑一次最短路，求出源点到每个点的最短距离（也是该点的初始势能）$h_i$。接下来和 Johnson 算法一样，对于一条从 $u$ 到 $v$，单位费用为 $w$ 的边，将其边权重置为 $w + h_u - h_v$。

可以发现，这样设置势能后新网络上的最短路径和原网络上的最短路径一定对应。

与常规的最短路问题不同的是，每次增广后图的形态会发生变化，这种情况下各点的势能需要更新。设增广后从源点到 $i$ 号点的最短距离为 $d'_i$，只需给 $h_i$ 加上 $d'_i$ 即可。

## 应用场景

- 最小费用最大流问题
- 资源分配问题
- 运输问题
- 二分图最小权匹配

## 相关主题

- [[../algorithms/max-flow|最大流]]：费用流的基础
- [[../algorithms/dijkstra|Dijkstra算法]]：原始对偶算法中用到
- [[../algorithms/strongly-connected-components|强连通分量]]：2-SAT 问题

## 参考资料

[^1]: [费用流 - OI Wiki](https://oi-wiki.org/graph/flow/min-cost/)
