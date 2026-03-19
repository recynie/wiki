---
title: 贝叶斯推断 (Bayesian Inference)
date: 2026-03-19
description: 将参数视为随机变量的推断范式，包括先验分布、后验分布和共轭先验
tags:
  - mathematical-statistics
  - bayesian-inference
draft: false
permalink:
---

# 贝叶斯推断 (Bayesian Inference)

## 贝叶斯统计的哲学基础

### 频率派 vs 贝叶斯派

| 观点 | 频率派 | 贝叶斯派 |
|------|--------|---------|
| 参数性质 | 固定常数（虽未知但非随机） | 随机变量，服从某个分布 |
| 样本作用 | 用于推断固定参数 | 用于更新对参数的认知 |
| 先验信息 | 不考虑 | 充分利用 |
| 推断结果 | 参数的点估计/区间估计 | 参数的后验分布 |

### 贝叶斯公式

贝叶斯统计的核心是**贝叶斯公式**：

$$
\pi(\theta \mid \mathbf{x}) = \frac{f(\mathbf{x} \mid \theta)\pi(\theta)}{m(\mathbf{x})}
$$

其中：
- $\pi(\theta)$：**先验分布（Prior Distribution）**，在观测数据之前对参数 $\theta$ 的认知
- $f(\mathbf{x} \mid \theta)$：**似然函数（Likelihood）**，给定参数时观测数据的概率密度
- $\pi(\theta \mid \mathbf{x})$：**后验分布（Posterior Distribution）**，综合了先验信息和样本数据后对 $\theta$ 的认知
- $m(\mathbf{x}) = \int f(\mathbf{x} \mid \theta)\pi(\theta)d\theta$：**边缘似然（Marginal Likelihood）**，与 $\theta$ 无关

**贝叶斯更新的思想**：先验分布 $\xrightarrow{\text{加入数据}}$ 后验分布 $\xrightarrow{\text{作为新的先验}}$ 持续更新。

## 先验分布

### 无信息先验

**无信息先验（Non-Informative Prior）** 尽量少地引入主观信息：

- 拉普拉斯先验：$\pi(\theta) \propto 1$（平坦先验）
- Jeffreys 先验：$\pi(\theta) \propto \sqrt{I(\theta)}$，其中 $I(\theta)$ 为费希尔信息量

### 共轭先验

**共轭先验（Conjugate Prior）**：若先验分布 $\pi(\theta)$ 与似然函数 $f(\mathbf{x} \mid \theta)$ 的组合使得后验分布 $\pi(\theta \mid \mathbf{x})$ 与先验分布 $\pi(\theta)$ 属于同一分布族，则称该先验为共轭先验。

共轭先验的**优点**：计算简便，后验分布有解析形式。

### 常见共轭先验

| 总体分布 | 未知参数 | 共轭先验 | 后验分布 |
|---------|---------|---------|---------|
| 二项 $Bin(n, p)$ | $p$ | $\text{Beta}(\alpha, \beta)$ | $\text{Beta}(\alpha + x, \beta + n - x)$ |
| 泊松 $P(\lambda)$ | $\lambda$ | $\text{Gamma}(\alpha, \beta)$ | $\text{Gamma}(\alpha + \sum x_i, \beta + n)$ |
| 正态 $N(\mu, \sigma^2)$（$\sigma^2$ 已知） | $\mu$ | $N(\mu_0, \sigma_0^2)$ | $N(\mu_n, \sigma_n^2)$ |
| 正态 $N(\mu, \sigma^2)$（$\mu$ 已知） | $\sigma^2$ | $\text{Inverse-}\chi^2(\nu_0, \sigma_0^2)$ | $\text{Inverse-}\chi^2(\nu_n, \sigma_n^2)$ |
| 指数 $\text{Exp}(\lambda)$ | $\lambda$ | $\text{Gamma}(\alpha, \beta)$ | $\text{Gamma}(\alpha + n, \beta + \sum x_i)$ |

### 示例：二项分布的贝叶斯推断

设 $X \sim Bin(n, p)$，先验 $p \sim \text{Beta}(\alpha, \beta)$。

**似然函数**：

$$
f(x \mid p) = \binom{n}{x}p^x(1-p)^{n-x}
$$

**后验分布**：

$$
\pi(p \mid x) \propto p^x(1-p)^{n-x} \cdot p^{\alpha-1}(1-p)^{\beta-1} = p^{x+\alpha-1}(1-p)^{n-x+\beta-1}
$$

即 $\text{Beta}(\alpha + x, \beta + n - x)$。

**后验均值**（贝叶斯估计）：

$$
\hat{p}_{B} = E[p \mid x] = \frac{\alpha + x}{\alpha + \beta + n}
$$

**直观理解**：后验均值是先验均值 $\frac{\alpha}{\alpha+\beta}$ 和样本均值 $\frac{x}{n}$ 的加权平均，权重与样本量和先验参数有关。

## 后验分布的计算

### 解析计算

对于共轭先验，后验分布可以直接写出解析形式（如上例）。

### 数值计算

对于非共轭情况，常用数值方法：

- **数值积分**：直接计算后验分布的归一化常数
- **马尔可夫链蒙特卡洛（MCMC）**：如 Gibbs 采样、Metropolis-Hastings 算法
- **变分推断（Variational Inference）**：近似后验分布

## 贝叶斯估计

### 点估计

从后验分布可以导出参数的点估计：

- **后验均值**：$\hat{\theta}_{ME} = E_{\pi}[\theta \mid \mathbf{x}]$
- **后验中位数**：$\tilde{\theta}$ 满足 $\int_{-\infty}^{\tilde{\theta}}\pi(\theta \mid \mathbf{x})d\theta = 0.5$
- **后验众数（MAP 估计）**：$\hat{\theta}_{MAP} = \arg\max_{\theta} \pi(\theta \mid \mathbf{x})$

### 区间估计：可信区间

贝叶斯派的区间估计称为**可信区间（Credible Interval）**，与置信区间有本质区别：

**$1-\alpha$ 可信区间**：满足

$$
P(\theta \in (a, b) \mid \mathbf{x}) = \int_a^b \pi(\theta \mid \mathbf{x})d\theta = 1 - \alpha
$$

> **与置信区别的关键**：可信区间是说"$\theta$ 落在区间内的概率为 $1-\alpha$"（$\theta$ 是随机变量），而置信区间是说"随机区间以 $1-\alpha$ 的概率包含固定参数"。

## 贝叶斯假设检验

### 贝叶斯因子

**贝叶斯因子（Bayes Factor）** $BF_{10}$ 是后验 odds 与先验 odds 的比值：

$$
BF_{10} = \frac{P(H_1 \mid \mathbf{x}) / P(H_0 \mid \mathbf{x})}{P(H_1) / P(H_0)} = \frac{m_1(\mathbf{x})}{m_0(\mathbf{x})}
$$

其中 $m_j(\mathbf{x}) = \int f(\mathbf{x} \mid H_j)\pi(\theta_j \mid H_j)d\theta_j$ 为在假设 $H_j$ 下的边缘似然。

**解释**：
- $BF_{10} > 1$：数据支持 $H_1$
- $BF_{10} < 1$：数据支持 $H_0$

### Jeffreys 准则

| $|BF_{10}|$ | 证据强度 |
|------------|---------|
| 1 ~ 3.2 | 微弱 |
| 3.2 ~ 10 | 中等 |
| 10 ~ 100 | 强 |
| $> 100$ | 极强 |

## 频率派与贝叶斯派的对比总结

| 方面 | 频率派 | 贝叶斯派 |
|------|--------|---------|
| 参数观 | 固定常数 | 随机变量 |
| 样本观 | 随机抽样 | 固定实现 |
| 推断基础 | 抽样分布 | 后验分布 |
| 置信/可信区间 | 依赖抽样分布 | 直接来自后验分布 |
| 对先验的态度 | 不使用先验信息 | 充分利用先验信息 |
| 计算复杂度 | 通常较低 | 通常较高（MCMC） |

## 相关章节

- [[../point-estimation|点估计]]：贝叶斯估计与频率派估计的比较
- [[../interval-estimation|区间估计]]：可信区间与置信区间的对比
- [[../hypothesis-testing|假设检验]]：贝叶斯因子与频率派检验的比较
- [[../sufficiency-and-data-reduction|充分性与数据压缩]]：贝叶斯充分性的概念
