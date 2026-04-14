---
title: 选择排序
date: 2026-03-31
description: 选择排序（Selection Sort）算法
tags:
  - sorting
  - selection-sort
draft: false
permalink:
---

## 简介

选择排序（Selection Sort）是一种简单直观的排序算法。其工作原理是：每次从待排序的数据元素中选出最小（或最大）的一个元素，存放在序列的起始位置，直到全部待排序的数据元素排完。[^1]

## 核心性质

### 时间复杂度

| 情况 | 时间复杂度 | 说明 |
|------|-----------|------|
| 最好 | $O(n^2)$ | 无论如何都需要遍历剩余未排序部分 |
| 平均 | $O(n^2)$ | $\frac{n(n-1)}{2}$ 次比较 |
| 最坏 | $O(n^2)$ | 每次都需要完整遍历 |

### 算法特性

- **不稳定性**：相等的元素可能因为交换而改变相对顺序
- **原地排序**：空间复杂度为 $O(1)$
- **交换次数**：最多 $n-1$ 次交换

选择排序的数学期望比较次数为：

$$
\sum_{i=1}^{n-1} (n-i) = \frac{n(n-1)}{2} = O(n^2)
$$

## 算法步骤

1. 在未排序序列中找到最小元素，记为 $min\_idx$
2. 将该元素与未排序序列的第一个元素交换
3. 排除已排序部分的第一个元素，重复步骤1-2
4. 直到所有元素排序完成

## 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

// 选择排序
void selectionSort(vector<int>& a) {
    int n = a.size();
    for (int i = 0; i < n - 1; ++i) {
        int minIdx = i;
        for (int j = i + 1; j < n; ++j) {
            if (a[j] < a[minIdx]) {
                minIdx = j;
            }
        }
        swap(a[i], a[minIdx]);
    }
}

// 双向选择排序（同时找最小和最大）
void doubleSelectionSort(vector<int>& a) {
    int n = a.size();
    int left = 0, right = n - 1;
    while (left < right) {
        int minIdx = left, maxIdx = left;
        for (int j = left; j <= right; ++j) {
            if (a[j] < a[minIdx]) minIdx = j;
            if (a[j] > a[maxIdx]) maxIdx = j;
        }
        swap(a[left], a[minIdx]);
        if (maxIdx == left) maxIdx = minIdx;  // 防止最大元素被换走
        swap(a[right], a[maxIdx]);
        ++left;
        --right;
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
    
    selectionSort(a);
    
    for (int x : a) {
        cout << x << " ";
    }
    return 0;
}
```

## 应用场景

- **教学用途**：排序算法入门，演示基本思想
- **内存受限环境**：只需要 $O(1)$ 额外空间
- **交换成本低**：当元素较大或交换成本高时（如磁盘排序），减少交换次数的优势
- **寻找第K小元素**：只需 $O(nk)$ 时间复杂度

## 与其他排序算法的对比

| 算法 | 最好 | 平均 | 最坏 | 空间 | 稳定 |
|------|------|------|------|------|------|
| 选择排序 | $O(n^2)$ | $O(n^2)$ | $O(n^2)$ | $O(1)$ | 否 |
| [[algorithms/insertion-sort|插入排序]] | $O(n)$ | $O(n^2)$ | $O(n^2)$ | $O(1)$ | 是 |
| [[algorithms/heap-sort|堆排序]] | $O(n\log n)$ | $O(n\log n)$ | $O(n\log n)$ | $O(1)$ | 否 |

## 参考资料

[^1]: 本词条内容来自 [OI Wiki - 选择排序](https://oi-wiki.org/basic/selection-sort/)
