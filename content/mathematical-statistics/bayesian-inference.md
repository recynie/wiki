---
title: 贝叶斯推断
date: 2026-03-19
description: 将参数视为随机变量的推断范式，包括先验分布、后验分布和共轭先验
tags:
  - mathematical-statistics
  - bayesian-inference
draft: false
permalink:
---

# 贝叶斯推断

## 哲学背景：频率派 vs 贝叶斯派

贝叶斯统计的核心不是“把未知参数估出来”这么简单，而是**把对参数的不确定性本身建模**。在贝叶斯视角里，参数不是一个已经固定、只是我们不知道的常数，而是带有概率分布的随机变量；这个分布表达的是我们当前对参数的信念强弱，而不是参数本身在重复试验中的频率。

### 频率派 vs 贝叶斯派

| 观点 | 频率派 | 贝叶斯派 |
|------|--------|---------|
| 参数性质 | 固定常数（虽未知但非随机） | 随机变量，服从某个分布 |
| 样本作用 | 用于推断固定参数 | 用于更新对参数的认知 |
| 先验信息 | 不考虑 | 充分利用 |
| 推断结果 | 参数的点估计/区间估计 | 参数的后验分布 |

### 为什么参数可以是随机变量？

这里的“随机”并不是说参数在物理世界里真的一会儿变大、一会儿变小，而是说：**在观测数据到来之前，我们对参数只有不完整的信息**。因此，可以用概率分布来描述“我们对参数的了解程度”。

随着数据不断到来，这个分布会被更新；所以贝叶斯推断本质上是一个“学习过程”，而不是一次性求解一个固定真值。

## 贝叶斯公式

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

## 先验与后验

### 先验分布

**无信息先验（Non-Informative Prior）** 尽量少地引入主观信息：

- 拉普拉斯先验：$\pi(\theta) \propto 1$（平坦先验）
- Jeffreys 先验：$\pi(\theta) \propto \sqrt{I(\theta)}$，其中 $I(\theta)$ 为费希尔信息量

### 后验分布

后验分布是把先验知识和样本信息融合后的结果。它不是“先验和数据的简单平均”，而是通过贝叶斯公式把两者按概率机制严格结合起来。

**直观理解**：

- 先验分布表示“数据到来之前，我们怎么看待参数”；
- 似然函数表示“如果参数取某个值，当前数据有多合理”；
- 后验分布表示“综合两者之后，我们应该怎样理解参数”。

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

## 共轭先验

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

**为什么共轭先验如此重要？** 在贝叶斯推断中，后验分布的计算涉及一个复杂的积分：

$$
\pi(\theta | \mathbf{x}) = \frac{f(\mathbf{x} | \theta)\pi(\theta)}{\int f(\mathbf{x} | \theta')\pi(\theta')d\theta'}
$$

分母这个积分往往难以解析求解。当我们使用共轭先验时，后验分布与先验分布属于同一分布族，积分可以被解析地消除——我们只需要更新分布的参数（超参数），而不需要处理复杂的积分运算。

以二项分布为例：如果我们用 $\text{Beta}(\alpha, \beta)$ 作为 $p$ 的先验，观测到 $x$ 次成功、$n-x$ 次失败后，后验分布为 $\text{Beta}(\alpha + x, \beta + n - x)$。参数更新只是简单的加法！这使得贝叶斯推断在计算上变得可行。[^1]

**共轭先验的另一优势**：当我们进行序贯更新（sequential update）时，共轭先验的优越性更加明显——每新增一次观测，只需要更新超参数，而不需要重新计算整个后验分布。

[^1]: 关于共轭先验的详细讲解，可参考 Gregory Gundersen 的博客 [Conjugacy in Bayesian Inference](https://gregorygundersen.com/blog/2019/03/16/conjugacy/)

## 贝叶斯估计

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

> **与置信区别的关键**：可信区间是说“$\theta$ 落在区间内的概率为 $1-\alpha$”（$\theta$ 是随机变量），而置信区间是说“随机区间以 $1-\alpha$ 的概率包含固定参数”。

## 应用与示例

### 后验分布的计算

对于共轭先验，后验分布可以直接写出解析形式（如上例）。

对于非共轭情况，常用数值方法：

- **数值积分**：直接计算后验分布的归一化常数
- **马尔可夫链蒙特卡洛（MCMC）**：如 Gibbs 采样、Metropolis-Hastings 算法
- **变分推断（Variational Inference）**：近似后验分布

### 贝叶斯假设检验

#### 贝叶斯因子

**贝叶斯因子（Bayes Factor）** $BF_{10}$ 是后验 odds 与先验 odds 的比值：

$$
BF_{10} = \frac{P(H_1 \mid \mathbf{x}) / P(H_0 \mid \mathbf{x})}{P(H_1) / P(H_0)} = \frac{m_1(\mathbf{x})}{m_0(\mathbf{x})}
$$

其中 $m_j(\mathbf{x}) = \int f(\mathbf{x} \mid H_j)\pi(\theta_j \mid H_j)d\theta_j$ 为在假设 $H_j$ 下的边缘似然。

**解释**：
- $BF_{10} > 1$：数据支持 $H_1$
- $BF_{10} < 1$：数据支持 $H_0$

#### Jeffreys 准则

| $|BF_{10}|$ | 证据强度 |
|------------|---------|
| 1 ~ 3.2 | 微弱 |
| 3.2 ~ 10 | 中等 |
| 10 ~ 100 | 强 |
| $> 100$ | 极强 |

### 相关章节

- [[../point-estimation|点估计]]：贝叶斯估计与频率派估计的比较
- [[../interval-estimation|区间估计]]：可信区间与置信区间的对比
- [[../hypothesis-testing|假设检验]]：贝叶斯因子与频率派检验的比较
- [[../sufficiency-and-data-reduction|充分性与数据压缩]]：贝叶斯充分性的概念

## 与频率派对比

| 方面 | 频率派 | 贝叶斯派 |
|------|--------|---------|
| 参数观 | 固定常数 | 随机变量 |
| 样本观 | 随机抽样 | 固定实现 |
| 推断基础 | 抽样分布 | 后验分布 |
| 置信／可信区间 | 依赖抽样分布 | 直接来自后验分布 |
| 对先验的态度 | 不使用先验信息 | 充分利用先验信息 |
| 计算复杂度 | 通常较低 | 通常较高（MCMC） |

### 更本质的差别

频率派关心的是“如果重复抽样无数次，会发生什么”，所以参数被看作一个固定但未知的真值；贝叶斯派关心的是“在已经看到数据以后，我应当如何更新认知”，所以参数被看作不确定对象，用分布来刻画。

这也是为什么贝叶斯方法天然适合：

- 小样本问题；
- 有先验知识的问题；
- 需要直接给出“参数取值概率”的问题；
- 层次模型与复杂模型的推断问题。
