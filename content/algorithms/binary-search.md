---
title: 二分查找
date: 2026-03-31
description: 二分查找（Binary Search）算法
tags:
  - binary-search
  - algorithm
draft: false
permalink:
---

## 定义

二分查找（Binary Search）是一种在**有序数组**中查找目标元素的高效算法。其核心思想是利用数组的有序性，每次将搜索范围缩小一半，从而实现 $O(\log n)$ 的时间复杂度。[^1]

二分查找的思想虽然简单，但实现时边界条件的处理容易出错，需要特别注意。

## 核心特性

### 时间复杂度

- **时间复杂度**：$O(\log n)$ — 每一次比较都将搜索范围缩小一半
- **空间复杂度**：$O(1)$（迭代实现）或 $O(\log n)$（递归实现）

### 适用条件

二分查找要求：
1. **数组必须有序**（升序或降序）
2. **能够通过索引随机访问**（数组，不适用于链表）
3. **存在明确的上下界**

## 算法步骤

1. 确定搜索范围的左边界 `left` 和右边界 `right`
2. 计算中点 $mid = \lfloor (left + right) / 2 \rfloor$
3. 比较 `arr[mid]` 与目标值 `target`：
   - 若 `arr[mid] == target`，找到目标，返回 `mid`
   - 若 `arr[mid] < target`，目标在右半部分，更新 `left = mid + 1`
   - 若 `arr[mid] > target`，目标在左半部分，更新 `right = mid - 1`
4. 重复步骤 2-3，直到 `left > right`（未找到）

## 代码实现

### 查找精确值

```cpp
#include <bits/stdc++.h>
using namespace std;

// 查找精确值，找到返回索引，未找到返回 -1
int binarySearch(vector<int>& arr, int target) {
    int left = 0, right = arr.size() - 1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;  // 防止整数溢出
        
        if (arr[mid] == target) {
            return mid;
        } else if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    
    return -1;  // 未找到
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    vector<int> arr = {1, 3, 5, 7, 9, 11, 13, 15};
    cout << binarySearch(arr, 7) << endl;   // 输出: 3
    cout << binarySearch(arr, 6) << endl;   // 输出: -1
    return 0;
}
```

### 查找左边界（第一个 $\geq$ target 的位置）

```cpp
#include <bits/stdc++.h>
using namespace std;

// 查找左边界：第一个 >= target 的元素索引
int lowerBound(vector<int>& arr, int target) {
    int left = 0, right = arr.size();  // 注意：right = n（不取等）
    
    while (left < right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid;  // 继续向左收缩
        }
    }
    
    return left;  // 可能等于 n（未找到）
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    vector<int> arr = {1, 3, 5, 7, 9, 11, 13, 15};
    cout << lowerBound(arr, 6) << endl;   // 输出: 3 (arr[3] = 7)
    cout << lowerBound(arr, 7) << endl;   // 输出: 3 (arr[3] = 7)
    cout << lowerBound(arr, 16) << endl;  // 输出: 8 (未找到，等于 n)
    return 0;
}
```

### 查找右边界（最后一个 $\leq$ target 的位置）

```cpp
#include <bits/stdc++.h>
using namespace std;

// 查找右边界：最后一个 <= target 的元素索引
int upperBound(vector<int>& arr, int target) {
    int left = 0, right = arr.size();  // 注意：right = n
    
    while (left < right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] <= target) {
            left = mid + 1;  // 继续向右收缩
        } else {
            right = mid;
        }
    }
    
    return left - 1;  // 可能等于 -1（未找到）
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    vector<int> arr = {1, 3, 5, 7, 9, 11, 13, 15};
    cout << upperBound(arr, 6) << endl;   // 输出: 2 (arr[2] = 5)
    cout << upperBound(arr, 7) << endl;  // 输出: 3 (arr[3] = 7)
    cout << upperBound(arr, 0) << endl;   // 输出: -1 (未找到)
    return 0;
}
```

### 通用模板

```cpp
#include <bits/stdc++.h>
using namespace std;

// 二分查找通用模板
int binarySearch(vector<int>& arr, int target) {
    int left = 0, right = arr.size() - 1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] == target) {
            return mid;
        } else if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    
    return -1;
}

// 寻找左侧边界
int lowerBound(vector<int>& arr, int target) {
    int left = 0, right = arr.size();
    
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (arr[mid] < target) left = mid + 1;
        else right = mid;
    }
    
    return left;
}

// 寻找右侧边界
int upperBound(vector<int>& arr, int target) {
    int left = 0, right = arr.size();
    
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (arr[mid] <= target) left = mid + 1;
        else right = mid;
    }
    
    return left - 1;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    vector<int> arr = {1, 3, 5, 7, 9, 11, 13, 15};
    
    cout << binarySearch(arr, 7) << endl;   // 3
    cout << lowerBound(arr, 6) << endl;     // 3
    cout << upperBound(arr, 6) << endl;     // 2
    
    return 0;
}
```

## 典型应用场景

### 查找问题

- 在有序数组中查找特定元素
- 查找元素的插入位置
- 检查元素是否存在

### 求极值问题

- **寻找峰值元素**：数组中某个元素大于其相邻元素
- **寻找旋转排序数组中的最小值**[^2]
- **在二维矩阵中搜索**（矩阵每行每列均递增）

### 最优化问题

二分查找常用于求解**最大化最小值**或**最小化最大值**问题：

- 分配最大资源最小化时间（"搬运货物"、"渡河问题"）
- 最大化最小距离（"天线安置"、"最小注射速率"）

这类问题的解决思路是：先验证答案可行性，再二分搜索最优值。

## 参考资料

[^1]: 二分查找由约翰·穆奇（John Mauchly）在1946年首次提出，是计算机科学中最基本的算法之一。

[^2]: 旋转排序数组的搜索是二分查找的经典变体，需要根据中点与左右端点的关系判断哪一半是有序的。
