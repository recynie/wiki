---
title: 链表
date: 2026-03-31
description: 链表（Linked List）数据结构
tags:
  - linked-list
  - data-structure
draft: false
permalink:
---

## 定义

链表（Linked List）是一种常见的**线性数据结构**，它通过**节点**（Node）之间的指针连接来存储数据。与数组不同，链表中的元素在内存中不必连续存放，每个节点包含数据和指向下一个节点的指针（单链表）或指向上下节点的指针（双链表）。[^1]

链表的基本类型包括：
- **单向链表**（Singly Linked List）：每个节点只包含数据和一个指向下一个节点的指针
- **双向链表**（Doubly Linked List）：每个节点包含数据和两个指针（指向前后节点）
- **循环链表**（Circular Linked List）：最后一个节点的指针指向头节点，形成环

## 核心特性

### 时间复杂度

| 操作 | 头部 | 尾部 | 指定位置 |
|------|------|------|----------|
| 访问（access） | $O(n)$ | $O(n)$ | $O(n)$ |
| 插入（insert） | $O(1)$ | $O(n)$ | $O(n)$ |
| 删除（delete） | $O(1)$ | $O(n)$ | $O(n)$ |

链表的核心优势在于**头部插入和删除的 $O(1)$ 时间复杂度**，这是数组做不到的（数组头部操作需要 $O(n)$ 来移动元素）。

### 空间复杂度

- 空间复杂度：$O(n)$
- 额外空间：相比数组，每个节点需要额外存储指针（双向链表需要两个指针）

### 与数组的对比

| 特性 | 链表 | 数组 |
|------|------|------|
| 存储方式 | 分散（指针连接） | 连续 |
| 访问方式 | 顺序访问 $O(n)$ | 随机访问 $O(1)$ |
| 头部插入/删除 | $O(1)$ | $O(n)$ |
| 尾部插入/删除 | $O(n)$ / $O(1)$（若保留尾指针） | $O(1)$ |
| 内存使用 | 额外指针开销 | 无额外开销 |
| 缓存友好性 | 差 | 好 |

## 代码实现

### 单链表节点结构

```cpp
#include <bits/stdc++.h>
using namespace std;

struct ListNode {
    int val;
    ListNode* next;
    ListNode(int x) : val(x), next(nullptr) {}
};
```

### 单链表基本操作

```cpp
// 在头部插入节点
void pushFront(ListNode** head, int val) {
    ListNode* newNode = new ListNode(val);
    newNode->next = *head;
    *head = newNode;
}

// 在尾部插入节点
void pushBack(ListNode** head, int val) {
    ListNode* newNode = new ListNode(val);
    if (*head == nullptr) {
        *head = newNode;
        return;
    }
    ListNode* cur = *head;
    while (cur->next) {
        cur = cur->next;
    }
    cur->next = newNode;
}

// 在指定位置插入节点
void insertAt(ListNode** head, int pos, int val) {
    if (pos == 0) {
        pushFront(head, val);
        return;
    }
    ListNode* cur = *head;
    for (int i = 0; i < pos - 1 && cur; i++) {
        cur = cur->next;
    }
    if (cur) {
        ListNode* newNode = new ListNode(val);
        newNode->next = cur->next;
        cur->next = newNode;
    }
}

// 删除头部节点
void popFront(ListNode** head) {
    if (*head == nullptr) return;
    ListNode* temp = *head;
    *head = (*head)->next;
    delete temp;
}

// 删除尾部节点
void popBack(ListNode** head) {
    if (*head == nullptr) return;
    if ((*head)->next == nullptr) {
        delete *head;
        *head = nullptr;
        return;
    }
    ListNode* cur = *head;
    while (cur->next && cur->next->next) {
        cur = cur->next;
    }
    delete cur->next;
    cur->next = nullptr;
}

// 删除指定位置的节点
void eraseAt(ListNode** head, int pos) {
    if (*head == nullptr) return;
    if (pos == 0) {
        popFront(head);
        return;
    }
    ListNode* cur = *head;
    for (int i = 0; i < pos - 1 && cur; i++) {
        cur = cur->next;
    }
    if (cur && cur->next) {
        ListNode* temp = cur->next;
        cur->next = temp->next;
        delete temp;
    }
}

// 查找节点
ListNode* find(ListNode* head, int val) {
    ListNode* cur = head;
    while (cur) {
        if (cur->val == val) return cur;
        cur = cur->next;
    }
    return nullptr;
}

// 反转链表
ListNode* reverse(ListNode* head) {
    ListNode* prev = nullptr;
    ListNode* cur = head;
    while (cur) {
        ListNode* next = cur->next;
        cur->next = prev;
        prev = cur;
        cur = next;
    }
    return prev;
}
```

### 完整示例

```cpp
#include <bits/stdc++.h>
using namespace std;

struct ListNode {
    int val;
    ListNode* next;
    ListNode(int x) : val(x), next(nullptr) {}
};

void printList(ListNode* head) {
    ListNode* cur = head;
    while (cur) {
        cout << cur->val;
        if (cur->next) cout << " -> ";
        cur = cur->next;
    }
    cout << endl;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    ListNode* head = nullptr;
    
    // 插入测试
    pushBack(&head, 1);
    pushBack(&head, 2);
    pushBack(&head, 3);
    pushFront(&head, 0);
    insertAt(&head, 2, 5);
    
    cout << "Original: ";
    printList(head);  // 0 -> 1 -> 5 -> 2 -> 3
    
    // 查找测试
    ListNode* node = find(head, 5);
    if (node) cout << "Found: " << node->val << endl;
    
    // 反转测试
    head = reverse(head);
    cout << "Reversed: ";
    printList(head);  // 3 -> 2 -> 5 -> 1 -> 0
    
    // 释放内存
    while (head) {
        popFront(&head);
    }
    
    return 0;
}
```

### 双向链表

```cpp
#include <bits/stdc++.h>
using namespace std;

struct DListNode {
    int val;
    DListNode* prev;
    DListNode* next;
    DListNode(int x) : val(x), prev(nullptr), next(nullptr) {}
};

class DoubleLinkedList {
private:
    DListNode* head;
    DListNode* tail;
    int size;
    
public:
    DoubleLinkedList() : head(nullptr), tail(nullptr), size(0) {}
    
    void pushFront(int val) {
        DListNode* newNode = new DListNode(val);
        if (!head) {
            head = tail = newNode;
        } else {
            newNode->next = head;
            head->prev = newNode;
            head = newNode;
        }
        size++;
    }
    
    void pushBack(int val) {
        DListNode* newNode = new DListNode(val);
        if (!tail) {
            head = tail = newNode;
        } else {
            newNode->prev = tail;
            tail->next = newNode;
            tail = newNode;
        }
        size++;
    }
    
    void popFront() {
        if (!head) return;
        DListNode* temp = head;
        head = head->next;
        if (head) head->prev = nullptr;
        else tail = nullptr;
        delete temp;
        size--;
    }
    
    void popBack() {
        if (!tail) return;
        DListNode* temp = tail;
        tail = tail->prev;
        if (tail) tail->next = nullptr;
        else head = nullptr;
        delete temp;
        size--;
    }
    
    int getSize() { return size; }
};
```

## 典型应用场景

### 栈和队列的实现

使用链表可以方便地实现[[../data-structure/stack|栈]]（先进后出）和队列（先进先出），特别是需要 $O(1)$ 的头部插入删除时。

### LRU 缓存

使用哈希表 + 双向链表实现 $O(1)$ 的 LRU（最近最少使用）缓存。

### 合并有序链表

```cpp
ListNode* mergeTwoLists(ListNode* l1, ListNode* l2) {
    ListNode dummy(0);
    ListNode* cur = &dummy;
    
    while (l1 && l2) {
        if (l1->val <= l2->val) {
            cur->next = l1;
            l1 = l1->next;
        } else {
            cur->next = l2;
            l2 = l2->next;
        }
        cur = cur->next;
    }
    cur->next = l1 ? l1 : l2;
    return dummy.next;
}
```

### 检测环形链表

```cpp
bool hasCycle(ListNode* head) {
    if (!head) return false;
    ListNode* slow = head;
    ListNode* fast = head;
    
    while (fast->next && fast->next->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) return true;
    }
    return false;
}
```

### 链表相交

```cpp
ListNode* getIntersectionNode(ListNode* headA, ListNode* headB) {
    if (!headA || !headB) return nullptr;
    
    ListNode* pa = headA;
    ListNode* pb = headB;
    
    while (pa != pb) {
        pa = pa ? pa->next : headB;
        pb = pb ? pb->next : headA;
    }
    
    return pa;
}
```

## 注意事项

1. **内存管理**：链表节点使用 `new` 分配，必须记得 `delete` 避免内存泄漏
2. **空指针检查**：进行任何节点操作前，应检查指针是否为空
3. **头尾指针**：对链表不熟悉时，使用**哨兵节点**（dummy head）可以简化边界处理
4. **迭代器失效**：在迭代过程中删除节点需要注意迭代器的正确更新

## 参考资料

[^1]: 链表的概念由艾伦·纽维尔（Allen Newell）、克里夫·肖（Cliff Shaw）和赫伯特·西蒙（Herbert A. Simon）在1955-1956年间提出，是最早的数据结构之一。
