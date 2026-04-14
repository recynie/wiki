---
title: 后缀数组
date: 2026-04-05
description: 后缀数组（Suffix Array）详解：定义、构造算法（倍增法）、LCP数组与Kasai算法，以及子串查找、不同子串计数等应用
tags:
  - string
  - suffix-array
draft: false
permalink:
---

## 定义

**后缀数组**（Suffix Array）是对一个字符串的所有后缀按字典序排序后形成的数组。设字符串 $S$ 长度为 $n$，则后缀数组 $SA$ 是一个长度为 $n$ 的排列，满足：

$$
SA[i] < SA[i+1] \text{ 当且仅当 } S[SA[i]..n] < S[SA[i+1]..n]
$$

其中 $S[l..r]$ 表示字符串 $S$ 从位置 $l$ 到 $r$ 的子串（包含两端）。

### 示例

以字符串 $S = \text{"abaab"}$ 为例：

| 后缀 | 起始位置 | 后缀内容 |
|------|----------|----------|
| $S[0..]$ | 0 | "abaab" |
| $S[1..]$ | 1 | "baab" |
| $S[2..]$ | 2 | "aab" |
| $S[3..]$ | 3 | "ab" |
| $S[4..]$ | 4 | "b" |

按字典序排序后：

| 排名 | 起始位置 | 后缀内容 |
|------|----------|----------|
| 0 | 2 | "aab" |
| 1 | 3 | "ab" |
| 2 | 0 | "abaab" |
| 3 | 4 | "b" |
| 4 | 1 | "baab" |

因此该字符串的后缀数组为 $SA = [2, 3, 0, 4, 1]$。

后缀数组是处理字符串问题的有力工具，常与 [[../algorithms/kmp|KMP算法]] 和 [[../algorithms/string-hash|字符串哈希]] 结合使用。

## 构造算法

### 朴素方法 $O(n^2 \log n)$

最直接的做法是生成所有后缀，对它们直接排序。每个后缀长度为 $O(n)$，比较两个后缀需要 $O(n)$ 时间，排序有 $O(n \log n)$ 次比较，因此总时间复杂度为 $O(n^2 \log n)$。这种方法在小规模数据外不可接受。

### 倍增法 $O(n \log n)$

倍增法（Doubling Method）是构造后缀数组最经典的算法，其核心思想是利用已经计算过的信息来加速排序。

#### 算法思想

对于每个位置 $i$，我们同时考虑以 $i$ 开头、长度分别为 $2^0, 2^1, 2^2, \ldots$ 的前缀。设 $k$ 为当前考虑的前缀长度，初始时 $k = 1$。

对每个位置 $i$，定义一个**二元组**：

$$
(第一关键字, 第二关键字) = (S[i], S[i+k]) \quad \text{（超出范围则为 -1）}
$$

当 $k$ 较小时，我们可以使用基数排序对这个二元组进行排序。随着 $k$ 不断倍增，当 $k \geq n$ 时，所有后缀都可以完全区分，此时即得到完整的后缀数组。

#### 算法步骤

设字符串 $S$ 长度为 $n$，下标从 $0$ 开始。

1. **初始化**：对每个字符进行排序，得到每个字符的排名；
2. **迭代**：对于 $k = 1, 2, 4, 8, \ldots$ 直到 $k \geq n$：
   - 按照 $(rank[i], rank[i+k])$ 进行基数排序，得到新的排名 $newRank[i]$；
   - 如果所有排名均不同，则结束；
   - 更新 $rank \leftarrow newRank$。

#### 正确性证明（简要）

我们证明算法结束时得到的 $rank$ 即为后缀的最终排名。

**引理**：当 $k$ 足够大时（$k \geq n$），每个后缀 $S[i..n-1]$ 可以唯一确定。

*证明*：此时每个后缀的前缀长度至少为 $n-i$，足以覆盖整个后缀，不同后缀必然不同。∎

**引理**：在第 $k$ 轮迭代中，如果两个后缀 $S[i..]$ 和 $S[j..]$ 在 $k$ 长度内的前缀不同，则 $newRank[i] \neq newRank[j]$。

*证明*：$newRank$ 取决于二元组 $(rank[i], rank[i+k])$。由于 $rank$ 已经区分了所有长度小于 $k$ 的前缀，不同的前缀必然导致至少一个关键字不同，从而二元组不同，排名不同。∎

**定理**：倍增法最终得到的后缀数组是正确的。

*证明*：随着 $k$ 不断倍增，存在某个 $k_0$ 使得所有后缀在长度 $k_0$ 内均可区分。根据引理，此后所有迭代的排名不再变化。由于最终排名与字典序一致（由基数排序保证），故算法正确。∎

#### C++ 实现

```cpp
#include <bits/stdc++.h>
using namespace std;

// 后缀数组倍增法 O(n log n)
vector<int> buildSA(const string& s) {
    int n = s.size();
    vector<int> sa(n), rank(n), tmp(n);
    
    // 初始化：单字符排序
    for (int i = 0; i < n; i++) {
        sa[i] = i;
        rank[i] = s[i];
    }
    
    // 倍增
    for (int k = 1; k < n; k <<= 1) {
        // 按第二关键字排序（使用 tmp 作为临时数组）
        auto cmp = [&](int i, int j) {
            if (rank[i] != rank[j]) return rank[i] < rank[j];
            int ri = i + k < n ? rank[i + k] : -1;
            int rj = j + k < n ? rank[j + k] : -1;
            return ri < rj;
        };
        sort(sa.begin(), sa.end(), cmp);
        
        // 计算新的排名
        tmp[sa[0]] = 0;
        for (int i = 1; i < n; i++) {
            tmp[sa[i]] = tmp[sa[i-1]] + (cmp(sa[i-1], sa[i]) ? 1 : 0);
        }
        for (int i = 0; i < n; i++) rank[i] = tmp[i];
        
        // 如果所有排名均不同，则结束
        if (rank[sa[n-1]] == n - 1) break;
    }
    
    return sa;
}
```

上述实现中，我们使用 `sort` 进行排序，时间复杂度为 $O(n \log^2 n)$。为了达到严格的 $O(n \log n)$，可以使用基数排序代替 `sort`，但实现复杂度稍高。在竞赛中，上述实现通常足够高效。

## LCP数组与Kasai算法

### LCP数组的定义

**最长公共前缀数组**（Longest Common Prefix Array，记为 $LCP$）存储了后缀数组中相邻后缀的最长公共前缀长度：

$$
LCP[i] = \operatorname{lcp}(S[SA[i]..], S[SA[i-1]..]) \quad (i \geq 1)
$$

其中 $\operatorname{lcp}(a, b)$ 表示字符串 $a$ 与 $b$ 的最长公共前缀长度。

### Kasai算法 $O(n)$

Kasai算法可以在 $O(n)$ 时间内由原字符串计算出 $LCP$ 数组，无需显式构建后缀数组。

#### 算法思想

考虑 $LCP[i]$ 与 $LCP[i-1]$ 之间的关系。设 $h = LCP[i]$，则相邻后缀 $S[SA[i]..]$ 与 $S[SA[i-1]..]$ 有长度为 $h$ 的公共前缀。

令 $j = SA[i-1]$，$k = SA[i]$，不妨设 $j < k$（否则交换）。则 $S[j..j+h-1] = S[k..k+h-1]$，且 $S[j+h] \neq S[k+h]$（或已到达字符串末尾）。

考虑后缀 $S[j+1..]$ 与 $S[k+1..]$。它们分别是 $S[j..]$ 与 $S[k..]$ 去掉第一个字符后的结果。关键观察是：这两个后缀的公共前缀长度至少为 $h-1$（除非 $h = 0$）。

这意味着如果我们按原字符串位置从左到右遍历，可以利用已计算的 $LCP$ 值来加速后续计算。

#### C++ 实现

```cpp
#include <bits/stdc++.h>
using namespace std;

// Kasai算法 O(n) 计算LCP数组
vector<int> buildLCP(const string& s, const vector<int>& sa) {
    int n = s.size();
    vector<int> rank(n), lcp(n);
    
    // rank[sa[i]] = i，即后缀i在排序后的排名
    for (int i = 0; i < n; i++) {
        rank[sa[i]] = i;
    }
    
    // h = LCP(i, i-1) 的初始值
    int h = 0;
    for (int i = 0; i < n; i++) {
        int r = rank[i];
        if (r > 0) {
            // 上一轮计算的 h
            int j = sa[r - 1];
            while (i + h < n && j + h < n && s[i + h] == s[j + h]) {
                h++;
            }
            lcp[r] = h;
            // h 至少减少1（如果 h > 0）
            if (h > 0) h--;
        } else {
            lcp[r] = 0;
        }
    }
    
    return lcp;
}
```

该算法的时间复杂度分析：指针 $i$ 从 $0$ 到 $n-1$ 单调递增，而 $h$ 的值在每次循环中最多增加 $1$、减少 $1$，因此总操作次数为 $O(n)$。

## 应用

### 子串查找

给定模式串 $P$ 和文本串 $T$，如何判断 $P$ 是否出现在 $T$ 中？

一种预处理 $T$ 的后缀数组的方法：首先对 $T$ 构建后缀数组和 $LCP$ 数组；然后通过二分查找在 $SA$ 中寻找以 $P$ 开头的后缀。二分查找时，利用 $LCP$ 数组快速比较 $P$ 与 $SA[mid]$ 对应后缀的公共前缀长度。

时间复杂度为 $O(|P| \log |T|)$，适合多模式匹配场景。

### 比较任意两个子串

设需要比较 $T[l_1..r_1]$ 与 $T[l_2..r_2]$ 的字典序大小（$r_1, r_2$ 为闭区间）。设这两个子串分别是后缀 $l_1$ 和 $l_2$ 的前缀。

比较过程：
1. 找到 $l_1$ 和 $l_2$ 在后缀数组中的排名 $rank[l_1]$ 和 $rank[l_2]$；
2. 假设 $rank[l_1] < rank[l_2]$，则子串 $T[l_1..r_1]$ 与 $T[l_2..r_2]$ 的字典序关系取决于 $LCP[rank[l_1]+1]$（即两后缀的公共前缀长度）与子串长度的关系：
   - 若 $LCP[rank[l_1]+1] > \min(|T[l_1..r_1]|, |T[l_2..r_2]|)$，则较短子串字典序更小；
   - 否则，比较第一个不同字符即可。

利用 $RMQ$（区间最小值查询）在 $O(1)$ 时间内回答 $LCP$ 查询，整体比较可在 $O(1)$（预处理后）完成。

### 不同子串计数

统计一个字符串中不同子串的数量，是后缀数组的经典应用。

**公式推导**：

字符串 $S$ 的所有子串可以表示为所有后缀的前缀。设 $|S| = n$，后缀数组为 $SA$。

考虑排名第 $i$ 的后缀 $S[SA[i]..]$，它贡献的新子串数量为：

$$
n - SA[i] - LCP[i]
$$

其中 $n - SA[i]$ 是该后缀的所有前缀数量，减去 $LCP[i]$ 个与前一个后缀重复的前缀。

因此，不同子串的总数为：

$$
\sum_{i=0}^{n-1} (n - SA[i] - LCP[i]) = \frac{n(n+1)}{2} - \sum_{i=0}^{n-1} LCP[i]
$$

该公式利用了 $\sum SA[i] = \frac{n(n-1)}{2}$ 的性质（虽然直接计算更直观）。

### 其他应用

- **最长重复子串**：在后缀数组中找到 $LCP$ 值最大的相邻后缀；
- **最长回文子串**：将字符串与其反转拼接，利用后缀数组求解；
- **字符串压缩**：利用不同子串数量评估字符串的压缩潜力。

## 参考资料

[^1]: 本页内容参考 [Suffix Array. CP-Algorithms.](https://cp-algorithms.com/string/suffix-array.html)
[^2]: 本页内容参考 [Kasai's Algorithm. CP-Algorithms.](https://cp-algorithms.com/string/suffix-array.html#kasais-algorithm)
[^3]: 后缀数组详解可参考 [后缀数组 - OI Wiki](https://oi-wiki.org/string/sa/)
