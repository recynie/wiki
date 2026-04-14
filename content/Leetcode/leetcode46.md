---
title: LeetCode 46 - 全排列
date: 2026-04-06
description: 给定一个不含重复数字的数组，返回其所有可能的排列
tags:
  - backtracking
  - leetcode
draft: false
permalink:
---

# LeetCode 46 - 全排列

## 题目描述

给定一个 **不含重复数字** 的整数数组 `nums`，返回其所有可能的**全排列**。你可以按任意顺序返回答案。

### 示例

```
输入：nums = [1,2,3]
输出：[[1,2,3],[1,3,2],[2,1,3],[2,3,1],[3,1,2],[3,2,1]]

输入：nums = [0,1]
输出：[[0,1],[1,0]]

输入：nums = [1]
输出：[[1]]
```

## 解法一：回溯法

### 思路分析

对于全排列问题，我们枚举每个位置可以放置的数：

1. 使用 `used` 数组标记哪些数字已经被使用
2. 使用 `path` 存储当前排列
3. 当 `path` 长度等于 `nums` 长度时，找到一个完整排列
4. 递归尝试所有未使用的数字

### 代码实现

```cpp
class Solution {
public:
    vector<vector<int>> permute(vector<int>& nums) {
        vector<vector<int>> result;
        vector<int> path;
        vector<int> used(nums.size(), 0);
        
        function<void()> backtrack = [&]() {
            // 找到完整排列
            if (path.size() == nums.size()) {
                result.push_back(path);
                return;
            }
            
            // 枚举可以放置的数字
            for (int i = 0; i < nums.size(); ++i) {
                if (used[i]) continue;  // 已被使用，跳过
                
                // 做选择
                used[i] = 1;
                path.push_back(nums[i]);
                
                // 递归
                backtrack();
                
                // 撤销选择
                path.pop_back();
                used[i] = 0;
            }
        };
        
        backtrack();
        return result;
    }
};
```

### 复杂度分析

| 复杂度 | 值 |
|--------|-----|
| 时间复杂度 | $O(N \times N!)$ |
| 空间复杂度 | $O(N)$ |

全排列共有 $N!$ 种，每种排列需要 $O(N)$ 时间构造。

## 解法二：库函数（简化写法）

C++ `std::next_permutation` 可以直接生成全排列：

```cpp
class Solution {
public:
    vector<vector<int>> permute(vector<int>& nums) {
        vector<vector<int>> result;
        sort(nums.begin(), nums.end());  // 必须先排序
        
        do {
            result.push_back(nums);
        } while (next_permutation(nums.begin(), nums.end()));
        
        return result;
    }
};
```

## 关键点

1. **回溯的三要素**：
   - 选择：从未使用的数字中选一个
   - 约束：数字不能重复使用
   - 目标：收集所有 $N!$ 种排列

2. **状态恢复**：递归返回后必须 `撤销选择`，即将 `used[i]` 置回 `0`

3. **剪枝**：通过 `used` 数组可以在 $O(1)$ 时间内判断数字是否可用

## 相关题目

| 题目 | 难度 | 说明 |
|------|------|------|
| [[leetcode47|47. 全排列 II]] | 中等 | 含重复数字的全排列 |
| [[leetcode77|77. 组合]] | 中等 | 给定范围数字的组合 |
| [[leetcode78|78. 子集]] | 中等 | 所有可能的子集 |

## 参考

- [LeetCode 46 - 全排列](https://leetcode.cn/problems/permutations/)
- [[../algorithms/backtracking|回溯法]]
