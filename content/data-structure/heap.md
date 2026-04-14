---
title: 堆（Heap）
date: 2026-03-31
description: 堆（Heap）数据结构与优先队列
tags:
  - heap
  - data-structure
  - priority-queue
draft: false
permalink:
---

## 定义

堆（Heap）是一种特殊的**完全二叉树**数据结构，它满足堆属性：

- **最大堆**（Max Heap）：每个节点的值都 $\geq$ 其子节点的值，堆顶是最大值
- **最小堆**（Min Heap）：每个节点的值都 $\leq$ 其子节点的值，堆顶是最小值[^1]

由于堆是一种完全二叉树，我们可以使用**数组**高效地存储堆，而不需要使用指针。每个节点在数组中的索引存在简单的数学关系：

- 父节点索引：$parent = (i - 1) / 2$
- 左子节点索引：$left = 2 \times i + 1$
- 右子节点索引：$right = 2 \times i + 2$

## 核心特性

### 时间复杂度

| 操作 | 时间复杂度 |
|------|------------|
| push（插入） | $O(\log n)$ |
| pop（删除堆顶） | $O(\log n)$ |
| top（获取堆顶） | $O(1)$ |
| heapify（批量建堆） | $O(n)$ |

### 空间复杂度

- 空间复杂度：$O(n)$

## 基本操作

### 堆化（Heapify）

堆化是将一个无序数组调整为堆结构的过程。从最后一个非叶子节点开始，向下调整每个节点。

```cpp
#include <bits/stdc++.h>
using namespace std;

// 向下调整：将索引为 i 的节点向下调整到正确位置
void heapify(vector<int>& arr, int n, int i) {
    int largest = i;        // 假设当前节点最大
    int left = 2 * i + 1;   // 左子节点
    int right = 2 * i + 2;  // 右子节点
    
    // 比较左子节点
    if (left < n && arr[left] > arr[largest]) {
        largest = left;
    }
    
    // 比较右子节点
    if (right < n && arr[right] > arr[largest]) {
        largest = right;
    }
    
    // 如果最大节点不是当前节点，交换并继续向下调整
    if (largest != i) {
        swap(arr[i], arr[largest]);
        heapify(arr, n, largest);
    }
}

// 批量建堆：从最后一个非叶子节点向上逐步堆化
void buildHeap(vector<int>& arr) {
    int n = arr.size();
    // 最后一个非叶子节点的索引为 (n/2 - 1)
    for (int i = n / 2 - 1; i >= 0; i--) {
        heapify(arr, n, i);
    }
}
```

### 插入操作（push）

```cpp
// 向堆中插入元素
void heapPush(vector<int>& arr, int val) {
    arr.push_back(val);
    int i = arr.size() - 1;
    
    // 向上调整
    while (i > 0) {
        int parent = (i - 1) / 2;
        if (arr[i] <= arr[parent]) break;  // 最小堆：arr[i] >= arr[parent]
        swap(arr[i], arr[parent]);
        i = parent;
    }
}
```

### 删除堆顶操作（pop）

```cpp
// 删除堆顶元素
void heapPop(vector<int>& arr) {
    if (arr.empty()) return;
    
    int n = arr.size();
    swap(arr[0], arr[n - 1]);  // 将堆顶与最后一个元素交换
    arr.pop_back();             // 删除原堆顶元素
    heapify(arr, arr.size(), 0);  // 向下调整
}
```

## STL 优先队列

C++ STL 提供了 `priority_queue` 模板类，实现了堆的所有功能。

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    // 默认是最大堆
    priority_queue<int> pq;
    pq.push(30);
    pq.push(10);
    pq.push(50);
    pq.push(20);
    
    cout << "最大堆 top: " << pq.top() << endl;  // 输出: 50
    pq.pop();  // 删除堆顶 50
    
    // 创建最小堆：使用 greater<int>
    priority_queue<int, vector<int>, greater<int>> minPq;
    minPq.push(30);
    minPq.push(10);
    minPq.push(50);
    minPq.push(20);
    
    cout << "最小堆 top: " << minPq.top() << endl;  // 输出: 10
    
    // 自定义比较函数（最大堆）
    priority_queue<pair<int, int>> pq2;
    pq2.push({1, 2});
    pq2.push({3, 4});
    pq2.top();  // {3, 4}
    
    return 0;
}
```

### priority_queue 成员函数

| 函数 | 功能 |
|------|------|
| `push(val)` | 插入元素 |
| `pop()` | 删除堆顶元素 |
| `top()` | 获取堆顶元素 |
| `size()` | 获取元素个数 |
| `empty()` | 判断是否为空 |

## 堆排序

利用堆的性质可以对数组进行排序，时间复杂度 $O(n \log n)$，空间复杂度 $O(1)$，但不稳定。

```cpp
#include <bits/stdc++.h>
using namespace std;

void heapSort(vector<int>& arr) {
    int n = arr.size();
    
    // 建堆：从最后一个非叶子节点向上
    for (int i = n / 2 - 1; i >= 0; i--) {
        // 向下调整
        int largest = i;
        int left = 2 * i + 1;
        int right = 2 * i + 2;
        
        if (left < n && arr[left] > arr[largest]) largest = left;
        if (right < n && arr[right] > arr[largest]) largest = right;
        
        if (largest != i) {
            swap(arr[i], arr[largest]);
            // 继续向下调整
            int j = largest;
            while (j < n / 2) {
                int left = 2 * j + 1;
                int right = 2 * j + 2;
                int largest = j;
                if (left < n && arr[left] > arr[largest]) largest = left;
                if (right < n && arr[right] > arr[largest]) largest = right;
                if (largest == j) break;
                swap(arr[j], arr[largest]);
                j = largest;
            }
        }
    }
    
    // 逐个取出堆顶
    for (int i = n - 1; i > 0; i--) {
        swap(arr[0], arr[i]);  // 将堆顶移到数组末尾
        // 重新堆化 0...i-1
        int j = 0;
        while (j < i / 2) {
            int left = 2 * j + 1;
            int right = 2 * j + 2;
            int largest = j;
            if (left < i && arr[left] > arr[largest]) largest = left;
            if (right < i && arr[right] > arr[largest]) largest = right;
            if (largest == j) break;
            swap(arr[j], arr[largest]);
            j = largest;
        }
    }
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    vector<int> arr = {64, 34, 25, 12, 22, 11, 90};
    heapSort(arr);
    
    for (int x : arr) cout << x << " ";
    cout << endl;
    return 0;
}
```

## 应用场景

### Top-K 问题

在一组数据中找出最大/最小的 K 个元素。使用最小堆保存 K 个最大元素，或最大堆保存 K 个最小元素。[^2]

```cpp
// 找出第 K 大的元素
int findKthLargest(vector<int>& nums, int k) {
    priority_queue<int, vector<int>, greater<int>> minPq;
    
    for (int num : nums) {
        minPq.push(num);
        if (minPq.size() > k) {
            minPq.pop();
        }
    }
    
    return minPq.top();
}
```

### 合并有序链表

使用最小堆将多个有序链表合并成一个有序链表。

```cpp
ListNode* mergeKLists(vector<ListNode*>& lists) {
    priority_queue<pair<int, ListNode*>, 
                   vector<pair<int, ListNode*>>, 
                   greater<pair<int, ListNode*>>> pq;
    
    for (auto list : lists) {
        if (list) pq.push({list->val, list});
    }
    
    ListNode dummy(0);
    ListNode* cur = &dummy;
    
    while (!pq.empty()) {
        auto [val, node] = pq.top();
        pq.pop();
        cur->next = node;
        cur = cur->next;
        if (node->next) pq.push({node->next->val, node->next});
    }
    
    return dummy.next;
}
```

### 优先队列调度

任务调度、进程优先级管理、中位数维护等。

### 对顶堆技巧

使用两个堆（最大堆 + 最小堆）动态维护数据流的中位数。

## 参考资料

[^1]: 堆首先由威廉姆斯（J. W. J. Williams）在1964年描述，用于实现堆排序算法。

[^2]: Top-K 问题的朴素解法是排序 $O(n \log n)$，而堆化方法只需 $O(n \log k)$，在 K 较小时效率更高。
