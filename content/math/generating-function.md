---
title: 生成函数（Generating Function）
date: 2026-04-06
description: 生成函数详解：普通生成函数（OGF）与指数生成函数（EGF），利用生成函数解递推关系
tags:
  - generating-function
  - combinatorics
  - recurrence
draft: false
permalink:
---

## 概述

**生成函数（Generating Function）** 是组合数学中连接递推关系与封闭形式的有力工具。[^1]

设有一数列 $\{a_n\}$，则其**普通生成函数（Ordinary Generating Function, OGF）**定义为：

$$
G(x) = \sum_{n=0}^{\infty} a_n x^n
$$

**指数生成函数（Exponential Generating Function, EGF）**定义为：

$$
EG(x) = \sum_{n=0}^{\infty} a_n \frac{x^n}{n!}
$$

## 普通生成函数（OGF）

### 常见OGF公式

| 序列 | OGF | 收敛域 |
|------|-----|--------|
| $a_n = 1$ | $\frac{1}{1-x}$ | $|x| < 1$ |
| $a_n = n$ | $\frac{x}{(1-x)^2}$ | $|x| < 1$ |
| $a_n = \binom{n}{k}$ | $\frac{x^k}{(1-x)^{k+1}}$ | $|x| < 1$ |
| $a_n = \binom{n+k}{n}$ | $\frac{1}{(1-x)^{k+1}}$ | $|x| < 1$ |
| $a_n = r^n$ | $\frac{1}{1-rx}$ | $|x| < \frac{1}{|r|}$ |
| $a_n = \text{Catalan}_n$ | $\frac{1-\sqrt{1-4x}}{2x}$ | $\|x\| < \frac{1}{4}$ |

### 利用OGF解递推关系

**例：斐波那契数列**

斐波那契数列满足 $F_n = F_{n-1} + F_{n-2}$，初始 $F_0 = 0, F_1 = 1$。

设 $F(x) = \sum_{n=0}^{\infty} F_n x^n$，则：

$$
F(x) - F_0 - F_1 x = \sum_{n=2}^{\infty} F_n x^n = \sum_{n=2}^{\infty} (F_{n-1} + F_{n-2}) x^n = x(F(x) - F_0) + x^2 F(x)
$$

代入 $F_0 = 0, F_1 = 1$：

$$
F(x) - x = x F(x) + x^2 F(x) \Rightarrow F(x) = \frac{x}{1 - x - x^2}
$$

展开即可得到 $F_n$ 的封闭形式。

## 指数生成函数（EGF）

EGF 适用于**标记组合**问题，如有标号物体的排列。

### 常见EGF公式

| 序列 | EGF | 说明 |
|------|-----|------|
| $a_n = 1$ | $e^x$ | |
| $a_n = n!$ | $\frac{1}{1-x}$ | |
| $a_n = \text{Bell}_n$ | $e^{e^x - 1}$ | 贝尔数 |
| $a_n = \text{Stirling2}_n$ | $e^{e^x - 1}$ | 第二类斯特林数 |

## 卡特兰数（Catalan Numbers）

卡特兰数 $C_n = \frac{1}{n+1}\binom{2n}{n}$ 满足递推：

$$
C_n = \sum_{i=0}^{n-1} C_i C_{n-1-i}, \quad C_0 = 1
$$

其OGF满足：

$$
C(x) = \frac{1 - \sqrt{1-4x}}{2x}
$$

### 卡特兰数的应用

1. **合法括号序列**：$n$ 对括号的合法排列数
2. **二叉树**：$n+1$ 个叶子的二叉树数量
3. **Dyck路径**：从 $(0,0)$ 到 $(2n,0)$ 不越过 $x$ 轴的路径数
4. **凸多边形三角剖分**：$n+2$ 边凸多边形的三角剖分数

## 形式幂级数运算

### 加法与乘法

$$
(A(x) + B(x))_n = a_n + b_n
$$

$$
(A(x) \cdot B(x))_n = \sum_{i=0}^{n} a_i b_{n-i}
$$

### 逆运算

若 $A(x) \cdot B(x) = 1$，则 $B(x) = A(x)^{-1}$。

### 复合运算

$$
(F(G(x)))_n = \sum_{k=1}^{n} f_k \cdot (G(x))^n_k
$$

其中 $(G(x))^n_k$ 表示从 $G(x)^n$ 中取 $x^k$ 的系数。

## 多项式与生成函数

在竞赛中，生成函数常与**多项式算法**结合：

- 多项式乘法（FFT/NTT）加速卷积
- 多项式求逆
- 多项式开方
- 多项式ln/exp

这些技术在求递推序列的封闭形式时非常有用。

## 应用场景

1. **递推关系求解**：将递推转化为代数问题
2. **计数问题**：如项链计数、棋盘路径
3. **概率生成函数**：随机变量的分布
4. **线性递推加速**：利用特征多项式求 $n$ 项

## 参考

[^1]: 本内容参考 [OI-Wiki 多项式与生成函数](https://oi-wiki.org/math/poly/)，内容经过验证和扩展。

---

*Last updated: 2026-04-06*