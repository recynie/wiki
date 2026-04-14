---
title: 组合数学
date: 2026-04-03
description: 组合数学（Combinatorics）基础
tags:
  - combinatorics
  - math
draft: false
permalink:
---

## 排列组合基础 (Basics)

### 排列 (Permutation)

从 $n$ 个不同元素中取出 $r$ 个元素排成一列，称为**排列**，记作 $P(n,r)$ 或 $A(n,r)$：

$$
P(n,r) = \frac{n!}{(n-r)!} = n \times (n-1) \times \cdots \times (n-r+1)
$$

特殊情况：当 $r = n$ 时，$P(n,n) = n!$

### 组合 (Combination)

从 $n$ 个不同元素中取出 $r$ 个元素成一组（不考虑顺序），称为**组合**，记作 $C(n,r)$ 或 $\binom{n}{r}$：

$$
C(n,r) = \frac{n!}{r!(n-r)!}
$$

**重要性质**：$C(n,r) = C(n,n-r)$（对称性）

### 组合数的性质

组合数满足递推关系（Pascal's triangle / 杨辉三角）：

$$
C(n,k) = C(n-1,k) + C(n-1,k-1)
$$

由此可得：

- $C(n,0) = C(n,n) = 1$
- $C(n,1) = C(n,n-1) = n$
- $\sum_{i=0}^{n} C(n,i) = 2^n$

---

## 二项式定理 (Binomial Theorem)

$$
(a+b)^n = \sum_{k=0}^{n} C(n,k) \cdot a^k \cdot b^{n-k} = \sum_{k=0}^{n} \binom{n}{k} a^k b^{n-k}
$$

### 推论

- 令 $a=b=1$：$\sum_{k=0}^{n} C(n,k) = 2^n$
- 令 $a=1, b=-1$：$\sum_{k=0}^{n} (-1)^k C(n,k) = 0$

---

## 常用计数技巧 (Common Counting Techniques)

### 加法原理与乘法原理

**加法原理**：完成一件事有 $n$ 类方式，第 $i$ 类方式有 $a_i$ 种方法，则总方法数为：

$$
N = a_1 + a_2 + \cdots + a_n
$$

**乘法原理**：完成一件事需要 $n$ 个步骤，第 $i$ 步有 $a_i$ 种方法，则总方法数为：

$$
N = a_1 \times a_2 \times \cdots \times a_n
$$

### 容斥原理 (Inclusion-Exclusion Principle)

用于计算若干集合的并集大小。设 $A_1, A_2, \ldots, A_n$ 为有限集合，则：

$$
\left| \bigcup_{i=1}^{n} A_i \right| = \sum_{i} |A_i| - \sum_{i<j} |A_i \cap A_j| + \sum_{i<j<k} |A_i \cap A_j \cap A_k| - \cdots + (-1)^{n-1} |A_1 \cap \cdots \cap A_n|
$$

**简单例子**（三个集合）：

$$
|A \cup B \cup C| = |A| + |B| + |C| - |A \cap B| - |B \cap C| - |A \cap C| + |A \cap B \cap C|
$$

### 抽屉原理 (Pigeonhole Principle)

**简单形式**：如果把 $n+1$ 个物体放入 $n$ 个盒子，则至少有一个盒子中有两个或更多物体。

**广义形式**：如果把 $n$ 个物体放入 $k$ 个盒子，则至少有一个盒子中含有至少 $\lceil n/k \rceil$ 个物体。

**应用示例**：在 $n+1$ 个整数中，必有两个整数的差是 $n$ 的倍数。

---

## 递推关系 (Recurrence Relations)

### 斐波那契数列 (Fibonacci Sequence)

$$
F_0 = 0,\quad F_1 = 1,\quad F_n = F_{n-1} + F_{n-2} \quad (n \geq 2)
$$

前几项：$0, 1, 1, 2, 3, 5, 8, 13, 21, 34, \ldots$

**通项公式**（Binet公式）：

$$
F_n = \frac{1}{\sqrt{5}} \left( \varphi^n - \psi^n \right)
$$

其中 $\varphi = \frac{1+\sqrt{5}}{2}$，$\psi = \frac{1-\sqrt{5}}{2}$

### Catalan 数

Catalan 数 $C_n$ 满足：

$$
C_0 = 1,\quad C_{n+1} = \sum_{i=0}^{n} C_i \cdot C_{n-i} = \frac{C(2n,n)}{n+1}
$$

前几项：$1, 1, 2, 5, 14, 42, 132, 429, \ldots$

**组合意义**：
- 含有 $n+1$ 个叶子的满二叉树的个数
- $n$ 对括号的有效匹配数
- $n \times n$ 棋盘从左下角到右上角不穿过对角线的路径数

### Stirling 数

**第一类 Stirling 数** $s(n,k)$：将 $n$ 个元素分成 $k$ 个非空循环排列的方法数。

递推公式：

$$
s(n,k) = s(n-1,k-1) + (n-1) \cdot s(n-1,k)
$$

**第二类 Stirling 数** $S(n,k)$：将 $n$ 个元素分成 $k$ 个非空集合的方法数。

递推公式：

$$
S(n,k) = S(n-1,k-1) + k \cdot S(n-1,k)
$$

---

## 母函数初步 (Generating Function Basics)

母函数（生成函数）用于解决递推关系和计数问题。

### 普通母函数

序列 $\{a_n\}$ 的普通母函数为：

$$
G(x) = \sum_{n=0}^{\infty} a_n x^n
$$

**示例**：对于 Fibonacci 数列，母函数为：

$$
G(x) = \frac{x}{1-x-x^2}
$$

### 指数母函数

序列 $\{a_n\}$ 的指数母函数为：

$$
EG(x) = \sum_{n=0}^{\infty} a_n \frac{x^n}{n!}
$$

### 经典应用

用母函数可以证明组合恒等式，例如：

$$
(1+x)^n = \sum_{k=0}^{n} C(n,k) x^k
$$

---

## 组合数取模 (Modular Combinatorics)

### Lucas 定理

当 $p$ 为素数时，组合数取模可以用 Lucas 定理高效计算：

$$
C(n,m) \equiv \prod_{i=0}^{k} C(n_i, m_i) \pmod{p}
$$

其中 $n = n_k p^k + \cdots + n_1 p + n_0$，$m = m_k p^k + \cdots + m_1 p + m_0$ 是 $p$ 进制表示。

**适用场景**：$n, m$ 很大（甚至达到 $10^{18}$），但 $p$ 较小（如 $p = 10^9+7$）。

### 预处理阶乘与逆阶乘

在模 $p$ 下计算组合数，通常需要预计算阶乘和逆阶乘：

```cpp
const int MOD = 1e9 + 7;
const int MAXN = 2e6 + 5;

int64_t fact[MAXN], infact[MAXN];

int64_t mod_pow(int64_t a, int64_t b) {
    int64_t res = 1;
    while (b) {
        if (b & 1) res = res * a % MOD;
        a = a * a % MOD;
        b >>= 1;
    }
    return res;
}

void init_factorials(int n) {
    fact[0] = 1;
    for (int i = 1; i <= n; i++) {
        fact[i] = fact[i-1] * i % MOD;
    }
    infact[n] = mod_pow(fact[n], MOD - 2);  // Fermat's little theorem
    for (int i = n; i > 0; i--) {
        infact[i-1] = infact[i] * i % MOD;
    }
}

int64_t C(int n, int m) {
    if (n < 0 || m < 0 || m > n) return 0;
    return fact[n] * infact[m] % MOD * infact[n-m] % MOD;
}
```

**注意**：
- 当 $p$ 为素数时，$a^{-1} \equiv a^{p-2} \pmod{p}$（Fermat 小定理）
- 使用 Lucas 定理时，递归计算即可

---

## 参考资料

[^1]: [组合数学 - OI Wiki](https://oi-wiki.org/math/combinatorics/)

[^2]: [排列组合基础 - OI Wiki](https://oi-wiki.org/math/combinatorics/basic/)

[^3]: [Catalan数 - OI Wiki](https://oi-wiki.org/math/combinatorics/catalan/)

[^4]: [Stirling数 - OI Wiki](https://oi-wiki.org/math/combinatorics/stirling/)

[^5]: [生成函数 - OI Wiki](https://oi-wiki.org/math/generating-function/)
