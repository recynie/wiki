---
title: Cache Optimization
date: 2026-04-13
description: CPU缓存层次结构与缓存无关算法
tags:
  - cache
  - optimization
  - algorithm
draft: true
permalink:
---

## CPU缓存层次结构

### 缓存层级与Cache Line

现代CPU采用分层缓存架构，通常包含三级：

| 层级 | 典型大小 | 访问延迟 | 共享方式 |
|------|----------|----------|----------|
| L1 | 32KB-64KB | 1-2周期 | 每核独有 |
| L2 | 256KB-1MB | 3-5周期 | 每核独有或共享 |
| L3 | 8MB-64MB | 10-20周期 | 核间共享 |

**Cache Line** 是CPU与内存交互的最小单位，通常为 **64字节**。当CPU访问一个字节时，实际上会将该字节所在的整个cache line加载到缓存中。

### 缓存关联性

缓存映射方式决定了如何将内存地址映射到缓存槽位：

**直接映射（Direct-Mapped）**：每个内存块只能映射到唯一一个缓存槽位
$$
\text{Cache Index} = \text{Memory Address} \bmod \text{Number of Cache Lines}
$$

**N路组相联（N-Way Set Associative）**：每个槽位组有N个可用位置
$$
\text{Set Index} = (\text{Memory Address} / \text{Line Size}) \bmod \text{Number of Sets}
$$

现代CPU的L1/L2缓存通常采用8路或16路组相联。

### 缓存未命中类型

**强制未命中（Compulsory Miss）**：首次访问数据时必定发生，也称为冷启动未命中。

**容量未命中（Capacity Miss）**：缓存容量不足导致已加载的数据被替换。发生条件：
$$
\text{working set size} > \text{cache capacity}
$$

**冲突未命中（Conflict Miss）**：由于关联性限制，同一映射组的多个数据相互驱逐。

总未命中数：
$$
M_{\text{total}} = M_{\text{compulsory}} + M_{\text{capacity}} + M_{\text{conflict}}
$$

### 缓存替换策略

**LRU（Least Recently Used）**：替换最久未使用的数据。实现成本高，硬件通常采用近似算法。

**FIFO（First In First Out）**：替换最早进入缓存的数据。实现简单但可能替换仍需使用的热点数据。

**随机替换（Random）**：随机选择被替换的行。实现最简单，某些场景下实际表现良好。

### 伪共享（False Sharing）

在多线程程序中，不同线程修改同一cache line上的不同变量会导致性能急剧下降。[^1]

```cpp
// 伪共享示例
struct alignas(64) Counter {
    int64_t count_a;  // 线程0修改
    int64_t count_b;  // 线程1修改
};
// 虽然count_a和count_b独立，但共享同一cache line
// 线程0修改count_a时，线程1的count_b所在cache line被无效化
```

避免伪共享的方法：
- 使用 `alignas(64)` 确保数据结构按cache line对齐
- 将频繁更新的变量分离到不同的cache line
- 使用线程局部变量，最后再合并结果

---

## 缓存无关算法

### 核心思想

缓存无关算法的设计目标是：**算法无需知道缓存参数M和块大小B，在各级缓存层次都能表现优异**。[^2]

传统算法分析使用RAM模型，忽略缓存层次；而缓存敏感算法需要针对特定M和B进行优化。缓存无关算法通过递归分治策略自动适配所有缓存层级。

### 缓存复杂度

定义：
- $M$：缓存大小（能容纳的元素数）
- $B$：块大小（每块能容纳的元素数）
- $Q(N, M, B)$：处理N个元素的缓存未命中次数

理想缓存模型（Ideal Cache Model）的复杂度：
$$
Q(N, M, B) = \Theta\left(\frac{N}{B} \cdot \left\lceil \log_M \frac{N}{B} \right\rceil + \frac{N}{M} \right)
$$

### 经典例子：矩阵乘法

**朴素矩阵乘法**：时间复杂度 $O(N^3)$，缓存未命中次数为 $O(N^3)$。

```cpp
// 朴素实现 - 缓存不友好
void naive_matrix_mul(const vector<vector<double>>& A,
                      const vector<vector<double>>& B,
                      vector<vector<double>>& C, int n) {
    for (int i = 0; i < n; i++)
        for (int j = 0; j < n; j++)
            for (int k = 0; k < n; k++)
                C[i][j] += A[i][k] * B[k][j];
}
```

**分块矩阵乘法**：通过增加数据局部性减少缓存未命中。

```cpp
// 分块实现 - 缓存友好
void blocked_matrix_mul(const vector<vector<double>>& A,
                        const vector<vector<double>>& B,
                        vector<vector<double>>& C, int n, int block_size) {
    for (int i = 0; i < n; i += block_size)
        for (int j = 0; j < n; j += block_size)
            for (int k = 0; k < n; k += block_size)
                // 处理每个小分块
                for (int ii = i; ii < min(i + block_size, n); ii++)
                    for (int jj = j; jj < min(j + block_size, n); jj++)
                        for (int kk = k; kk < min(k + block_size, n); kk++)
                            C[ii][jj] += A[ii][kk] * B[kk][jj];
}
```

设块大小为 $B \times B$，则缓存复杂度优化为：
$$
Q(N, M, B) = \Theta\left(\frac{N^3}{B \sqrt{M}}\right) \quad \text{当 } M \geq B^2
$$

### Tall Cache与Short Cache

**Tall Cache Assumption**：缓存高度（深度）远大于宽度：
$$
M / B \gg B
$$
即缓存能容纳的块数远大于每块的元素数。多数现代CPU满足此假设。

**Short Cache Assumption**：缓存接近方形：
$$
M / B \approx B
$$

不同假设下，矩阵乘法的最优分块策略不同。缓存无关算法通过递归分解自动适应这两种情况。

### Van Emde Boas布局

Van Emde Boas（VEB）布局是一种递归的树形内存布局，特别适合缓存无关算法。[^3]

对于大小为N的数组，VEB布局将数组分为：
$$
\sqrt{N} \text{ 个大小为 } \sqrt{N} \text{ 的子数组}
$$
递归地，每个子数组也采用相同的布局方式。

```
数组索引:     0    1    2    3    4    5    6    7
第一级分组: [ 0    1 ] [ 2    3 ] [ 4    5 ] [ 6    7 ]
第二级分组: [0][1] [2][3] [4][5] [6][7]
内存布局:    0 1 2 3 4 5 6 7  (按此顺序访问)
```

VEB布局的优点：
- 任何大小为 $N$ 的子数组在内存中连续
- 支持递归访问模式，适合分治算法
- 缓存复杂度为 $\Theta\left(\frac{N}{B}\right)$ 而非 $O\left(\frac{N}{B} \log_M \frac{N}{B}\right)$

### 快速傅里叶变换（FFT）

缓存无关的FFT算法通过Cooley-Tukey的递归分解实现：

```cpp
// 缓存无关FFT框架
void cache_oblivious_fft(complex<double>* A, int n, complex<double>* output) {
    if (n == 1) {
        output[0] = A[0];
        return;
    }
    
    // 递归分治，数据自然按cache line对齐
    cache_oblivious_fft(A, n/2, output);           // 偶数位
    cache_oblivious_fft(A + n/2, n/2, output + n/2); // 奇数位
    
    // 蝶形运算合并
    // ...
}
```

缓存复杂度分析：
$$
Q_{\text{FFT}}(N, M, B) = O\left(\frac{N}{B} + \frac{N}{M} \log_2 \frac{N}{M}\right)
$$

---

## 参考资料

[^1]: False Sharing - Wikipedia. https://en.wikipedia.org/wiki/False_sharing

[^2]: Frigo, M., Leiserson, C. E., Prokop, H., & Ramachandran, S. (1999). Cache-oblivious algorithms. Proceedings of the 40th Annual Symposium on Foundations of Computer Science.

[^3]: Prokop, H. (1999). Cache-oblivious algorithms. Master's thesis, MIT.
