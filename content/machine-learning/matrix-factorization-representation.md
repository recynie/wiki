---
title: 矩阵分解与表示学习
date: 2026-04-21
description: SVD、PCA与神经网络表示学习的数学基础
tags:
  - deep-learning
  - linear-algebra
  - matrix-factorization
  - representation-learning
  - svd
  - pca
draft: false
permalink:
---

# 矩阵分解与表示学习

矩阵分解是现代机器学习和深度学习的核心工具。从主成分分析（PCA）到词嵌入（Word2Vec），矩阵分解的思想贯穿始终。本章系统介绍矩阵分解的数学原理及其在深度学习中的应用。

## 奇异值分解（SVD）

### 定义

任意实矩阵 $\mathbf{A} \in \mathbb{R}^{m \times n}$ 可以分解为：

$$
\mathbf{A} = \mathbf{U}\mathbf{\Sigma}\mathbf{V}^T
$$

其中：
- $\mathbf{U} \in \mathbb{R}^{m \times m}$：左奇异向量矩阵（$\mathbf{U}^T\mathbf{U} = \mathbf{I}$）
- $\mathbf{V} \in \mathbb{R}^{n \times n}$：右奇异向量矩阵（$\mathbf{V}^T\mathbf{V} = \mathbf{I}$）
- $\mathbf{\Sigma} \in \mathbb{R}^{m \times n}$：奇异值对角矩阵，$\sigma_1 \geq \sigma_2 \geq \cdots \geq 0$

### 几何意义

SVD将线性变换分解为三个基本操作：

1. **旋转** $\mathbf{V}^T$：在输入空间中旋转到主轴方向
2. **缩放** $\mathbf{\Sigma}$：沿主轴缩放（奇异值即为缩放因子）
3. **旋转** $\mathbf{U}$：在输出空间中旋转

### 与特征值分解的关系

对于对称矩阵 $\mathbf{A} = \mathbf{A}^T$：

$$
\mathbf{A} = \mathbf{Q}\mathbf{\Lambda}\mathbf{Q}^T
$$

其中特征值可以是负数。SVD可看作对任意矩阵的"特征值分解"：

$$
\mathbf{A}^T\mathbf{A} = \mathbf{V}\mathbf{\Sigma}^T\mathbf{\Sigma}\mathbf{V}^T = \mathbf{V}\mathbf{\Sigma}^2\mathbf{V}^T
$$

奇异值 $\sigma_i$ 是 $\mathbf{A}^T\mathbf{A}$ 特征值的平方根。

## 最佳低秩近似

### Eckart-Young-Mirsky定理

设 $\mathbf{A} \in \mathbb{R}^{m \times n}$，秩为 $r$，其SVD为 $\mathbf{A} = \mathbf{U}\mathbf{\Sigma}\mathbf{V}^T$。

则秩为 $k < r$ 的最佳近似矩阵为：

$$
\mathbf{A}_k = \mathbf{U}_k \mathbf{\Sigma}_k \mathbf{V}_k^T
$$

其中 $\mathbf{U}_k, \mathbf{\Sigma}_k, \mathbf{V}_k$ 取前 $k$ 个奇异分量。

**最优性**：$\mathbf{A}_k$ 是Frobenius范数意义下的最优近似：

$$
\min_{\mathbf{B}: \text{rank}(\mathbf{B}) = k} \|\mathbf{A} - \mathbf{B}\|_F = \|\mathbf{A} - \mathbf{A}_k\|_F = \sqrt{\sum_{i=k+1}^{r} \sigma_i^2}
$$

### 截断SVD的应用

```python
import numpy as np

def truncated_svd(A, k):
    """
    截断SVD：返回秩为k的最佳近似
    A: (m, n)
    k: 目标秩
    """
    U, s, Vt = np.linalg.svd(A, full_matrices=False)
    return U[:, :k] @ np.diag(s[:k]) @ Vt[:k, :]
```

## 主成分分析（PCA）

### PCA的矩阵推导

设数据矩阵 $\mathbf{X} \in \mathbb{R}^{n \times d}$，每行一个样本。

**数据中心化**：$\mathbf{X}_c = \mathbf{X} - \bar{\mathbf{x}}$

**协方差矩阵**：$\mathbf{C} = \frac{1}{n-1}\mathbf{X}_c^T\mathbf{X}_c \in \mathbb{R}^{d \times d}$

**PCA目标**：找到正交矩阵 $\mathbf{P} \in \mathbb{R}^{d \times k}$，使得：

$$
\mathbf{Y} = \mathbf{X}_c \mathbf{P}
$$

最大化 $\text{Var}(\mathbf{Y})$。

### 闭式解

协方差矩阵的特征值分解：

$$
\mathbf{C} = \mathbf{V}\mathbf{\Lambda}\mathbf{V}^T
$$

主成分为对应最大 $k$ 个特征值的特征向量（列）。

### PCA与SVD的关系

设数据中心化矩阵为 $\mathbf{X}_c$，其SVD为 $\mathbf{X}_c = \mathbf{U}\mathbf{\Sigma}\mathbf{V}^T$。

则：
- 主成分方向：$\mathbf{V}$（右奇异向量）
- 投影坐标：$\mathbf{U}\mathbf{\Sigma}$（左奇异向量乘以奇异值）
- 解释的方差：$\sigma_i^2 / (n-1)$

```python
import numpy as np
from sklearn.decomposition import PCA

def pca_manual(X, n_components):
    """手动实现PCA"""
    # 中心化
    X_centered = X - X.mean(axis=0)
    
    # SVD分解
    U, s, Vt = np.linalg.svd(X_centered, full_matrices=False)
    
    # 取前n_components个主成分
    components = Vt[:n_components]
    explained_variance = s[:n_components]**2 / (len(s) - 1)
    
    # 投影
    X_transformed = X_centered @ components.T
    
    return X_transformed, components, explained_variance
```

### PCA在深度学习中的应用

| 应用 | 说明 |
|------|------|
| **数据降维** | 可视化、压缩存储 |
| **特征提取** | 预处理步骤，减少维度 |
| **初始化** | 预训练权重的PCA初始化 |
| **正则化** | 去相关、降低过拟合 |

## 矩阵分解与嵌入学习

### 词共现矩阵分解

设词共现矩阵 $\mathbf{C} \in \mathbb{R}^{|V| \times |V|}$，其中 $\mathbf{C}_{ij}$ 表示词 $i$ 和词 $j$ 的共现次数。

**目标**：找到词嵌入 $\mathbf{E} \in \mathbb{R}^{|V| \times d}$，使得：

$$
\mathbf{C} \approx \mathbf{E}\mathbf{E}^T
$$

### Word2Vec与矩阵分解的等价性

GloVe (Pennington et al., 2014) 证明了Word2Vec Skip-gram与矩阵分解的等价性：

$$
\mathbf{w}_i^T \tilde{\mathbf{w}}_j = \log(\mathbf{C}_{ij}) - b_i - b_j
$$

这解释了为什么基于共现矩阵的GloVe和基于上下文预测的Skip-gram能学到相似的词向量。

### 推荐系统中的矩阵分解

设用户-物品交互矩阵 $\mathbf{R} \in \mathbb{R}^{|U| \times |I|}$：

$$
\mathbf{R} \approx \mathbf{U}\mathbf{V}^T
$$

其中 $\mathbf{U} \in \mathbb{R}^{|U| \times k}$ 为用户嵌入，$\mathbf{V} \in \mathbb{R}^{|I| \times k}$ 为物品嵌入。

**损失函数**：

$$
\mathcal{L} = \sum_{(u,i) \in \mathcal{O}} (r_{ui} - \mathbf{u}_i^T \mathbf{v}_j)^2 + \lambda(\|\mathbf{U}\|_F^2 + \|\mathbf{V}\|_F^2)
$$

其中 $\mathcal{O}$ 为观测到的交互集合。

```python
import torch
import torch.nn as nn

class MatrixFactorization(nn.Module):
    """矩阵分解推荐模型"""
    def __init__(self, num_users, num_items, embedding_dim):
        super().__init__()
        self.user_embedding = nn.Embedding(num_users, embedding_dim)
        self.item_embedding = nn.Embedding(num_items, embedding_dim)
        
        # 偏置项
        self.user_bias = nn.Embedding(num_users, 1)
        self.item_bias = nn.Embedding(num_items, 1)
        
        self.global_bias = nn.Parameter(torch.zeros(1))
    
    def forward(self, user_ids, item_ids):
        user_emb = self.user_embedding(user_ids)
        item_emb = self.item_embedding(item_ids)
        
        # 点积 + 偏置
        interaction = (user_emb * item_emb).sum(dim=-1)
        user_b = self.user_bias(user_ids).squeeze()
        item_b = self.item_bias(item_ids).squeeze()
        
        return interaction + user_b + item_b + self.global_bias
```

## 神经网络的矩阵视角

### 权重矩阵的秩

神经网络的表达能力与其权重矩阵的秩密切相关：

- **低秩瓶颈**：某些层可能存在低秩结构
- **权重空间维度**：有效参数维度可能小于名义参数维度
- **信息瓶颈**：神经网络可能在某层丢失信息

### 低秩适应（LoRA）

低秩适应是一种高效的微调方法。设预训练权重 $\mathbf{W}_0 \in \mathbb{R}^{d \times k}$，微调时冻结 $\mathbf{W}_0$，只更新低秩分解：

$$
\mathbf{W} = \mathbf{W}_0 + \mathbf{A}\mathbf{B}, \quad \mathbf{A} \in \mathbb{R}^{d \times r}, \mathbf{B} \in \mathbb{R}^{r \times k}
$$

其中 $r \ll \min(d, k)$ 为秩。

```python
class LoRALinear(nn.Module):
    """LoRA线性层"""
    def __init__(self, in_features, out_features, rank=4, alpha=1.0):
        super().__init__()
        self.rank = rank
        self.alpha = alpha
        self.scaling = alpha / rank
        
        # 冻结原始权重
        self.weight = nn.Parameter(torch.zeros(out_features, in_features))
        self.weight.requires_grad = False
        
        # 可训练的低秩分解
        self.lora_A = nn.Parameter(torch.randn(in_features, rank) * 0.01)
        self.lora_B = nn.Parameter(torch.zeros(rank, out_features))
    
    def forward(self, x):
        # W_0 + A * B
        return x @ (self.weight + self.lora_A @ self.lora_B * self.scaling)
```

### SVD在模型压缩中的应用

#### 权重分解

设卷积核 $\mathbf{W} \in \mathbb{R}^{c_{out} \times c_{in} \times k \times k}$，可分解为：

$$
\mathbf{W} = \mathbf{U}\mathbf{\Sigma}\mathbf{V}^T
$$

近似为两个小卷积的组合：
- 第一个：$1 \times 1$ 卷积（$\mathbf{U}$）
- 第二个：$k \times k$ 卷积（$\mathbf{V}^T$）

#### 剪枝与重构

网络剪枝后，权重矩阵可能接近低秩。SVD可以找到最佳重构：

```python
def svd_prune(model, sparsity=0.5):
    """基于SVD的权重剪枝"""
    for name, param in model.named_parameters():
        if 'weight' in name and param.dim() >= 2:
            # SVD分解
            U, s, Vt = torch.linalg.svd(param.data, full_matrices=False)
            
            # 保留前k个奇异值
            k = int(len(s) * (1 - sparsity))
            param.data = U[:, :k] @ torch.diag(s[:k]) @ Vt[:k, :]
```

## 自编码器与表示学习

### 自编码器架构

自编码器通过编码器 $\mathbf{z} = f(\mathbf{x})$ 和解码器 $\hat{\mathbf{x}} = g(\mathbf{z})$ 学习数据的紧凑表示。

$$
\mathcal{L} = \|\mathbf{x} - g(f(\mathbf{x}))\|^2
$$

### PCA自编码器

当编码器是线性层、解码器也是线性层，且损失为MSE时，自编码器学到的表示正好是PCA：

$$
\mathbf{z} = \mathbf{x}\mathbf{W}_e, \quad \hat{\mathbf{x}} = \mathbf{z}\mathbf{W}_d
$$

最优 $\mathbf{W}_e$ 的列空间与PCA主成分一致。

### 变分自编码器（VAE）

VAE引入概率建模：

$$
q_\phi(\mathbf{z}|\mathbf{x}) = \mathcal{N}(\mathbf{z}; \boldsymbol{\mu}_\phi(\mathbf{x}), \boldsymbol{\sigma}_\phi^2(\mathbf{x}))
$$
$$
p_\theta(\mathbf{x}|\mathbf{z}) = \mathcal{N}(\mathbf{x}; \boldsymbol{\mu}_\theta(\mathbf{z}), \boldsymbol{\sigma}_\theta^2(\mathbf{z}))
$$

**损失函数**：

$$
\mathcal{L} = \mathbb{E}_{q_\phi}[\log p_\theta(\mathbf{x}|\mathbf{z})] - D_{KL}(q_\phi(\mathbf{z}|\mathbf{x}) \| p(\mathbf{z}))
$$

```python
class VAE(nn.Module):
    """变分自编码器"""
    def __init__(self, input_dim, latent_dim):
        super().__init__()
        
        # 编码器
        self.encoder = nn.Sequential(
            nn.Linear(input_dim, 256),
            nn.ReLU(),
            nn.Linear(256, 128),
            nn.ReLU()
        )
        self.fc_mu = nn.Linear(128, latent_dim)
        self.fc_logvar = nn.Linear(128, latent_dim)
        
        # 解码器
        self.decoder = nn.Sequential(
            nn.Linear(latent_dim, 128),
            nn.ReLU(),
            nn.Linear(128, 256),
            nn.ReLU(),
            nn.Linear(256, input_dim)
        )
    
    def reparameterize(self, mu, logvar):
        std = torch.exp(0.5 * logvar)
        eps = torch.randn_like(std)
        return mu + eps * std
    
    def forward(self, x):
        h = self.encoder(x)
        mu = self.fc_mu(h)
        logvar = self.fc_logvar(h)
        z = self.reparameterize(mu, logvar)
        return self.decoder(z), mu, logvar
```

## 谱范数与神经网络稳定性

### 谱范数定义

矩阵 $\mathbf{A}$ 的谱范数（2-范数）：

$$
\|\mathbf{A}\|_2 = \max_{\mathbf{x} \neq 0} \frac{\|\mathbf{A}\mathbf{x}\|_2}{\|\mathbf{x}\|_2} = \sigma_{\max}(\mathbf{A})
$$

### 谱归一化

谱归一化 (Spectral Normalization) 强制权重矩阵的谱范数为1：

$$
\mathbf{W}_{SN} = \frac{\mathbf{W}}{\|\mathbf{W}\|_2}
$$

**作用**：
- 满足Lipschitz约束（GAN判别器）
- 训练更稳定
- 有正则化效果

```python
class SpectralNorm(nn.Module):
    """谱归一化包装器"""
    def __init__(self, layer, n_iter=1):
        super().__init__()
        self.layer = layer
        self.n_iter = n_iter
        self.u = None
        self._init_u()
    
    def _init_u(self):
        # 初始化随机向量
        w = self.layer.weight.data
        self.u = nn.Parameter(w.new_empty(w.size(0)).normal_(0, 1))
    
    def _compute_spectral_norm(self):
        w = self.layer.weight.data
        for _ in range(self.n_iter):
            # 幂迭代
            v = torch.nn.functional.normalize(w @ self.u, dim=1)
            self.u.data = torch.nn.functional.normalize(w.T @ v, dim=0)
        return torch.dot(self.u, w @ v)
    
    def forward(self, x):
        w = self.layer.weight.data / self._compute_spectral_norm()
        return nn.functional.linear(x, w, self.layer.bias)
```

## 参考

[^1]: Eckart, Young. "The Approximation of One Matrix by Another of Lower Rank". 1936.
[^2]: Pennington et al. "GloVe: Global Vectors for Word Representation". EMNLP 2014.
[^3]: Hu et al. "LoRA: Low-Rank Adaptation of Large Language Models". ICLR 2022.

---

*相关词条：[[matrix-calculus-dl|矩阵微积分与深度学习]]，[[nn-layers-linear-algebra|线性代数视角下的神经网络层]]，[[deep-vib|深度变分信息瓶颈]]*
