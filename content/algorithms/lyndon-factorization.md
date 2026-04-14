---
title: Lyndon分解
date: 2026-04-06
description: Lyndon分解——将字符串唯一分解为若干个字典序最小的后缀
tags:
  - string-algorithm
  - suffix
draft: false
permalink:
---

## 定义

给定字符串 $s$，其 **Lyndon 分解**（Lyndon Factorization）是将 $s$ 唯一地分解为若干个子串：

$$
s = w_1 w_2 \dots w_k
$$

其中每个 $w_i$ 都是 **Lyndon 词**，且满足 $w_1 \geq w_2 \geq \dots \geq w_k$（字典序递减）。[^1]

### Lyndon 词的定义

一个字符串 $t$ 是 Lyndon 词，当且仅当：
1. $t$ 是其自身的一个真周期
2. 或者等价地，$t$ 的字典序严格小于其所有真后缀的字典序

形式化地：

$$
t \prec t[i..] \quad \text{对所有 } 1 < i \leq |t|
$$

其中 $\prec$ 表示字典序小于。

### 例子

| 字符串 | Lyndon 分解 |
|--------|------------|
| `banana` | `b a n a n a` |
| `aababc` | `aa b ab c` |
| `aaaa` | `a a a a` |
| `abcabc` | `abc abc` |

---

## Duval 算法

Lyndon 分解可以在 $O(n)$ 时间内通过 **Duval 算法** 计算。[^2]

### 算法思想

维护三个指针：
- $i$：当前处理位置的起点
- $j$：候选 Lyndon 词内部的比较位置
- $k$：Lyndon 词的长度边界

### 伪代码

```
function lyndon(s):
    n = |s|
    result = []
    i = 0
    while i < n:
        j = i + 1
        k = i
        while j < n and s[k] <= s[j]:
            if s[k] < s[j]:
                k = i
            else:
                k = k + 1
            j = j + 1
        
        // 当前 Lyndon 词长度
        len = j - k
        while i <= k:
            result.append(s[i : i + len])
            i = i + len
    return result
```

### C++ 实现

```cpp
#include <bits/stdc++.h>
using namespace std;

vector<string> lyndonFactorization(const string& s) {
    int n = s.size();
    vector<string> result;
    int i = 0;
    
    while (i < n) {
        int j = i + 1;
        int k = i;
        
        while (j < n && s[k] <= s[j]) {
            if (s[k] < s[j]) {
                k = i;
            } else {
                k++;
            }
            j++;
        }
        
        int len = j - k;
        while (i <= k) {
            result.push_back(s.substr(i, len));
            i += len;
        }
    }
    
    return result;
}
```

### 运行示例

以 `s = "aababc"` 为例：

```
初始：i = 0
j = 1, k = 0: s[0]='a' <= s[1]='a', s[0]=s[1] → k=1
j = 2, k = 1: s[1]='a' <= s[2]='b', s[1]<s[2] → k=0
j = 3, k = 0: s[0]='a' <= s[3]='a', s[0]=s[3] → k=1
j = 4, k = 1: s[1]='a' <= s[4]='b', s[1]<s[4] → k=0
j = 5, k = 0: s[0]='a' <= s[5]='c', s[0]<s[5] → k=0
退出循环，len = 5 - 0 = 5

添加 s[0:5] = "aabab"
i = 5

继续处理剩余字符...
```

---

## 性质

### 唯一性

Lyndon 分解是唯一的——每个字符串只有一种分解方式。

### 与 Z 函数的关系

Lyndon 分解与 [[../algorithms/z-function|Z 函数]] 有密切关系：

- 如果 $s[i..i+len-1]$ 是 Lyndon 词，则 $Z[i] \geq len - 1$

### 与后缀数组的关系

每个 Lyndon 分解中的子串都是其对应后缀的最小前缀。

---

## 经典应用

### 1. 求字符串的最小表示

通过 Lyndon 分解可以 $O(n)$ 求字符串的最小表示：

```cpp
string minRotation(const string& s) {
    string ss = s + s;
    auto factors = lyndonFactorization(ss);
    
    // 找到第一个分解点
    int n = s.size();
    int pos = 0;
    for (const auto& f : factors) {
        if (pos + f.size() >= n) break;
        pos += f.size();
    }
    
    return ss.substr(pos, n);
}
```

### 2. 求字符串的最小周期

```cpp
int minPeriod(const string& s) {
    int n = s.size();
    auto factors = lyndonFactorization(s);
    
    int total = 0;
    for (const auto& f : factors) total += f.size();
    
    return n - total + factors.back().size();
}
```

### 3. 在文本中查找模式（结合其他算法）

Lyndon 分解可用于构造后缀数组或解决重复子串问题。

---

## 例题

### 例题：验证 Lyndon 分解

#### 问题

给定字符串 $s$ 和一种划分，验证这是否是 $s$ 的有效 Lyndon 分解。

#### 验证条件

1. 每个子串 $w_i$ 是 Lyndon 词
2. $w_1 \geq w_2 \geq \dots \geq w_k$（字典序递减）

#### 验证算法

```cpp
bool isLyndonWord(const string& t) {
    int n = t.size();
    for (int i = 1; i < n; i++) {
        if (t.substr(i) < t) return false;
    }
    return true;
}

bool verifyLyndonFactorization(const string& s, const vector<string>& factors) {
    string reconstructed;
    for (const auto& f : factors) {
        if (!isLyndonWord(f)) return false;
        reconstructed += f;
    }
    if (reconstructed != s) return false;
    
    for (size_t i = 1; i < factors.size(); i++) {
        if (factors[i-1] < factors[i]) return false;
    }
    
    return true;
}
```

---

## 与其他字符串算法的比较

| 算法 | 时间复杂度 | 用途 |
|------|-----------|------|
| Lyndon 分解 | $O(n)$ | 字符串唯一分解 |
| Z 函数 | $O(n)$ | 字符串匹配 |
| KMP | $O(n)$ | 模式匹配 |
| 后缀数组 | $O(n \log n)$ | 字符串排序 |

---

## 参考资料

[^1]: Lyndon, R. C. (1954). On Burnside's problem. Transactions of the American Mathematical Society, 77(2), 202-215.

[^2]: Duval, J. P. (1983). Factorizing words over an ordered alphabet. Journal of Algorithms, 4(4), 363-381.

[^3]: Lyndon Factorization - Wikipedia. https://en.wikipedia.org/wiki/Lyndon_factorization

[^4]: Lyndon 分解 - OI Wiki. https://oi-wiki.org/string/lyndon/
