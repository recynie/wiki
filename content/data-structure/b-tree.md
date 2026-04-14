---
title: B树与B+树
date: 2026-04-06
description: B树与B+树——大规模数据存储和数据库索引的核心数据结构
tags:
  - data-structure
  - b-tree
  - database
draft: false
permalink:
---

## 定义

B 树（B-Tree）是一种自平衡的多路搜索树，每个节点可以拥有多个子节点和键值。B+ 树（B+ Tree）是 B 树的变体，所有数据都存储在叶子节点，内部节点仅存储键值。[^1]

它们被广泛应用于：
- **数据库索引**：MySQL InnoDB 使用 B+ 树
- **文件系统**：如 NTFS、HFS+
- **键值存储**：RocksDB、LevelDB

---

## B 树

### 性质

对于一棵 $m$ 阶（每个节点最多有 $m$ 个子节点）的 B 树：

| 性质 | 说明 |
|------|------|
| 每个节点最多 $m$ 个子节点 | |
| 每个内部节点至少有 $\lceil m/2 \rceil$ 个子节点 | 除根节点外 |
| 根节点至少有两个子节点 | 如果不是叶子 |
| 所有叶子节点在同一层 | |
| 每个节点最多 $m-1$ 个键值 | |

### 节点结构

```cpp
struct BTreeNode {
    bool isLeaf;           // 是否为叶子节点
    int n;                 // 当前键值数量
    vector<int> keys;       // 键值（最多 m-1 个）
    vector<BTreeNode*> children;  // 子节点指针（最多 m 个）
};
```

### 搜索

类似于二叉搜索树，但多路分支：

```cpp
BTreeNode* search(BTreeNode* node, int key) {
    int i = 0;
    while (i < node->n && key > node->keys[i]) i++;
    
    if (i < node->n && key == node->keys[i])
        return node;  // 找到
    
    if (node->isLeaf)
        return nullptr;  // 未找到
    
    return search(node->children[i], key);
}
```

### 插入

1. 找到合适的叶子节点插入
2. 如果叶子节点键值数量超过 $m-1$，则 **分裂**（split）：
   - 将节点分成两部分
   - 中间的键值上移到父节点
   - 如果父节点也满，则继续分裂向上传播

### 分裂操作

```
节点满时（有 m-1 个键）分裂为两个节点各 ⌈(m-1)/2⌉ 个键
中间键值提升到父节点
```

### 删除

删除操作较复杂，需要：
1. 如果键在叶子节点且叶子节点有多余键值，直接删除
2. 否则需要 **合并** 或 **借来**（borrow）键值以维持平衡

---

## B+ 树

### 与 B 树的主要区别

| 特征 | B 树 | B+ 树 |
|------|------|-------|
| 数据存储位置 | 所有节点 | 仅叶子节点 |
| 叶子节点连接 | 无 | 有序链表 |
| 查询稳定性 | 可能在中途找到 | 必须到叶子 |
| 空间效率 | 较低（所有节点都可能存数据） | 较高（仅叶子存数据） |
| 范围查询 | 需要遍历树 | 叶子链表 $O(1)$ |

### 结构示意

```
                    B+ 树 (m=3)
                         
         [ 20 ]                    [ 40 ]
        /   |   \                  /   |   \
   [5,10] [20] [30,35] [40] [50,60] [70,80] [90]
    |      |      |      |      |       |       |
    v      v      v      v      v       v       v
  数据   数据    数据   数据   数据    数据    数据
```

所有数据通过叶子节点的有序链表连接，支持高效的范围查询。

---

## C++ 实现（B+ 树核心部分）

```cpp
#include <bits/stdc++.h>
using namespace std;

const int M = 4;  // 阶数

struct BPlusNode {
    bool isLeaf;
    int n;  // 当前键值数量
    vector<int> keys;  // 键值
    vector<BPlusNode*> children;  // 子节点指针
    BPlusNode* next;  // 叶子节点的下一个叶子（用于范围查询）
    vector<int> values;  // 叶子节点存储的值
    
    BPlusNode(bool leaf = false) : isLeaf(leaf), n(0), next(nullptr) {}
};

class BPlusTree {
private:
    BPlusNode* root;

    void splitChild(BPlusNode* parent, int idx) {
        BPlusNode* child = parent->children[idx];
        BPlusNode* newNode = new BPlusNode(child->isLeaf);
        
        int mid = (M - 1) / 2;
        
        if (child->isLeaf) {
            // 叶子节点分裂
            newNode->n = child->n - mid - 1;
            child->n = mid;
            
            for (int i = mid + 1; i < child->n + mid + 1; i++) {
                newNode->keys.push_back(child->keys[i]);
                newNode->values.push_back(child->values[i - (mid + 1)]);
            }
            
            newNode->next = child->next;
            child->next = newNode;
        } else {
            // 内部节点分裂
            newNode->n = child->n - mid - 1;
            child->n = mid;
            
            for (int i = mid + 1; i < child->n + mid + 1; i++) {
                newNode->keys.push_back(child->keys[i]);
                newNode->children.push_back(child->children[i]);
            }
            newNode->children.push_back(child->children[child->n + mid + 1]);
            
            for (auto& c : newNode->children) {
                if (c) c->fa = newNode;
            }
        }
        
        parent->keys.insert(parent->keys.begin() + idx, child->keys[mid]);
        parent->children.insert(parent->children.begin() + idx + 1, newNode);
        parent->n++;
        
        if (!child->isLeaf) {
            child->keys.erase(child->keys.begin() + mid, child->keys.end());
        }
    }

    void insertNonFull(BPlusNode* node, int key, int value) {
        if (node->isLeaf) {
            // 叶子节点：找到合适位置插入
            int i = node->n - 1;
            while (i >= 0 && node->keys[i] > key) {
                i--;
            }
            node->keys.insert(node->keys.begin() + i + 1, key);
            node->values.insert(node->values.begin() + i + 1, value);
            node->n++;
        } else {
            // 内部节点：找到合适的子节点
            int i = 0;
            while (i < node->n && key > node->keys[i]) i++;
            
            if (node->children[i]->n == M - 1) {
                splitChild(node, i);
                if (key > node->keys[i]) i++;
            }
            insertNonFull(node->children[i], key, value);
        }
    }

public:
    BPlusTree() {
        root = new BPlusNode(true);
    }

    void insert(int key, int value) {
        if (root->n == M - 1) {
            // 根节点满，需要分裂
            BPlusNode* newRoot = new BPlusNode(false);
            newRoot->children.push_back(root);
            splitChild(newRoot, 0);
            root = newRoot;
        }
        insertNonFull(root, key, value);
    }

    // 范围查询 [l, r]
    vector<int> rangeQuery(int l, int r) {
        vector<int> result;
        BPlusNode* node = root;
        
        // 找到左边界所在叶子
        while (!node->isLeaf) {
            int i = 0;
            while (i < node->n && l > node->keys[i]) i++;
            node = node->children[i];
        }
        
        // 在叶子节点中遍历
        while (node) {
            for (int i = 0; i < node->n; i++) {
                if (node->keys[i] >= l && node->keys[i] <= r)
                    result.push_back(node->values[i]);
                else if (node->keys[i] > r)
                    return result;
            }
            node = node->next;
        }
        return result;
    }
};
```

---

## 时间复杂度

| 操作 | 平均时间复杂度 |
|------|---------------|
| 搜索 | $O(\log_m n)$ |
| 插入 | $O(\log_m n)$ |
| 删除 | $O(\log_m n)$ |
| 范围查询 | $O(\log_m n + k)$，$k$ 为结果数量 |

其中 $m$ 是 B 树的阶，由于 $m$ 通常较大（如 100-1000），实际查询深度非常小（通常 2-3 层）。

---

## 应用场景

### 数据库索引

```
CREATE INDEX idx ON table(column);

-- MySQL InnoDB 使用 B+ 树索引
-- 每个节点大小设为页大小（通常 16KB）
-- 三层 B+ 树可存储约 2000 万行数据
```

### 文件系统

- NTFS、HFS+ 使用 B+ 树管理文件目录
- ext4 使用 extent（B+ 树的变体）

### 内存数据库

- RocksDB、LevelDB 使用 B+ 树（或其变体）存储数据

---

## 参考资料

[^1]: Bayer, R., & McCreight, E. (1972). Organization and maintenance of large ordered indices. Acta Informatica, 1(3), 173-189.

[^2]: B树 - OI Wiki. https://oi-wiki.org/ds/b-tree/

[^3]: 数据库系统概论 - B+树索引原理
