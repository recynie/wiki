---
title: Manacher算法
date: 2026-04-03
description: Manacher 算法 $O(n)$ 时间复杂度求最长回文子串
tags:
  - string
  - palindrome
  - manacher
draft: false
permalink:
---

## 定义

**回文串**：正向读和反向读完全相同的字符串，例如 "aba"、"abba"。

**Manacher 算法** 由 Glenn K. Manacher 于 1975 年提出，能在 $O(n)$ 时间内求出字符串中所有回文子串的信息。

## 核心概念

对于字符串 s 的每个位置 i，我们计算：

- $d_1[i]$：以 i 为中心的**奇数长度**回文串个数（即半径长度）
- $d_2[i]$：以 i 为中心的**偶数长度**回文串个数

$d_1[i]$ 的含义是以 i 为中心、最长回文串的半径（包含字符个数）。类似地，$d_2[i]$ 表示以 i 和 i-1 为中心的偶数长度回文串半径。

## 算法思想

维护已找到的**最靠右**的回文串边界 $[l, r]$（即具有最大 r 值的回文串）。

### 状态转移

对于位置 i：

1. **i > r**：使用朴素算法继续扩展
2. **i ≤ r**：利用对称性，令 $j = l + (r - i)$，则 $d_1[i]$ 至少等于 $d_1[j]$

### 边界情况

当以 j 为中心的回文串超出外部回文串边界时，不能直接复制，需要截断：

$$
d_1[i] = \min(d_1[j], r - i + 1)
$$

然后再用朴素算法尝试继续扩展。

## 代码实现

### 计算奇数长度回文串 $d_1$

```cpp
vector<int> d1(n);
for (int i = 0, l = 0, r = -1; i < n; i++) {
    int k = (i > r) ? 1 : min(d1[l + r - i], r - i + 1);
    while (0 <= i - k && i + k < n && s[i - k] == s[i + k]) {
        k++;
    }
    d1[i] = k--;
    if (i + k > r) {
        l = i - k;
        r = i + k;
    }
}
```

### 计算偶数长度回文串 $d_2$

```cpp
vector<int> d2(n);
for (int i = 0, l = 0, r = -1; i < n; i++) {
    int k = (i > r) ? 0 : min(d2[l + r - i + 1], r - i + 1);
    while (i - k - 1 >= 0 && i + k < n && s[i - k - 1] == s[i + k]) {
        k++;
    }
    d2[i] = k--;
    if (i + k > r) {
        l = i - k - 1;
        r = i + k;
    }
}
```

### 统一版本

为简化代码，可以先将字符串处理成奇数长度形式：

```cpp
// 在每两个字符之间插入 '#'
string t = "#";
for (char c : s) {
    t += c;
    t += '#';
}
```

这样只需要计算 $d_1$ 数组，原字符串位置 i 的回文长度就是 $d_1[i+1] - 1$。

## 复杂度分析

- **时间复杂度**：$O(n)$
  - 朴素算法每次迭代都会使 r 增加 1
  - r 在算法运行过程中从不减小
  - 因此朴素算法总共只进行 $O(n)$ 次迭代

- **空间复杂度**：$O(n)$

## 完整示例

```cpp
#include <bits/stdc++.h>
using namespace std;

// Manacher 算法求最长回文子串
pair<int, int> longest_palindrome(const string& s) {
    int n = s.size();
    vector<int> d1(n), d2(n);
    
    // 奇数长度
    for (int i = 0, l = 0, r = -1; i < n; i++) {
        int k = (i > r) ? 1 : min(d1[l + r - i], r - i + 1);
        while (i - k >= 0 && i + k < n && s[i - k] == s[i + k]) k++;
        d1[i] = k--;
        if (i + k > r) {
            l = i - k;
            r = i + k;
        }
    }
    
    // 偶数长度
    for (int i = 0, l = 0, r = -1; i < n; i++) {
        int k = (i > r) ? 0 : min(d2[l + r - i + 1], r - i + 1);
        while (i - k - 1 >= 0 && i + k < n && s[i - k - 1] == s[i + k]) k++;
        d2[i] = k--;
        if (i + k > r) {
            l = i - k - 1;
            r = i + k;
        }
    }
    
    // 找最长回文
    int max_len = 1, pos = 0;
    for (int i = 0; i < n; i++) {
        if (d1[i] * 2 - 1 > max_len) {
            max_len = d1[i] * 2 - 1;
            pos = i;
        }
        if (d2[i] * 2 > max_len) {
            max_len = d2[i] * 2;
            pos = i;
        }
    }
    
    int start = pos - (max_len - 1) / 2;
    return {start, max_len};
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    string s;
    cin >> s;
    auto [start, len] = longest_palindrome(s);
    cout << s.substr(start, len) << endl;
    cout << "Length: " << len << endl;
    
    return 0;
}
```

## 应用场景

- **最长回文子串**：直接应用
- **回文子串计数**：$\sum d_1[i] + \sum d_2[i]$
- **不同回文串个数**：可用后缀数组或哈希进一步处理
- **构造回文**：通过已知回文结构进行字符串构造

## 与其他算法的对比

| 算法 | 时间复杂度 | 适用场景 |
|------|-----------|----------|
| 朴素算法 | $O(n^2)$ | 小规模数据 |
| 字符串哈希 + 二分 | $O(n \log n)$ | 需要多次查询 |
| Manacher | $O(n)$ | 求所有回文信息 |

## 相关主题

- [[../algorithms/kmp|KMP 算法]]：字符串匹配
- [[../algorithms/string-hash|字符串哈希]]：可用于回文判断
- [[../algorithms/z-function|Z 函数]]：类似思想的字符串算法

## 参考资料

[^1]: [Manacher - OI Wiki](https://oi-wiki.org/string/manacher/)
