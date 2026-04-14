---
title: KD-Tree
date: 2026-04-07
description: KD-Tree（K维树）是一种用于处理K维空间信息的数据结构
tags:
  - data-structure
  - kd-tree
  - computational-geometry
draft: false
permalink:
---

## 定义

KD-Tree（K-Dimensional Tree）是一种用于处理K维空间信息的数据结构，本质上是一棵二叉搜索树，每个节点代表K维空间中的一个点。

**核心思想**：在树的每一层，按不同的维度进行划分。交替使用各个维度作为划分依据，形成近似平衡的二叉树。

## 构造算法

### 二维KD-Tree（常用场景）

1. 预处理：计算所有点的方差，选择方差最大的维度作为当前层的划分维度
2. 按该维度排序，取中位数作为当前节点
3. 递归处理左右子集

```cpp
struct Node {
    Point p;           // 点的坐标
    int l, r;          // 左右孩子
    int mn[2], mx[2];  // 子树覆盖的矩形范围
};
```

### 中位数分割

使用 `nth_element` 在 O(n) 时间内找到中位数，保证树的平衡性。

## 基本操作

### 插入

类似 BST 插入，根据当前层维度选择左右方向。需要记录子树范围。

### 最近邻查询

1. 先递归到叶子节点，沿路记录当前最近距离
2. 回溯时，判断另一子树是否可能包含更近的点（通过计算当前维度的距离）
3. 如果可能，递归搜索另一子树

复杂度：平均 O(log n)，最坏 O(n)

### K近邻查询

类似最近邻，但维护一个大小为 K 的优先队列。

### 范围查询

查询矩形 $[rx_1, rx_2] \times [ry_1, ry_2]$ 内的所有点。

## 代码模板

给出完整的二维 KD-Tree 实现，包括：
- 节点结构
- 构造函数（递归建树）
- 插入
- 最近邻查询
- K近邻查询
- 范围查询

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Point {
    double x, y;
    int id;
};

struct Node {
    Point p;
    int l, r;
    double mn[2], mx[2];  // 子树覆盖的矩形范围
};

struct KDTree {
    vector<Node> tr;
    int root;

    // 比较函数，按维度排序
    static bool cmpX(const Point& a, const Point& b) { return a.x < b.x; }
    static bool cmpY(const Point& a, const Point& b) { return a.y < b.y; }

    // 更新节点管辖范围
    void update(int u) {
        int l = tr[u].l, r = tr[u].r;
        for (int i = 0; i < 2; i++) {
            tr[u].mn[i] = tr[u].mx[i] = i == 0 ? tr[u].p.x : tr[u].p.y;
            if (l) {
                tr[u].mn[i] = min(tr[u].mn[i], tr[l].mn[i]);
                tr[u].mx[i] = max(tr[u].mx[i], tr[l].mx[i]);
            }
            if (r) {
                tr[u].mn[i] = min(tr[u].mn[i], tr[r].mn[i]);
                tr[u].mx[i] = max(tr[u].mx[i], tr[r].mx[i]);
            }
        }
    }

    // 递归建树
    int build(vector<Point>& ps, int l, int r, int dep) {
        if (l > r) return 0;
        int mid = (l + r) >> 1;
        int dim = dep % 2;
        nth_element(ps.begin() + l, ps.begin() + mid, ps.begin() + r + 1,
                     dim == 0 ? cmpX : cmpY);
        int u = tr.size();
        tr.push_back({ps[mid], 0, 0});
        tr[u].l = build(ps, l, mid - 1, dep + 1);
        tr[u].r = build(ps, mid + 1, r, dep + 1);
        update(u);
        return u;
    }

    // 插入点
    void insert(Point p, int& u, int dep) {
        if (!u) {
            u = tr.size();
            tr.push_back({p, 0, 0});
            update(u);
            return;
        }
        int dim = dep % 2;
        if (dim == 0 ? p.x < tr[u].p.x : p.y < tr[u].p.y) {
            insert(p, tr[u].l, dep + 1);
        } else {
            insert(p, tr[u].r, dep + 1);
        }
        update(u);
    }

    // 计算点到矩形的最小距离
    double distToRect(double x, double y, int u) {
        double res = 0;
        if (x < tr[u].mn[0]) res += (tr[u].mn[0] - x) * (tr[u].mn[0] - x);
        if (x > tr[u].mx[0]) res += (x - tr[u].mx[0]) * (x - tr[u].mx[0]);
        if (y < tr[u].mn[1]) res += (tr[u].mn[1] - y) * (tr[u].mn[1] - y);
        if (y > tr[u].mx[1]) res += (y - tr[u].mx[1]) * (y - tr[u].mx[1]);
        return res;
    }

    // 最近邻查询
    double best;
    Point bestPoint;
    void queryNearest(Point q, int u, int dep) {
        if (!u) return;
        // 更新当前最近距离
        double d = hypot(q.x - tr[u].p.x, q.y - tr[u].p.y);
        if (d < best) {
            best = d;
            bestPoint = tr[u].p;
        }
        int dim = dep % 2;
        int l = tr[u].l, r = tr[u].r;
        // 先搜索更可能包含近邻的子树
        int first = l, second = r;
        if (dim == 0 ? q.x > tr[u].p.x : q.y > tr[u].p.y) {
            swap(first, second);
        }
        if (first) queryNearest(q, first, dep + 1);
        // 判断另一子树是否可能包含更近的点
        if (second) {
            double d2 = distToRect(q.x, q.y, second);
            if (d2 < best) queryNearest(q, second, dep + 1);
        }
    }

    double nearest(Point q) {
        best = 1e18;
        queryNearest(q, root, 0);
        return best;
    }

    // K近邻查询
    struct NodeDist {
        double d;
        Point p;
        bool operator<(const NodeDist& nd) const { return d < nd.d; }
    };
    vector<NodeDist> kbest;
    void queryKnn(Point q, int u, int dep, int k) {
        if (!u) return;
        double d = hypot(q.x - tr[u].p.x, q.y - tr[u].p.y);
        if (d < best || (int)kbest.size() < k) {
            if ((int)kbest.size() == k) kbest.pop_back();
            kbest.push_back({d, tr[u].p});
            push_heap(kbest.begin(), kbest.end());
            best = kbest.front().d;
        }
        int dim = dep % 2;
        int l = tr[u].l, r = tr[u].r;
        int first = l, second = r;
        if (dim == 0 ? q.x > tr[u].p.x : q.y > tr[u].p.y) {
            swap(first, second);
        }
        if (first) queryKnn(q, first, dep + 1, k);
        if (second) {
            double d2 = distToRect(q.x, q.y, second);
            if (d2 < best || (int)kbest.size() < k) queryKnn(q, second, dep + 1, k);
        }
    }

    vector<Point> knearest(Point q, int k) {
        best = 1e18;
        kbest.clear();
        queryKnn(q, root, 0, k);
        vector<Point> res;
        for (auto& nd : kbest) res.push_back(nd.p);
        return res;
    }

    // 范围查询
    vector<Point> ans;
    void queryRange(Point q1, Point q2, int u) {
        if (!u) return;
        // 检查当前点是否在范围内
        if (tr[u].p.x >= q1.x && tr[u].p.x <= q2.x &&
            tr[u].p.y >= q1.y && tr[u].p.y <= q2.y) {
            ans.push_back(tr[u].p);
        }
        int l = tr[u].l, r = tr[u].r;
        // 判断子树是否与查询矩形有交集
        if (l && !(tr[l].mx[0] < q1.x || tr[l].mn[0] > q2.x ||
                   tr[l].mx[1] < q1.y || tr[l].mn[1] > q2.y)) {
            queryRange(q1, q2, l);
        }
        if (r && !(tr[r].mx[0] < q1.x || tr[r].mn[0] > q2.x ||
                   tr[r].mx[1] < q1.y || tr[r].mn[1] > q2.y)) {
            queryRange(q1, q2, r);
        }
    }

    vector<Point> rangeQuery(Point q1, Point q2) {
        ans.clear();
        queryRange(q1, q2, root);
        return ans;
    }

    void init(vector<Point>& ps) {
        tr.clear();
        root = build(ps, 0, (int)ps.size() - 1, 0);
    }
};
```

## 应用场景

1. **最近点对** - 二维平面上的最近点对问题[[^1]]
2. **K近邻** - 机器学习中的 KNN 算法基础
3. **范围搜索** - 矩形/圆形区域内的点查询

## 替罪羊树与KD-Tree对比

| 特性 | 替罪羊树 | KD-Tree |
|------|---------|---------|
| 维度 | 一维 | 多维 |
| 平衡方式 | 替罪羊重构 | 中位数分割 |
| 应用 | 普通平衡树 | 空间查询 |

## 模板题

- 洛谷 P1429 平面最近点对（KD-Tree）[[^1]]
- 洛谷 P4148 简单题（KD-Tree 范围和）

## 参考资料

[^1]: 本段参考了[洛谷 KD-Tree 学习笔记](https://www.luogu.com.cn/blog/515-hardball/learning-kd-tree)和 [OI Wiki](https://oi-wiki.org/ds/kdt/)
