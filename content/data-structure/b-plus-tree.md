---
title: B+ Tree
date: 2026-04-12
description: B+树的定义、结构、操作（搜索、插入、删除）及其在数据库索引中的应用
tags:
  - b-tree
  - data-structure
  - database
  - index
draft: false
permalink:
---

## 定义

B+ 树（B+ Tree）是一种 $m$ 阶（m-ary）的自平衡树数据结构，属于 B 树（B-Tree）的变体。其核心特点是**所有数据（键值对）都存储在叶子节点**，内部节点仅存储用于导航的键值。

作为一种多路搜索树，B+ 树特别适合**大规模数据存储和高效范围查询**的场景，是数据库索引和文件系统的核心数据结构。[^1]

### 与 B 树的核心区别

| 特征 | B 树 | B+ 树 |
|------|------|-------|
| 数据存储位置 | 所有节点（含内部节点） | 仅叶子节点 |
| 内部节点键值 | 键值 + 指针 | 仅键值（用于索引） |
| 叶子节点连接 | 无 | 有序双向链表 |
| 搜索稳定性 | 可能在中途找到 | 必查到叶子 |
| 范围查询效率 | 需要回溯遍历 | 叶子链表 $O(1)$ 遍历 |

---

## 结构

B+ 树的节点分为三种类型：**根节点（Root）**、**内部节点（Internal Node）** 和 **叶子节点（Leaf Node）**。

### 树结构示意

```
                    B+ 树结构示意图 (m=4)
                    
                         根节点
                          [30]
                         / | \
                        /  |  \
            内部节点  [15] [30] [50, 70]  ← 内部节点仅存键值
                    / | \   / \    / | \
                   /  |  \ /  \   /  |  \
            叶子→ L1  L2 L3  L4  L5  L6  L7   ← 所有数据在叶子
             ↓    ↓    ↓   ↓    ↓   ↓    ↓
           [5]  [15] [20] [30] [50] [70] [90]  ← 叶子节点存数据
             |_____|_____|_____|_____|_____|     ↓
                                            有序链表连接
```

### 节点结构

#### 内部节点

```
内部节点结构：

┌─────────────────────────────────────────────────────────┐
│  子节点指针0  │  键值0  │  子节点指针1  │  键值1  │ ... │
└─────────────────────────────────────────────────────────┘

n 个键值：k₀ < k₁ < ... < kₙ₋₁
n+1 个子指针：P₀, P₁, ..., Pₙ
```

- **键值数量**：$n$（满足 $\lceil m/2 \rceil - 1 \leq n \leq m-1$）
- **子指针数量**：$n + 1$
- **子节点范围**：$P_i$ 中的所有键值满足 $k_{i-1} < key < k_i$

#### 叶子节点

```
叶子节点结构：

┌─────────────────────────────────────────────────────────┐
│  数据0  │  键值0  │  数据1  │  键值1  │  数据2  │  键值2 │ ... │
└─────────────────────────────────────────────────────────┘
          ↕ 链表前向指针                                    ↕
        prev   next ────────────────────────────────► next
```

- **键值数量**：$n$（满足 $\lceil m/2 \rceil - 1 \leq n \leq m-1$）
- **数据/值**：可以是实际数据或 RID（记录标识符）
- **兄弟指针**：双向链表连接相邻叶子节点

---

## 关键性质

### 1. 完全平衡（Perfectly Balanced）

所有**叶子节点位于同一深度**，这确保了任何搜索路径的长度相同，查询时间可预测。

### 2. 高扇出（High Fanout）

典型的 B+ 树扇出度为 **100 到 1000+**，这意味着：
- 树的高度极低（通常 2-4 层）
- 每次 I/O 操作可读取大量节点
- 减少了磁盘访问次数

### 3. 最小占用率 50%

除根节点外，每个节点的**键值数量必须至少为 $\lceil m/2 \rceil - 1$**：

| 节点类型 | 最小键值数 | 最大键值数 |
|----------|-----------|-----------|
| 根节点 | 1 | $m - 1$ |
| 内部节点 | $\lceil m/2 \rceil - 1$ | $m - 1$ |
| 叶子节点 | $\lceil m/2 \rceil - 1$ | $m - 1$ |

这种设计保证了空间利用率不低于 50%。

### 4. 有序链表

叶子节点通过**双向链表**连接，支持：
- $O(1)$ 时间复杂度进行范围扫描
- 高效的顺序访问
- 便捷的相邻节点查找

---

## 搜索操作

B+ 树的搜索过程如下：

```
Search(key):
    node ← root
    
    while not leaf(node):
        i ← binarySearch(node.keys, key)
        if i < node.n and key == node.keys[i]:
            i ← i + 1  // 相同键值向右
        node ← node.children[i]
    
    // 在叶子节点中线性查找
    i ← binarySearch(node.keys, key)
    if i < node.n and key == node.keys[i]:
        return node.values[i]  // 找到
    return NOT_FOUND            // 未找到
```

### 搜索示例

```
        [20 | 50]
       /    |    \
   [5,10] [20] [30,35] [50] [60,70] [80,90]
   
搜索 key = 35:
1. 从根节点开始，35 > 20，指向右子树
2. 在第二层，20 < 35 < 50，指向中间节点
3. 到达叶子节点 [30,35]，二分查找找到 35
```

### 复杂度分析

- **时间复杂度**：$O(\log_m n)$
- **I/O 复杂度**：$O(\log_m n)$（每次访问可能产生一次磁盘 I/O）
- **搜索深度**：约 $\log_{m/2} n$（考虑最小占用率）

---

## 插入操作

### 插入算法

```
Insert(key, value):
    leaf ← findLeaf(key)
    
    if leaf.n < m - 1:
        insertIntoLeaf(leaf, key, value)
    else:
        splitLeaf(leaf, key, value)  // 分裂叶子
```

### 分裂操作详解

当叶子节点已满（$n = m - 1$）时，需要**分裂**：

```
叶子节点分裂示例 (m=4，最大3个键值):

分裂前: [10, 20, 30] ← 已满
        插入 25

分裂后:
    左叶子: [10, 20]        (保留较小的一半)
    右叶子: [30]            (较大的键值移到这里)
    
    父节点需要插入新的键值:
        原来的键值 [20, 30] → 上移 20 或 30
    
    链表更新:
    ... → [10,20] → [30] → ...
```

### 内部节点分裂

内部节点分裂时，**中间的键值上移到父节点**，左右两半形成新的子节点：

```
内部节点分裂示例:

分裂前 (m=4):
        [20 | 50 | 80]
        
分裂后:
        父节点: [50]
              /    \
    左节点: [20]    右节点: [80]
    
    上移 50 键值
```

### 插入传播

如果父节点在插入新键值后也满，则继续分裂，可能一直传播到根节点，导致**树高增加**。

---

## 删除操作

### 删除算法

```
Delete(key):
    leaf ← findLeaf(key)
    removeFromLeaf(leaf, key)
    
    if leaf.n < minKeys:  // 最小键值数 = ⌈m/2⌉ - 1
        if leftSibling.n > minKeys:
            borrowFromLeft(leaf, leftSibling)
        else if rightSibling.n > minKeys:
            borrowFromRight(leaf, rightSibling)
        else:
            merge(leaf, sibling)  // 合并
```

### 借来（Borrow）操作

当相邻兄弟节点有足够的键值时，可以**借来**一个键值：

```
借来操作示例:

删除后左叶子键值不足:
  左兄弟: [10, 20, 30]  →  借走后: [10, 20]
  当前叶: [40]          →  借来后: [30, 40]
  
  父节点键值调整: 原来的 [20|40] → [20|30]
```

### 合并（Merge）操作

当两个相邻节点都不足以借出时，进行**合并**：

```
合并操作示例:

两个叶子节点都只有 1 个键值:
  左叶子: [10]    右叶子: [50]
  
合并后:
  合并叶子: [10, 50]
  删除右叶子
  
  父节点键值调整: 删除原来的分隔键值
```

合并可能向上传播，导致树高减少。

---

## 批量加载（Bulk Loading）

对于已有大量有序数据需要建索引的场景，**批量加载**比逐条插入更高效。

### 算法思想

```
BulkLoad(sorted_keys, values):
    // 1. 计算叶子节点数量
    leafCount = ceil(n / (m - 1))
    
    // 2. 创建所有叶子节点（填充到最小占用率以上）
    for i in range(leafCount):
        create leaf with keys[(m-1)*i : (m-1)*(i+1)]
        link leaves in order
    
    // 3. 从底层向上构建内部节点
    currentLevel = leaves
    while currentLevel.size > 1:
        nextLevel = []
        for i in range(0, currentLevel.size, m):
            // 每 m 个子节点创建一个父节点
            parent = createInternalNode(
                keys: [child[0] for child in group],  // 第一个键值
                children: group
            )
            nextLevel.append(parent)
        currentLevel = nextLevel
    
    root = currentLevel[0]
```

### 效率优势

| 方法 | 时间复杂度 | I/O 次数 |
|------|-----------|---------|
| 逐条插入 | $O(n \log_m n)$ | $O(n \log_m n)$ |
| 批量加载 | $O(n)$ | $O(n / m)$ |

批量加载利用了**磁盘顺序读写**的特性，比随机插入快 10-100 倍。

---

## 性能优势

### B+ 树 vs B 树

| 特性 | B+ 树优势 | 原因 |
|------|----------|------|
| **范围查询** | ✅ 更高效 | 叶子链表 $O(1)$ 遍历 |
| **查询稳定性** | ✅ 稳定 | 始终查到叶子 |
| **空间效率** | ✅ 更高 | 内部节点不存储数据 |
| **I/O 效率** | ✅ 更好 | 内部节点更小，可缓存更多 |
| **点查询** | ≈ 相近 | 复杂度相同 |

### 理论复杂度

| 操作 | 时间复杂度 | 说明 |
|------|-----------|------|
| 搜索 | $O(\log_m n)$ | $m$ 为阶数 |
| 插入 | $O(\log_m n)$ | 含分裂 |
| 删除 | $O(\log_m n)$ | 含合并/借来 |
| 范围查询 | $O(\log_m n + k)$ | $k$ 为结果数 |
| 顺序访问 | $O(n)$ | 遍历所有叶子 |

---

## 应用场景

### 数据库索引

B+ 树是**关系型数据库**最常用的索引结构：

#### PostgreSQL

```sql
-- PostgreSQL 默认使用 B+ 树索引
CREATE INDEX idx_name ON table(column);
CREATE INDEX idx_composite ON table(col1, col2);

-- 索引类型
CREATE INDEX idx USING btree ON table(column);
```

- **PostgreSQL 12+**：使用 **B-Tree**，但优化后与 B+ 树性能相近
- 叶子节点存储 **TID（Tuple ID）** 指向实际数据行
- 支持 **Index Only Scan** 减少回表

#### MySQL (InnoDB)

```sql
-- MySQL InnoDB 使用 B+ 树
CREATE INDEX idx_name ON table(column);
PRIMARY KEY -> 聚簇索引 (clustered index)
```

- **聚簇索引**：主键索引的叶子节点存储完整行数据
- **二级索引**：叶子节点存储主键值，再回表查找

#### 索引层次计算

```
假设：
- 页大小 = 16KB
- 主键 BIGINT = 8 字节
- 每行记录指针 = 8 字节
- 每键值需要 16 字节（键 + 指针）

每个节点可存储: 16KB / 16 = 1024 个键值

三层 B+ 树容量:
- 第1层: 1 个根节点 (1024^0 = 1)
- 第2层: 1024 个内部节点 (1024^1 = 1024)
- 第3层: 1024 × 1024 = 1,048,576 个叶子节点

每个叶子 16KB，可存约 1000 条记录

总容量: ~10 亿条记录
```

### 文件系统

| 文件系统 | B+ 树应用 |
|----------|----------|
| **NTFS** | 使用 B+ 树管理文件目录（MFT） |
| **ext4** | 使用 HTree（extent B+ 树变体） |
| **HFS+** | 使用 B+ 树存储文件元数据 |
| **APFS** | 使用 B+ 树组织文件系统结构 |

#### NTFS MFT 示例

```
Master File Table (MFT) 结构:
├── $MFT (B+ 树根)
├── $MFTMirr
├── $LogFile
└── ...

文件查找: B+ 树搜索文件名 → 获取文件记录 → 读取文件数据
```

### 键值存储

- **RocksDB**、**LevelDB**：使用 LSM 树（基于 B+ 树变体）
- **Cassandra**：使用 **B+ 树**（本地索引）
- **Berkeley DB**：B+ 树作为主要访问方法

---

## C++ 实现

```cpp
#include <bits/stdc++.h>
using namespace std;

const int M = 4;  // B+ 树阶数

struct BPlusNode {
    bool isLeaf;              // 是否为叶子节点
    int n;                     // 当前键值数量
    vector<int> keys;          // 键值
    vector<BPlusNode*> children; // 子节点指针（内部节点）
    vector<int> values;        // 数据值（叶子节点）
    BPlusNode* next;           // 叶子节点的下一个叶子
    BPlusNode* prev;           // 叶子节点的上一个叶子
    
    BPlusNode(bool leaf = false) : isLeaf(leaf), n(0), next(nullptr), prev(nullptr) {}
};

class BPlusTree {
private:
    BPlusNode* root;
    int minKeys;  // 最小键值数 = ceil(M/2) - 1

    // 分裂叶子节点
    void splitLeaf(BPlusNode* leaf) {
        BPlusNode* newLeaf = new BPlusNode(true);
        int mid = (M - 1) / 2;  // 中间位置
        
        // 分裂键值
        newLeaf->n = leaf->n - mid - 1;
        leaf->n = mid + 1;
        
        // 移动键值和数据到新节点
        for (int i = mid + 1; i < leaf->n + newLeaf->n + 1; i++) {
            newLeaf->keys.push_back(leaf->keys[i]);
            newLeaf->values.push_back(leaf->values[i - (mid + 1)]);
        }
        
        // 更新链表指针
        newLeaf->next = leaf->next;
        if (leaf->next) leaf->next->prev = newLeaf;
        leaf->next = newLeaf;
        newLeaf->prev = leaf;
        
        // 插入新键值到父节点
        insertIntoParent(leaf, newLeaf->keys[0], newLeaf);
    }
    
    // 分裂内部节点
    void splitInternal(BPlusNode* internal, int idx) {
        BPlusNode* child = internal->children[idx];
        BPlusNode* newInternal = new BPlusNode(false);
        int mid = (M - 1) / 2;
        
        // 分裂
        newInternal->n = child->n - mid - 1;
        child->n = mid;
        
        // 移动键值和子节点
        for (int i = mid + 1; i < child->n + mid + 1; i++) {
            newInternal->keys.push_back(child->keys[i]);
            newInternal->children.push_back(child->children[i]);
        }
        newInternal->children.push_back(child->children[child->n + mid + 1]);
        
        // 移动中间的键值到新节点（作为第一个键值）
        int upKey = child->keys[mid];
        
        // 更新父节点
        internal->keys.insert(internal->keys.begin() + idx, upKey);
        internal->children.insert(internal->children.begin() + idx + 1, newInternal);
        internal->n++;
    }
    
    // 插入到父节点
    void insertIntoParent(BPlusNode* left, int key, BPlusNode* right) {
        BPlusNode* parent = nullptr;
        // 获取父节点（简化处理，实际需要维护父子关系）
        
        if (left == root) {
            // 创建新根
            BPlusNode* newRoot = new BPlusNode(false);
            newRoot->keys.push_back(key);
            newRoot->children.push_back(left);
            newRoot->children.push_back(right);
            newRoot->n = 1;
            root = newRoot;
            return;
        }
        
        // 找到父节点并插入（简化实现）
        // 实际需要遍历或维护 parent 指针
    }

    // 查找键应所在的叶子节点
    BPlusNode* findLeaf(int key) {
        BPlusNode* node = root;
        while (!node->isLeaf) {
            int i = 0;
            while (i < node->n && key > node->keys[i]) i++;
            node = node->children[i];
        }
        return node;
    }

public:
    BPlusTree() {
        root = new BPlusNode(true);
        minKeys = (M + 1) / 2 - 1;
    }

    // 插入键值对
    void insert(int key, int value) {
        BPlusNode* leaf = findLeaf(key);
        
        // 查找插入位置
        int i = 0;
        while (i < leaf->n && leaf->keys[i] < key) i++;
        
        // 插入键值和数据
        leaf->keys.insert(leaf->keys.begin() + i, key);
        leaf->values.insert(leaf->values.begin() + i, value);
        leaf->n++;
        
        // 检查是否需要分裂
        if (leaf->n == M) {
            splitLeaf(leaf);
        }
    }

    // 范围查询 [l, r]
    vector<int> rangeQuery(int l, int r) {
        vector<int> result;
        BPlusNode* leaf = findLeaf(l);
        
        while (leaf) {
            for (int i = 0; i < leaf->n; i++) {
                if (leaf->keys[i] >= l && leaf->keys[i] <= r)
                    result.push_back(leaf->values[i]);
                else if (leaf->keys[i] > r)
                    return result;
            }
            leaf = leaf->next;
        }
        return result;
    }

    // 搜索
    int* search(int key) {
        BPlusNode* leaf = findLeaf(key);
        for (int i = 0; i < leaf->n; i++) {
            if (leaf->keys[i] == key)
                return &leaf->values[i];
        }
        return nullptr;
    }
};
```

---

## 总结

B+ 树通过以下设计实现了**大规模数据存储**的高效支持：

1. **高扇出**：减少树高，降低 I/O 开销
2. **叶子链表**：支持高效范围查询
3. **50% 最小占用率**：保证空间利用率
4. **完全平衡**：查询时间可预测
5. **批量加载**：快速构建大规模索引

这些特性使 B+ 树成为**数据库索引**和**文件系统**的事实标准。

---

## 参考资料

[^1]: Bayer, R., & McCreight, E. (1972). Organization and maintenance of large ordered indices. Acta Informatica, 1(3), 173-189.

[^2]: Database System Concepts - B+ Tree Index. https://www.db-book.com/

[^3]: PostgreSQL Documentation - Indexes. https://www.postgresql.org/docs/

[^4]: MySQL Documentation - InnoDB Index. https://dev.mysql.com/doc/refman/
