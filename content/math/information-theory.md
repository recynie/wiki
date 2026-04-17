---
title: 信息论基础
date: 2026-04-17
description: 信息论核心概念：熵、互信息、KL散度、交叉熵及其在机器学习中的应用
tags:
  - information-theory
  - entropy
  - mutual-information
  - kl-divergence
draft: false
permalink:
---

# 信息论基础

信息论（Information Theory）由克劳德·香农于1948年在《通信的数学理论》中奠定基础，是研究信息量化、存储和传输的数学分支。[^1] 在机器学习和深度学习中，信息论提供了理解模型行为、优化训练过程的重要理论框架。

## 熵（Entropy）

### 信息量

理解熵之前，先了解**信息量**（Self-Information）的概念。对于事件 $x$，其信息量为：

$$
i(x) = -\log_b P(x)
$$

- 当 $b=2$ 时，单位为 **比特（bits）**
- 当 $b=e$ 时，单位为 **奈特（nats）**

**关键洞察**：稀有事件携带更多信息。例如：
- 硬币正面（$P=0.5$）：$i = 1$ bit
- 连续三次正面（$P=0.125$）：$i = 3$ bits
- 极稀有事件（$P=0.001$）：$i \approx 10$ bits

### 香农熵的定义

**香农熵**（Shannon Entropy）是对概率分布不确定性的度量：

$$
H(X) = \mathbb{E}_{p(x)}[i(x)] = -\sum_{x \in \mathcal{X}} p(x) \log p(x)
$$

或等价形式：

$$
H(X) = -\mathbb{E}_{p(x)}\left[\log \frac{1}{p(X)}\right]
$$

### 熵的性质

| 性质 | 描述 |
|------|------|
| 非负性 | $H(X) \geq 0$ |
| 对称性 | $H(X_1, X_2, \ldots, X_n) = H(X_{\pi(1)}, X_{\pi(2)}, \ldots, X_{\pi(n)})$ |
| 可加性 | $H(X, Y) = H(X) + H(Y \mid X)$（链式法则） |
| 最大值 | 均匀分布熵最大 |
| 极值性 | 确定分布熵为 0 |

### 联合熵与条件熵

**联合熵**：

$$
H(X, Y) = -\sum_{x,y} p(x,y) \log p(x,y)
$$

**条件熵**：

$$
H(Y \mid X) = \sum_{x} p(x) H(Y \mid X=x) = -\sum_{x,y} p(x,y) \log p(y \mid x)
$$

### 熵的链式法则

$$
H(X, Y) = H(X) + H(Y \mid X) = H(Y) + H(X \mid Y)
$$

### 代码实现

```python
import numpy as np

def shannon_entropy(p):
    """计算香农熵（使用自然对数，单位为nats）"""
    p = np.array(p)
    # 过滤掉概率为0的项（0*log(0) = 0）
    mask = (p > 0) & (p < 1)
    return -np.sum(p[mask] * np.log(p[mask]))

def joint_entropy(p_xy):
    """计算联合熵"""
    p_xy = np.array(p_xy)
    mask = (p_xy > 0) & (p_xy < 1)
    return -np.sum(p_xy[mask] * np.log(p_xy[mask]))

def conditional_entropy(p_y_given_x, p_x):
    """计算条件熵 H(Y|X)"""
    return sum(p_x[i] * shannon_entropy(p_y_given_x[i]) for i in range(len(p_x)))

# 示例：二元分类中的类别分布
p_balanced = [0.5, 0.5]  # 平衡数据集
p_imbalanced = [0.9, 0.1]  # 不平衡数据集

print(f"平衡数据熵: {shannon_entropy(p_balanced):.4f}")  # ~0.693
print(f"不平衡数据熵: {shannon_entropy(p_imbalanced):.4f}")  # ~0.325
```

---

## 互信息（Mutual Information）

### 定义

互信息衡量两个随机变量之间的依赖程度：

$$
I(X; Y) = \sum_{x,y} p(x,y) \log \frac{p(x,y)}{p(x)p(y)}
$$

### 与熵的关系

互信息可以表示为多种等价形式：

$$
I(X; Y) = H(X) - H(X \mid Y) = H(Y) - H(Y \mid X) = H(X) + H(Y) - H(X, Y)
$$

**物理意义**：$I(X; Y)$ 表示已知 $Y$ 后对 $X$ 不确定性的减少量。

### 维恩图表示

```
        ┌───────────────────────────┐
        │                           │
        │    H(X, Y)                │
        │   ┌─────────────┐         │
        │   │    H(X|Y)   │    H(Y|X)│
        │   └──────┬──────┘         │
        │          │                │
        │          │ I(X;Y)         │
        │          │                │
        └──────────┴─────────────────┘
```

### 性质

| 性质 | 公式 |
|------|------|
| 对称性 | $I(X; Y) = I(Y; X)$ |
| 非负性 | $I(X; Y) \geq 0$ |
| 独立性 | $I(X; Y) = 0 \iff X \perp Y$ |
| 自信息 | $I(X; X) = H(X)$ |
| 数据处理不等式（DPI） | $X \rightarrow Y \rightarrow Z \Rightarrow I(X; Z) \leq I(X; Y)$ |

### 在机器学习中的应用

#### 特征选择

利用互信息选择与目标变量高度相关的特征：

```python
def mutual_information(p_xy, p_x, p_y):
    """计算互信息"""
    return np.sum(p_xy * np.log(p_xy / (np.outer(p_x, p_y) + 1e-10) + 1e-10))
```

#### InfoGAN 中的应用

InfoGAN 通过最大化隐变量 $c$ 与生成图像 $G(z,c)$ 之间的互信息来学习解耦表示：

$$
\min_{G,Q} \max_{D} \; L_I(G, Q) + L_C(G, D)
$$

---

## KL 散度（Kullback-Leibler Divergence）

### 定义

KL 散度衡量两个概率分布之间的差异：

$$
D_{KL}(P \| Q) = \sum_{x} P(x) \log \frac{P(x)}{Q(x)} = \mathbb{E}_P\left[\log \frac{P(X)}{Q(X)}\right]
$$

对于连续分布：

$$
D_{KL}(P \| Q) = \int P(x) \log \frac{P(x)}{Q(x)} \, dx
$$

### 关键性质

| 性质 | 描述 |
|------|------|
| 非负性 | $D_{KL}(P \| Q) \geq 0$（吉布斯不等式） |
| 非对称性 | $D_{KL}(P \| Q) \neq D_{KL}(Q \| P)$ |
| 链式法则 | $D_{KL}(P(X,Y) \| Q(X,Y)) = D_{KL}(P(X)\|Q(X)) + D_{KL}(P(Y\|X)\|Q(Y\|X))$ |
| 积性 | $\log P(x) + \log Q(x) = \log [P(x) \cdot Q(x)]$ |

### 与交叉熵的关系

**交叉熵**定义为：

$$
H(P, Q) = -\sum_{x} P(x) \log Q(x)
$$

三者之间的关系：

$$
H(P, Q) = H(P) + D_{KL}(P \| Q)
$$

因此，当 $H(P)$ 固定时，最小化交叉熵等价于最小化 KL 散度。

### 代码实现

```python
import scipy.stats

def kl_divergence(p, q):
    """计算 KL 散度 D_KL(P || Q)"""
    p = np.array(p, dtype=np.float64)
    q = np.array(q, dtype=np.float64)
    # 过滤掉 P 中为 0 的项
    mask = (p > 0) & (q > 0)
    return np.sum(p[mask] * np.log(p[mask] / q[mask]))

# 使用 scipy 验证
p = np.array([0.3, 0.7])
q = np.array([0.4, 0.6])
print(f"D_KL(P||Q): {scipy.stats.entropy(p, q):.4f}")
```

---

## 交叉熵与损失函数

### 交叉熵损失函数

在分类任务中，交叉熵损失衡量真实分布 $p$ 与预测分布 $q$ 之间的差异：

$$
L_{CE} = -\sum_{c=1}^{C} y_c \log(\hat{y}_c)
$$

其中 $y_c$ 是 one-hot 编码的真实标签，$\hat{y}_c$ 是预测概率。

### 二元交叉熵（Binary Cross-Entropy, BCE）

对于二分类问题：

$$
L_{BCE} = -\frac{1}{N} \sum_{i=1}^{N} [y_i \log(\hat{y}_i) + (1-y_i)\log(1-\hat{y}_i)]
$$

### PyTorch 实现

```python
import torch
import torch.nn as nn

# 多分类示例
criterion = nn.CrossEntropyLoss()
logits = torch.randn(32, 10)  # batch_size=32, 10个类别
targets = torch.randint(0, 10, (32,))
loss = criterion(logits, targets)

# 二分类示例
criterion_bce = nn.BCEWithLogitsLoss()
logits = torch.randn(32, 1)
targets = torch.randint(0, 2, (32, 1)).float()
loss = criterion_bce(logits, targets)
```

### 交叉熵损失的优势

1. **良好的梯度特性**：当预测错误时提供强梯度信号
2. **概率解释**：与最大似然估计（MLE）一致
3. **与信息论的联系**：最小化交叉熵等价于最大化数据似然

### 标签平滑（Label Smoothing）

标签平滑是一种正则化技术，将硬标签替换为软标签：

$$
y_i^{smooth} = y_i(1-\epsilon) + \frac{\epsilon}{K}
$$

其中 $K$ 是类别数，$\epsilon$ 是平滑参数（通常取 $0.1$）。

```python
def label_smoothing(labels, num_classes, epsilon=0.1):
    """标签平滑"""
    return labels * (1 - epsilon) + epsilon / num_classes
```

---

## 最大熵原理（Maximum Entropy Principle）

### 原理阐述

在只知道部分约束的情况下，应该选择熵最大的概率分布，即不确定性最大的分布。这一原理在统计学和物理学中都有重要应用。

### 数学形式

给定约束 $\mathbb{E}[f_k(X)] = c_k$ for $k=1, \ldots, m$，最大化：

$$
\max_{p} \quad H(X) = -\sum_{x} p(x) \log p(x)
$$

subject to:
- $\sum_x p(x) = 1$
- $\sum_x p(x) f_k(x) = c_k, \quad k=1, \ldots, m$

### 拉格朗日求解

引入拉格朗日乘子 $\lambda_0, \lambda_1, \ldots, \lambda_m$：

$$
\mathcal{L} = -\sum_x p(x) \log p(x) + \lambda_0\left(\sum_x p(x) - 1\right) + \sum_{k=1}^m \lambda_k\left(\sum_x p(x)f_k(x) - c_k\right)
$$

求导并令为 0：

$$
\frac{\partial \mathcal{L}}{\partial p(x)} = -1 - \log p(x) + \lambda_0 + \sum_{k=1}^m \lambda_k f_k(x) = 0
$$

解得：

$$
p(x) = \exp\left(\lambda_0 - 1\right) \cdot \exp\left(\sum_{k=1}^m \lambda_k f_k(x)\right) = \frac{1}{Z}\exp\left(\sum_{k=1}^m \lambda_k f_k(x)\right)
$$

其中 $Z = \exp(1-\lambda_0)$ 是**归一化常数**（配分函数）。

### 在机器学习中的应用

#### 逻辑回归

逻辑回归本质上是最**大熵分类器**在二分类情况下的特例，其输出是满足给定特征期望约束的最大熵分布。

#### 最大熵马尔可夫模型（MEMM）

MEMM 使用最大熵原理进行序列标注，结合了 HMM 的发射概率和最大熵的全局特征建模能力。

---

## 信息论在深度学习中的核心应用

### 注意力机制的信息论视角

注意力机制可以被理解为一种信息路由和选择过程：

$$
\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

从信息论角度：
- **Query $Q$**：表示需要查询的信息目标
- **Key $K$**：代表可用的信息键
- **Value $V$**：包含实际的信息内容
- Softmax 操作计算 Query 与各 Key 之间的互信息近似

### 对比学习中的 InfoNCE

InfoNCE 是对比学习中常用的损失函数，源自互信息的下界估计：

$$
\mathcal{L}_{InfoNCE} = -\mathbb{E}\left[\log \frac{\exp(s(x, x^+)/\tau)}{\sum_{i=1}^{N} \exp(s(x, x_i)/\tau)}\right]
$$

其中 $s(\cdot, \cdot)$ 是相似度函数，$\tau$ 是温度参数。

**与互信息的联系**：InfoNCE 损失是互信息的下界估计，最小化 InfoNCE 损失等价于最大化正样本对之间的互信息。

---

## 核心公式速查表

| 概念 | 公式 |
|------|------|
| **信息量** | $i(x) = -\log P(x)$ |
| **熵** | $H(X) = -\sum_x P(x)\log P(x)$ |
| **联合熵** | $H(X,Y) = -\sum_{x,y}P(x,y)\log P(x,y)$ |
| **条件熵** | $H(Y\mid X) = \sum_x P(x)H(Y\mid X=x)$ |
| **互信息** | $I(X;Y) = H(X) - H(X\mid Y)$ |
| **KL散度** | $D_{KL}(P\mid\mid Q) = \sum_x P(x)\log\frac{P(x)}{Q(x)}$ |
| **交叉熵** | $H(P,Q) = -\sum_x P(x)\log Q(x)$ |
| **关系** | $H(P,Q) = H(P) + D_{KL}(P\mid\mid Q)$ |

---

## 参考

[^1]: Shannon, C.E. (1948). "A Mathematical Theory of Communication". *Bell System Technical Journal*, 27(3), 379-423.
[^2]: Cover, T.M. & Thomas, J.A. (2006). *Elements of Information Theory*. Wiley-Interscience.
[^3]: Machine Learning Mastery. "From Shannon to Modern AI: A Complete Information Theory Guide for Machine Learning". https://machinelearningmastery.com/from-shannon-to-modern-ai-a-complete-information-theory-guide-for-machine-learning/
