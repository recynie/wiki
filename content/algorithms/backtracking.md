---
title: 回溯法
date: 2026-04-06
description: 回溯法（Backtracking）是一种通过枚举所有可能来求解问题的算法思想
tags:
  - backtracking
  - dfs
  - algorithm
draft: false
permalink:
---

# 回溯法

回溯法（Backtracking）是一种基于深度优先搜索的算法思想，通过**枚举所有可能**来求解问题。回溯法在搜索过程中会「尝试」做出选择，如果发现当前选择无法到达目标解，就「撤销」选择并尝试其他选项。[^1]

[^1]: 本段参考[回溯 - OI Wiki](https://oi-wiki.org/search/dfs/)

## 核心思想

回溯法的核心三要素：

| 要素 | 含义 |
|------|------|
| **选择（Choice）** | 在每一步可以做出的决策 |
| **约束（Constraint）** | 哪些选择是合法的 |
| **目标（Goal）** | 如何判断找到了解 |

回溯法本质上是一种**暴力搜索**，但通过剪枝可以跳过大量无效搜索空间。

## 算法模板

```cpp
void backtrack(参数) {
    if (满足结束条件) {
        记录解;
        return;
    }
    
    for (选择 in 当前可以做出的选择) {
        if (if 满足约束条件) {
            做选择;
            backtrack(下一层);
            撤销选择;  // 回溯
        }
    }
}
```

## 经典问题

### 1. 全排列

给定一个不含重复数字的数组，返回其所有可能的排列。

```cpp
class Solution {
public:
    vector<vector<int>> permute(vector<int>& nums) {
        vector<vector<int>> result;
        vector<int> path;
        vector<int> used(nums.size(), 0);
        
        function<void()> backtrack = [&]() {
            if (path.size() == nums.size()) {
                result.push_back(path);
                return;
            }
            
            for (int i = 0; i < nums.size(); ++i) {
                if (used[i]) continue;
                
                path.push_back(nums[i]);
                used[i] = 1;
                backtrack();
                path.pop_back();
                used[i] = 0;
            }
        };
        
        backtrack();
        return result;
    }
};
```

详细题解见：[[../Leetcode/leetcode46|LeetCode 46 - 全排列]]

### 2. N皇后问题

在 $N \times N$ 的棋盘上放置 $N$ 个皇后，使得它们互不攻击。

```cpp
class Solution {
public:
    vector<vector<string>> solveNQueens(int n) {
        vector<vector<string>> result;
        vector<string> board(n, string(n, '.'));
        vector<int> diag1(2*n, 0), diag2(2*n, 0), cols(n, 0);
        
        function<void(int)> backtrack = [&](int row) {
            if (row == n) {
                result.push_back(board);
                return;
            }
            
            for (int col = 0; col < n; ++col) {
                int d1 = row - col + n, d2 = row + col;
                if (cols[col] || diag1[d1] || diag2[d2]) continue;
                
                board[row][col] = 'Q';
                cols[col] = diag1[d1] = diag2[d2] = 1;
                backtrack(row + 1);
                board[row][col] = '.';
                cols[col] = diag1[d1] = diag2[d2] = 0;
            }
        };
        
        backtrack(0);
        return result;
    }
};
```

### 3. 子集问题

给定一个整数数组，返回所有可能的子集。

```cpp
vector<vector<int>> subsets(vector<int>& nums) {
    vector<vector<int>> result;
    vector<int> path;
    
    function<void(int)> backtrack = [&](int start) {
        result.push_back(path);
        
        for (int i = start; i < nums.size(); ++i) {
            path.push_back(nums[i]);
            backtrack(i + 1);
            path.pop_back();
        }
    };
    
    backtrack(0);
    return result;
}
```

## 剪枝优化

剪枝（Pruning）是提升回溯效率的关键技术。

### 1. 约束剪枝

在选择时提前判断是否满足约束条件：

```cpp
// 组合总和：找到所有和为target的组合
void backtrack(int start, int target) {
    if (target == 0) {
        result.push_back(path);
        return;
    }
    
    for (int i = start; i < candidates.size(); ++i) {
        // 剪枝：如果当前元素大于目标值，跳过
        if (candidates[i] > target) continue;
        
        path.push_back(candidates[i]);
        backtrack(i, target - candidates[i]);
        path.pop_back();
    }
}
```

### 2. 排序剪枝

对候选数组排序后，可以更早地发现无效分支：

```cpp
// 排序后剪枝：跳过重复元素
sort(candidates.begin(), candidates.end());
for (int i = start; i < n; ++i) {
    if (i > start && candidates[i] == candidates[i-1]) continue;
    // ...
}
```

## 回溯与递归的关系

回溯本质上是递归的一种应用场景：

| 特征 | 递归 | 回溯 |
|------|------|------|
| 终止条件 | 必须有 | 必须有 |
| 状态恢复 | 不需要 | 需要（撤销选择） |
| 搜索空间 | 子问题树 | 解空间树 |
| 典型应用 | 树遍历 | 组合、排列、搜索 |

## 复杂度分析

回溯法的时间复杂度通常为 $O(N!)$（排列问题），通过剪枝可以显著降低实际搜索量。

空间复杂度为 $O(N)$，主要用于递归栈和路径存储。

## 常见应用

- **排列组合问题**：全排列、组合、子集
- **棋盘问题**：N皇后、数独
- **分割问题**：复原IP地址、分割回文串
- **图论问题**： Hamiltonian路径、图的着色
- **字符串问题**：括号生成、单词搜索

## 参考

- [回溯 - OI Wiki](https://oi-wiki.org/search/dfs/)
- [LeetCode 46 - 全排列](https://leetcode.cn/problems/permutations/)
