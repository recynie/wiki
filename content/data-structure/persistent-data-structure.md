---
title: 可持久化数据结构
date: 2026-04-08
description: 可持久化数据结构（Persistent Data Structures）详解：线段树、平衡树、Trie的持久化
tags:
  - persistent-data-structure
  - segment-tree
  - trie
  - data-structure
draft: false
permalink:
---

## 概述

可持久化数据结构（Persistent Data Structure）是指支持查询历史版本的数据结构。对于每次修改操作，都会创建一个新的版本，同时保留历史版本供查询。[^1]

### 核心思想

**空间换时间**：通过共享未修改的部分来压缩空间。

```
原版本: [1, 2, 3, 4, 5]
修改后: [1, 2, 10, 4, 5]
         ↓   ↓   ↑
        共享 共享 新节点
```

### 实现方式

| 方式 | 原理 | 适用场景 |
|------|------|---------|
| **路径复制** | 只复制从根到修改节点的路径 | 完全可持久化 |
| **部分复制** | 只复制需要修改的部分 | 牺牲部分灵活性节省空间 |

---

## 可持久化线段树

### 概念

可持久化线段树（Persistent Segment Tree）是在线段树的基础上，支持保存历史版本。每次更新时，只创建 $O(\log n)$ 个新节点，其他节点被历史版本共享。[^2]

### 节点结构

```cpp
struct Node {
    int l = 0, r = 0;      // 左右子节点指针/索引
    int sum = 0;           // 区间和（或其他聚合值）
    
    Node() {}
    Node(int sum) : sum(sum) {}
    Node(int l, int r, int sum) : l(l), r(r), sum(sum) {}
};

vector<Node> seg;          // 节点池
int root[N];               // 各版本的根节点索引
```

### 建树

```cpp
int build(int l, int r) {
    int p = seg.size();
    seg.emplace_back(0);
    if (l == r) {
        seg[p].sum = a[l];
    } else {
        int m = (l + r) >> 1;
        seg[p].l = build(l, m);
        seg[p].r = build(m + 1, r);
        seg[p].sum = seg[seg[p].l].sum + seg[seg[p].r].sum;
    }
    return p;
}
```

### 单点更新

```cpp
int update(int now, int l, int r, int pos, int val) {
    int p = seg.size();
    seg.push_back(seg[now]);               // 复制当前版本
    if (l == r) {
        seg[p].sum = val;                    // 更新值
    } else {
        int m = (l + r) >> 1;
        if (pos <= m) {
            seg[p].l = update(seg[p].l, l, m, pos, val);
        } else {
            seg[p].r = update(seg[p].r, m + 1, r, pos, val);
        }
        seg[p].sum = seg[seg[p].l].sum + seg[seg[p].r].sum;
    }
    return p;
}
```

### 版本查询

```cpp
int query(int p, int l, int r, int pos) {
    if (l == r) return seg[p].sum;
    int m = (l + r) >> 1;
    if (pos <= m) return query(seg[p].l, l, m, pos);
    else return query(seg[p].r, m + 1, r, pos);
}
```

### 区间查询

```cpp
int query(int p, int l, int r, int ql, int qr) {
    if (ql <= l && r <= qr) return seg[p].sum;
    int m = (l + r) >> 1;
    int ans = 0;
    if (ql <= m) ans += query(seg[p].l, l, m, ql, qr);
    if (qr > m) ans += query(seg[p].r, m + 1, r, ql, qr);
    return ans;
}
```

---

## 可持久化Trie

### 概念

可持久化Trie（Persistent Trie）用于高效存储和查询字符串集合的多个历史版本。每次插入新字符串时，只创建 $O(|S|)$ 个新节点。

### 节点结构

```cpp
struct TrieNode {
    int next[26];          // 26个字母的子节点索引，0表示不存在
    int cnt = 0;           // 以该节点为前缀的字符串数量
    
    TrieNode() {
        fill(begin(next), end(next), 0);
    }
};

vector<TrieNode> trie;
```

### 插入操作

```cpp
int insert(int prev, const string& s) {
    int p = trie.size();
    trie.emplace_back(trie[prev]);           // 复制前一版本
    
    int cur = p;
    for (char c : s) {
        int idx = c - 'a';
        int child = trie.size();
        trie.emplace_back();
        
        // 复制旧节点的子树
        if (trie[cur].next[idx]) {
            trie[child] = trie[trie[cur].next[idx]];
        }
        trie[cur].next[idx] = child;
        cur = child;
    }
    trie[cur].cnt++;                          // 字符串计数
    return p;
}
```

### 前缀查询

```cpp
int queryPrefix(int root, const string& pref) {
    int p = root;
    for (char c : pref) {
        int idx = c - 'a';
        if (!trie[p].next[idx]) return 0;
        p = trie[p].next[idx];
    }
    return trie[p].cnt;
}
```

---

## 可持久化平衡树（可持久化Treap）

### 概念

可持久化Treap通过路径复制实现，每次插入/删除只影响从根到操作节点的路径。

### 节点结构

```cpp
struct TreapNode {
    int key, priority;
    int l = 0, r = 0;
    int size = 1;
    
    TreapNode(int key) : key(key), priority(rand()) {}
};
```

### 分割操作

```cpp
void split(int p, int key, int& l, int& r) {
    if (!p) {
        l = r = 0;
        return;
    }
    if (treap[p].key <= key) {
        split(treap[p].r, key, treap[p].r, r);
        l = p;
    } else {
        split(treap[p].l, key, l, treap[p].l);
        r = p;
    }
}
```

### 合并操作

```cpp
int merge(int l, int r) {
    if (!l || !r) return l ? l : r;
    if (treap[l].priority > treap[r].priority) {
        treap[l].r = merge(treap[l].r, r);
        return l;
    } else {
        treap[r].l = merge(l, treap[r].l);
        return r;
    }
}
```

### 插入操作

```cpp
int insert(int prev, int key) {
    int l, r;
    split(prev, key, l, r);
    return merge(merge(l, newNode(key)), r);
}
```

### 删除操作

```cpp
int erase(int prev, int key) {
    int l, mid, r;
    split(prev, key - 1, l, mid);
    split(mid, key, mid, r);
    return merge(l, r);                       // 跳过mid节点
}
```

---

## 经典应用

### 1. 区间第k小数（可持久化线段树）

每次在原数组位置插入一个值，建立可持久化权值线段树，查询历史区间第k小。

```cpp
// 查询第r版本中[l,r]区间的第k小数
int queryKth(int lRoot, int rRoot, int l, int r, int k) {
    if (l == r) return l;
    int leftCnt = seg[seg[rRoot].l].sum - seg[seg[lRoot].l].sum;
    int m = (l + r) >> 1;
    if (k <= leftCnt) {
        return queryKth(seg[lRoot].l, seg[rRoot].l, l, m, k);
    } else {
        return queryKth(seg[lRoot].r, seg[rRoot].r, m + 1, r, k - leftCnt);
    }
}
```

### 2. 字符串版本管理（可持久化Trie）

如版本控制系统中的字符串管理，查询历史版本中字符串出现次数。

### 3. 函数式编程

在函数式编程语言中，可持久化数据结构是实现不可变数据的基础。

---

## 复杂度分析

| 操作 | 时间复杂度 | 空间复杂度 |
|------|-----------|-----------|
| 可持久化线段树 | $O(\log n)$ | $O(n \log n)$ |
| 可持久化Trie | $O(|S|)$ | $O(total\_len)$ |
| 可持久化Treap | $O(\log n)$ | $O(n \log n)$ |

---

## 参考资料

[^1]: 本词条参考[可持久化数据结构 - OI Wiki](https://oi-wiki.org/ds/persistent/)

[^2]: 可持久化线段树（主席树）广泛应用于区间查询优化

---

## 相关主题

- [[segment-tree|线段树]] - 可持久化的基础
- [[fenwick|树状数组]] - 另一种区间查询结构
- [[trie|Trie]] - 字符串处理的利器
