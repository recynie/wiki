---
title: 半平面交
date: 2026-04-05
description: 半平面交（Half-plane Intersection）算法详解：求多个半平面的交集，用于线性规划、多边形核等
tags:
  - computational-geometry
  - half-plane
draft: false
permalink:
---

## 定义

**半平面**是直线及其一侧构成的点集，解析式为 $Ax + By + C \ge 0$。[^1]

**半平面交**是多个半平面的交集，在平面直角坐标系中围成一个区域（通常为凸多边形）。

### 多边形的核

如果一个点集中的点与多边形上任意一点的连线都在多边形内部，则该点集为多边形的核。

求解多边形的核：将每条边看成首尾相连的向量，内部方向的半平面交即为多边形的核。

## S&I 算法

使用单调队列维护，时间复杂度 $O(n \log n)$。

### 极角排序

```cpp
struct Line {
    Point a, b;  // 有向线段，左侧为半平面
};

double atan2(Point p) {
    return atan2(p.y, p.x);
}

bool cmp(const Line& x, const Line& y) {
    double t1 = atan2(x.b - x.a);
    double t2 = atan2(y.b - y.a);
    if (fabs(t1 - t2) > eps) return t1 < t2;
    // 共线时，取左侧的
    return (y.b - x.a) * (y.a - x.a) > eps;
}
```

### 求两直线交点

```cpp
Point intersect(Line l1, Line l2) {
    Point p = l1.a, v = l1.b - l1.a;
    Point q = l2.a, w = l2.b - l2.a;
    double t = (w ^ (q - p)) / (v ^ w);
    return p + v * t;
}
```

### 判断点是否在直线右侧

```cpp
double cross(Point a, Point b, Point c) {
    return (b - a) ^ (c - a);
}

bool onRight(Line l, Point p) {
    return cross(l.a, l.b, p) < -eps;
}
```

### 单调队列维护

```cpp
vector<Point> halfPlaneIntersection(vector<Line> lines) {
    sort(lines.begin(), lines.end(), cmp);
    
    // 去重（极角相同保留最左侧的）
    vector<Line> uniq;
    for (int i = 0; i < lines.size(); ) {
        int j = i;
        while (j < lines.size() && fabs(atan2(lines[j].b - lines[j].a) - 
                                         atan2(lines[i].b - lines[i].a)) < eps) {
            j++;
        }
        // 保留最靠左的
        Line keep = lines[i];
        for (int k = i + 1; k < j; k++) {
            if ((lines[k].a - keep.a) * (lines[k].b - keep.a) > eps) {
                keep = lines[k];
            }
        }
        uniq.push_back(keep);
        i = j;
    }
    lines = uniq;
    
    int n = lines.size();
    deque<Line> q;
    deque<Point> pts;
    
    for (int i = 0; i < n; i++) {
        if (q.size() >= 2 && !isParallel(q.back(), lines[i])) {
            // 检查队尾
            while (q.size() >= 2 && onRight(q.back(), pts.back())) {
                q.pop_back();
                pts.pop_back();
            }
            pts.push_back(intersect(q.back(), lines[i]));
        }
        if (q.size() >= 2 && isParallel(q.back(), lines[i])) continue;
        
        while (q.size() >= 2 && onRight(q.front(), pts.back())) {
            q.pop_back();
            pts.pop_back();
        }
        
        q.push_back(lines[i]);
        if (q.size() >= 2) {
            pts.push_back(intersect(q[q.size()-2], q.back()));
        }
    }
    
    // 收尾处理
    while (q.size() > 2 && onRight(q.back(), pts.back())) {
        q.pop_back();
        pts.pop_back();
    }
    while (q.size() > 2 && onRight(q.front(), pts.front())) {
        q.pop_front();
        pts.pop_front();
    }
    
    vector<Point> result;
    for (auto& p : pts) result.push_back(p);
    result.push_back(intersect(q.front(), q.back()));
    return result;
}
```

## 复杂度分析

| 步骤 | 时间复杂度 |
|------|------------|
| 极角排序 | $O(n \log n)$ |
| 单调队列 | $O(n)$ |
| 总计 | $O(n \log n)$ |

## 应用场景

1. **线性规划**：求可行域
2. **多边形核**：判断多边形是否有核
3. **半平面交面积**：求凸多边形面积
4. **视锥裁剪**：计算机图形学

## 例题

**POJ 1279 - Art Gallery**：求多边形的核

```cpp
// 核心思路：将每条边转为指向内部的有向线段，求半平面交
int main() {
    int n;
    while (cin >> n && n) {
        vector<Point> p(n);
        for (int i = 0; i < n; i++) {
            cin >> p[i].x >> p[i].y;
        }
        
        vector<Line> lines;
        for (int i = 0; i < n; i++) {
            Point a = p[i];
            Point b = p[(i + 1) % n];
            // 保证内部在左侧
            Line l = {a, b};
            if (cross(a, b, p[(i + 2) % n]) < 0) {
                swap(l.a, l.b);
            }
            lines.push_back(l);
        }
        
        vector<Point>核 = halfPlaneIntersection(lines);
        if (核.empty()) {
            cout << 0 << endl;
        } else {
            double area = 0;
            for (int i = 0; i <核.size(); i++) {
                area += cross(核[i],核[(i+1)%核.size()]);
            }
            cout << fixed << setprecision(2) << fabs(area) / 2 << endl;
        }
    }
    return 0;
}
```

## 参考资料

[^1]: [半平面交 - OI Wiki](https://oi-wiki.org/geometry/half-plane/)
