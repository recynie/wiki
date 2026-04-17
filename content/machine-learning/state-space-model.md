---
title: 状态空间模型与Mamba
date: 2026-04-17
description: SSM理论基础、Mamba选择性机制及与Transformer对比
tags:
  - deep-learning
  - ssm
  - mamba
  - state-space-model
draft: false
permalink:
---

# 状态空间模型与Mamba

状态空间模型（State Space Model, SSM）是一类用于建模时序数据的经典方法。2023年，Gu和Dao提出了Mamba，通过选择性机制将SSM扩展为可以与Transformer竞争的高效序列建模架构。

## SSM基础理论

### 连续时间状态空间模型

SSM将输入信号 $u(t) \in \mathbb{R}$ 映射到隐状态 $x(t) \in \mathbb{R}^N$ 和输出 $y(t) \in \mathbb{R}$：

$$
\begin{aligned}
\frac{dx(t)}{dt} &= \mathbf{A} x(t) + \mathbf{B} u(t) \\
y(t) &= \mathbf{C} x(t) + \mathbf{D} u(t)
\end{aligned}
$$

其中：
- $\mathbf{A} \in \mathbb{R}^{N \times N}$：状态转移矩阵
- $\mathbf{B} \in \mathbb{R}^{N \times 1}$：输入矩阵
- $\mathbf{C} \in \mathbb{R}^{1 \times N}$：输出矩阵
- $\mathbf{D} \in \mathbb{R}$：直接馈通（通常设为0）

### 离散化

计算机处理离散数据，需要将连续模型离散化。

**Zero-Order Hold (ZOH)**：

$$
x_{k+1} = \bar{\mathbf{A}} x_k + \bar{\mathbf{B}} u_k
$$
$$
y_k = \bar{\mathbf{C}} x_k + \bar{\mathbf{D}} u_k
$$

离散化参数：

$$
\bar{\mathbf{A}} = e^{\mathbf{A} \Delta t}
$$
$$
\bar{\mathbf{B}} = (\bar{\mathbf{A}} - I) \mathbf{A}^{-1} \mathbf{B}
$$

其中 $\Delta t$ 是采样间隔。

### S4模型的贡献

S4（Structured State Space Sequence Model）解决了离散SSM的两个关键问题：

1. **高效计算**：通过矩阵分解实现 $O(N)$ 线性复杂度
2. **长程依赖**：使用HiPPO矩阵初始化保证信息保留

## Mamba：选择性状态空间模型

### 核心洞察

传统SSM（如S4）的参数是**输入无关**的（input-independent）：

$$
x_{k+1} = \mathbf{A} x_k + \mathbf{B} u_k
$$

Mamba的关键创新是让参数变得**输入依赖**（input-dependent）：

$$
\begin{aligned}
x_{k+1} &= \mathbf{A}_k x_k + \mathbf{B}_k u_k \\
y_k &= \mathbf{C}_k x_k
\end{aligned}
$$

其中 $\mathbf{A}_k, \mathbf{B}_k, \mathbf{C}_k$ 是根据输入 $u_k$ 计算得到的。

### 选择性扫描算法

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np

class MambaBlock(nn.Module):
    """
    Mamba块的核心实现
    """
    def __init__(self, d_model, d_state=16, d_conv=4, expand=2):
        super().__init__()
        self.d_model = d_model
        self.d_state = d_state
        self.d_conv = d_conv
        self.d_inner = int(expand * d_model)
        
        # 投影输入
        self.in_proj = nn.Linear(d_model, self.d_inner * 2, bias=False)
        
        # 卷积层
        self.conv1d = nn.Conv1d(
            in_channels=self.d_inner,
            out_channels=self.d_inner,
            kernel_size=d_conv,
            padding=d_conv - 1,
            groups=self.d_inner,
        )
        
        # SSM参数投影
        # x -> (B, dt, C, D)
        self.x_proj = nn.Linear(self.d_inner, d_state * 2 + 1, bias=False)
        
        # dt投影
        self.dt_proj = nn.Linear(1, self.d_inner)
        
        # A矩阵（初始化为HiPPO风格）
        self.A_log = nn.Parameter(torch.arange(1, d_state + 1, dtype=torch.float).unsqueeze(0).repeat(self.d_inner, 1))
        self.D = nn.Parameter(torch.ones(self.d_inner))
        
        # 输出投影
        self.out_proj = nn.Linear(self.d_inner, d_model, bias=False)
    
    def selective_scan(self, x, dt, A, B, C, D):
        """
        选择性扫描算法
        x: (batch, seq_len, d_inner)
        dt: (batch, seq_len, d_inner)
        A: (d_inner, d_state)
        B: (batch, seq_len, d_state)
        C: (batch, seq_len, d_state)
        """
        batch, seq_len, d_inner = x.shape
        d_state = A.shape[1]
        
        # 离散化A, B
        # dt = softplus(dt) 确保正数
        dt = F.softplus(dt)
        dA = torch.exp(dt.unsqueeze(-1) * A)  # (batch, seq, d_inner, d_state)
        dB = dt.unsqueeze(-1) * B.unsqueeze(2)  # (batch, seq, d_inner, d_state)
        
        # 扫描
        h = torch.zeros(batch, d_inner, d_state, device=x.device, dtype=x.dtype)
        ys = []
        
        for i in range(seq_len):
            h = dA[:, i] * h + dB[:, i] * x[:, i].unsqueeze(-1)
            y = torch.einsum('bdn,bn->bd', h, C[:, i])
            ys.append(y)
        
        y = torch.stack(ys, dim=1)  # (batch, seq, d_inner)
        y = y + D * x
        
        return y
    
    def forward(self, x):
        # x: (batch, seq_len, d_model)
        batch, seq_len, d_model = x.shape
        
        # 输入投影并分割
        xz = self.in_proj(x)  # (batch, seq, d_inner * 2)
        x_inner, z = xz.chunk(2, dim=-1)  # 各 (batch, seq, d_inner)
        
        # 卷积
        x_conv = self.conv1d(x_inner.transpose(1, 2))[:, :, :seq_len].transpose(1, 2)
        x_conv = F.silu(x_conv)
        
        # 计算SSM参数
        x_proj_out = self.x_proj(x_conv)  # (batch, seq, d_state * 2 + 1)
        dt, B, C = x_proj_out.split([1, self.d_state, self.d_state], dim=-1)
        
        # dt归一化
        dt = F.softplus(self.dt_proj(dt))  # (batch, seq, d_inner)
        
        # 选择性扫描
        A = -torch.exp(self.A_log.float())  # (d_inner, d_state)
        y = self.selective_scan(x_conv, dt, A, B, C, self.D)
        
        # 门控
        y = y * F.silu(z)
        
        # 输出投影
        return self.out_proj(y)
```

### 选择性机制详解

**为什么选择性重要？**

考虑序列选择任务：
- **信息压缩**：选择性地将信息从输入压缩到状态
- **RNN行为**：隐状态可以无限期地记住重要信息
- **线性复杂度**：选择性的同时保持 $O(N)$ 计算

```python
# 选择性机制的效果示意
def selective_vs_fixed():
    """
    固定SSM vs 选择性SSM
    """
    # 固定SSM：所有输入同等对待
    # 状态更新：x_{k+1} = Ax_k + Bu_k
    # 参数A, B, C对所有输入相同
    
    # 选择性SSM：根据输入决定如何更新状态
    # x_{k+1} = A_k x_k + B_k u_k
    # 参数A_k, B_k, C_k依赖于输入
    
    """
    例如：处理 "The movie was [good/bad] and I [loved/hated] it"
    
    固定SSM：必须均匀地传播所有信息
    选择性SSM：可以专注于关键词，忽略无关词
    """
```

## Mamba vs Transformer

### 复杂度对比

| 特性 | Transformer | Mamba |
|------|-------------|-------|
| 注意力复杂度 | $O(N^2)$ | $O(N)$ |
| 推理速度 | 慢（KV cache大） | 快（5倍+） |
| 序列长度 | 受限于显存 | 可处理更长序列 |
| 并行训练 | 高效 | 高效 |

### 选择性机制的优势

1. **内容感知**：根据输入内容选择性地处理信息
2. **遗忘机制**：不需要显式的遗忘门
3. **推理效率**：RNN式生成，无KV cache

```python
# Transformer解码（自回归）
class TransformerDecoder:
    def forward(self, prefix_tokens):
        # 需要保存所有KV缓存
        for i in range(generate_len):
            # 每步需要注意力计算
            output = self.attention(q=current_token, 
                                   k=torch.cat([kv_cache]), 
                                   v=torch.cat([kv_cache]))
            kv_cache.append(output)
        # 显存使用: O(seq_len)

# Mamba解码
class MambaDecoder:
    def forward(self, prefix_tokens):
        # 隐状态压缩信息
        h = torch.zeros(batch, d_state)
        outputs = []
        for token in prefix_tokens:
            output, h = self.ssm(token, h)
            outputs.append(output)
        # 显存使用: O(d_state)，恒定
```

### 基准性能

| 任务 | Transformer | Mamba |
|------|-------------|-------|
| LRA | 0.67 | 0.70 |
| ImageNet | 91.0% | 91.5% |
| WikiText-103 | 20.5 PPL | 19.3 PPL |
| Pile (1B) | 15.1 PPL | 14.8 PPL |

## Mamba-2：SSM与Transformer的统一

### 状态空间对偶性（SSD）

Mamba-2提出了SSM与注意力机制的理论联系：

$$
\text{Attention}(Q, K, V) \iff \text{SSM}(A, B, C)
$$

核心发现：
- 注意力可以看作SSM的特殊形式
- 可以共享参数和计算

### Mamba-2架构

```python
class Mamba2Block(nn.Module):
    """
    Mamba-2块
    - 简化的选择机制
    - 更好的并行化
    """
    def __init__(self, d_model, d_state=128, expand=2):
        super().__init__()
        self.d_model = d_model
        self.d_state = d_state
        self.d_inner = expand * d_model
        
        # 输入投影
        self.in_proj = nn.Linear(d_model, self.d_inner * 2, bias=False)
        
        # SSM参数
        self.x_proj = nn.Linear(self.d_inner, d_state * 2 + 1, bias=False)
        
        # 简化的A矩阵（对角化）
        A = torch.arange(1, d_state + 1, dtype=torch.float)
        self.A = nn.Parameter(A.unsqueeze(0).repeat(self.d_inner, 1))
        
        self.D = nn.Parameter(torch.ones(self.d_inner))
        self.out_proj = nn.Linear(self.d_inner, d_model, bias=False)
    
    def forward(self, x):
        batch, seq_len, _ = x.shape
        
        xz = self.in_proj(x)
        x_inner, z = xz.chunk(2, dim=-1)
        
        # 参数投影
        x_proj_out = self.x_proj(F.silu(x_inner))
        dt, B, C = x_proj_out.split([1, self.d_state, self.d_state], dim=-1)
        dt = F.softplus(dt)
        
        # SSD扫描（并行化版本）
        y = self.ssd_scan(x_inner, dt, self.A, B, C, self.D)
        
        y = y * F.silu(z)
        return self.out_proj(y)
```

## 使用Mamba

### 安装

```bash
pip install mamba-ssm
```

### 预训练模型使用

```python
from mamba_ssm import MambaLMHeadModel
from transformers import AutoTokenizer

# 加载模型
model = MambaLMHeadModel.from_pretrained(
    'state-spaces/mamba-1.4b',
    device='cuda',
    dtype=torch.float16
)
tokenizer = AutoTokenizer.from_pretrained('EleutherAI/gpt-neox-20b')

# 生成
input_ids = tokenizer("The future of AI is", return_tensors="pt")['input_ids'].to('cuda')

generated_ids = model.generate(
    input_ids,
    max_length=100,
    temperature=0.9,
    top_p=0.9,
    eos_token_id=tokenizer.eos_token_id
)

print(tokenizer.decode(generated_ids[0]))
```

### 自定义Mamba层

```python
class MambaLayer(nn.Module):
    def __init__(self, d_model, d_state=16, expand=2):
        super().__init__()
        self.mamba = MambaBlock(d_model, d_state, expand=expand)
        self.norm = nn.LayerNorm(d_model)
    
    def forward(self, x):
        # Pre-norm架构
        return x + self.mamba(self.norm(x))

class MambaClassifier(nn.Module):
    def __init__(self, vocab_size, d_model, num_classes, n_layers):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, d_model)
        self.layers = nn.ModuleList([
            MambaLayer(d_model) for _ in range(n_layers)
        ])
        self.classifier = nn.Linear(d_model, num_classes)
    
    def forward(self, x):
        embedded = self.embedding(x)
        for layer in self.layers:
            embedded = layer(embedded)
        
        # 使用最后位置的表示
        return self.classifier(embedded[:, -1])
```

## 应用场景

### 长文本处理

Mamba的 $O(N)$ 复杂度使其适合处理超长文档：
- 书籍摘要
- 视频帧序列
- 蛋白质序列

### 时间序列预测

```python
class TimeSeriesMamba(nn.Module):
    def __init__(self, n_features, d_model, n_layers, d_state=16):
        super().__init__()
        self.proj = nn.Linear(n_features, d_model)
        self.layers = nn.ModuleList([
            MambaBlock(d_model, d_state) for _ in range(n_layers)
        ])
        self.fc = nn.Linear(d_model, n_features)
    
    def forward(self, x):
        # x: (batch, seq_len, n_features)
        x = self.proj(x)
        for layer in self.layers:
            x = layer(x)
        return self.fc(x)
```

### 音频处理

Mamba在音频建模中也表现出色：
- 语音识别
- 音乐生成
- 音频压缩

## 参考

[^1]: Gu & Dao, "Mamba: Linear-Time Sequence Modeling with Selective State Spaces", 2023
[^2]: Dao & Gu, "Mamba-2: Scalable State Space Sequence Modeling", 2024
[^3]: Gu et al., "Efficiently Modeling Long Sequences with Structured State Spaces", ICLR 2022
[^4]: Mamba Official Implementation. https://github.com/state-spaces/mamba
[^5]: State Space Models Blog Series. https://hazyint.studio/
