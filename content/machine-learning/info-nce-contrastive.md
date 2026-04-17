---
title: 对比学习与InfoNCE
date: 2026-04-17
description: InfoNCE损失函数的原理、与互信息的联系，以及在对比学习中的应用
tags:
  - machine-learning
  - deep-learning
  - contrastive-learning
  - self-supervised
  - information-theory
draft: false
permalink:
---

# 对比学习与InfoNCE

**对比学习**（Contrastive Learning）是自监督学习的核心技术之一，通过对比正样本对和负样本对来学习数据的有效表示。**InfoNCE**（Noise-Contrastive Estimation）是其中最重要的损失函数之一，源于互信息的下界估计。

## 核心思想

### 对比学习的目标

学习一个表示 $z$，使得：
- **正样本对** $(x, x^+)$ 的表示尽量**相似**
- **负样本对** $(x, x^-)$ 的表示尽量**不同**

```
正样本对: 同一图像的不同增强视图
         x ────────(+)───▶ z
         x_aug ──(+)───▶ z_aug
         
负样本对: 不同图像的表示
         x ────────(-)───▶ z
         x_neg ──(-)───▶ z_neg
```

### 对比损失的形式

常见的对比损失包括：

| 损失函数 | 公式 | 特点 |
|---------|------|------|
| **Contrastive Loss** | $\mathcal{L} = y d^2 + (1-y)\max(0, \epsilon - d)^2$ | 二元形式 |
| **Triplet Loss** | $\mathcal{L} = \max(0, d(a,n) - d(a,p) + \alpha)$ | 三元组 |
| **NT-Xent** | $\mathcal{L} = -\log \frac{\exp(s_{i,j}/\tau)}{\sum_{k \neq i} \exp(s_{i,k}/\tau)}$ | 无监督 |
| **InfoNCE** | 基于互信息估计 | 理论基础更强 |

---

## InfoNCE 损失函数

### 从互信息到InfoNCE

InfoNCE 损失源自 **Noise-Contrastive Estimation (NCE)** 方法，核心思想是将密度估计问题转化为分类问题。

### 推导过程

**目标**：估计互信息 $I(x; x^+)$

**已知**：互信息可以表示为

$$
I(x; x^+) = \mathbb{E}_{p(x,x^+)}\left[\log \frac{p(x^+|x)}{p(x^+)}\right]
$$

**问题**：分母 $p(x^+)$ 未知且难以计算。

**解决方案**：使用 NCE 将 $p(x^+)$ 替换为噪声分布 $p_n(x^+)$：

$$
I_N(x; x^+) = \mathbb{E}_{p(x,x^+)}\left[\log \frac{p(x^+|x)}{p(x^+|x) + \alpha \cdot p_n(x^+)}\right]
$$

### InfoNCE 损失

对于一个 batch 中的 $N$ 个样本，使用 $N-1$ 个负样本：

$$
\mathcal{L}_{InfoNCE} = -\mathbb{E}\left[\log \frac{\exp(s(x, x^+)/\tau)}{\sum_{i=1}^{N} \exp(s(x, x_i)/\tau)}\right]
$$

其中：
- $s(\cdot, \cdot)$ 是**相似度函数**（通常用余弦相似度）
- $\tau$ 是**温度参数**，控制分布的 sharpness
- $x^+$ 是 $x$ 的正样本（正样本对）
- $x_i$ 包括 $x$ 的正样本和 $N-1$ 个负样本

### 代码实现

```python
import torch
import torch.nn.functional as F
import torch.nn as nn

def info_nce_loss(z_i, z_j, temperature=0.07):
    """
    InfoNCE 损失函数
    
    Args:
        z_i: 第一个视图的表示 (batch_size, d)
        z_j: 第二个视图的表示 (batch_size, d)
        temperature: 温度参数 τ
    
    Returns:
        loss: InfoNCE 损失
    """
    batch_size = z_i.shape[0]
    
    # L2 归一化
    z_i = F.normalize(z_i, dim=1)
    z_j = F.normalize(z_j, dim=1)
    
    # 计算相似度矩阵
    # [2N, 2N] 矩阵：两个视图拼接后计算
    z = torch.cat([z_i, z_j], dim=0)  # (2N, d)
    sim_matrix = torch.matmul(z, z.T) / temperature  # (2N, 2N)
    
    # 对角线是同一图像两个视图的相似度（正样本）
    # 我们需要区分哪些是正对，哪些是负对
    
    # 创建 mask：正样本对的位置
    # (i, i+N) 和 (i+N, i) 是正对
    N = batch_size
    labels = torch.arange(N, device=z_i.device)
    
    # 组合相似度: [z_i|z_j] 和 [z_j|z_i] 都要计算
    # 对于 z_i，正样本是 z_j（索引 N+i）
    # 对于 z_j，正样本是 z_i（索引 i）
    
    # 方式1：SimCLR 风格
    sim_ij = torch.sum(z_i * z_j, dim=1) / temperature  # (N,)
    sim_ji = sim_ij  # 对称
    
    # 构建全部 logits
    # row i 对应 z_i concatenation 后的第 i 行
    logits = torch.cat([
        torch.cat([z_i[:0], z_i], dim=0),   # z_i 与 z_i 的相似度（排除自身）
        torch.cat([z_j, z_j[:0]], dim=0)    # z_j 与 z_j 的相似度（排除自身）
    ], dim=1) / temperature
    
    # 简化实现
    def contrastive_loss(z_i, z_j, temperature):
        """
        简洁的 SimCLR 风格 InfoNCE 实现
        """
        batch_size = z_i.shape[0]
        
        # 归一化
        z_i = F.normalize(z_i, dim=1)
        z_j = F.normalize(z_j, dim=1)
        
        # 相似度矩阵
        # [z_i, z_j] 是 2N x d
        # [z_i, z_j]^T @ [z_i, z_j] 是 2N x 2N
        z = torch.cat([z_i, z_j], dim=0)
        sim = torch.mm(z, z.T) / temperature
        
        # 掩码：排除自身与跨视图
        # 正样本：(i, i+N) 和 (i+N, i)
        N = batch_size
        sim_i_pos = torch.diag(sim, N)      # z_i 与对应 z_j 的相似度
        sim_j_pos = torch.diag(sim, -N)     # z_j 与对应 z_i 的相似度
        
        # 拼接成正样本 logits
        pos = torch.cat([sim_i_pos, sim_j_pos], dim=0)
        
        # 所有 logits（包含自身）需要掩码
        mask = torch.eye(2 * N, device=sim.device, dtype=torch.bool)
        sim.masked_fill_(mask, -float('inf'))
        
        # 负样本 logits
        neg = sim.logsumexp(dim=1)
        
        # InfoNCE 损失
        loss = -torch.mean(pos - neg)
        
        return loss
    
    return contrastive_loss(z_i, z_j, temperature)
```

### PyTorch Metric Learning 实现

```python
from pytorch_metric_learning import distances, losses, reducers, testers
from pytorch_metric_learning.utils.accuracy_control import NVRLoss

# 使用预置的 InfoNCE loss
class InfoNCELoss(nn.Module):
    def __init__(self, temperature=0.07, use_cosine_similarity=True):
        super().__init__()
        self.temperature = temperature
        self.use_cosine_similarity = use_cosine_similarity
        
        if use_cosine_similarity:
            self.distance = distances.CosineSimilarity()
        else:
            self.distance = distances.LpDistance()
    
    def forward(self, embeddings, labels):
        """
        Args:
            embeddings: (N, d) 表示向量
            labels: (N,) 类别标签（用于确定正负样本）
        """
        reducer = reducers.MeanReducer()
        loss_func = losses.NTXentLoss(
            temperature=self.temperature,
            distance=self.distance,
            reducer=reducer
        )
        return loss_func(embeddings, labels)
```

---

## InfoNCE 与互信息的关系

### 理论联系

**关键结论**：InfoNCE 损失是互信息的**下界估计**：

$$
I(x; x^+) \geq \log(N) - \mathcal{L}_{InfoNCE}
$$

### 推导

对于正样本 $x^+$ 和负样本 $\{x_i^-\}_{i=1}^{N-1}$：

$$
\begin{aligned}
\mathcal{L}_{InfoNCE} &= -\log \frac{\exp(s(x, x^+)/\tau)}{\sum_i \exp(s(x, x_i)/\tau)} \\
&= -\log \frac{p(x^+|x)}{p(x^+|x) + \sum_{i=1}^{N-1} p(x_i^-|x)} + \log N \\
&\approx -\log \frac{p(x^+|x)}{p(x^+|x) + (N-1)p_n(x)} + \log N \\
&= -\log \sigma(\log p(x^+|x) - \log [(N-1)p_n(x)]) + \log N
\end{aligned}
$$

当 $\tau \to 0$ 时：

$$
\lim_{\tau \to 0} \mathcal{L}_{InfoNCE} = -\log \frac{p(x^+|x)}{p(x^+|x) + (N-1)p_n(x)} + \log N
$$

这正是 $I(x; x^+)$ 的下界。

### 下界性质的意义

| 特性 | 含义 |
|------|------|
| **下界保证** | 最小化 InfoNCE 损失 → 最大化互信息下界 → 最大化真实互信息 |
| **样本效率** | 负样本数 $N$ 越大，下界越紧 |
| **温度作用** | $\tau$ 影响分布的 sharpness |

---

## 温度参数的作用

### $\tau$ 的影响

| $\tau$ 值 | 分布特性 | 效果 |
|-----------|----------|------|
| **小** (e.g., 0.01-0.1) | sharp，重点关注最相似的负样本 | 学习更细粒度的区分 |
| **中等** (e.g., 0.5-1.0) | 平滑，平衡正负样本 | 标准设置 |
| **大** (e.g., >1.0) | uniform，几乎所有负样本同等重要 | 关注全局结构 |

### 温度调度的策略

```python
class CosineAnnealingTemperature:
    """余弦退火温度调度"""
    
    def __init__(self, T_max, T_min=0.01):
        self.T_max = T_max
        self.T_min = T_min
    
    def get_temperature(self, epoch):
        return self.T_min + 0.5 * (self.T_max - self.T_min) * \
               (1 + np.cos(np.pi * epoch / self.T_max))
```

---

## 典型对比学习方法

### SimCLR

SimCLR（Simple Contrastive Learning of Visual Representations）是经典的对比学习方法：[^1]

```python
class SimCLR(nn.Module):
    """
    SimCLR: 简化对比学习框架
    
    流程:
    1. 图像 x 经过两种数据增强 -> x_i, x_j
    2. 编码器 f(·) -> 表示 h_i, h_j
    3. 投影头 g(·) -> 表示 z_i, z_j
    4. InfoNCE 损失
    """
    
    def __init__(self, encoder, projection_dim=128):
        super().__init__()
        self.encoder = encoder
        self.projection_head = nn.Sequential(
            nn.Linear(encoder.output_dim, 256),
            nn.ReLU(),
            nn.Linear(256, projection_dim)
        )
    
    def forward(self, x):
        # 两个视图
        x_i = self.augment(x)
        x_j = self.augment(x)
        
        # 编码
        h_i = self.encoder(x_i)
        h_j = self.encoder(x_j)
        
        # 投影
        z_i = self.projection_head(h_i)
        z_j = self.projection_head(h_j)
        
        # InfoNCE 损失
        loss = info_nce_loss(z_i, z_j, temperature=0.07)
        
        return loss, z_i, z_j
    
    def get_representations(self, x):
        """获取表示用于下游任务"""
        return self.encoder(x)
    
    @staticmethod
    def augment(x):
        """数据增强"""
        # SimCLR 使用的增强组合
        transforms = [
            transforms.RandomResizedCrop(224),
            transforms.RandomHorizontalFlip(),
            transforms.ColorJitter(0.4, 0.4, 0.4, 0.1),
            transforms.RandomGrayscale(p=0.2),
            transforms.GaussianBlur(kernel_size=23),
            transforms.ToTensor(),
            transforms.Normalize(mean=[0.485, 0.456, 0.406], 
                                std=[0.229, 0.224, 0.225])
        ]
        return transforms
```

### MoCo

MoCo（Momentum Contrast）使用动量更新的队列维护大量负样本：[^2]

```python
class MoCo(nn.Module):
    """
    MoCo: 动量对比学习
    
    关键创新:
    - 使用动量编码器维持一致的负样本表示
    - 使用队列存储大量负样本
    """
    
    def __init__(self, encoder, projection_dim=128, K=65536, m=0.999):
        super().__init__()
        self.K = K  # 负样本队列大小
        self.m = m  # 动量系数
        
        # 查询编码器
        self.encoder_q = encoder
        self.projection_q = nn.Linear(encoder.output_dim, projection_dim)
        
        # 键编码器（动量更新）
        self.encoder_k = copy.deepcopy(encoder)
        self.projection_k = nn.Linear(encoder.output_dim, projection_dim)
        
        for param_q, param_k in zip(
            self.encoder_q.parameters(), 
            self.encoder_k.parameters()
        ):
            param_k.data.copy_(param_q.data)
            param_k.requires_grad = False
        
        # 负样本队列
        self.register_buffer('queue', torch.randn(projection_dim, K))
        self.queue = F.normalize(self.queue, dim=0)
        self.register_buffer('queue_ptr', torch.zeros(1, dtype=torch.long))
    
    @torch.no_grad()
    def _momentum_update_key_encoder(self):
        """动量更新键编码器"""
        for param_q, param_k in zip(
            self.encoder_q.parameters(),
            self.encoder_k.parameters()
        ):
            param_k.data = param_k.data * self.m + param_q.data * (1 - self.m)
    
    @torch.no_grad()
    def _dequeue_and_enqueue(self, keys):
        """更新负样本队列"""
        batch_size = keys.shape[0]
        ptr = int(self.queue_ptr)
        
        self.queue[:, ptr:ptr+batch_size] = keys.T
        self.queue_ptr[0] = (ptr + batch_size) % self.K
    
    def forward(self, x_q, x_k):
        # 查询
        q = self.projection_q(self.encoder_q(x_q))
        q = F.normalize(q, dim=1)
        
        # 键（动量编码器，不梯度更新）
        with torch.no_grad():
            k = self.projection_k(self.encoder_k(x_k))
            k = F.normalize(k, dim=1)
        
        # 正样本 logits
        l_pos = torch.einsum('nc,nc->n', q, k).unsqueeze(-1)
        
        # 负样本 logits
        l_neg = torch.einsum('nc,ck->nk', q, self.queue.clone())
        
        # 总 logits
        logits = torch.cat([l_pos, l_neg], dim=1) / 0.07
        
        # 标签（全0，正样本在第一个位置）
        labels = torch.zeros(logits.shape[0], dtype=torch.long, device=logits.device)
        
        # 交叉熵损失
        loss = F.cross_entropy(logits, labels)
        
        # 更新键编码器和队列
        self._momentum_update_key_encoder()
        self._dequeue_and_enqueue(k)
        
        return loss
```

### BYOL 和 SimSiam

BYOL 和 SimSiam 采用孪生网络架构，不需要负样本：[^3]

```python
class SimSiam(nn.Module):
    """
    SimSiam: 简化的孪生网络
    
    关键创新:
    - 不需要负样本
    - 使用 stop-gradient 避免崩溃解
    - 预测器网络增强表示
    """
    
    def __init__(self, encoder, projection_dim=2048, prediction_dim=512):
        super().__init__()
        
        self.encoder = encoder
        
        # 投影网络
        self.projection = nn.Sequential(
            nn.Linear(encoder.output_dim, projection_dim),
            nn.BatchNorm1d(projection_dim),
            nn.ReLU(),
            nn.Linear(projection_dim, projection_dim)
        )
        
        # 预测网络
        self.predictor = nn.Sequential(
            nn.Linear(projection_dim, prediction_dim),
            nn.BatchNorm1d(prediction_dim),
            nn.ReLU(),
            nn.Linear(prediction_dim, projection_dim)
        )
    
    def forward(self, x1, x2):
        # 编码
        r1 = self.projection(self.encoder(x1))
        r2 = self.projection(self.encoder(x2))
        
        # 预测
        p1 = self.predictor(r1)
        p2 = self.predictor(r2)
        
        # 损失：对称的均方误差
        # 注意：r1, r2 使用 stop-gradient
        loss = 0.5 * (
            F.mse_loss(p1, r2.detach()) + 
            F.mse_loss(p2, r1.detach())
        )
        
        return loss
```

---

## 扩展与改进

### Hard Negative Mining

```python
def info_nce_with_hard_negatives(z_i, z_j, temperature=0.07, hard_ratio=0.5):
    """
    InfoNCE with Hard Negative Mining
    
    选择最难的负样本（相似度最高的）进行更强烈的惩罚
    """
    z_i = F.normalize(z_i, dim=1)
    z_j = F.normalize(z_j, dim=1)
    
    N = z_i.shape[0]
    z = torch.cat([z_i, z_j], dim=0)
    
    # 相似度矩阵
    sim = torch.mm(z, z.T) / temperature
    
    # 掩码
    mask = torch.eye(2 * N, dtype=torch.bool, device=sim.device)
    sim.masked_fill_(mask, -float('inf'))
    
    # 选取 hard negatives
    top_k = int(N * hard_ratio)
    hard_sim, _ = sim[:, N:].topk(top_k, dim=1)
    
    # 只对 hard negatives 计算损失
    logits = torch.cat([
        torch.diag(sim[:N, N:]).unsqueeze(1),  # 正样本
        hard_sim  # hard 负样本
    ], dim=1)
    
    labels = torch.zeros(N, dtype=torch.long, device=logits.device)
    
    return F.cross_entropy(logits, labels)
```

### 对比损失的正则化

```python
class RegularizedInfoNCE(nn.Module):
    """
    带正则化的 InfoNCE
    
    添加:
    - 方差正则: 避免表示坍塌
    - 均匀性正则: 鼓励表示均匀分布
    - 对齐性正则: 正样本表示应该对齐
    """
    
    def __init__(self, temperature=0.07, lambda_uniform=0.1, lambda_align=0.1):
        super().__init__()
        self.temperature = temperature
        self.lambda_uniform = lambda_uniform
        self.lambda_align = lambda_align
    
    def uniform_loss(self, z):
        """均匀性损失：基于 jensen-Shannon 散度"""
        z = F.normalize(z, dim=1)
        batch_size = z.shape[0]
        
        # 归一化后的点积
        pairwise_sim = torch.mm(z, z.T)
        
        # 均匀性：所有点应该等间距
        # 使用方差作为近似
        variance = torch.var(pairwise_sim)
        return variance
    
    def align_loss(self, z_i, z_j):
        """对齐性损失：正样本应该接近"""
        return F.mse_loss(z_i, z_j)
    
    def forward(self, z_i, z_j):
        # 标准 InfoNCE
        nce = info_nce_loss(z_i, z_j, self.temperature)
        
        # 均匀性正则
        uniform = self.uniform_loss(torch.cat([z_i, z_j], dim=0))
        
        # 对齐性正则
        align = self.align_loss(
            F.normalize(z_i, dim=1), 
            F.normalize(z_j, dim=1)
        )
        
        return nce + self.lambda_uniform * uniform + self.lambda_align * align
```

---

## 核心公式速查

| 概念 | 公式 |
|------|------|
| **InfoNCE** | $\mathcal{L} = -\log \frac{\exp(s_{i,j}/\tau)}{\sum_k \exp(s_{i,k}/\tau)}$ |
| **余弦相似度** | $s(a,b) = \frac{a \cdot b}{\|a\|\|b\|}$ |
| **互信息下界** | $I(x; x^+) \geq \log(N) - \mathcal{L}_{InfoNCE}$ |
| **温度作用** | 小 $\tau$ → sharp 分布；大 $\tau$ → uniform 分布 |

---

## 参考

[^1]: Chen, T., Kornblith, S., Norouzi, M., & Hinton, G. (2020). "A Simple Framework for Contrastive Learning of Visual Representations". *ICML*.
[^2]: He, K., Fan, H., Wu, Y., Xie, S., & Girshick, R. (2020). "Momentum Contrast for Unsupervised Visual Representation Learning". *CVPR*.
[^3]: Grill, J.B., et al. (2020). "Bootstrap Your Own Latent: A New Approach to Self-Supervised Learning". *NeurIPS*.

## 相关文章

- [[../math/information-theory|信息论基础]] — 熵、互信息、KL散度
- [[../math/information-bottleneck|信息瓶颈理论]] — IB 目标与深度学习
- [[../machine-learning/deep-vib|Deep VIB]] — 变分信息瓶颈实现
