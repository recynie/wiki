---
title: 归并排序
date: 2026-03-31
description: 归并排序（Merge Sort）算法
tags:
  - sorting
  - algorithm
draft: false
permalink:
---

## 定义

归并排序（Merge Sort）是一种基于**分治法**的高效排序算法，由约翰·冯·诺依曼（John von Neumann）于1945年提出。它的核心思想是将数组不断地二分，直到每个子数组只包含一个元素，然后**合并**这些有序子数组，最终得到完全有序的数组。[^1]

归并排序的关键在于"合并"过程：如何将两个已排序的子数组合并成一个有序数组。这个过程只需要线性时间 $O(n)$，因此整体时间复杂度为 $O(n \log n)$。

## 核心特性

### 时间复杂度

- **所有情况时间复杂度**：$O(n \log n)$ — 这是一个非常重要的性质！
- **空间复杂度**：$O(n)$ — 需要额外的辅助数组进行合并

无论输入数据的初始状态如何，归并排序都保证 $O(n \log n)$ 的时间复杂度，这使其适合对性能有严格要求的场景。[^2]

### 空间复杂度

归并排序需要 $O(n)$ 的额外空间来存储临时合并结果，这是它相对于快速排序的主要劣势。

### 稳定性

归并排序是**稳定**的排序算法。相等元素的相对顺序在排序后保持不变，这在某些应用场景中非常重要（如排序对象包含多个字段时）。

## 算法步骤

1. **分解**（Divide）：将数组从中间分成两半
2. **递归**（Conquer）：递归地对左右两半进行归并排序
3. **合并**（Merge）：将两个有序子数组合并成一个有序数组

合并过程是归并排序的核心，其步骤如下：
- 创建两个指针，分别指向两个子数组的起始位置
- 比较两个指针指向的元素，将较小的元素放入结果数组
- 重复上述过程，直到某个子数组全部处理完毕
- 将另一个子数组剩余的元素全部拷贝到结果数组

## 代码实现

### 自顶向下的递归实现

```cpp
#include <bits/stdc++.h>
using namespace std;

void merge(vector<int>& arr, int left, int mid, int right, vector<int>& temp) {
    // 合并两个有序数组：arr[left...mid] 和 arr[mid+1...right]
    int i = left;      // 左半部分的指针
    int j = mid + 1;   // 右半部分的指针
    int k = left;      // temp数组的指针
    
    // 比较并合并
    while (i <= mid && j <= right) {
        if (arr[i] <= arr[j]) {
            temp[k++] = arr[i++];
        } else {
            temp[k++] = arr[j++];
        }
    }
    
    // 处理剩余元素
    while (i <= mid) {
        temp[k++] = arr[i++];
    }
    while (j <= right) {
        temp[k++] = arr[j++];
    }
    
    // 将合并结果拷贝回原数组
    for (int i = left; i <= right; i++) {
        arr[i] = temp[i];
    }
}

void mergeSort(vector<int>& arr, int left, int right, vector<int>& temp) {
    if (left >= right) return;  // 递归终止条件
    
    int mid = left + (right - left) / 2;  // 防止整数溢出
    
    // 递归排序左右两半
    mergeSort(arr, left, mid, temp);
    mergeSort(arr, mid + 1, right, temp);
    
    // 合并两个有序子数组
    merge(arr, left, mid, right, temp);
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    vector<int> arr = {64, 34, 25, 12, 22, 11, 90};
    vector<int> temp(arr.size());
    mergeSort(arr, 0, arr.size() - 1, temp);
    
    for (int x : arr) cout << x << " ";
    cout << endl;
    return 0;
}
```

### 自底向上的迭代实现

```cpp
#include <bits/stdc++.h>
using namespace std;

void mergeSortIterative(vector<int>& arr) {
    int n = arr.size();
    vector<int> temp(n);
    
    // 逐步增大子数组大小
    for (int sz = 1; sz < n; sz *= 2) {
        // 对所有 sz 大小的子数组进行两两合并
        for (int left = 0; left < n - sz; left += 2 * sz) {
            int mid = min(left + sz - 1, n - 1);
            int right = min(left + 2 * sz - 1, n - 1);
            
            // 合并 arr[left...mid] 和 arr[mid+1...right]
            int i = left, j = mid + 1, k = left;
            while (i <= mid && j <= right) {
                if (arr[i] <= arr[j]) temp[k++] = arr[i++];
                else temp[k++] = arr[j++];
            }
            while (i <= mid) temp[k++] = arr[i++];
            while (j <= right) temp[k++] = arr[j++];
            
            for (int i = left; i <= right; i++) arr[i] = temp[i];
        }
    }
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    vector<int> arr = {64, 34, 25, 12, 22, 11, 90};
    mergeSortIterative(arr);
    
    for (int x : arr) cout << x << " ";
    cout << endl;
    return 0;
}
```

## 应用场景

- **外部排序**：当数据量过大无法全部加载到内存时，归并排序的分块读写特性非常适合
- **链表排序**：只需要 $O(1)$ 的额外空间（[[../data-structure/linked-list|链表]]）
- **稳定性要求**：需要保持相等元素相对顺序时
- **求逆序对数量**：在合并过程中可以统计逆序对[^3]

## 与其他排序算法的比较

| 特性 | 归并排序 | [[../algorithms/quick-sort|快速排序]] | [[../data-structure/heap|堆排序]] |
|------|----------|----------------|---------------|
| 平均时间复杂度 | $O(n \log n)$ | $O(n \log n)$ | $O(n \log n)$ |
| 最坏时间复杂度 | $O(n \log n)$ | $O(n^2)$ | $O(n \log n)$ |
| 空间复杂度 | $O(n)$ | $O(\log n)$ | $O(1)$ |
| 稳定性 | 稳定 | 不稳定 | 不稳定 |

## 参考资料

[^1]: 归并排序是最早被正式提出的 $O(n \log n)$ 排序算法之一，其理论分析奠定了算法分析的基础。

[^2]: 归并排序的 $O(n \log n)$ 时间复杂度是**确定性的**，不像快速排序依赖于输入数据的分布和 pivot 的选择。

[^3]: 求逆序对是归并排序的一个重要应用场景，可以在 $O(n \log n)$ 时间内同时完成排序和逆序对统计。
