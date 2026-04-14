---
title: 无重复字符的最长子串 (LeetCode 3)
date: 2026-04-11
description: 使用滑动窗口求无重复字符的最长子串
tags:
  - sliding-window
  - string
  - hash-map
draft: false
permalink:
---

## 无重复字符的最长子串 (LeetCode 3)

给定一个字符串 `s`，找出其中不含有重复字符的**最长子串**的长度。

### 算法思路：滑动窗口

使用双指针维护一个滑动窗口 $[left, right)$，其中保证窗口内没有重复字符：

1. **扩展右边界**：`right` 指针不断右移，将字符加入窗口
2. **收缩左边界**：当出现重复字符时，将 `left` 指针右移直至窗口内无重复
3. **更新答案**：每次扩展后记录窗口长度

### 哈希表实现

使用 `unordered_map<char, int>` 记录每个字符最后一次出现的位置：

- 当遇到重复字符时，更新 `left = max(left, lastPos + 1)`
- 每次更新字符位置，并计算最大长度

### C++ 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

class Solution {
public:
    int lengthOfLongestSubstring(string s) {
        unordered_map<char, int> lastPos;
        int left = 0, maxLen = 0;
        
        for (int right = 0; right < s.size(); ++right) {
            char c = s[right];
            // 若字符已存在，更新左边界（防止左边界回退）
            if (lastPos.count(c)) {
                left = max(left, lastPos[c] + 1);
            }
            lastPos[c] = right;
            maxLen = max(maxLen, right - left + 1);
        }
        return maxLen;
    }
};
```

### 优化：数组代替哈希表

由于字符 ASCII 码范围有限（0-127），可用数组代替 unordered_map，提升效率：

```cpp
class Solution {
public:
    int lengthOfLongestSubstring(string s) {
        vector<int> lastPos(128, -1);
        int left = 0, maxLen = 0;
        
        for (int right = 0; right < s.size(); ++right) {
            char c = s[right];
            if (lastPos[c] >= left) {
                left = lastPos[c] + 1;
            }
            lastPos[c] = right;
            maxLen = max(maxLen, right - left + 1);
        }
        return maxLen;
    }
};
```

### 复杂度分析

- **时间复杂度**：$O(n)$，每个字符最多被访问两次（左右指针各一次）
- **空间复杂度**：$O(1)$ 或 $O(n)$（字符集大小固定或使用哈希表）

### 参考

- [[sliding-window|滑动窗口技巧]]
