---
title: 红黑树
date: 2026-04-07
description: 红黑树（Red-Black Tree）是一种自平衡二叉查找树，通过颜色标记和旋转操作保持近似平衡。
tags:
  - data-structure
  - balanced-tree
  - red-black-tree
draft: false
permalink:
---

## 定义与性质

红黑树是一种自平衡的[[../data-structure/avl|AVL树]]的近似变体，通过颜色标记和旋转操作保持近似平衡。[^1]

**五大性质**：

1. 每个节点非红即黑
2. 根节点是黑色
3. 每个叶子节点（NIL）是黑色
4. 如果一个节点是红色，则它的两个子节点都是黑色（即不存在两个连续的红色节点）
5. 对每个节点，从该节点到其所有后代叶子节点的路径上，黑色节点数量相同

**黑高（Black Height）**：从某节点出发（不包括该节点）到任意叶子的路径上，黑色节点的个数。[^2]

**复杂度**：

| 操作 | 时间复杂度 |
|------|------------|
| 查找 | O(log n) |
| 插入 | O(log n) |
| 删除 | O(log n) |

## 与AVL树的比较

| 操作 | AVL | 红黑树 |
|------|-----|--------|
| 查找 | O(log n) | O(log n) |
| 插入 | O(log n) | O(log n) |
| 删除 | O(log n) | O(log n) |
| 旋转 | 最多2次 | 最多2次 |
| 平衡度 | 高度平衡 | 弱平衡 |

AVL树适合查找多、修改少的场景；红黑树适合综合场景（C++ STL的map/set）。[^3]

## 插入操作

1. 先进行BST插入，将新节点设为红色
2. 修复红黑性质（可能违反性质2、4）：
   - **Case 1**：新节点是根节点 → 染成黑色
   - **Case 2**：父节点是黑色 → 无需修复
   - **Case 3**：父节点和叔节点都是红色 → 染黑父节点和叔节点，染红祖父节点，递归向上处理
   - **Case 4**：父红叔黑（或者NIL视为黑）+ LL/RR型 → 对父节点单旋转，交换父节点和祖父节点的颜色
   - **Case 5**：父红叔黑 + LR/RL型 → 对新节点双旋转

### 插入代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

enum Color { RED, BLACK };

struct Node {
    int key;
    Color color;
    Node *left, *right, *parent;
    Node(int k) : key(k), color(RED), left(nullptr), right(nullptr), parent(nullptr) {}
};

class RedBlackTree {
private:
    Node *root;
    
    void leftRotate(Node *x) {
        Node *y = x->right;
        x->right = y->left;
        if (y->left) y->left->parent = x;
        y->parent = x->parent;
        if (!x->parent) root = y;
        else if (x == x->parent->left) x->parent->left = y;
        else x->parent->right = y;
        y->left = x;
        x->parent = y;
    }
    
    void rightRotate(Node *x) {
        Node *y = x->left;
        x->left = y->right;
        if (y->right) y->right->parent = x;
        y->parent = x->parent;
        if (!x->parent) root = y;
        else if (x == x->parent->left) x->parent->left = y;
        else x->parent->right = y;
        y->right = x;
        x->parent = y;
    }
    
    void insertFixup(Node *z) {
        while (z->parent && z->parent->color == RED) {
            if (z->parent == z->parent->parent->left) {
                Node *y = z->parent->parent->right;
                // Case 3: 父节点和叔节点都是红色
                if (y && y->color == RED) {
                    z->parent->color = BLACK;
                    y->color = BLACK;
                    z->parent->parent->color = RED;
                    z = z->parent->parent;
                } else {
                    // Case 4 or 5: LR或LL型
                    if (z == z->parent->right) {
                        z = z->parent;
                        leftRotate(z);
                    }
                    z->parent->color = BLACK;
                    z->parent->parent->color = RED;
                    rightRotate(z->parent->parent);
                }
            } else {
                Node *y = z->parent->parent->left;
                if (y && y->color == RED) {
                    z->parent->color = BLACK;
                    y->color = BLACK;
                    z->parent->parent->color = RED;
                    z = z->parent->parent;
                } else {
                    if (z == z->parent->left) {
                        z = z->parent;
                        rightRotate(z);
                    }
                    z->parent->color = BLACK;
                    z->parent->parent->color = RED;
                    leftRotate(z->parent->parent);
                }
            }
        }
        root->color = BLACK;
    }
    
public:
    RedBlackTree() : root(nullptr) {}
    
    void insert(int key) {
        Node *z = new Node(key);
        Node *y = nullptr;
        Node *x = root;
        
        while (x) {
            y = x;
            if (z->key < x->key) x = x->left;
            else x = x->right;
        }
        z->parent = y;
        
        if (!y) root = z;
        else if (z->key < y->key) y->left = z;
        else y->right = z;
        
        insertFixup(z);
    }
    
    Node* search(int key) {
        Node *x = root;
        while (x) {
            if (key == x->key) return x;
            else if (key < x->key) x = x->left;
            else x = x->right;
        }
        return nullptr;
    }
};
```

## 删除操作

删除操作更复杂，需要考虑：

- 被删除节点是红色 → 直接删除，不影响黑高
- 被删除节点是黑色 → 需要修复黑高

### 删除修复的主要情况

当删除的节点是黑色且具有红色子节点时，需要用红色子节点替换并染黑。当删除的节点是黑色且子节点都是NIL时，需要通过旋转和重新着色来修复。

### 删除代码实现

```cpp
void deleteNode(Node *z) {
    Node *y = z;
    Node *x;
    Color y_original_color = y->color;
    
    if (!z->left) {
        x = z->right;
        transplant(z, z->right);
    } else if (!z->right) {
        x = z->left;
        transplant(z, z->left);
    } else {
        y = minimum(z->right);
        y_original_color = y->color;
        x = y->right;
        if (y->parent == z) {
            if (x) x->parent = y;
        } else {
            transplant(y, y->right);
            y->right = z->right;
            if (y->right) y->right->parent = y;
        }
        transplant(z, y);
        y->left = z->left;
        if (y->left) y->left->parent = y;
        y->color = z->color;
    }
    
    if (y_original_color == BLACK && x) {
        deleteFixup(x);
    }
}

void deleteFixup(Node *x) {
    while (x != root && (!x || x->color == BLACK)) {
        if (x == x->parent->left) {
            Node *w = x->parent->right;
            if (w && w->color == RED) {
                w->color = BLACK;
                x->parent->color = RED;
                leftRotate(x->parent);
                w = x->parent->right;
            }
            if ((!w->left || w->left->color == BLACK) &&
                (!w->right || w->right->color == BLACK)) {
                w->color = RED;
                x = x->parent;
            } else {
                if (!w->right || w->right->color == BLACK) {
                    if (w->left) w->left->color = BLACK;
                    w->color = RED;
                    rightRotate(w);
                    w = x->parent->right;
                }
                w->color = x->parent->color;
                x->parent->color = BLACK;
                if (w->right) w->right->color = BLACK;
                leftRotate(x->parent);
                x = root;
            }
        } else {
            Node *w = x->parent->left;
            if (w && w->color == RED) {
                w->color = BLACK;
                x->parent->color = RED;
                rightRotate(x->parent);
                w = x->parent->left;
            }
            if ((!w->left || w->left->color == BLACK) &&
                (!w->right || w->right->color == BLACK)) {
                w->color = RED;
                x = x->parent;
            } else {
                if (!w->left || w->left->color == BLACK) {
                    if (w->right) w->right->color = BLACK;
                    w->color = RED;
                    leftRotate(w);
                    w = x->parent->left;
                }
                w->color = x->parent->color;
                x->parent->color = BLACK;
                if (w->left) w->left->color = BLACK;
                rightRotate(x->parent);
                x = root;
            }
        }
    }
    if (x) x->color = BLACK;
}
```

## 模板题

- 洛谷 P3369 【模板】普通平衡树（可用红黑树实现）

## 相关主题

- [[../data-structure/avl|AVL树]] — 另一种自平衡BST
- [[../data-structure/treap|Treap]] — 随机化平衡树
- [[../data-structure/splay|Splay树]] — 伸展树

[^1]: 红黑树通过对任何一条从根到叶子的路径上各个节点着色方式的限制，确保没有一条路径会比其他路径长出两倍，因而是接近平衡的。
[^2]: 黑高是红黑树保持平衡的关键概念，用于证明红黑树的高度上界。
[^3]: C++标准模板库中的map和set底层实现通常采用红黑树，以在插入、删除操作上获得较好的平均性能。
