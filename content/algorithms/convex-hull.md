---
title: 凸包
date: 2026-04-03
description: 凸包（Convex Hull）算法详解
tags:
  - convex-hull
  - computational-geometry
  - algorithm
draft: false
permalink:
---

## 定义（Definition）

**凸多边形（Convex Polygon）**：若多边形内任意两点的连线均完全位于多边形内部，则该多边形为凸多边形。

**凸包（Convex Hull）**：给定平面点集 $P$，凸包是包含 $P$ 的最小凸多边形。换言之，凸包是能够包裹所有给定点的最短凸边界。

## 几何理解（Geometric Understanding）

### 橡皮筋模型（Rubber Band Analogy）

想象在平面上放置所有给定点，在最外层点群周围套上一圈橡皮筋。橡皮筋收紧后会紧贴最外层的点，这些点即为凸包的顶点。

```
        p3
       /  \
      p2   p4
     /       \
    p1       p5
     \       /
      p0---p6
```

## Andrew's Algorithm（Monotone Chain）

Andrew's 算法是一种 $O(n \log n)$ 的凸包构造算法，因按单调顺序处理点而得名。

### 算法步骤

1. **排序**：将所有点按 $x$ 坐标升序排序，若 $x$ 相同则按 $y$ 升序排序
2. **构建下凸壳**：从左到右遍历，维护一个栈，每次加入新点时不断弹出栈顶直到最后三个点呈"左转"
3. **构建上凸壳**：从右到左遍历，同理构建上凸壳
4. **合并**：将上凸壳（去掉首尾两点）接到下凸壳后方

### 叉积判断方向（Cross Product for Orientation）

对于三个点 $A(x_1, y_1)$、$B(x_2, y_2)$、$C(x_3, y_3)$，叉积：

$$
\overrightarrow{AB} \times \overrightarrow{AC} = (x_2 - x_1)(y_3 - y_1) - (y_2 - y_1)(x_3 - x_1)
$$

- 结果 $> 0$：$C$ 在 $\overrightarrow{AB}$ 左侧，逆时针方向（Left Turn）
- 结果 $= 0$：三点共线（Collinear）
- 结果 $< 0$：$C$ 在 $\overrightarrow{AB}$ 右侧，顺时针方向（Right Turn）

```cpp
struct Point {
    long long x, y;
    bool operator<(const Point& other) const {
        if (x != other.x) return x < other.x;
        return y < other.y;
    }
};

long long cross(const Point& O, const Point& A, const Point& B) {
    return (A.x - O.x) * (B.y - O.y) - (A.y - O.y) * (B.x - O.x);
}

vector<Point> convexHull(vector<Point>& pts) {
    sort(pts.begin(), pts.end());
    pts.erase(unique(pts.begin(), pts.end(), [](const Point& a, const Point& b) {
        return a.x == b.x && a.y == b.y;
    }), pts.end());
    
    int n = pts.size();
    if (n <= 1) return pts;
    
    vector<Point> hull;
    // Build lower hull
    for (int i = 0; i < n; i++) {
        while (hull.size() >= 2 && cross(hull[hull.size()-2], hull[hull.size()-1], pts[i]) <= 0) {
            hull.pop_back();
        }
        hull.push_back(pts[i]);
    }
    
    // Build upper hull
    int lower_size = hull.size();
    for (int i = n - 2; i >= 0; i--) {
        while (hull.size() > lower_size && cross(hull[hull.size()-2], hull[hull.size()-1], pts[i]) <= 0) {
            hull.pop_back();
        }
        hull.push_back(pts[i]);
    }
    
    hull.pop_back(); // Remove duplicate endpoint
    return hull;
}
```

### 代码说明

- **去重**：`unique` 函数去除重复点，避免构建出退化的凸包
- **`<= 0`**：在构建下凸壳时使用 `<= 0`（含共线点），可去掉共线点；若要保留边界点，改用 `< 0`
- **首尾去重**：合并时 `pop_back()` 去除上凸壳的起点与下凸壳终点重复的问题

## 应用（Applications）

### 最近点对（Closest Pair）

最近点对问题可在 $O(n \log n)$ 时间内解决：先按 $x$ 坐标排序，再递归划分，在合并时利用带状区域筛选。[^1]

### 旋转卡尺（Rotating Calipers）

旋转卡尺是一系列可在凸包上 $O(n)$ 时间内解决的问题统称，包括：

- 最小面积外接矩形
- 最小宽度外接矩形
- 直径（最远点对）
- 切线问题

```cpp
// Rotating calipers for diameter
long long rotatingCaliipersDiameter(const vector<Point>& hull) {
    int n = hull.size();
    if (n == 1) return 0;
    
    long long max_dist = 0;
    int j = 1;
    for (int i = 0; i < n; i++) {
        int ni = (i + 1) % n;
        // Find farthest point from edge hull[i]-hull[ni]
        while (true) {
            int nj = (j + 1) % n;
            if (abs(cross(hull[i], hull[ni], hull[j])) < 
                abs(cross(hull[i], hull[ni], hull[nj]))) {
                j = nj;
            } else {
                break;
            }
        }
        max_dist = max(max_dist, dist(hull[i], hull[j]));
        max_dist = max(max_dist, dist(hull[ni], hull[j]));
    }
    return max_dist;
}
```

### 多边形面积（Polygon Area）

给定凸包顶点顺序，可利用叉积计算多边形面积（也叫"鞋带公式"）：

$$
S = \frac{1}{2} \left| \sum_{i=0}^{n-1} (x_i y_{i+1} - x_{i+1} y_i) \right|
$$

```cpp
long long polygonArea(const vector<Point>& hull) {
    long long area = 0;
    int n = hull.size();
    for (int i = 0; i < n; i++) {
        int j = (i + 1) % n;
        area += hull[i].x * hull[j].y;
        area -= hull[j].x * hull[i].y;
    }
    return abs(area);
}
```

## 例题（Sample Problems）

| 题目 | 来源 | 难度 | 考察点 |
|------|------|------|--------|
| 凸包基础 | LOJ #1266 | 入门 | 基本凸包构建 |
| 最小包围矩形 | POJ 1113 | 中等 | 旋转卡尺 |
| 凸包 + 双指针 | AtCoder ABC/ARC | 中等 | 凸包性质应用 |

## 参考（References）

[^1]: 最近点对问题 — [OI Wiki](https://oi-wiki.org/geometry/closest-pair/)
[^2]: 凸包 — [OI Wiki](https://oi-wiki.org/geometry/convex-hull/)

> **延伸阅读**：旋转卡尺（Rotating Calipers）的完整实现可参考 [Shamos 算法](https://en.wikipedia.org/wiki/Rotating_calipers)。
