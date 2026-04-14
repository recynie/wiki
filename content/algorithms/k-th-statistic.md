---
title: 第k顺序统计量与选择算法
date: 2026-04-06
description: 选择第k小/第k大元素的高效算法：QuickSelect与BFPRT（Median of Medians），最坏O(n)复杂度的选择
tags:
  - selection
  - k-th-statistic
  - algorithm
  - quick-select
  - bfprt
draft: false
permalink:
---

## 概述

**第k顺序统计量（K-th Order Statistic）** 是指在 $n$ 个元素中找出第 $k$ 小的元素。[^1]

这个问题与排序不同：我们只需要找到第 $k$ 小的元素，而不需要对所有元素排序。

| 问题 | 时间复杂度（下界） |
|------|------------------|
| 完全排序 | $O(n \log n)$ |
| 找第 $k$ 小 | $O(n)$（期望） |

## QuickSelect（快速选择）

### 算法思想

类似快速排序的partition过程，每次划分后确定pivot的位置：
- 若 pivot 正好是第 $k$ 个，则找到答案
- 若 $k$ 在 pivot 左侧，则在左半部分继续查找
- 若 $k$ 在 pivot 右侧，则在右半部分继续查找

### 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

int partition(vector<int>& a, int l, int r) {
    int pivot = a[r];
    int i = l;
    for (int j = l; j < r; j++) {
        if (a[j] <= pivot) {
            swap(a[i], a[j]);
            i++;
        }
    }
    swap(a[i], a[r]);
    return i;
}

int quickSelect(vector<int>& a, int l, int r, int k) {
    // k 是相对于 l 的偏移量，即找第 k 小的元素
    if (l == r) return a[l];
    
    int pos = partition(a, l, r);
    int cnt = pos - l + 1;  // pivot 是第 cnt 小的元素
    
    if (k == cnt) return a[pos];
    else if (k < cnt) return quickSelect(a, l, pos - 1, k);
    else return quickSelect(a, pos + 1, r, k - cnt);
}

// 封装接口
int findKthSmallest(vector<int>& a, int k) {
    // k 从 1 开始
    return quickSelect(a, 0, a.size() - 1, k);
}
```

### 复杂度分析

- **期望时间复杂度**：$O(n)$
  - 每次划分期望将问题规模减少约25%，服从线性递归 $T(n) = T(n/4) + O(n) = O(n)$
- **最坏时间复杂度**：$O(n^2)$（当 pivot 总是最小/最大时）

### 随机化版本

为避免最坏情况，常使用随机 pivot：

```cpp
int randomPartition(vector<int>& a, int l, int r) {
    int k = l + rand() % (r - l + 1);
    swap(a[k], a[r]);
    return partition(a, l, r);
}
```

## BFPRT（Median of Medians）

### 算法思想

BFPRT 由 Blum、Floyd、Pratt、Rivest、Tarjan 于1973年提出，保证**最坏情况** $O(n)$ 时间复杂度。

核心思想：选择更好的 pivot，而不是随机选择。

### 算法步骤

1. **分组**：将数组分成 $\lceil n/5 \rceil$ 组，每组5个元素（最后一组可能不足5个）
2. **组内排序并取中位数**：对每组排序，取中位数（共 $\lceil n/5 \rceil$ 个中位数）
3. **递归求中位数的中位数**：对所有中位数递归执行 BFPRT，得到 pivot
4. **用 pivot 划分**：根据 pivot 进行 partition
5. **递归查找**：根据 pivot 位置决定在左侧还是右侧继续

### 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

const int INF = 1e9;

// 获取中位数的下标（假设 l 到 r 元素已按5组划分）
int getMedianIndex(int l, int r) {
    // 对于5个元素的组，返回中位数下标
    // 实际实现中需要根据 l 和 r 的关系计算
    return l + (r - l) / 2;
}

int medianOfMedians(vector<int>& a, int l, int r) {
    if (r - l < 5) {
        sort(a.begin() + l, a.begin() + r + 1);
        return l + (r - l) / 2;
    }
    
    // 求每组中位数，放到前面
    int cnt = 0;
    for (int i = l; i <= r; i += 5) {
        int left = i;
        int right = min(i + 4, r);
        sort(a.begin() + left, a.begin() + right + 1);
        int mid = left + (right - left) / 2;
        swap(a[cnt + l], a[mid]);
        cnt++;
    }
    
    // 递归求中位数的中位数
    int medPos = l + cnt / 2;
    return medianOfMedians(a, l, l + cnt - 1);
}

int bfprt(vector<int>& a, int l, int r, int k) {
    if (l == r) return a[l];
    
    // 求出中位数的中位数作为 pivot
    int pivotIdx = medianOfMedians(a, l, r);
    swap(a[pivotIdx], a[r]);  // 将 pivot 放到末尾
    
    int i = l, j = r - 1;
    while (i <= j) {
        while (a[i] < a[r]) i++;
        while (j >= l && a[j] >= a[r]) j--;
        if (i < j) swap(a[i], a[j]);
    }
    swap(a[i], a[r]);
    
    int cnt = i - l + 1;
    if (k == cnt) return a[i];
    else if (k < cnt) return bfprt(a, l, i - 1, k);
    else return bfprt(a, i + 1, r, k - cnt);
}
```

### 复杂度分析

- **最坏时间复杂度**：$O(n)$
- **空间复杂度**：$O(\log n)$（递归栈）
- **常数因子**：比 QuickSelect 大，但保证最坏情况

## 算法对比

| 算法 | 期望时间 | 最坏时间 | 额外空间 | 稳定性 |
|------|---------|---------|---------|--------|
| QuickSelect（随机） | $O(n)$ | $O(n^2)$ | $O(1)$ | 不稳定 |
| BFPRT | $O(n)$ | $O(n)$ | $O(\log n)$ | 不稳定 |
| 排序后选择 | $O(n \log n)$ | $O(n \log n)$ | $O(1)$ | 稳定 |

## 应用场景

1. **Top-K 问题**：找最大的 $k$ 个元素（$O(n)$ 而非 $O(n \log k)$）
2. **中位数计算**：$k = n/2$
3. **流数据中的顺序统计**：数据流太大无法全部存入内存
4. **快速查分**：快速找到分裂点

## 例题：数组中出现次数超过一半的数

**题目**：数组中有一个数出现次数超过一半，找出这个数。

**思路**：利用中位数特性，超过一半的数一定是中位数。可以用 BFPRT $O(n)$ 找出。

```cpp
int majorityElement(vector<int>& nums) {
    int n = nums.size();
    return bfprt(nums, 0, n - 1, n / 2 + 1);
}
```

## 参考

[^1]: 本内容参考 [OI-Wiki 第k小/第k大](https://oi-wiki.org/graph/k-th-statistic/)，内容经过验证和扩展。

---

*Last updated: 2026-04-06*