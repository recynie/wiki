---
title: 充分性与数据压缩 (Sufficiency and Data Reduction)
date: 2026-03-19
description: 研究如何不损失信息地提取样本特征，包括充分统计量和指数族分布
tags:
  - mathematical-statistics
  - sufficiency
  - exponential-family
draft: false
permalink:
---

# 充分性与数据压缩 (Sufficiency and Data Reduction)

## 充分统计量的定义

**充分统计量（Sufficient Statistic）** 是统计学中最重要的概念之一。

**定义**：设 $X_1, \ldots, X_n$ 为来自总体 $F(x;\theta)$ 的样本，$T = T(X_1, \ldots, X_n)$ 为统计量。如果在给定 $T = t$ 的条件下，样本的条件分布与参数 $\theta$ 无关，即：

$$
P_{\theta}(X_1, \ldots, X_n \in A \mid T = t) \quad \text{不依赖于 } \theta, \forall A
$$

则称 $T$ 为 $\theta$ 的**充分统计量**。

### 直观理解

充分统计量包含了样本中关于参数 $\theta$ 的**全部信息**。给定充分统计量的值后，样本不再提供任何关于 $\theta$ 的额外信息。

### 例子：正态分布的样本均值

设 $X_1, \ldots, X_n \sim N(\mu, \sigma^2)$，其中 $\sigma^2$ 已知。则样本均值 $\bar{X}$ 是 $\mu$ 的充分统计量。

**验证**：在给定 $\bar{X} = \bar{x}$ 的条件下，样本的条件分布与 $\mu$ 无关。

## Fisher-Neyman 因子分解定理

**因子分解定理（Factorization Theorem）** 提供了判断充分统计量的简便方法。

**定理**：设总体分布有概率密度函数（或概率质量函数）$f(x;\theta)$，$X_1, \ldots, X_n$ 为样本。统计量 $T$ 是 $\theta$ 的充分统计量，当且仅当存在函数 $g(t, \theta)$ 和 $h(x_1, \ldots, x_n)$ 使得：

$$
L(\theta; x_1, \ldots, x_n) = f(x_1;\theta)\cdots f(x_n;\theta) = g(T(x_1, \ldots, x_n), \theta) \cdot h(x_1, \ldots, x_n)
$$

即似然函数可以分解为两部分：一部分仅通过 $T$ 依赖于 $\theta$，另一部分与 $\theta$ 无关。

### 判别方法

1. 写出似然函数 $L(\theta; \mathbf{x})$
2. 识别出与 $\theta$ 相关的部分
3. 如果这部分仅通过某个统计量 $T$ 依赖于 $\mathbf{x}$，则 $T$ 是充分统计量

## 指数族分布

### 定义

若总体分布的概率密度函数可以写成：

$$
f(x;\theta) = c(\theta)h(x)\exp\left(\sum_{j=1}^{k} w_j(\theta)t_j(x)\right)
$$

则称该分布属于**指数族分布（Exponential Family）**。

### 正则形式

将参数 $\theta$ 替换为自然参数 $\eta$：

$$
f(x;\eta) = h(x)c(\eta)\exp\left(\sum_{j=1}^{k}\eta_j t_j(x)\right)
$$

其中 $\eta = (\eta_1, \ldots, \eta_k)$ 为自然参数，$t_j(x)$ 为充分统计量。

### 常见指数族分布

| 分布 | 概率密度函数 | 自然参数 | 充分统计量 |
|------|-------------|---------|-----------|
| 正态 $N(\mu, \sigma^2)$（$\sigma^2$ 已知） | $\propto \exp(-\frac{x^2}{2\sigma^2}+\frac{\mu}{\sigma^2}x)$ | $\eta = \mu/\sigma^2$ | $(\sum x_i, \sum x_i^2)$ |
| 泊松 $P(\lambda)$ | $\propto \exp(-\lambda + x\log\lambda)$ | $\eta = \log\lambda$ | $\sum x_i$ |
| 二项 $B(n, p)$ | $\propto \exp(x\log\frac{p}{1-p}+n\log(1-p))$ | $\eta = \log\frac{p}{1-p}$ | $\sum x_i$ |
| 伽马 $\Gamma(\alpha, \beta)$ | $\propto \exp(-(\beta-1)\log x + (-\beta)x)$ | $\eta_1 = -\beta, \eta_2 = \alpha-1$ | $(\sum x_i, \sum \log x_i)$ |

### 指数族的重要性质

1. **存在充分统计量**：对于指数族分布，存在 $k$ 维充分统计量 $T(x) = (t_1(x), \ldots, t_k(x))$
2. **数据压缩**：可以用 $k$ 个统计量代替 $n$ 个原始观测值，且不损失关于 $\theta$ 的信息
3. **MLE 的解析形式**：对于指数族，MLE 满足 $\sum_{i=1}^{n}t_j(x_i) = n \cdot E_{\theta}[t_j(X)]$

## 极小充分统计量

**极小充分统计量（Minimal Sufficient Statistic）** 是在充分性基础上进一步追求"最小"的概念。

**定义**：若 $T$ 是充分统计量，且对任何其他充分统计量 $T'$，存在函数 $g$ 使得 $T = g(T')$，则称 $T$ 为极小充分统计量。

**意义**：极小充分统计量在所有充分统计量中信息量相同但维度最低，是数据压缩的极致。

## 常见分布的充分统计量

| 总体分布 | 未知参数 | 充分统计量 |
|---------|---------|-----------|
| $N(\mu, \sigma^2)$ | $\mu$ 已知，$\sigma^2$ 未知 | $\sum X_i^2$ |
| $N(\mu, \sigma^2)$ | $\sigma^2$ 已知，$\mu$ 未知 | $\sum X_i$ |
| $N(\mu, \sigma^2)$ | $\mu, \sigma^2$ 均未知 | $(\sum X_i, \sum X_i^2)$ |
| $P(\lambda)$ | $\lambda$ 未知 | $\sum X_i$ |
| $U(0, \theta)$ | $\theta$ 未知 | $X_{(n)} = \max(X_1, \ldots, X_n)$ |
| $Bin(n, p)$ | $p$ 未知 | $\sum X_i$ |
| $\Gamma(\alpha, \beta)$ | $\alpha, \beta$ 均未知 | $(\sum X_i, \sum \log X_i)$ |

## 辅助统计量与完备性

### 辅助统计量

**辅助统计量（Ancillary Statistic）**：其分布不依赖于参数 $\theta$。

**例子**：设 $X_1, X_2 \sim N(\mu, \sigma^2)$，则 $X_1 - X_2$ 的分布是 $N(0, 2\sigma^2)$，与 $\mu$ 无关但依赖于 $\sigma^2$。

### 完备性

**完备性（Completeness）**：设 $T$ 为统计量，若对任意函数 $g$，由 $E_{\theta}[g(T)] = 0$ 对所有 $\theta$ 成立可以推出 $g(T) = 0$（几乎处处），则称 $T$ 为完备统计量。

完备性在证明最优性时非常有用。

## 相关章节

- [[../three-sampling-distributions|三大抽样分布]]：充分统计量的分布在抽样分布理论中的角色
- [[../point-estimation|点估计]]：充分性与最优估计量的关系（Basu 定理、CRLB）
- [[../order-statistics|次序统计量]]：极值统计量作为充分统计量的例子
