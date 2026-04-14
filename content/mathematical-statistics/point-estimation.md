---
title: 点估计
date: 2026-03-19
description: 通过样本给出参数的具体数值估计，包括矩估计法和极大似然估计
tags:
  - mathematical-statistics
  - point-estimation
  - mle
draft: false
permalink:
---

# 点估计

## 定义

**点估计** 是指用样本数据构造一个统计量 $\hat{\theta}$，作为未知参数 $\theta$ 的一个具体数值估计。

它回答的是“参数大概取多少”这个问题：在样本有限、总体参数未知时，我们希望用一个可计算的统计量尽可能逼近真实参数。

### 估计量与估计值

- **估计量（Estimator）**：$\hat{\theta} = \hat{\theta}(X_1, \ldots, X_n)$，是统计量。
- **估计值（Estimate）**：将样本观测值代入估计量后得到的具体数值。

点估计通常是后续[[../interval-estimation|区间估计]]和[[../hypothesis-testing|假设检验]]的基础。

**为什么需要多个评判标准？** 想象你要估计一座山的海拔高度。不同的人用不同的方法测量可能得到不同的结果：有人用 GPS（可能非常精确但可能有系统偏差），有人用气压计（受天气影响），有人看地图等。我们需要多个标准来判断哪个估计"更好"——这正是估计量评判标准存在的意义。

无偏性关注的是"多次测量结果的平均值是否等于真实值"；
有效性关注的是"测量的波动大小"；
一致性关注的是"样本量增大时估计是否越来越准"；
MSE 则综合考虑偏差和方差的影响。

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

**MLE 的直观理解**：可以把似然函数想象成一幅"参数-可能性"的地形图。给定观测数据后，不同的参数值对应着不同的"可能性高度"。MLE 的目标就是找到这座山的"峰顶"——即使观测数据出现概率最大的参数值。

举例来说，如果抛掷 10 次硬币出现 7 次正面，MLE 估计的 $p = 0.7$ 正是因为这个值使得"观测到 7 次正面"这件事最容易发生。[^5]

### MLE 的求解步骤

1. 写出似然函数 $L(\theta; \mathbf{x}) = \prod_{i=1}^{n}f(x_i; \theta)$。
2. 取对数得到对数似然函数 $\ell(\theta) = \log L(\theta; \mathbf{x})$。
3. 求导并令导数为零：$\frac{\partial \ell(\theta)}{\partial \theta} = 0$。
4. 解方程得到 $\hat{\theta}_{MLE}$。
5. 验证二阶导数小于零（确保是极大值）。

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

## 评判标准

之所以需要多个标准，是因为估计量之间通常存在权衡：

- **无偏性**只保证长期平均不偏离真值，但不一定方差小；
- **有效性**关注方差大小，但通常是在无偏估计量之间比较；
- **一致性**强调大样本下是否趋近真值，却不保证小样本时足够稳定；
- **MSE**把偏差和方差统一起来，更适合综合比较。

因此，一个估计量往往不可能在所有维度上都“最优”，实际应用中需要根据样本规模、可接受偏差、计算复杂度和推断目的综合选择。

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

## 应用

### 常用估计量总结

| 总体分布 | 未知参数 | MLE | 是否无偏 |
|---------|---------|-----|---------|
| $N(\mu, \sigma^2)$ | $\mu$ | $\bar{X}$ | 是 |
| $N(\mu, \sigma^2)$ | $\sigma^2$ | $\frac{1}{n}\sum(X_i-\bar{X})^2$ | 否（除 $n-1$ 才无偏） |
| $P(\lambda)$ | $\lambda$ | $\bar{X}$ | 是 |
| $U(0, \theta)$ | $\theta$ | $X_{(n)}$ | 有偏（乘 $\frac{n+1}{n}$ 才无偏） |
| $Bin(n, p)$ | $p$ | $\bar{X}/n$ | 是 |

### 相关章节

- [[../foundations-of-sampling|抽样分布基础]]：样本矩的概念。
- [[../sufficiency-and-data-reduction|充分性与数据压缩]]：MLE 与充分统计量的关系。
- [[../interval-estimation|区间估计]]：点估计的区间化。
- [[../hypothesis-testing|假设检验]]：估计与检验的对偶关系。

[^1]: 茆诗松、程依明、濮晓龙：《概率论与数理统计教程》。

[^2]: 盛骤、谢式千、潘承毅：《概率论与数理统计》。

[^3]: 相关内容可继续参见[[../foundations-of-sampling|抽样分布基础]]、[[../sufficiency-and-data-reduction|充分性与数据压缩]]、[[../interval-estimation|区间估计]]、[[../hypothesis-testing|假设检验]]。

[^4]: 本文中的矩估计、极大似然估计、无偏性、有效性、一致性与均方误差定义，均属于数理统计中的基础推断框架。

[^5]: 关于最大似然估计的详细介绍，可参考 [Maximum Likelihood Estimation - A Comprehensive Guide](https://www.33rdsquare.com/maximum-likelihood-estimation-a-comprehensive-guide/)。
