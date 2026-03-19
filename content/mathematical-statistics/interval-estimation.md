---
title: 区间估计 (Interval Estimation)
date: 2026-03-19
description: 给出参数可能存在的范围及其置信程度，包括置信水平和枢轴量法
tags:
  - mathematical-statistics
  - interval-estimation
  - confidence-interval
draft: false
permalink:
---

# 区间估计 (Interval Estimation)

## 基本概念

**区间估计（Interval Estimation）** 是用两个统计量 $T_1$ 和 $T_2$ 构成区间 $(T_1, T_2)$，使得该区间以一定概率包含未知参数 $\theta$。

### 置信水平

设 $\theta$ 为未知参数，若有：

$$
P_{\theta}(T_1 \leq \theta \leq T_2) \geq 1 - \alpha, \quad \forall \theta \in \Theta
$$

则称 $1 - \alpha$ 为**置信水平（Confidence Level）**，区间 $(T_1, T_2)$ 为 $\theta$ 的**置信水平为 $1-\alpha$ 的置信区间（Confidence Interval）**。

### 置信区间的解释

> **正确理解**：置信区间是随机的，而参数是固定的。置信水平 $1-\alpha$ 表示在大量重复抽样中，约有 $100(1-\alpha)\%$ 的置信区间会包含真实参数值。

> **常见误解**：不能说"参数落在 $(T_1, T_2)$ 内的概率为 $1-\alpha$"，因为参数虽然未知但是固定值，不是随机变量。

## 枢轴量法

**枢轴量法（Pivotal Quantity Method）** 是构造置信区间的通用方法。

### 枢轴量的定义

**枢轴量（Pivotal Quantity）** $Q(\mathbf{X}, \theta)$ 是样本和参数的函数，满足：
1. $Q$ 的分布不依赖于任何未知参数
2. $Q$ 是 $\theta$ 的单调函数

### 构造步骤

1. 选取合适的枢轴量 $Q(\mathbf{X}, \theta)$
2. 根据 $Q$ 的已知分布，确定常数 $a, b$ 使得：

$$
P_{\theta}(a \leq Q(\mathbf{X}, \theta) \leq b) = 1 - \alpha
$$

3. 通过不等式变换，得到 $\theta$ 的置信区间

### 示例：正态总体均值的置信区间

设 $X_1, \ldots, X_n \sim N(\mu, \sigma^2)$，$\sigma^2$ 已知。

**选取枢轴量**：

$$
Q = \frac{\bar{X} - \mu}{\sigma / \sqrt{n}} \sim N(0, 1)
$$

**确定常数**：设 $z_{\alpha/2}$ 为标准正态分布的上 $\alpha/2$ 分位数，则：

$$
P\left(-z_{\alpha/2} \leq \frac{\bar{X} - \mu}{\sigma / \sqrt{n}} \leq z_{\alpha/2}\right) = 1 - \alpha
$$

**变换不等式**：

$$
P\left(\bar{X} - z_{\alpha/2}\frac{\sigma}{\sqrt{n}} \leq \mu \leq \bar{X} + z_{\alpha/2}\frac{\sigma}{\sqrt{n}}\right) = 1 - \alpha
$$

**置信区间**：

$$
\left(\bar{X} - z_{\alpha/2}\frac{\sigma}{\sqrt{n}},\quad \bar{X} + z_{\alpha/2}\frac{\sigma}{\sqrt{n}}\right)
$$

## 单侧与双侧置信区间

### 双侧置信区间

两端都有界的置信区间，如上面的 $(\bar{X} \pm z_{\alpha/2}\frac{\sigma}{\sqrt{n}})$。

### 单侧置信区间

只有一侧有界的置信区间：

- **下限置信区间**：$(\bar{X} - z_{\alpha}\frac{\sigma}{\sqrt{n}}, +\infty)$
- **上限置信区间**：$(-\infty, \bar{X} + z_{\alpha}\frac{\sigma}{\sqrt{n}})$

### 选取原则

- 当关心参数是否在某个范围内时，使用**双侧置信区间**
- 当只关心参数是否**小于**（或**大于**）某个值时，使用**单侧置信区间**

## 常见置信区间汇总

### 正态总体均值的置信区间

| 情形 | 置信区间 |
|------|---------|
| $\sigma^2$ 已知 | $\bar{X} \pm z_{\alpha/2}\frac{\sigma}{\sqrt{n}}$ |
| $\sigma^2$ 未知（大样本） | $\bar{X} \pm z_{\alpha/2}\frac{S}{\sqrt{n}}$ |
| $\sigma^2$ 未知（小样本） | $\bar{X} \pm t_{\alpha/2}(n-1)\frac{S}{\sqrt{n}}$ |

其中 $S$ 为样本标准差，$t_{\alpha/2}(n-1)$ 为自由度 $n-1$ 的 $t$ 分布上 $\alpha/2$ 分位数。

### 正态总体方差的置信区间

设 $X_1, \ldots, X_n \sim N(\mu, \sigma^2)$，枢轴量：

$$
\frac{(n-1)S^2}{\sigma^2} \sim \chi^2(n-1)
$$

置信区间：

$$
\left(\frac{(n-1)S^2}{\chi^2_{\alpha/2}(n-1)},\quad \frac{(n-1)S^2}{\chi^2_{1-\alpha/2}(n-1)}\right)
$$

### 两个正态总体均值差的置信区间

设 $X_1, \ldots, X_{n_1} \sim N(\mu_1, \sigma_1^2)$，$Y_1, \ldots, Y_{n_2} \sim N(\mu_2, \sigma_2^2)$，$\sigma_1^2 = \sigma_2^2 = \sigma^2$ 已知/未知。

**$\sigma^2$ 已知**：

$$
(\bar{X} - \bar{Y}) \pm z_{\alpha/2}\sqrt{\frac{\sigma_1^2}{n_1} + \frac{\sigma_2^2}{n_2}}
$$

**$\sigma^2$ 未知**：

$$
(\bar{X} - \bar{Y}) \pm t_{\alpha/2}(n_1+n_2-2)S_p\sqrt{\frac{1}{n_1} + \frac{1}{n_2}}
$$

其中 $S_p^2 = \frac{(n_1-1)S_1^2 + (n_2-1)S_2^2}{n_1+n_2-2}$ 为合并方差。

### 比率 $p$ 的置信区间（大样本）

设 $\hat{p}$ 为二项比例的估计（大样本 $n\hat{p} \geq 5, n(1-\hat{p}) \geq 5$）：

$$
\hat{p} \pm z_{\alpha/2}\sqrt{\frac{\hat{p}(1-\hat{p})}{n}}
$$

## 寻找枢轴量的技巧

1. **从点估计出发**：许多枢轴量是"点估计减去参数"除以某个标准量
2. **利用抽样分布**：三大抽样分布（$\chi^2$、$t$、$F$）是构造枢轴量的基础
3. **对称变换**：对于对称分布（如正态、$t$），利用对称性构造双侧置信区间
4. **单调变换**：如果 $Q$ 是枢轴量，则 $g(Q)$ 也是枢轴量（$g$ 单调）

## 置信区间与假设检验的关系

置信区间与假设检验存在对偶关系：

- 参数 $\theta$ 的置信水平 $1-\alpha$ 的置信区间，等价于检验水平 $\alpha$ 下所有不被拒绝的 $\theta_0$ 构成的集合
- 双侧检验与双侧置信区间对应
- 单侧置信区间与单侧检验对应

## 相关章节

- [[../three-sampling-distributions|三大抽样分布]]：枢轴量分布的理论基础
- [[../point-estimation|点估计]]：点估计与区间估计的关系
- [[../hypothesis-testing|假设检验]]：置信区间与假设检验的对偶性
- [[../order-statistics|次序统计量]]：非参数置信区间的构造
