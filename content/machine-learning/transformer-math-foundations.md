---
title: Transformer数学基础
date: 2026-04-21
description: Self-Attention、位置编码与高效注意力的数学原理
tags:
  - transformer
  - self-attention
  - positional-encoding
  - flash-attention
  - deep-learning
draft: false
permalink:
---

# Transformer数学基础

Transformer架构的核心是**自注意力机制**（Self-Attention），它通过并行计算序列中所有位置之间的依赖关系，彻底解决了传统RNN的序列依赖问题。本章深入剖析Transformer的数学原理。

## Self-Attention机制

### 核心公式

标准Scaled Dot-Product Attention定义为：

$$
\text{Attention}(\mathbf{Q}, \mathbf{K}, \mathbf{V}) = \text{softmax}\left(\frac{\mathbf{Q}\mathbf{K}^T}{\sqrt{d_k}}\right)\mathbf{V}
$$

### 逐步骤推导

#### Step 1：QKV投影

设输入序列 $\mathbf{X} = [\mathbf{x}_1, \mathbf{x}_2, \ldots, \mathbf{x}_n] \in \mathbb{R}^{n \times d}$：

$$
\mathbf{Q} = \mathbf{X}\mathbf{W}_Q, \quad \mathbf{K} = \mathbf{X}\mathbf{W}_K, \quad \mathbf{V} = \mathbf{X}\mathbf{W}_V
$$

其中：
- $\mathbf{W}_Q, \mathbf{W}_K, \mathbf{W}_V \in \mathbb{R}^{d \times d_k}$：可学习的投影矩阵
- $\mathbf{Q}, \mathbf{K}, \mathbf{V} \in \mathbb{R}^{n \times d_k}$：查询、键、值矩阵

#### Step 2：计算注意力分数

$$
\mathbf{S} = \mathbf{Q}\mathbf{K}^T \in \mathbb{R}^{n \times n}
$$

矩阵元素 $\mathbf{S}_{ij}$ 表示第 $i$ 个位置对第 $j$ 个位置的注意力权重（未归一化）。

#### Step 3：缩放

$$
\mathbf{S}_{\text{scaled}} = \frac{\mathbf{S}}{\sqrt{d_k}}
$$

**为什么需要缩放？**

假设 $\mathbf{q}$ 和 $\mathbf{k}$ 的各分量是均值为0、方差为1的独立随机变量，则：

$$
\text{Var}[\mathbf{q} \cdot \mathbf{k}] = \text{Var}\left[\sum_{i=1}^{d_k} q_i k_i\right] = d_k \cdot \text{Var}[q_i] \cdot \text{Var}[k_i] = d_k
$$

因此 $\mathbf{q} \cdot \mathbf{k}$ 的方差与 $d_k$ 成正比。当 $d_k$ 较大时，点积的量级会很大，导致Softmax进入饱和区域（梯度趋近于0）。

缩放因子 $\sqrt{d_k}$ 将点积的方差归一化为1。

#### Step 4：Softmax归一化

$$
\mathbf{A} = \text{softmax}(\mathbf{S}_{\text{scaled}}) \in \mathbb{R}^{n \times n}
$$

其中 $\text{softmax}$ 按行归一化：

$$
\mathbf{A}_{ij} = \frac{e^{\mathbf{S}_{ij}}}{\sum_{k=1}^{n} e^{\mathbf{S}_{ik}}}
$$

#### Step 5：加权求和

$$
\mathbf{Y} = \mathbf{A}\mathbf{V} \in \mathbb{R}^{n \times d_k}
$$

输出是值的加权平均，权重由注意力矩阵决定。

### 计算复杂度分析

| 操作 | 时间复杂度 | 空间复杂度 |
|------|-----------|-----------|
| $\mathbf{Q}, \mathbf{K}, \mathbf{V}$ 投影 | $O(n \cdot d \cdot d_k)$ | $O(n \cdot d_k)$ |
| $\mathbf{Q}\mathbf{K}^T$ | $O(n^2 \cdot d_k)$ | $O(n^2)$ |
| Softmax | $O(n^2)$ | $O(n^2)$ |
| $\mathbf{A}\mathbf{V}$ | $O(n^2 \cdot d_k)$ | $O(n \cdot d_k)$ |
| **总计** | **$O(n^2 \cdot d_k)$** | **$O(n^2)$** |

**核心瓶颈**：$O(n^2)$ 的空间和时间复杂度是处理长序列的主要障碍。

## Softmax数值稳定性

### 问题分析

Softmax函数为：

$$
\text{softmax}(x_i) = \frac{e^{x_i}}{\sum_j e^{x_j}}
$$

当 $x_i$ 很大时，$e^{x_i}$ 可能上溢为 $\infty$；当 $x_i$ 很小时，$e^{x_i}$ 下溢为 0，导致数值不稳定。

### Log-Sum-Exp技巧

数学恒等式：

$$
\text{softmax}(x_i) = \frac{e^{x_i - \max(x)}}{\sum_j e^{x_j - \max(x)}}
$$

减去最大值后，所有指数都在 $(0, 1]$ 范围内，避免上溢。

### PyTorch实现

```python
import torch
import torch.nn.functional as F

def stable_softmax(logits, dim=-1):
    """数值稳定的softmax实现"""
    # 减去最大值
    logits_minus_max = logits - logits.max(dim=dim, keepdim=True).values
    exp_logits = torch.exp(logits_minus_max)
    return exp_logits / exp_logits.sum(dim=dim, keepdim=True)

# PyTorch内置实现已经是数值稳定的
probs = F.softmax(logits, dim=-1)
```

## 多头注意力

### 数学定义

$$
\text{MultiHead}(\mathbf{Q}, \mathbf{K}, \mathbf{V}) = \text{Concat}(\text{head}_1, \ldots, \text{head}_h)\mathbf{W}^O
$$

其中每个注意力头：

$$
\text{head}_i = \text{Attention}(\mathbf{Q}\mathbf{W}_Q^i, \mathbf{K}\mathbf{W}_K^i, \mathbf{V}\mathbf{W}_V^i)
$$

### 多头的几何意义

| 视角 | 解释 |
|------|------|
| **子空间分解** | 每个头在不同的 $d_k$ 维子空间中计算注意力 |
| **多关系建模** | 不同头捕获不同类型的依赖关系（句法、语义、位置等） |
| **特征解耦** | 允许头之间学习独立的信息流 |
| **信息融合** | $\mathbf{W}^O$ 融合各头的输出 |

### 参数量分析

设 $d_{\text{model}} = 512$，$h = 8$，$d_k = d_v = 64$：

| 参数 | 数量 |
|------|------|
| $\mathbf{W}_Q^i, \mathbf{W}_K^i, \mathbf{W}_V^i$ | $3 \times h \times d_{\text{model}} \times d_k = 3 \times 512 \times 64 = 98,304$ |
| $\mathbf{W}^O$ | $h \times d_v \times d_{\text{model}} = 8 \times 64 \times 512 = 262,144$ |
| **总计** | **约360K参数**（与单头 $512 \times 512 = 262K$ 相当） |

### PyTorch实现

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, num_heads, dropout=0.1):
        super().__init__()
        assert d_model % num_heads == 0
        
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        self.scale = math.sqrt(self.d_k)
        
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, query, key, value, mask=None):
        B = query.size(0)
        
        # 线性投影并分头: (B, n, d_model) -> (B, h, n, d_k)
        Q = self.W_q(query).view(B, -1, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(key).view(B, -1, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(value).view(B, -1, self.num_heads, self.d_k).transpose(1, 2)
        
        # 注意力计算: (B, h, n, n)
        scores = torch.matmul(Q, K.transpose(-2, -1)) / self.scale
        
        if mask is not None:
            scores = scores.masked_fill(mask == 0, -1e9)
        
        attn_weights = F.softmax(scores, dim=-1)
        attn_weights = self.dropout(attn_weights)
        
        # 加权求和: (B, h, n, d_k) -> (B, n, d_model)
        context = torch.matmul(attn_weights, V)
        context = context.transpose(1, 2).contiguous().view(B, -1, self.d_model)
        
        return self.W_o(context)
```

## 位置编码

### 为什么需要位置编码

自注意力机制是**置换不变**的：打乱输入序列的顺序，输出不变。这与语言/图像的顺序敏感性矛盾，因此需要注入位置信息。

### Sinusoidal位置编码（原始Transformer）

Vaswani et al. (2017) 提出：

$$
PE_{(pos, 2i)} = \sin\left(\frac{pos}{10000^{2i/d_{\text{model}}}}\right)
$$

$$
PE_{(pos, 2i+1)} = \cos\left(\frac{pos}{10000^{2i/d_{\text{model}}}}\right)
$$

#### 核心性质

| 性质 | 公式 | 意义 |
|------|------|------|
| **唯一性** | $PE_{pos} \neq PE_{pos'}$ for $pos \neq pos'$ | 每个位置有唯一编码 |
| **有界性** | $PE \in [-1, 1]$ | 防止数值问题 |
| **可推广性** | 任意位置可外推 | 无需学习所有位置 |

#### 相对位置的几何表示

利用三角恒等式：

$$
\sin(\alpha + \beta) = \sin\alpha\cos\beta + \cos\alpha\sin\beta
$$
$$
\cos(\alpha + \beta) = \cos\alpha\cos\beta - \sin\alpha\sin\beta
$$

这意味着 $PE_{pos+k}$ 可由 $PE_{pos}$ 通过旋转矩阵得到，因此Sinusoidal编码**隐式**编码了相对位置信息。

#### PyTorch实现

```python
class PositionalEncoding(nn.Module):
    def __init__(self, d_model, max_len=5000, dropout=0.1):
        super().__init__()
        self.dropout = nn.Dropout(p=dropout)
        
        pe = torch.zeros(max_len, d_model)
        position = torch.arange(0, max_len).unsqueeze(1).float()
        div_term = torch.exp(torch.arange(0, d_model, 2).float() * 
                           (-math.log(10000.0) / d_model))
        pe[:, 0::2] = torch.sin(position * div_term)
        pe[:, 1::2] = torch.cos(position * div_term)
        pe = pe.unsqueeze(0)  # (1, max_len, d_model)
        self.register_buffer('pe', pe)
    
    def forward(self, x):
        # x: (batch_size, seq_len, d_model)
        x = x + self.pe[:, :x.size(1)]
        return self.dropout(x)
```

### Rotary Position Embedding (RoPE)

#### 设计目标

RoPE (Su et al., 2022) 寻找函数 $f_q(\mathbf{x}_m, m)$ 和 $f_k(\mathbf{x}_n, n)$，使得：

$$
\langle f_q(\mathbf{x}_m, m), f_k(\mathbf{x}_n, n) \rangle = g(\mathbf{x}_m, \mathbf{x}_n, m-n)
$$

即：**通过绝对位置编码实现相对位置感知**。

#### 2D情况推导

设旋转矩阵：

$$
\mathbf{R}_m = \begin{pmatrix}
\cos(m\theta) & -\sin(m\theta) \\
\sin(m\theta) & \cos(m\theta)
\end{pmatrix}
$$

则：

$$
\langle \mathbf{R}_m \mathbf{q}, \mathbf{R}_n \mathbf{k} \rangle = \langle \mathbf{R}_{m-n} \mathbf{q}, \mathbf{k} \rangle
$$

**关键性质**：旋转不改变点积的大小关系，只改变方向。这意味着相对位置信息被编码在点积中。

#### n维推广

将向量分成 $d/2$ 对，对每对应用独立的2D旋转：

$$
\mathbf{R}_{\Theta,m} = \begin{pmatrix}
\cos(m\theta_1) & -\sin(m\theta_1) & 0 & \cdots \\
\sin(m\theta_1) & \cos(m\theta_1) & 0 & \cdots \\
\vdots & \vdots & \ddots & \vdots
\end{pmatrix}
$$

#### 高效实现

使用逐元素运算替代矩阵乘法：

```python
def apply_rotary_pos_emb(x, cos, sin):
    """
    x: (batch, heads, seq_len, d_k) where d_k is even
    """
    d_k = x.shape[-1]
    x1 = x[..., :d_k//2]  # 奇数维度
    x2 = x[..., d_k//2:]  # 偶数维度
    
    # 旋转：x' = x * cos + (-x2, x1) * sin
    return torch.cat([
        x1 * cos - x2 * sin,
        x1 * sin + x2 * cos
    ], dim=-1)
```

#### RoPE vs Sinusoidal

| 特性 | Sinusoidal | RoPE |
|------|-----------|------|
| 编码方式 | 加到embedding | 旋转Q/K向量 |
| 相对位置 | 隐式 | **显式** |
| KV Cache兼容 | 需重新计算 | ✅ 自然兼容 |
| 长上下文 | 需外推方法 | NTK-Scaling等 |

## 高效注意力机制

### FlashAttention核心思想

**IO-Aware设计**：减少GPU HBM（高带宽内存）和SRAM之间的数据传输。

```
标准Attention:
1. 计算完整 QK^T → 存HBM (O(n²)空间)
2. Softmax → 存HBM
3. 乘V → 存HBM
4. 输出 → 存HBM

FlashAttention (Tiling):
1. 将Q/K/V分块读入SRAM
2. 逐块计算局部attention
3. 在线更新最终结果
4. 无需存储完整attention矩阵
```

### FlashAttention数学等价性

FlashAttention**精确**等价于标准attention，无近似误差。

### 复杂度改进

| 指标 | 标准 | FlashAttention |
|------|------|----------------|
| 时间 | $O(n^2d)$ | $O(n^2d)$（相同） |
| HBM访问 | $O(n^2)$ | $O(nd)$ |
| 显存 | $O(n^2)$ | **$O(n)$** |

### FlashAttention-2优化

- 更好的warp分工
- 更大的block size
- 更高效的softmax

```python
# 使用FlashAttention
from flash_attn import flash_attn_func

# Q, K, V: (batch, seq_len, num_heads, head_dim)
output = flash_attn_func(Q, K, V, causal=True)
```

## Transformer编码器与解码器

### 编码器结构

```python
class EncoderLayer(nn.Module):
    def __init__(self, d_model, num_heads, d_ff, dropout=0.1):
        super().__init__()
        self.self_attn = MultiHeadAttention(d_model, num_heads)
        self.ffn = nn.Sequential(
            nn.Linear(d_model, d_ff),
            nn.GELU(),
            nn.Linear(d_ff, d_model)
        )
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, x, mask=None):
        # Pre-norm架构（优于原始post-norm）
        x = x + self.dropout(self.self_attn(self.norm1(x), mask))
        x = x + self.dropout(self.ffn(self.norm2(x)))
        return x
```

### 解码器结构

```python
class DecoderLayer(nn.Module):
    def __init__(self, d_model, num_heads, d_ff, dropout=0.1):
        super().__init__()
        self.self_attn = MultiHeadAttention(d_model, num_heads)
        self.cross_attn = MultiHeadAttention(d_model, num_heads)
        self.ffn = nn.Sequential(
            nn.Linear(d_model, d_ff),
            nn.GELU(),
            nn.Linear(d_ff, d_model)
        )
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
        self.norm3 = nn.LayerNorm(d_model)
    
    def forward(self, x, encoder_output, src_mask=None, tgt_mask=None):
        # 掩码自注意力
        x = x + self.self_attn(self.norm1(x), tgt_mask)
        # 交叉注意力
        x = x + self.cross_attn(self.norm2(x), encoder_output, src_mask)
        # FFN
        x = x + self.ffn(self.norm3(x))
        return x
```

### Pre-Norm vs Post-Norm

| 架构 | 公式 | 特点 |
|------|------|------|
| **Post-Norm**（原始） | $x_{l+1} = \text{Norm}(x_l + \text{SubLayer}(x_l))$ | 训练不稳定，需warmup |
| **Pre-Norm**（现代） | $x_{l+1} = x_l + \text{SubLayer}(\text{Norm}(x_l))$ | 训练更稳定，效果更好 |

研究表明，Pre-Norm在深层网络中能更好地保持梯度稳定。[^1]

## 参考

[^1]: Nguyen, Salazar. "Transformers without Normalization". 2023.
[^2]: Su et al. "RoFormer: Enhanced Transformer with Rotary Position Embedding". 2022.
[^3]: Dao et al. "FlashAttention-2: Faster Attention with Better Parallelism and Work Partitioning". 2023.

---

*相关词条：[[transformer-and-attention|Transformer与注意力机制]]，[[llm/llm-theory|LLM理论]]，[[llm/transformer-evolution|Transformer演进]]*
