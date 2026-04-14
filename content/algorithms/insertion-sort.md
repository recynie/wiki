---
title: 插入排序
date: 2026-03-31
description: 插入排序（Insertion Sort）算法
tags:
  - sorting
  - insertion-sort
draft: false
permalink:
---

## 简介

插入排序（Insertion Sort）是一种简单直观的排序算法，通过构建有序序列，对于未排序的元素，在已排序的序列中从后向前扫描，找到合适的位置并插入。[^1]

## 核心性质

### 时间复杂度

| 情况 | 时间复杂度 | 说明 |
|------|-----------|------|
| 最好 | $O(n)$ | 输入数组已经有序 |
| 平均 | $O(n^2)$ | 基本比较和交换 |
| 最坏 | $O(n^2)$ | 输入数组逆序 |

### 算法特性

- **稳定性**：是稳定的排序算法，相同元素的相对顺序不变
- **原地排序**：空间复杂度为 $O(1)$
- **在线排序**：可以边接收数据边排序，适合数据流场景

## 算法步骤

1. 从第二个元素开始（假设第一个元素已排序）
2. 对于当前元素 $key$：
   - 与已排序部分的元素从后向前比较
   - 若比较元素大于 $key$，则向后移动一位
   - 直到找到小于等于 $key$ 的元素或到达序列开头
3. 将 $key$ 插入到空出的位置
4. 重复步骤2-3直到所有元素排序完成

## 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

// 插入排序
void insertionSort(vector<int>& a) {
    int n = a.size();
    for (int i = 1; i < n; ++i) {
        int key = a[i];
        int j = i - 1;
        while (j >= 0 && a[j] > key) {
            a[j + 1] = a[j];
            --j;
        }
        a[j + 1] = key;
    }
}

// 带哨兵的插入排序（减少判断）
void insertionSortWithSentinel(vector<int>& a) {
    int n = a.size();
    // 将最小元素放到首位作为哨兵
    int minIdx = 0;
    for (int i = 1; i < n; ++i) {
        if (a[i] < a[minIdx]) {
            minIdx = i;
        }
    }
    swap(a[0], a[minIdx]);
    
    for (int i = 2; i < n; ++i) {
        int key = a[i];
        int j = i - 1;
        while (a[j] > key) {
            a[j + 1] = a[j];
            --j;
        }
        a[j + 1] = key;
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
    
    insertionSort(a);
    
    for (int x : a) {
        cout << x << " ";
    }
    return 0;
}
```

## 应用场景

- **小规模数据**：对于 $n \leq 50$ 的数据，插入排序性能优于高级排序算法
- **几乎有序的数据**：当数据基本有序时，插入排序可达到 $O(n)$ 时间
- **链表排序**：在链表中插入操作只需 $O(1)$，整体 $O(n^2)$
- **在线排序**：服务器接收数据流，边接收边排序输出
- **结合其他排序**：如 [[algorithms/heap-sort|堆排序]] 的子过程，用于优化小区间排序

## 与其他排序算法的对比

| 算法 | 最好 | 平均 | 最坏 | 空间 | 稳定 |
|------|------|------|------|------|------|
| 插入排序 | $O(n)$ | $O(n^2)$ | $O(n^2)$ | $O(1)$ | 是 |
| [[algorithms/selection-sort|选择排序]] | $O(n^2)$ | $O(n^2)$ | $O(n^2)$ | $O(1)$ | 否 |
| [[algorithms/heap-sort|堆排序]] | $O(n\log n)$ | $O(n\log n)$ | $O(n\log n)$ | $O(1)$ | 否 |

## 参考资料

[^1]: 本词条内容来自 [OI Wiki - 插入排序](https://oi-wiki.org/basic/insertion-sort/)
