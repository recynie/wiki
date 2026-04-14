---
title: 每日温度 (LeetCode 739)
date: 2026-04-11
description: 使用单调栈求每个温度后面第一个更高温度出现的距离
tags:
  - monotonic-stack
  - stack
draft: false
permalink:
---

## 每日温度 (LeetCode 739)

给定一个整数数组 `temperatures` 表示每日的温度，返回一个数组 `answer`，其中 `answer[i]` 表示对于第 `i` 天，需要等待多少天才能等到更高的温度。如果之后都没有更高的温度，则 `answer[i] = 0`。

### 算法思路：单调递减栈

维护一个存储索引的**单调递减栈**（栈中元素对应的温度依次递减）：

- 遍历温度数组，对于每个位置 `i`
- 若栈不为空且 `temperatures[i] > temperatures[st.top()]`，说明找到了栈顶日期之后的第一个更高温度
- 弹出栈顶，计算等待天数：`i - st.top()`
- 重复直至栈空或温度不满足条件
- 将当前索引 `i` 入栈

### 核心思想

单调栈用于解决"下一个更大/更小元素"问题：

- **递减栈**：找下一个更大元素
- **递增栈**：找下一个更小元素

### C++ 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

class Solution {
public:
    vector<int> dailyTemperatures(vector<int>& temperatures) {
        int n = temperatures.size();
        vector<int> answer(n, 0);
        stack<int> st;  // 存储索引，栈中温度单调递减
        
        for (int i = 0; i < n; ++i) {
            // 若当前温度高于栈顶日期的温度，找到栈顶日期的答案
            while (!st.empty() && temperatures[i] > temperatures[st.top()]) {
                int prevDay = st.top();
                st.pop();
                answer[prevDay] = i - prevDay;
            }
            st.push(i);
        }
        
        return answer;
    }
};
```

### 执行流程示例

```
temperatures = [73, 74, 75, 71, 69, 72, 76, 73]

i=0: st=[0], answer=[0,0,0,0,0,0,0,0]
i=1: 74>73, pop 0, answer[0]=1; st=[1]
i=2: 75>74, pop 1, answer[1]=1; st=[2]
i=3: 71<75, push; st=[2,3]
i=4: 69<71, push; st=[2,3,4]
i=5: 72>69(pop),72>71(pop),72<75(push); st=[2,5], answer[4]=1,answer[3]=1
i=6: 76>72(pop),76>75(pop); st=[6], answer[5]=1,answer[2]=4
i=7: 73<76, push; st=[6,7]
最终: answer = [1,1,4,2,1,1,0,0]
```

### 复杂度分析

- **时间复杂度**：$O(n)$，每个元素最多入栈和出栈各一次
- **空间复杂度**：$O(n)$，最坏情况下栈存储所有索引

### 参考

- [[../data-structure/monotonous-stack|单调栈]]
