---
title: LFU Cache（最不经常使用缓存）
date: 2026-04-13
description: LFU缓存置换算法，在O(1)时间复杂度内实现get和put操作
tags:
  - data-structure
  - cache
  - lfu
draft: false
permalink:
---

## 概述

LFU（Least Frequently Used，最不经常使用）缓存是一种缓存置换策略，淘汰访问频率最低的条目。与 LRU 不同，LFU 基于访问次数而非最近访问时间进行淘汰。

## 问题定义

设计一个 LFU 缓存数据结构，支持：
- `get(key)`：获取指定 key 的值，如果存在则返回
- `put(key, value)`：插入或更新 key-value 对

约束条件：
- 容量满时，淘汰访问频率最低的条目
- 频率相同时，淘汰最久未访问的条目（LRU 作为 tiebreaker）
- 所有操作均摊 $O(1)$ 时间复杂度

## 数据结构设计

### 核心思想：频率分层链表

为实现 $O(1)$ 复杂度，需要三个数据结构协同工作：

```
┌─────────────────────────────────────────────────────────┐
│  freq_to_list: HashMap<frequency, DoublyLinkedList>    │
│                                                         │
│  freq=1: [key1] ↔ [key3]                               │
│  freq=2: [key2] ↔ [key5]                               │
│  freq=3: [key4]                                        │
│                                                         │
│  min_freq = 1  (追踪最低频率)                           │
├─────────────────────────────────────────────────────────┤
│  key_to_node: HashMap<key, Node>                      │
│                                                         │
│  key1 → Node(value=1, freq=1, prev=, next=)          │
└─────────────────────────────────────────────────────────┘
```

### 晋升机制

当 key 被访问时：
1. 从当前频率链表中移除
2. 如果当前链表为空且 freq == min_freq，则 min_freq++
3. 插入 freq+1 对应的链表头部
4. 更新 min_freq = min(min_freq, freq+1)

### 淘汰机制

当容量满时：
1. 从 min_freq 对应的链表尾部（最久未访问）淘汰
2. 如果淘汰后链表为空，则 min_freq++

## C++ 实现

```cpp
#include <unordered_map>
#include <list>
#include <vector>

class LFUCache {
    struct Node {
        int key, value, freq;
        Node(int k, int v, int f) : key(k), value(v), freq(f) {}
    };
    
    int capacity;
    int minFreq;
    std::unordered_map<int, std::list<Node>::iterator> keyToIter;
    std::unordered_map<int, std::list<Node>> freqToList;

    void update(int key, int value = -1) {
        auto it = keyToIter[key];
        int oldFreq = it->freq;
        int newFreq = oldFreq + 1;
        
        // 从旧频率链表移除
        freqToList[oldFreq].erase(it);
        if (freqToList[oldFreq].empty()) {
            freqToList.erase(oldFreq);
            if (minFreq == oldFreq) minFreq = newFreq;
        }
        
        // 如果是新插入
        if (value != -1) {
            it->value = value;
        }
        it->freq = newFreq;
        
        // 插入新频率链表头部
        freqToList[newFreq].push_front(*it);
        keyToIter[key] = freqToList[newFreq].begin();
    }

public:
    LFUCache(int cap) : capacity(cap), minFreq(0) {}
    
    int get(int key) {
        if (capacity == 0) return -1;
        if (keyToIter.find(key) == keyToIter.end()) return -1;
        update(key);  // 频率+1
        return keyToIter[key]->value;
    }
    
    void put(int key, int value) {
        if (capacity == 0) return;
        
        if (keyToIter.find(key) != keyToIter.end()) {
            // 已存在，更新值并晋升
            update(key, value);
        } else {
            // 新插入
            if (keyToIter.size() >= capacity) {
                // 淘汰最低频率链表的尾部
                auto& list = freqToList[minFreq];
                int evictKey = list.back().key;
                list.pop_back();
                keyToIter.erase(evictKey);
                if (list.empty()) {
                    freqToList.erase(minFreq);
                }
            }
            
            // 插入新节点，频率为1
            Node node(key, value, 1);
            freqToList[1].push_front(node);
            keyToIter[key] = freqToList[1].begin();
            minFreq = 1;
        }
    }
};
```

## Python 实现

```python
from collections import OrderedDict, defaultdict

class LFUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.min_freq = float('inf')
        self.key_to_val_freq = {}      # key -> (value, frequency)
        self.freq_to_keys = defaultdict(OrderedDict)  # freq -> OrderedDict of keys
        
    def _update(self, key, value=None):
        val, freq = self.key_to_val_freq[key]
        
        # 删除旧频率
        del self.freq_to_keys[freq][key]
        if not self.freq_to_keys[freq]:
            del self.freq_to_keys[freq]
            if self.min_freq == freq:
                self.min_freq = freq + 1
        
        # 更新值（如果提供）
        if value is not None:
            val = value
        new_freq = freq + 1
        
        # 插入新频率
        self.key_to_val_freq[key] = (val, new_freq)
        self.freq_to_keys[new_freq][key] = None
        self.min_freq = min(self.min_freq, new_freq)
    
    def get(self, key: int) -> int:
        if key not in self.key_to_val_freq:
            return -1
        self._update(key)
        return self.key_to_val_freq[key][0]
    
    def put(self, key: int, value: int) -> None:
        if self.capacity <= 0:
            return
        
        if key in self.key_to_val_freq:
            self._update(key, value)
        else:
            if len(self.key_to_val_freq) >= self.capacity:
                # 淘汰最旧的
                evict_key, _ = self.freq_to_keys[self.min_freq].popitem(last=False)
                del self.key_to_val_freq[evict_key]
            
            self.key_to_val_freq[key] = (value, 1)
            self.freq_to_keys[1][key] = None
            self.min_freq = 1
```

## 复杂度分析

| 操作 | 时间复杂度 | 空间复杂度 |
|------|-----------|-----------|
| get | $O(1)$ | $O(n)$ |
| put | $O(1)$ | $O(n)$ |

## LFU vs LRU vs LRU-K

| 特性 | LFU | LRU | LRU-K |
|------|-----|-----|-------|
| 淘汰策略 | 访问次数最少 | 最近最久未访问 | 历史K次访问 |
| 实现难度 | 复杂 | 简单 | 中等 |
| 突发热点 | 表现差 | 表现好 | 表现中等 |
| 频率污染 | 可能（历史热点残留） | 无 | 可缓解 |

## 应用场景

1. **数据库缓冲池**：如 InnoDB Buffer Pool
2. **CPU Cache 模拟**
3. **CDN 内容缓存**
4. **浏览器缓存策略**
5. **Redis 内存淘汰策略**：LFU 是 Redis 4.0+ 支持的淘汰策略

## 参考

- [An O(1) algorithm for implementing the LFU cache eviction scheme](http://dhruvbird.com/lfu.pdf)
- [LeetCode 460 - LFU Cache](https://leetcode.cn/problems/lfu-cache/)
- [LFU Cache - Webcoderspeed](https://webcoderspeed.com/blog/dsa-system-design/02-lfu-cache)
