---
title: 扫描线（ Sweep Line ）
date: 2026-04-07
description: 扫描线算法是一种高效处理计算几何问题的技术，通过假想的线沿某一方向扫描来计算目标值。
tags:
  - sweep-line
  - computational-geometry
draft: false
permalink:
---

## 算法思想

扫描线算法的核心思想是用一条「假想的线」沿着某一方向扫描，当线移动到特定位置时事件被触发，从而计算目标值。类似于[[../data-structure/segment-tree|线段树]]的懒标记更新。[^1]

扫描线算法主要应用于以下场景：

1. **矩形面积并** — 求多个矩形覆盖的总面积
2. **矩形周长并** — 求多个矩形覆盖的边界总长度
3. **平面点集最近点对** — 分治法

---

## 矩形面积并

矩形面积并是扫描线算法最经典的应用之一。给定 $n$ 个矩形，求它们覆盖的总面积。[^2]

### 算法步骤

1. **离散化 $y$ 坐标**：将所有矩形的 $y$ 坐标（上下边界）存入数组 $Y$，排序去重后用于线段树建图。

2. **事件处理**：
   - 每个矩形拆成两条水平边（上边和下边）
   - 下边（矩形进入）：在对应 $y$ 区间加入 $+1$ 标记
   - 上边（矩形离开）：在对应 $y$ 区间加入 $-1$ 标记

3. **扫描过程**：
   - 按 $x$ 坐标从左到右扫描所有事件
   - 用线段树维护当前 $x$ 位置各 $y$ 区间的覆盖长度
   - 面积累加：$S += \text{覆盖长度} \times \Delta x$

4. **线段树实现要点**：
   - 节点维护两个值：$cnt$（覆盖次数）和 $len$（实际被覆盖的长度）
   - 当 $cnt > 0$ 时，该区间完全被覆盖
   - 使用懒标记下传实现区间更新

---

## 代码模板

```cpp
#include <bits/stdc++.h>
using namespace std;

struct SegmentTree {
    struct Node {
        int cnt;        // 覆盖次数，cnt > 0 表示被覆盖
        long long len; // 区间实际被覆盖的长度
    };
    vector<Node> tree;
    vector<long long> Y; // 离散化的y坐标
    int n;
    
    SegmentTree(const vector<long long>& y) {
        Y = y;
        n = (int)Y.size() - 1; // 区间数为n
        tree.resize(n * 4);
        build(1, 0, n);
    }
    
    void build(int id, int l, int r) {
        tree[id].cnt = 0;
        tree[id].len = 0;
        if (l == r) return;
        int mid = (l + r) >> 1;
        build(id << 1, l, mid);
        build(id << 1 | 1, mid + 1, r);
    }
    
    // 区间 [L, R] 增加 val（val 为 +1 或 -1）
    void update(int id, int l, int r, int L, int R, int val) {
        if (L <= l && r <= R) {
            tree[id].cnt += val;
            push_up(id, l, r);
            return;
        }
        int mid = (l + r) >> 1;
        if (L <= mid) update(id << 1, l, mid, L, R, val);
        if (R > mid) update(id << 1 | 1, mid + 1, r, L, R, val);
        push_up(id, l, r);
    }
    
    void push_up(int id, int l, int r) {
        if (tree[id].cnt > 0) {
            // 被覆盖，直接用Y计算长度
            tree[id].len = Y[r + 1] - Y[l];
        } else if (l == r) {
            tree[id].len = 0;
        } else {
            tree[id].len = tree[id << 1].len + tree[id << 1 | 1].len;
        }
    }
    
    long long query() {
        return tree[1].len;
    }
};

struct Edge {
    long long x;      // 扫描线的x坐标
    int y1, y2;       // 边的y坐标（离散化前的原始值）
    int type;         // +1 表示下边（进入），-1 表示上边（离开）
    // 比较时按x排序，x相同则进入边先于离开边
};

long long rectangleUnion(vector<array<long long, 4>>& rects) {
    // rects[i] = {x1, y1, x2, y2}，左下角(x1,y1)，右上角(x2,y2)
    vector<long long> ys;
    vector<Edge> edges;
    
    for (auto& r : rects) {
        long long x1 = r[0], y1 = r[1], x2 = r[2], y2 = r[3];
        ys.push_back(y1);
        ys.push_back(y2);
        // 下边：x1, +1
        edges.push_back({x1, (int)ys.size() - 2, (int)ys.size() - 1, +1});
        // 上边：x2, -1
        edges.push_back({x2, (int)ys.size() - 2, (int)ys.size() - 1, -1});
    }
    
    // 离散化y坐标
    sort(ys.begin(), ys.end());
    ys.erase(unique(ys.begin(), ys.end()), ys.end());
    
    // 更新边的y坐标为离散化后的索引
    for (auto& e : edges) {
        e.y1 = (int)(lower_bound(ys.begin(), ys.end(), ys[e.y1]) - ys.begin());
        e.y2 = (int)(lower_bound(ys.begin(), ys.end(), ys[e.y2]) - ys.begin()) - 1;
    }
    
    // 按x坐标排序
    sort(edges.begin(), edges.end(), [](const Edge& a, const Edge& b) {
        if (a.x != b.x) return a.x < b.x;
        return a.type > b.type; // 进入边先于离开边
    });
    
    SegmentTree seg(ys);
    long long area = 0;
    long long lastX = edges[0].x;
    
    for (size_t i = 0; i < edges.size(); i++) {
        long long curX = edges[i].x;
        // 累加面积：覆盖长度 × dx
        area += seg.query() * (curX - lastX);
        
        // 处理所有在curX位置的边
        size_t j = i;
        while (j < edges.size() && edges[j].x == curX) {
            seg.update(1, 0, (int)ys.size() - 2, 
                       edges[j].y1, edges[j].y2, edges[j].type);
            j++;
        }
        lastX = curX;
        i = j - 1;
    }
    
    return area;
}
```

### 使用示例

```cpp
int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n;
    cin >> n;
    vector<array<long long, 4>> rects(n);
    for (int i = 0; i < n; i++) {
        cin >> rects[i][0] >> rects[i][1] >> rects[i][2] >> rects[i][3];
    }
    
    cout << rectangleUnion(rects) << endl;
    return 0;
}
```

---

## 矩形周长并

矩形周长并问题的求解思路与面积并类似，但需要额外维护一些信息。[^3]

### 与面积并的区别

- **面积并**：只需要维护「被覆盖的区间总长度」
- **周长并**：需要额外维护「区间边界数量」，用于计算周长贡献

具体来说，在扫描线移动时：
- 水平边贡献：$|\Delta x| \times \text{覆盖的横边数}$
- 竖直边贡献：需要根据覆盖状态的变化计算

```cpp
struct PerimeterSegTree {
    struct Node {
        int cnt;        // 覆盖次数
        long long len;  // 区间覆盖长度
        int numSeg;     // 区间内有多少段连续被覆盖的部分
        int lBound, rBound; // 区间左右端点是否有覆盖（用于合并）
    };
    // ... 类似结构，需要在push_up时维护numSeg和Bound信息
};
```

---

## 平面最近点对

平面点集最近点对问题可以在 $O(n \log n)$ 时间内解决，使用分治法结合扫描线思想。[^4]

### 算法流程

1. **预处理**：按 $x$ 坐标从小到大排序所有点。

2. **递归求解**：
   - 将点集分成左右两半，递归求解左右两半的最小距离 $d$。

3. **合并阶段**：
   - 只考虑距离中线小于 $d$ 的点（候选点）
   - 在中线附近维护一个按 $y$ 坐标排序的窗口
   - 对于每个候选点，只需检查其后面 $y$ 距离小于 $d$ 的点（最多 7 个）

### 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Point {
    double x, y;
};

bool cmpX(const Point& a, const Point& b) {
    return a.x < b.x;
}

bool cmpY(const Point& a, const Point& b) {
    return a.y < b.y;
}

double dist(const Point& a, const Point& b) {
    return sqrt((a.x - b.x) * (a.x - b.x) + (a.y - b.y) * (a.y - b.y));
}

double closestPair(vector<Point>& pts, int l, int r) {
    if (r - l <= 1) {
        // 叶子节点，返回极大值
        return 1e18;
    }
    
    int mid = (l + r) >> 1;
    double midX = pts[mid].x;
    
    // 递归求解左右两半
    double d = min(closestPair(pts, l, mid), closestPair(pts, mid, r));
    
    // 合并：按y坐标归并
    vector<Point> temp;
    int i = l, j = mid;
    while (i < mid && j < r) {
        if (pts[i].y < pts[j].y) temp.push_back(pts[i++]);
        else temp.push_back(pts[j++]);
    }
    while (i < mid) temp.push_back(pts[i++]);
    while (j < r) temp.push_back(pts[j++]);
    for (int k = 0; k < (int)temp.size(); k++) {
        pts[l + k] = temp[k];
    }
    
    // 收集距离中线小于d的点
    vector<Point> band;
    for (int i = l; i < r; i++) {
        if (abs(pts[i].x - midX) < d) {
            band.push_back(pts[i]);
        }
    }
    
    // 在band中查找y距离小于d的点对
    int m = band.size();
    for (int i = 0; i < m; i++) {
        for (int j = i + 1; j < m && band[j].y - band[i].y < d; j++) {
            d = min(d, dist(band[i], band[j]));
        }
    }
    
    return d;
}

double closestPair(vector<Point>& pts) {
    sort(pts.begin(), pts.end(), cmpX);
    return closestPair(pts, 0, (int)pts.size());
}
```

### 复杂度分析

| 阶段 | 时间复杂度 |
| ---- | ---------- |
| 排序 | $O(n \log n)$ |
| 递归求解 | $O(n \log n)$ |
| 总计 | $O(n \log n)$ |

---

## 相关主题

- [[../data-structure/segment-tree|线段树]] — 扫描线的核心数据结构，用于高效维护区间覆盖信息
- [[../algorithms/convex-hull|凸包]] — 另一类计算几何问题，与扫描线同属经典算法

---

[^1]: 扫描线算法通过将二维问题转化为一维扫描，结合区间数据结构实现高效计算。
[^2]: 矩形面积并是扫描线最经典的应用，时间复杂度为 $O(n \log n)$。
[^3]: 矩形周长并需要同时维护覆盖长度和边界数量，比面积并更复杂。
[^4]: 平面最近点对问题使用分治法可以达到 $O(n \log n)$ 的时间复杂度。
