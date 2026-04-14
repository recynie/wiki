---
title: 柱状图中最大的矩形 (LeetCode 84)
date: 2026-03-12
description: 使用单调栈解决柱状图中最大矩形面积问题
tags:
  - monotonous-stack
  - stack
draft: false
permalink:
---

## 柱状图中最大的矩形 (LeetCode 84)

给定 `n` 个非负整数表示柱状图的柱高，求该柱状图中能够勾勒出的最大矩形面积。

### 算法推导

对于任意柱子 $h[i]$，若以其高度作为最终矩形的高度，则该矩形的最大宽度取决于其向左和向右延伸的极限：

- **左边界**：左侧第一个高度**严格小于** $h[i]$ 的柱子所在位置
- **右边界**：右侧第一个高度**严格小于** $h[i]$ 的柱子所在位置

因此，问题的核心转化为：**求解每个元素左侧和右侧第一个比它小的元素**，需要使用**单调递增栈**。

### 边界处理与虚拟节点

若数组本身单调递增，栈内元素无法被触发弹出并计算面积；若最低的柱子一直保留在栈中，其左边界也无法正确判定。需要在原高度数组的首尾各添加一个高度为 `0` 的虚拟节点：

- **首部加 `0`**：防止当弹出栈顶元素后，栈变为空而无法确定左边界的情况
- **尾部加 `0`**：强制使得遍历到末尾时，所有原数组内的非零元素都能被破坏单调性，从而触发出栈与面积计算

### 面积计算公式

当遍历到 $i$ 且 $h[i] < h[st.top()]$ 时，触发计算：

1. 取出栈顶元素索引 `mid = st.top()`，此时 `h[mid]` 即为矩形的高度
2. 弹出 `mid` 后，新的栈顶元素 `left = st.top()` 即为 `mid` 的左边界
3. 当前元素 `i` 即为 `mid` 的右边界
4. 宽度计算：`w = i - left - 1`
5. 面积更新：`area = h[mid] * w`

### C++ 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

class Solution {
public:
    int largestRectangleArea(vector<int>& heights) {
        // 首尾添加虚拟高度 0
        vector<int> h(heights.size() + 2, 0);
        for (int i = 0; i < heights.size(); ++i) {
            h[i + 1] = heights[i];
        }
        
        stack<int> st;
        int max_area = 0;
        
        for (int i = 0; i < h.size(); ++i) {
            // 维护单调递增栈
            while (!st.empty() && h[i] < h[st.top()]) {
                int mid = st.top();
                st.pop();
                
                if (!st.empty()) {
                    int left = st.top();
                    int right = i;
                    int width = right - left - 1;
                    int area = h[mid] * width;
                    max_area = max(max_area, area);
                }
            }
            st.push(i);
        }
        return max_area;
    }
};
```

### 复杂度分析

- **时间复杂度**：$O(n)$，每个元素最多入栈和出栈各一次
- **空间复杂度**：$O(n)$，最坏情况下栈需要存储所有元素索引

### 参考

- [[../data-structure/monotonous-stack|单调栈]]
