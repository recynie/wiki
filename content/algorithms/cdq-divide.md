---
title: CDQ分治
date: 2026-04-05
description: CDQ分治（CDQ Divide and Conquer）算法详解：解决三维偏序、动态DP等问题的高效离线算法
tags:
  - cdq-divide
  - offline-algorithm
  - divide-and-conquer
draft: false
permalink:
---

## 定义

**CDQ分治**是一种高效处理偏序问题的离线算法，由IOI金牌选手陈丹琦提出。[^1]

核心思想：将问题分解为「只涉及左半部分」、「只涉及右半部分」和「跨越左右两部分」三类，分别处理后合并。

## 经典问题：三维偏序

### 问题描述

给定 $n$ 个元素，每个元素有三维属性 $(x, y, z)$。求满足 $x_i < x_j$，$y_i < y_j$，$z_i < z_j$ 的有序对 $(i, j)$ 的数量。

### 解决思路

1. 按 $x$ 排序，消除第一维的影响
2. 在分治过程中，按 $y$ 进行归并排序
3. 用树状数组处理第三维 $z$

```cpp
struct Node {
    int x, y, z;
    int id;      // 原始编号
    int ans;     // 答案
};

int n;
vector<Node> a;

// 按y排序的归并过程
void cdq(int l, int r) {
    if (l >= r) return;
    
    int mid = (l + r) >> 1;
    cdq(l, mid);
    cdq(mid + 1, r);
    
    // 归并过程中处理跨越左右的情况
    vector<Node> tmp(r - l + 1);
    int i = l, j = mid + 1, k = 0;
    
    while (i <= mid && j <= r) {
        if (a[i].y <= a[j].y) {
            // 左半部分的元素，对右半部分产生贡献
            BIT.add(a[i].z, 1);
            tmp[k++] = a[i++];
        } else {
            // 右半部分的元素，统计来自左半部分的贡献
            a[j].ans += BIT.query(a[j].z);
            tmp[k++] = a[j++];
        }
    }
    
    while (i <= mid) {
        BIT.add(a[i].z, 1);
        tmp[k++] = a[i++];
    }
    
    while (j <= r) {
        a[j].ans += BIT.query(a[j].z);
        tmp[k++] = a[j++];
    }
    
    // 清理树状数组
    for (int p = l; p <= mid; p++) {
        BIT.add(a[p].z, -1);
    }
    
    // 复制回原数组
    for (int p = 0; p < k; p++) {
        a[l + p] = tmp[p];
    }
}
```

## 动态DP问题

### 问题描述

有 $n$ 个物品，每个有重量 $w_i$ 和价值 $v_i$。有两种操作：
- 查询：前 $i$ 个物品中总重量不超过 $W$ 的最大价值
- 更新：修改某个物品的重量或价值

### CDQ优化DP

```cpp
struct Query {
    int type;    // 0=更新, 1=查询
    int idx;     // 物品编号
    int w, v;    // 重量、价值（更新时）
    int W;       // 查询的容量
    long long ans;
};

vector<Query> queries;

// CDQ分治处理
void cdq(int l, int r) {
    if (l >= r) return;
    
    int mid = (l + r) >> 1;
    cdq(l, mid);
    cdq(mid + 1, r);
    
    // 处理更新对查询的影响
    vector<Query> tmp(r - l + 1);
    int i = l, j = mid + 1, k = 0;
    
    // 按重量排序
    while (i <= mid && j <= r) {
        if (queries[i].w <= queries[j].w) {
            if (queries[i].type == 0) {
                // 更新操作，加入数据结构
                BIT.add(queries[i].v, queries[i].idx);
            }
            tmp[k++] = queries[i++];
        } else {
            if (queries[j].type == 1) {
                // 查询操作，统计所有<=当前重量的更新
                queries[j].ans = max(queries[j].ans, 
                    BIT.query(queries[j].W));
            }
            tmp[k++] = queries[j++];
        }
    }
    
    while (j <= r) {
        if (queries[j].type == 1) {
            queries[j].ans = max(queries[j].ans, 
                BIT.query(queries[j].W));
        }
        tmp[k++] = queries[j++];
    }
    
    // 复制回并清理
    for (int p = l; p <= mid; p++) {
        if (queries[p].type == 0) {
            BIT.clear(queries[p].v);
        }
    }
    
    for (int p = 0; p < k; p++) {
        queries[l + p] = tmp[p];
    }
}
```

## 复杂度分析

| 问题规模 | 时间复杂度 | 空间复杂度 |
|----------|------------|------------|
| 三维偏序 | $O(n \log^2 n)$ | $O(n)$ |
| 动态DP | $O(n \log^2 n)$ | $O(n)$ |

## 与其他算法的对比

| 算法 | 适用场景 | 复杂度 |
|------|----------|--------|
| CDQ分治 | 偏序统计、DP优化 | $O(n \log^k n)$ |
| 莫队 | 区间查询 | $O(n \sqrt{n})$ |
| 整体二分 | 判定性二分 | $O(n \log n)$ |
| 线段树 | 动态单点修改区间查询 | $O(n \log n)$ |

## 典型应用

1. **三维偏序**：统计满足多个条件的点对
2. **动态逆序对**：带修改的逆序对统计
3. **DP优化**：斜率优化、单调队列无法处理的情况
4. **离线CDQ**：将在线问题转为离线处理
5. **树状数组套CDQ**：处理更复杂的偏序

## 参考资料

[^1]: [CDQ分治 - OI Wiki](https://oi-wiki.org/misc/cdq-divide/)
