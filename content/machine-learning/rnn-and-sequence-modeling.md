---
title: 循环神经网络与序列建模
date: 2026-04-13
description: RNN、LSTM、GRU的原理与序列建模方法
tags:
  - machine-learning
  - deep-learning
  - rnn
  - nlp
draft: false
permalink:
---

# 循环神经网络与序列建模

循环神经网络（RNN）专为处理序列数据设计，通过隐藏状态传递历史信息，是自然语言处理和时间序列分析的基础。

## RNN基本原理

### 网络结构

RNN通过时间步展开，将序列信息逐步处理：

```python
import torch
import torch.nn as nn

class SimpleRNN(nn.Module):
    def __init__(self, input_size, hidden_size, output_size):
        super().__init__()
        self.hidden_size = hidden_size
        self.rnn = nn.RNN(input_size, hidden_size, batch_first=True)
        self.fc = nn.Linear(hidden_size, output_size)
    
    def forward(self, x):
        # x: (batch, seq_len, input_size)
        output, hidden = self.rnn(x)
        # output: (batch, seq_len, hidden_size)
        # hidden: (1, batch, hidden_size)
        return self.fc(output)
```

### 梯度消失与爆炸

RNN在反向传播时沿时间步展开，链式求导导致：

- **梯度消失**：长依赖问题，早期信息难以影响后期输出
- **梯度爆炸**：参数更新过大，训练不稳定

```python
# 梯度裁剪防止梯度爆炸
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
```

## LSTM（长短期记忆网络）

LSTM通过门控机制选择性保留信息，有效缓解梯度消失问题。

### 核心组件

1. **遗忘门（Forget Gate）**：决定丢弃哪些信息
2. **输入门（Input Gate）**：决定更新哪些信息
3. **输出门（Output Gate）**：决定输出什么信息

```python
class LSTMCell(nn.Module):
    def __init__(self, input_size, hidden_size):
        super().__init__()
        self.hidden_size = hidden_size
        
        # 四个门控单元
        self.Wf = nn.Linear(input_size + hidden_size, hidden_size)  # 遗忘门
        self.Wi = nn.Linear(input_size + hidden_size, hidden_size)  # 输入门
        self.Wc = nn.Linear(input_size + hidden_size, hidden_size)  # 候选记忆
        self.Wo = nn.Linear(input_size + hidden_size, hidden_size)  # 输出门
    
    def forward(self, x, state):
        h, c = state  # 隐藏状态和细胞状态
        
        # 遗忘门
        f = torch.sigmoid(self.Wf(torch.cat([x, h], dim=-1)))
        # 输入门
        i = torch.sigmoid(self.Wi(torch.cat([x, h], dim=-1)))
        # 候选记忆
        c_tilde = torch.tanh(self.Wc(torch.cat([x, h], dim=-1)))
        # 输出门
        o = torch.sigmoid(self.Wo(torch.cat([x, h], dim=-1)))
        
        # 更新细胞状态
        c_new = f * c + i * c_tilde
        # 更新隐藏状态
        h_new = o * torch.tanh(c_new)
        
        return h_new, c_new
```

### PyTorch实现

```python
class LSTMClassifier(nn.Module):
    def __init__(self, vocab_size, embed_dim, hidden_size, num_layers, num_classes):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.lstm = nn.LSTM(embed_dim, hidden_size, num_layers, 
                           batch_first=True, bidirectional=True)
        self.fc = nn.Linear(hidden_size * 2, num_classes)  # 双向LSTM
    
    def forward(self, x):
        # x: (batch, seq_len)
        embedded = self.embedding(x)  # (batch, seq_len, embed_dim)
        output, (h_n, c_n) = self.lstm(embedded)
        # output: (batch, seq_len, hidden_size * 2)
        # 取最后一个时间步的隐藏状态
        hidden = torch.cat([h_n[-2], h_n[-1]], dim=1)  # 双向拼接
        return self.fc(hidden)
```

## GRU（门控循环单元）

GRU是LSTM的简化版本，只有两个门控（更新门和重置门），参数量更少：

```python
class GRUCell(nn.Module):
    def __init__(self, input_size, hidden_size):
        super().__init__()
        self.Wz = nn.Linear(input_size + hidden_size, hidden_size)  # 更新门
        self.Wr = nn.Linear(input_size + hidden_size, hidden_size)  # 重置门
        self.Wh = nn.Linear(input_size + hidden_size, hidden_size)  # 候选隐藏状态
    
    def forward(self, x, h):
        z = torch.sigmoid(self.Wz(torch.cat([x, h], dim=-1)))  # 更新门
        r = torch.sigmoid(self.Wr(torch.cat([x, h], dim=-1)))  # 重置门
        h_tilde = torch.tanh(self.Wh(torch.cat([x, r * h], dim=-1)))
        h_new = (1 - z) * h + z * h_tilde
        return h_new
```

## 序列建模应用

### 文本分类

```python
# 使用双向LSTM进行情感分类
class TextClassifier(nn.Module):
    def __init__(self, vocab_size, embed_dim, hidden_size, num_classes):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim, padding_idx=0)
        self.lstm = nn.LSTM(embed_dim, hidden_size, num_layers=2,
                           batch_first=True, bidirectional=True, dropout=0.3)
        self.attention = AttentionLayer(hidden_size * 2)
        self.classifier = nn.Linear(hidden_size * 2, num_classes)
    
    def forward(self, x):
        mask = (x != 0)  # padding mask
        embedded = self.embedding(x)
        output, _ = self.lstm(embedded)
        attended, weights = self.attention(output, mask)
        return self.classifier(attended)

class AttentionLayer(nn.Module):
    def __init__(self, hidden_size):
        super().__init__()
        self.attention = nn.Linear(hidden_size, 1)
    
    def forward(self, output, mask):
        # output: (batch, seq_len, hidden_size)
        scores = self.attention(output).squeeze(-1)  # (batch, seq_len)
        scores = scores.masked_fill(mask == 0, -1e9)
        weights = torch.softmax(scores, dim=-1)
        weighted = torch.sum(output * weights.unsqueeze(-1), dim=1)
        return weighted, weights
```

### 序列到序列（Seq2Seq）

```python
class Encoder(nn.Module):
    def __init__(self, input_dim, embed_dim, hidden_dim):
        super().__init__()
        self.embedding = nn.Embedding(input_dim, embed_dim)
        self.lstm = nn.LSTM(embed_dim, hidden_dim, batch_first=True)
    
    def forward(self, x):
        embedded = self.embedding(x)
        outputs, (hidden, cell) = self.lstm(embedded)
        return hidden, cell

class Decoder(nn.Module):
    def __init__(self, output_dim, embed_dim, hidden_dim):
        super().__init__()
        self.embedding = nn.Embedding(output_dim, embed_dim)
        self.lstm = nn.LSTM(embed_dim, hidden_dim, batch_first=True)
        self.fc = nn.Linear(hidden_dim, output_dim)
    
    def forward(self, x, hidden, cell):
        x = x.unsqueeze(1)  # 添加序列维度
        embedded = self.embedding(x)
        output, (hidden, cell) = self.lstm(embedded, (hidden, cell))
        prediction = self.fc(output.squeeze(1))
        return prediction, hidden, cell
```

## RNN变体对比

| 特性 | 简单RNN | LSTM | GRU |
|------|--------|------|-----|
| 门控数量 | 0 | 3 | 2 |
| 参数量 | 少 | 多 | 中 |
| 长依赖 | 差 | 好 | 好 |
| 训练速度 | 快 | 慢 | 中 |

## 实践技巧

1. **使用双向LSTM**：同时利用前后文信息
2. **层归一化**：`nn.LayerNorm` 稳定训练
3. **注意力机制**：让模型自动关注关键信息
4. **梯度裁剪**：防止梯度爆炸

## 参考

[^1]: Hochreiter & Schmidhuber, Long Short-Term Memory
[^2]: Cho et al., Learning Phrase Representations using RNN Encoder-Decoder

