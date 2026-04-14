---
title: 树状数组
date: 2026-04-01
description: 树状数组（Fenwick Tree / Binary Indexed Tree）是一种高效实现前缀和查询与单点更新的数据结构。
tags:
  - fenwick
  - bit
  - data-structure
draft: false
permalink:
---

## 1. 定义

树状数组（Fenwick Tree），又称**二进制索引树**（Binary Indexed Tree，简称 BIT），由 Peter Fenwick 于 1994 年提出。它是一种能够在 $O(\log n)$ 时间内完成**单点更新**与**前缀和查询**的数据结构，空间复杂度为 $O(n)$。

### 为什么需要树状数组？

传统的数组：
- 前缀和查询：$O(1)$（预计算）
- 单点更新：$O(1)$

如果数据**静态不变**，预计算前缀和即可。但实际应用中，数据往往**动态变化**——每次更新后需要重新计算前缀和，导致 $O(n)$ 的时间复杂度。

树状数组正是为解决这一矛盾而生：它同时支持动态环境下的快速更新与查询。

> 本质上，树状数组是用**二进制表示**巧妙地分解了前缀和，将区间 $[1, i]$ 拆分为若干子区间，每个子区间对应树中的一个节点。[^1]

---

## 2. 核心原理

### 2.1 低bit运算

树状数组的核心是 `lowbit(i)` 操作：

$$
\text{lowbit}(i) = i \& (-i)
$$

它返回 $i$ 在二进制下**最低位的 1** 所对应的值。

举例说明：

| $i$ | 二进制 | $\text{lowbit}(i)$ | 含义 |
|-----|--------|-------------------|------|
| $1$  | $0001$ | $1$  | $2^0$ |
| $2$  | $0010$ | $2$  | $2^1$ |
| $3$  | $0011$ | $1$  | $2^0$ |
| $4$  | $0100$ | $4$  | $2^2$ |
| $8$  | $1000$ | $8$  | $2^3$ |
| $12$ | $1100$ | $4$  | $2^2$ |

### 2.2 数据结构视图

树状数组将数组 $A[1..n]$ 划分为若干区间，每个区间长度为 $2^k$（$k$ 为该位置二进制中 trailing zeros 的数量）。

```
下标:     1   2   3   4   5   6   7   8
         ┌───┬───────┬───────────────┐
BIT[1]:  │ A[1] │ A[2] │ A[3] │       │ A[4] │ A[5] │ A[6] │ A[7] │       │ A[8] │
         └───┴───────┴───────────────┘
管理区间: [1]  [1,2]  [3]  [1,4]  [5]  [5,6]  [7]  [1,8]
lowbit:   1    2     1     4     1     2     1     8
```

**关键性质**：
- `BIT[i]` 管理的区间为 $(i - \text{lowbit}(i), i]$，长度为 $\text{lowbit}(i)$
- 父节点为 $i + \text{lowbit}(i)$（如果不超过 $n$）

---

## 3. 基本操作

### 3.1 单点更新 `update(i, delta)`

将 $A[i]$ 增加 $delta$，同时更新所有受影响的 BIT 节点：

```cpp
void update(int i, long long delta) {
    for (; i <= n; i += i & -i) {
        bit[i] += delta;
    }
}
```

时间复杂度：$O(\log n)$

### 3.2 前缀和查询 `query(i)`

查询 $A[1] + A[2] + \cdots + A[i]$：

```cpp
long long query(int i) {
    long long res = 0;
    for (; i > 0; i -= i & -i) {
        res += bit[i];
    }
    return res;
}
```

时间复杂度：$O(\log n)$

### 3.3 区间和查询 `query(l, r)`

查询 $A[l] + A[l+1] + \cdots + A[r]$：

```cpp
long long query(int l, int r) {
    return query(r) - query(l - 1);
}
```

---

## 4. 完整实现

```cpp
#include <bits/stdc++.h>
using namespace std;

class Fenwick {
private:
    int n;
    vector<long long> bit;

public:
    Fenwick(int n = 0) : n(n), bit(n + 1, 0) {}

    // 单点更新：将 A[i] 增加 delta
    void update(int i, long long delta) {
        for (; i <= n; i += i & -i) {
            bit[i] += delta;
        }
    }

    // 前缀和查询：sum[1..i]
    long long query(int i) const {
        long long res = 0;
        for (; i > 0; i -= i & -i) {
            res += bit[i];
        }
        return res;
    }

    // 区间和查询：sum[l..r]
    long long query(int l, int r) const {
        if (l > r) return 0;
        return query(r) - query(l - 1);
    }

    // 初始化：从已有数组构建 BIT
    void build(const vector<long long>& arr) {
        for (int i = 1; i <= n; ++i) {
            bit[i] = arr[i];
            int j = i + (i & -i);
            if (j <= n) bit[j] += bit[i];
        }
    }
};
```

**使用示例**：

```cpp
int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    int n, m;
    cin >> n >> m;
    Fenwick fw(n);

    vector<long long> a(n + 1);
    for (int i = 1; i <= n; ++i) {
        cin >> a[i];
    }
    fw.build(a);

    // 演示单点更新和查询
    fw.update(3, 5);    // A[3] += 5
    cout << fw.query(1, 5) << endl;  // 查询 [1,5] 的和

    return 0;
}
```

---

## 5. 树状数组 vs 线段树

| 特征 | 树状数组 | 线段树 |
|------|---------|--------|
| 时间复杂度（查询/更新） | $O(\log n)$ | $O(\log n)$ |
| 空间复杂度 | $O(n)$ | $O(2n)$（通常 $4n$） |
| 功能丰富度 | 仅支持前缀和 | 支持区间最值、区间修改等 |
| 实现难度 | 较简单 | 中等 |
| 常数因子 | 较小 | 较大 |

### 选择建议

- 只需**前缀和 + 单点更新**：优先选择**树状数组**，常数更小，实现更简洁。
- 需要**区间最值**、**区间修改**、**复杂聚合**：选择[[segment-tree|线段树]]。
- 二者的时间复杂度均为 $O(\log n)$，但树状数组的常数通常是线段树的 $\frac{1}{2} \sim \frac{1}{3}$。[^1]

---

## 6. 扩展应用

### 6.1 查找第 k 小（或第 k 大）元素

利用 BIT 的二分查找性质，可以在 $O(\log n)$ 时间内找到第 $k$ 小元素：

```cpp
// 在值域 [1, maxVal] 中查找第 k 小的数
int kth(int k) {
    int idx = 0;
    int bitMask = 1 << (31 - __builtin_clz(n));  // 找到不超过 n 的最大 2 的幂

    for (int step = bitMask; step > 0; step >>= 1) {
        int next = idx + step;
        if (next <= n && bit[next] < k) {
            idx = next;
            k -= bit[next];
        }
    }
    return idx + 1;
}
```

**应用场景**：维护动态序列的第 $k$ 小值，支持插入和删除。

### 6.2 频率表操作

树状数组天然适合处理**频率统计**问题：

```cpp
// 统计 ≤ x 的元素个数
int countLessOrEqual(int x) {
    return query(x);
}

// 统计 [l, r] 范围内的元素个数
int countRange(int l, int r) {
    return query(r) - query(l - 1);
}
```

### 6.3 二维树状数组

将一维 BIT 扩展到二维，支持 $O(\log^2 n)$ 的矩形查询和单点更新：

```cpp
class Fenwick2D {
private:
    int n, m;
    vector<vector<long long>> bit;

public:
    Fenwick2D(int n, int m) : n(n), m(m), bit(n + 1, vector<long long>(m + 1, 0)) {}

    void update(int x, int y, long long delta) {
        for (int i = x; i <= n; i += i & -i) {
            for (int j = y; j <= m; j += j & -j) {
                bit[i][j] += delta;
            }
        }
    }

    long long query(int x, int y) {
        long long res = 0;
        for (int i = x; i > 0; i -= i & -i) {
            for (int j = y; j > 0; j -= j & -j) {
                res += bit[i][j];
            }
        }
        return res;
    }

    // 矩形查询 [1,1] 到 [x,y]
    long long queryRect(int x1, int y1, int x2, int y2) {
        return query(x2, y2) - query(x1 - 1, y2) 
             - query(x2, y1 - 1) + query(x1 - 1, y1 - 1);
    }
};
```

**典型应用**：二维平面上的动态点计数、图像处理中的前缀和计算。

---

## 7. 相关主题

- [[segment-tree|线段树]]：更通用的区间数据结构
- [[heap|堆]]：另一种重要的优先队列实现，用于 [[../algorithms/dijkstra|Dijkstra 算法]] 等场景

---

## 8. 参考资料

[^1]: 本篇内容参考了 [树状数组 - OI Wiki](https://oi-wiki.org/ds/fenwick/)。
