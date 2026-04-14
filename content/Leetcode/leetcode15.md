---
title: 三数之和 (LeetCode 15)
date: 2026-04-11
description: 使用双指针法求解三数之和问题
tags:
  - two-pointers
  - array
draft: false
permalink:
---

## 三数之和 (LeetCode 15)

给你一个整数数组 `nums`，判断是否存在三元组 `[nums[i], nums[j], nums[k]]` 满足 `i != j`、`i != k` 且 `j != k`，同时满足 `nums[i] + nums[j] + nums[k] == 0`。返回所有满足条件的三元组，答案中不可包含重复的三元组。

### 算法思路

排序 + 双指针是解决此类问题的经典方法：

1. **排序**：将数组升序排列，便于使用双指针和去重
2. **枚举第一个数**：固定三元组的第一个数 $nums[i]$
3. **双指针找后两数**：在 $nums[i+1..]$ 区间内使用双指针 left 和 right
   - 若 $nums[i] + nums[left] + nums[right] == 0$，则找到一个解
   - 若和大于 0，说明 right 需要左移减小和
   - 若和小于 0，说明 left 需要右移增大和

### 去重策略

排序后，相同元素会相邻，可以通过以下规则去重：

- 第一个数 $nums[i]$：若 $nums[i] == nums[i-1]$，跳过（避免重复三元组）
- 双指针找到解后：同时移动 left 和 right，并跳过相同的元素

### C++ 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

class Solution {
public:
    vector<vector<int>> threeSum(vector<int>& nums) {
        vector<vector<int>> ans;
        sort(nums.begin(), nums.end());
        int n = nums.size();
        
        for (int i = 0; i < n - 2; ++i) {
            // 去重：跳过相同的第一个数
            if (i > 0 && nums[i] == nums[i - 1]) continue;
            
            // 优化：若最小的三个数之和大于0，无解
            if (nums[i] + nums[i + 1] + nums[i + 2] > 0) break;
            // 优化：若最大的三个数之和小于0，继续
            if (nums[i] + nums[n - 1] + nums[n - 2] < 0) continue;
            
            int left = i + 1, right = n - 1;
            while (left < right) {
                int sum = nums[i] + nums[left] + nums[right];
                if (sum == 0) {
                    ans.push_back({nums[i], nums[left], nums[right]});
                    // 去重：跳过相同的left和right
                    while (left < right && nums[left] == nums[left + 1]) ++left;
                    while (left < right && nums[right] == nums[right - 1]) --right;
                    ++left;
                    --right;
                } else if (sum < 0) {
                    ++left;
                } else {
                    --right;
                }
            }
        }
        return ans;
    }
};
```

### 复杂度分析

- **时间复杂度**：$O(n^2)$，双层循环遍历
- **空间复杂度**：$O(1)$ 或 $O(n)$（排序栈帧开销）

### 参考

- [[two-pointers|双指针技巧]]
