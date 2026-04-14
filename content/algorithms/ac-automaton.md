---
title: AC自动机
date: 2026-04-03
description: AC 自动机（Aho-Corasick Automaton）多模式字符串匹配算法
tags:
  - string
  - automaton
  - ac-automaton
draft: false
permalink:
---

## 概述

AC 自动机是**以 Trie 为基础，结合 KMP 思想**建立的自动机，用于解决多模式匹配问题。

在阅读本文之前，请先阅读 [[../algorithms/kmp|KMP 算法]] 和 [[../data-structure/trie|字典树 (Trie)]]。

## 核心思想

AC 自动机的核心是**失配指针（fail 指针）**。

对于状态 u，其 fail 指针指向另一个状态 v，其中 v 是 u 的最长后缀（即所有后缀状态中字典序最长的那个）。

与 KMP 的 next 指针相比：
- **共同点**：失配时用于跳转
- **不同点**：KMP 求最长相同前后缀，AC 自动机求所有模式串前缀中匹配当前状态的最长后缀

## Trie 构建

将所有模式串插入 Trie 树，每个结点表示一个状态。

```cpp
struct Node {
    int next[26];  // 字典树转移边
    int fail;      // 失配指针
    int cnt;       // 结束标记（该结点表示的模式串个数）
    Node() {
        memset(next, 0, sizeof(next));
        fail = 0;
        cnt = 0;
    }
};

vector<Node> ac(1);  // 根结点编号为 0
```

## 构建失配指针

使用 BFS 按层构建 fail 指针。构建原则：

1. 如果 `trie[fail(p), c]` 存在，则 `fail(u) = trie[fail(p), c]`
2. 否则继续跳 fail 指针直到根
3. 根结点的 fail 指针为 0

### 代码实现

```cpp
void build() {
    queue<int> q;
    // 根结点的直接子结点入队
    for (int i = 0; i < 26; i++) {
        if (ac[0].next[i]) {
            ac[ac[0].next[i]].fail = 0;
            q.push(ac[0].next[i]);
        }
    }
    
    while (!q.empty()) {
        int u = q.front();
        q.pop();
        
        for (int i = 0; i < 26; i++) {
            int v = ac[u].next[i];
            if (v) {
                // 子结点：设置 fail 指针
                ac[v].fail = ac[ac[u].fail].next[i];
                q.push(v);
            } else {
                // 非子结点：构建自动机边（优化）
                ac[u].next[i] = ac[ac[u].fail].next[i];
            }
        }
    }
}
```

### 优化：字典图

上述代码中的 `else` 分支将不存在的字典树边指向 fail 指针对应状态，从而将 Trie 压缩成**字典图**。这一优化使得匹配时只需 O(1) 转移。

## 多模式匹配

给定文本串 T，遍历文本串并沿自动机转移：

```cpp
int query(const string& text) {
    int u = 0, res = 0;
    for (char c : text) {
        u = ac[u].next[c - 'a'];
        for (int j = u; j && ac[j].cnt != -1; j = ac[j].fail) {
            res += ac[j].cnt;
            ac[j].cnt = -1;  // 避免重复统计
        }
    }
    return res;
}
```

如果需要统计每个模式串的出现次数，需要记录每个结点对应的模式串编号。

## 完整模板

```cpp
#include <bits/stdc++.h>
using namespace std;

struct AhoCorasick {
    struct Node {
        int next[26];
        int fail;
        int out;  // 输出信息
        Node() {
            memset(next, 0, sizeof(next));
            fail = 0;
            out = 0;
        }
    };
    
    vector<Node> t;
    
    AhoCorasick() {
        t.emplace_back();
    }
    
    void insert(const string& s) {
        int u = 0;
        for (char c : s) {
            int idx = c - 'a';
            if (!t[u].next[idx]) {
                t[u].next[idx] = t.size();
                t.emplace_back();
            }
            u = t[u].next[idx];
        }
        t[u].out++;  // 结束标记
    }
    
    void build() {
        queue<int> q;
        for (int i = 0; i < 26; i++) {
            if (t[0].next[i]) {
                t[t[0].next[i]].fail = 0;
                q.push(t[0].next[i]);
            } else {
                t[0].next[i] = 0;  // 优化
            }
        }
        
        while (!q.empty()) {
            int u = q.front();
            q.pop();
            
            for (int i = 0; i < 26; i++) {
                int v = t[u].next[i];
                if (v) {
                    t[v].fail = t[t[u].fail].next[i];
                    q.push(v);
                } else {
                    t[u].next[i] = t[t[u].fail].next[i];
                }
            }
        }
    }
    
    // 在文本中搜索模式串
    vector<int> search(const string& text) {
        vector<int> result(t.size(), 0);
        int u = 0;
        for (char c : text) {
            u = t[u].next[c - 'a'];
            result[u]++;  // 记录经过该状态的次数
        }
        // 沿 fail 链传递
        for (int i = t.size() - 1; i >= 0; i--) {
            if (t[i].fail) {
                result[t[i].fail] += result[i];
            }
        }
        return result;
    }
};
```

## 应用场景

AC 自动机广泛应用于：

- **多模式匹配**：在一篇文章中查找多个模式串
- **字符串统计**：统计各模式串在文本中的出现次数
- **敏感词过滤**：文本过滤系统
- **DNA 序列匹配**：生物信息学中的应用

## 相关主题

- [[../algorithms/kmp|KMP 算法]]：单模式匹配
- [[../data-structure/trie|字典树]]：AC 自动机的基础结构
- [[../algorithms/string-hash|字符串哈希]]：另一种字符串匹配方法

## 参考资料

[^1]: [AC 自动机 - OI Wiki](https://oi-wiki.org/string/ac-automaton/)
