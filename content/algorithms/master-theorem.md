---
title: Master Theorem
date: 2026-04-12
description: 递归式求解的Master定理，用于分析分治算法的时间复杂度
tags:
  - complexity-analysis
  - algorithms
  - recursion
draft: false
permalink:
---

## 1. 递归式简介

### 1.1 什么是递归式

递归式（Recurrence Relation）是用于定义序列的数学方程，其中序列的每一项都通过前面若干项来定义。递归式在计算机科学中广泛用于描述**分治算法**的时间复杂度。

一个递归式包含两个部分：

- **基础情况（Base Case）**：递归终止条件
- **递归情况（Recursive Case）**：将问题分解为更小的子问题

### 1.2 常见递归式示例

#### 斐波那契数列

最经典的递归式例子是斐波那契数列：

$$
F(n) = F(n-1) + F(n-2), \quad F(0) = 0, F(1) = 1
$$

```cpp
int fib(int n) {
    if (n <= 1) return n;
    return fib(n - 1) + fib(n - 2);
}
```

#### 归并排序

归并排序的递归式为：

$$
T(n) = 2T\left(\frac{n}{2}\right) + O(n)
$$

```cpp
void mergeSort(vector<int>& arr, int left, int right) {
    if (left >= right) return;
    int mid = left + (right - left) / 2;
    mergeSort(arr, left, mid);
    mergeSort(arr, mid + 1, right);
    merge(arr, left, mid, right);  // O(n) 合并操作
}
```

---

## 2. 代入法求解递归式

代入法（Substitution Method）是求解递归式最基本的方法，通过**猜测**解的形式，然后用数学归纳法证明。

### 2.1 方法步骤

1. 猜测解的形式
2. 假设解对某个常数 $c$ 成立
3. 用数学归纳法证明

### 2.2 示例：证明 $T(n) = 2T(n/2) + n$ 的解为 $O(n \log n)$

**猜测**：$T(n) \leq c \cdot n \log n$（$c$ 为某个正常数）

**基础情况**：设 $n \leq 2$，则 $T(n) \leq O(1) \leq c \cdot n \log n$（当 $n$ 足够大时成立）

**归纳假设**：假设对所有小于 $n$ 的值，不等式成立，即 $T(k) \leq c \cdot k \log k$

**归纳证明**：

$$
\begin{aligned}
T(n) &= 2T(n/2) + n \\
&\leq 2 \cdot c \cdot \frac{n}{2} \log\left(\frac{n}{2}\right) + n \\
&= c \cdot n (\log n - \log 2) + n \\
&= c \cdot n \log n - c \cdot n + n \\
&= c \cdot n \log n - (c - 1)n
\end{aligned}
$$

当 $c \geq 1$ 时，$T(n) \leq c \cdot n \log n$ 成立。

**得证**：$T(n) = O(n \log n)$

### 2.3 代入法的局限性

代入法需要**猜测**解的形式，对于某些递归式难以猜测正确解。这时我们可以使用**递归树方法**来辅助猜测。

---

## 3. Master 定理

Master 定理（Master Theorem）提供了一种直接求解形如以下递归式的方法：

$$
T(n) = aT\left(\frac{n}{b}\right) + f(n)
$$

其中：
- $a \geq 1$：子问题数量
- $b > 1$：子问题规模的缩减比例
- $f(n)$：分解和合并子问题的代价

### 3.1 Master 定理的三种情况

令 $n^{\log_b a}$ 为**关键项**，比较 $f(n)$ 与 $n^{\log_b a}$ 的相对增长速度：

| 情况 | 条件 | 结论 |
|------|------|------|
| **情况 1** | $f(n) = O\left(n^{\log_b a - \varepsilon}\right)$，其中 $\varepsilon > 0$ | $T(n) = \Theta\left(n^{\log_b a}\right)$ |
| **情况 2** | $f(n) = \Theta\left(n^{\log_b a} \log^k n\right)$，其中 $k \geq 0$ | $T(n) = \Theta\left(n^{\log_b a} \log^{k+1} n\right)$ |
| **情况 3** | $f(n) = \Omega\left(n^{\log_b a + \varepsilon}\right)$，其中 $\varepsilon > 0$ | $T(n) = \Theta(f(n))$ |

**情况 2 的关键**：当 $f(n)$ 与 $n^{\log_b a}$ **多项式级别相等**时（差一个对数因子），解需要再乘以一个对数因子。

### 3.2 Master 定理的几何直观

```
         n                    n
        / \                  / \
       /   \                /   \
      /     \              /     \
    n/2     n/2          n/4     n/4
    / \     / \          / \     / \
   /   \   /   \        /   \   /   \
  ...  ...       =>    ...  ...  ...
  
  每层代价: f(n)       子问题数: a
  子问题规模: n/b     深度: log_b n
```

每层处理的代价为 $f(n)$，共 $\log_b n$ 层，共有 $a^{\log_b n} = n^{\log_b a}$ 个叶子节点。

---

## 4. 经典应用示例

### 4.1 二分查找

**递归式**：

$$
T(n) = T\left(\frac{n}{2}\right) + O(1)
$$

**参数**：$a = 1$，$b = 2$，$f(n) = O(1)$

**分析**：

$$
n^{\log_b a} = n^{\log_2 1} = n^0 = 1
$$

比较 $f(n) = O(1)$ 与 $n^{\log_b a} = 1$：

- $f(n) = \Theta(1) = \Theta\left(n^{\log_2 1}\right)$

属于情况 2（$k = 0$），所以：

$$
T(n) = \Theta\left(n^{\log_2 1} \log^{0+1} n\right) = \Theta(\log n)
$$

```cpp
int binarySearch(vector<int>& arr, int target, int left, int right) {
    if (left > right) return -1;
    int mid = left + (right - left) / 2;
    if (arr[mid] == target) return mid;
    else if (arr[mid] < target) 
        return binarySearch(arr, target, mid + 1, right);
    else 
        return binarySearch(arr, target, left, mid - 1);
}
```

### 4.2 归并排序

**递归式**：

$$
T(n) = 2T\left(\frac{n}{2}\right) + O(n)
$$

**参数**：$a = 2$，$b = 2$，$f(n) = O(n)$

**分析**：

$$
n^{\log_b a} = n^{\log_2 2} = n^1 = n
$$

比较 $f(n) = O(n)$ 与 $n^{\log_b a} = n$：

- $f(n) = \Theta(n) = \Theta\left(n^{\log_2 2}\right)$

属于情况 2（$k = 0$），所以：

$$
T(n) = \Theta\left(n^{\log_2 2} \log^{0+1} n\right) = \Theta(n \log n)
$$

### 4.3 Strassen 矩阵乘法

Strassen 算法将两个 $n \times n$ 矩阵乘法分解为 7 个子矩阵乘法：

**递归式**：

$$
T(n) = 7T\left(\frac{n}{2}\right) + O(n^2)
$$

**参数**：$a = 7$，$b = 2$，$f(n) = O(n^2)$

**分析**：

$$
n^{\log_b a} = n^{\log_2 7} \approx n^{2.807}
$$

比较 $f(n) = O(n^2)$ 与 $n^{\log_2 7} \approx n^{2.807}$：

- $n^2 = O\left(n^{\log_2 7 - \varepsilon}\right)$，其中 $\varepsilon \approx 0.807 > 0$

属于情况 1，所以：

$$
T(n) = \Theta\left(n^{\log_2 7}\right) \approx \Theta\left(n^{2.807}\right)
$$

这比朴素的 $O(n^3)$ 矩阵乘法更快。

---

## 5. 扩展 Master 定理

标准 Master 定理的情况 2 仅适用于 $f(n)$ 与 $n^{\log_b a}$ **多项式级别相等**的情况。扩展 Master 定理（Akra-Bazzi 定理的特例）可以处理**多对数因子**的情况。

### 5.1 扩展形式

| 情况 | 条件 | 结论 |
|------|------|------|
| **情况 2a** | $f(n) = \Theta\left(n^{\log_b a} \log^k n\right)$，其中 $k \geq 0$ | $T(n) = \Theta\left(n^{\log_b a} \log^{k+1} n\right)$ |
| **情况 2b** | $f(n) = \Theta\left(n^{\log_b a} (\log n)^{k}\right)$，其中 $k \geq 0$ | $T(n) = \Theta\left(n^{\log_b a} (\log n)^{k+1}\right)$ |

### 5.2 示例：递归式 $T(n) = 2T(n/2) + O(n \log n)$

**参数**：$a = 2$，$b = 2$，$f(n) = O(n \log n)$

**分析**：

$$
n^{\log_b a} = n^{\log_2 2} = n
$$

$f(n) = O(n \log n) = \Theta\left(n^{\log_2 2} \log n\right)$，属于情况 2a（$k = 1$），所以：

$$
T(n) = \Theta\left(n \log^{1+1} n\right) = \Theta(n \log^2 n)
$$

---

## 6. Master 定理的局限性

Master 定理并非万能，以下情况**不能**直接使用：

| 限制 | 示例 | 说明 |
|------|------|------|
| 子问题规模不同 | $T(n) = T(n-1) + O(n)$ | 不是 $n/b$ 形式 |
| $f(n)$ 不是多项式 | $T(n) = 2T(n/2) + 2^n$ | $f(n)$ 增长过快 |
| $a$ 不是常数 | $T(n) = n! \cdot T(n/2) + O(1)$ | $a$ 必须为常数 |
| 缺少多项式级别差距 | $T(n) = 2T(n/2) + \frac{n}{\log n}$ | 介于情况 1 和 2 之间 |

对于无法直接应用 Master 定理的递归式，可以使用**递归树法**或**Akra-Bazzi 定理**求解。

---

## 7. 总结

| 方法 | 适用场景 | 特点 |
|------|----------|------|
| 代入法 | 所有递归式 | 需要猜测解的形式 |
| 递归树法 | 复杂递归式 | 直观，但计算繁琐 |
| Master 定理 | $T(n) = aT(n/b) + f(n)$ | 快速得到精确答案 |
| Akra-Bazzi 定理 | 更一般的递归式 | $T(x) = \sum a_i T(b_i x) + g(x)$ |

掌握这些技术，可以快速分析大多数分治算法的时间复杂度，从而更好地理解算法效率。

---

## 参考资料

[^1]: Cormen, T. H., Leiserson, C. E., Rivest, R. L., & Stein, C. *Introduction to Algorithms* (第三版). MIT Press.

[^2]: GeeksforGeeks. *Analysis of Algorithm | Set 4 (Solving Recurrences)*. https://www.geeksforgeeks.org/analysis-algorithm-set-4-solving-recurrences/

[^3]: 清华大学《数据结构与算法》课程讲义。
