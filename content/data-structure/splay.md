---
title: Splay树
date: 2026-04-05
description: Splay树（伸展树）详解：基于伸展操作的自平衡二叉搜索树，支持旋转、伸展、插入、删除、查找等操作
tags:
  - splay
  - balanced-tree
  - data-structure
draft: false
permalink:
---

## 定义

**Splay树**（伸展树）是一种基于伸展操作的自平衡二叉搜索树，由 Daniel Sleator 和 Robert Tarjan 于1985年发明。[^1]

其核心思想是：每次访问（查找、插入、删除）某个节点后，将该节点通过一系列**伸展操作**旋转到根节点的位置，从而保持树的平衡。伸展操作能保证均摊 $O(\log n)$ 的时间复杂度。

## 基本结构

Splay树是一棵二叉查找树，满足左子树所有节点值 < 根节点值 < 右子树所有节点值的性质。

### 节点信息

| 属性 | 含义 |
|------|------|
| `fa[i]` | 节点 $i$ 的父亲 |
| `ch[i][0/1]` | 节点 $i$ 的左/右儿子（0为左，1为右） |
| `val[i]` | 节点权值 |
| `cnt[i]` | 权值出现次数 |
| `sz[i]` | 子树大小 |

### 辅助操作

```cpp
// 判断节点x是其父亲的左儿子还是右儿子
bool dir(int x) { return x == ch[fa[x]][1]; }

// 更新节点信息
void push_up(int x) { sz[x] = cnt[x] + sz[ch[x][0]] + sz[ch[x][1]]; }
```

## 旋转操作

旋转的作用是将某个节点上移一个位置，同时保持二叉查找树的性质。

```cpp
void rotate(int x) {
    int y = fa[x], z = fa[y];
    bool r = dir(x);
    ch[y][r] = ch[x][!r];
    if (ch[y][r]) fa[ch[y][r]] = y;
    ch[x][!r] = y;
    if (z) ch[z][dir(y)] = x;
    fa[y] = x;
    fa[x] = z;
    push_up(y);
    push_up(x);
}
```

## 伸展操作

伸展操作是Splay树的核心，每次访问节点后将节点旋转到根。

### 三种伸展步骤

1. **zig**：当 $p$ 是根节点时，直接旋转 $x$
2. **zig-zig**：当 $x$ 和 $p$ 都是左/右子节点时，先旋转 $p$ 再旋转 $x$
3. **zig-zag**：当 $x$ 和 $p$一个是左子节点一个是右子节点时，旋转两次 $x$

```cpp
void splay(int& rt, int x) {
    for (int y = fa[x]; (y = fa[x]) != 0; rotate(x)) {
        if (fa[y] != 0) rotate(dir(x) == dir(y) ? y : x);
    }
    rt = x;
}
```

### 时间复杂度

对大小为 $n$ 的Splay树进行 $m$ 次伸展操作，复杂度为 $O((n+m)\log n)$，单次均摊为 $O(\log n)$。

## 平衡树操作

### 插入

```cpp
void insert(int& rt, int x, int f = 0) {
    if (!rt) {
        rt = ++tot;
        val[rt] = x;
        cnt[rt] = sz[rt] = 1;
        fa[rt] = f;
        ch[rt][0] = ch[rt][1] = 0;
        return;
    }
    if (x == val[rt]) {
        cnt[rt]++;
        sz[rt]++;
        return;
    }
    if (x < val[rt]) insert(ch[rt][0], x, rt);
    else insert(ch[rt][1], x, rt);
    push_up(rt);
}
```

### 按值查找

```cpp
int find(int rt, int x) {
    if (val[rt] == x) return rt;
    if (x < val[rt]) return find(ch[rt][0], x);
    else return find(ch[rt][1], x);
}
```

### 查询排名

```cpp
int kth(int rt, int k) {
    if (k <= sz[ch[rt][0]]) return kth(ch[rt][0], k);
    if (k <= sz[ch[rt][0]] + cnt[rt]) return rt;
    return kth(ch[rt][1], k - sz[ch[rt][0]] - cnt[rt]);
}
```

### 查询前驱/后继

```cpp
int lower(int rt, int x) {
    if (!rt) return 0;
    if (val[rt] < x) {
        int res = lower(ch[rt][1], x);
        return res ? res : rt;
    }
    return lower(ch[rt][0], x);
}

int upper(int rt, int x) {
    if (!rt) return 0;
    if (val[rt] > x) {
        int res = upper(ch[rt][0], x);
        return res ? res : rt;
    }
    return upper(ch[rt][1], x);
}
```

## 序列操作（区间翻转）

Splay树可以用于序列操作，如区间翻转。通过将区间端点旋转到根，然后对子树打懒标记。

### 建树

```cpp
int build(int l, int r, int f) {
    if (l > r) return 0;
    int mid = (l + r) >> 1;
    int cur = ++tot;
    val[cur] = a[mid];
    fa[cur] = f;
    ch[cur][0] = build(l, mid - 1, cur);
    ch[cur][1] = build(mid + 1, r, cur);
    push_up(cur);
    return cur;
}
```

### 区间翻转

```cpp
void push_down(int x) {
    if (rev[x]) {
        swap(ch[x][0], ch[x][1]);
        if (ch[x][0]) rev[ch[x][0]] ^= 1;
        if (ch[x][1]) rev[ch[x][1]] ^= 1;
        rev[x] = 0;
    }
}

void reverse(int l, int r) {
    int tl = kth(rt, l - 1);
    int tr = kth(rt, r + 1);
    splay(tl, 0);
    splay(ch[tl][1], tl);
    int target = ch[ch[tl][1]][0];
    rev[target] ^= 1;
}
```

## 与Treap的比较

| 特性 | Splay | Treap |
|------|-------|-------|
| 平衡方式 | 伸展操作 | 随机优先级 |
| 实现复杂度 | 较高 | 较低 |
| 单次操作 | 最坏 $O(n)$，均摊 $O(\log n)$ | $O(\log n)$ |
| 区间操作 | 天然支持 | 需隐式Treap |

## 应用场景

- 高效的键值对存储与查询
- 序列区间操作（插入、删除、反转）
- LRU Cache 实现
- 伸展操作使最近访问节点靠近根，适合局部性场景

## 参考资料

[^1]: [Splay树 - OI Wiki](https://oi-wiki.org/ds/splay/)
