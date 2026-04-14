---
title: Transformer与注意力机制
date: 2026-04-13
description: 自注意力机制、Transformer架构及大语言模型基础
tags:
  - machine-learning
  - deep-learning
  - transformer
  - attention
draft: false
permalink:
---

# Transformer与注意力机制

Transformer通过自注意力机制（Self-Attention）实现并行序列建模，彻底改变了自然语言处理领域的格局。

## 注意力机制原理

### 缩放点积注意力

$$
\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

其中 $Q$（Query）、$K$（Key）、$V$（Value）分别表示查询、键、值向量。

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

def scaled_dot_product_attention(Q, K, V, mask=None):
    """
    Q: (batch, num_heads, seq_len, d_k)
    K: (batch, num_heads, seq_len, d_k)
    V: (batch, num_heads, seq_len, d_v)
    """
    d_k = Q.size(-1)
    # 计算注意力分数
    scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(d_k)
    
    # 应用掩码（用于padding或解码时的未来信息）
    if mask is not None:
        scores = scores.masked_fill(mask == 0, -1e9)
    
    # 归一化得到注意力权重
    attn_weights = F.softmax(scores, dim=-1)
    # 加权求和
    output = torch.matmul(attn_weights, V)
    
    return output, attn_weights
```

### 多头注意力

将输入分割成多个头并行计算注意力，捕捉不同子空间的特征：

```python
class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, num_heads):
        super().__init__()
        assert d_model % num_heads == 0
        
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
    
    def forward(self, query, key, value, mask=None):
        batch_size = query.size(0)
        
        # 线性变换后分头
        Q = self.W_q(query).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(key).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(value).view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        
        # 缩放点积注意力
        x, attn_weights = scaled_dot_product_attention(Q, K, V, mask)
        
        # 合并多头
        x = x.transpose(1, 2).contiguous().view(batch_size, -1, self.d_model)
        
        return self.W_o(x)
```

## Transformer架构

### 编码器（Encoder）

每个编码器层包含两个子层：
1. **多头自注意力**
2. **前馈神经网络**

每个子层都有残差连接和层归一化：

```python
class EncoderLayer(nn.Module):
    def __init__(self, d_model, num_heads, d_ff, dropout=0.1):
        super().__init__()
        self.self_attn = MultiHeadAttention(d_model, num_heads)
        self.ffn = nn.Sequential(
            nn.Linear(d_model, d_ff),
            nn.ReLU(),
            nn.Linear(d_ff, d_model)
        )
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, x, mask=None):
        # 自注意力子层
        attn_output = self.self_attn(x, x, x, mask)
        x = self.norm1(x + self.dropout(attn_output))
        
        # 前馈网络子层
        ffn_output = self.ffn(x)
        x = self.norm2(x + self.dropout(ffn_output))
        
        return x

class Encoder(nn.Module):
    def __init__(self, num_layers, d_model, num_heads, d_ff, dropout=0.1):
        super().__init__()
        self.layers = nn.ModuleList([
            EncoderLayer(d_model, num_heads, d_ff, dropout)
            for _ in range(num_layers)
        ])
    
    def forward(self, x, mask=None):
        for layer in self.layers:
            x = layer(x, mask)
        return x
```

### 解码器（Decoder）

解码器包含三个子层：
1. **掩码多头自注意力**（防止看到未来信息）
2. **编码器-解码器注意力**（关注源序列）
3. **前馈神经网络**

```python
class DecoderLayer(nn.Module):
    def __init__(self, d_model, num_heads, d_ff, dropout=0.1):
        super().__init__()
        self.self_attn = MultiHeadAttention(d_model, num_heads)
        self.cross_attn = MultiHeadAttention(d_model, num_heads)
        self.ffn = nn.Sequential(
            nn.Linear(d_model, d_ff),
            nn.ReLU(),
            nn.Linear(d_ff, d_model)
        )
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
        self.norm3 = nn.LayerNorm(d_model)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, x, encoder_output, src_mask=None, tgt_mask=None):
        # 掩码自注意力
        attn = self.self_attn(x, x, x, tgt_mask)
        x = self.norm1(x + self.dropout(attn))
        
        # 编码器-解码器注意力
        attn = self.cross_attn(x, encoder_output, encoder_output, src_mask)
        x = self.norm2(x + self.dropout(attn))
        
        # 前馈网络
        ffn = self.ffn(x)
        x = self.norm3(x + self.dropout(ffn))
        
        return x
```

### 完整Transformer

```python
class Transformer(nn.Module):
    def __init__(self, src_vocab_size, tgt_vocab_size, d_model=512, 
                 num_heads=8, num_layers=6, d_ff=2048, dropout=0.1):
        super().__init__()
        self.encoder = Encoder(num_layers, d_model, num_heads, d_ff, dropout)
        self.decoder = Decoder(num_layers, d_model, num_heads, d_ff, dropout)
        self.src_embed = nn.Embedding(src_vocab_size, d_model)
        self.tgt_embed = nn.Embedding(tgt_vocab_size, d_model)
        self.pos_encoding = PositionalEncoding(d_model, dropout)
        self.fc = nn.Linear(d_model, tgt_vocab_size)
    
    def forward(self, src, tgt, src_mask=None, tgt_mask=None):
        src_emb = self.pos_encoding(self.src_embed(src))
        tgt_emb = self.pos_encoding(self.tgt_embed(tgt))
        
        encoder_output = self.encoder(src_emb, src_mask)
        decoder_output = self.decoder(tgt_emb, encoder_output, src_mask, tgt_mask)
        
        return self.fc(decoder_output)

class PositionalEncoding(nn.Module):
    def __init__(self, d_model, dropout=0.1, max_len=5000):
        super().__init__()
        self.dropout = nn.Dropout(p=dropout)
        
        pe = torch.zeros(max_len, d_model)
        position = torch.arange(0, max_len).unsqueeze(1).float()
        div_term = torch.exp(torch.arange(0, d_model, 2).float() * 
                           (-math.log(10000.0) / d_model))
        pe[:, 0::2] = torch.sin(position * div_term)
        pe[:, 1::2] = torch.cos(position * div_term)
        pe = pe.unsqueeze(0)
        self.register_buffer('pe', pe)
    
    def forward(self, x):
        x = x + self.pe[:, :x.size(1)]
        return self.dropout(x)
```

## Transformer变体

| 模型 | 特点 | 应用场景 |
|------|------|----------|
| BERT | 双向编码器，只做编码器 | 文本分类、命名实体识别 |
| GPT | 单向解码器，生成式 | 文本生成、对话 |
| T5 | Encoder-Decoder架构 | 文本生成、翻译、摘要 |
| ViT | 图像分块 + Transformer | 图像分类 |

## 现代大语言模型（LLM）

### ChatGPT/GPT-4架构

基于GPT架构，特点：
- 超大参数量（175B+）
- 人类反馈强化学习（RLHF）
- 指令微调（Instruction Tuning）

### LLaMA架构

Meta开源的LLM基础模型，采用：
- RMSNorm归一化
- SwiGLU激活函数
- Rotary Position Embedding（RoPE）

## 参考

[^1]: Vaswani et al., Attention Is All You Need
[^2]: Devlin et al., BERT: Pre-training of Deep Bidirectional Transformers

