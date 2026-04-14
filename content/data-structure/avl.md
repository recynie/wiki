---
title: AVL树
date: 2026-04-06
description: AVL树——最早提出的自平衡二叉搜索树，保证查找、插入、删除操作的时间复杂度为 $O(\log n)$
tags:
  - data-structure
  - balanced-tree
  - bst
draft: false
permalink:
---

## 定义

AVL 树是由 G. M. Adelson-Velsky 和 E. M. Landis 于 1962 年提出的自平衡二叉搜索树。[^1]

其核心性质：**对于任意节点，其左右子树的高度差（平衡因子）不超过 1**。

当插入或删除节点导致平衡性被破坏时，通过 **旋转** 操作在 $O(1)$ 时间内恢复平衡。

---

## 性质

| 性质 | 说明 |
|------|------|
| BST 性质 | 左子树所有节点小于根，右子树所有节点大于根 |
| 平衡条件 | 任意节点的平衡因子 $\in \{-1, 0, 1\}$ |
| 高度 | $h = O(\log n)$，因此各操作时间为 $O(\log n)$ |

### 平衡因子

$$
\text{BF}(x) = \text{height}(\text{left}(x)) - \text{height}(\text{right}(x))
$$

---

## 旋转操作

当某个节点的平衡因子绝对值大于 1 时，需要通过旋转来恢复平衡。

### 单旋（两种情况）

#### 左左情况（LL）

```
     z                      y
    / \                    / \
   y   T4      -->        x   z
  / \                      /  / \
 x   T3                   T1 T2  T3
/ \
T1 T2
```

#### 右右情况（RR）

```
   z                      y
  / \                    / \
 T1  y       -->        z   x
    / \                / \  / \
   T2  x               T1 T2 T3 T4
      / \
     T3  T4
```

### 双旋（两种情况）

#### 左右情况（LR）

```
   z                      z                      x
  / \                    / \                    / \
 y   T4      -->        x   T4      -->        y   z
/ \                      / \                  / \ / \
T1  x                    y  T3               T1 T2 T3 T4
   / \                  / \
  T2 T3                T1 T2
```

先对 $y$ 做 RR 旋转，再对 $z$ 做 LL 旋转。

#### 右左情况（RL）

```
 z                       z                      x
/ \                     / \                    / \
T1  y       -->       T1  x        -->        z   y
   / \                    / \              / \ / \
  x  T4                   T2  y           T1 T2 T3 T4
  / \                        / \
 T2 T3                      T3 T4
```

先对 $y$ 做 LL 旋转，再对 $z$ 做 RR 旋转。

---

## C++ 实现

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Node {
    int key;           // 键值
    int height;        // 节点高度
    Node *left;        // 左子节点
    Node *right;       // 右子节点
    Node(int k) : key(k), height(1), left(nullptr), right(nullptr) {}
};

class AVLTree {
private:
    Node* root;

    // 获取节点高度
    int height(Node* node) {
        return node ? node->height : 0;
    }

    // 获取平衡因子
    int balanceFactor(Node* node) {
        return node ? height(node->left) - height(node->right) : 0;
    }

    // 更新节点高度
    void updateHeight(Node* node) {
        if (node) {
            node->height = 1 + max(height(node->left), height(node->right));
        }
    }

    // 左左旋转（LL）
    Node* rotateLL(Node* z) {
        Node* y = z->left;
        z->left = y->right;
        y->right = z;
        updateHeight(z);
        updateHeight(y);
        return y;
    }

    // 右右旋转（RR）
    Node* rotateRR(Node* z) {
        Node* y = z->right;
        z->right = y->left;
        y->left = z;
        updateHeight(z);
        updateHeight(y);
        return y;
    }

    // 左右旋转（LR）
    Node* rotateLR(Node* z) {
        z->left = rotateRR(z->left);
        return rotateLL(z);
    }

    // 右左旋转（RL）
    Node* rotateRL(Node* z) {
        z->right = rotateLL(z->right);
        return rotateRR(z);
    }

    // 重新平衡
    Node* rebalance(Node* node) {
        updateHeight(node);
        int bf = balanceFactor(node);

        // 左子树更高
        if (bf > 1) {
            if (balanceFactor(node->left) >= 0) {
                // 左左情况
                return rotateLL(node);
            } else {
                // 左右情况
                return rotateLR(node);
            }
        }
        // 右子树更高
        if (bf < -1) {
            if (balanceFactor(node->right) <= 0) {
                // 右右情况
                return rotateRR(node);
            } else {
                // 右左情况
                return rotateRL(node);
            }
        }
        return node;
    }

    // 插入
    Node* insert(Node* node, int key) {
        if (!node) return new Node(key);

        if (key < node->key) {
            node->left = insert(node->left, key);
        } else if (key > node->key) {
            node->right = insert(node->right, key);
        } else {
            // 重复键值，不插入
            return node;
        }

        return rebalance(node);
    }

    // 查找最小节点
    Node* minNode(Node* node) {
        while (node->left) node = node->left;
        return node;
    }

    // 删除
    Node* erase(Node* node, int key) {
        if (!node) return nullptr;

        if (key < node->key) {
            node->left = erase(node->left, key);
        } else if (key > node->key) {
            node->right = erase(node->right, key);
        } else {
            // 找到要删除的节点
            if (!node->left || !node->right) {
                Node* temp = node->left ? node->left : node->right;
                delete node;
                return temp;
            } else {
                // 左右子树都存在，用后继替换
                Node* successor = minNode(node->right);
                node->key = successor->key;
                node->right = erase(node->right, successor->key);
            }
        }

        return rebalance(node);
    }

    // 中序遍历（用于调试）
    void inorder(Node* node) {
        if (!node) return;
        inorder(node->left);
        cout << node->key << " ";
        inorder(node->right);
    }

public:
    AVLTree() : root(nullptr) {}

    void insert(int key) {
        root = insert(root, key);
    }

    void erase(int key) {
        root = erase(root, key);
    }

    bool find(int key) {
        Node* cur = root;
        while (cur) {
            if (key < cur->key) cur = cur->left;
            else if (key > cur->key) cur = cur->right;
            else return true;
        }
        return false;
    }

    void inorder() {
        inorder(root);
        cout << endl;
    }
};
```

### 使用示例

```cpp
int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    AVLTree avl;
    vector<int> keys = {10, 20, 30, 40, 50, 25};

    for (int key : keys) {
        avl.insert(key);
        cout << "插入 " << key << " 后的中序遍历: ";
        avl.inorder();
    }

    avl.erase(30);
    cout << "删除 30 后的中序遍历: ";
    avl.inorder();

    cout << "查找 25: " << (avl.find(25) ? "找到" : "未找到") << endl;
    cout << "查找 100: " << (avl.find(100) ? "找到" : "未找到") << endl;

    return 0;
}
```

---

## 时间复杂度

| 操作 | 平均时间复杂度 | 最坏时间复杂度 |
|------|---------------|---------------|
| 查找 | $O(\log n)$ | $O(\log n)$ |
| 插入 | $O(\log n)$ | $O(\log n)$ |
| 删除 | $O(\log n)$ | $O(\log n)$ |

---

## 与其他平衡树的比较

| 数据结构 | 插入/删除 | 查找 | 实现难度 | 备注 |
|----------|----------|------|---------|------|
| AVL 树 | $O(\log n)$ | $O(\log n)$ | 中等 | 高度平衡，查找性能好 |
| 红黑树 | $O(\log n)$ | $O(\log n)$ | 中等 | 高度大致平衡，插入删除开销小 |
| Treap | $O(\log n)$ | $O(\log n)$ | 简单 | 随机化，代码简洁 |
| Splay 树 | $O(\log n)$（均摊） | $O(\log n)$（均摊） | 简单 | 访问局部性好 |

AVL 树比红黑树更严格，因此查找性能更好；但插入删除可能需要更多旋转。

---

## 参考资料

[^1]: Adelson-Velsky, G., & Landis, E. (1962). An algorithm for the organization of information. Soviet Mathematics Doklady, 3, 1259-1263.

[^2]: AVL Tree - OI Wiki. https://oi-wiki.org/ds/avl/

[^3]: Introduction to Algorithms (CLRS) - Chapter 13: Balanced Search Trees
