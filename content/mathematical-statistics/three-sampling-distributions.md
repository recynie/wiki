---
title: 三大抽样分布 (The Three Sampling Distributions)
date: 2026-03-19
description: 基于正态总体的卡方分布、t分布和F分布，是假设检验的基石
tags:
  - mathematical-statistics
  - sampling-distribution
  - chi-square
  - t-distribution
  - f-distribution
draft: false
permalink:
---

# 三大抽样分布 (The Three Sampling Distributions)

三大抽样分布——卡方分布、$t$ 分布和 $F$ 分布——均基于正态总体衍生而来，是参数估计和假设检验的理论工具。

## 卡方分布（Chi-Square Distribution）

### 定义

设 $Z_1, Z_2, \ldots, Z_n$ 为独立同分布的标准正态随机变量 $N(0,1)$，则：

$$
\chi^2 = Z_1^2 + Z_2^2 + \cdots + Z_n^2 \sim \chi^2(n)
$$

服从**自由度为 $n$ 的卡方分布**，记作 $\chi^2(n)$。

### 概率密度函数

$\chi^2(n)$ 分布的概率密度函数为：

$$
f(x; n) = \frac{1}{2^{n/2}\Gamma(n/2)}x^{n/2-1}e^{-x/2}, \quad x > 0
$$

其中 $\Gamma(\cdot)$ 为 Gamma 函数。

### 图像特征

- **定义域**：$x > 0$（右偏分布）
- **形状**：随自由度 $n$ 增大，分布趋于正态
- $n=1$ 时为标准正态平方，密度在原点附近奇异
- $n \geq 3$ 时，密度函数先增后减，在 $x = n-2$ 处取得峰值

### 可加性

若 $X \sim \chi^2(m)$，$Y \sim \chi^2(n)$，且独立，则：

$$
X + Y \sim \chi^2(m+n)
$$

### 与样本均值、样本方差的关系

设 $X_1, X_2, \ldots, X_n \sim N(\mu, \sigma^2)$，样本均值为 $\bar{X}$，样本方差为 $S^2$，则：

$$
\frac{(n-1)S^2}{\sigma^2} \sim \chi^2(n-1)
$$

---

## $t$ 分布（Student's $t$ Distribution）

### 定义

设 $Z \sim N(0,1)$，$U \sim \chi^2(n)$，且 $Z$ 与 $U$ 独立，则：

$$
T = \frac{Z}{\sqrt{U/n}} \sim t(n)
$$

服从**自由度为 $n$ 的学生 $t$ 分布**，记作 $t(n)$。

### 概率密度函数

$$
f(t; n) = \frac{\Gamma((n+1)/2)}{\sqrt{n\pi}\Gamma(n/2)}\left(1+\frac{t^2}{n}\right)^{-(n+1)/2}, \quad t \in \mathbb{R}
$$

### 图像特征

- **对称性**：关于 $t=0$ 对称（与标准正态类似）
- **尾部**：比正态分布更厚（"重尾"）
- 当 $n \to \infty$ 时，$t(n) \to N(0,1)$
- $n=1$ 时为柯西分布

### 与样本均值的关系

设 $X_1, X_2, \ldots, X_n \sim N(\mu, \sigma^2)$，样本均值为 $\bar{X}$，样本方差为 $S^2$，则：

$$
\frac{\bar{X} - \mu}{S/\sqrt{n}} \sim t(n-1)
$$

---

## $F$ 分布（Fisher-Snedecor Distribution）

### 定义

设 $U \sim \chi^2(n_1)$，$V \sim \chi^2(n_2)$，且 $U$ 与 $V$ 独立，则：

$$
F = \frac{U/n_1}{V/n_2} \sim F(n_1, n_2)
$$

服从**自由度为 $(n_1, n_2)$ 的 $F$ 分布**，记作 $F(n_1, n_2)$。

### 概率密度函数

$$
f(x; n_1, n_2) = \frac{\sqrt{\frac{(n_1 x)^{n_1} n_2^{n_2}}{(n_1 x + n_2)^{n_1+n_2}}}}{x \cdot B(n_1/2, n_2/2)}, \quad x > 0
$$

其中 $B(\cdot, \cdot)$ 为 Beta 函数。

### 图像特征

- **定义域**：$x > 0$（右偏分布）
- **峰值**：随 $n_1, n_2$ 增大，分布趋于对称
- 当 $n_2 > 2$ 时，密度函数先增后减

### 与两个样本方差的关系

设 $X_1, \ldots, X_{n_1} \sim N(\mu_1, \sigma_1^2)$，$Y_1, \ldots, Y_{n_2} \sim N(\mu_2, \sigma_2^2)$，两个样本独立，样本方差分别为 $S_1^2$、$S_2^2$，则：

$$
\frac{S_1^2 / \sigma_1^2}{S_2^2 / \sigma_2^2} \sim F(n_1-1, n_2-1)
$$

### $F$ 分布的分位数关系

若 $F \sim F(n_1, n_2)$，则：

$$
F_{1-\alpha}(n_1, n_2) = \frac{1}{F_{\alpha}(n_2, n_1)}
$$

这一性质常用于置信区间和假设检验。

---

## 分位数表

三大抽样分布的分位数表是统计推断的重要工具，常见的表格形式包括：

| 分布 | 常用分位数 | 应用场景 |
|------|-----------|----------|
| $\chi^2(n)$ | $\chi^2_{\alpha}(n)$ | 方差估计、拟合优度检验 |
| $t(n)$ | $t_{\alpha/2}(n)$ | 均值置信区间、$t$ 检验 |
| $F(n_1, n_2)$ | $F_{\alpha}(n_1, n_2)$ | 方差齐性检验、方差分析 |

> **实际使用**：现在通常使用统计软件（如 R、Python scipy）直接计算分位数，已较少依赖纸质分位数表。

## 三大分布的联系

```
标准正态 Z ~ N(0,1)
    │
    ├── Z² ──────────────────→ χ²(n)
    │                              │
    │                              │
    └── Z / √(χ²(n)/n) ──────────→ t(n)
    │                              │
    │                              │
χ²(n₁)/n₁ ─┐                       │
           ├─→ (χ²(n₁)/n₁)/(χ²(n₂)/n₂) → F(n₁, n₂)
χ²(n₂)/n₂ ─┘
```

## 相关章节

- [[../foundations-of-sampling|抽样分布基础]]：理解抽样分布的基本概念
- [[../hypothesis-testing|假设检验]]：三大分布在假设检验中的应用
- [[../interval-estimation|区间估计]]：利用抽样分布构造置信区间
