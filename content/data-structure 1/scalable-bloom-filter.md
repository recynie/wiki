---
title: Scalable Bloom Filter
date: 2026-04-13
description: 可扩展布隆过滤器（Scalable Bloom Filter）通过动态添加新的布隆过滤器来应对元素数量增长。
tags:
  - bloom-filter
  - data-structure
draft: true
permalink:
---

## 概述

Scalable Bloom Filter（可扩展布隆过滤器）解决了标准布隆过滤器在元素数量预估不准确时的容量问题。[^1]

其核心思想是：当元素数量增长超过当前容量时，动态创建新的子布隆过滤器，而不是重建整个过滤器。新插入的元素放入新的子过滤器，查询时需要检查所有子过滤器。

## 工作原理

### 核心参数

- **tightening ratio $r$**：控制新增子过滤器的误差范围，取值范围 $(0, 1)$，通常设为 0.8 或 0.9
- **初始容量 $s$**：第一个子过滤器的预期容量
- **增长因子 $g$**：新子过滤器容量相对于旧过滤器的倍数

### 扩容策略

设第 $i$ 个子过滤器的容量为 $s_i$，假阳性率为 $p_i$：

$$
s_i = s \cdot g^i
$$

$$
p_i = p_0 \cdot r^i
$$

其中 $g = 1/(1-r)$，保证整体假阳性率有上界。

### 操作流程

**插入操作**

1. 检查当前子过滤器数量是否足以容纳新元素
2. 如果空间不足，创建新的子过滤器
3. 将新元素插入最新的子过滤器

**查询操作**

按倒序检查所有子过滤器（从最新到最旧），只要任一子过滤器返回存在，则元素存在。

## 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

class StandardBloomFilter {
private:
    vector<bool> bits;
    int k;
    int m;
    
public:
    StandardBloomFilter(int m, int k) : m(m), k(k), bits(m, false) {}
    
    size_t hash(int idx, const string& item) {
        hash<string> h;
        return (h(item) + idx * 1315423911) % m;
    }
    
    void insert(const string& item) {
        for (int i = 0; i < k; ++i) {
            bits[hash(i, item)] = true;
        }
    }
    
    bool contains(const string& item) const {
        for (int i = 0; i < k; ++i) {
            if (!bits[hash(i, item)]) {
                return false;
            }
        }
        return true;
    }
};

class ScalableBloomFilter {
private:
    vector<StandardBloomFilter*> filters;
    double p;       // 目标假阳性率上限
    double r;       // tightening ratio
    int s;          // 初始容量
    int k;          // 哈希函数数量
    int currentSize;
    
public:
    ScalableBloomFilter(double p = 0.01, double r = 0.8, int s = 1000) 
        : p(p), r(r), s(s), currentSize(0) {
        addNewFilter();
    }
    
    ~ScalableBloomFilter() {
        for (auto f : filters) delete f;
    }
    
    void addNewFilter() {
        int m = filters.empty() ? s * 10 : filters.back()->getM();
        int newM = static_cast<int>(m / r);  // 扩大以保持误差
        int newK = StandardBloomFilter::optimalK(newM, s);
        filters.push_back(new StandardBloomFilter(newM, newK));
    }
    
    void insert(const string& item) {
        if (filters.back()->getSize() >= s) {
            addNewFilter();
        }
        filters.back()->insert(item);
        currentSize++;
    }
    
    bool contains(const string& item) const {
        // 从最新的过滤器开始查询
        for (int i = filters.size() - 1; i >= 0; --i) {
            if (filters[i]->contains(item)) {
                return true;
            }
        }
        return false;
    }
    
    int getFilterCount() const { return filters.size(); }
};
```

## 关键特性

### 优点

- **动态扩展**：无需重建整个过滤器即可增加容量
- **误差可控**：通过 tightening ratio $r$ 控制整体假阳性率上界
- **空间效率**：子过滤器按需创建，避免初始容量预估不准确的问题

### 缺点

- **查询开销增加**：需要检查所有子过滤器，时间复杂度为 $O(N)$，其中 $N$ 为子过滤器数量
- **删除不支持**：可扩展布隆过滤器通常不支持删除操作
- **空间浪费**：旧过滤器中的元素无法利用新过滤器的空间

### 容量增长曲线

当元素数量达到第 $i$ 个子过滤器的容量时，会创建第 $i+1$ 个子过滤器：

| 子过滤器编号 | 容量 | 累计容量 |
|-------------|------|---------|
| 0 | $s$ | $s$ |
| 1 | $s/r$ | $s(1 + 1/r)$ |
| 2 | $s/r^2$ | $s(1 + 1/r + 1/r^2)$ |
| $i$ | $s/r^i$ | $s \cdot \frac{1 - r^{-(i+1)}}{1 - r}$ |

## 应用场景

可扩展布隆过滤器适用于以下场景：

- **未知规模的集合**：无法预估元素总数量的场景
- **长期运行系统**：如数据库连接池、会话管理等
- **分布式缓存**：需要渐进式扩展的缓存系统
- **数据流处理**：元素数量持续增长的流式数据场景

## 与标准布隆过滤器的比较

```cpp
// 标准布隆过滤器 - 需要预估容量
StandardBloomFilter bf(1000000, 7);  // 预分配大空间
for (int i = 0; i < 10000000; ++i) {
    bf.insert(to_string(i));  // 可能超出容量
}

// 可扩展布隆过滤器 - 自动扩容
ScalableBloomFilter sbf(0.01, 0.8, 1000);  // 从小容量开始
for (int i = 0; i < 10000000; ++i) {
    sbf.insert(to_string(i));  // 自动创建新过滤器
}
```

| 特性 | 标准布隆过滤器 | 可扩展布隆过滤器 |
|------|---------------|-----------------|
| 容量管理 | 静态，预分配 | 动态，按需扩展 |
| 查询复杂度 | $O(k)$ | $O(k \cdot N)$ |
| 删除支持 | ✅ | ❌ |
| 空间效率 | 取决于预估准确性 | 初期较高，后期可能冗余 |
| 适用场景 | 容量可预估 | 容量不可预估 |

## 参考文献

[^1]: 塞维利亚大学，Almeida 等人提出可扩展布隆过滤器，用于解决标准布隆过滤器的静态容量问题。
