---
title: 寻找旋转排序数组中的最小值 (LeetCode 153)
date: 2026-04-11
description: 使用二分查找在旋转排序数组中寻找最小元素
tags:
  - binary-search
draft: false
permalink:
---

## 寻找旋转排序数组中的最小值 (LeetCode 153)

已知一个长度为 `n` 的升序数组在某个位置进行了旋转（例如 `[0,1,2,4,5,6,7]` 旋转后成为 `[4,5,6,7,0,1,2]`）。

给定一个旋转后的数组 `nums`，数组中的**所有元素都互不相同**，返回其中的最小元素。

### 二分查找思路

旋转排序数组具有**部分有序**的特性：

- 最小元素将数组分为两部分：左半部分和右半部分
- 左半部分的元素始终大于右半部分的元素
- 最小元素是右半部分的第一个元素

通过二分查找，比较中点元素与右端点元素：

- 若 $nums[mid] > nums[right]$，说明最小元素在右侧：`left = mid + 1`
- 若 $nums[mid] \leq nums[right]$，说明最小元素在左侧或就是 `mid`：`right = mid`

### C++ 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

class Solution {
public:
    int findMin(vector<int>& nums) {
        int left = 0, right = nums.size() - 1;
        
        while (left < right) {
            int mid = left + (right - left) / 2;
            // 若中点大于右端点，最小值在右侧
            if (nums[mid] > nums[right]) {
                left = mid + 1;
            } else {
                // nums[mid] <= nums[right]，最小值在左侧或就是mid
                right = mid;
            }
        }
        return nums[left];
    }
};
```

### 关键点说明

1. **比较右端点而非左端点**：避免数组完全有序时的边界问题
2. **循环终止条件**：`left == right`，此时指向最小元素
3. **严格小于等于**：`nums[mid] <= nums[right]` 时包含 `mid` 为最小值的情况

### 复杂度分析

- **时间复杂度**：$O(\log n)$，每次迭代将搜索区间减半
- **空间复杂度**：$O(1)$

### 相关题目

- [[leetcode154|LeetCode 154]] - 寻找旋转排序数组中的最小值 II（包含重复元素）

### 参考

- [[binary-search|二分查找模板]]
