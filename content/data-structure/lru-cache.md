---
title: LRU Cache
date: 2026-04-12
description: 最近最少使用缓存的实现原理，使用哈希表与双向链表达到O(1)操作复杂度
tags:
  - cache
  - data-structure
  - hash-table
  - linked-list
draft: false
permalink:
---

## 1. 定义

**LRU Cache（Least Recently Used Cache，最近最少使用缓存）** 是一种常用的缓存淘汰策略。当缓存容量满时，淘汰最近最少使用的元素，为新元素腾出空间。

### 核心思想

- **时间局部性原理**：最近访问的数据很可能在未来也被频繁访问
- **保留热点数据**：保持最近访问的数据，淘汰长时间未访问的数据
- **O(1) 操作要求**：缓存的 `get` 和 `put` 操作都必须达到常数时间复杂度

### 应用场景

- CPU 缓存淘汰
- 浏览器页面缓存
- 数据库查询缓存
- 分布式系统本地缓存

## 2. 缓存操作

LRU Cache 需要支持两种核心操作：

| 操作 | 描述 | 时间复杂度要求 |
|------|------|----------------|
| `get(key)` | 获取指定键对应的值，若不存在返回 -1 | O(1) |
| `put(key, value)` | 插入或更新键值对，若超过容量则淘汰最久未使用的项 | O(1) |

### 淘汰规则

当缓存满时，调用 `put` 需要：
1. 删除**最久未使用**的条目（链表尾部）
2. 插入新条目到链表头部

## 3. 为什么朴素方法无法达到 O(1)

### 方法一：数组 + 线性查找

```cpp
vector<pair<int, int>> cache;  // {key, value}
```

| 操作 | 实现 | 时间复杂度 |
|------|------|------------|
| `get` | 线性扫描数组查找 key | O(n) |
| `put` | 查找是否已存在，不存在则插入数组尾部，容量满则淘汰 | O(n) |

**问题**：每次操作都需要 O(n) 扫描，无法满足要求。

### 方法二：单向链表 + 记录尾部

```cpp
struct Node {
    int key, value;
    Node* next;
};
Node* head;  // 头部插入，尾部淘汰
```

| 操作 | 实现 | 时间复杂度 |
|------|------|------------|
| `get` | 需要从头遍历查找 key | O(n) |
| `put` | 插入头部 O(1)，删除尾部 O(1) | get O(n) |

**问题**：单向链表无法快速找到某个节点的前驱节点来删除它。

### 方法三：数组记录位置 + 单向链表

尝试用数组记录每个 key 对应的链表位置：

| 操作 | 实现 | 时间复杂度 |
|------|------|------------|
| `get` | O(1) 查位置，但移动元素需要 O(n) | O(n) |

**问题**：数组位置记录在删除/移动元素后需要更新，复杂度高。

## 4. 最优解：哈希表 + 双向链表

### 核心思想

| 数据结构 | 作用 |
|----------|------|
| **哈希表** | O(1) 时间找到指定节点 |
| **双向链表** | O(1) 时间完成插入、删除、移动节点 |

### 双向链表的优势

1. **O(1) 插入**：在头部插入新节点
2. **O(1) 删除**：已知节点指针，直接修改前后的指针
3. **O(1) 移动**：将节点从原位置删除，插入到头部
4. **O(1) 淘汰**：尾部节点就是最久未使用的

### 数据结构设计

```
                            最近使用
                               ↓
head <-> [node1] <-> [node2] <-> [node3] <-> tail
                               ↑
                          最久未使用
```

- **head**：指向最近使用的节点（虚拟头节点）
- **tail**：指向最久未使用的节点（虚拟尾节点）
- **越靠近 head 表示越最近使用，越靠近 tail 表示越久未使用**

## 5. C++ 实现

```cpp
#include <bits/stdc++.h>
using namespace std;

/**
 * LRU Cache 实现
 * 数据结构：哈希表 + 双向链表
 * 时间复杂度：get O(1), put O(1)
 * 空间复杂度：O(capacity)
 */
class LRUCache {
private:
    // 双向链表节点结构
    struct Node {
        int key;        // 需要存储 key 以便在删除时能从哈希表中移除
        int value;
        Node* prev;     // 前驱指针
        Node* next;     // 后继指针
        
        Node(int k = 0, int v = 0) : key(k), value(v), prev(nullptr), next(nullptr) {}
    };
    
    int capacity;           // 缓存容量
    unordered_map<int, Node*> mp;  // 哈希表：key -> 节点指针
    Node* head;             // 虚拟头节点（最近使用的节点）
    Node* tail;             // 虚拟尾节点（最久未使用的节点）

    // 将节点移动到链表头部（最近使用的位置）
    void moveToFront(Node* node) {
        if (node == head->next) return;  // 已经在头部，无需移动
        
        // 1. 从原位置删除节点
        node->prev->next = node->next;
        node->next->prev = node->prev;
        
        // 2. 插入到头部
        node->next = head->next;
        node->prev = head;
        head->next->prev = node;
        head->next = node;
    }
    
    // 删除链表尾部节点（最久未使用的）
    void removeTail() {
        if (tail->prev == head) return;  // 缓存为空
        
        Node* node = tail->prev;  // 要删除的节点
        node->prev->next = tail;
        tail->prev = node->prev;
        
        mp.erase(node->key);      // 从哈希表中删除
        delete node;              // 释放内存
    }

public:
    LRUCache(int cap) : capacity(cap) {
        // 创建虚拟头尾节点（哨兵节点，简化边界处理）
        head = new Node();
        tail = new Node();
        head->next = tail;
        tail->prev = head;
    }
    
    ~LRUCache() {
        Node* cur = head;
        while (cur) {
            Node* next = cur->next;
            delete cur;
            cur = next;
        }
    }
    
    /**
     * 获取缓存值
     * @param key 查找的键
     * @return 如果存在返回对应值，否则返回 -1
     */
    int get(int key) {
        auto it = mp.find(key);
        if (it == mp.end()) {
            return -1;  // 缓存未命中
        }
        
        Node* node = it->second;
        moveToFront(node);  // 将访问的节点移到头部
        return node->value;
    }
    
    /**
     * 插入或更新缓存
     * @param key 键
     * @param value 值
     */
    void put(int key, int value) {
        Node* node = nullptr;
        
        auto it = mp.find(key);
        if (it != mp.end()) {
            // 键已存在，更新值并移到头部
            node = it->second;
            node->value = value;
            moveToFront(node);
        } else {
            // 键不存在，创建新节点
            node = new Node(key, value);
            mp[key] = node;
            
            // 插入到头部
            node->next = head->next;
            node->prev = head;
            head->next->prev = node;
            head->next = node;
            
            // 检查是否超出容量
            if ((int)mp.size() > capacity) {
                removeTail();  // 淘汰最久未使用的
            }
        }
    }
};
```

### 完整测试代码

```cpp
#include <bits/stdc++.h>
using namespace std;

class LRUCache {
private:
    struct Node {
        int key, value;
        Node *prev, *next;
        Node(int k = 0, int v = 0) : key(k), value(v), prev(nullptr), next(nullptr) {}
    };
    
    int capacity;
    unordered_map<int, Node*> mp;
    Node *head, *tail;
    
    void moveToFront(Node* node) {
        if (node == head->next) return;
        node->prev->next = node->next;
        node->next->prev = node->prev;
        node->next = head->next;
        node->prev = head;
        head->next->prev = node;
        head->next = node;
    }
    
    void removeTail() {
        if (tail->prev == head) return;
        Node* node = tail->prev;
        node->prev->next = tail;
        tail->prev = node->prev;
        mp.erase(node->key);
        delete node;
    }

public:
    LRUCache(int cap) : capacity(cap) {
        head = new Node();
        tail = new Node();
        head->next = tail;
        tail->prev = head;
    }
    
    ~LRUCache() {
        Node* cur = head;
        while (cur) {
            Node* next = cur->next;
            delete cur;
            cur = next;
        }
    }
    
    int get(int key) {
        auto it = mp.find(key);
        if (it == mp.end()) return -1;
        moveToFront(it->second);
        return it->second->value;
    }
    
    void put(int key, int value) {
        Node* node = nullptr;
        auto it = mp.find(key);
        if (it != mp.end()) {
            node = it->second;
            node->value = value;
            moveToFront(node);
        } else {
            node = new Node(key, value);
            mp[key] = node;
            node->next = head->next;
            node->prev = head;
            head->next->prev = node;
            head->next = node;
            if ((int)mp.size() > capacity) {
                removeTail();
            }
        }
    }
};

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    // 测试用例
    LRUCache cache(2);  // 容量为 2
    
    cache.put(1, 1);
    cache.put(2, 2);
    cout << cache.get(1) << endl;  // 返回 1
    
    cache.put(3, 3);  // 淘汰 key=2（最久未使用）
    cout << cache.get(2) << endl;  // 返回 -1（已被淘汰）
    
    cache.put(4, 4);  // 淘汰 key=1（最久未使用）
    cout << cache.get(1) << endl;  // 返回 -1（已被淘汰）
    cout << cache.get(3) << endl;  // 返回 3
    cout << cache.get(4) << endl;  // 返回 4
    
    return 0;
}
```

**输出**：
```
1
-1
-1
3
4
```

## 6. 逐步执行示例

以容量为 2 的 LRU Cache 为例：

### 初始状态

```
head <-> tail
```

### 操作序列

| 操作 | 数据结构状态 | 说明 |
|------|-------------|------|
| `put(1, 1)` | head <-> [1:1] <-> tail | 插入节点 1 |
| `put(2, 2)` | head <-> [2:2] <-> [1:1] <-> tail | 插入节点 2 |
| `get(1)` | head <-> [1:1] <-> [2:2] <-> tail | 节点 1 移到头部 |
| `put(3, 3)` | head <-> [3:3] <-> [1:1] <-> tail | 节点 2 被淘汰 |
| `get(2)` | head <-> [3:3] <-> [1:1] <-> tail | 返回 -1，节点 2 不存在 |
| `put(4, 4)` | head <-> [4:4] <-> [3:3] <-> tail | 节点 1 被淘汰 |
| `get(1)` | head <-> [4:4] <-> [3:3] <-> tail | 返回 -1，节点 1 不存在 |
| `get(3)` | head <-> [3:3] <-> [4:4] <-> tail | 返回 3，节点 3 移到头部 |
| `get(4)` | head <-> [4:4] <-> [3:3] <-> tail | 返回 4，节点 4 移到头部 |

### 详细状态变化

**Step 1: `put(1, 1)`**
```
head <-> [1:1] <-> tail
mp: {1 -> node1}
```

**Step 2: `put(2, 2)`**
```
head <-> [2:2] <-> [1:1] <-> tail
mp: {1 -> node1, 2 -> node2}
```

**Step 3: `get(1)` - 找到节点 1，移到头部**
```
head <-> [1:1] <-> [2:2] <-> tail
mp: {1 -> node1, 2 -> node2}
```

**Step 4: `put(3, 3)` - 插入新节点，淘汰最久未使用的（节点 2）**
```
head <-> [3:3] <-> [1:1] <-> tail
mp: {1 -> node1, 3 -> node3}
```

**Step 5: `get(2)` - 返回 -1（节点 2 已被淘汰）**

**Step 6: `put(4, 4)` - 插入新节点，淘汰最久未使用的（节点 1）**
```
head <-> [4:4] <-> [3:3] <-> tail
mp: {3 -> node3, 4 -> node4}
```

**Step 7-9: 后续 get 操作**
```
get(1) -> -1
get(3) -> head <-> [3:3] <-> [4:4] <-> tail -> 返回 3
get(4) -> head <-> [4:4] <-> [3:3] <-> tail -> 返回 4
```

## 7. 复杂度分析

| 操作 | 时间复杂度 | 空间复杂度 |
|------|-----------|-----------|
| `get(key)` | O(1) | - |
| `put(key, value)` | O(1) | - |
| 总空间 | - | O(capacity) |

### 各操作详细分析

| 操作 | 哈希表 | 双向链表 | 总计 |
|------|--------|----------|------|
| `get` | 查找 O(1) | 移动节点 O(1) | O(1) |
| `put`（已存在） | 查找 O(1) | 更新值 + 移动 O(1) | O(1) |
| `put`（不存在） | 插入 O(1) | 插入头部 O(1)，容量超限删除尾部 O(1) | O(1) |

## 8. Python 实现：OrderedDict

Python 的 `collections.OrderedDict` 提供了类似功能，内部实现基于双向链表。

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()
    
    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        # 移动到末尾（最近使用）
        self.cache.move_to_end(key)
        return self.cache[key]
    
    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            # 更新并移动到末尾
            self.cache.move_to_end(key)
            self.cache[key] = value
        else:
            # 插入新节点
            self.cache[key] = value
            # 超容量则删除头部（最久未使用）
            if len(self.cache) > self.capacity:
                self.cache.popitem(last=False)
```

### 使用示例

```python
cache = LRUCache(2)
cache.put(1, 1)
cache.put(2, 2)
print(cache.get(1))  # 返回 1
cache.put(3, 3)      # 淘汰 key=2
print(cache.get(2))  # 返回 -1
cache.put(4, 4)      # 淘汰 key=1
print(cache.get(1))  # 返回 -1
print(cache.get(3))  # 返回 3
print(cache.get(4))  # 返回 4
```

## 9. 关键点总结

### 面试核心要点

1. **为什么需要双向链表**：单向链表无法 O(1) 删除任意节点
2. **为什么需要哈希表**：链表查找需要 O(n)，需要哈希表实现 O(1) 查找
3. **哨兵节点的作用**：简化边界处理（空链表、只有一个节点等情况）
4. **存储 key 的原因**：删除尾部节点时需要从哈希表中移除

### 常见变形

| 变形 | 区别 |
|------|------|
| LFU Cache | 淘汰最少使用的（按频率） |
| FIFO Cache | 淘汰最早进入的（按时间） |
| TTL Cache | 淘汰超过生存时间的 |

### 延伸阅读

- [[hash-table|哈希表]]：了解哈希表的 O(1) 查找原理
- [[linked-list|链表]]：了解双向链表的操作

## 参考资料

[^1]: GeeksforGeeks. *LRU Cache*. https://www.geeksforgeeks.org/lru-cache-implementation/

[^2]: AlgoTree. *LRU Cache*. https://www.geeksforgeeks.org/lru-cache-implementation/
