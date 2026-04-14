---
title: 贪心算法
date: 2026-03-31
description: 贪心算法（Greedy Algorithm）
tags:
  - greedy
  - algorithm
draft: false
permalink:
---

## 定义

贪心算法（Greedy Algorithm）是一种在每一步选择中都采取**当前状态下最优（即局部最优）**的决策，期望通过局部最优达到全局最优的算法设计策略。

贪心算法有两个关键性质：

1. **贪心选择性质**（Greedy Choice Property）：局部最优选择可以导致全局最优解
2. **最优子结构**（Optimal Substructure）：问题的最优解包含其子问题的最优解[^1]

**重要提示**：贪心算法并不总是能得到最优解。只有当问题满足上述两个性质时，贪心算法才是正确的。因此，在使用贪心算法前必须**严格证明**其正确性。

## 贪心 vs 分治 vs 动态规划

| 特性 | 贪心 | 分治 | 动态规划 |
|------|------|------|----------|
| 决策时机 | 单步决策，不回溯 | 递归分解，合并子问题 | 考虑所有子问题 |
| 最优性保证 | 需要证明 | 通常能得到最优解 | 一定能得到最优解 |
| 时间复杂度 | 通常较低 | 通常较高 | 通常较高 |
| 适用场景 | 满足贪心选择性质的问题 | 子问题独立 | 子问题重叠 |

## 经典贪心算法

### 活动选择问题

给定 $n$ 个活动，每个活动有一个开始时间 $s_i$ 和结束时间 $f_i$。找出最大数量的互不重叠的活动。

**贪心策略**：选择最早结束且与已选活动兼容的活动。

```cpp
#include <bits/stdc++.h>
using namespace std;

// 活动选择问题
int activitySelection(vector<pair<int, int>>& activities) {
    // 按结束时间排序
    sort(activities.begin(), activities.end(), 
         [](const pair<int, int>& a, const pair<int, int>& b) {
             return a.second < b.second;
         });
    
    int count = 0;
    int lastEnd = -1;
    
    for (auto& activity : activities) {
        if (activity.first >= lastEnd) {
            count++;
            lastEnd = activity.second;
        }
    }
    
    return count;
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    vector<pair<int, int>> activities = {
        {1, 4}, {3, 5}, {0, 6}, {5, 7}, {3, 9}, {5, 9}, {6, 10}, {8, 11}, {8, 12}, {2, 14}, {12, 16}
    };
    
    cout << activitySelection(activities) << endl;
    
    return 0;
}
```

### 哈夫曼编码

哈夫曼编码是一种最优前缀码，用于数据压缩。

**贪心策略**：每次选择频率最小的两个节点合并。

```cpp
#include <bits/stdc++.h>
using namespace std;

struct HuffmanNode {
    char ch;
    int freq;
    HuffmanNode* left;
    HuffmanNode* right;
    HuffmanNode(char c, int f) : ch(c), freq(f), left(nullptr), right(nullptr) {}
};

struct Compare {
    bool operator()(HuffmanNode* a, HuffmanNode* b) {
        return a->freq > b->freq;  // 小顶堆
    }
};

HuffmanNode* buildHuffmanTree(vector<char>& chars, vector<int>& freq) {
    priority_queue<HuffmanNode*, vector<HuffmanNode*>, Compare> pq;
    
    for (int i = 0; i < chars.size(); i++) {
        pq.push(new HuffmanNode(chars[i], freq[i]));
    }
    
    while (pq.size() > 1) {
        HuffmanNode* left = pq.top();
        pq.pop();
        HuffmanNode* right = pq.top();
        pq.pop();
        
        HuffmanNode* parent = new HuffmanNode('$', left->freq + right->freq);
        parent->left = left;
        parent->right = right;
        
        pq.push(parent);
    }
    
    return pq.top();
}

void printCodes(HuffmanNode* root, string code = "") {
    if (!root) return;
    if (root->ch != '$') {
        cout << root->ch << ": " << code << endl;
    }
    printCodes(root->left, code + "0");
    printCodes(root->right, code + "1");
}
```

### 钱币找零问题

给定不同面额的硬币，求找零所需的最少硬币数。

```cpp
#include <bits/stdc++.h>
using namespace std;

// 假设硬币面额为 [1, 5, 10, 25, 50]
int coinChange(vector<int>& coins, int amount) {
    sort(coins.begin(), coins.end(), greater<int>());  // 降序排列
    
    int count = 0;
    for (int coin : coins) {
        if (amount <= 0) break;
        count += amount / coin;
        amount %= coin;
    }
    
    return amount == 0 ? count : -1;  // 无法找零返回 -1
}

// 注意：上述贪心策略对某些硬币系统不适用！
// 例如硬币面额为 [1, 3, 4]，amount = 6
// 贪心得到 4+1+1=3 枚，但最优解是 3+3=2 枚
```

### 分数背包问题

给定物品的重量和价值，在容量为 $W$ 的背包中装入价值最大的物品（可以取部分）。

**贪心策略**：按单位价值（$value/weight$）降序选择。

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Item {
    int weight;
    int value;
    double ratio;  // 单位价值
    Item(int w, int v) : weight(w), value(v), ratio((double)v / w) {}
};

double fractionalKnapsack(vector<Item>& items, int capacity) {
    sort(items.begin(), items.end(), [](const Item& a, const Item& b) {
        return a.ratio > b.ratio;
    });
    
    double totalValue = 0;
    int currentWeight = 0;
    
    for (const auto& item : items) {
        if (currentWeight + item.weight <= capacity) {
            currentWeight += item.weight;
            totalValue += item.value;
        } else {
            int remaining = capacity - currentWeight;
            totalValue += item.ratio * remaining;
            break;
        }
    }
    
    return totalValue;
}
```

### 区间调度问题

求最多有多少个不重叠的区间。

```cpp
#include <bits/stdc++.h>
using namespace std;

int intervalScheduling(vector<pair<int, int>>& intervals) {
    // 按结束时间排序
    sort(intervals.begin(), intervals.end(), 
         [](const pair<int, int>& a, const pair<int, int>& b) {
             return a.second < b.second;
         });
    
    int count = 0;
    int lastEnd = 0;
    
    for (auto& interval : intervals) {
        if (interval.first >= lastEnd) {
            count++;
            lastEnd = interval.second;
        }
    }
    
    return count;
}
```

### 柠檬水找零

```cpp
#include <bits/stdc++.h>
using namespace std;

bool lemonadeChange(vector<int>& bills) {
    int five = 0, ten = 0;
    
    for (int bill : bills) {
        if (bill == 5) {
            five++;
        } else if (bill == 10) {
            if (five == 0) return false;
            five--;
            ten++;
        } else {  // bill == 20
            if (ten > 0 && five > 0) {
                ten--;
                five--;
            } else if (five >= 3) {
                five -= 3;
            } else {
                return false;
            }
        }
    }
    
    return true;
}
```

## 贪心算法正确性证明

贪心算法的关键在于正确性证明。常用的证明方法有：

### 交换论证（Exchange Argument）

假设存在一个最优解 $O$，将其与贪心解 $G$ 进行比较，通过一系列不降低解质量的交换，最终将 $O$ 转换为 $G$，从而证明 $G$ 也是最优的。

### 归纳论证（Inductive Argument）

证明在每一步选择局部最优后，存在一个最优解包含这个选择，然后通过归纳法证明整个贪心解是最优的。

## 参考资料

[^1]: 贪心算法的正确性必须严格证明。仅通过"看起来正确"或"大多数情况正确"是不可接受的。

[^2]: 0/1 背包问题不能使用贪心算法（因为物品不可分割），必须使用动态规划。
