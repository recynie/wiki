---
title: Treap（随机化平衡树）
date: 2026-04-05
description: Treap（树堆）是一种结合二叉搜索树和堆特性的随机化数据结构
tags:
  - treap
  - balanced-tree
  - data-structure
  - randomized
draft: false
permalink:
---

## 定义

Treap（Tree + Heap）是一种巧妙结合二叉搜索树（Binary Search Tree）与堆（Heap）特性的随机化数据结构[^1]。

Treap 存储若干个二元组 $(X, Y)$：

- **BST 性质**：以 $X$（键值）为关键字，满足左子树所有节点 $X$ 不大于根节点， 右子树所有节点 $X$ 大于根节点
- **堆性质**：以 $Y$（优先级）为权值，满足父节点 $Y$ 不小于子节点（最大堆），或父节点 $Y$ 不大于子节点（最小堆）

由于这种结构可以在笛卡尔平面（Cartesian Plane）上直观表示，Treap 又被称为 **Cartesian Tree**（笛卡尔树）。

### 为什么需要随机化？

如果单纯使用 BST，插入已排序序列时会退化为 $O(N)$ 的链表；而使用平衡树（如 AVL、红黑树）则实现复杂。Treap 通过为每个节点分配**随机优先级**，利用随机化手段保证期望 $O(\log N)$ 的树高，从而实现简洁而高效的数据结构。

---

## 核心操作

### Split（分裂）

将一棵 Treap 分裂为两棵：左子树 $L$ 包含所有键值 $\leq key$ 的节点，右子树 $R$ 包含所有键值 $> key$ 的节点。

```
Split(T, key) → (L, R)
```

### Merge（合并）

将两棵满足所有键值关系的 Treap 合并为一棵：$T_1$ 中所有键值小于 $T_2$ 中所有键值。

```
Merge(T1, T2) → T
```

### Insert（插入）

1. 生成随机优先级 $pri$
2. 调用 `Split(root, key)` 得到 $(L, R)$
3. 新建节点 $new = (key, pri)$
4. 调用 `Merge(Merge(L, new), R)`

### Erase（删除）

1. 调用 `Split(root, key)` 得到 $(L, R_1)$，其中 $R_1$ 包含 $> key$ 的节点
2. 对 $R_1$ 调用 `Split(R_1, key)` 得到 $(mid, R)$，其中 $mid$ 包含键值 $= key$ 的节点
3. 调用 `Merge(L, R)` 完成删除

### Build（构建）

对已排序的 $N$ 个键值，可以 $O(N)$ 构建 Treap：维护一个栈，用类似单调栈的方式维护优先级递减的路径。

---

## 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Node {
    int key;       // BST 键值
    int prior;     // 随机优先级
    int l, r;      // 左右孩子
    int cnt;       // 子树节点数（含重复）
    int sz;        // 子树大小
};
using pnode = int;

pnode Merge(pnode a, pnode b) {
    if (!a || !b) return a ? a : b;
    if (tr[a].prior < tr[b].prior) {
        tr[a].r = Merge(tr[a].r, b);
        up(a);
        return a;
    } else {
        tr[b].l = Merge(a, tr[b].l);
        up(b);
        return b;
    }
}

void Split(pnode p, int key, pnode &l, pnode &r) {
    if (!p) {
        l = r = 0;
        return;
    }
    if (tr[p].key <= key) {
        Split(tr[p].r, key, tr[p].r, r);
        l = p;
    } else {
        Split(tr[p].l, key, l, tr[p].l);
        r = p;
    }
    up(p);
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    // 使用示例...
}
```

---

## 隐式 Treap（Implicit Treap）

隐式 Treap 是 Treap 的强大变体[^2]，广泛用于**序列操作**。

### 核心思想

不显式存储键值，而是用**数组索引**作为键值。每个节点的"位置"由其在中序遍历中的排名决定。

### 支持的操作

| 操作 | 说明 | 时间复杂度 |
|------|------|------------|
| 插入到位置 $pos$ | 在序列第 $pos$ 个位置插入 | $O(\log N)$ |
| 删除位置 $pos$ | 删除序列第 $pos$ 个元素 | $O(\log N)$ |
| 区间查询 | 查询 $[l, r]$ 区间的和/最大值等 | $O(\log N)$ |
| 区间更新 | 对 $[l, r]$ 区间执行加法/赋值等 | $O(\log N)$ |
| 区间反转 | 反转 $[l, r]$ 区间 | $O(\log N)$ |

### 隐式 Treap 代码

```cpp
struct Node {
    int val;       // 节点值
    int prior;     // 随机优先级
    int sz;        // 子树大小
    int l, r;      // 左右孩子
    bool rev;      // 反转标记
};
```

---

## 典型应用

### 1. 名次树（K-th Element）

利用 `cnt` 维护子树大小，可在 $O(\log N)$ 内找到第 $k$ 大或第 $k$ 小的元素。

### 2. 序列操作

- **插入/删除**：在任意位置插入或删除元素
- **区间反转**：如 [[segment-tree|线段树]] 的懒标记，隐式 Treap 通过 `rev` 标记实现
- **拖拽**：将序列中某一段移动到其他位置

### 3. 平衡树替代

相比 [[../data-structure/segment-tree|线段树]]，隐式 Treap 更适合需要频繁插入/删除的动态序列问题。

---

## 例题

!!! question "例题：维护序列"

    给定一个长度为 $N$ 的序列，支持以下 $Q$ 次操作：

    1. `Insert pos x`：在位置 $pos$ 后插入数 $x$
    2. `Delete pos`：删除位置 $pos$ 的数
    3. `Query l r`：查询区间 $[l, r]$ 的和

**解法**：使用隐式 Treap，每个节点维护子树大小 `sz` 和区间和 `sum`。

```cpp
struct Node {
    int val, sum;
    int sz, prior;
    int l, r;
    bool rev;
};

void push(pnode p) {
    if (tr[p].rev) {
        swap(tr[p].l, tr[p].r);
        if (tr[p].l) tr[tr[p].l].rev ^= 1;
        if (tr[p].r) tr[tr[p].r].rev ^= 1;
        tr[p].rev = false;
    }
}

int kth(pnode p, int k) {
    // 返回第 k 小的节点
    push(p);
    int left_sz = tr[tr[p].l].sz;
    if (k <= left_sz) return kth(tr[p].l, k);
    else if (k == left_sz + 1) return p;
    else return kth(tr[p].r, k - left_sz - 1);
}
```

时间复杂度：每次操作 $O(\log N)$，总复杂度 $O(Q \log N)$。

---

## 参考资料

[^1]: 本段内容参考 [Treap - CP-Algorithms](https://cp-algorithms.com/data_structures/treap.html)

[^2]: 隐式 Treap 的详细讨论可参考各OI/ACM选手的博客
