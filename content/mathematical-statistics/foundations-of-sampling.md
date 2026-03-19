---
title: 抽样分布基础 (Foundations of Sampling)
date: 2026-03-19
description: 研究从总体到样本的随机性映射，包括统计量的定义和格利文科-坎泰利定理
tags:
  - mathematical-statistics
  - sampling-distribution
draft: false
permalink:
---

# 抽样分布基础 (Foundations of Sampling)

## 总体与样本

**总体（Population）** 是研究对象的全体，其数量特征由概率分布描述。设总体服从分布 $F(x;\theta)$，其中 $\theta$ 为未知参数。

**样本（Sample）** 是从总体中随机抽取的部分观测值。设 $X_1, X_2, \ldots, X_n$ 为来自总体 $F(x;\theta)$ 的**简单随机样本**，则：

- $X_1, X_2, \ldots, X_n$ 相互独立
- 每个 $X_i$ 与总体 $F(x;\theta)$ 同分布

样本是进行统计推断的基石。

## 统计量的定义

**统计量（Statistic）** 是样本 $X_1, X_2, \ldots, X_n$ 的函数 $T = T(X_1, X_2, \ldots, X_n)$，且**不包含任何未知参数**。

常见统计量包括：
- 样本均值：$\bar{X} = \frac{1}{n}\sum_{i=1}^{n}X_i$
- 样本方差：$S^2 = \frac{1}{n-1}\sum_{i=1}^{n}(X_i - \bar{X})^2$
- 样本标准差：$S = \sqrt{S^2}$
- 样本矩（见下节）

> 注意：统计量完全由样本决定，不依赖于任何未知参数。因此，我们可以根据样本直接计算统计量的值。

## 经验分布函数

设 $X_1, X_2, \ldots, X_n$ 为总体 $F(x)$ 的样本，将它们按从小到大排列：

$$
X_{(1)} \leq X_{(2)} \leq \cdots \leq X_{(n)}
$$

**经验分布函数（Empirical Distribution Function, EDF）** 定义为：

$$
F_n(x) = \frac{1}{n}\sum_{i=1}^{n}\mathbf{1}_{\{X_i \leq x\}}
$$

其中 $\mathbf{1}_{\{X_i \leq x\}}$ 为示性函数，当 $X_i \leq x$ 时取值为 1，否则为 0。

**物理意义**：$F_n(x)$ 表示样本中不超过 $x$ 的观测值所占的比例，是总体分布函数 $F(x)$ 的自然估计。

## 样本矩

### 原点矩

**$k$ 阶原点矩（$k$-th Raw Moment）**：

$$
m_k = \frac{1}{n}\sum_{i=1}^{n}X_i^k
$$

- $k=1$ 时，$m_1 = \bar{X}$（样本均值）

### 中心矩

**$k$ 阶中心矩（$k$-th Central Moment）**：

$$
M_k = \frac{1}{n}\sum_{i=1}^{n}(X_i - \bar{X})^k
$$

- $k=2$ 时，$M_2 = \frac{1}{n}\sum_{i=1}^{n}(X_i - \bar{X})^2$（样本方差分母为 $n$，而非 $n-1$）

> 注意：样本矩与总体矩相对应。当样本容量足够大时，样本矩依概率收敛于相应的总体矩，这正是矩估计法的理论依据。

## 格利文科-坎泰利定理

**格利文科-坎泰利定理（Glivenko-Cantelli Theorem）** 是经验分布函数理论的核心结果：

设 $X_1, X_2, \ldots, X_n$ 为来自总体分布 $F(x)$ 的简单随机样本，$F_n(x)$ 为经验分布函数，则：

$$
P\left(\lim_{n \to \infty} \sup_{x \in \mathbb{R}} |F_n(x) - F(x)| = 0\right) = 1
$$

**定理含义**：
- 经验分布函数 $F_n(x)$ 以概率 1 **一致收敛**于总体分布函数 $F(x)$
- 当样本容量 $n$ 足够大时，$F_n(x)$ 可以作为 $F(x)$ 的近似，且误差可控

**直观理解**：随着样本量增加，样本中各观测值出现的频率逐渐逼近总体各区间对应的概率。

**应用价值**：该定理为数理统计中的许多非参数方法提供了理论基础，表明用样本推断总体是可靠的。

## 相关章节

- [[../three-sampling-distributions|三大抽样分布]]：基于正态总体的衍生分布
- [[../order-statistics|次序统计量]]：样本排序后的统计特性
- [[../point-estimation|点估计]]：利用样本矩估计总体参数
