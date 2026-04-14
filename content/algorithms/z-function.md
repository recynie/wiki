---
title: Z函数（扩展KMP）
date: 2026-04-03
description: Z 函数（Z-function）算法详解，又称扩展 KMP
tags:
  - string
  - z-function
  - kmp
draft: false
permalink:
---

## 定义

**Z 函数**（Z-function），又称 **Z 算法**，用于计算字符串 s 的每个位置 i 到字符串末尾的子串与整个字符串的最长公共前缀长度。

对于字符串 s，长度为 n，Z 函数定义为：

$$
Z[i] = \max\{l \mid s[i \dots i+l-1] = s[0 \dots l-1]\}, \quad i \in [1, n-1]
$$

即 $Z[i]$ 表示从位置 i 开始、与 s 的前缀匹配的最长子串长度。

规定 $Z[0] = 0$（或 n）。

## 朴素算法

对每个位置 i，朴素算法直接比较 s[i] 与 s[0]，时间复杂度为 $O(n^2)$。

## Z 算法

Z 算法的核心思想是**维护已匹配的区间 $[l, r]$**（最靠右的有效匹配段）。

### 算法过程

1. 初始化 $l = r = 0$
2. 对于每个位置 i：
   - 如果 $i > r$，使用朴素算法扩展，重新设置 $[l, r]$
   - 否则，利用之前计算的结果：$Z[i] = \min(r - i + 1, Z[i - l])$

### 代码实现

```cpp
vector<int> z_function(const string& s) {
    int n = s.size();
    vector<int> z(n);
    int l = 0, r = 0;
    
    for (int i = 1; i < n; i++) {
        if (i > r) {
            // i 在当前维护区间外，使用朴素算法
            l = r = i;
            while (r < n && s[r - l] == s[r]) {
                r++;
            }
            z[i] = r - l;
            r--;  // 修正：r 指向右边界
        } else {
            // i 在区间内，可以利用之前的结果
            int k = i - l;
            // Z[i - l] 可能超出区间，需要取 min
            z[i] = min(z[k], max(0, r - i + 1));
            // 继续扩展
            while (r + 1 < n && s[z[i]] == s[i + z[i]]) {
                z[i]++;
            }
            // 更新区间
            if (i + z[i] - 1 > r) {
                l = i;
                r = i + z[i] - 1;
            }
        }
    }
    
    return z;
}
```

### 简化版本

```cpp
vector<int> z_function(const string& s) {
    int n = s.size();
    vector<int> z(n);
    int l = 0, r = 0;
    
    for (int i = 1; i < n; i++) {
        if (i <= r) {
            z[i] = min(r - i + 1, z[i - l]);
        }
        while (i + z[i] < n && s[z[i]] == s[i + z[i]]) {
            z[i]++;
        }
        if (i + z[i] - 1 > r) {
            l = i;
            r = i + z[i] - 1;
        }
    }
    
    return z;
}
```

## 复杂度分析

- **时间复杂度**：$O(n)$
  - 每次 $r$ 只会向右移动，总共最多移动 $n$ 次
  - 每次比较要么使 $r$ 增加，要么使 $i$ 增加

- **空间复杂度**：$O(n)$

## 应用场景

### 1. 字符串匹配

给定文本 T 和模式 P，求 P 在 T 中的所有出现位置。

**方法**：构造字符串 $P + \# + T$，计算 Z 函数，找到 Z 值等于 $|P|$ 的位置。

```cpp
vector<int> match(const string& text, const string& pattern) {
    string s = pattern + "#" + text;
    vector<int> z = z_function(s);
    vector<int> pos;
    int m = pattern.size();
    
    for (int i = m + 1; i < s.size(); i++) {
        if (z[i] == m) {
            pos.push_back(i - m - 1);  // 文本中的起始位置
        }
    }
    return pos;
}
```

### 2. 最长公共前缀查询

Z 函数可以快速回答"第 i 个后缀和第 j 个后缀的最长公共前缀是多少"这类问题。

### 3. 字符串压缩

用于检测字符串中的重复模式。

## Z 函数与 KMP 的关系

KMP 的 next 数组可以通过 Z 函数计算：

$$
next[i] = Z[i] \text{ 对于处理后的字符串}
$$

但 KMP 和 Z 函数的侧重点不同：
- **KMP**：适用于单模式匹配，关注前缀函数
- **Z 函数**：适用于多后缀匹配，关注与整个字符串的匹配

## 相关主题

- [[../algorithms/kmp|KMP 算法]]：前缀函数的应用
- [[../algorithms/ac-automaton|AC 自动机]]：多模式匹配
- [[../algorithms/manacher|Manacher 算法]]：回文串问题

## 参考资料

[^1]: [Z 函数（扩展 KMP）- OI Wiki](https://oi-wiki.org/string/z-func/)
