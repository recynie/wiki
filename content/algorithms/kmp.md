---
title: KMP算法
date: 2026-03-31
description: KMP字符串匹配算法
tags:
  - string
  - kmp
  - pattern-matching
draft: false
permalink:
---

## 简介

KMP（Knuth-Morris-Pratt）算法是一种高效的字符串匹配算法，由 Donald Knuth、Vaughan Pratt 和 James Morris 于1977年联合提出。[^1]

传统的暴力匹配算法在匹配失败时，会将模式串回溯到开头，文本串回溯到上一次匹配位置的下一个字符，导致大量重复比较。KMP算法的核心思想是：**利用已经匹配的信息，避免回溯已经比较过的字符**。

## 核心性质

### 前缀函数（π函数）

前缀函数 $\pi[i]$ 定义为：模式串 $P[0...i]$ 中，既是严格真前缀又是严格真后缀的最长字符串长度。

例如，对于模式串 `ABAB`：

| $i$ | $P[0...i]$ | 前缀集合 | 后缀集合 | $\pi[i]$ |
|-----|------------|----------|----------|----------|
| 0   | A          | {}       | {}       | 0        |
| 1   | AB         | {A}      | {B}      | 0        |
| 2   | ABA        | {A, AB}  | {A, BA}  | 1        |
| 3   | ABAB       | {A, AB, ABA} | {B, AB, BAB} | 2    |

### 匹配过程

设文本串为 $T$，模式串为 $P$。当匹配到 $T[i]$ 和 $P[j]$ 发生失配时（即 $T[i] \neq P[j]$），可以利用前缀函数将模式串向右滑动 $\pi[j-1]$ 个位置，继续比较 $T[i]$ 和 $P[\pi[j-1]]$。

这使得KMP的时间复杂度达到 $O(n+m)$，其中 $n = |T|$，$m = |P|$。

## 算法步骤

### 构建前缀函数

1. 初始化 $\pi[0] = 0$
2. 遍历 $i = 1$ 到 $m-1$：
   - 令 $j = \pi[i-1]$
   - 当 $j > 0$ 且 $P[i] \neq P[j]$ 时，$j = \pi[j-1]$
   - 若 $P[i] == P[j]$，则 $j++$
   - 令 $\pi[i] = j$

### KMP匹配

1. 初始化 $j = 0$（当前已匹配的模式串长度）
2. 遍历 $i = 0$ 到 $n-1$：
   - 当 $j > 0$ 且 $T[i] \neq P[j]$ 时，$j = \pi[j-1]$
   - 若 $T[i] == P[j]$，则 $j++$
   - 若 $j == m$，说明找到一个匹配，起始位置为 $i - m + 1$

## 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

// 构建前缀函数
vector<int> buildPrefix(const string& P) {
    int m = P.length();
    vector<int> pi(m, 0);
    for (int i = 1; i < m; ++i) {
        int j = pi[i - 1];
        while (j > 0 && P[i] != P[j]) {
            j = pi[j - 1];
        }
        if (P[i] == P[j]) {
            ++j;
        }
        pi[i] = j;
    }
    return pi;
}

// KMP匹配
vector<int> kmp(const string& T, const string& P) {
    int n = T.length(), m = P.length();
    vector<int> pi = buildPrefix(P);
    vector<int> matches;
    int j = 0;  // 已匹配的模式串长度
    for (int i = 0; i < n; ++i) {
        while (j > 0 && T[i] != P[j]) {
            j = pi[j - 1];
        }
        if (T[i] == P[j]) {
            ++j;
        }
        if (j == m) {
            matches.push_back(i - m + 1);
            j = pi[j - 1];
        }
    }
    return matches;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    string T, P;
    cin >> T >> P;
    vector<int> pos = kmp(T, P);
    for (int idx : pos) {
        cout << idx << " ";
    }
    return 0;
}
```

## 应用场景

- **字符串搜索**：在文本中查找模式串的所有出现位置
- **字符串去重**：利用前缀函数判断字符串是否由重复子串构成
- **最长公共前缀**：结合其他数据结构用于字符串比较
- **正则表达式实现**：KMP是许多正则表达式引擎的核心组件

## 参考资料

[^1]: 本词条内容来自 [OI Wiki - KMP算法](https://oi-wiki.org/string/kmp/)
