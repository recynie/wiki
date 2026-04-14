---
title: 旋转卡壳
date: 2026-04-05
description: 旋转卡壳（Rotating Calipers）算法用于在凸包上高效求解最值问题
tags:
  - rotating-calipers
  - computational-geometry
  - convex-hull
  - algorithm
draft: false
permalink:
---

## 定义

旋转卡壳（Rotating Calipers）是一种基于几何优化的算法设计技术，主要用于在[[convex-hull|凸包]]上高效求解各类最值问题。

其核心思想是：在凸多边形外部构造两条平行线，使其与凸多边形相切，然后旋转这两条线，在旋转过程中维护某种极值关系。由于凸多边形的特殊性质，每个顶点最多被访问两次，因此总时间复杂度为 $O(n)$。

旋转卡壳可以解决以下问题：

- 凸包直径（最远点对）
- 凸包宽度
- 最小外接矩形
- 凸多边形直径

## 核心原理

### 对跖点性质

对于凸多边形上的任意一条边，其对跖点（antipodal point）是凸多边形上距离该边所在直线最远的顶点。当一条边旋转时，其对跖点沿着凸包边界单调移动。

### 算法流程

设凸包上有两条平行线 $L_1$ 和 $L_2$：

1. 初始化时，$L_1$ 固定在某一条边，$L_2$ 找到与其平行的边
2. 旋转 $L_1$，同时移动 $L_2$ 寻找对跖点
3. 在旋转过程中更新答案
4. 当 $L_1$ 旋转一周后，算法结束

### 复杂度分析

由于凸多边形的每个顶点最多作为对跖点被访问两次，因此总时间复杂度为 $O(n)$，空间复杂度为 $O(1)$。

## 典型应用

### 凸包直径（最远点对）

给定平面上的 $n$ 个点，求距离最远的两点之间的距离。

最远点对必然位于凸包的某两个顶点上。旋转卡壳在 $O(n)$ 时间内找到所有对跖点对，并维护最大距离。

### 凸包宽度

凸包宽度定义为凸多边形的一对平行支撑线之间的最小距离。通过旋转卡壳可以在 $O(n)$ 时间内找到凸包的宽度。

### 最小外接矩形

最小面积外接矩形（Minimum Area Bounding Rectangle）可以通过旋转卡壳求解。枚举凸包每条边作为矩形的某一边，构造与之平行的支撑线即可。

### 凸多边形直径

对于任意凸多边形，距离最远的两点必然在顶点上。旋转卡壳同样适用。

## 代码实现

### 凸包直径

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Point {
    long long x, y;
    bool operator<(const Point& other) const {
        if (x != other.x) return x < other.x;
        return y < other.y;
    }
    bool operator==(const Point& other) const {
        return x == other.x && y == other.y;
    }
};

Point operator-(const Point& a, const Point& b) {
    return {a.x - b.x, a.y - b.y};
}

long long cross(const Point& a, const Point& b) {
    return a.x * b.y - a.y * b.x;
}

long long cross(const Point& o, const Point& a, const Point& b) {
    return cross(a - o, b - o);
}

long long dist2(const Point& a, const Point& b) {
    long long dx = a.x - b.x;
    long long dy = a.y - b.y;
    return dx * dx + dy * dy;
}

// Andrew算法求凸包
vector<Point> convexHull(vector<Point>& pts) {
    sort(pts.begin(), pts.end());
    pts.erase(unique(pts.begin(), pts.end()), pts.end());
    int n = pts.size();
    if (n <= 1) return pts;
    
    vector<Point> hull;
    // 下半部分
    for (int i = 0; i < n; i++) {
        while (hull.size() >= 2 && cross(hull[hull.size()-2], hull[hull.size()-1], pts[i]) <= 0) {
            hull.pop_back();
        }
        hull.push_back(pts[i]);
    }
    // 上半部分
    int lower = hull.size();
    for (int i = n - 2; i >= 0; i--) {
        while (hull.size() > lower && cross(hull[hull.size()-2], hull[hull.size()-1], pts[i]) <= 0) {
            hull.pop_back();
        }
        hull.push_back(pts[i]);
    }
    hull.pop_back();
    return hull;
}

// 旋转卡壳求直径
long long rotatingCalipersDiameter(vector<Point>& hull) {
    int n = hull.size();
    if (n == 1) return 0;
    if (n == 2) return dist2(hull[0], hull[1]);
    
    long long maxDist = 0;
    int j = 1;
    for (int i = 0; i < n; i++) {
        // 找到与边 (i, i+1) 对应的对跖点
        int ni = (i + 1) % n;
        Point edge = hull[ni] - hull[i];
        while (abs(cross(edge, hull[(j + 1) % n] - hull[j])) > abs(cross(edge, hull[j] - hull[i]))) {
            j = (j + 1) % n;
        }
        // 更新答案
        maxDist = max(maxDist, dist2(hull[i], hull[j]));
        maxDist = max(maxDist, dist2(hull[ni], hull[j]));
    }
    return maxDist;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n;
    cin >> n;
    vector<Point> pts(n);
    for (int i = 0; i < n; i++) {
        cin >> pts[i].x >> pts[i].y;
    }
    
    vector<Point> hull = convexHull(pts);
    long long diameter = rotatingCalipersDiameter(hull);
    cout << diameter << endl;
    
    return 0;
}
```

### 求最小外接矩形

```cpp
long long rotatingCalipersMinAreaRectangle(vector<Point>& hull) {
    int n = hull.size();
    if (n == 1) return 0;
    if (n == 2) {
        long long d = dist2(hull[0], hull[1]);
        return d;  // 面积为0，但返回距离平方
    }
    
    long long minArea = LLONG_MAX;
    int j = 1, k = 1;
    for (int i = 0; i < n; i++) {
        int ni = (i + 1) % n;
        Point edge = hull[ni] - hull[i];
        
        // 找宽度方向的对跖点
        while (abs(cross(edge, hull[(j + 1) % n] - hull[j])) > abs(cross(edge, hull[j] - hull[i]))) {
            j = (j + 1) % n;
        }
        // 找高度方向的对跖点
        while (cross(edge, hull[(k + 1) % n] - hull[k]) > cross(edge, hull[k] - hull[i])) {
            k = (k + 1) % n;
        }
        
        // 计算当前矩形的宽和高
        long long width2 = dist2(hull[i], hull[ni]);
        long long height = abs(cross(edge, hull[j] - hull[i])) / sqrt(width2);
        long long width = abs(cross(edge, hull[k] - hull[i])) / sqrt(width2);
        
        minArea = min(minArea, width * height);
    }
    return minArea;
}
```

## 与凸包的关系

旋转卡壳算法依赖于[[convex-hull|凸包]]的存在，其工作流程如下：

1. **构建凸包**：使用 Andrew's 算法或 Graham's 算法在 $O(n \log n)$ 时间内构建凸包
2. **旋转卡壳**：在凸包上执行旋转卡壳，通常是 $O(n)$ 的追加操作

因此，总时间复杂度为 $O(n \log n)$，其中 $n$ 为输入点数。旋转卡壳是凸包算法的自然延伸，许多凸包相关问题都可以通过旋转卡壳高效解决。[^1]

## 例题

### 【模板】旋转卡壳

**题目描述**：给定 $n$ 个平面上的点 $(x_i, y_i)$，求最远点对的距离平方。

**输入格式**：第一行包含整数 $n$，接下来 $n$ 行每行包含两个整数 $x_i, y_i$。

**输出格式**：输出最远点对距离的平方。

**解题思路**：

1. 使用 Andrew 算法构建凸包
2. 使用旋转卡壳求凸包直径
3. 输出直径（即最远点对距离的平方）

**参考实现**：见上方代码实现部分。

---

## 参考资料

[^1]: 本页面内容参考 [CP-Algorithms - Rotating Calipers](https://cp-algorithms.com/geometry/rotating-calipers.html) 及 Codeforces 相关博客。
