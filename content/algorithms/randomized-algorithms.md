---
title: 随机算法
date: 2026-04-08
description: 随机算法（Randomized Algorithms）— 利用随机性的算法，包括Monte Carlo和Las Vegas两类，随机化快排、随机化选择等经典应用
tags:
  - randomized-algorithm
  - probability
  - monte-carlo
  - las-vegas
draft: false
permalink:
---

## 概述

随机算法在执行过程中允许使用随机数（掷硬币、随机选择等），根据随机结果做出决策。这与确定性算法的区别在于：对于相同输入，随机算法每次运行可能得到不同结果，但其**期望性能**或**成功概率**是有理论保证的。

### 算法分类

| 类型 | 特点 | 时间复杂度 | 结果正确性 |
|------|------|------------|------------|
| **Las Vegas** | 必定产生正确结果 | 随机 | 确定性 |
| **Monte Carlo** | 运行时间确定 | 确定性 | 概率性 |
| **随机化分析** | 期望/均摊复杂度 | - | - |

## 随机化快速排序（Randomized QuickSort）

### 核心思想

快速排序的平均时间复杂度为 $O(n \log n)$，但最坏情况为 $O(n^2)$。通过**随机选择主元**，可以使期望时间复杂度为 $O(n \log n)$，且对任何输入都有较高概率保持良好性能。

```cpp
#include <bits/stdc++.h>
using namespace std;

// 随机化快速排序
class RandomizedQuickSort {
private:
    mt19937 rng;  // Mersenne Twister随机数生成器
    
    // 随机选择主元
    int randomPivot(int left, int right) {
        uniform_int_distribution<int> dist(left, right);
        return dist(rng);
    }
    
    // 划分函数
    int partition(vector<int>& arr, int left, int right) {
        int pivotIdx = randomPivot(left, right);
        int pivot = arr[pivotIdx];
        
        // 将主元移到末尾
        swap(arr[pivotIdx], arr[right]);
        int i = left - 1;
        
        for (int j = left; j < right; j++) {
            if (arr[j] <= pivot) {
                i++;
                swap(arr[i], arr[j]);
            }
        }
        swap(arr[i + 1], arr[right]);
        return i + 1;
    }
    
    // 三数取中划分（另一种随机化策略）
    int medianOfThree(vector<int>& arr, int left, int right) {
        int mid = left + (right - left) / 2;
        
        // 随机交换后再取中
        if (arr[mid] < arr[left]) swap(arr[mid], arr[left]);
        if (arr[right] < arr[left]) swap(arr[right], arr[left]);
        if (arr[right] < arr[mid]) swap(arr[right], arr[mid]);
        
        return mid;
    }
    
public:
    RandomizedQuickSort() {
        random_device rd;
        rng = mt19937(rd());
    }
    
    void sort(vector<int>& arr) {
        quickSort(arr, 0, arr.size() - 1);
    }
    
private:
    void quickSort(vector<int>& arr, int left, int right) {
        if (left < right) {
            int pivotIdx = partition(arr, left, right);
            quickSort(arr, left, pivotIdx - 1);
            quickSort(arr, pivotIdx + 1, right);
        }
    }
};

// 使用示例
int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    vector<int> arr = {64, 34, 25, 12, 22, 11, 90};
    
    RandomizedQuickSort rqs;
    rqs.sort(arr);
    
    for (int x : arr) cout << x << " ";
    cout << "\n";
    
    return 0;
}
```

### 复杂度分析

**期望时间复杂度**：$O(n \log n)$

设 $T(n)$ 为期望时间，有递归式：

$$T(n) = \frac{1}{n} \sum_{k=1}^{n-1} [T(k) + T(n-k-1)] + O(n)$$

求解得 $T(n) = O(n \log n)$。

**最坏情况**：$O(n^2)$，但发生概率极低（约 $1/n!$）。

## 随机化选择算法（Randomized Selection）

### 问题

在无序数组中找到第 $k$ 小的元素。

### Quickselect算法

```cpp
#include <bits/stdc++.h>
using namespace std;

// 随机化选择算法 - 找第k小元素
class RandomizedSelect {
private:
    mt19937 rng;
    
    int randomPivot(int left, int right) {
        uniform_int_distribution<int> dist(left, right);
        return dist(rng);
    }
    
    // 划分
    int partition(vector<int>& arr, int left, int right) {
        int pivotIdx = randomPivot(left, right);
        int pivot = arr[pivotIdx];
        swap(arr[pivotIdx], arr[right]);
        
        int i = left;
        for (int j = left; j < right; j++) {
            if (arr[j] < pivot) {
                swap(arr[i], arr[j]);
                i++;
            }
        }
        swap(arr[i], arr[right]);
        return i;
    }
    
public:
    RandomizedSelect() {
        random_device rd;
        rng = mt19937(rd());
    }
    
    // 查找第k小元素（k从0开始）
    int select(vector<int>& arr, int left, int right, int k) {
        if (left == right) return arr[left];
        
        int pivotIdx = partition(arr, left, right);
        int num Smaller = pivotIdx - left;  // 比主元小的元素个数
        
        if (k < numSmaller) {
            return select(arr, left, pivotIdx - 1, k);
        } else if (k > numSmaller) {
            return select(arr, pivotIdx + 1, right, k - numSmaller - 1);
        } else {
            return arr[pivotIdx];
        }
    }
};
```

### 期望复杂度

**期望时间**：$O(n)$

类似于随机化快排，每次划分随机选择主元，递归深度期望为 $O(\log n)$，总期望比较次数为 $O(n)$。

### BFPRT（最坏情况O(n)选择）

```cpp
#include <bits/stdc++.h>
using namespace std;

// BFPRT算法 - 最坏情况O(n)选择
// 核心思想：中位数的主元划分保证递归深度
class BFPRT {
private:
    const int THRESHOLD = 5;  // 阈值：小于此值使用插入排序
    
    // 插入排序（用于小数据量）
    void insertionSort(vector<int>& arr, int left, int right) {
        for (int i = left + 1; i <= right; i++) {
            int key = arr[i];
            int j = i - 1;
            while (j >= left && arr[j] > key) {
                arr[j + 1] = arr[j];
                j--;
            }
            arr[j + 1] = key;
        }
    }
    
    // 获取中位数的中位数
    int medianOfMedians(vector<int>& arr, int left, int right) {
        int n = right - left + 1;
        int numGroups = (n + 4) / 5;  // 向上取整
        
        for (int i = 0; i < numGroups; i++) {
            int groupLeft = left + i * 5;
            int groupRight = min(groupLeft + 4, right);
            insertionSort(arr, groupLeft, groupRight);
            // 将中位数移到前面
            int medianIdx = groupLeft + (groupRight - groupLeft) / 2;
            swap(arr[left + i], arr[medianIdx]);
        }
        
        // 递归找中位数的中位数
        if (numGroups == 1) {
            return arr[left];
        }
        return select(arr, left, left + numGroups - 1, numGroups / 2);
    }
    
    int select(vector<int>& arr, int left, int right, int k) {
        if (right - left + 1 <= THRESHOLD) {
            insertionSort(arr, left, right);
            return arr[left + k];
        }
        
        int pivot = medianOfMedians(arr, left, right);
        
        // 用pivot划分
        int i = left;
        for (int j = left; j <= right; j++) {
            if (arr[j] == pivot) {
                swap(arr[j], arr[right]);
                break;
            }
        }
        
        int storeIdx = left;
        for (int j = left; j < right; j++) {
            if (arr[j] < pivot) {
                swap(arr[storeIdx], arr[j]);
                storeIdx++;
            }
        }
        swap(arr[storeIdx], arr[right]);
        
        int num Smaller = storeIdx - left;
        if (k < numSmaller) {
            return select(arr, left, storeIdx - 1, k);
        } else if (k > numSmaller) {
            return select(arr, storeIdx + 1, right, k - numSmaller - 1);
        } else {
            return pivot;
        }
    }
    
public:
    int selectKth(vector<int>& arr, int k) {
        return select(arr, 0, arr.size() - 1, k);
    }
};
```

## 随机化最小割（Karger算法）

### 问题

在无向图中找到最小权的割（将图分成两部分的边集）。

### Monte Carlo算法

```cpp
#include <bits/stdc++.h>
using namespace std;

// Karger随机化最小割算法
struct Edge {
    int u, v;
    int w;
};

class RandomMinCut {
private:
    mt19937 rng;
    
    // 并查集
    vector<int> parent, rank;
    
    int find(int x) {
        if (parent[x] != x) {
            parent[x] = find(parent[x]);
        }
        return parent[x];
    }
    
    void unite(int x, int y) {
        x = find(x);
        y = find(y);
        if (x == y) return;
        
        if (rank[x] < rank[y]) {
            parent[x] = y;
        } else if (rank[x] > rank[y]) {
            parent[y] = x;
        } else {
            parent[y] = x;
            rank[x]++;
        }
    }
    
public:
    RandomMinCut() {
        random_device rd;
        rng = mt19937(rd());
    }
    
    // Karger算法：随机合并顶点
    int minCut(int n, vector<Edge> edges) {
        parent.resize(n);
        rank.assign(n, 0);
        for (int i = 0; i < n; i++) parent[i] = i;
        
        int currentN = n;
        vector<Edge> remaining = edges;
        
        while (currentN > 2) {
            // 随机选择一条边
            uniform_int_distribution<int> dist(0, remaining.size() - 1);
            int idx = dist(rng);
            Edge e = remaining[idx];
            
            // 合并边的两个端点
            int u = find(e.u);
            int v = find(e.v);
            if (u != v) {
                unite(u, v);
                currentN--;
            }
            
            // 移除自环边
            vector<Edge> newRemaining;
            for (auto& edge : remaining) {
                int fu = find(edge.u);
                int fv = find(edge.v);
                if (fu != fv) {
                    newRemaining.push_back({fu, fv, edge.w});
                }
            }
            remaining = move(newRemaining);
        }
        
        // 统计割的权重
        int cutWeight = 0;
        for (auto& e : remaining) {
            cutWeight += e.w;
        }
        return cutWeight;
    }
    
    // 多次运行取最小（提高成功率）
    int minCutRepeated(int n, vector<Edge> edges, int trials) {
        int best = INT_MAX;
        for (int i = 0; i < trials; i++) {
            best = min(best, minCut(n, edges));
        }
        return best;
    }
};
```

### 成功率分析

Karger算法的成功率约为 $1/n^2$。通过重复 $n^2$ 次可将失败概率降到可接受范围。

## 随机化跳跃表（Skip List）

### 概念

跳跃表是一种概率数据结构，实现 $O(\log n)$ 期望复杂度的搜索、插入、删除。

```cpp
#include <bits/stdc++.h>
using namespace std;

// 随机化跳跃表实现
template<typename T>
class SkipList {
private:
    struct Node {
        T value;
        vector<Node*> forward;  // 每层的前向指针
        Node(T val, int level) : value(val), forward(level + 1, nullptr) {}
    };
    
    Node* header;
    int maxLevel;
    double probability;
    mt19937 rng;
    
    // 随机生成层数
    int randomLevel() {
        int level = 0;
        uniform_real_distribution<double> dist(0.0, 1.0);
        while (dist(rng) < probability && level < maxLevel) {
            level++;
        }
        return level;
    }
    
public:
    SkipList(int maxLvl = 16, double p = 0.5) 
        : maxLevel(maxLvl), probability(p) {
        random_device rd;
        rng = mt19937(rd());
        header = new Node(T{}, maxLevel);
    }
    
    bool search(T target) {
        Node* current = header;
        for (int i = maxLevel; i >= 0; i--) {
            while (current->forward[i] && current->forward[i]->value < target) {
                current = current->forward[i];
            }
        }
        current = current->forward[0];
        return current && current->value == target;
    }
    
    void insert(T value) {
        Node* current = header;
        vector<Node*> update(maxLevel + 1, nullptr);
        
        for (int i = maxLevel; i >= 0; i--) {
            while (current->forward[i] && current->forward[i]->value < value) {
                current = current->forward[i];
            }
            update[i] = current;
        }
        
        int level = randomLevel();
        Node* newNode = new Node(value, level);
        
        for (int i = 0; i <= level; i++) {
            newNode->forward[i] = update[i]->forward[i];
            update[i]->forward[i] = newNode;
        }
    }
    
    bool remove(T value) {
        Node* current = header;
        vector<Node*> update(maxLevel + 1, nullptr);
        
        for (int i = maxLevel; i >= 0; i--) {
            while (current->forward[i] && current->forward[i]->value < value) {
                current = current->forward[i];
            }
            update[i] = current;
        }
        
        current = current->forward[0];
        if (!current || current->value != value) return false;
        
        for (int i = 0; i <= maxLevel; i++) {
            if (update[i]->forward[i] != current) break;
            update[i]->forward[i] = current->forward[i];
        }
        
        delete current;
        return true;
    }
};
```

**跳跃表高度**：期望 $O(\log n)$，因为每层被"提升"的概率为 $p$。

## 随机化负载均衡

### 一致性哈希

```cpp
#include <bits/stdc++.h>
using namespace std;

// 一致性哈希（简化实现）
class ConsistentHash {
private:
    struct Node {
        string key;
        int virtualNodes;
    };
    
    vector<Node> physicalNodes;
    sorted_map<int, string> hashRing;  // 哈希值到节点key的映射
    mt19937 rng;
    
    long long hash(const string& str) {
        // 简单的哈希函数
        hash<string> h;
        return h(str);
    }
    
public:
    void addNode(const string& key, int vNodeCount = 100) {
        for (int i = 0; i < vNodeCount; i++) {
            string vNodeKey = key + "#" + to_string(i);
            int hashVal = hash(vNodeKey);
            hashRing[hashVal] = key;
        }
    }
    
    string getNode(const string& dataKey) {
        int hashVal = hash(dataKey);
        auto it = hashRing.lower_bound(hashVal);
        
        if (it == hashRing.end()) {
            it = hashRing.begin();  // 环绕到开头
        }
        
        return it->second;
    }
};
```

## 概率分析基础

### 指示器随机变量

设 $X_i$ 为第 $i$ 个元素在正确位置的指示器，则：

$$E[X] = \sum_{i=1}^{n} E[X_i] = \sum_{i=1}^{n} \frac{1}{n} = 1$$

### 期望的线性性质

对于随机化快排的期望比较次数：

$$E[T(n)] = E\left[\frac{1}{n} \sum_{k=1}^{n-1} (T(k) + T(n-k-1) + n)\right] = O(n \log n)$$

## 应用场景

| 应用 | 算法类型 | 用途 |
|------|----------|------|
| 排序 | Las Vegas | 随机化快排 |
| 选择 | Las Vegas/Monte Carlo | Quickselect/BFPRT |
| 图论 | Monte Carlo | 最小割(Karger) |
| 数据结构 | Las Vegas | 跳跃表 |
| 负载均衡 | Monte Carlo | 一致性哈希 |
| 密码学 | Monte Carlo | 随机数生成 |
| 机器学习 | Monte Carlo | 随机梯度下降 |

## 参考文献

[^1]: Introduction to Algorithms (CLRS) - Chapter 5: Probabilistic Analysis
[^2]: Probability and Computing - Michael Mitzenmacher, Eli Upfal
[^3]: OI Wiki - 随机算法 https://oi-wiki.org/ds/rand/

---

*本页面内容基于算法经典教材整理*
