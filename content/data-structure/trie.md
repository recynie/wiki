---
title: 字典树
date: 2026-03-31
description: 字典树（Trie）数据结构
tags:
  - trie
  - string
  - data-structure
draft: false
permalink:
---

## 简介

字典树（Trie），又称前缀树（Prefix Tree），是一种树形数据结构，用于高效地存储和检索字符串集合中的键。[^1]

## 核心性质

### 节点结构

每个节点代表一个字符，包含：

- `children`：从该节点出发的所有可能边的映射（通常用数组或哈希表实现）
- `isEnd`：标记从根节点到该节点的路径是否构成一个完整字符串

对于小写字母表，可用数组 `children[26]`，时间复杂度为 $O(1)$ 寻址；若字符集较大或稀疏，使用哈希表更节省空间。

### 时间复杂度

| 操作 | 时间复杂度 | 说明 |
|------|-----------|------|
| 插入（Insert） | $O(|S|)$ | $|S|$ 为字符串长度 |
| 查找（Search） | $O(|S|)$ | $|S|$ 为字符串长度 |
| 前缀查询（Prefix） | $O(|P|)$ | $|P|$ 为前缀长度 |

空间复杂度：$O(\sum |S| \cdot \Sigma)$，其中 $\Sigma$ 为字符集大小。

## 算法步骤

### 插入操作

1. 从根节点开始
2. 对于字符串中的每个字符 $c$：
   - 若当前节点的 `children[c]` 存在，则移动到该节点
   - 否则创建新节点，移动到该节点
3. 遍历完成后，将最后一个节点的 `isEnd` 设为 `true`

### 查找操作

1. 从根节点开始
2. 对于字符串中的每个字符 $c$：
   - 若 `children[c]` 不存在，则查找失败
   - 否则移动到 `children[c]`
3. 遍历完成后，检查最后一个节点的 `isEnd`

## 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

struct TrieNode {
    TrieNode* children[26];
    bool isEnd;
    TrieNode() {
        isEnd = false;
        memset(children, 0, sizeof(children));
    }
};

class Trie {
private:
    TrieNode* root;
public:
    Trie() {
        root = new TrieNode();
    }
    
    void insert(const string& word) {
        TrieNode* node = root;
        for (char c : word) {
            int idx = c - 'a';
            if (!node->children[idx]) {
                node->children[idx] = new TrieNode();
            }
            node = node->children[idx];
        }
        node->isEnd = true;
    }
    
    bool search(const string& word) {
        TrieNode* node = root;
        for (char c : word) {
            int idx = c - 'a';
            if (!node->children[idx]) {
                return false;
            }
            node = node->children[idx];
        }
        return node->isEnd;
    }
    
    bool startsWith(const string& prefix) {
        TrieNode* node = root;
        for (char c : prefix) {
            int idx = c - 'a';
            if (!node->children[idx]) {
                return false;
            }
            node = node->children[idx];
        }
        return true;
    }
};

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    Trie trie;
    int n;
    cin >> n;
    while (n--) {
        string op, s;
        cin >> op >> s;
        if (op == "insert") {
            trie.insert(s);
        } else if (op == "search") {
            cout << (trie.search(s) ? "true" : "false") << "\n";
        } else if (op == "startsWith") {
            cout << (trie.startsWith(s) ? "true" : "false") << "\n";
        }
    }
    return 0;
}
```

## 应用场景

- **自动补全**：搜索引擎、IDE 的代码补全
- **前缀匹配**：IP路由中的最长前缀匹配
- **字符串检索**：词典查询、敏感词过滤
- **字符串排序**：通过遍历获取所有字符串的有序序列
- **与[[algorithms/kmp|KMP算法]]结合**：构建AC自动机用于多模式串匹配

## 参考资料

[^1]: 本词条内容来自 [OI Wiki - Trie树](https://oi-wiki.org/string/trie/)
