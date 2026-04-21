---
title: 线性代数视角下的神经网络层
date: 2026-04-21
description: 从矩阵运算角度理解神经网络各层的本质
tags:
  - deep-learning
  - linear-algebra
  - neural-network
  - convolution
  - attention
draft: false
permalink:
---

# 线性代数视角下的神经网络层

神经网络可以看作是一系列线性变换与非线性激活的交织。本章从线性代数的视角，分析各类神经网络层的数学本质，揭示它们之间的内在联系。

## 全连接层

### 数学定义

全连接层（Fully Connected Layer / Dense Layer）是最基础的神经网络层：

$$
\mathbf{y} = \sigma(\mathbf{W}\mathbf{x} + \mathbf{b})
$$

其中：
- $\mathbf{x} \in \mathbb{R}^{d_{in}}$：输入向量
- $\mathbf{W} \in \mathbb{R}^{d_{in} \times d_{out}}$：权重矩阵
- $\mathbf{b} \in \mathbb{R}^{d_{out}}$：偏置向量
- $\sigma$：逐元素激活函数

### 几何解释

全连接层实现了**仿射变换**（Affine Transformation）：

1. **线性变换**：$\mathbf{W}\mathbf{x}$ 将输入向量旋转、反射、缩放、投影
2. **平移**：$+\mathbf{b}$ 将原点移动

**几何变换示例**：

| 权重矩阵类型 | 几何效果 |
|-------------|---------|
| 正交矩阵 $\mathbf{W}^T\mathbf{W} = \mathbf{I}$ | 纯旋转/反射 |
| 对角矩阵 | 沿坐标轴缩放 |
| 稀疏矩阵 | 选择性丢弃某些维度 |

### PyTorch实现

```python
import torch
import torch.nn as nn

# 手动实现
def fc_layer(x, W, b, sigma):
    """
    x: (batch, d_in)
    W: (d_in, d_out)
    b: (d_out,)
    """
    return sigma(x @ W + b)

# nn.Module实现
class FullyConnected(nn.Module):
    def __init__(self, d_in, d_out):
        super().__init__()
        self.weight = nn.Parameter(torch.randn(d_in, d_out) * 0.01)
        self.bias = nn.Parameter(torch.zeros(d_out))
    
    def forward(self, x):
        return torch.relu(x @ self.weight + self.bias)
```

## 卷积层

### 卷积的矩阵形式

卷积操作可以通过**Toeplitz矩阵**转化为矩阵乘法。

#### 一维卷积

对于输入 $\mathbf{x} \in \mathbb{R}^n$ 和卷积核 $\mathbf{k} \in \mathbb{R}^k$：

$$
y_i = \sum_{j=1}^{k} k_j \cdot x_{i+j-1}
$$

Toeplitz矩阵 $\mathbf{C} \in \mathbb{R}^{n \times (n+k-1)}$：

$$
\mathbf{C} = \begin{pmatrix}
k_1 & k_2 & k_3 & \cdots & k_k & 0 & \cdots & 0 \\
0 & k_1 & k_2 & k_3 & \cdots & k_k & \cdots & 0 \\
\vdots & \ddots & \ddots & \ddots & \ddots & \ddots & \ddots & \vdots \\
0 & \cdots & 0 & k_1 & k_2 & k_3 & \cdots & k_k
\end{pmatrix}
$$

则卷积等价于：

$$
\mathbf{y} = \mathbf{C} \cdot \mathbf{x}_{\text{padded}}
$$

#### 二维卷积（im2col）

对于图像 $\mathbf{X} \in \mathbb{R}^{H \times W}$ 和卷积核 $\mathbf{K} \in \mathbb{R}^{k_h \times k_w}$：

im2col将每个滑动窗口展开为一行：

$$
\mathbf{X}_{\text{col}} = \begin{pmatrix}
x_{11} & x_{12} & \cdots & x_{1k_w} & x_{21} & \cdots & x_{k_h k_w} \\
x_{12} & x_{13} & \cdots & x_{1,k_w+1} & x_{22} & \cdots & x_{k_h+1,k_w+1} \\
\vdots & \vdots & \ddots & \vdots & \vdots & \ddots & \vdots
\end{pmatrix} \in \mathbb{R}^{N_{\text{patches}} \times (k_h \cdot k_w)}
$$

卷积核展成列向量：

$$
\mathbf{k} = \text{vec}(\mathbf{K}) \in \mathbb{R}^{k_h \cdot k_w}
$$

则输出为：

$$
\mathbf{y} = \mathbf{X}_{\text{col}} \cdot \mathbf{k}
$$

### 卷积的几何意义

| 视角 | 解释 |
|------|------|
| **线性代数** | 输入与卷积核的稀疏矩阵乘法 |
| **信号处理** | 相关性/卷积运算 |
| **计算机视觉** | 局部特征提取器 |
| **群论** | 等变变换（Equivariant Transformation） |

### PyTorch卷积实现

```python
import torch
import torch.nn as nn

# 标准2D卷积
conv2d = nn.Conv2d(in_channels=3, out_channels=64, kernel_size=3, padding=1)

# im2col实现示意
def im2col(x, kernel_size, stride, padding):
    """
    将2D图像转换为矩阵列
    x: (B, C, H, W)
    """
    x = torch.nn.functional.pad(x, (padding,)*4)
    B, C, H, W = x.shape
    
    # 计算输出尺寸
    out_H = (H - kernel_size) // stride + 1
    out_W = (W - kernel_size) // stride + 1
    
    # 使用unfold进行im2col
    cols = x.unfold(2, kernel_size, stride).unfold(3, kernel_size, stride)
    cols = cols.contiguous().view(B, C * kernel_size * kernel_size, -1)
    
    return cols, (out_H, out_W)

# 手写卷积（用于理解）
def conv2d_manual(x, weight, bias, stride=1, padding=0):
    B, C, H, W = x.shape
    K, _, kH, kW = weight.shape
    
    x_padded = torch.nn.functional.pad(x, (padding,)*4)
    out_H = (H + 2*padding - kH) // stride + 1
    out_W = (W + 2*padding - kW) // stride + 1
    
    out = torch.zeros(B, K, out_H, out_W)
    
    for i in range(out_H):
        for j in range(out_W):
            h_start = i * stride
            w_start = j * stride
            window = x_padded[:, :, h_start:h_start+kH, w_start:w_start+kW]
            out[:, :, i, j] = (window.unsqueeze(1) * weight).sum(dim=(2,3,4)) + bias
    
    return out
```

## 注意力层

### Self-Attention的矩阵形式

Self-Attention是Transformer的核心组件：

$$
\text{Attention}(\mathbf{Q}, \mathbf{K}, \mathbf{V}) = \text{softmax}\left(\frac{\mathbf{Q}\mathbf{K}^T}{\sqrt{d_k}}\right)\mathbf{V}
$$

设 $\mathbf{X} \in \mathbb{R}^{n \times d}$ 为输入序列，则：

$$
\mathbf{Q} = \mathbf{X}\mathbf{W}_Q, \quad \mathbf{K} = \mathbf{X}\mathbf{W}_K, \quad \mathbf{V} = \mathbf{X}\mathbf{W}_V
$$

**矩阵形式分解**：

1. **投影**：$\mathbf{Q}, \mathbf{K}, \mathbf{V}$ 是 $\mathbf{X}$ 的三个不同线性投影
2. **相似度矩阵**：$\mathbf{S} = \mathbf{Q}\mathbf{K}^T \in \mathbb{R}^{n \times n}$
3. **注意力权重**：$\mathbf{A} = \text{softmax}(\mathbf{S}/\sqrt{d_k})$
4. **加权聚合**：$\mathbf{Y} = \mathbf{A}\mathbf{V}$

### 注意力的线性代数视角

| 视角 | 解释 |
|------|------|
| **矩阵分解** | $\mathbf{Y} \approx \mathbf{Q}(\mathbf{K}^T\mathbf{V})$ |
| **图神经网络** | 每个token是从所有其他token聚合信息 |
| **核方法** | softmax是径向基函数核的归一化形式 |
| **信息检索** | Query-Key-Value对应检索系统 |

### 多头注意力的矩阵拼接

$$
\text{MultiHead}(\mathbf{X}) = \text{Concat}(\text{head}_1, \ldots, \text{head}_h)\mathbf{W}^O
$$

其中每个头的输出：

$$
\text{head}_i = \text{Attention}(\mathbf{X}\mathbf{W}_Q^i, \mathbf{X}\mathbf{W}_K^i, \mathbf{X}\mathbf{W}_V^i)
$$

### PyTorch实现

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class SelfAttention(nn.Module):
    def __init__(self, d_model, num_heads):
        super().__init__()
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
    
    def forward(self, x, mask=None):
        B, N, C = x.shape
        
        # 线性投影并分头
        Q = self.W_q(x).view(B, N, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(x).view(B, N, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(x).view(B, N, self.num_heads, self.d_k).transpose(1, 2)
        
        # 缩放点积注意力
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        if mask is not None:
            scores = scores.masked_fill(mask == 0, -1e9)
        
        attn_weights = F.softmax(scores, dim=-1)
        context = torch.matmul(attn_weights, V)
        
        # 合并多头
        context = context.transpose(1, 2).contiguous().view(B, N, C)
        
        return self.W_o(context)
```

## Embedding层

### 数学定义

Embedding层本质上是一个**查表操作**：

$$
\mathbf{E} \in \mathbb{R}^{|V| \times d}, \quad \mathbf{e}_i = \mathbf{E}[i, :]
$$

给定词索引 $w$，输出对应词向量：

$$
\mathbf{v}_w = \mathbf{E}[w, :]
$$

### 矩阵视角

Embedding可以看作是一个**稀疏矩阵乘法**：

$$
\mathbf{Y} = \mathbf{X}_{\text{one-hot}} \mathbf{E}
$$

其中 $\mathbf{X}_{\text{one-hot}} \in \mathbb{R}^{B \times |V|}$ 是one-hot编码矩阵。

**实际实现优化**：直接用索引查表，避免稀疏矩阵运算。

### 几何解释

- 语义相似的词在向量空间中距离更近
- 词向量之间的夹角反映语义关系
- 线性关系（如 $\mathbf{v}_{\text{king}} - \mathbf{v}_{\text{man}} \approx \mathbf{v}_{\text{queen}}$）体现语言规律

### PyTorch实现

```python
import torch
import torch.nn as nn

class TokenEmbedding(nn.Module):
    def __init__(self, vocab_size, d_model):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, d_model)
        self.scale = math.sqrt(d_model)
    
    def forward(self, x):
        # x: (batch_size, seq_len) 整数索引
        return self.embedding(x) * self.scale

# 位置编码也可以看作是一种embedding
class PositionalEncoding(nn.Module):
    def __init__(self, d_model, max_len=5000):
        super().__init__()
        pe = torch.zeros(max_len, d_model)
        position = torch.arange(0, max_len).unsqueeze(1).float()
        div_term = torch.exp(torch.arange(0, d_model, 2).float() * (-math.log(10000.0) / d_model))
        pe[:, 0::2] = torch.sin(position * div_term)
        pe[:, 1::2] = torch.cos(position * div_term)
        self.register_buffer('pe', pe.unsqueeze(0))
    
    def forward(self, x):
        # x: (batch_size, seq_len, d_model)
        return x + self.pe[:, :x.size(1)]
```

## 归一化层

### BatchNorm的矩阵形式

对于形状 $(B, C, H, W)$ 的张量：

$$
\mu_c = \frac{1}{BHW}\sum_{b,i,j} x_{b,c,i,j}, \quad \sigma_c^2 = \frac{1}{BHW}\sum_{b,i,j}(x_{b,c,i,j} - \mu_c)^2
$$

$$
\hat{x}_{b,c,i,j} = \frac{x_{b,c,i,j} - \mu_c}{\sqrt{\sigma_c^2 + \epsilon}}
$$

$$
y_{b,c,i,j} = \gamma_c \hat{x}_{b,c,i,j} + \beta_c
$$

### LayerNorm的矩阵形式

对于形状 $(B, L, D)$ 的张量（Transformer采用）：

$$
\mu^{(i)} = \frac{1}{D}\sum_{d=1}^{D} x_d^{(i)}, \quad \sigma^{(i)} = \sqrt{\frac{1}{D}\sum_{d=1}^{D}(x_d^{(i)} - \mu^{(i)})^2}
$$

$$
\hat{x}^{(i)} = \frac{x^{(i)} - \mu^{(i)}}{\sigma^{(i)} + \epsilon}
$$

**矩阵视角**：LayerNorm对每个样本的嵌入维度进行归一化，相当于在特征空间做单位球面投影。

### RMSNorm

$$
\hat{x} = \frac{x}{\text{RMS}(x)}, \quad \text{RMS}(x) = \sqrt{\frac{1}{D}\sum_{d=1}^{D}x_d^2}
$$

去掉均值中心化，保留RMS归一化，计算更高效。

## Dropout与矩阵稀疏化

### Dropout的矩阵形式

训练时以概率 $p$ 丢弃神经元：

$$
\mathbf{y} = \mathbf{M} \odot \sigma(\mathbf{W}\mathbf{x} + \mathbf{b})
$$

其中 $\mathbf{M} \sim \text{Bernoulli}(1-p)$ 是掩码矩阵。

### Dropout的期望

$$
\mathbb{E}[\mathbf{y}] = (1-p) \cdot \sigma(\mathbf{W}\mathbf{x} + \mathbf{b})
$$

因此推理时使用 $1/(1-p)$ 的缩放，保持期望一致。

### Dropout的几何解释

- 强制网络不依赖单个神经元
- 在高维空间中进行稀疏采样
- 近似贝叶斯推断中的模型集成

## 层之间的联系

### 全连接 vs 卷积

| 特性 | 全连接层 | 卷积层 |
|------|---------|--------|
| 权重共享 | 无 | 局部权重共享 |
| 稀疏性 | 密集连接 | 稀疏连接 |
| 等变性 | 不具备 | 具有平移等变性 |
| 参数量 | $O(d_{in} \cdot d_{out})$ | $O(k_h \cdot k_w \cdot c_{in} \cdot c_{out})$ |

### 卷积 vs 注意力

| 特性 | 卷积层 | 注意力层 |
|------|--------|---------|
| 感受野 | 局部（kernel大小） | 全局（整个序列） |
| 连接模式 | 固定（局部邻域） | 自适应（数据驱动） |
| 计算复杂度 | $O(n \cdot k^2 \cdot c)$ | $O(n^2 \cdot d)$ |

### 现代架构的统一视图

**MetaFormer**（Yu et al., 2022）提出：Transformer = Token Mixer + Channel Mixer

| 组件 | Token Mixer | Channel Mixer |
|------|-------------|---------------|
| Transformer | Self-Attention | FFN |
| CNN | Convolution | 1x1 Conv |
| MLP-Mixer | 线性层（跨通道） | 线性层（跨Token） |

## 参考

[^1]: Dumoulin, Visin. "A guide to convolution arithmetic for deep learning". 2016.
[^2]: Vaswani et al. "Attention Is All You Need". NeurIPS 2017.
[^3]: Yu et al. "MetaFormer is Actually What You Need for Vision". CVPR 2022.

---

*相关词条：[[transformer-and-attention|Transformer与注意力机制]]，[[resnet|ResNet与残差学习]]，[[lstm|LSTM门控机制]]*
