---
title: 堆排序
date: 2026-03-31
description: 堆排序（Heap Sort）算法
tags:
  - sorting
  - heap-sort
  - heap
draft: false
permalink:
---

## 简介

堆排序（Heap Sort）是一种基于二叉堆（Binary Heap）数据结构的排序算法，由 Robert W. Floyd 于1964年提出。[^1]

堆排序利用堆这种数据结构的特点：堆是一棵完全二叉树，父节点的值总是大于（或小于）其子节点的值。最大堆（Max Heap）用于升序排序，最小堆（Min Heap）用于降序排序。

## 核心性质

### 堆的表示

堆可以用数组直接表示，无需指针：

- 父节点 $i$ 的左子节点：$2i + 1$
- 父节点 $i$ 的右子节点：$2i + 2$
- 子节点 $i$ 的父节点：$\lfloor (i-1)/2 \rfloor$

### 时间复杂度

| 操作 | 时间复杂度 | 说明 |
|------|-----------|------|
| 建堆 | $O(n)$ | 自底向上堆化 |
| 堆排序 | $O(n \log n)$ | 每次取出最大元素并堆化 |

### 算法特性

- **不稳定性**：相等的元素可能改变相对顺序
- **原地排序**：空间复杂度为 $O(1)$
- **时间复杂度稳定**：无论输入如何都是 $O(n \log n)$

## 算法步骤

### 建堆（Heapify）

将数组调整为最大堆，从最后一个非叶子节点 $\lfloor n/2 \rfloor - 1$ 开始，自底向上对每个节点执行下滤操作。

### 排序过程

1. 将堆顶元素（最大值）与堆的最后一个元素交换
2. 将堆的大小减1
3. 对新的堆顶元素执行下滤操作
4. 重复步骤1-3，直到堆大小为1

## 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

// 下滤操作，将第 i 个节点为根的子树调整为最大堆
void heapify(vector<int>& a, int n, int i) {
    int largest = i;
    int left = 2 * i + 1;
    int right = 2 * i + 2;
    
    if (left < n && a[left] > a[largest]) {
        largest = left;
    }
    if (right < n && a[right] > a[largest]) {
        largest = right;
    }
    if (largest != i) {
        swap(a[i], a[largest]);
        heapify(a, n, largest);
    }
}

// 建堆
void buildHeap(vector<int>& a) {
    int n = a.size();
    for (int i = n / 2 - 1; i >= 0; --i) {
        heapify(a, n, i);
    }
}

// 堆排序
void heapSort(vector<int>& a) {
    int n = a.size();
    buildHeap(a);
    for (int i = n - 1; i > 0; --i) {
        swap(a[0], a[i]);  // 将最大元素移到末尾
        heapify(a, i, 0);  // 减小堆大小，重新堆化
    }
}

// 堆排序（不使用额外函数，直接在排序过程中维护堆）
void heapSortCompact(vector<int>& a) {
    int n = a.size();
    // 建堆
    for (int i = n / 2 - 1; i >= 0; --i) {
        int parent = i;
        while (parent * 2 + 1 < n) {
            int child = parent * 2 + 1;
            if (child + 1 < n && a[child] < a[child + 1]) {
                ++child;
            }
            if (a[parent] < a[child]) {
                swap(a[parent], a[child]);
                parent = child;
            } else {
                break;
            }
        }
    }
    // 排序
    for (int i = n - 1; i > 0; --i) {
        swap(a[0], a[i]);
        int parent = 0;
        while (parent * 2 + 1 < i) {
            int child = parent * 2 + 1;
            if (child + 1 < i && a[child] < a[child + 1]) {
                ++child;
            }
            if (a[parent] < a[child]) {
                swap(a[parent], a[child]);
                parent = child;
            } else {
                break;
            }
        }
    }
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n;
    cin >> n;
    vector<int> a(n);
    for (int i = 0; i < n; ++i) {
        cin >> a[i];
    }
    
    heapSort(a);
    
    for (int x : a) {
        cout << x << " ";
    }
    return 0;
}
```

## 应用场景

- **优先级队列**：堆排序可用于实现高效的优先级队列
- **Top-K 问题**：维护大小为K的最小堆，可以高效找出前K个最大元素
- **滑动窗口**：在滑动窗口问题中维护有序结构
- **外排序**：当数据无法全部加载到内存时，堆排序的分批处理能力有用
- **与[[data-structure/heap|堆数据结构]]结合**：深入理解堆的性质和实现

## 与其他排序算法的对比

| 算法 | 最好 | 平均 | 最坏 | 空间 | 稳定 |
|------|------|------|------|------|------|
| 堆排序 | $O(n\log n)$ | $O(n\log n)$ | $O(n\log n)$ | $O(1)$ | 否 |
| [[algorithms/quick-sort|快速排序]] | $O(n\log n)$ | $O(n\log n)$ | $O(n^2)$ | $O(\log n)$ | 否 |
| [[algorithms/merge-sort|归并排序]] | $O(n\log n)$ | $O(n\log n)$ | $O(n\log n)$ | $O(n)$ | 是 |

## 参考资料

[^1]: 本词条内容来自 [OI Wiki - 堆排序](https://oi-wiki.org/basic/heap-sort/)
