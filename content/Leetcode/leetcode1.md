---
title: LeetCode 1 - 两数之和
date: 2026-04-06
description: 给定一个整数数组nums和目标值target，返回两数之和等于target的索引
tags:
  - hash-table
  - array
  - leetcode
draft: false
permalink:
---

# LeetCode 1 - 两数之和

## 题目描述

给定一个整数数组 `nums` 和一个整数目标值 `target`，请你在该数组中找出**两数之和**等于 `target` 的两个整数索引，并返回它们。

假设每种输入只会对应**一个答案**，且同样的元素不能被重复利用。

### 示例

```
输入：nums = [2,7,11,15], target = 9
输出：[0,1]
解释：nums[0] + nums[1] = 2 + 7 = 9

输入：nums = [3,2,4], target = 6
输出：[1,2]

输入：nums = [3,3], target = 6
输出：[0,1]
```

## 解法一：哈希表（最优解）

### 思路分析

遍历数组，对于每个元素 `nums[i]`，检查 `target - nums[i]` 是否已经在哈希表中：
- 如果在，返回 `{map[target-nums[i]], i}`
- 如果不在，将 `nums[i]` 存入哈希表

### 代码实现

```cpp
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        unordered_map<int, int> mp;  // value -> index
        
        for (int i = 0; i < nums.size(); ++i) {
            int complement = target - nums[i];
            if (mp.count(complement)) {
                return {mp[complement], i};
            }
            mp[nums[i]] = i;
        }
        
        return {};  // 题目保证有解，不会执行到这里
    }
};
```

### 复杂度分析

| 复杂度 | 值 |
|--------|-----|
| 时间复杂度 | $O(n)$ |
| 空间复杂度 | $O(n)$ |

## 解法二：暴力枚举（不推荐）

```cpp
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        for (int i = 0; i < nums.size(); ++i) {
            for (int j = i + 1; j < nums.size(); ++j) {
                if (nums[i] + nums[j] == target) {
                    return {i, j};
                }
            }
        }
        return {};
    }
};
```

### 复杂度分析

| 复杂度 | 值 |
|--------|-----|
| 时间复杂度 | $O(n^2)$ |
| 空间复杂度 | $O(1)$ |

## 解法三：双指针（需排序，会丢失原始索引）

```cpp
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        vector<pair<int, int>> v;
        for (int i = 0; i < nums.size(); ++i) {
            v.push_back({nums[i], i});
        }
        
        sort(v.begin(), v.end());
        int l = 0, r = nums.size() - 1;
        
        while (l < r) {
            int sum = v[l].first + v[r].first;
            if (sum == target) {
                return {v[l].second, v[r].second};
            } else if (sum < target) {
                ++l;
            } else {
                --r;
            }
        }
        
        return {};
    }
};
```

## 三种解法比较

| 解法 | 时间复杂度 | 空间复杂度 | 特点 |
|------|-----------|-----------|------|
| 哈希表 | $O(n)$ | $O(n)$ | **最优**，一遍遍历 |
| 暴力枚举 | $O(n^2)$ | $O(1)$ | 简单但低效 |
| 双指针 | $O(n \log n)$ | $O(n)$ | 需排序，丢失索引 |

## 关键点

1. **哈希表的优势**：
   - $O(1)$ 的查找复杂度
   - 一遍遍历即可完成
   - 空间换时间的经典应用

2. **为什么不用双指针在原数组上**：
   - 原数组无序时双指针无效
   - 排序会改变元素原始索引

3. **哈希表的空间优化**：
   - 只需存储已遍历的元素
   - 不需要事先存储所有元素

## 相关题目

| 题目 | 难度 | 说明 |
|------|------|------|
| [[leetcode15|15. 三数之和]] | 中等 | 数组中三数之和为零 |
| [[leetcode18|18. 四数之和]] | 中等 | 四数之和 |
| [[leetcode454|454. 四数相加 II]] | 中等 | 四数相加等于目标 |

## 参考

- [LeetCode 1 - 两数之和](https://leetcode.cn/problems/two-sum/)
- [[../data-structure/hash-table|哈希表]]
