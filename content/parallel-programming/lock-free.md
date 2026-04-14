---
title: Lock-free Data Structures
date: 2026-04-10
description: 无锁数据结构——基于CAS原子操作的高并发数据结构设计
tags:
  - lock-free
  - concurrency
  - cas
  - atomic-operations
draft: false
permalink:
---

## 概述

传统并发数据结构依赖**锁**来保证线程安全——每次访问共享数据都需要获取锁，在高并发场景下，锁竞争成为性能瓶颈。**无锁数据结构（Lock-free Data Structures）** 使用原子操作（通常是CAS——Compare-And-Swap）来实现并发安全，允许多线程同时访问数据结构时系统整体依然保持进展。[^1]

无锁数据结构的核心优势：
- **无死锁**：不依赖锁，因此永远不会因为锁顺序导致死锁
- **高并发**：多个线程可以同时尝试修改结构
- **实时友好**：锁持有时间最小化，对响应时间更可预测

## CAS原子操作

### 什么是CAS

CAS是CPU提供的硬件级原子指令。在x86架构上对应 `LOCK_CMPXCHG` 指令：

```cpp
// CAS伪代码
bool CAS(T* addr, T expected, T desired) {
    if (*addr == expected) {
        *addr = desired;
        return true;  // 交换成功
    }
    return false;     // 失败，值已被其他线程修改
}
```

整个操作是原子的——硬件保证检查和赋值之间不会被中断。

### C++中的CAS

```cpp
#include <atomic>

std::atomic<int> counter{0};

// 尝试将counter从expected更新为desired
int expected = counter.load();
int desired = expected + 1;
counter.compare_exchange_strong(expected, desired);
// 如果成功，counter已被原子地递增
```

C++提供两种CAS变体：
- `compare_exchange_strong`：失败时立即返回
- `compare_exchange_weak`：可能spurious失败，但性能更好（适合循环中）

## 无锁队列实现

无锁队列是最常见的无锁数据结构之一。Michael-Scott队列（Java `ConcurrentLinkedQueue` 的基础）是最经典的实现。[^2]

### 队列节点结构

```cpp
struct Node {
    T* data;
    std::atomic<Node*> next;
};

class LockFreeQueue {
private:
    std::atomic<Node*> head;
    std::atomic<Node*> tail;
    
public:
    LockFreeQueue() {
        Node* dummy = new Node();
        dummy->data = nullptr;
        head.store(dummy);
        tail.store(dummy);
    }
    
    void enqueue(T* value) {
        Node* new_node = new Node();
        new_node->data = value;
        new_node->next.store(nullptr);
        
        Node* old_tail;
        while (true) {
            old_tail = tail.load();
            Node* next = old_tail->next.load();
            if (next == nullptr) {
                // 尝试将新节点接到tail后面
                if (old_tail->next.compare_exchange_weak(next, new_node)) {
                    break;
                }
            } else {
                // tail落后了，推进它
                tail.compare_exchange_weak(old_tail, next);
            }
        }
        // 更新tail
        tail.compare_exchange_weak(old_tail, new_node);
    }
    
    T* dequeue() {
        Node* old_head;
        while (true) {
            old_head = head.load();
            Node* tail = tail.load();
            Node* next = old_head->next.load();
            
            if (old_head == tail.load()) {
                if (next == nullptr) {
                    return nullptr;  // 队列空
                }
                // tail落后，推进它
                tail.compare_exchange_weak(tail, next);
            } else {
                if (head.compare_exchange_weak(old_head, next)) {
                    T* result = next->data;
                    // 可以选择释放旧head节点
                    return result;
                }
            }
        }
    }
};
```

### 单CAS入队/出队优化

2004年的研究提出了更高效的变体，使用双向链表实现**单CAS操作**完成入队和出队：

```
入队：CAS(tail, old_tail, new_node)  // 仅需一次CAS
出队：CAS(head, old_head, old_head->next)  // 仅需一次CAS
```

关键在于维护`prev`指针，使得在CAS之前就能确认节点有效性。[^3]

## ABA问题

### 问题描述

CAS最大的陷阱是**ABA问题**：

```
线程A: 读取地址X，值为A
线程B: 将X从A改为B，再改回A
线程A: CAS(X, A, C) 成功！
```

线程A无法感知中间发生过变化。在某些算法中，这会导致严重错误。

### 解决方案

**1. 指针标记（Tagged Pointers）**

将计数器与指针打包：

```cpp
struct TaggedPtr {
    Node* ptr;
    uint64_t tag;
};

std::atomic<TaggedPtr> head;
```

每次修改时递增tag，CAS时比较整个结构。

**2. Double-Word CAS（DCAS）**

在支持双字CAS的架构上，可以原子地更新两个指针。

**3. Hazard Pointer / RCU**

让每个线程声明"当前访问的节点"，垃圾回收器延迟释放节点。

## 无锁等级

Herlihy定义的并发数据结构能力等级：[^4]

| 等级 | 描述 | 示例 |
|------|------|------|
| **Obstruction-free** | 无竞争时单线程可完成操作 | 基础自旋锁 |
| **Lock-free** | 系统整体保证进展 | 无锁队列 |
| **Wait-free** | 每个线程都保证有限步内完成 | Wait-free队列 |

Lock-free是实际应用中最常见的等级——它保证"某些线程"会取得进展，而非每个线程。

## 无锁栈

无锁栈是最简单的无锁结构之一：

```cpp
class LockFreeStack {
private:
    std::atomic<Node*> top{nullptr};
    
public:
    void push(T* value) {
        Node* new_node = new Node(value);
        Node* old_top;
        do {
            old_top = top.load();
            new_node->next.store(old_top);
        } while (!top.compare_exchange_weak(old_top, new_node));
    }
    
    T* pop() {
        Node* old_top;
        Node* next;
        do {
            old_top = top.load();
            if (old_top == nullptr) return nullptr;
            next = old_top->next.load();
        } while (!top.compare_exchange_weak(old_top, next));
        
        T* result = old_top->data;
        delete old_top;  // 注意：实际需要 hazard pointer 防止use-after-free
        return result;
    }
};
```

## 应用场景

- **高性能消息队列**：如Java `ConcurrentLinkedQueue`
- **内存分配器**：如jemalloc使用无锁结构
- **计数器**：如`std::atomic`操作
- **索引结构**：分布式系统中的无锁跳表

## 局限性

1. **实现困难**：正确性难以保证，调试成本高
2. **不一定更快**：低竞争场景下，锁可能更简单更快
3. **内存管理复杂**：需要额外的机制解决回收问题
4. **不是银弹**：对于需要复杂组合操作的结构，可能需要事务内存

## 参考

[^1]: [A Go Engineer's Guide to Lock-Free Data Structures](https://medium.com/@speedcraft21/a-go-engineers-guide-to-designing-lock-free-data-structures-5781d31e9146)
[^2]: [Michael & Scott, Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms](https://people.csail.mit.edu/shanir/publications/FIFO_Queues.pdf)
[^3]: [Fast and Lock-Free Concurrent Queue Algorithms](https://www.cl.cam.ac.uk/research/srg/netos/papers/2007-cpwl.pdf)
[^4]: [Stroustrup - Lock-Free Vector](http://stroustrup.com/lock-free-vector.pdf)
