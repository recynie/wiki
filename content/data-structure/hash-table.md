---
title: 哈希表
date: 2026-03-31
description: 哈希表（Hash Table）数据结构
tags:
  - hash
  - data-structure
draft: false
permalink:
---

## 定义

哈希表（Hash Table）是一种以**键-值对**（key-value）形式存储数据的数据结构。它通过**哈希函数**（Hash Function）将键映射到数组的特定位置，从而实现 $O(1)$ 平均时间复杂度的查找、插入和删除操作。[^1]

哈希表的核心思想是：
- **哈希函数**：将键转换为数组索引
- **冲突处理**：当多个键映射到同一位置时，需要处理冲突

## 核心特性

### 时间复杂度

| 操作 | 平均时间复杂度 | 最坏时间复杂度 |
|------|---------------|---------------|
| 查找（search） | $O(1)$ | $O(n)$ |
| 插入（insert） | $O(1)$ | $O(n)$ |
| 删除（delete） | $O(1)$ | $O(n)$ |

在理想情况下，哈希表的所有操作都是常数时间的。但在最坏情况下（如所有键都哈希到同一位置），会退化为线性查找。

### 空间复杂度

- 空间复杂度：$O(n)$，其中 $n$ 是存储的键值对数量

## 哈希函数

### 好的哈希函数应满足

1. **均匀性**：哈希值应均匀分布在哈希表中，减少冲突
2. **确定性**：相同的键必须产生相同的哈希值
3. **高效性**：计算哈希值应在常数时间内完成

### 常用哈希函数

```cpp
// 整数哈希（直接取模）
int hash1(int key, int tableSize) {
    return abs(key) % tableSize;
}

// 字符串哈希（多项式哈希）
int hash2(const string& s, int tableSize) {
    int hash = 0;
    for (char c : s) {
        hash = hash * 31 + c;  // 31 是常用的质数
    }
    return abs(hash) % tableSize;
}

// 组合多个整数键
int hash3(int key1, int key2, int tableSize) {
    return ((long long)key1 * 31 + key2) % tableSize;
}
```

## 冲突处理

当两个或多个键映射到同一位置时，需要冲突处理策略。

### 链地址法（Chaining）

将哈希到同一位置的元素用链表存储。

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Node {
    string key;
    int value;
    Node* next;
    Node(string k, int v) : key(k), value(v), next(nullptr) {}
};

class HashTable {
private:
    vector<Node*> table;
    int tableSize;
    
    int hash(const string& key) {
        int hash = 0;
        for (char c : key) {
            hash = hash * 31 + c;
        }
        return abs(hash) % tableSize;
    }
    
public:
    HashTable(int size = 100) : tableSize(size) {
        table.resize(tableSize, nullptr);
    }
    
    ~HashTable() {
        for (Node* node : table) {
            while (node) {
                Node* temp = node;
                node = node->next;
                delete temp;
            }
        }
    }
    
    void insert(const string& key, int value) {
        int idx = hash(key);
        Node* node = table[idx];
        
        // 检查是否已存在
        while (node) {
            if (node->key == key) {
                node->value = value;
                return;
            }
            node = node->next;
        }
        
        // 头插法
        Node* newNode = new Node(key, value);
        newNode->next = table[idx];
        table[idx] = newNode;
    }
    
    int* find(const string& key) {
        int idx = hash(key);
        Node* node = table[idx];
        
        while (node) {
            if (node->key == key) {
                return &node->value;
            }
            node = node->next;
        }
        return nullptr;
    }
    
    void remove(const string& key) {
        int idx = hash(key);
        Node* node = table[idx];
        Node* prev = nullptr;
        
        while (node) {
            if (node->key == key) {
                if (prev) prev->next = node->next;
                else table[idx] = node->next;
                delete node;
                return;
            }
            prev = node;
            node = node->next;
        }
    }
};
```

### 开放地址法（Open Addressing）

当发生冲突时，探测其他可用位置。

```cpp
#include <bits/stdc++.h>
using namespace std;

// 线性探测
class LinearProbingHash {
private:
    vector<pair<string, int>> table;
    int tableSize;
    int count;
    
    int hash(const string& key) {
        int hash = 0;
        for (char c : key) {
            hash = hash * 31 + c;
        }
        return abs(hash) % tableSize;
    }
    
public:
    LinearProbingHash(int size = 100) : tableSize(size), count(0) {
        table.resize(tableSize, {"", 0});
    }
    
    void insert(const string& key, int value) {
        if (count >= tableSize * 0.7) {
            // 扩容（省略实现）
            return;
        }
        
        int idx = hash(key);
        while (!table[idx].first.empty()) {
            if (table[idx].first == key) {
                table[idx].second = value;
                return;
            }
            idx = (idx + 1) % tableSize;  // 线性探测
        }
        table[idx] = {key, value};
        count++;
    }
    
    int* find(const string& key) {
        int idx = hash(key);
        int startIdx = idx;
        
        while (!table[idx].first.empty()) {
            if (table[idx].first == key) {
                return &table[idx].second;
            }
            idx = (idx + 1) % tableSize;
            if (idx == startIdx) break;
        }
        return nullptr;
    }
};
```

## STL unordered_map

C++ STL 提供了 `unordered_map` 和 `unordered_set`，基于哈希表实现。

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    // unordered_map 基本用法
    unordered_map<string, int> umap;
    
    // 插入
    umap["apple"] = 1;
    umap["banana"] = 2;
    umap.insert({"cherry", 3});
    umap.emplace("date", 4);
    
    // 查找
    if (umap.find("apple") != umap.end()) {
        cout << "apple: " << umap["apple"] << endl;
    }
    
    // 访问（注意：如果键不存在会插入默认值）
    cout << "grape: " << umap["grape"] << endl;  // 输出 0
    
    // 删除
    umap.erase("banana");
    
    // 遍历
    for (const auto& [key, value] : umap) {
        cout << key << ": " << value << endl;
    }
    
    // unordered_set
    unordered_set<int> uset;
    uset.insert(1);
    uset.insert(2);
    uset.insert(3);
    cout << "Contains 2: " << (uset.count(2) ? "yes" : "no") << endl;
    
    // 自定义键类型
    struct Point {
        int x, y;
        bool operator==(const Point& other) const {
            return x == other.x && y == other.y;
        }
    };
    
    struct PointHash {
        size_t operator()(const Point& p) const {
            return hash<int>()(p.x * 31 + p.y);
        }
    };
    
    unordered_map<Point, int, PointHash> pointMap;
    pointMap[{1, 2}] = 10;
    
    return 0;
}
```

### unordered_map 常用函数

| 函数 | 功能 |
|------|------|
| `find(key)` | 查找键，返回迭代器 |
| `count(key)` | 返回键出现的次数（0 或 1） |
| `operator[key]` | 访问或插入键值对 |
| `insert({key, value})` | 插入键值对 |
| `erase(key)` | 删除键值对 |
| `size()` | 返回元素个数 |
| `empty()` | 判断是否为空 |

## 应用场景

### 字符串计数

```cpp
// 统计字符出现频率
string s = "hello world";
unordered_map<char, int> freq;
for (char c : s) {
    freq[c]++;
}
```

### 布隆过滤器（Bloom Filter）

使用多个哈希函数和位数组实现的高效存在性检测数据结构，可用于缓存穿透防护。

### LRU Cache

使用哈希表 + 双向链表实现 $O(1)$ 的 LRU 缓存。

### 集合运算

求两个集合的交集、并集、差集。

## 哈希冲突攻击

恶意用户可能通过构造特殊输入使哈希表退化为链表，导致性能下降。防御措施包括：

- 使用密码学安全的哈希函数（如 `std::hash<string>` 的改进版本）
- 随机化哈希种子
- 限制单个桶的链表长度

## 参考资料

[^1]: 哈希表的概念由汉斯·彼得·卢恩（Hans Peter Luhn）在1953年提出，是计算机科学中最重要的数据结构之一。
