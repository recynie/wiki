---
title: Bloom Filter
date: 2026-04-12
description: 布隆过滤器的原理、概率分析、变体及在数据库缓存中的应用
tags:
  - probabilistic-data-structure
  - data-structure
  - hash
draft: false
permalink:
---

## 1. 定义

布隆过滤器（Bloom Filter）是一种**概率型数据结构**（Probabilistic Data Structure），由 Burton Howard Bloom 于 1970 年提出。它主要用于**成员资格检测**（Membership Testing），即判断一个元素是否属于某个集合。

布隆过滤器的核心特点是：

- **空间效率高**：相比传统的哈希表，占用空间极小
- **查询速度快**：时间复杂度为 $O(k)$，其中 $k$ 为哈希函数个数
- **允许假阳性**：可能将不在集合中的元素错误地判断为存在
- **不允许假阴性**：如果查询返回"不存在"，则元素一定不在集合中

## 2. 核心原理

布隆过滤器由两个组件构成：

| 组件 | 说明 |
|------|------|
| **位数组** | 长度为 $m$ 的位数组，初始时所有位均为 0 |
| **哈希函数** | $k$ 个相互独立的哈希函数，每个函数将元素映射到 $[0, m-1]$ 范围内的位置 |

### 2.1 数据结构示意

```
位数组（m = 18 bits）:
Index:  0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17
Bits:   0  1  0  1  0  0  0  1  0  0  1  0  0  0  1  0  0  1
                ↑     ↑           ↑        ↑     ↑           ↑
              h1(x) h2(x)       h3(x)    h4(x) h5(x)       hk(x)
```

## 3. 基本操作

### 3.1 插入（Insert）

将元素 $x$ 插入布隆过滤器：

```
对于每个哈希函数 hi (1 ≤ i ≤ k):
    设置位数组中位置 hi(x) 的值为 1
```

**示例**：插入元素 "apple"

```
假设 k = 3
h1("apple") = 2  → 设置 bit[2] = 1
h2("apple") = 5  → 设置 bit[5] = 1
h3("apple") = 9  → 设置 bit[9] = 1
```

### 3.2 查询（Lookup）

查询元素 $x$ 是否存在：

```
对于每个哈希函数 hi (1 ≤ i ≤ k):
    如果 bit[hi(x)] == 0:
        返回 false（元素一定不存在）
返回 true（元素可能存在，存在假阳性）
```

### 3.3 为什么不能删除？

由于多个不同的元素可能共享相同的比特位置，删除一个元素可能会影响其他元素的查询结果。

```
示例冲突场景：
元素 "apple" 设置了 bit[2], bit[5], bit[9]
元素 "banana" 设置了 bit[5], bit[8], bit[9]

如果删除 "apple"，将 bit[5] 和 bit[9] 设为 0
则查询 "banana" 时会得到错误结果（false），虽然 "banana" 实际已插入
```

## 4. 假阳性分析

### 4.1 概率计算

设位数组有 $m$ 位，插入 $n$ 个元素，使用 $k$ 个哈希函数。

**单个哈希函数将某位设置为 0 的概率**：

$$
P(\text{某位为 0}) = \left(1 - \frac{1}{m}\right)
$$

**经过 $k$ 个哈希函数后，某位仍为 0 的概率**：

$$
P(\text{某位仍为 0}) = \left(1 - \frac{1}{m}\right)^k
$$

**插入 $n$ 个元素后，某位仍为 0 的概率**（各次插入独立）：

$$
P(\text{某位仍为 0}) = \left(1 - \frac{1}{m}\right)^{kn}
$$

**某位被设置为 1 的概率**：

$$
P(\text{某位为 1}) = 1 - \left(1 - \frac{1}{m}\right)^{kn}
$$

**假阳性概率**（查询时 $k$ 个指定位置恰好全为 1）：

$$
P(\text{假阳性}) = \left[1 - \left(1 - \frac{1}{m}\right)^{kn}\right]^k
$$

当 $m$ 很大时，利用极限 $\lim_{x \to \infty} \left(1 - \frac{1}{x}\right)^x = e^{-1}$：

$$
P(\text{假阳性}) \approx \left(1 - e^{-kn/m}\right)^k
$$

### 4.2 最优哈希函数数量

对 $P(\text{假阳性})$ 关于 $k$ 求导，令导数为零，可得**最优哈希函数数量**：

$$
k_{\text{opt}} = \frac{m}{n} \ln 2 \approx \frac{m}{n} \times 0.693
$$

此时**最小假阳性概率**为：

$$
P_{\min} = \left(\frac{1}{2}\right)^{\frac{m}{n} \ln 2} = (0.6185)^{\frac{m}{n}}
$$

### 4.3 空间-准确性权衡

| $m/n$（每位元素占用位数） | 最优 $k$ | 假阳性率约等于 |
|--------------------------|---------|---------------|
| 8 bits/element | 5.5 | 2.0% |
| 10 bits/element | 6.9 | 0.81% |
| 16 bits/element | 11.1 | 0.01% |

**结论**：每位元素分配约 10 bits 时，可将假阳性率控制在 1% 以下。

## 5. 变体

### 5.1 可扩展布隆过滤器（Scalable Bloom Filter）

当元素数量超过预期时，标准布隆过滤器需要重新创建。**可扩展布隆过滤器**通过动态扩容解决问题。

**基本思想**：维护一系列布隆过滤器 $BF_1, BF_2, \ldots, BF_t$，每个使用不同的哈希函数和更大的位数组。

```
扩容过程：
1. 当前过滤器 BF_i 假阳性率超过阈值时
2. 创建新的过滤器 BF_{i+1}，位数组大小为 s * r（s 为前一个大小，r 为扩展因子）
3. 新插入的元素放入 BF_{i+1}
4. 查询时检查所有过滤器
```

### 5.2 计数布隆过滤器（Counting Bloom Filter）

为了支持删除操作，将位数组扩展为计数数组：

```
标准 BF:  bit[i] ∈ {0, 1}
计数 BF:  count[i] ∈ {0, 1, 2, 3, ...}
```

**插入**：对应位置的计数器 +1  
**删除**：对应位置的计数器 -1  
**查询**：检查计数器是否大于 0

**代价**：空间开销增加（通常使用 3-4 位计数器）

### 5.3 其他变体

| 变体 | 特点 |
|------|------|
| **分层布隆过滤器** | 多层结构，用于减少假阳性 |
| **谱布隆过滤器** | 可估计元素出现次数 |
| **遗忘布隆过滤器** | 支持元素撤销 |
| **稳定布隆过滤器** | 用于处理无限数据流 |

## 6. 应用场景

### 6.1 数据库查询优化

**场景**：数据库查询磁盘前，先用布隆过滤器判断记录是否可能存在。

```
查询流程：
1. 客户端提交查询条件
2. 数据库在访问磁盘前，先查询布隆过滤器
3. 如果布隆过滤器返回 false → 记录一定不存在，跳过磁盘访问
4. 如果返回 true → 继续访问磁盘（可能产生磁盘 I/O）
```

**收益**：大幅减少不必要的磁盘 I/O 操作，提升查询性能。

### 6.2 Web 缓存

**场景**：在 CDN 或代理服务器中，避免向源站发起无效请求。

```
用户请求 URL：
1. 检查布隆过滤器，判断 URL 是否可能已被缓存
2. 返回 false → 一定未缓存，向源站请求
3. 返回 true → 可能已缓存，查询缓存（可能 cache miss）
```

### 6.3 恶意 URL 检测

**场景**：浏览器或安全网关快速判断 URL 是否在恶意列表中。

```
访问请求：
1. 查询布隆过滤器
2. 返回 false → URL 安全，直接放行
3. 返回 true → 可能恶意，进行详细安全检查
```

### 6.4 用户名/密码检查

**场景**：注册时快速检查用户名是否已被占用。

```
用户注册 "john_doe"：
1. 查询布隆过滤器（存储已注册用户名）
2. 返回 false → 用户名一定可用
3. 返回 true → 可能已被占用，查询数据库确认
```

### 6.5 其他应用

| 领域 | 应用 |
|------|------|
| **垃圾邮件过滤** | 判断邮件地址是否在黑名单 |
| **区块链** | SPV 节点快速验证交易 |
| **拼写检查** | 判断单词是否在词典中 |
| **分布式系统** | 减少网络请求和数据传输 |

## 7. C++ 实现

### 7.1 基础实现

```cpp
#include <bits/stdc++.h>
using namespace std;

class BloomFilter {
private:
    int m;                          // 位数组大小
    int k;                          // 哈希函数数量
    vector<uint64_t> bit_array;     // 位数组（使用 uint64_t 提升效率）
    
    // MurmurHash3 哈希函数
    uint64_t murmurHash3(const string& key, uint64_t seed = 0) const {
        const uint64_t c1 = 0xcc9e2d51ULL;
        const uint64_t c2 = 0x1b873593ULL;
        
        uint64_t h1 = seed;
        uint64_t len = key.length();
        const uint64_t* data = (const uint64_t*)key.data();
        uint64_t words = len / 8;
        
        // 处理完整的 8 字节块
        for (uint64_t i = 0; i < words; ++i) {
            uint64_t k1 = data[i];
            k1 *= c1;
            k1 = (k1 << 31) | (k1 >> 33);  // ROTL64(k1, 31)
            k1 *= c2;
            h1 ^= k1;
            h1 = (h1 << 27) | (h1 >> 37);  // ROTL64(h1, 27)
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
        
        // 最终化
        h1 ^= len;
        h1 ^= h1 >> 33;
        h1 *= 0xff51afd7ed558ccdULL;
        h1 ^= h1 >> 33;
        h1 *= 0xc4ceb9fe1a85ec53ULL;
        h1 ^= h1 >> 33;
        
        return h1;
    }
    
public:
    // 构造函数：给定预期元素数和假阳性率
    BloomFilter(int expectedElements, double falsePositiveRate) {
        // 计算最优位数组大小和哈希函数数量
        double n = expectedElements;
        double p = falsePositiveRate;
        
        // m = -n * ln(p) / (ln(2)^2)
        m = (int)ceil(-n * log(p) / (log(2) * log(2)));
        // k = (m/n) * ln(2)
        k = (int)ceil((double)m / n * log(2));
        
        // 确保位数组大小是 64 的倍数
        m = ((m + 63) / 64) * 64;
        bit_array.assign(m / 64, 0);
    }
    
    // 构造函数：给定位数组大小和哈希函数数量
    BloomFilter(int m, int k) : m(m), k(k) {
        m = ((m + 63) / 64) * 64;
        bit_array.assign(m / 64, 0);
    }
    
    // 插入元素
    void insert(const string& key) {
        for (int i = 0; i < k; ++i) {
            uint64_t hash = murmurHash3(key, i);
            int pos = hash % m;
            bit_array[pos / 64] |= (1ULL << (pos % 64));
        }
    }
    
    // 查询元素
    bool contains(const string& key) const {
        for (int i = 0; i < k; ++i) {
            uint64_t hash = murmurHash3(key, i);
            int pos = hash % m;
            if ((bit_array[pos / 64] & (1ULL << (pos % 64))) == 0) {
                return false;
            }
        }
        return true;
    }
    
    // 获取当前假阳性率估计
    double getFalsePositiveRate(int numElements) const {
        // P = (1 - e^(-kn/m))^k
        double rate = pow(1 - exp(-(double)k * numElements / m), k);
        return rate;
    }
    
    // 获取位数组大小（bits）
    int size() const { return m; }
    
    // 获取哈希函数数量
    int getHashCount() const { return k; }
};
```

### 7.2 计数布隆过滤器实现

```cpp
#include <bits/stdc++.h>
using namespace std;

class CountingBloomFilter {
private:
    int m;
    int k;
    int counter_bits;           // 每个计数器使用的位数
    vector<uint8_t> counters;    // 计数器数组
    uint8_t max_counter;         // 计数器的最大值
    
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
        
        m = (int)ceil(-n * log(p) / (log(2) * log(2)));
        k = (int)ceil((double)m / n * log(2));
        m = ((m + 63) / 64) * 64;  // 对齐
        
        max_counter = (1 << counter_bits) - 1;
        counters.assign((m * counter_bits + 7) / 8, 0);
    }
    
    void insert(const string& key) {
        for (int i = 0; i < k; ++i) {
            uint64_t hash = murmurHash3(key, i);
            int pos = hash % m;
            uint8_t val = getCounter(pos);
            if (val < max_counter) {
                setCounter(pos, val + 1);
            }
        }
    }
    
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
};
```

### 7.3 使用示例

```cpp
int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    // 创建布隆过滤器：预计 10000 个元素，期望假阳性率 1%
    BloomFilter bf(10000, 0.01);
    
    cout << "位数组大小: " << bf.size() << " bits" << endl;
    cout << "哈希函数数量: " << bf.getHashCount() << endl;
    
    // 插入元素
    vector<string> words = {"apple", "banana", "cherry", "date", "elderberry"};
    for (const auto& w : words) {
        bf.insert(w);
    }
    
    // 查询测试
    cout << "\n查询测试:" << endl;
    cout << "\"apple\": " << (bf.contains("apple") ? "可能存在" : "不存在") << endl;
    cout << "\"grape\": " << (bf.contains("grape") ? "可能存在" : "不存在") << endl;
    
    // 假阳性测试
    int falsePositives = 0;
    int testCount = 100000;
    for (int i = 0; i < testCount; ++i) {
        string s = "test_" + to_string(i);
        if (bf.contains(s) && find(words.begin(), words.end(), s) == words.end()) {
            falsePositives++;
        }
    }
    cout << "\n假阳性测试 (共 " << testCount << " 个不存在元素):" << endl;
    cout << "实际假阳性数: " << falsePositives << endl;
    cout << "实际假阳性率: " << fixed << setprecision(4) 
         << (double)falsePositives / testCount * 100 << "%" << endl;
    cout << "理论假阳性率: " << bf.getFalsePositiveRate(5) * 100 << "%" << endl;
    
    return 0;
}
```

## 8. 总结

布隆过滤器是一种高效的概率型数据结构，通过牺牲确定性换取空间效率。理解其核心原理、概率分析和使用场景，对于设计高性能系统具有重要意义。

| 优点 | 缺点 |
|------|------|
| 空间效率高 | 不支持删除操作（标准版本） |
| 查询速度快 | 存在假阳性 |
| 无假阴性 | 不能获取元素本身，只能判断存在性 |
| 易于实现 | 参数选择需要预估数据规模 |

## 参考文献

[^1]: Baeldung. "Bloom Filter: A Probabilistic Data Structure". https://www.baeldung.com/java-bloom-filter

[^2]: Broder, A., & Mitzenmacher, M. "Network Applications of Bloom Filters: A Survey". Internet Mathematics, 2004.

[^3]: Redis Documentation. "Bloom Filter". https://redis.io/docs/data-types/probabilistic/bloom-filter/

[^4]: Mitzenmacher, M. "Compressed Bloom Filters". IEEE/ACM Transactions on Networking, 2002.
