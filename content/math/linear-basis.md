---
title: 线性基
date: 2026-04-05
description: 线性基（Linear Basis）用于处理XOR相关问题的向量空间结构
tags:
  - linear-basis
  - xor
  - bit-manipulation
  - linear-algebra
draft: false
permalink:
---

## 定义

线性基（Linear Basis）是一种用于处理**异或（XOR）**相关问题的向量空间结构，本质上是向量空间 $GF(2)^n$（二元域）中的一组基[^1]。

在线性基中，我们选择一组**线性无关**的向量，使得集合中任意元素的异或和都可以唯一表示为这组基向量的线性组合。

### 基本概念

- **向量空间**：在二元域 $GF(2)$ 上，所有 $n$ 位二进制数构成一个向量空间
- **基向量**：线性无关的向量组，可以生成整个空间
- **秩**：线性基中基向量的个数，记为 $r$，满足 $r \leq n$

### 核心性质

1. **线性无关**：基中任意非空子集的异或和都不为 $0$
2. **秩的性质**：基的秩 $r \leq$ 原始元素个数 $n$
3. **唯一表示**：任意向量可以唯一表示为基向量的线性组合
4. **上三角结构**：构造时保持最高位递减，便于贪心操作

## 构造算法

### 核心思想

将每个数插入线性基中，类比高斯消元的二进制版本。维护一个**上三角矩阵**，对角线上是各基向量的最高有效位[^2]。

### 插入操作（add）

```cpp
void add(long long x) {
    for (int i = 63; i >= 0; i--) {
        if (!(x >> i & 1)) continue;
        if (!basis[i]) {
            basis[i] = x;
            break;
        }
        x ^= basis[i];
    }
}
```

### 构建完整基

```cpp
LinearBasis(vector<long long> a) {
    basis.assign(64, 0);
    for (long long x : a) add(x);
    // 进一步消元，得到上三角形式
    for (int i = 63; i > 0; i--) {
        if (basis[i]) {
            for (int j = i - 1; j >= 0; j--) {
                if (basis[j]) basis[j] ^= basis[i];
            }
        }
    }
}
```

## 代码实现

### 完整类实现

```cpp
struct LinearBasis {
    vector<long long> basis;
    int n;  // 秩
    
    LinearBasis(int maxbit = 63) : basis(maxbit + 1, 0), n(0) {}
    
    void add(long long x) {
        for (int i = 63; i >= 0; i--) {
            if (!(x >> i & 1)) continue;
            if (!basis[i]) {
                basis[i] = x;
                n++;
                break;
            }
            x ^= basis[i];
        }
    }
    
    // 最大异或和
    long long getMax() {
        long long res = 0;
        for (int i = 63; i >= 0; i--) {
            if ((res ^ basis[i]) > res) {
                res ^= basis[i];
            }
        }
        return res;
    }
    
    // 查询能否得到特定值
    bool query(long long x) {
        for (int i = 63; i >= 0; i--) {
            if (x >> i & 1) {
                if (!basis[i]) return false;
                x ^= basis[i];
            }
        }
        return x == 0;
    }
    
    // k-th最大异或和（需先排序基向量）
    long long kthMax(long long k) {
        vector<long long> v;
        for (int i = 0; i <= 63; i++) {
            if (basis[i]) v.push_back(basis[i]);
        }
        sort(v.begin(), v.end());
        long long res = 0;
        for (int i = v.size() - 1; i >= 0; i--) {
            if (k >> (v.size() - 1 - i) & 1) {
                res ^= v[i];
            }
        }
        return res;
    }
};
```

### 使用示例

```cpp
int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n;
    cin >> n;
    vector<long long> a(n);
    for (int i = 0; i < n; i++) cin >> a[i];
    
    LinearBasis lb;
    for (long long x : a) lb.add(x);
    
    cout << "最大异或和: " << lb.getMax() << endl;
    
    long long k;
    cin >> k;
    if (k < (1LL << lb.n)) {
        cout << "第k大: " << lb.kthMax(k) << endl;
    }
    
    long long x;
    cin >> x;
    cout << (lb.query(x) ? "YES" : "NO") << endl;
    
    return 0;
}
```

## 典型应用

### 1. 最大异或和

从高到低遍历基向量，若异或后变大则取该基向量[^3]：

```cpp
long long maxXor = lb.getMax();
```

### 2. 判断能否得到特定值

将目标值在基向量构成的矩阵上做消元，无法继续消元时若剩余 $0$ 则可表示。

### 3. 第 k 小/大异或和

将基向量按最高位排序后，k 的二进制表示决定选取哪些基向量。

### 4. 区间最大异或和

> 参见 [[../data-structure/trie|字典树]] 的应用场景，线性基可处理静态区间最大异或问题。

## 与位运算的关系

线性基本质上是 [[../algorithms/bit-manipulation|位运算]] 的高级应用，利用了 XOR 运算在 $GF(2)$ 上的向量空间性质。

### 位运算基础

- XOR 满足交换律、结合律
- $a \oplus a = 0$，$a \oplus 0 = a$
- 最高位为 $1$ 表示该位对异或和有贡献

### 扩展应用

线性基常与**字典树**（[[../data-structure/trie|字典树]]）结合使用：
- 字典树处理动态插入/查询
- 线性基处理静态集合的全局最优解

## 例题

### 最大异或和（模板题）

**题目**：给定 $n$ 个整数 $(1 \leq n \leq 10^5)$，选出若干数使得异或和最大，输出最大异或和。

**思路**：将所有数插入线性基，贪心选取使结果变大的基向量。

**复杂度**：$O(n \cdot \log A)$，其中 $A$ 为数值的最大值。

### Code

```cpp
#include <bits/stdc++.h>
using namespace std;

struct LinearBasis {
    vector<long long> basis;
    LinearBasis(int bits = 63) : basis(bits + 1, 0) {}
    
    void add(long long x) {
        for (int i = 63; i >= 0; i--) {
            if (!(x >> i & 1)) continue;
            if (!basis[i]) {
                basis[i] = x;
                return;
            }
            x ^= basis[i];
        }
    }
    
    long long getMax() {
        long long res = 0;
        for (int i = 63; i >= 0; i--) {
            if ((res ^ basis[i]) > res) {
                res ^= basis[i];
            }
        }
        return res;
    }
};

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n;
    cin >> n;
    LinearBasis lb;
    for (int i = 0; i < n; i++) {
        long long x;
        cin >> x;
        lb.add(x);
    }
    
    cout << lb.getMax() << endl;
    return 0;
}
```

---

## 参考资料

[^1]: 向量空间与线性基的理论基础可参考任意线性代数教材
[^2]: 构造算法参考 [CP-Algorithms: Linear Basis](https://cp-algorithms.com/linear_algebra/linear-basis.html)
[^3]: USACO Guide: XOR Basis, https://usaco.guide/gold/xor-basis