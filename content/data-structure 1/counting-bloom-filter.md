---
title: Counting Bloom Filter
date: 2026-04-13
description: 计数布隆过滤器（Counting Bloom Filter）用计数器数组代替标准布隆过滤器的比特数组，支持元素的删除操作。
tags:
  - bloom-filter
  - data-structure
draft: true
permalink:
---

## 概述

Counting Bloom Filter（计数布隆过滤器）是标准布隆过滤器的变体，用计数器数组替代比特数组，从而支持元素的删除操作。[^1]

在标准布隆过滤器中，每个槽位只有 0/1 两种状态，无法实现删除。而计数布隆过滤器每个槽位使用多个比特（通常为 4 比特）存储计数器，可以对插入次数进行计数。

## 工作原理

### 数据结构

假设有 $k$ 个哈希函数，位向量长度为 $m$：

- 标准布隆过滤器：$B[0..m-1]$，每个元素为 1 bit
- 计数布隆过滤器：$C[0..m-1]$，每个元素为 $c$ bits（通常 $c=4$）

### 基本操作

**插入操作**

当插入元素 $x$ 时，对每个哈希函数 $h_i(x)$ 对应的计数器执行 +1：

$$
C[h_i(x)] = C[h_i(x)] + 1, \quad i \in [1, k]
$$

**查询操作**

查询元素是否存在时，检查所有对应计数器是否大于 0：

$$
\text{exists} = \forall i \in [1, k], \ C[h_i(x)] > 0
$$

**删除操作**

当删除元素 $x$ 时，对每个哈希函数对应的计数器执行 -1（保证计数器不为负）：

$$
C[h_i(x)] = \max(0, C[h_i(x)] - 1), \quad i \in [1, k]
$$

## 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

class CountingBloomFilter {
private:
    vector<uint8_t> counters;  // 每个计数器使用8比特
    int m;                      // 位向量长度
    int k;                      // 哈希函数数量
    vector<size_t> seeds;       // 哈希种子
    
    size_t hash(int idx, const string& item) {
        hash<string> h;
        return (h(item) + idx * 1315423911) % m;
    }
    
public:
    CountingBloomFilter(int m, int k) : m(m), k(k), counters(m, 0) {
        random_device rd;
        for (int i = 0; i < k; ++i) {
            seeds.push_back(rd() % 1000000);
        }
    }
    
    void insert(const string& item) {
        for (int i = 0; i < k; ++i) {
            size_t pos = hash(i, item);
            if (counters[pos] < 255) {  // 防止溢出
                counters[pos]++;
            }
        }
    }
    
    bool contains(const string& item) const {
        for (int i = 0; i < k; ++i) {
            size_t pos = hash(i, item);
            if (counters[pos] == 0) {
                return false;
            }
        }
        return true;
    }
    
    void remove(const string& item) {
        for (int i = 0; i < k; ++i) {
            size_t pos = hash(i, item);
            if (counters[pos] > 0) {
                counters[pos]--;
            }
        }
    }
};
```

## 关键特性

### 优点

- **支持删除操作**：相比标准布隆过滤器，可以删除已插入的元素
- **动态计数**：可以获取元素的插入次数（近似值）

### 缺点与风险

- **计数器溢出（Counter Overflow）**：当某个槽位被频繁哈希时，计数器可能溢出，导致假阴性
- **空间开销更大**：每个槽位需要多个比特（标准实现用 4 bits，约为标准布隆过滤器的 4 倍）
- **假阳性率更高**：相同空间下，计数布隆过滤器的假阳性率高于标准布隆过滤器

| 特性 | 标准布隆过滤器 | 计数布隆过滤器 |
|------|---------------|---------------|
| 空间复杂度 | $O(m)$ | $O(m \cdot c)$ |
| 删除支持 | ❌ | ✅ |
| 计数器溢出风险 | 无 | 有 |
| 假阳性率 | 较低 | 较高 |

## 应用场景

计数布隆过滤器适用于需要动态删除元素的场景：

- **缓存系统**：如 Redis 的 `BF.REMOVER` 操作
- **数据库查询优化**：减少磁盘 I/O 操作
- **网络安全**：IP 地址过滤与撤销
- **分布式系统**：协调节点间的数据同步

## 与标准布隆过滤器的比较

```cpp
// 标准布隆过滤器
BloomFilter bf(10000, 7);
bf.insert("item1");
bf.contains("item1");  // true
bf.remove("item1");   // 无法实现！

// 计数布隆过滤器
CountingBloomFilter cbf(10000, 7);
cbf.insert("item1");
cbf.contains("item1");  // true
cbf.remove("item1");    // true，元素被删除
cbf.contains("item1");  // false
```

## 参考

[^1]: 计数布隆过滤器由 Fan 等人在 2000 年提出，用于替代标准布隆过滤器支持删除操作。
