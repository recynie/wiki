---
title: 后缀自动机（Suffix Automaton）
date: 2026-04-05
description: 后缀自动机是一种在线构造的最小 DFA，能高效地处理字符串的所有子串相关查询
tags:
  - suffix-automaton
  - string
  - automaton
  - data-structure
draft: false
permalink:
---

## 概述

后缀自动机（Suffix Automaton，简称 SAM）是一种用于处理字符串问题的强大数据结构。它是一个**确定性有限自动机（DFA）**，接受给定字符串的所有后缀，并且是所有接受这些后缀的 DFA 中状态数最少的[【1】](#参考)。

后缀自动机的核心思想是：将字符串的所有子串压缩成一个有限状态机，每个状态代表一组具有相同「出现位置集合」的子串（即 `endpos` 等价类）。这种压缩方式使得我们可以用 **$O(n)$ 的时间复杂度和空间复杂度**（实际实现中空间约为 $2n-1$ 个状态）来处理大多数字符串查询问题。

## endpos 等价类

设字符串 $S$ 的长度为 $n$，对于 $S$ 的任意子串 $t$，定义 $endpos(t)$ 为所有**结束位置**的集合：

$$
endpos(t) = \{ i \mid S[1..i] \text{ 以 } t \text{ 结尾} \}
$$

如果两个子串 $t_1$ 和 $t_2$ 满足 $endpos(t_1) = endpos(t_2)$，则称它们属于同一个 **endpos 等价类**。

### 性质

endpos 等价类具有以下重要性质[【1】](#参考)：

1. **同一等价类中的子串长度连续**：若等价类中子串的最大长度为 $L_{max}$，最小长度为 $L_{min}$，则该类包含所有长度为 $L_{min}, L_{min}+1, \ldots, L_{max}$ 的子串。
2. **后缀链接关系**：较短子串的后缀可能属于另一个等价类，这种关系构成一条链。

## 后缀链接（Suffix Link）

后缀链接是后缀自动机中连接不同状态的「指针」，又称 **link**。

设状态 $v$ 代表等价类 $C_v$，包含所有长度为 $[l_v, r_v]$ 的子串（其中 $r_v$ 是该类中最长子串的长度，$l_v = r_v - \text{len}(v) + 1$）。则 $v$ 的后缀链接 $link(v)$ 指向另一个状态 $u$，满足：

- $u$ 代表的等价类包含所有 $C_v$ 中子串的**真后缀**
- $u$ 是满足这一条件的状态中 $r_u$ 最大的（即最长后缀所在的等价类）

### 性质

- 所有状态的后缀链接构成一棵树，称为 **link 树**
- 根节点（初始状态）的 $link = -1$
- 从任意状态沿 link 向上走，会依次得到该状态代表子串的：后缀 → 后缀的后缀 → ... → 空串

## 状态结构

后缀自动机的每个状态存储以下信息：

| 属性 | 含义 |
|------|------|
| `len` | 该状态代表的最长子串长度（即 $r_v$） |
| `link` | 后缀链接指向的状态编号 |
| `next` | 转移函数，$next[c]$ 表示从该状态接受字符 $c$ 后的目标状态 |

## 构造算法

后缀自动机采用**在线算法**构造，每次读入一个新字符 $c$，执行 `extend(c)` 操作。算法保持自动机的最小性，确保始终是正确的 DFA[【1】](#参考)。

### extend 操作

设当前字符串为 $S$，已构造好的自动机对应 $S$ 的所有子串。现在要在末尾添加字符 $c$，得到新字符串 $S' = S + c$。

**步骤如下：**

1. **创建新状态**：新建状态 $cur$，令 $\text{len}(cur) = \text{len}(last) + 1$（其中 $last$ 是添加字符前代表整个字符串 $S$ 的状态）

2. **沿后缀链接寻找插入位置**：从 $last$ 开始，沿 link 向上跳，找到第一个可以通过字符 $c$ 转移的状态 $p$，即满足 $p.next[c] \neq \text{null}$

3. **建立转移**：令 $cur.next[c] = cur$

4. **处理未找到 $p$ 的情况**：如果一直跳到根节点（$p = -1$）仍未找到可转移的状态，则 $link(cur) = 0$（根状态）

5. **处理找到 $p$ 的情况**：设 $q = p.next[c]$
   - 若 $\text{len}(p) + 1 = \text{len}(q)$，则 $link(cur) = q$
   - 否则需要**克隆**状态 $q$：

### 状态克隆

当 $\text{len}(p) + 1 \neq \text{len}(q)$ 时，需要创建克隆状态 $clone$：

- $\text{len}(clone) = \text{len}(p) + 1$
- $clone.next = q.next$（复制所有转移）
- $link(clone) = link(q)$
- 然后将 $q$ 和 $cur$ 的 link 都指向 $clone$

克隆操作保证了自动机的最小性：将 $q$ 分裂成两个状态，使得 $\text{len}$ 值连续。

### 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

struct state {
    int len = 0;           // 最长字符串长度
    int link = -1;         // 后缀链接
    map<char, int> next;   // 转移
};

class SuffixAutomaton {
private:
    vector<state> st;
    int last = 0;          // 最后添加的状态

public:
    SuffixAutomaton(int maxLen = 0) {
        st.reserve(2 * maxLen);
        st.push_back(state());  // 初始化根状态
        st[0].len = 0;
        st[0].link = -1;
        last = 0;
    }

    void extend(char c) {
        int cur = st.size();
        st.push_back(state());
        st[cur].len = st[last].len + 1;

        int p = last;
        // 寻找可转移的位置
        while (p != -1 && !st[p].next.count(c)) {
            st[p].next[c] = cur;
            p = st[p].link;
        }

        if (p == -1) {
            st[cur].link = 0;
        } else {
            int q = st[p].next[c];
            if (st[p].len + 1 == st[q].len) {
                st[cur].link = q;
            } else {
                // 克隆状态
                int clone = st.size();
                st.push_back(st[q]);  // 复制 q 的所有信息
                st[clone].len = st[p].len + 1;
                // link 不变（在下一步被修改）

                // 调整 q 的后缀链接
                while (p != -1 && st[p].next[c] == q) {
                    st[p].next[c] = clone;
                    p = st[p].link;
                }
                st[q].link = st[cur].link = clone;
            }
        }
        last = cur;
    }

    // 查询：检查模式串是否为 S 的子串
    bool contains(const string& t) const {
        int v = 0;
        for (char c : t) {
            if (!st[v].next.count(c)) return false;
            v = st[v].next.at(c);
        }
        return true;
    }

    // 查询：S 中不同子串的个数
    long long countDistinctSubstrings() const {
        long long ans = 0;
        for (int i = 1; i < (int)st.size(); i++) {
            ans += st[i].len - st[st[i].link].len;
        }
        return ans;
    }

    // 查询：两字符串的最长公共子串长度
    int longestCommonSubstring(const string& t) const {
        int v = 0, l = 0, best = 0;
        for (char c : t) {
            while (v && !st[v].next.count(c)) {
                v = st[v].link;
                l = st[v].len;
            }
            if (st[v].next.count(c)) {
                v = st[v].next.at(c);
                l++;
            }
            best = max(best, l);
        }
        return best;
    }
};
```

### 时间复杂度分析

构造后缀自动机的总时间复杂度为 $O(n)$，其中 $n$ 为字符串长度。证明思路：

- 每次 `extend` 操作中，沿 link 向上跳的总次数均摊为 $O(1)$
- 克隆操作不增加状态总数（最多 $2n-1$ 个状态）

## 应用场景

### 1. 子串查询

检查模式串 $P$ 是否为原串 $S$ 的子串，只需从初始状态出发，沿 $P$ 的每个字符进行转移。若转移失败或最终不在合法状态，则 $P$ 不是子串。

**复杂度**：$O(|P|)$

### 2. 不同子串计数

根据自动机结构，状态 $v$ 对应的 endpos 等价类包含长度为 $(\text{len}(\text{link}(v))+1)$ 到 $\text{len}(v)$ 的子串，共 $\text{len}(v) - \text{len}(\text{link}(v))$ 个[【1】](#参考)。

对所有状态求和即可得到不同子串总数：

$$
\text{ans} = \sum_{v=1}^{|SAM|-1} (\text{len}(v) - \text{len}(\text{link}(v)))
$$

**复杂度**：$O(n)$

### 3. 最长公共子串（Longest Common Substring）

给定两字符串 $S$ 和 $T$，求它们的最长公共子串长度。

**思路**：对 $S$ 构造后缀自动机，用 $T$ 在自动机上运行，同时维护当前匹配长度 $l$。当遇到无法继续的字符时，沿 link 上跳并调整 $l$。整个过程中 $l$ 的最大值即为答案。

**复杂度**：$O(|T|)$

### 4. 字典树序第 $k$ 小子串

在自动机的 DAG 上动态规划，可按字典序遍历所有子串。

### 5. 多个字符串的最长公共子串

对第一个字符串构造 SAM，其余字符串在自动机上运行，记录每个状态被各字符串覆盖的「深度」最小值。

## 与其他数据结构的比较

| 数据结构 | 构造复杂度 | 查询能力 | 空间 |
|----------|------------|----------|------|
| [[suffix-array|后缀数组]] | $O(n \log n)$ | 子串排名 | $O(n)$ |
| [[kmp|KMP]] | $O(n)$ | 模式匹配 | $O(n)$ |
| **后缀自动机** | $O(n)$ | 多种子串查询 | $O(2n)$ |

后缀自动机的优势在于：**一种结构，多种查询**，且均为线性或常数复杂度。

## 模板题

- [[../CCF-CSP/CSP202312B|小区买菜]] — 可用后缀自动机优化
- [[../CCF-CSP/CSP202309C|禁止钓鱼]] — 字符串匹配应用

## 参考

[^1]: 本段参考自 [Suffix Automaton - CP-Algorithms](https://cp-algorithms.com/string/suffix-automaton.html)，内容经过验证和扩展。

---

*Last updated: 2026-04-05*
