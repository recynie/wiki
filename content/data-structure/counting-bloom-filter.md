---
title: Counting Bloom Filter 与 Scalable Bloom Filter
date: 2026-04-13
description: 计数布隆过滤器和可扩展布隆过滤器的原理、实现及应用
tags:
  - probabilistic-data-structure
  - data-structure
  - bloom-filter
draft: false
permalink:
---

## 1. 概述

在[[bloom-filter|布隆过滤器（Bloom Filter）]]的基本形式中，位数组的每一位只能是 0 或 1，这使得布隆过滤器**不支持删除操作**。删除一个元素可能会影响其他元素的查询结果，因为多个元素可能共享相同的比特位。

本文介绍两种重要的布隆过滤器变体：

| 变体 | 核心改进 | 主要应用场景 |
|------|----------|--------------|
| **计数布隆过滤器**（CBF） | 用计数器替代比特，支持删除 | 需要动态添加/删除元素的场景 |
| **可扩展布隆过滤器**（SBF） | 动态扩容，支持无限增长 | 容量未知或元素数量剧增的场景 |

---

## 2. 计数布隆过滤器（Counting Bloom Filter）

### 2.1 定义与动机

计数布隆过滤器（Counting Bloom Filter，简称 CBF）由 Fan 等人于 2000 年提出[^1]，核心思想是将标准布隆过滤器中的**位数组替换为计数数组**。

```
标准布隆过滤器:  bit[i] ∈ {0, 1}
计数布隆过滤器:  count[i] ∈ {0, 1, 2, 3, ..., MAX}
```

每个计数器记录对应位置被多少个元素"击中"。通过检查计数器的值是否大于 0 来判断元素是否存在，通过增减计数器来实现元素的插入和删除。

### 2.2 数据结构

计数布隆过滤器的核心组件：

| 组件 | 说明 |
|------|------|
| **计数数组** | 长度为 $m$ 的计数器数组，初始时均为 0 |
| **计数器宽度** | 每个计数器占用的位数（通常为 3-4 位） |
| **哈希函数** | $k$ 个相互独立的哈希函数 |

### 2.3 基本操作

#### 2.3.1 插入（Add）

将元素 $x$ 插入计数布隆过滤器：

```
对于每个哈希函数 hi (1 ≤ i ≤ k):
    pos = hi(x)
    如果 count[pos] < MAX:
        count[pos]++
```

#### 2.3.2 查询（Query）

查询元素 $x$ 是否存在：

```
对于每个哈希函数 hi (1 ≤ i ≤ k):
    pos = hi(x)
    如果 count[pos] == 0:
        返回 false（元素一定不存在）
返回 true（元素可能存在）
```

#### 2.3.3 删除（Delete）

从计数布隆过滤器中删除元素 $x$：

```
对于每个哈希函数 hi (1 ≤ i ≤ k):
    pos = hi(x)
    如果 count[pos] > 0:
        count[pos]--
```

### 2.4 假阳性与假阴性分析

#### 2.4.1 假阳性（False Positive）

与标准布隆过滤器类似，查询可能返回"可能存在"但实际不存在。假阳性概率分析相同：

$$
P_{\text{FP}} \approx \left(1 - e^{-kn/m}\right)^k
$$

其中 $n$ 是插入元素数，$m$ 是计数器数组长度，$k$ 是哈希函数数量。

#### 2.4.2 假阴性（False Negative）

**这是计数布隆过滤器引入的新问题**：查询返回"不存在"但元素实际存在。

假阴性发生在以下情况：

```
场景：元素 x 被插入后又被删除
元素 x 插入时：count[pos]++
元素 x 删除时：count[pos]--（如果此时 count[pos] > 0）

如果 x 是唯一映射到这些位置的元素，且删除后计数器变为 0，
则查询 x 会返回 false（假阴性）
```

#### 2.4.3 计数器溢出

当多个元素映射到相同位置时，计数器可能溢出：

```
假设 counter_bits = 3，则 MAX = 7

插入第 8 个映射到同一位置的元素时，count[pos] 仍为 7
删除一个元素后，count[pos] 变为 6
此时查询可能错误地认为该位置"从未被使用"
```

**解决方案**：

- 使用更大的计数器位数（如 4-5 位）
- 或在删除时检测到计数器已处于最大值，不进行递减

### 2.5 空间分析

标准布隆过滤器使用 1 bit per slot，而计数布隆过滤器使用 $c$ bits per counter（$c$ 为计数器位数）。

| 计数器位数 | 空间开销 | 可表示最大值 | 溢出风险 |
|-----------|---------|-------------|---------|
| 3 bits | 3x | 7 | 较高 |
| 4 bits | 4x | 15 | 中等 |
| 5 bits | 5x | 31 | 较低 |

设元素数为 $n$，标准布隆过滤器的最优空间为 $m = \frac{n \ln P}{\ln 2^2}$ bits。

使用 $c$ 位计数器的计数布隆过滤器空间为 $m \cdot c$ bits。

### 2.6 应用场景

#### 2.6.1 网络路由器

在网络路由器中，流量统计需要追踪活跃的数据流。当数据流结束时，需要从过滤器中删除记录。

```
应用示例：网络入侵检测系统（IDS）
- 新数据包到达 → 插入流记录
- 流超时或结束 → 删除流记录
- 查询流是否存在 → 判断是否为已知攻击
```

#### 2.6.2 缓存淘汰策略

在 LFU（Least Frequently Used）缓存中，需要追踪每个缓存项的访问频率。

```
LFU 缓存实现思路：
- 将计数布隆过滤器与缓存结合
- 插入缓存项时增加计数器
- 淘汰时选择计数器值最小的项
```

#### 2.6.3 分布式系统

在分布式消息队列中，需要追踪已处理的消息以实现去重和确认机制。

```
消息处理流程：
1. 消息生产者发送消息
2. 消费者处理消息后插入 CBF
3. 消费者崩溃重启后，通过 CBF 判断哪些消息已处理
4. 定时清理已确认的消息（删除操作）
```

---

## 3. 可扩展布隆过滤器（Scalable Bloom Filter）

### 3.1 问题定义

标准布隆过滤器的容量在创建时固定，当插入元素数量超过预期时：

1. **假阳性率急剧上升**：位数组过于拥挤
2. **无法扩容**：不能简单地增加位数，否则历史数据失效

可扩展布隆过滤器（Scalable Bloom Filter，简称 SBF）由 Almeida 等人于 2007 年提出[^2]，通过动态扩容解决这一问题。

### 3.2 核心思想

SBF 维护一个**递增的布隆过滤器序列** $BF_1, BF_2, \ldots, BF_t$：

```
SBF 结构示意：

┌─────────────────────────────────────────────────┐
│  BF_1 (容量已满)    BF_2 (容量已满)    BF_3 (当前) │
│  ┌─────────┐       ┌─────────┐       ┌─────────┐ │
│  │ m bits  │       │ r·m bits│       │ r²·m bits│ │
│  │ p₁      │       │ p₂      │       │ p₃       │ │
│  └─────────┘       └─────────┘       └─────────┘ │
│        ↑               ↑               ↑       │
│   插入的元素      扩容后插入      新插入的元素    │
└─────────────────────────────────────────────────┘
```

**关键参数**：

| 参数 | 说明 |
|------|------|
| $r$ | 扩展因子（Tightening Factor），通常取 0.8-0.9 |
| $s_i$ | 第 $i$ 个过滤器的位数组大小 |
| $p_i$ | 第 $i$ 个过滤器的假阳性率 |

### 3.3 基本操作

#### 3.3.1 插入（Add）

将元素 $x$ 插入 SBF：

```
算法：SBF-Insert(x)
1. 计算 h = Hash(x)
2. 如果 h < f(t)（元素应属于前 t 个过滤器之一）:
       在 BF_h 中插入 x
   否则:
       在当前 BF_t 中插入 x
       如果 BF_t 装满:
           创建新 BF_{t+1}
           f(t+1) = f(t) + 剩余错误预算
```

**元素所属过滤器**的计算：

$$
f(i) = \sum_{j=1}^{i} \frac{p_j}{p} \quad \text{（累积错误预算）}
$$

其中 $p$ 是初始设定的总体假阳性率。

#### 3.3.2 查询（Query）

查询元素 $x$ 是否存在：

```
算法：SBF-Query(x)
对于每个过滤器 BF_i (i = 1 到 t):
    如果 BF_i 查询返回 true:
        返回 true
返回 false
```

### 3.4 假阳性率分析

设 SBF 包含 $t$ 个过滤器，第 $i$ 个过滤器的假阳性率为 $p_i$。

**总体假阳性率**：

$$
P_{\text{SBF}} = 1 - \prod_{i=1}^{t} (1 - p_i)
$$

当所有过滤器具有相同的假阳性率 $p$ 时：

$$
P_{\text{SBF}} = 1 - (1 - p)^t \approx tp \quad \text{（当 $p$ 很小时）}
$$

#### 3.4.1 扩展因子的影响

设初始过滤器容量为 $m_1$，扩展因子为 $r$，则第 $i$ 个过滤器容量为 $m_i = r^{i-1} \cdot m_1$。

为了保持总体假阳性率不变，每个后续过滤器的假阳性率需要降低：

$$
p_i = r \cdot p_{i-1}
$$

这确保了错误预算的均匀消耗。

### 3.5 与标准布隆过滤器的比较

| 特性 | 标准 BF | 可扩展 BF |
|------|---------|-----------|
| **容量** | 固定 | 动态增长 |
| **假阳性率** | 固定 | 随扩容略微上升 |
| **空间复杂度** | $O(n)$ | $O(n \cdot \frac{1}{1-r})$ |
| **查询复杂度** | $O(k)$ | $O(k \cdot t)$ |
| **实现复杂度** | 简单 | 中等 |

### 3.6 应用场景

#### 3.6.1 未知容量的数据存储

当无法预估数据规模时，SBF 提供了一种优雅的扩容机制。

```
应用示例：日志分析系统
- 初始只关注最近的日志
- 随着时间推移，日志量可能爆炸式增长
- SBF 自动扩容，无需手动干预
```

#### 3.6.2 分布式键值存储

在 DynamoDB 或 Cassandra 等分布式数据库中，使用 SBF 进行快速成员资格检测。

```
查询流程：
1. 客户端查询某个键是否存在
2. 协调节点查询本地 SBF
3. 根据结果决定是否需要访问底层存储
```

---

## 4. 可扩展计数布隆过滤器（Scalable CBF）

### 4.1 组合的挑战

将 SBF 与 CBF 组合面临一个核心问题：**如何知道一个元素属于哪个过滤器，从而正确删除它？**

```
问题示例：
- 元素 x 插入 BF_2
- 后续查询返回"可能存在"
- 需要删除 x，但不知道它在哪一个过滤器中

可能的解决方案：
1. 记录每个元素所属的过滤器
2. 在每个过滤器中存储元素签名
3. 定期合并或重建过滤器
```

### 4.2 一种实用方案

Scalable Counting Bloom Filter（SCBF）的一种实现思路：

```
1. 维护一个辅助数据结构（如哈希表）记录 "元素 → 过滤器编号"
2. 插入时：计算应插入的过滤器，记录映射
3. 查询时：遍历所有过滤器
4. 删除时：根据映射关系在对应过滤器中删除

代价：额外空间开销用于存储映射关系
```

### 4.3 使用场景

SCBF 适用于以下场景：

| 场景 | 说明 |
|------|------|
| **短期数据追踪** | 如滑动窗口内的数据去重 |
| **分层缓存系统** | 不同层级的缓存需要独立淘汰 |
| **分布式会话管理** | 用户会话的动态创建和销毁 |

---

## 5. 实现注意事项

### 5.1 计数器大小的选择

#### 5.1.1 理论分析

设每个计数器为 $c$ 位，则最大计数值为 $2^c - 1$。

对于 $n$ 个独立元素，每个位置被击中的次数服从二项分布 $B(n, k/m)$。

为确保计数器溢出概率极低，需满足：

$$
2^c - 1 \gg n \cdot \frac{k}{m}
$$

#### 5.1.2 经验建议

| 应用场景 | 推荐计数器位数 |
|---------|---------------|
| 一般用途 | 4 bits（最大计数值 15） |
| 高并发写入 | 5 bits（最大计数值 31） |
| 极端情况 | 8 bits（最大计数值 255） |

### 5.2 哈希函数选择

#### 5.2.1 MurmurHash3

MurmurHash3 是一种非加密哈希函数，具有良好的分布均匀性：

```cpp
uint64_t murmurHash3(const string& key, uint64_t seed = 0) const {
    const uint64_t c1 = 0xcc9e2d51ULL;
    const uint64_t c2 = 0x1b873593ULL;
    
    uint64_t h1 = seed;
    uint64_t len = key.length();
    const uint64_t* data = (const uint64_t*)key.data();
    uint64_t words = len / 8;
    
    for (uint64_t i = 0; i < words; ++i) {
        uint64_t k1 = data[i];
        k1 *= c1;
        k1 = (k1 << 31) | (k1 >> 33);
        k1 *= c2;
        h1 ^= k1;
        h1 = (h1 << 27) | (h1 >> 37);
        h1 = h1 * 5 + 0x52dce729;
    }
    
    // 处理剩余字节
    const uint8_t* tail = (const uint8_t*)(data + words);
    uint64_t k1 = 0;
    
    switch (len & 7) {
        case 7: k1 ^= uint64_t(tail[6]) << 48;
        case 6: k1 ^= uint64_t(tail[5]) << 40;
        case 5: k1 ^= uint64_t(tail[4]) << 32;
        case 4: k1 ^= uint64_t(tail[3]) << 24;
        case 3: k1 ^= uint64_t(tail[2]) << 16;
        case 2: k1 ^= uint64_t(tail[1]) << 8;
        case 1: k1 ^= uint64_t(tail[0]) << 0;
                k1 *= c1;
                k1 = (k1 << 31) | (k1 >> 33);
                k1 *= c2;
                h1 ^= k1;
    }
    
    h1 ^= len;
    h1 ^= h1 >> 33;
    h1 *= 0xff51afd7ed558ccdULL;
    h1 ^= h1 >> 33;
    h1 *= 0xc4ceb9fe1a85ec53ULL;
    h1 ^= h1 >> 33;
    
    return h1;
}
```

#### 5.2.2 FNV-1a

FNV-1a（Fowler-Noll-Vo）是另一种常用的字符串哈希算法：

```cpp
uint64_t fnv1a(const string& key) const {
    uint64_t hash = 14695981039346656037ULL;  // FNV offset basis
    for (unsigned char c : key) {
        hash ^= c;
        hash *= 1099511628211ULL;  // FNV prime
    }
    return hash;
}
```

#### 5.2.3 双重哈希

使用两个基础哈希函数生成 $k$ 个哈希值：

```cpp
// h(i) = h1 + i * h2
uint64_t getHash(const string& key, int i) const {
    uint64_t h1 = murmurHash3(key, 0);
    uint64_t h2 = murmurHash3(key, h1);
    return (h1 + i * h2) % m;
}
```

### 5.3 选择指南

| 场景 | 推荐选择 |
|------|---------|
| **内存敏感** | 标准 BF（最低内存） |
| **需要删除操作** | CBF（4-bit counters） |
| **容量未知/可能爆炸** | SBF |
| **既需删除又需扩容** | SBF + 外部映射 或 定期重建 |
| **追求高性能** | MurmurHash3 + 双哈希 |

---

## 6. C++ 实现

### 6.1 计数布隆过滤器

```cpp
#include <bits/stdc++.h>
using namespace std;

class CountingBloomFilter {
private:
    int m;                    // 计数器数组长度
    int k;                    // 哈希函数数量
    int counter_bits;         // 每个计数器的位数
    vector<uint8_t> counters; // 计数器数组
    uint8_t max_counter;      // 最大计数器值
    
    uint64_t murmurHash3(const string& key, uint64_t seed = 0) const {
        const uint64_t c1 = 0xcc9e2d51ULL;
        const uint64_t c2 = 0x1b873593ULL;
        
        uint64_t h1 = seed;
        uint64_t len = key.length();
        const uint64_t* data = (const uint64_t*)key.data();
        uint64_t words = len / 8;
        
        for (uint64_t i = 0; i < words; ++i) {
            uint64_t k1 = data[i];
            k1 *= c1;
            k1 = (k1 << 31) | (k1 >> 33);
            k1 *= c2;
            h1 ^= k1;
            h1 = (h1 << 27) | (h1 >> 37);
            h1 = h1 * 5 + 0x52dce729;
        }
        
        const uint8_t* tail = (const uint8_t*)(data + words);
        uint64_t k1 = 0;
        
        switch (len & 7) {
            case 7: k1 ^= uint64_t(tail[6]) << 48;
            case 6: k1 ^= uint64_t(tail[5]) << 40;
            case 5: k1 ^= uint64_t(tail[4]) << 32;
            case 4: k1 ^= uint64_t(tail[3]) << 24;
            case 3: k1 ^= uint64_t(tail[2]) << 16;
            case 2: k1 ^= uint64_t(tail[1]) << 8;
            case 1: k1 ^= uint64_t(tail[0]) << 0;
                    k1 *= c1;
                    k1 = (k1 << 31) | (k1 >> 33);
                    k1 *= c2;
                    h1 ^= k1;
        }
        
        h1 ^= len;
        h1 ^= h1 >> 33;
        h1 *= 0xff51afd7ed558ccdULL;
        h1 ^= h1 >> 33;
        h1 *= 0xc4ceb9fe1a85ec53ULL;
        h1 ^= h1 >> 33;
        
        return h1;
    }
    
    // 读取指定位置的计数器值
    uint8_t getCounter(int pos) const {
        int idx = pos * counter_bits / 8;
        int offset = pos * counter_bits % 8;
        uint8_t mask = (1 << counter_bits) - 1;
        
        uint8_t val = counters[idx] >> offset;
        if (offset + counter_bits > 8) {
            val |= (counters[idx + 1] << (8 - offset));
        }
        return val & mask;
    }
    
    // 设置指定位置的计数器值
    void setCounter(int pos, uint8_t val) {
        int idx = pos * counter_bits / 8;
        int offset = pos * counter_bits % 8;
        uint8_t mask = (1 << counter_bits) - 1;
        
        counters[idx] &= ~(mask << offset);
        counters[idx] |= (val & mask) << offset;
        
        if (offset + counter_bits > 8) {
            counters[idx + 1] &= ~(mask >> (8 - offset));
            counters[idx + 1] |= (val & mask) >> (8 - offset);
        }
    }
    
public:
    CountingBloomFilter(int expectedElements, double falsePositiveRate, int counterBits = 4)
        : counter_bits(counterBits) {
        double n = expectedElements;
        double p = falsePositiveRate;
        
        // 最优 m 和 k 的计算与标准 BF 相同
        m = (int)ceil(-n * log(p) / (log(2) * log(2)));
        k = (int)ceil((double)m / n * log(2));
        m = ((m + 63) / 64) * 64;  // 对齐到 64 位
        
        max_counter = (1 << counter_bits) - 1;
        counters.assign((m * counter_bits + 7) / 8, 0);
    }
    
    // 插入元素
    void add(const string& key) {
        for (int i = 0; i < k; ++i) {
            uint64_t hash = murmurHash3(key, i);
            int pos = hash % m;
            uint8_t val = getCounter(pos);
            if (val < max_counter) {
                setCounter(pos, val + 1);
            }
        }
    }
    
    // 查询元素
    bool contains(const string& key) const {
        for (int i = 0; i < k; ++i) {
            uint64_t hash = murmurHash3(key, i);
            int pos = hash % m;
            if (getCounter(pos) == 0) {
                return false;
            }
        }
        return true;
    }
    
    // 删除元素
    void remove(const string& key) {
        for (int i = 0; i < k; ++i) {
            uint64_t hash = murmurHash3(key, i);
            int pos = hash % m;
            uint8_t val = getCounter(pos);
            if (val > 0) {
                setCounter(pos, val - 1);
            }
        }
    }
    
    // 获取大小
    int size() const { return m; }
    int getHashCount() const { return k; }
};
```

### 6.2 可扩展布隆过滤器

```cpp
#include <bits/stdc++.h>
using namespace std;

class ScalableBloomFilter {
private:
    int initial_m;           // 初始位数组大小
    int initial_k;            // 初始哈希函数数量
    double initial_p;          // 初始假阳性率
    double tightening_factor; // 扩展因子 r
    
    struct BloomFilter {
        int m;
        int k;
        double p;
        vector<uint64_t> bit_array;
        
        BloomFilter(int m, int k, double p) : m(m), k(k), p(p) {
            bit_array.assign((m + 63) / 64, 0);
        }
        
        void setBit(int pos) {
            bit_array[pos / 64] |= (1ULL << (pos % 64));
        }
        
        bool getBit(int pos) const {
            return (bit_array[pos / 64] & (1ULL << (pos % 64))) != 0;
        }
    };
    
    vector<BloomFilter> filters;
    uint64_t murmurHash3(const string& key, uint64_t seed = 0) const {
        const uint64_t c1 = 0xcc9e2d51ULL;
        const uint64_t c2 = 0x1b873593ULL;
        
        uint64_t h1 = seed;
        uint64_t len = key.length();
        const uint64_t* data = (const uint64_t*)key.data();
        uint64_t words = len / 8;
        
        for (uint64_t i = 0; i < words; ++i) {
            uint64_t k1 = data[i];
            k1 *= c1;
            k1 = (k1 << 31) | (k1 >> 33);
            k1 *= c2;
            h1 ^= k1;
            h1 = (h1 << 27) | (h1 >> 37);
            h1 = h1 * 5 + 0x52dce729;
        }
        
        const uint8_t* tail = (const uint8_t*)(data + words);
        uint64_t k1 = 0;
        
        switch (len & 7) {
            case 7: k1 ^= uint64_t(tail[6]) << 48;
            case 6: k1 ^= uint64_t(tail[5]) << 40;
            case 5: k1 ^= uint64_t(tail[4]) << 32;
            case 4: k1 ^= uint64_t(tail[3]) << 24;
            case 3: k1 ^= uint64_t(tail[2]) << 16;
            case 2: k1 ^= uint64_t(tail[1]) << 8;
            case 1: k1 ^= uint64_t(tail[0]) << 0;
                    k1 *= c1;
                    k1 = (k1 << 31) | (k1 >> 33);
                    k1 *= c2;
                    h1 ^= k1;
        }
        
        h1 ^= len;
        h1 ^= h1 >> 33;
        h1 *= 0xff51afd7ed558ccdULL;
        h1 ^= h1 >> 33;
        h1 *= 0xc4ceb9fe1a85ec53ULL;
        h1 ^= h1 >> 33;
        
        return h1;
    }
    
    // 检查当前过滤器是否需要扩容
    bool needsExpansion() const {
        if (filters.empty()) return true;
        const auto& last = filters.back();
        // 简单策略：当位数组填充率超过 50% 时扩容
        int set_bits = 0;
        for (auto v : last.bit_array) {
            set_bits += __builtin_popcountll(v);
        }
        return set_bits > last.m / 2;
    }
    
public:
    ScalableBloomFilter(int expectedElements, double falsePositiveRate, double r = 0.9)
        : tightening_factor(r) {
        double n = expectedElements;
        double p = falsePositiveRate;
        
        initial_m = (int)ceil(-n * log(p) / (log(2) * log(2)));
        initial_k = (int)ceil((double)initial_m / n * log(2));
        initial_p = p;
        
        // 创建第一个过滤器
        int m = ((initial_m + 63) / 64) * 64;
        filters.emplace_back(m, initial_k, initial_p);
    }
    
    // 插入元素
    void add(const string& key) {
        if (filters.empty()) {
            int m = ((initial_m + 63) / 64) * 64;
            filters.emplace_back(m, initial_k, initial_p);
        }
        
        // 尝试插入最后一个过滤器
        auto& last = filters.back();
        for (int i = 0; i < last.k; ++i) {
            uint64_t hash = murmurHash3(key, i);
            last.setBit(hash % last.m);
        }
        
        // 如果需要扩容，创建新过滤器
        if (needsExpansion()) {
            int new_m = (int)(last.m * tightening_factor);
            new_m = ((new_m + 63) / 64) * 64;
            double new_p = last.p * tightening_factor;
            int new_k = (int)ceil((double)new_m / initial_m * log(2));
            filters.emplace_back(new_m, new_k, new_p);
        }
    }
    
    // 查询元素
    bool contains(const string& key) const {
        for (const auto& bf : filters) {
            bool found = true;
            for (int i = 0; i < bf.k; ++i) {
                uint64_t hash = murmurHash3(key, i);
                if (!bf.getBit(hash % bf.m)) {
                    found = false;
                    break;
                }
            }
            if (found) return true;
        }
        return false;
    }
    
    // 获取过滤器数量
    int getFilterCount() const { return filters.size(); }
    
    // 获取总体假阳性率估计
    double getFalsePositiveRate() const {
        double overall = 1.0;
        for (const auto& bf : filters) {
            overall *= (1 - bf.p);
        }
        return 1 - overall;
    }
};
```

### 6.3 使用示例

```cpp
int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    // 测试计数布隆过滤器
    cout << "=== 计数布隆过滤器测试 ===" << endl;
    CountingBloomFilter cbf(1000, 0.01, 4);
    
    vector<string> words = {"apple", "banana", "cherry", "date", "elderberry"};
    
    cout << "插入元素..." << endl;
    for (const auto& w : words) {
        cbf.add(w);
    }
    
    cout << "查询测试:" << endl;
    cout << "\"apple\": " << (cbf.contains("apple") ? "可能存在" : "不存在") << endl;
    cout << "\"grape\": " << (cbf.contains("grape") ? "可能存在" : "不存在") << endl;
    
    cout << "\n删除 \"apple\" 后:" << endl;
    cbf.remove("apple");
    cout << "\"apple\": " << (cbf.contains("apple") ? "可能存在" : "不存在") << endl;
    
    // 测试可扩展布隆过滤器
    cout << "\n=== 可扩展布隆过滤器测试 ===" << endl;
    ScalableBloomFilter sbf(100, 0.1);
    
    cout << "插入 200 个元素..." << endl;
    for (int i = 0; i < 200; ++i) {
        sbf.add("element_" + to_string(i));
    }
    
    cout << "过滤器数量: " << sbf.getFilterCount() << endl;
    cout << "估计假阳性率: " << sbf.getFalsePositiveRate() << endl;
    
    cout << "\n查询测试:" << endl;
    cout << "\"element_50\": " << (sbf.contains("element_50") ? "可能存在" : "不存在") << endl;
    cout << "\"element_999\": " << (sbf.contains("element_999") ? "可能存在" : "不存在") << endl;
    
    return 0;
}
```

---

## 7. 总结

| 特性 | 标准 BF | 计数 BF | 可扩展 BF |
|------|---------|---------|-----------|
| **删除操作** | ❌ 不支持 | ✅ 支持 | ❌ 不支持 |
| **假阴性** | ❌ 不可能 | ⚠️ 可能 | ❌ 不可能 |
| **动态扩容** | ❌ 固定容量 | ❌ 固定容量 | ✅ 支持 |
| **空间开销** | 最低 | 中等（计数器） | 略高（多过滤器） |
| **适用场景** | 简单去重 | 动态集合 | 未知容量 |

**选择建议**：

- 需要删除但容量可预估 → **计数布隆过滤器**
- 容量未知或可能剧增 → **可扩展布隆过滤器**
- 既要删除又要扩容 → 考虑定期重建或使用外部映射

---

## 参考文献

[^1]: Fan, L., Cao, P., Almeida, J., & Broder, A. Z. (2000). Summary cache: a scalable wide-area web cache sharing protocol. IEEE/ACM Transactions on Networking, 8(3), 281-293.

[^2]: Almeida, P. S., Baquero, C., Preguiça, N., & Hutchinson, D. (2007). Scalable Bloom Filters. Information Processing Letters, 101(6), 255-261.

[^3]: Broder, A., & Mitzenmacher, M. (2004). Network Applications of Bloom Filters: A Survey. Internet Mathematics, 1(4), 485-509.

[^4]: Mitzenmacher, M. (2002). Compressed Bloom Filters. IEEE/ACM Transactions on Networking, 10(5), 604-612.
