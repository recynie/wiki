---
title: Transformer 架构演进
date: 2026-04-14
description: 从 2017 到 2025，Transformer 架构的演进历程
tags:
  - transformer
  - llm
  - architecture
  - attention
draft: false
permalink:
---

## 概述

2017 年，Google 在论文《Attention Is All You Need》[^1] 中首次提出了 Transformer 架构，这一革新性设计彻底改变了自然语言处理（NLP）乃至整个深度学习领域的格局。Transformer 摒弃了传统的循环神经网络（RNN）结构，完全基于注意力机制（Attention Mechanism），实现了并行计算与长距离依赖建模的双重优势。

Transformer 的核心贡献在于：

- **多头自注意力机制**（Multi-Head Self-Attention）：允许模型同时关注不同位置的表示，学习序列内部的复杂关系
- **并行化训练**：摆脱了 RNN 的时序依赖，大幅提升训练效率
- **可扩展性**：为后续大规模语言模型奠定了架构基础

从 BERT[^2] 到 GPT 系列[^3]，从 LLaMA[^4] 到 Gemini[^5]，Transformer 架构经历了持续的优化与演进。本文将系统梳理其关键技术的发展脉络。

## 原始 Transformer（2017）

### 架构概述

原始 Transformer 采用 **Encoder-Decoder** 结构，最初设计用于序列到序列（Sequence-to-Sequence）的翻译任务。

```
┌─────────────────────────────────────────────────────────┐
│                      Transformer                         │
├────────────────────────┬────────────────────────────────┤
│       Encoder          │           Decoder              │
│  ┌──────────────────┐  │  ┌──────────────────────────┐   │
│  │  Multi-Head       │  │  │  Masked Multi-Head      │   │
│  │  Self-Attention  │  │  │  Self-Attention         │   │
│  └────────┬─────────┘  │  └────────────┬───────────┘   │
│           │              │               │               │
│           ▼              │               ▼               │
│  ┌──────────────────┐  │  ┌──────────────────────────┐   │
│  │  Feed-Forward    │  │  │  Encoder-Decoder         │   │
│  │  Network (FFN)   │  │  │  Attention               │   │
│  └────────┬─────────┘  │  └────────────┬───────────┘   │
│           │              │               │               │
│           ▼              │               ▼               │
│  ┌──────────────────┐  │  ┌──────────────────────────┐   │
│  │  Add & Norm      │  │  │  Feed-Forward            │   │
│  │  (Post-norm)     │  │  │  Network                 │   │
│  └──────────────────┘  │  └────────────┬───────────┘   │
│                        │               │               │
│                        │               ▼               │
│                        │  ┌──────────────────────────┐   │
│                        │  │  Add & Norm (Post-norm)  │   │
│                        │  └──────────────────────────┘   │
└────────────────────────┴────────────────────────────────┘
```

### Post-norm 与残差连接

原始 Transformer 采用 **Post-norm** 结构，即每个子层的输出先经过残差连接，再进行 Layer Normalization：

$$
\mathbf{x}{\ell} = \text{LayerNorm}(\mathbf{x}{\ell-1} + \text{SubLayer}(\mathbf{x}_{\ell-1}))
$$

其中 $\text{SubLayer}(\cdot)$ 可以是 Multi-Head Attention 或 Feed-Forward Network。

### 位置编码

原始 Transformer 使用**正弦位置编码**（Sinusoidal Position Encoding）[^1]：

$$
PE_{(pos, 2i)} = \sin\left(\frac{pos}{10000^{2i/d_{\text{model}}}}\right)
$$
$$
PE_{(pos, 2i+1)} = \cos\left(\frac{pos}{10000^{2i/d_{\text{model}}}}\right)
$$

这种编码方式具有以下特点：

- **确定性**：无需学习，直接计算
- **外推性**：理论上可以处理任意长度的序列（通过正弦函数的周期性）
- **相对位置信息**：不同位置编码的內积可以编码相对距离

### C++ 实现示例

以下是一个简化的 Multi-Head Attention C++ 实现：

```cpp
#include <bits/stdc++.h>
using namespace std;

struct MultiHeadAttention {
    int d_model, num_heads, d_k;
    vector<vector<vector<double>>> W_q, W_k, W_v, W_o;
    
    MultiHeadAttention(int d_model, int num_heads) 
        : d_model(d_model), num_heads(num_heads), d_k(d_model / num_heads) {
        // 初始化权重矩阵
        auto init_weights = [&](auto& W) {
            W.resize(num_heads);
            for (int h = 0; h < num_heads; ++h) {
                W[h] = vector<vector<double>>(d_model, vector<double>(d_k));
                for (int i = 0; i < d_model; ++i)
                    for (int j = 0; j < d_k; ++j)
                        W[h][i][j] = (rand() % 1000 - 500) / 500.0 * sqrt(2.0 / d_k);
            }
        };
        init_weights(W_q);
        init_weights(W_k);
        init_weights(W_v);
        init_weights(W_o);
    }
    
    vector<vector<double>> forward(
        const vector<vector<double>>& X,
        const vector<vector<double>>& mask = {}) {
        
        int seq_len = X.size();
        vector<vector<double>> output(seq_len, vector<double>(d_model, 0.0));
        
        for (int h = 0; h < num_heads; ++h) {
            // 计算 Q, K, V
            vector<vector<double>> Q(num_heads, vector<double>(seq_len * d_k));
            vector<vector<double>> K(num_heads, vector<double>(seq_len * d_k));
            vector<vector<double>> V(num_heads, vector<double>(seq_len * d_k));
            
            // 计算注意力分数
            vector<vector<double>> scores(seq_len, vector<double>(seq_len));
            for (int i = 0; i < seq_len; ++i)
                for (int j = 0; j < seq_len; ++j) {
                    double sum = 0.0;
                    for (int k = 0; k < d_k; ++k)
                        sum += Q[h][i * d_k + k] * K[h][j * d_k + k];
                    scores[i][j] = sum / sqrt(d_k);
                }
            
            // Softmax + 输出
            for (int i = 0; i < seq_len; ++i) {
                double max_val = *max_element(scores[i].begin(), scores[i].end());
                double sum = 0.0;
                for (int j = 0; j < seq_len; ++j)
                    scores[i][j] = exp(scores[i][j] - max_val);
                for (int j = 0; j < seq_len; ++j)
                    sum += scores[i][j];
                for (int j = 0; j < seq_len; ++j)
                    scores[i][j] /= sum;
            }
        }
        return output;
    }
};
```

## 关键架构演进

### Pre-Norm vs Post-Norm

在 Transformer 的发展中，**Pre-Norm**（也称 Pre-Layer Normalization）逐渐取代 Post-norm 成为主流选择。

| 特性 | Post-norm | Pre-norm |
|------|-----------|----------|
| 归一化位置 | 残差连接之后 | 残差连接之前 |
| 计算顺序 | $\text{LayerNorm}(x + \text{SubLayer}(x))$ | $x + \text{SubLayer}(\text{LayerNorm}(x))$ |
| 梯度稳定性 | 训练初期梯度较小 | 梯度更稳定 |
| 使用情况 | 原始 Transformer | GPT-2, LLaMA, BERT 等 |

Pre-norm 的数学表达式为：

$$
\mathbf{x}{\ell} = \mathbf{x}{\ell-1} + \text{SubLayer}\left(\text{LayerNorm}(\mathbf{x}_{\ell-1})\right)
$$

这种设计使得：

1. **梯度流动更顺畅**：每个子层的输入已经被归一化，梯度不会经过剧烈的数值变化
2. **训练更稳定**：适合更深层的网络
3. **内存效率略低**：需要保存每层的归一化输入（但现代框架已有优化）

### RMSNorm

**RMSNorm**（Root Mean Square Normalization）[^6] 由 Tencent AI Lab 于 2018 年提出，是一种更高效的归一化方案。

传统的 LayerNorm 计算：

$$
\bar{a}i = \frac{a_i - \mu}{\sqrt{\sigma^2 + \epsilon}} \cdot g_i, \quad \mu = \frac{1}{n}\sum_{i=1}^n a_i, \quad \sigma^2 = \frac{1}{n}\sum_{i=1}^n (a_i - \mu)^2
$$

RMSNorm 只需计算 RMS（均方根）：

$$
\bar{a}_i = \frac{a_i}{\text{RMS}(\mathbf{a})} \cdot g_i, \quad \text{RMS}(\mathbf{a}) = \sqrt{\frac{1}{n}\sum_{i=1}^n a_i^2}
$$

**核心观察**：LayerNorm 中的均值 centering 贡献较小，移除后性能几乎不变，但计算量减少约 7-64%。

```python
import torch
import torch.nn as nn

class RMSNorm(nn.Module):
    def __init__(self, dim: int, eps: float = 1e-6):
        super().__init__()
        self.eps = eps
        self.weight = nn.Parameter(torch.ones(dim))
    
    def forward(self, x: torch.Tensor) -> torch.Tensor:
        # RMSNorm: 只计算均方根，省略均值
        rms = torch.rsqrt(x.pow(2).mean(-1, keepdim=True) + self.eps)
        return x * rms * self.weight
```

### Grouped-Query Attention（GQA）

**Grouped-Query Attention**[^7] 是对 Multi-Query Attention（MQA）的扩展，在效率与质量之间取得平衡。

| 注意力类型 | Key 头数 | Query 头数 | 特点 |
|-----------|---------|-----------|------|
| Multi-Head Attention (MHA) | $H$ | $H$ | 参数量大，质量高 |
| Multi-Query Attention (MQA) | 1 | $H$ | 参数量小，速度快，质量略降 |
| Grouped-Query Attention (GQA) | $G$ ($1 < G < H$) | $H$ | 平衡方案 |

其中 $G$ 是 Key（和 Value）头的数量，$H$ 是 Query 头的数量。通常取 $G = H/4$ 或 $G = 8$。

数学上，对于第 $h$ 个 Query 头：

$$
\text{Attention}(\mathbf{Q}_h, \mathbf{K}, \mathbf{V}) = \text{softmax}\left(\frac{\mathbf{Q}_h \mathbf{K}^T}{\sqrt{d_k}}\right) \mathbf{V}
$$

GQA 将 Key/Value 头分组，每组共享相同的 Key 和 Value：

$$
\text{Attention}_h = \text{Attention}(\mathbf{Q}_h, \mathbf{K}_{\lfloor h \cdot G / H \rfloor}, \mathbf{V}_{\lfloor h \cdot G / H \rfloor})
$$

## 位置编码演进

### 演进历程

| 年份 | 方案 | 特点 |
|------|------|------|
| 2017 | Sinusoidal | 确定性，周期性，外推性强 |
| 2018 | Learned Absolute PE | 可学习，但外推性差 |
| 2018-2020 | Relative Position Bias (T5) | 编码相对位置 |
| 2021 | RoPE (Rotary Position Embedding) | 旋转矩阵，无显式绝对位置 |
| 2023 | Position Interpolation | RoPE 扩展，支持更长上下文 |
| 2024 | YaRN, LongRoPE | 进一步优化长上下文 |

### RoPE 旋转位置编码

**RoPE**[^8] 由 Su Jianlin 等人在 2021 年提出，核心思想是用旋转矩阵编码位置信息。

对于二维子空间（$d=2$），RoPE 的定义为：

$$
\mathbf{R}(\theta, m) = \begin{pmatrix} \cos(m\theta) & -\sin(m\theta) \\ \sin(m\theta) & \cos(m\theta) \end{pmatrix}
$$

对于 $d$ 维向量，将其划分为 $d/2$ 个二维子空间，每个子空间应用不同的旋转：

$$
\mathbf{q}_m = \mathbf{R}(\Theta, m) \mathbf{q}, \quad \mathbf{k}_n = \mathbf{R}(\Theta, n) \mathbf{k}
$$

其中 $\Theta = {\theta_i = b^{-2i/d} \dots}$，$b$ 通常取 10000$。

**关键性质**：两个旋转向量的內积只依赖于相对位置 $m-n$：

$$
\langle \mathbf{R}(\Theta, m)\mathbf{q}, \mathbf{R}(\Theta, n)\mathbf{k} \rangle = \langle \mathbf{q}, \mathbf{R}(\Theta, n-m)\mathbf{k} \rangle
$$

这使得 RoPE 能够自然地编码相对位置信息，同时保持绝对位置的可解释性。

### RoPE 扩展：位置插值

当需要扩展上下文窗口时，直接外推会导致性能下降。**位置插值**[^9]（Position Interpolation）通过缩放位置索引来适应更长的上下文：

$$
\text{PI}: pos \rightarrow \frac{pos}{L_{new}/L_{old}} = \frac{pos \cdot L_{old}}{L_{new}}
$$

例如，将 4096 上下文扩展到 32768 时，只需将位置索引除以 8。

后续工作如 **YaRN**[^10] 和 **LongRoPE**[^11] 进一步优化了插值策略，减少了对短上下文的性能影响。

## 主流模型家族

### BERT（2018）：Encoder-Only

**BERT**（Bidirectional Encoder Representations from Transformers）[^2] 由 Google 提出，革新了 NLP 预训练范式。

**核心特点**：

- **架构**：纯 Encoder，12/24 层，768/1024 隐藏维度
- **预训练任务**：掩码语言模型（Masked Language Modeling, MLM）+ 下句预测（Next Sentence Prediction, NSP）
- **创新点**：双向注意力，首次展示"预训练 + 微调"范式的威力

```python
# BERT MLM 示意
def bert_mlm_loss(logits, labels, mask):
    """
    logits: [batch, seq_len, vocab_size]
    labels: [batch, seq_len]  - 被 mask 的位置有标签
    mask: [batch, seq_len]    - 标记哪些位置需要计算 loss
    """
    loss_fn = nn.CrossEntropyLoss(reduction='none')
    loss = loss_fn(logits.view(-1, logits.size(-1)), labels.view(-1))
    loss = (loss * mask.view(-1)).sum() / mask.sum()
    return loss
```

**相关链接**：[[../machine-learning/recommendation-systems|推荐系统]] 中广泛使用了 BERT 的文本编码能力。

### GPT 系列（2018-2024）：Decoder-Only

**GPT**（Generative Pre-trained Transformer）[^3] 系列代表了自回归语言模型的演进：

| 模型 | 年份 | 层数 | 参数量 | 关键创新 |
|------|------|------|--------|----------|
| GPT-1 | 2018 | 12 | 117M | 初代 GPT，堆叠式预训练 |
| GPT-2 | 2019 | 48 | 1.5B | 多任务学习，Zero-shot |
| GPT-3 | 2020 | 96 | 175B | In-context Learning |
| GPT-4 | 2023 | - | - | 多模态，RLHF |
| GPT-4o | 2024 | - | - | 原生多模态，实时推理 |

**GPT 的核心特点**：

- **架构**：Decoder-only，使用**因果掩码**（Causal Mask）确保自回归生成
- **预训练任务**：下一个 Token 预测（Next Token Prediction）
- **涌现能力**：随着规模增大，出现 In-context Learning、Chain-of-Thought 等能力

```cpp
// GPT 因果注意力掩码
vector<vector<double>> create_causal_mask(int seq_len) {
    vector<vector<double>> mask(seq_len, vector<double>(seq_len, 0.0));
    for (int i = 0; i < seq_len; ++i)
        for (int j = i + 1; j < seq_len; ++j)
            mask[i][j] = -1e9;  // 掩码掉未来位置
    return mask;
}
```

### LLaMA（2023-）：开源领袖

**LLaMA**[^4] 由 Meta AI 提出，是开源大模型的重要里程碑。

**架构特点**：

- **Pre-norm + RMSNorm**：结合了 Pre-norm 的稳定性和 RMSNorm 的效率
- **RoPE**：使用旋转位置编码，支持较长上下文
- **SwiGLU 激活函数**：Swish + Gated Linear Unit，提升性能
- **分组查询注意力（GQA）**：平衡效率与质量

**LLaMA 系列版本**：

| 版本 | 参数量 | 上下文 | 特点 |
|------|--------|--------|------|
| LLaMA 1 | 7B-65B | 2048 | 首个开源千亿模型 |
| LLaMA 2 | 7B-70B | 4096 | 开放权重，ChatFine-tuning |
| LLaMA 3 | 8B-70B | 8192 | 128K 上下文，指令微调 |
| LLaMA 4 | - | - | 多模态，下一代 |

### Claude：Anthropic 的对齐技术

**Claude** 系列以** Constitutional AI**[^12] 和** RLHF** 改进著称：

- **RLHF（Reinforcement Learning from Human Feedback）**：通过人类反馈进行强化学习对齐
- **Constitutional AI**：通过一套"宪法"规则引导模型行为
- **无害性优先**：在保持有用性的同时最大化无害性

### Gemini：Google 的多模态模型

**Gemini**[^5] 是 Google DeepMind 的多模态大模型：

- **原生多模态**：统一处理文本、图像、音频、视频
- **TPU 优化**：针对 Google TPU 进行了深度优化
- **长上下文**：支持高达 100 万 token 的上下文窗口

## 高效 Transformer 技术

### Flash Attention

**Flash Attention**[^13] 由 Tri Dao 等人于 2022 年提出，是加速注意力计算的关键技术。

**核心思想**：通过 **IO-aware** 算法，减少 GPU 内存访问（HBM）与 SRAM 之间的数据传输。

**传统注意力的问题**：

- 需要计算完整的 $N \times N$ 注意力矩阵（$N$ 为序列长度）
- $O(N^2)$ 显存复杂度，无法处理长序列

**Flash Attention 的创新**：

1. **分块计算**（Tiling）：将 $N \times N$ 矩阵划分为小块，逐一处理
2. **在线 softmax**：在累积过程中计算 softmax，无需保存完整矩阵
3. **利用 SRAM**：在高速缓存中完成主要计算

```python
# Flash Attention 核心逻辑（伪代码）
def flash_attention(Q, K, V, block_size=64):
    """
    Q, K, V: [seq_len, d_head]
    关键优化：分块计算 + 在线 softmax
    """
    n = Q.shape[0]
    scale = 1.0 / math.sqrt(Q.shape[1])
    
    # 分块计算 O(N^2/d) 显存
    acc_attn = torch.zeros(n, n)
    acc_normalizer = torch.zeros(n)
    
    for i in range(0, n, block_size):
        Q_block = Q[i:i+block_size]
        
        # 加载到 SRAM
        K_block = K[i:i+block_size]
        V_block = V[i:i+block_size]
        
        # 局部注意力
        block_attn = (Q_block @ K_block.T) * scale
        block_attn = softmax(block_attn, dim=-1)
        
        # 累积到全局
        acc_normalizer[i:i+block_size] += block_attn.sum(dim=-1)
        
    # 最终归一化
    return acc_attn / acc_normalizer.unsqueeze(-1)
```

**性能提升**：

- **显存**：从 $O(N^2)$ 降至 $O(N)$
- **速度**：提升 2-4 倍
- **数值稳定性**：与标准实现结果一致

### Sparse Attention

**Sparse Attention**[^14] 通过选择性计算来降低复杂度：

**局部窗口注意力**（Sliding Window）：

- 只计算每个位置周围 $w$ 个token的注意力
- 复杂度：$O(N \cdot w)$ 而非 $O(N^2)$
- 适合捕捉局部依赖

**全局注意力**（Global Attention）：

- 选择性 token（如 CLS、特殊标记）与所有位置交互
- 保持全局信息流通

**稀疏模式组合**：

```
┌─────────────────────────────┐
│ ★  ★  ★  ★  ★  ★  ★  ★  ★ │  ← 全局注意力（所有位置）
│ ┌─────────────────────────┐ │
│ │ ○  ○  ○  ○  ○  ○  ○  ○ │ │  ← 窗口注意力 w=3
│ │ ○  ○  ○  ○  ○  ○  ○  ○ │ │
│ │ ○  ○  ○  ○  ○  ○  ○  ○ │ │
└─┴─────────────────────────┴─┘
```

### Ring Attention

**Ring Attention**[^15] 用于分布式长序列处理：

**核心思想**：将序列分片，每个设备处理一部分，通过环形通信传递 Key/Value。

```
Device 0        Device 1        Device 2        Device 3
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ Q_0     │    │ Q_1     │    │ Q_2     │    │ Q_3     │  ← Query 分片
│ ↓       │    │ ↓       │    │ ↓       │    │ ↓       │
│ K_0,V_0 │───▶│ K_1,V_1 │───▶│ K_2,V_2 │───▶│ K_3,V_3 │  ← KV 环式传递
└─────────┘    └─────────┘    └─────────┘    └─────────┘
```

**计算流程**：

1. 每个设备持有 $Q_i, K_i, V_i$ 的一个分片
2. Ring 传递 $K, V$ 分片，逐步累积注意力结果
3. 最终每个设备得到完整的输出分片

## 状态空间模型（SSM）

### Mamba：选择性状态空间模型

**Mamba**[^16] 由 Carnegie Mellon University 和 NVIDIA 于 2023 年提出，是一种选择性状态空间模型（Selective State Space Model）。

**核心创新**：将 Transformer 的选择性机制引入 SSM，使得模型能够根据输入内容动态决定保留或忽略历史信息。

**SSM 基础**：HiPPO（High-order Polynomial Projection Operator）[^17] 提出了连续时间的状态空间模型：

$$
\mathbf{h}'(t) = \mathbf{A}\mathbf{h}(t) + \mathbf{B}\mathbf{x}(t)
$$
$$
\mathbf{y}(t) = \mathbf{C}\mathbf{h}(t) + \mathbf{D}\mathbf{x}(t)
$$

离散化后：

$$
\mathbf{h}_t = \bar{\mathbf{A}}\mathbf{h}_{t-1} + \bar{\mathbf{B}}\mathbf{x}_t
$$
$$
\mathbf{y}_t = \mathbf{C}\mathbf{h}_t + \mathbf{D}\mathbf{x}_t
$$

**Mamba 的选择机制**：

- **选择性扫描**（Selective Scan）：输入 $x_t$ 决定是否忽略历史状态 $\mathbf{h}_{t-1}$
- **并行化计算**：通过硬件感知算法实现高效并行

```python
# Mamba 选择性机制示意
class MambaBlock(nn.Module):
    def __init__(self, d_model, d_state=16, d_conv=4, expand=2):
        super().__init__()
        self.d_model = d_model
        self.d_state = d_state
        
        # 选择性投影 - 输入决定参数
        self.x_proj = nn.Linear(d_model, d_state * 2 + 1, bias=False)
        self.dt_proj = nn.Linear(1, d_model, bias=True)
        
        # 状态矩阵
        self.A_log = nn.Parameter(torch.randn(d_model, d_state))
        self.D = nn.Parameter(torch.ones(d_model))
        
    def forward(self, x):
        # x: [batch, seq_len, d_model]
        batch, seq_len, _ = x.shape
        
        # 输入相关选择性参数
        x_dbl = self.x_proj(x)
        dt, B, C = x_dbl.split([1, self.d_state, self.d_state], dim=-1)
        dt = torch.softplus(self.dt_proj(dt))  # 正参数
        
        # 选择性扫描 - 关键创新
        y = self.selective_scan(x, dt, A, B, C)
        
        return y + x  # 残差连接
```

**Mamba vs Transformer**：

| 特性 | Transformer | Mamba |
|------|-------------|-------|
| 复杂度 | $O(N^2)$ | $O(N)$ |
| 推理速度 | 慢（KV 缓存大） | 快（恒定状态） |
| 长依赖 | 显式注意力 | 隐式状态压缩 |
| 并行训练 | 容易 | 需要专用算法 |

### RWKV：无需注意力的 Transformer 替代

**RWKV**[^18]（Receptance Weighted Key Value）由 Peng Bo 提出，是一种结合了 RNN 与 Transformer 优点的架构。

**核心设计**：

- **线性注意力**：将 $O(N^2)$ 转化为 $O(N)$
- **RNN 形式推理**：推理时类似 RNN，无需缓存历史 KV
- **可并行训练**：训练时等价于 Transformer

**RWKV 的时间混合**：

$$
\mathbf{o}_t = \underbrace{\left(1 - \frac{e^{w_t} \cdot \mathbf{r}_t}{\sum_{i=1}^t e^{w_i}}\right)}_{\text{线性衰减}} \odot \mathbf{wkv}_t + \underbrace{\frac{e^{w_t}}{\sum_{i=1}^t e^{w_i}}}_{\text{当前位置权重}} \odot (\mathbf{u} \odot \mathbf{k}_t) \cdot \mathbf{v}_t
$$

其中 $w_t$ 是位置编码的 logits，$\mathbf{r}_t$ 是 receptance 向量。

**RWKV 特点**：

- **推理高效**：恒定时间复杂度，无 KV 缓存
- **长上下文**：理论上可处理无限长度
- **可解释性**：类似 RNN 的隐状态

## 未来趋势

### 混合专家（Mixture of Experts, MoE）

**MoE**[^19] 通过稀疏激活实现大规模模型的高效训练：

**核心思想**：每个输入只激活少数"专家"（Expert）网络，而非全部激活。

**架构**：

$$
\mathbf{y} = \sum_{i=1}^{N} G(\mathbf{x})_i \cdot E_i(\mathbf{x})
$$

其中 $G(\mathbf{x})$ 是门控函数（如 Top-K），$E_i$ 是第 $i$ 个专家网络。

```
┌────────────────────────────────────────┐
│              输入 x                     │
│                ▼                        │
│         ┌──────────┐                   │
│         │  门控 G   │ ──▶ 选择 Top-2   │
│         └──────────┘                   │
│        ▼           ▼                    │
│   ┌────────┐  ┌────────┐                │
│   │ Expert │  │ Expert │  ← 稀疏激活    │
│   │   1    │  │   5    │                │
│   └────────┘  └────────┘                │
│        ▼           ▼                    │
│         输出加权和                       │
└────────────────────────────────────────┘
```

**优势**：

- **参数量大**：可以拥有数千个专家，实际激活的参数量远小于总参数量
- **计算效率高**：每次前向只计算少量专家的输出
- **扩展性好**：可以在保持计算量可控的情况下扩展模型规模

**代表模型**：

| 模型 | 总参数量 | 激活参数量 | 专家数 |
|------|----------|-----------|--------|
| GShard | 600B | 6B | 2048 |
| Switch Transformer | 1.6T | 1.6B | 2048 |
| Mixtral 8x7B | 46.7B | 12.9B | 8 |
| DBRX | 132B | 36B | 16 |

### 动态计算分配

**动态计算**旨在根据输入复杂度动态分配计算资源：

**自适应计算时间**：

- **Early Exit**：简单样本在浅层退出
- **Skip Connection**：动态跳过某些层

```python
# Early Exit 示意
class AdaptiveTransformer(nn.Module):
    def __init__(self, num_layers, d_model, num_exits):
        super().__init__()
        self.layers = nn.ModuleList([
            TransformerLayer(d_model) for _ in range(num_layers)
        ])
        self.exit_classifiers = nn.ModuleList([
            nn.Linear(d_model, 1) for _ in range(num_exits)
        ])
    
    def forward(self, x, threshold=0.9):
        for i, layer in enumerate(self.layers):
            x = layer(x)
            
            # 检查是否满足退出条件
            if i < len(self.exit_classifiers):
                exit_logit = self.exit_classifiers[i](x[:, 0])  # [CLS]
                if torch.sigmoid(exit_logit) > threshold:
                    return x  # Early exit
        
        return x
```

**Adaptive Attention**：

- 根据Query的重要性动态决定Key的长度
- 不重要的Query只与局部Key交互

**稀疏 MoE 与动态计算的结合**：

- 不仅是专家的选择，还包括计算深度的动态调整
- 根据样本难度决定模型使用的计算量

## 总结

Transformer 架构自 2017 年诞生以来，经历了多轮重要演进：

| 阶段 | 核心创新 | 代表工作 |
|------|---------|----------|
| 2017 | 注意力机制、Encoder-Decoder | Attention Is All You Need |
| 2018-2019 | Pre-norm、位置编码进化 | BERT、GPT-2 |
| 2020-2021 | Flash Attention、RoPE | GPT-3、Longformer |
| 2022-2023 | GQA、RMSNorm、LLaMA | LLaMA、Mamba |
| 2024-2025 | MoE、动态计算、长上下文 | Mixtral、Gemini |

**核心演进趋势**：

1. **效率优化**：从 Post-norm 到 Pre-norm + RMSNorm，从 MHA 到 GQA，Flash Attention
2. **位置编码**：从绝对位置到相对位置（RoPE），支持更长上下文
3. **架构多样化**：Encoder-only（BERT）、Decoder-only（GPT）、混合架构
4. **范式扩展**：SSM（Mamba、RWKV）提供线性复杂度的替代方案
5. **规模与效率平衡**：MoE 架构实现稀疏激活

展望未来，Transformer 及其变体将继续演进，可能的方向包括：

- 更高效的长上下文建模
- 动态计算与自适应机制
- 多模态融合
- 硬件协同设计

理解这些演进对于 [[./llm-theory|LLM 理论]] 研究和应用开发都至关重要。

---

[^1]: Vaswani et al. "Attention Is All You Need". *NeurIPS 2017*. [https://arxiv.org/abs/1706.03762](https://arxiv.org/abs/1706.03762)

[^2]: Devlin et al. "BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding". *NAACL 2019*. [https://arxiv.org/abs/1810.04805](https://arxiv.org/abs/1810.04805)

[^3]: Radford et al. "Language Models are Unsupervised Multitask Learners". *OpenAI Technical Report 2019*. [https://d4mucfpksywv.cloudfront.net/better-language-models/language_models_are_unsupervised_multitask_learners.pdf](https://d4mucfpksywv.cloudfront.net/better-language-models/language_models_are_unsupervised_multitask_learners.pdf)

[^4]: Touvron et al. "LLaMA: Open and Efficient Foundation Language Models". *Meta AI 2023*. [https://arxiv.org/abs/2302.13971](https://arxiv.org/abs/2302.13971)

[^5]: Gemini Team. "Gemini: A Family of Highly Capable Multimodal Models". *Google DeepMind 2023*. [https://arxiv.org/abs/2312.11805](https://arxiv.org/abs/2312.11805)

[^6]: Zhang and Sennrich. "Root Mean Square Layer Normalization". *NeurIPS 2019*. [https://arxiv.org/abs/1910.07467](https://arxiv.org/abs/1910.07467)

[^7]: Ainslie et al. "GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints". *Google Research 2023*. [https://arxiv.org/abs/2305.13245](https://arxiv.org/abs/2305.13245)

[^8]: Su et al. "RoFormer: Enhanced Transformer with Rotary Position Embedding". *arXiv 2021*. [https://arxiv.org/abs/2104.09864](https://arxiv.org/abs/2104.09864)

[^9]: Chen et al. "Extending Context is Hard, But Not Impossible". *arXiv 2023*. [https://arxiv.org/abs/2309.17711](https://arxiv.org/abs/2309.17711)

[^10]: Peng et al. "YaRN: Efficient Context Window Extension of Large Language Models". *arXiv 2023*. [https://arxiv.org/abs/2309.00071](https://arxiv.org/abs/2309.00071)

[^11]: Yao et al. "LongRoPE: Extending LLM Context Window Beyond 2 Million Tokens". *arXiv 2024*. [https://arxiv.org/abs/2402.13753](https://arxiv.org/abs/2402.13753)

[^12]: Bai et al. "Constitutional AI: Harmlessness from AI Feedback". *Anthropic 2022*. [https://arxiv.org/abs/2212.08073](https://arxiv.org/abs/2212.08073)

[^13]: Dao et al. "FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness". *NeurIPS 2022*. [https://arxiv.org/abs/2205.14135](https://arxiv.org/abs/2205.14135)

[^14]: Beltagy et al. "Longformer: The Long-Document Transformer". *arXiv 2020*. [https://arxiv.org/abs/2004.05150](https://arxiv.org/abs/2004.05150)

[^15]: Li et al. "Ring Attention with Blockwise Transformers for Near-Infinite Context". *ICLR 2024*. [https://arxiv.org/abs/2310.01889](https://arxiv.org/abs/2310.01889)

[^16]: Gu and Dao. "Mamba: Linear-Time Sequence Modeling with Selective State Spaces". *arXiv 2023*. [https://arxiv.org/abs/2312.00752](https://arxiv.org/abs/2312.00752)

[^17]: Gu et al. "HiPPO: Recurrent Memory with Optimal Polynomial Projections". *NeurIPS 2020*. [https://arxiv.org/abs/2008.07669](https://arxiv.org/abs/2008.07669)

[^18]: Peng et al. "RWKV: Reinventing RNNs for the Transformer Era". *EMNLP 2023*. [https://arxiv.org/abs/2305.13048](https://arxiv.org/abs/2305.13048)

[^19]: Shazeer et al. "Outrageously Large Neural Networks: The Sparsely-Gated Mixture-of-Experts Layer". *ICLR 2017*. [https://arxiv.org/abs/1701.06538](https://arxiv.org/abs/1701.06538)
