---
title: 离散化
date: 2026-04-05
description: 离散化（Discretization）技术详解：将连续值映射到离散索引，用于大规模坐标压缩
tags:
  - discretization
  - technique
draft: false
permalink:
---

## 定义

**离散化**是一种将连续值或大范围值域压缩到小范围索引的技术。[^1]

在算法问题中，经常遇到值域很大（如 $10^9$）但实际元素个数不多（如 $10^5$）的情况。离散化通过将所有出现过的值映射到 $[1, n]$ 的连续整数，使得可以用数组/桶等数据结构处理。

## 适用场景

- 值域很大但数据量较小的问题
- 需要用数组下标表示原始值
- 配合线段树、树状数组使用
- 离线查询的坐标压缩

## 基本方法

### 1. 无重复离散化

```cpp
vector<int> a = {100, 200, 300, 200, 400};  // 原始数组

// 复制并排序去重
vector<int> vals = a;
sort(vals.begin(), vals.end());
vals.erase(unique(vals.begin(), vals.end()), vals.end());

// 映射：原值 -> 离散化后的索引(从1开始)
for (int i = 0; i < a.size(); i++) {
    a[i] = lower_bound(vals.begin(), vals.end(), a[i]) - vals.begin() + 1;
}
// 结果：a = {1, 2, 3, 2, 4}
```

### 2. 保留重复信息

如果需要保留重复元素的位置信息，使用上面的方法即可，`lower_bound` 会返回第一个不小于该值的位置。

### 3. 二维离散化

```cpp
// 对二维坐标进行离散化
struct Point { int x, y; };

vector<Point> pts = {{100, 200}, {500, 600}, {100, 600}};

vector<int> xs, ys;
for (auto& p : pts) {
    xs.push_back(p.x);
    ys.push_back(p.y);
}

sort(xs.begin(), xs.end());
xs.erase(unique(xs.begin(), xs.end()), xs.end());
sort(ys.begin(), ys.end());
ys.erase(unique(ys.begin(), ys.end()), ys.end());

for (auto& p : pts) {
    p.x = lower_bound(xs.begin(), xs.end(), p.x) - xs.begin() + 1;
    p.y = lower_bound(ys.begin(), ys.end(), p.y) - ys.begin() + 1;
}
```

## 应用实例

### 例1：区间加法（坐标压缩 + 线段树）

```cpp
// 题目：多次操作 [l, r] += v，最后查询每点值
// l, r 是大数，但实际不同坐标数 <= 1e5

struct Op { int l, r, v; };

int main() {
    vector<Op> ops = {{1, 1000000000, 5}, {500, 600, 3}};
    
    // 收集所有端点
    vector<int> points;
    for (auto& op : ops) {
        points.push_back(op.l);
        points.push_back(op.r);
        points.push_back(op.r + 1);  // 区间加法通常需要右端点+1
    }
    
    sort(points.begin(), points.end());
    points.erase(unique(points.begin(), points.end()), points.end());
    
    int n = points.size();
    vector<long long> seg(n * 4);
    
    for (auto& op : ops) {
        int l = lower_bound(points.begin(), points.end(), op.l) - points.begin() + 1;
        int r = lower_bound(points.begin(), points.end(), op.r) - points.begin() + 1;
        // 对 [l, r] 执行加法
        update(1, 1, n, l, r, op.v);
    }
    return 0;
}
```

### 例2：并行查询（类似离散化）

```cpp
// 题目：数组初始全0，Q次操作将 [l, r] 区间加1，最后求每个位置的值
// 需要处理大规模稀疏操作

int main() {
    vector<pair<int, int>> ops = {{1, 5}, {3, 7}, {10, 15}};
    
    // 离散化所有端点
    vector<int> idx;
    for (auto& p : ops) {
        idx.push_back(p.first);
        idx.push_back(p.second + 1);  // 差分需要
    }
    sort(idx.begin(), idx.end());
    idx.erase(unique(idx.begin(), idx.end()), idx.end());
    
    vector<long long> diff(idx.size() + 1);
    
    for (auto& p : ops) {
        int l = lower_bound(idx.begin(), idx.end(), p.first) - idx.begin();
        int r = lower_bound(idx.begin(), idx.end(), p.second + 1) - idx.begin();
        diff[l] += 1;
        diff[r] -= 1;
    }
    
    // 前缀和得到结果
    vector<long long> ans(idx.size());
    long long cur = 0;
    for (int i = 0; i < idx.size(); i++) {
        cur += diff[i];
        ans[i] = cur;
    }
    return 0;
}
```

## 注意事项

1. **右端点+1**：区间操作常用差分数组，需要对 $r+1$ 也做标记
2. **浮点数离散化**：将浮点数乘以足够大的倍数，或使用 `map<double, int>`
3. **字符离散化**：用 `map<char, int>` 建立映射

## 与其他技术结合

| 技术 | 作用 |
|------|------|
| 线段树 | 离散化后的区间操作 |
| 树状数组 | 离散化后的前缀和 |
| 扫描线 | 坐标压缩减少状态 |
| 哈希表 | 大规模稀疏坐标处理 |

## 参考资料

[^1]: [离散化 - OI Wiki](https://oi-wiki.org/misc/discrete/)
