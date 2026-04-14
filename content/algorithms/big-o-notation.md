---
title: Big-O Notation
date: 2026-04-12
description: 算法复杂度分析中的渐进记号，重点讲解Big-O的含义与应用
tags:
  - complexity-analysis
  - algorithms
  - theory
draft: false
permalink:
---

## 概述

在计算机科学中，**算法复杂度分析**是评估算法性能的核心工具。当我们说一个算法"快"或"慢"时，需要一个客观、可量化的标准来描述这种性能差异。Big-O 记号正是这样一种数学工具，它用于描述算法运行时间或空间需求随输入规模增长的趋势。

复杂度分析之所以重要，是因为它：

- 让我们能够在不了解具体硬件的情况下比较算法性能
- 揭示了算法在处理大规模数据时的行为特征
- 帮助我们在设计系统时做出明智的算法选择

本文将系统讲解 Big-O 记号的数学定义、常见复杂度等级、如何从代码分析复杂度，以及常见的分析误区。

## 数学定义

### 形式化定义

设 $f(n)$ 和 $g(n)$ 是定义在自然数集合上的函数，若存在正常数 $c > 0$ 和 $n_0$，使得对于所有 $n \geq n_0$，都有：

$$
f(n) \leq c \cdot g(n)
$$

则称 **$f(n) = O(g(n))$**，读作"$f(n)$ 是 $g(n)$ 的大O"。

这一定义的核心含义是：**$f(n)$ 的增长速度不会超过 $g(n)$ 的某个常数倍**。换言之，$g(n)$ 给出了 $f(n)$ 增长的上界。

### 直观理解

从函数图像的角度来看，$f(n) = O(g(n))$ 意味着从某个点 $n_0$ 开始，$f(n)$ 的值始终位于 $c \cdot g(n)$ 曲线之下：

$$
\forall n \geq n_0: \quad f(n) \leq c \cdot g(n)
$$

其中常数 $c$ 的作用是"缩放" $g(n)$，使其始终位于 $f(n)$ 上方（允许一定的常数倍差异）。

### 其他渐进记号

除 Big-O 外，还有其他几种重要的渐进记号：

| 记号 | 含义 | 定义 |
|------|------|------|
| $\Theta(n)$ | 同阶 | $f(n) = \Theta(g(n)) \iff f(n) = O(g(n))$ 且 $f(n) = \Omega(g(n))$ |
| $\Omega(n)$ | 下界 | $f(n) = \Omega(g(n)) \iff \exists c > 0, \exists n_0: \forall n \geq n_0, f(n) \geq c \cdot g(n)$ |
| $o(n)$ | 低阶 | $f(n) = o(g(n)) \iff \lim_{n \to \infty} \frac{f(n)}{g(n)} = 0$ |

在算法分析中，Big-O 最常用，因为它描述的是算法性能的**上界**——即"不会比…更差"。

## 常见复杂度等级

从最优到最差，常见的时间复杂度排列如下：

$$
O(1) < O(\log n) < O(n) < O(n \log n) < O(n^2) < O(2^n)
$$

### O(1) — 常数时间

算法的执行时间与输入规模无关，始终为固定值。

```cpp
// 判断一个数的奇偶性
bool isEven(int n) {
    return n % 2 == 0;
}
```

### O(log n) — 对数时间

常见于每次迭代将问题规模减半的算法，如[[binary-search|二分查找]]。

```cpp
// 二分查找
int binarySearch(const vector<int>& arr, int target) {
    int left = 0, right = arr.size() - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```

### O(n) — 线性时间

遍历一次输入即可完成的算法，如[[../data-structure/prefix-sum|前缀和]]计算。

```cpp
// 求数组所有元素的和
long long sumArray(const vector<int>& arr) {
    long long sum = 0;
    for (int x : arr) {
        sum += x;
    }
    return sum;
}
```

### O(n log n) — 线性对数时间

许多高效排序算法（如[[merge-sort|归并排序]]、快速排序）的复杂度。

```cpp
// 归并排序的核心合并过程
void merge(vector<int>& arr, int left, int mid, int right) {
    vector<int> temp(right - left + 1);
    int i = left, j = mid + 1, k = 0;
    while (i <= mid && j <= right) {
        if (arr[i] <= arr[j]) temp[k++] = arr[i++];
        else temp[k++] = arr[j++];
    }
    while (i <= mid) temp[k++] = arr[i++];
    while (j <= right) temp[k++] = arr[j++];
    for (int p = 0; p < k; p++) arr[left + p] = temp[p];
}
```

### O(n²) — 平方时间

嵌套循环遍历的算法，如[[../sorting/insertion-sort|插入排序]]。

```cpp
// 冒泡排序
void bubbleSort(vector<int>& arr) {
    int n = arr.size();
    for (int i = 0; i < n - 1; i++) {
        for (int j = 0; j < n - i - 1; j++) {
            if (arr[j] > arr[j + 1]) {
                swap(arr[j], arr[j + 1]);
            }
        }
    }
}
```

### O(2ⁿ) — 指数时间

典型的暴力搜索或递归枚举，如计算斐波那契数列的朴素递归解法。

```cpp
// 朴素递归计算斐波那契数
long long fibonacci(int n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}
```

### 复杂度对比图

| n | O(1) | O(log n) | O(n) | O(n log n) | O(n²) | O(2ⁿ) |
|---|------|----------|------|------------|-------|-------|
| 10 | 1 | 3 | 10 | 33 | 100 | 1024 |
| 100 | 1 | 7 | 100 | 664 | 10,000 | 1.27×10³⁰ |
| 1000 | 1 | 10 | 1000 | 9,966 | 10⁶ | 溢出 |

从上表可以看出，**指数级算法在 n 稍大时即变得完全不可用**。

## 最好、平均、最坏情况

分析算法复杂度时，通常需要考虑三种情况：

### 定义

- **最好情况（Best Case）**：输入最有利时的复杂度，如数组已排序时快速排序的表现
- **平均情况（Average Case）**：随机输入下的期望复杂度
- **最坏情况（Worst Case）**：输入最不利时的复杂度，算法设计的主要依据

### 示例：数组查找

```cpp
// 在无序数组中查找目标元素
int find(const vector<int>& arr, int target) {
    for (int i = 0; i < arr.size(); i++) {
        if (arr[i] == target) return i;
    }
    return -1;
}
```

| 情况 | 条件 | 复杂度 |
|------|------|--------|
| 最好 | 目标在第一个位置 | O(1) |
| 平均 | 目标在任意位置概率均等 | O(n/2) = O(n) |
| 最坏 | 目标不在数组中或最后一位 | O(n) |

### 为什么关注最坏情况？

1. **确定性**：最坏情况是确定的边界，不会因输入分布而变化
2. **防御性**：保证算法在任何输入下都不会超过这个上界
3. **简化分析**：通常最坏情况的分析和最好、平均相比更直接

## 如何确定 Big-O——系统分析方法

分析一段代码的复杂度，核心原则是：**找到算法执行的基本操作数量与输入规模 n 之间的关系**。

### 步骤一：识别基本操作

确定算法中的"原子操作"——那些花费时间不变的操作，如：
- 算术运算（加、减、乘、除）
- 比较操作
- 赋值操作

### 步骤二：识别循环结构

循环是影响复杂度的最主要因素：
- **单层循环**：通常贡献 O(n)
- **嵌套循环**：嵌套深度决定多项式阶数
- **循环变量按倍数变化**：可能贡献 O(log n)

### 步骤三：应用求和规则

- **顺序语句**：复杂度取最大值（$O(\max(f, g))$）
- **嵌套循环**：复杂度相乘（$O(f \times g)$）
- **条件分支**：复杂度取各分支的最大值

### 实例分析

```cpp
void analyzeExample(vector<vector<int>>& matrix) {
    // 第1部分：O(n)
    int sum = 0;
    for (int x : matrix[0]) {
        sum += x;
    }
    
    // 第2部分：O(n²)
    for (int i = 0; i < matrix.size(); i++) {
        for (int j = 0; j < matrix[i].size(); j++) {
            cout << matrix[i][j] << " ";
        }
        cout << endl;
    }
    
    // 第3部分：O(n log n)
    sort(matrix[0].begin(), matrix[0].end());
}
```

整体复杂度为 $O(\max(n, n^2, n \log n)) = O(n^2)$。

### 更多示例

**示例1：双指针变体**

```cpp
// O(n)，每个元素最多被访问两次
int removeDuplicates(vector<int>& nums) {
    if (nums.empty()) return 0;
    int slow = 0;
    for (int fast = 1; fast < nums.size(); fast++) {
        if (nums[fast] != nums[slow]) {
            nums[++slow] = nums[fast];
        }
    }
    return slow + 1;
}
```

**示例2：递归复杂度（主定理）**

对于形如 $T(n) = aT(n/b) + f(n)$ 的递归方程：

- 若 $f(n) = O(n^{\log_b a - \epsilon})$，则 $T(n) = \Theta(n^{\log_b a})$
- 若 $f(n) = \Theta(n^{\log_b a})$，则 $T(n) = \Theta(n^{\log_b a} \log n)$
- 若 $f(n) = \Omega(n^{\log_b a + \epsilon})$，则 $T(n) = \Theta(f(n))$

归并排序的递归式 $T(n) = 2T(n/2) + O(n)$ 属于第二种情况，故 $T(n) = O(n \log n)$。[^1]

## 常见误区与陷阱

### 误区一：忽略常数因子

Big-O 记号忽略常数因子，因此 $O(2n)$ 和 $O(n)$ 是等价的。但这并不意味着：
- $O(n)$ 的算法一定比 $O(n^2)$ 的快（对于小规模输入，常数大的 $O(n)$ 可能更慢）
- 常数因子在实际应用中可能非常重要（尤其在嵌入式系统或高频交易中）

### 误区二：关注低阶项而忽略高阶项

```cpp
// 错误分析
void wrongAnalysis(int n) {
    // O(n²) 是主导项
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) { /* ... */ }
    }
    // 这部分 O(n) 可以忽略
    for (int i = 0; i < n; i++) { /* ... */ }
}
```

正确的分析：整体复杂度是 $O(n^2 + n) = O(n^2)$，低阶项 $n$ 在 n 趋向无穷时可忽略。

### 误区三：将性能问题归咎于算法复杂度

程序性能不仅取决于算法复杂度，还受：
- **缓存局部性**：遍历二维数组时，按行遍历通常比按列遍历快
- **系统负载**：CPU 调度、GC 等因素
- **实现质量**：同一算法，不同实现可能有数倍性能差异

### 误区四：混淆不同操作的代价

```cpp
// 看似 O(n)，但字符串拼接是 O(n) 每次
string concat(vector<string>& strs) {
    string result;
    for (string s : strs) {
        result = result + s;  // 每次拼接都复制已有内容
    }
    return result;
}
```

正确做法是使用 `string result; result.reserve(total_length);` 预分配空间，或使用 `stringstream`。

### 误区五：忘记考虑空间复杂度

有时为了降低时间复杂度，会增加空间复杂度（**空间换时间**），反之亦然。完整的算法分析应同时考虑：
- **时间复杂度**：算法运行所需步数
- **空间复杂度**：算法运行所需内存

## 空间复杂度

与时间复杂度类似，空间复杂度描述算法所需存储空间随输入规模增长的趋势。

### 常见空间复杂度

| 复杂度 | 名称 | 典型场景 |
|--------|------|----------|
| O(1) | 常数空间 | 原地算法，只用几个变量 |
| O(log n) | 对数空间 | 递归深度为 log n（如二分递归） |
| O(n) | 线性空间 | 大小为 n 的数组或向量 |
| O(n²) | 平方空间 | n×n 的二维矩阵 |

### 示例：快速排序 vs 归并排序

```cpp
// 快速排序 - O(log n) 额外空间（递归栈）
void quickSort(vector<int>& arr, int low, int high) {
    if (low < high) {
        int pi = partition(arr, low, high);
        quickSort(arr, low, pi - 1);
        quickSort(arr, pi + 1, high);
    }
}

// 归并排序 - O(n) 额外空间（临时数组）
void mergeSort(vector<int>& arr, int left, int right) {
    if (left < right) {
        int mid = left + (right - left) / 2;
        mergeSort(arr, left, mid);
        mergeSort(arr, mid + 1, right);
        merge(arr, left, mid, right);  // 需要 O(n) 临时空间
    }
}
```

## 实际应用指南

### 如何选择算法

| 输入规模 n | 可接受的复杂度 | 示例应用 |
|-----------|---------------|----------|
| n ≤ 10 | O(n!), O(2ⁿ) | 暴力搜索、生成全排列 |
| n ≤ 20 | O(2ⁿ) | 状态压缩 DP、枚举子集 |
| n ≤ 100 | O(n³) |  Floyd-Warshall、矩阵乘法 |
| n ≤ 1000 | O(n²) | 动态规划、图论算法 |
| n ≤ 10⁶ | O(n log n) | 排序、优先队列 |
| n ≤ 10⁷ | O(n) | 线性扫描、哈希表 |
| n 任意 | O(1) | 位运算、常数时间查找 |

### 优化思路

当算法复杂度过高时，考虑以下方向：

1. **数学化简**：观察问题是否有数学性质可利用
2. **数据预处理**：排序、建立索引等
3. **缓存结果**：避免重复计算（如 [[dynamic-planning|动态规划]]）
4. **近似算法**：在精度和速度间权衡（如 [[approximation-algorithms|近似算法]]）
5. **随机化**：期望复杂度更优的算法（如快速排序的随机化版本）

## 小结

Big-O 记号是描述算法复杂度的标准语言，其核心思想是**忽略常数和低阶项，只关注增长趋势**。掌握复杂度分析，需要：

1. 理解 $f(n) = O(g(n))$ 的数学定义
2. 熟记常见复杂度等级及其图像特征
3. 学会系统分析代码复杂度的步骤
4. 警惕常见的分析误区
5. 平衡时间与空间，必要时进行权衡

复杂度分析不是绝对的性能预测，而是一种**渐近的、上界的定性描述**。在实际工程中，它应与实验测量 Profiling 结合使用。

## 参考资料

[^1]: 本节关于递归复杂度分析的内容参考了 [GeeksforGeeks - Analysis of Algorithms | Recurrence Relation](https://www.geeksforgeeks.org/analysis-of-algorithms-recurrence-relations/) 中关于主定理的讲解。

[^2]: Stanford CS106B 课程材料中对算法复杂度基础概念的解释。

[^3]: Cormen, T. H., et al. *Introduction to Algorithms* (3rd ed.). MIT Press, 2009. （《算法导论》）
