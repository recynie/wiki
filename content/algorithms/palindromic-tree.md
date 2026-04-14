---
title: 回文树
date: 2026-04-06
description: 回文树（Palindromic Tree/Eertree）——高效处理字符串回文问题的数据结构
tags:
  - string-algorithm
  - palindrome
  - data-structure
draft: false
permalink:
---

## 定义

回文树（Palindromic Tree），又称 Eertree，是一种专门用于处理字符串回文问题的数据结构。由 Mikhail Rubinchik 和 Arseny M. Shur 于 2015 年提出。[^1]

回文树的核心思想是：在线构建一个由所有不同回文子串组成的树状结构，每个节点代表一个不同的回文，支持在线插入字符，时间复杂度为 $O(n)$。

### 适用场景

- 统计一个字符串中不同回文的个数
- 求每个回文出现的次数
- 求最长回文子串
- 求回文子串的种类数
- 字符串的回文匹配问题

---

## 结构

回文树包含两种特殊节点和若干普通节点：

### 特殊节点

| 节点 | 长度 | 含义 |
|------|------|------|
| 奇根（odd root） | $-1$ | 虚拟节点，用于匹配长度为 1 的回文 |
| 偶根（even root） | $0$ | 代表空字符串 |

### 普通节点

每个普通节点存储：
- `len`：回文的长度
- `link`：后缀链接，指向该回文的最长真回文后缀
- `next`：转移边，字符到子节点的映射
- `occ`：该回文出现的次数（用于统计）
- `first_pos`：该回文第一次出现的位置结尾

### 边

| 边类型 | 含义 | 图示 |
|--------|------|------|
| 字符边（character edge） | 在回文两端添加相同字符 | 向下黑色 |
| 后缀边（suffix edge） | 连接到最长真回文后缀 | 向上蓝色 |

---

## 构建过程

回文树在字符串末尾逐字符插入构建。设当前已处理字符串 $s$，新字符为 $c$，位置为 $pos$。

### 插入步骤

1. **寻找可扩展节点**：从 `last`（最长回文后缀）开始，通过后缀边不断寻找，直到找到节点 $p$ 使得 $s[pos-1-len(p)] = c$
2. **检查是否已存在**：若 $p$ 通过字符 $c$ 的转移已存在，直接复用
3. **创建新节点**：若不存在，创建新节点 $cur$，$len[cur] = len[p] + 2$
4. **建立后缀链接**：对于长度大于 1 的节点，通过跳过后缀边找到最长真回文后缀

### 正确性保证

关键性质：每个位置最多只会产生一个新的回文节点。

因为对于位置 $pos$，若字符 $c$ 能形成新的回文，则该回文必须是唯一的——它是 $s[pos]$ 向左延伸的最长回文。因此总节点数为 $O(n)$。

---

## C++ 实现

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Node {
    int len;                    // 回文长度
    int link;                   // 后缀链接
    int next[26];               // 转移边（字符->节点）
    long long occ;             // 出现次数
    int first_pos;              // 首次出现位置
    Node(int l = 0) : len(l), link(0), occ(0), first_pos(-1) {
        memset(next, 0, sizeof(next));
    }
};

class PalindromicTree {
private:
    vector<Node> tree;
    string s;
    int last;                   // 最长回文后缀对应的节点
    int curPos;                 // 当前处理的位置

public:
    PalindromicTree() {
        // 初始化两个特殊根节点
        tree.push_back(Node(-1));   // 奇根，长度 -1
        tree[0].link = 0;           // 指向自己
        tree.push_back(Node(0));    // 偶根，长度 0
        tree[1].link = 0;           // 指向奇根
        last = 1;                   // 初始为偶根
        curPos = -1;
    }

    // 寻找可扩展节点
    int getFail(int v) {
        while (true) {
            int L = tree[v].len;
            if (curPos - 1 - L >= 0 && s[curPos - 1 - L] == s[curPos])
                return v;
            v = tree[v].link;
        }
    }

    // 添加字符
    void addChar(char c) {
        s += c;
        curPos++;
        int pos = curPos;
        int chr = c - 'a';

        // 步骤1：找到可扩展的节点
        int cur = getFail(last);

        // 步骤2：检查是否已存在
        if (tree[cur].next[chr]) {
            last = tree[cur].next[chr];
            tree[last].occ++;
            return;
        }

        // 步骤3：创建新节点
        tree.push_back(Node(tree[cur].len + 2));
        int newNode = (int)tree.size() - 1;
        tree[cur].next[chr] = newNode;
        tree[newNode].occ = 1;
        tree[newNode].first_pos = pos;

        // 步骤4：建立后缀链接
        if (tree[newNode].len == 1) {
            tree[newNode].link = 1;  // 指向偶根
        } else {
            int linkCandidate = getFail(tree[cur].link);
            tree[newNode].link = tree[linkCandidate].next[chr];
        }

        last = newNode;
    }

    // 构建（一次性构建）
    void build(const string& str) {
        for (char c : str) addChar(c);
    }

    // 统计出现次数（从长到短传播）
    void propagate() {
        // 按长度降序排列节点
        vector<int> order(tree.size());
        iota(order.begin(), order.end(), 0);
        sort(order.begin(), order.end(), [&](int a, int b) {
            return tree[a].len > tree[b].len;
        });

        for (int v : order) {
            if (v > 1) {  // 跳过两个根节点
                tree[tree[v].link].occ += tree[v].occ;
            }
        }
    }

    // 不同回文数量
    int distinctPalindromes() {
        return (int)tree.size() - 2;  // 减去两个根节点
    }

    // 最长回文（长度）
    pair<int, string> longestPalindrome() {
        int bestNode = 1;
        for (int i = 2; i < (int)tree.size(); i++) {
            if (tree[i].len > tree[bestNode].len)
                bestNode = i;
        }
        int L = tree[bestNode].len;
        int end = tree[bestNode].first_pos;
        int start = end - L + 1;
        return {L, s.substr(start, L)};
    }

    // 获取某个回文节点的信息
    Node& operator[](int idx) { return tree[idx]; }
    int size() { return (int)tree.size(); }
};
```

### 使用示例

```cpp
int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    string s;
    cin >> s;

    PalindromicTree pt;
    pt.build(s);
    pt.propagate();

    cout << "不同回文数量: " << pt.distinctPalindromes() << endl;

    auto [len, pal] = pt.longestPalindrome();
    cout << "最长回文长度: " << len << endl;
    cout << "最长回文: " << pal << endl;

    return 0;
}
```

---

## 算法复杂度

| 操作 | 时间复杂度 | 空间复杂度 |
|------|-----------|-----------|
| 添加字符 | $O(1)$（均摊） | $O(n)$ |
| 构建 | $O(n)$ | $O(n)$ |
| 统计次数 | $O(n)$ | $O(1)$ |
| 查询最长回文 | $O(n)$ | $O(1)$ |

---

## 例题

### 例题一：回文子串计数

#### 问题

给定字符串 $s$，统计所有回文子串的数量（重复计）。

#### 分析

利用回文树，插入每个字符后，当前的 `last` 节点对应的回文一定以该字符结尾。因此，我们可以累加每个插入位置后新增的节点数。

实际上，更简单的方法是：插入完成后，将所有 `occ` 累加即可。

#### 代码

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Node {
    int len, link, next[26];
    long long occ;
    Node(int l = 0) : len(l), link(0), occ(0) {
        memset(next, 0, sizeof(next));
    }
};

class PalindromicTree {
    vector<Node> tree;
    string s;
    int last, curPos;
public:
    PalindromicTree() {
        tree.emplace_back(-1);
        tree[0].link = 0;
        tree.emplace_back(0);
        tree[1].link = 0;
        last = 1;
        curPos = -1;
    }

    int getFail(int v) {
        while (curPos - 1 - tree[v].len >= 0 &&
               s[curPos - 1 - tree[v].len] != s[curPos])
            v = tree[v].link;
        return v;
    }

    void addChar(char c) {
        s += c;
        curPos++;
        int chr = c - 'a';
        int cur = getFail(last);

        if (!tree[cur].next[chr]) {
            tree.emplace_back(tree[cur].len + 2);
            int newNode = (int)tree.size() - 1;
            tree[cur].next[chr] = newNode;
            if (tree[newNode].len == 1) {
                tree[newNode].link = 1;
            } else {
                int linkCand = getFail(tree[cur].link);
                tree[newNode].link = tree[linkCand].next[chr];
            }
        }
        last = tree[cur].next[chr];
        tree[last].occ++;
    }

    long long countPalindromes() {
        long long total = 0;
        for (int i = 2; i < (int)tree.size(); i++)
            total += tree[i].occ;
        return total;
    }
};

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    string s;
    cin >> s;
    PalindromicTree pt;
    for (char c : s) pt.addChar(c);
    cout << pt.countPalindromes() << endl;
    return 0;
}
```

---

## 与其他回文算法的比较

| 算法 | 时间复杂度 | 空间复杂度 | 特点 |
|------|-----------|-----------|------|
| 中心扩展法 | $O(n^2)$ | $O(1)$ | 简单，适合单次查询 |
| Manacher | $O(n)$ | $O(n)$ | 求最长回文，线性 |
| DP 表 | $O(n^2)$ | $O(n^2)$ | 求所有子串是否是回文 |
| 回文树 | $O(n)$ | $O(n)$ | 在线，支持多种回文查询 |

回文树的独特优势：
- **在线构建**：可以随时添加字符，实时更新
- **自动去重**：每个不同回文只存储一个节点
- **统计便捷**：通过 `occ` 字段快速统计出现次数

---

## 应用场景

1. **统计不同回文**：直接返回节点数 $-2$
2. **回文频率统计**：插入后累加 `occ`，再通过 `propagate` 统计
3. **最长回文查询**：维护或遍历找到最大 `len`
4. **回文子串枚举**：DFS 遍历树的节点
5. **字符串匹配**：在文本中插入模式串的回文树

---

## 参考资料

[^1]: EERTREE: An Efficient Data Structure for Processing Palindromes in Strings. Mikhail Rubinchik and Arseny M. Shur. https://arxiv.org/pdf/1506.04862

[^2]: 回文树 (Eertree) - OI Wiki. https://oi-wiki.org/string/pam/

[^3]: Palindromic Tree: A Practical Guide - TheLinuxCode. https://thelinuxcode.com/palindromic-tree-a-practical-guide-to-building-and-using-it-in-2026/
