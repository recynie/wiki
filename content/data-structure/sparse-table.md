---
title: ST表（稀疏表）
date: 2026-04-03
description: 稀疏表（Sparse Table）是一种用于处理静态区间查询的数据结构，支持 O(1) 的区间最值查询
tags:
  - sparse-table
  - rmq
  - data-structure
draft: false
permalink:
---

## 定义

稀疏表（Sparse Table）是一种用于处理**静态**区间查询的数据结构。它通过预处理将区间信息存储在二维表中，使得区间查询可以在 O(1) 时间内完成。

稀疏表的核心思想是：**将区间 $[l, r]$ 拆分为两个有重叠的子区间，利用幂次大小的区间覆盖整个查询范围**。

## 可重复贡献问题

稀疏表适用于**可重复贡献问题**（Idempotent/Repeatable Contribution Problems），即满足以下条件的问题：

> 对于操作 $\ op$，若满足 $x \ op \ x = x$，则该操作具有**幂等性**；若满足 $a \ op \ b = b \ op \ a$（交换律），则可以处理重叠区间的贡献。

常见满足可重复贡献的操作：
- $\max()$、$\min()$
- $\gcd()$
- $\text{AND}$、$\text{OR}$（位运算）

### 关键性质

若区间查询操作 $\ op$ 满足**结合律**和**幂等性**（或交换律），则：

$$
ans(l, r) = ans(l, l+2^k-1) \ op \ ans(r-2^k+1, r)
$$

其中 $k = \lfloor \log_2(r-l+1) \rfloor$。

## RMQ（区间最值查询）

RMQ（Range Minimum/Maximum Query）是稀疏表最经典的应用：给定数组，求区间 $[l, r]$ 的最大值或最小值。

### 预处理

设 $f[i][j]$ 表示区间 $[i, i+2^j-1]$ 的最值，则：

$$
f[i][j] = \max/min(f[i][j-1],\ f[i+2^{j-1}][j-1])
$$

```cpp
// 预处理 log2 数组
log2[1] = 0;
for (int i = 2; i <= n; i++) {
    log2[i] = log2[i/2] + 1;
}

// 预处理稀疏表
for (int i = 1; i <= n; i++) {
    f[i][0] = a[i];  // 区间长度为 1
}
for (int j = 1; j <= LOG; j++) {
    for (int i = 1; i + (1<<j) - 1 <= n; i++) {
        f[i][j] = max(f[i][j-1], f[i + (1<<(j-1))][j-1]);
    }
}
```

### 查询 O(1)

对于区间 $[l, r]$，设 $k = \log_2(r-l+1)$，则：

$$
\text{RMQ}(l, r) = \max(f[l][k],\ f[r-2^k+1][k])
$$

```cpp
int query(int l, int r) {
    int k = log2[r - l + 1];
    return max(f[l][k], f[r - (1<<k) + 1][k]);
}
```

> **注意**：两个区间 $[l, l+2^k-1]$ 和 $[r-2^k+1, r]$ 有重叠，但 $\max$ 操作是幂等的，所以重叠不影响结果。

## 应用场景

| 应用 | 操作 | 幂等性 |
|------|------|--------|
| 区间最小值 | $\min$ | ✅ |
| 区间最大值 | $\max$ | ✅ |
| 区间最大公约数 | $\gcd$ | ✅（$\gcd(x,x)=x$）|
| 区间按位与 | $\text{AND}$ | ✅ |
| 区间按位或 | $\text{OR}$ | ✅ |

## C++ 代码模板

```cpp
#include <bits/stdc++.h>
using namespace std;

class SparseTable {
private:
    int n, LOG;
    vector<int> log2;                    // log2[i] = floor(log2(i))
    vector<vector<int>> st;             // st[i][j] = min/max in [i, i+2^j-1]

public:
    SparseTable(const vector<int>& a) {
        n = a.size() - 1;                // a[1..n]
        LOG = log2[n] + 1;
        log2.resize(n + 1);
        st.assign(n + 1, vector<int>(LOG));

        // 预处理 log2
        log2[1] = 0;
        for (int i = 2; i <= n; i++) {
            log2[i] = log2[i/2] + 1;
        }

        // 预处理稀疏表（以最小值为例，max 类似）
        for (int i = 1; i <= n; i++) {
            st[i][0] = a[i];
        }
        for (int j = 1; j < LOG; j++) {
            for (int i = 1; i + (1<<j) - 1 <= n; i++) {
                st[i][j] = min(st[i][j-1], st[i + (1<<(j-1))][j-1]);
            }
        }
    }

    // 区间 [l, r] 的最小值查询
    int query_min(int l, int r) {
        int k = log2[r - l + 1];
        return min(st[l][k], st[r - (1<<k) + 1][k]);
    }

    // 区间 [l, r] 的最大值查询
    int query_max(int l, int r) {
        int k = log2[r - l + 1];
        return max(st[l][k], st[r - (1<<k) + 1][k]);
    }
};

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    int n, q;
    cin >> n >> q;
    vector<int> a(n + 1);
    for (int i = 1; i <= n; i++) {
        cin >> a[i];
    }

    SparseTable st(a);

    while (q--) {
        int l, r;
        cin >> l >> r;
        cout << st.query_min(l, r) << '\n';
    }

    return 0;
}
```

### 复杂度分析

| 操作 | 时间复杂度 | 空间复杂度 |
|------|-----------|-----------|
| 预处理 | $O(n \log n)$ | $O(n \log n)$ |
| 单次查询 | $O(1)$ | — |

## 与线段树的比较

| 特性 | 稀疏表 | 线段树 |
|------|--------|--------|
| 查询复杂度 | $O(1)$ | $O(\log n)$ |
| 预处理复杂度 | $O(n \log n)$ | $O(n)$ |
| 空间复杂度 | $O(n \log n)$ | $O(n)$ |
| 支持更新 | ❌ 静态 | ✅ 动态 |
| 适用操作 | 幂等操作（min/max/gcd） | 任意操作 |
| 灵活性 | 低（仅静态查询） | 高（支持单点/区间修改） |

### 选择建议

- **选择稀疏表**：数据**静态**（无修改），查询次数极多，需要 O(1) 查询
- **选择线段树**：需要**动态修改**，或操作不满足幂等性（如区间和、区间加法）

## 总结

稀疏表是一种高效的静态区间查询数据结构，利用幂等操作的性质，通过 O(1) 时间完成区间最值查询。预处理 O(n log n) 空间 O(n log n)，适合查询密集型场景。

## 参考资料

[^1]: [ST 表 - OI Wiki](https://oi-wiki.org/ds/sparse-table/)

[^2]: [RMQ 问题 - OI Wiki](https://oi-wiki.org/ds/rmq/)
