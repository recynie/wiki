---
title: 递归&分治
date: 2026-03-31
description: 递归与分治（Recursion & Divide and Conquer）
tags:
  - divide-and-conquer
  - recursion
  - algorithm
draft: false
permalink:
---

## 定义

**递归**（Recursion）是指函数调用自身来解决问题的编程技巧。递归将原问题分解为**更小的同类子问题**，直到子问题足够简单可以直接解决，然后逐层回溯合并子问题的解得到原问题的解。

**分治法**（Divide and Conquer）是一种基于递归的算法设计策略。它将问题分成 $n$ 个规模较小且结构与原问题相似的子问题，递归解决这些子问题，然后再合并其结果，得到原问题的解。[^1]

## 递归三要素

一个正确的递归程序必须满足：

1. **基本情况**（Base Case）：递归终止条件，确保不会无限递归
2. **递归调用**：将问题规模缩小
3. **状态传递**：确保子问题的解能正确合并

## 分治法步骤

分治法的一般步骤：

1. **分解**（Divide）：将原问题分割成若干个规模较小的子问题
2. **解决**（Conquer）：递归解决每个子问题，如果子问题足够小则直接求解
3. **合并**（Combine）：将子问题的解合并为原问题的解

## 经典分治算法

### 归并排序

[[../algorithms/merge-sort|归并排序]]是分治法的典型应用：

```cpp
#include <bits/stdc++.h>
using namespace std;

void merge(vector<int>& arr, int left, int mid, int right, vector<int>& temp) {
    int i = left, j = mid + 1, k = left;
    
    while (i <= mid && j <= right) {
        if (arr[i] <= arr[j]) temp[k++] = arr[i++];
        else temp[k++] = arr[j++];
    }
    while (i <= mid) temp[k++] = arr[i++];
    while (j <= right) temp[k++] = arr[j++];
    
    for (int i = left; i <= right; i++) {
        arr[i] = temp[i];
    }
}

void mergeSort(vector<int>& arr, int left, int right, vector<int>& temp) {
    if (left >= right) return;  // 基本情况
    
    int mid = left + (right - left) / 2;  // 分解
    
    mergeSort(arr, left, mid, temp);        // 解决左半部分
    mergeSort(arr, mid + 1, right, temp);   // 解决右半部分
    
    merge(arr, left, mid, right, temp);      // 合并
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

### 快速排序

[[../algorithms/quick-sort|快速排序]]同样采用分治策略：

```cpp
#include <bits/stdc++.h>
using namespace std;

int partition(vector<int>& arr, int left, int right) {
    int pivot = arr[right];
    int i = left - 1;
    
    for (int j = left; j < right; j++) {
        if (arr[j] <= pivot) {
            i++;
            swap(arr[i], arr[j]);
        }
    }
    swap(arr[i + 1], arr[right]);
    return i + 1;
}

void quickSort(vector<int>& arr, int left, int right) {
    if (left >= right) return;  // 基本情况
    
    int pivotPos = partition(arr, left, right);  // 分解
    
    quickSort(arr, left, pivotPos - 1);   // 解决左半部分
    quickSort(arr, pivotPos + 1, right);  // 解决右半部分
}
```

### 二分查找

[[../algorithms/binary-search|二分查找]]是分治思想的简单应用：

```cpp
#include <bits/stdc++.h>
using namespace std;

int binarySearch(vector<int>& arr, int target, int left, int right) {
    if (left > right) return -1;  // 基本情况：未找到
    
    int mid = left + (right - left) / 2;  // 分解
    
    if (arr[mid] == target) {
        return mid;  // 解决：找到目标
    } else if (arr[mid] < target) {
        return binarySearch(arr, target, mid + 1, right);  // 解决右半部分
    } else {
        return binarySearch(arr, target, left, mid - 1);  // 解决左半部分
    }
}
```

### 快速幂

分治法可以高效计算幂运算：

```cpp
#include <bits/stdc++.h>
using namespace std;

// 计算 a^n（n 可以很大）
long long fastPower(long long a, long long n) {
    if (n == 0) return 1;  // 基本情况
    
    long long half = fastPower(a, n / 2);  // 分解并解决子问题
    
    if (n % 2 == 0) {
        return half * half;  // 合并
    } else {
        return half * half * a;  // 合并
    }
}

// 矩阵快速幂
vector<vector<long long>> matrixMultiply(
    const vector<vector<long long>>& A,
    const vector<vector<long long>>& B) {
    int n = A.size();
    vector<vector<long long>> C(n, vector<long long>(n, 0));
    
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            for (int k = 0; k < n; k++) {
                C[i][j] += A[i][k] * B[k][j];
            }
        }
    }
    return C;
}

vector<vector<long long>> matrixPower(
    vector<vector<long long>> A, long long n) {
    int m = A.size();
    vector<vector<long long>> result(m, vector<long long>(m, 0));
    
    // 单位矩阵
    for (int i = 0; i < m; i++) {
        result[i][i] = 1;
    }
    
    while (n > 0) {
        if (n % 2 == 1) {
            result = matrixMultiply(result, A);
        }
        A = matrixMultiply(A, A);
        n /= 2;
    }
    
    return result;
}
```

## 递归算法分析

### 时间复杂度分析

递归算法的时间复杂度通常用**主定理**（Master Theorem）来分析：

对于形如 $T(n) = a T(n/b) + f(n)$ 的递归式，其中 $a \geq 1, b > 1$：

- 若 $f(n) = O(n^{\log_b a - \epsilon})$，则 $T(n) = \Theta(n^{\log_b a})$
- 若 $f(n) = \Theta(n^{\log_b a} \log^k n)$，则 $T(n) = \Theta(n^{\log_b a} \log^{k+1} n)$
- 若 $f(n) = \Omega(n^{\log_b a + \epsilon})$，且 $a f(n/b) \leq c f(n)$，则 $T(n) = \Theta(f(n))$

例如，归并排序的递归式为 $T(n) = 2T(n/2) + O(n)$，因此 $T(n) = O(n \log n)$。[^2]

## 递归与迭代的比较

| 特性 | 递归 | 迭代 |
|------|------|------|
| 代码简洁性 | 通常更简洁 | 较复杂 |
| 空间复杂度 | $O(n)$ 栈空间 | $O(1)$ |
| 执行效率 | 函数调用开销 | 无额外开销 |
| 可读性 | 直观表达问题结构 | 需要理解循环不变式 |

## 尾递归优化

如果递归调用是函数的最后一步操作，某些编译器可以优化为迭代，避免栈空间增长：

```cpp
// 尾递归版本
long long factorialTail(long long n, long long acc = 1) {
    if (n == 0) return acc;
    return factorialTail(n - 1, n * acc);  // 尾递归
}

// 编译器可能优化为：
long long factorialIter(long long n) {
    long long acc = 1;
    while (n > 0) {
        acc *= n;
        n--;
    }
    return acc;
}
```

## 参考资料

[^1]: 分治法是一种古老而强大的算法策略，最早在欧几里得算法（求最大公约数）中就有体现。

[^2]: 主定理由乔恩·本杰明·拜耳（Jon Bentley）、赫伯特·黄（Herbert H. W. Lam）和德米特里·罗斯（Dorothy M. R. Ross）于1986年正式提出。
