---
title: 假设检验 (Hypothesis Testing)
date: 2026-03-19
description: 基于样本证据对总体的某种断言进行决策，包括Neyman-Pearson引理和似然比检验
tags:
  - mathematical-statistics
  - hypothesis-testing
draft: false
permalink:
---

# 假设检验 (Hypothesis Testing)

## 基本概念

**假设检验（Hypothesis Testing）** 是利用样本数据对关于总体的假设进行决策的统计方法。

### 原假设与备择假设

- **原假设（Null Hypothesis）** $H_0$：待检验的假设，通常表示"无效应"或"无差异"
- **备择假设（Alternative Hypothesis）** $H_1$：与 $H_0$ 对立的假设，通常表示我们希望证明的结论

### 两类错误

| 错误类型 | 含义 | 记号 |
|---------|------|------|
| 第一类错误（弃真） | $H_0$ 为真，但我们拒绝了它 | $P(\text{reject } H_0 \mid H_0 \text{ true}) = \alpha$ |
| 第二类错误（取伪） | $H_0$ 为假，但我们没有拒绝它 | $P(\text{accept } H_0 \mid H_0 \text{ false}) = \beta$ |

**显著性水平 $\alpha$**：人为设定的第一类错误的上界，通常取 $0.01, 0.05, 0.10$。

**威力（Power）**：$1 - \beta$，正确拒绝假的原假设的概率。

## 检验的基本流程

1. **建立假设**：明确 $H_0$ 和 $H_1$
2. **选择检验统计量**：构造在 $H_0$ 下分布已知的统计量
3. **确定拒绝域**：根据显著性水平 $\alpha$ 确定拒绝域
4. **计算检验统计量的值**：代入样本观测值
5. **做出决策**：若统计量落在拒绝域，则拒绝 $H_0$

## $p$ 值

**$p$ 值（$p$-value）** 是在原假设 $H_0$ 成立的条件下，观察到比当前样本更极端结果的概率。

### 决策规则

- 若 $p \leq \alpha$，拒绝 $H_0$
- 若 $p > \alpha$，不拒绝 $H_0$

### 优点

- $p$ 值是一个**数值**，反映了样本数据与 $H_0$ 的吻合程度
- 避免了固定显著性水平下的"一刀切"决策
- $p$ 值越小，证据越强烈地反对 $H_0$

## 威力函数

**威力函数（Power Function）** $\beta(\theta)$ 定义为在参数为 $\theta$ 时拒绝 $H_0$ 的概率：

$$
\beta(\theta) = P_{\theta}(\text{reject } H_0)
$$

**性质**：
- 当 $\theta \in H_0$ 时，$\beta(\theta) \leq \alpha$（第一类错误控制）
- 当 $\theta \in H_1$ 时，$\beta(\theta)$ 越大越好（第二类错误越小）

**威力曲线**：以 $\theta$ 为横轴，$\beta(\theta)$ 为纵轴的图像，用于评估检验的性能。

## Neyman-Pearson 基本引理

**Neyman-Pearson 基本引理（Neyman-Pearson Lemma）** 是假设检验理论的基础，给出了最优检验的构造方法。

### 引理内容

设样本 $\mathbf{X} = (X_1, \ldots, X_n)$ 的密度为 $f(\mathbf{x}; \theta)$，考虑简单假设检验：

$$
H_0: \theta = \theta_0 \quad \text{vs} \quad H_1: \theta = \theta_1
$$

构造似然比：

$$
\Lambda(\mathbf{x}) = \frac{L(\theta_0; \mathbf{x})}{L(\theta_1; \mathbf{x})} = \frac{f(\mathbf{x}; \theta_0)}{f(\mathbf{x}; \theta_1)}
$$

则在显著性水平 $\alpha$ 下，**似然比检验**：

$$
\Lambda(\mathbf{x}) \leq c \quad \Rightarrow \quad \text{拒绝 } H_0
$$

是最强检验（MP 检验），其中常数 $c$ 由 $\alpha$ 决定。

### 直观理解

似然比越小，说明样本更可能在 $H_1$ 下出现，而非 $H_0$ 下，因此应拒绝 $H_0$。

## 似然比检验

**广义似然比检验（Generalized Likelihood Ratio Test, GLRT）** 将 Neyman-Pearson 引理推广到复合假设：

$$
\Lambda(\mathbf{x}) = \frac{\sup_{\theta \in \Theta_0} L(\theta; \mathbf{x})}{\sup_{\theta \in \Theta} L(\theta; \mathbf{x})}
$$

其中 $\Theta_0$ 为 $H_0$ 下的参数空间。

**决策规则**：$\Lambda(\mathbf{x}) \leq \lambda$ 时拒绝 $H_0$，$\lambda$ 由显著性水平决定。

## 常用假设检验

### $Z$ 检验（正态总体方差已知）

**单样本 $Z$ 检验**：

$$
H_0: \mu = \mu_0 \quad \text{vs} \quad H_1: \mu \neq \mu_0
$$

检验统计量：

$$
Z = \frac{\bar{X} - \mu_0}{\sigma / \sqrt{n}} \sim N(0, 1) \quad (\text{当 } H_0 \text{ 成立})
$$

**双样本 $Z$ 检验**：比较两个总体均值是否有差异。

### $t$ 检验（正态总体方差未知）

**单样本 $t$ 检验**：

$$
H_0: \mu = \mu_0 \quad \text{vs} \quad H_1: \mu \neq \mu_0
$$

检验统计量：

$$
t = \frac{\bar{X} - \mu_0}{S / \sqrt{n}} \sim t(n-1)
$$

**配对 $t$ 检验**：适用于配对样本。

**两独立样本 $t$ 检验**：比较两个正态总体的均值差异。

### $\chi^2$ 拟合优度检验

**$\chi^2$ 拟合优度检验（Chi-Square Goodness-of-Fit Test）** 用于检验样本是否来自某个特定分布。

**检验统计量**：

$$
\chi^2 = \sum_{i=1}^{k}\frac{(O_i - E_i)^2}{E_i}
$$

其中 $O_i$ 为观测频数，$E_i$ 为期望频数，$k$ 为类别数。

**分布**：在 $H_0$ 成立且样本量足够大时，$\chi^2 \sim \chi^2(k-1-m)$，其中 $m$ 为被估计的参数个数。

### 示例：检验骰子是否均匀

抛掷骰子 600 次，观测到各面的次数为 $(95, 100, 110, 105, 90, 100)$，检验骰子是否均匀。

**期望频数**：每个面期望出现 $600/6 = 100$ 次。

$$
\chi^2 = \frac{(95-100)^2}{100} + \frac{(100-100)^2}{100} + \cdots + \frac{(100-100)^2}{100} = 2.5
$$

**自由度**：$6-1 = 5$，查表得 $\chi^2_{0.05}(5) = 11.07$。

由于 $2.5 < 11.07$，不拒绝 $H_0$，即没有显著证据表明骰子不均匀。

## 检验的比较

| 检验方法 | 适用场景 | 检验统计量分布 |
|---------|---------|--------------|
| $Z$ 检验 | 正态总体，$\sigma^2$ 已知，大样本 | $N(0,1)$ |
| $t$ 检验 | 正态总体，$\sigma^2$ 未知，小样本 | $t(n-1)$ |
| $\chi^2$ 检验 | 分类数据，拟合优度 | $\chi^2(k-1-m)$ |
| $F$ 检验 | 两正态总体方差比较 | $F(n_1-1, n_2-1)$ |

## 检验与置信区间的对偶关系

设参数 $\theta$ 的置信水平 $1-\alpha$ 的置信区间为 $(T_1, T_2)$，则：

- 双侧检验 $H_0: \theta = \theta_0$ vs $H_1: \theta \neq \theta_0$ 在水平 $\alpha$ 下拒绝 $H_0$，当且仅当 $\theta_0 \notin (T_1, T_2)$
- 单侧检验 $H_0: \theta \leq \theta_0$ vs $H_1: \theta > \theta_0$ 在水平 $\alpha$ 下拒绝 $H_0$，当且仅当 $\theta_0 < T_1$

## 相关章节

- [[../three-sampling-distributions|三大抽样分布]]：检验统计量的分布基础
- [[../interval-estimation|区间估计]]：置信区间与假设检验的对偶关系
- [[../point-estimation|点估计]]：估计量与检验统计量的联系
- [[../bayesian-inference|贝叶斯推断]]：频率派假设检验与贝叶斯方法的对比
