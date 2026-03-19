---
title: 点估计 (Point Estimation)
date: 2026-03-19
description: 通过样本给出参数的具体数值估计，包括矩估计法和极大似然估计
tags:
  - mathematical-statistics
  - point-estimation
  - mle
draft: false
permalink:
---

# 点估计 (Point Estimation)

## 基本概念

**点估计（Point Estimation）** 是用样本数据构造一个统计量 $\hat{\theta}$，作为未知参数 $\theta$ 的估计值。

### 估计量与估计值

- **估计量（Estimator）**：$\hat{\theta} = \hat{\theta}(X_1, \ldots, X_n)$，是统计量
- **估计值（Estimate）**：将样本观测值代入估计量后得到的具体数值

### 估计量的评判标准

一个好的估计量应当满足以下性质：

| 性质 | 定义 | 意义 |
|------|------|------|
| 无偏性 | $E_{\theta}(\hat{\theta}) = \theta$ | 估计量在平均意义上等于真实参数 |
| 有效性 | $\text{Var}(\hat{\theta}_1) < \text{Var}(\hat{\theta}_2)$ | 方差越小越有效 |
| 一致性 | $\hat{\theta} \xrightarrow{P} \theta$ 当 $n \to \infty$ | 样本量越大，估计越准确 |
| 均方误差 | $\text{MSE}(\hat{\theta}) = E[(\hat{\theta} - \theta)^2]$ | 综合反映无偏性和方差 |

## 矩估计法

### 原理

**矩估计法（Method of Moments, MOM）** 的思想是用样本矩估计相应的总体矩。

设总体有 $k$ 个未知参数 $\theta_1, \ldots, \theta_k$。总体矩是参数的函数：

$$
\mu_j = E_{\theta}[X^j] = g_j(\theta_1, \ldots, \theta_k), \quad j = 1, 2, \ldots, k
$$

样本矩：

$$
m_j = \frac{1}{n}\sum_{i=1}^{n}X_i^j
$$

令总体矩等于样本矩，解方程组得到参数估计：

$$
\begin{cases}
\mu_1(\theta_1, \ldots, \theta_k) = m_1 \\
\mu_2(\theta_1, \ldots, \theta_k) = m_2 \\
\vdots \\
\mu_k(\theta_1, \ldots, \theta_k) = m_k
\end{cases}
$$

### 示例：正态分布

设 $X_1, \ldots, X_n \sim N(\mu, \sigma^2)$，有两个未知参数 $\mu, \sigma^2$。

总体矩：
- $\mu_1 = E[X] = \mu$
- $\mu_2 = E[X^2] = \mu^2 + \sigma^2$

样本矩：
- $m_1 = \bar{X}$
- $m_2 = \frac{1}{n}\sum X_i^2$

方程组：
- $\hat{\mu} = \bar{X}$
- $\hat{\mu}^2 + \hat{\sigma}^2 = \frac{1}{n}\sum X_i^2$

解得：
- $\hat{\mu}_{MOM} = \bar{X}$
- $\hat{\sigma}^2_{MOM} = \frac{1}{n}\sum(X_i - \bar{X})^2$

> 注意：$\hat{\sigma}^2_{MOM}$ 使用除以 $n$（而非 $n-1$）的样本方差。

## 极大似然估计

### 似然函数

**似然函数（Likelihood Function）** 是给定参数 $\theta$ 时，观测到当前样本的概率（密度）：

$$
L(\theta; \mathbf{x}) = \prod_{i=1}^{n}f(x_i; \theta)
$$

### 极大似然估计的定义

**极大似然估计（Maximum Likelihood Estimation, MLE）** $\hat{\theta}_{MLE}$ 满足：

$$
L(\hat{\theta}_{MLE}; \mathbf{x}) = \max_{\theta \in \Theta} L(\theta; \mathbf{x})
$$

或等价地，最大化对数似然函数：

$$
\ell(\theta) = \log L(\theta; \mathbf{x}), \quad \frac{\partial \ell(\theta)}{\partial \theta}\bigg|_{\theta = \hat{\theta}_{MLE}} = 0
$$

### MLE 的求解步骤

1. 写出似然函数 $L(\theta; \mathbf{x}) = \prod_{i=1}^{n}f(x_i; \theta)$
2. 取对数得到对数似然函数 $\ell(\theta) = \log L(\theta; \mathbf{x})$
3. 求导并令导数为零：$\frac{\partial \ell(\theta)}{\partial \theta} = 0$
4. 解方程得到 $\hat{\theta}_{MLE}$
5. 验证二阶导数小于零（确保是极大值）

### 示例：正态分布

设 $X_1, \ldots, X_n \sim N(\mu, \sigma^2)$，参数均未知。

似然函数：
$$
L(\mu, \sigma^2) = \prod_{i=1}^{n}\frac{1}{\sqrt{2\pi\sigma^2}}\exp\left(-\frac{(x_i-\mu)^2}{2\sigma^2}\right)
$$

对数似然函数：
$$
\ell(\mu, \sigma^2) = -\frac{n}{2}\log(2\pi) - \frac{n}{2}\log\sigma^2 - \frac{1}{2\sigma^2}\sum_{i=1}^{n}(x_i-\mu)^2
$$

对 $\mu$ 求偏导并令为零：
$$
\frac{\partial \ell}{\partial \mu} = \frac{1}{\sigma^2}\sum_{i=1}^{n}(x_i - \mu) = 0 \Rightarrow \hat{\mu}_{MLE} = \bar{X}
$$

对 $\sigma^2$ 求偏导并令为零：
$$
\frac{\partial \ell}{\partial \sigma^2} = -\frac{n}{2\sigma^2} + \frac{1}{2(\sigma^2)^2}\sum_{i=1}^{n}(x_i-\mu)^2 = 0 \Rightarrow \hat{\sigma}^2_{MLE} = \frac{1}{n}\sum_{i=1}^{n}(x_i - \bar{X})^2
$$

## 估计量的评判标准详解

### 无偏性

**无偏估计量**：$E_{\theta}(\hat{\theta}) = \theta, \forall \theta$

**渐近无偏**：$\lim_{n \to \infty} E_{\theta}(\hat{\theta}) = \theta$

**有偏估计量**：偏差 $\text{Bias}(\hat{\theta}) = E_{\theta}(\hat{\theta}) - \theta$

> 例子：$\hat{\sigma}^2_{MOM} = \frac{1}{n}\sum(X_i-\bar{X})^2$ 是有偏的（偏差为 $-\sigma^2/n$），而 $\hat{\sigma}^2 = \frac{1}{n-1}\sum(X_i-\bar{X})^2$ 是无偏的。

### 有效性（CRLB 下界）

**克拉美-罗不等式（Cramér-Rao Lower Bound, CRLB）** 给出了无偏估计量方差的理论下界。

若 $\hat{\theta}$ 是 $\theta$ 的无偏估计量，且满足正则条件，则：

$$
\text{Var}(\hat{\theta}) \geq \frac{1}{nI(\theta)}
$$

其中 $I(\theta) = E_{\theta}\left[\left(\frac{\partial \log f(X)}{\partial \theta}\right)^2\right]$ 为**费希尔信息量**。

若无偏估计量达到 CRLB，则称其为**有效估计量（Efficient Estimator）**。

### 一致性（相合性）

**一致估计量（Consistent Estimator）**：当样本容量 $n \to \infty$ 时，

$$
\hat{\theta}_n \xrightarrow{P} \theta
$$

即估计量依概率收敛于真实参数。

### 均方误差

$$
\text{MSE}(\hat{\theta}) = E_{\theta}[(\hat{\theta} - \theta)^2] = \text{Var}(\hat{\theta}) + [\text{Bias}(\hat{\theta})]^2
$$

MSE 统一衡量了估计量的方差和偏差。

## 常用估计量总结

| 总体分布 | 未知参数 | MLE | 是否无偏 |
|---------|---------|-----|---------|
| $N(\mu, \sigma^2)$ | $\mu$ | $\bar{X}$ | 是 |
| $N(\mu, \sigma^2)$ | $\sigma^2$ | $\frac{1}{n}\sum(X_i-\bar{X})^2$ | 否（除 $n-1$ 才无偏） |
| $P(\lambda)$ | $\lambda$ | $\bar{X}$ | 是 |
| $U(0, \theta)$ | $\theta$ | $X_{(n)}$ | 有偏（乘 $\frac{n+1}{n}$ 才无偏） |
| $Bin(n, p)$ | $p$ | $\bar{X}/n$ | 是 |

## 相关章节

- [[../foundations-of-sampling|抽样分布基础]]：样本矩的概念
- [[../sufficiency-and-data-reduction|充分性与数据压缩]]：MLE 与充分统计量的关系
- [[../interval-estimation|区间估计]]：点估计的区间化
- [[../hypothesis-testing|假设检验]]：估计与检验的对偶关系
