---
title: Link-Cut Tree
date: 2026-04-06
description: Link-Cut Tree（LCT）——动态树操作的利器，支持路径查询、修改和树的动态连接
tags:
  - data-structure
  - dynamic-tree
  - tree
draft: false
permalink:
---

## 定义

Link-Cut Tree（简称 LCT）是一种用于处理动态树问题的数据结构，由 Robert Tarjan 和 Daniel Sleator 于 1980 年代提出。[^1]

动态树问题是指在一个不断变化的森林中，支持以下操作：
- **link(u, v)**：将节点 $u$ 连接到节点 $v$（需保证 $u$ 是所在树的根）
- **cut(u)**：删除节点 $u$ 与其父节点的边
- **query(u, v)**：查询从 $u$ 到 $v$ 的路径上的信息

LCT 能在 $O(\log n)$ 均摊时间内完成上述操作。

---

## 核心思想

###  Preferred Path（首选路径）

LCT 的核心思想是将树分解为若干条 **splay 树** 表示的路径（称为 preferred path）。每个节点通过 `preferred child` 边连接到其子树中的某个子节点，使得从每个节点沿 `preferred child` 向下走，形成的路径连接了从根到该节点的路径。

### Access 操作

`access(v)` 是 LCT 的基础操作，它将根到 $v$ 的路径变为一条 preferred path。具体过程：

1. 将 $v$ 与其父节点的边变为非 preferred
2. 沿 $v$ 向上，将经过的节点依次 splay 到根

### 虚边与实边

- **实边（solid edge）**：父节点与子节点之间的边，被该父节点的 splay 树管理
- **虚边（dashed edge）**：不在同一 splay 树中的边，连接不同的辅助树

---

## 数据结构

每个节点存储：
- `ch[2]`：左右子节点指针
- `fa`：父指针（指向 splay 树中的父节点）
- `val`：节点权值
- `sum`：该节点为根的子树（对应 splay 中的中序遍历）的权值和
- `rev`：懒标记，表示是否需要翻转子树

### 辅助函数

```cpp
bool isRoot(int x) {
    int f = tree[x].fa;
    if (!f) return true;
    return tree[f].ch[0] != x && tree[f].ch[1] != x;
}
```

`isRoot(x)` 判断 $x$ 是否是其所在 splay 树的根节点。

---

## C++ 实现

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Node {
    int ch[2];      // 左右子节点
    int fa;         // 父节点
    int val;        // 权值
    int sum;        // 子树和
    bool rev;       // 翻转标记
    Node() : fa(0), val(0), sum(0), rev(false) {
        ch[0] = ch[1] = 0;
    }
};

class LinkCutTree {
private:
    vector<Node> tree;
    int n;

    // 上推：更新节点信息
    void pushUp(int x) {
        tree[x].sum = tree[x].val;
        if (tree[x].ch[0]) {
            tree[x].sum ^= tree[tree[x].ch[0]].sum;
        }
        if (tree[x].ch[1]) {
            tree[x].sum ^= tree[tree[x].ch[1]].sum;
        }
    }

    // 下推：传递翻转标记
    void pushDown(int x) {
        if (tree[x].rev) {
            swap(tree[x].ch[0], tree[x].ch[1]);
            if (tree[x].ch[0]) tree[tree[x].ch[0]].rev ^= 1;
            if (tree[x].ch[1]) tree[tree[x].ch[1]].rev ^= 1;
            tree[x].rev = false;
        }
    }

    // 判断节点方向
    int direction(int x) {
        int f = tree[x].fa;
        if (!f) return -1;
        if (tree[f].ch[0] == x) return 0;
        if (tree[f].ch[1] == x) return 1;
        return -1;
    }

    // 旋转
    void rotate(int x) {
        int y = tree[x].fa;
        int z = tree[y].fa;
        int dx = direction(x);
        int dy = direction(y);

        pushDown(y);
        pushDown(x);

        if (!isRoot(y)) {
            tree[z].ch[dy] = x;
        }
        tree[x].fa = z;

        int b = tree[x].ch[dx ^ 1];
        if (b) tree[b].fa = y;
        tree[y].ch[dx] = b;

        tree[x].ch[dx ^ 1] = y;
        tree[y].fa = x;

        pushUp(y);
        pushUp(x);
    }

    // 从上到下下推所有标记
    void pushAll(int x) {
        if (!isRoot(x)) pushAll(tree[x].fa);
        pushDown(x);
    }

    // Splay：将节点 x 旋转到其所在 splay 树的根
    void splay(int x) {
        pushAll(x);
        while (!isRoot(x)) {
            int y = tree[x].fa;
            int z = tree[y].fa;
            if (!isRoot(y)) {
                if ((tree[y].ch[0] == x) ^ (tree[z].ch[0] == y))
                    rotate(x);
                else
                    rotate(y);
            }
            rotate(x);
        }
        pushUp(x);
    }

    // 判断是否为根（在辅助森林中）
    bool isRoot(int x) {
        int f = tree[x].fa;
        if (!f) return true;
        return tree[f].ch[0] != x && tree[f].ch[1] != x;
    }

public:
    LinkCutTree(int n_) : n(n_) {
        tree.resize(n + 1);
    }

    // 初始化节点权值
    void init(int x, int val) {
        tree[x].val = val;
        tree[x].sum = val;
    }

    // Access：将根到 x 的路径变为一条 preferred path
    void access(int x) {
        int last = 0;
        while (x) {
            splay(x);
            tree[x].ch[1] = last;
            pushUp(x);
            last = x;
            x = tree[x].fa;
        }
    }

    // makeRoot：将 x 变为所在树的根
    void makeRoot(int x) {
        access(x);
        splay(x);
        tree[x].rev ^= 1;
    }

    // findRoot：找到 x 所在树的根节点
    int findRoot(int x) {
        access(x);
        splay(x);
        pushDown(x);
        while (tree[x].ch[0]) {
            x = tree[x].ch[0];
            pushDown(x);
        }
        splay(x);
        return x;
    }

    // split：将 x 到 y 的路径提取为一条 splay 树
    void split(int x, int y) {
        makeRoot(x);
        access(y);
        splay(y);
    }

    // link：连接 x 和 y（x 必须是根）
    void link(int x, int y) {
        makeRoot(x);
        if (findRoot(y) != x) {
            tree[x].fa = y;
        }
    }

    // cut：断开 x 与其父节点的边
    void cut(int x, int y) {
        makeRoot(x);
        access(y);
        splay(y);
        pushDown(x);
        if (tree[y].ch[0] == x && !tree[x].ch[1] && !tree[x].ch[0]) {
            tree[y].ch[0] = 0;
            tree[x].fa = 0;
            pushUp(y);
        }
    }

    // 查询路径权值和
    int query(int x, int y) {
        split(x, y);
        return tree[y].sum;
    }

    // 修改节点权值
    void update(int x, int val) {
        access(x);
        splay(x);
        tree[x].val = val;
        pushUp(x);
    }
};
```

### 使用示例

```cpp
int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    int n, q;
    cin >> n >> q;
    LinkCutTree lct(n);

    for (int i = 1; i <= n; i++) {
        int v;
        cin >> v;
        lct.init(i, v);
    }

    // 建树：连接 n-1 条边
    for (int i = 1; i < n; i++) {
        int u, v;
        cin >> u >> v;
        lct.link(u, v);
    }

    while (q--) {
        string op;
        cin >> op;
        if (op == "link") {
            int u, v;
            cin >> u >> v;
            lct.link(u, v);
        } else if (op == "cut") {
            int u, v;
            cin >> u >> v;
            lct.cut(u, v);
        } else if (op == "query") {
            int u, v;
            cin >> u >> v;
            cout << lct.query(u, v) << endl;
        }
    }

    return 0;
}
```

---

## 时间复杂度

| 操作 | 时间复杂度（均摊） |
|------|------------------|
| access | $O(\log n)$ |
| link | $O(\log n)$ |
| cut | $O(\log n)$ |
| query | $O(\log n)$ |
| makeRoot | $O(\log n)$ |
| findRoot | $O(\log n)$ |

---

## 经典应用

### 1. 动态树连通性

使用 `findRoot` 判断两点是否在同一棵树中。

```cpp
bool connected(int u, int v) {
    return lct.findRoot(u) == lct.findRoot(v);
}
```

### 2. 路径求和/异或

使用 `split` 提取路径，然后查询 `sum`。

### 3. 树上路径最大值

将 `sum` 改为 `max` 即可。

### 4. 动态树的深度

可以通过 `makeRoot` 和 `access` 来维护树的形态变化。

---

## 例题：动态树路径和

#### 问题

给定一个 $n$ 个节点的树，初始有边 $(1,2), (2,3), \dots, (n-1, n)$。支持 $q$ 次操作：
- `link u v`：连接 $u, v$（$u$ 必须是根）
- `cut u v`：删除边 $(u, v)$
- `sum u v`：查询从 $u$ 到 $v$ 路径上所有节点的权值和

#### 分析

裸的动态树问题，直接使用 LCT 即可。

#### 代码

见上方使用示例。

---

## 参考资料

[^1]: Sleator, D. D., & Tarjan, R. E. (1983). A data structure for dynamic trees. Journal of Computer and System Sciences, 26(3), 362-391.

[^2]: Link-Cut Tree - OI Wiki. https://oi-wiki.org/ds/lct/

[^3]: 动态树入门 - 经典LCT题解. https://www.luogu.com.cn/problem/solution/P3690
