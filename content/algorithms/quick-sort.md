---
title: 快速排序
date: 2026-03-31
description: 快速排序（Quick Sort）算法
tags:
  - sorting
  - algorithm
draft: false
permalink:
---

## 定义

快速排序是一种高效的排序算法，由英国计算机科学家托尼·霍尔（Tony Hoare）于1959年发明。它采用**分治法**（Divide and Conquer）策略，通过递归将问题不断缩小规模，最终得到有序结果。

快速排序的核心思想是：选择一个**基准元素**（pivot），将数组划分为两个子数组，使得左子数组中的所有元素都小于等于 pivot，右子数组中的所有元素都大于等于 pivot，然后递归地对这两个子数组进行排序。[^1]

## 核心特性

### 时间复杂度

- **平均时间复杂度**：$O(n \log n)$
- **最坏时间复杂度**：$O(n^2)$，当输入数组已经有序或完全逆序时发生
- **最佳时间复杂度**：$O(n \log n)$

快速排序的常数因子较小，在实际应用中往往比同复杂度的其他排序算法更快，因此被广泛采用。[^2]

### 空间复杂度

快速排序是**原地排序**（in-place sorting），空间复杂度为 $O(\log n)$，主要用于递归调用栈。

### 稳定性

快速排序是**不稳定**的排序算法。即相等元素的相对顺序可能在排序后发生改变。

## 算法步骤

1. **选择基准元素**（pivot）：可以选择首元素、尾元素、中间元素或随机元素
2. **分区**（partition）：将数组划分为两个部分
   - 左侧：所有元素 $\leq$ pivot
   - 右侧：所有元素 $>$ pivot
3. **递归排序**：对左右两个子数组分别递归执行快速排序
4. **合并**：由于是原地排序，无需额外合并步骤

## 代码实现

### 基础版本

```cpp
#include <bits/stdc++.h>
using namespace std;

// 分区函数：返回pivot的最终位置
int partition(vector<int>& arr, int left, int right) {
    int pivot = arr[right];  // 选择右端元素作为pivot
    int i = left - 1;       // i是小于pivot区域的最后一个索引
    
    for (int j = left; j < right; j++) {
        if (arr[j] <= pivot) {
            i++;
            swap(arr[i], arr[j]);
        }
    }
    swap(arr[i + 1], arr[right]);  // 将pivot放到正确位置
    return i + 1;
}

void quickSort(vector<int>& arr, int left, int right) {
    if (left < right) {
        int pivotPos = partition(arr, left, right);
        quickSort(arr, left, pivotPos - 1);   // 排序左半部分
        quickSort(arr, pivotPos + 1, right);  // 排序右半部分
    }
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    vector<int> arr = {64, 34, 25, 12, 22, 11, 90};
    quickSort(arr, 0, arr.size() - 1);
    
    for (int x : arr) cout << x << " ";
    cout << endl;
    return 0;
}
```

### 双指针版本（更常用）

```cpp
#include <bits/stdc++.h>
using namespace std;

void quickSort(vector<int>& arr, int left, int right) {
    if (left >= right) return;
    
    int pivot = arr[(left + right) / 2];  // 选择中间元素作为pivot
    int i = left, j = right;
    
    while (i <= j) {
        while (arr[i] < pivot) i++;
        while (arr[j] > pivot) j--;
        if (i <= j) {
            swap(arr[i], arr[j]);
            i++;
            j--;
        }
    }
    
    quickSort(arr, left, j);   // 排序左半部分
    quickSort(arr, i, right);  // 排序右半部分
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    vector<int> arr = {64, 34, 25, 12, 22, 11, 90};
    quickSort(arr, 0, arr.size() - 1);
    
    for (int x : arr) cout << x << " ";
    cout << endl;
    return 0;
}
```

## 应用场景

- **通用排序需求**：当需要高效排序且不需要稳定排序时
- **大规模数据排序**：由于 $O(\log n)$ 的空间复杂度，适合内存受限场景
- **Top-K 问题**：可用于快速找到第 K 大的元素
- **数组划分**：如荷兰国旗问题、颜色分类等[^3]

## 与其他排序算法的比较

| 特性 | 快速排序 | [[../data-structure/merge-sort|归并排序]] | [[../algorithms/quick-sort|堆排序]] |
|------|----------|--------------------|---------------|
| 平均时间复杂度 | $O(n \log n)$ | $O(n \log n)$ | $O(n \log n)$ |
| 最坏时间复杂度 | $O(n^2)$ | $O(n \log n)$ | $O(n \log n)$ |
| 空间复杂度 | $O(\log n)$ | $O(n)$ | $O(1)$ |
| 稳定性 | 不稳定 | 稳定 | 不稳定 |

## 参考资料

[^1]: 快速排序的思想最初由约翰·冯·诺依曼（John von Neumann）提出，但霍尔的版本是最常用实现。参考：[Quick Sort - OI Wiki](https://oi-wiki.org/basic/quick-sort)

[^2]: 快速排序的实际性能高度依赖于 pivot 的选择策略。随机化选择可以有效避免最坏情况的发生。

[^3]: 快速排序在处理近乎有序的数组时性能会退化到 $O(n^2)$，此时可以考虑使用[[../data-structure/merge-sort|归并排序]]或其他基于比较的排序算法。
