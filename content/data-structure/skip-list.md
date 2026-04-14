---
title: Skip List（跳跃表）
date: 2026-04-13
description: 跳跃表是一种概率型数据结构，支持平均O(log n)复杂度的搜索、插入和删除操作
tags:
  - data-structure
  - skip-list
  - probabilistic
draft: false
permalink:
---

## 概述

跳跃表（Skip List）是一种概率型数据结构，由 William Pugh 于1989年发明。它通过在有序链表上建立多层索引，实现类似平衡树的搜索效率，同时具有更简单的实现和更好的并发特性。

> "Skip lists are a probabilistic alternative to balanced trees. Skip list algorithms have the same asymptotic expected time bounds as balanced trees and are simpler, faster and use less space." — William Pugh

## 结构原理

### 多层链表结构

跳跃表由多层有序链表组成：

```
层3:  HEAD ──────────────────────────► NIL
层2:  HEAD ─────────►───────►──────► NIL  
层1:  HEAD ────►───►───►───►───►───► NIL
层0:  HEAD ──►──►──►──►──►──►──►──►──► NIL
       1    2    3    4    5    6    7   (数据)
```

- **底层（层0）**：普通有序链表，包含所有元素
- **上层索引**：每层是下层的"快速通道"，跳过部分中间节点
- **概率晋升**：节点按固定概率 $p$（通常 $p=1/2$ 或 $p=1/4$）晋升到更高层

### 搜索过程

搜索从最高层开始水平遍历，找到两个连续节点（一个小于目标，一个大于等于目标）后，下降到下一层继续搜索：

```
Search(x):
    node = head
    for level = maxLevel down to 0:
        while node.next[level].key < x:
            node = node.next[level]
    node = node.next[0]
    return node.key == x ? node : NOT_FOUND
```

## 时间复杂度

| 操作 | 平均复杂度 | 最坏复杂度 |
|------|-----------|-----------|
| 搜索 | $O(\log n)$ | $O(n)$ |
| 插入 | $O(\log n)$ | $O(n)$ |
| 删除 | $O(\log n)$ | $O(n)$ |
| 空间 | $O(n)$ | $O(n \log n)$ |

## C++ 实现

```cpp
#include <random>
#include <vector>

template<typename T>
class SkipList {
    struct Node {
        T key;
        std::vector<Node*> next;  // 每层的后继指针
        int level;                 // 节点层数
        Node(T k, int l) : key(k), level(l) {
            next.resize(l + 1, nullptr);
        }
    };
    
    Node* head;           // 头节点（负无穷）
    int maxLevel;         // 最大层数
    int size;             // 元素数量
    std::mt19937 rng;     // 随机数生成器
    
    // 随机生成层数（概率晋升）
    int randomLevel() {
        int level = 0;
        while (level < maxLevel && (rng() % 2 == 0)) {
            level++;
        }
        return level;
    }

public:
    SkipList(int maxLvl = 16) : maxLevel(maxLvl), size(0), rng(std::random_device{}()) {
        head = new Node(T{}, maxLevel);
    }
    
    ~SkipList() {
        Node* curr = head;
        while (curr) {
            Node* next = curr->next[0];
            delete curr;
            curr = next;
        }
    }
    
    bool search(T key) {
        Node* curr = head;
        for (int i = maxLevel; i >= 0; i--) {
            while (curr->next[i] && curr->next[i]->key < key) {
                curr = curr->next[i];
            }
        }
        curr = curr->next[0];
        return curr && curr->key == key;
    }
    
    void insert(T key) {
        Node* curr = head;
        std::vector<Node*> update(maxLevel + 1, nullptr);
        
        // 找到每层需要更新的位置
        for (int i = maxLevel; i >= 0; i--) {
            while (curr->next[i] && curr->next[i]->key < key) {
                curr = curr->next[i];
            }
            update[i] = curr;
        }
        curr = curr->next[0];
        
        if (curr && curr->key == key) return;  // 已存在
        
        int newLevel = randomLevel();
        Node* newNode = new Node(key, newLevel);
        
        for (int i = 0; i <= newLevel; i++) {
            newNode->next[i] = update[i]->next[i];
            update[i]->next[i] = newNode;
        }
        size++;
    }
    
    bool erase(T key) {
        Node* curr = head;
        std::vector<Node*> update(maxLevel + 1, nullptr);
        
        for (int i = maxLevel; i >= 0; i--) {
            while (curr->next[i] && curr->next[i]->key < key) {
                curr = curr->next[i];
            }
            update[i] = curr;
        }
        curr = curr->next[0];
        
        if (!curr || curr->key != key) return false;
        
        for (int i = 0; i <= curr->level; i++) {
            update[i]->next[i] = curr->next[i];
        }
        delete curr;
        size--;
        return true;
    }
};
```

## 应用场景

### 著名应用

| 项目 | 用途 |
|------|------|
| **Redis** | 有序集合（ZSet）的实现 |
| **MemSQL** | 主索引结构（无锁跳跃表） |
| **LevelDB/RocksDB** | MemTable 默认实现 |
| **Java ConcurrentSkipListMap/Set** | 并发安全集合 |
| **Linux CPU调度器（MuQSS）** | 优先级队列 |
| **Cyrus IMAP** | 数据库后端 |
| **Lucene** | 倒排表搜索 |

### 为什么选择跳跃表而非平衡树

1. **实现简单**：无需复杂的旋转操作
2. **更好的并发性**：部分操作可无锁进行
3. **易于扩展**：可轻松调整层数参数
4. **内存效率**：节点独立管理指针，无需额外平衡信息

## 变体与优化

### 确定性跳跃表

使用确定性算法（如完美哈希）代替随机晋升，避免最坏情况。

### 索引扩展

为每条链接存储宽度（width），实现 $O(\log n)$ 的随机访问：

```
宽度为k的链接：跨度为k个底层节点
通过累加宽度可找到第i个元素
```

## 参考

- [Wikipedia: Skip List](https://en.wikipedia.org/wiki/Skip_list)
- Pugh, W. (1990). "Skip lists: A probabilistic alternative to balanced trees"
- [Open Data Structures - Chapter 4](http://opendatastructures.org/versions/edition-0.1e/ods-java/4_Skiplists.html)
