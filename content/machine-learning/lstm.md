---
title: LSTM长短期记忆网络
date: 2026-04-20
description: LSTM门控机制、梯度分析、变体与PyTorch实现
tags:
  - deep-learning
  - rnn
  - lstm
  - sequence-modeling
draft: false
permalink:
---

# LSTM长短期记忆网络

LSTM（Long Short-Term Memory）由Hochreiter和Schmidhuber于1997年提出[^1]，是处理长序列最重要的循环神经网络变体之一。LSTM通过引入门控机制，解决了标准RNN的梯度消失和爆炸问题，能够有效地学习长距离依赖关系。

## 问题背景：标准RNN的梯度困境

### 梯度消失

对于标准RNN，假设激活函数为恒等函数（简化分析），则：

$$
h_t = h_{t-1} + x_t
$$

反向传播时，梯度从时刻 $t$ 传递到时刻 $s$：

$$
\frac{\partial h_t}{\partial h_s} = \prod_{i=s+1}^{t} \frac{\partial h_i}{\partial h_{i-1}} = \prod_{i=s+1}^{t} I = I
$$

但对于一般的RNN（$h_t = \tanh(W_h h_{t-1} + W_x x_t + b)$）：

$$
\frac{\partial h_t}{\partial h_{s}} = \prod_{i=s+1}^{t} \text{diag}(\sigma'(z_i)) W_h
$$

当 $|W_h| < 1$ 且激活导数小于1时，梯度随时间步指数衰减，导致早期时刻的梯度接近于零，模型无法学习长距离依赖。

### 梯度爆炸

相反，当 $|W_h| > 1$ 时，梯度会指数增长，导致训练不稳定。

## LSTM核心设计

### 常数误差carousel

LSTM的核心创新是**常数误差carousel**：通过让信息在细胞状态（cell state）中恒定传递，保持梯度流动不受衰减影响。

```
┌─────────────────────────────────────────────────────────────────┐
│                        LSTM结构概览                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  x_t ──┬───────┐                                               │
│        │       ↓                                               │
│        │   ┌─────────┐                                          │
│        │   │  遗忘门  │  f_t = σ(W_f·[h_{t-1}, x_t] + b_f)      │
│        │   └────┬────┘                                          │
│        │        ↓                                                │
│        │   ┌─────────┐     ┌─────────┐                          │
│        │   │  输入门  │────▶│  细胞   │     h_t = o_t ⊙ tanh(c_t│
│        │   │  i_t     │     │  状态   │                          │
│        │   └────┬────┘     │  c_t    │                          │
│        │        ↓          │         │                          │
│        │   ┌─────────┐     │ c_t =   │                          │
│        │   │候选值   │     │ f_t⊙c_{t-│
│        │   │ g_t     │     │ + i_t⊙g_t│                          │
│        │   └─────────┘     └─────────┘                          │
│        │        ↑        ↑                                      │
│        └────────│────────│───────────────────────────────────── │
│                 │        │                                      │
│        h_{t-1} ─┴────────┴── x_t                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 数学公式

LSTM包含四个门控，核心公式如下：

#### 遗忘门（Forget Gate）

决定从细胞状态中丢弃什么信息：

$$
f_t = \sigma(W_f \cdot [h_{t-1}, x_t] + b_f)
$$

其中 $\sigma$ 是sigmoid函数，$[h_{t-1}, x_t]$ 表示向量拼接。

#### 输入门（Input Gate）

决定保留多少新信息：

$$
i_t = \sigma(W_i \cdot [h_{t-1}, x_t] + b_i)
$$

#### 候选细胞状态（Candidate Cell State）

生成新的候选值：

$$
\tilde{c}_t = \tanh(W_C \cdot [h_{t-1}, x_t] + b_C)
$$

#### 细胞状态更新

$$
c_t = f_t \odot c_{t-1} + i_t \odot \tilde{c}_t
$$

这里 $\odot$ 表示逐元素乘法（Hadamard积）。

**核心洞察**：遗忘门和输入门的设置决定了每个时刻保留多少旧信息、添加多少新信息。

#### 输出门（Output Gate）

决定输出什么信息：

$$
o_t = \sigma(W_o \cdot [h_{t-1}, x_t] + b_o)
$$

#### 隐藏状态

$$
h_t = o_t \odot \tanh(c_t)
$$

## PyTorch实现

### 基础LSTM实现

```python
import torch
import torch.nn as nn

class LSTMCell(nn.Module):
    """单步LSTM单元"""
    def __init__(self, input_size, hidden_size):
        super().__init__()
        self.input_size = input_size
        self.hidden_size = hidden_size
        
        # 四个门的权重
        self.W_i = nn.Linear(input_size + hidden_size, hidden_size)
        self.W_f = nn.Linear(input_size + hidden_size, hidden_size)
        self.W_c = nn.Linear(input_size + hidden_size, hidden_size)
        self.W_o = nn.Linear(input_size + hidden_size, hidden_size)
    
    def forward(self, x, state):
        """
        Args:
            x: (batch, input_size) - 当前输入
            state: (h, c) - 隐藏状态和细胞状态
        Returns:
            h, c - 更新后的隐藏状态和细胞状态
        """
        h, c = state
        
        # 拼接输入和隐藏状态
        combined = torch.cat([x, h], dim=1)
        
        # 计算四个门
        i = torch.sigmoid(self.W_i(combined))  # 输入门
        f = torch.sigmoid(self.W_f(combined))    # 遗忘门
        g = torch.tanh(self.W_c(combined))      # 候选值
        o = torch.sigmoid(self.W_o(combined))    # 输出门
        
        # 更新细胞状态
        c_next = f * c + i * g
        
        # 计算隐藏状态
        h_next = o * torch.tanh(c_next)
        
        return h_next, c_next
```

### 多层LSTM实现

```python
class MultiLayerLSTM(nn.Module):
    """多层LSTM"""
    def __init__(self, input_size, hidden_size, num_layers, dropout=0.0):
        super().__init__()
        self.num_layers = num_layers
        self.hidden_size = hidden_size
        
        self.layers = nn.ModuleList([
            LSTMCell(
                input_size if i == 0 else hidden_size,
                hidden_size
            )
            for i in range(num_layers)
        ])
        
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, x, states=None):
        """
        Args:
            x: (seq_len, batch, input_size) - 输入序列
            states: list of (h, c) tuples, or None
        Returns:
            outputs: (seq_len, batch, hidden_size)
            final_states: list of (h, c) tuples
        """
        seq_len, batch, _ = x.shape
        
        # 初始化状态
        if states is None:
            device = x.device
            h = [torch.zeros(batch, self.hidden_size, device=device) 
                 for _ in range(self.num_layers)]
            c = [torch.zeros(batch, self.hidden_size, device=device) 
                 for _ in range(self.num_layers)]
            states = list(zip(h, c))
        
        outputs = []
        current_input = x
        
        for layer_idx, layer in enumerate(self.layers):
            layer_outputs = []
            h, c = states[layer_idx]
            
            for t in range(seq_len):
                h, c = layer(current_input[t], (h, c))
                layer_outputs.append(h)
            
            current_input = torch.stack(layer_outputs, dim=0)
            
            # 层间dropout（最后一层不加）
            if layer_idx < self.num_layers - 1:
                current_input = self.dropout(current_input)
            
            states[layer_idx] = (h, c)
        
        return current_input, states
```

### 使用PyTorch内置LSTM

```python
class PyTorchLSTM(nn.Module):
    """使用PyTorch内置LSTM模块"""
    def __init__(self, input_size, hidden_size, num_layers, dropout=0.0, batch_first=False):
        super().__init__()
        self.lstm = nn.LSTM(
            input_size=input_size,
            hidden_size=hidden_size,
            num_layers=num_layers,
            dropout=dropout if num_layers > 1 else 0,
            batch_first=batch_first  # 输入格式: (batch, seq, feature)
        )
        
        # 初始化遗忘门偏置为1（LSTM默认值）
        # 有助于训练初期保留更多信息
        for name, param in self.lstm.named_parameters():
            if 'bias' in name and 'lstm' in name:
                # 设置遗忘门偏置为1
                n = param.size(0)
                param.data[n//4:n//2].fill_(1)
    
    def forward(self, x, state=None):
        """
        Args:
            x: (seq_len, batch, input_size) 或 (batch, seq_len, input_size)
            state: tuple (h, c) 或 None
        Returns:
            output: (seq_len, batch, hidden_size) 或 (batch, seq_len, hidden_size)
            (h, c): final hidden and cell states
        """
        return self.lstm(x, state)
```

## 梯度流动分析

### 为什么LSTM能避免梯度消失

考虑LSTM的细胞状态更新：

$$
c_t = f_t \odot c_{t-1} + i_t \odot \tilde{c}_t
$$

反向传播计算 $\frac{\partial c_t}{\partial c_{t-1}}$：

$$
\frac{\partial c_t}{\partial c_{t-1}} = \text{diag}(f_t)
$$

当遗忘门 $f_t \approx 1$ 时，梯度**几乎无损**地传递！

### 形式化证明

设 $t$ 时刻的损失为 $\mathcal{L}_t$，定义：

$$
\frac{\partial \mathcal{L}_t}{\partial c_{t-1}} = \frac{\partial \mathcal{L}_t}{\partial c_t} \cdot \frac{\partial c_t}{\partial c_{t-1}}
= \frac{\partial \mathcal{L}_t}{\partial c_t} \cdot \text{diag}(f_t)
$$

递推得到：

$$
\frac{\partial \mathcal{L}_t}{\partial c_s} = \frac{\partial \mathcal{L}_t}{\partial c_t} \prod_{i=s+1}^{t} \text{diag}(f_i)
$$

由于 $f_i \in (0, 1)$，我们可以控制遗忘门使梯度衰减速率：
- 如果 $f_i = 1$，梯度完全保留
- 如果 $f_i = 0.99$，每步衰减1%，100步后仍有 $0.99^{100} \approx 0.37$

### 对比标准RNN

| 方面 | 标准RNN | LSTM |
|------|---------|------|
| 状态传递 | $h_t = \tanh(W h_{t-1} + ...)$ | $c_t = f_t \odot c_{t-1} + ...$ |
| 梯度路径 | 经过 $\tanh$ 和 $W$ 的乘积 | 绕过非线性，仅通过乘法 |
| 梯度消失 | 指数衰减 | 可控衰减（通过遗忘门） |
| 长期依赖 | 难以学习 | 可以学习 |

## LSTM变体

### GRU（Gated Recurrent Unit）

GRU由Cho等人于2014年提出，是LSTM的简化版本，只有两个门：[^2]

$$
\begin{aligned}
z_t &= \sigma(W_z \cdot [h_{t-1}, x_t]) \quad \text{（更新门）}\\
r_t &= \sigma(W_r \cdot [h_{t-1}, x_t]) \quad \text{（重置门）}\\
\tilde{h}_t &= \tanh(W \cdot [r_t \odot h_{t-1}, x_t]) \\
h_t &= (1 - z_t) \odot h_{t-1} + z_t \odot \tilde{h}_t
\end{aligned}
$$

```python
class GRUCell(nn.Module):
    """GRU单元"""
    def __init__(self, input_size, hidden_size):
        super().__init__()
        self.W_z = nn.Linear(input_size + hidden_size, hidden_size)
        self.W_r = nn.Linear(input_size + hidden_size, hidden_size)
        self.W = nn.Linear(input_size + hidden_size, hidden_size)
    
    def forward(self, x, h):
        combined = torch.cat([x, h], dim=1)
        
        z = torch.sigmoid(self.W_z(combined))  # 更新门
        r = torch.sigmoid(self.W_r(combined)) # 重置门
        
        h_tilde = torch.tanh(self.W(torch.cat([r * h, x], dim=1)))
        h_next = (1 - z) * h + z * h_tilde
        
        return h_next
```

**LSTM vs GRU**：

| 特性 | LSTM | GRU |
|------|------|-----|
| 门数量 | 4个（遗忘、输入、输出 + 候选） | 2个（更新、重置） |
| 记忆单元 | 有单独的细胞状态 $c_t$ | 无，直接用隐藏状态 |
| 参数数量 | 更多 | 更少 |
| 训练难度 | 较难 | 较易 |
| 表达能力 | 更强 | 略弱但通常足够 |

### Peephole LSTM

Peephole连接允许门直接"看到"细胞状态：

$$
\begin{aligned}
f_t &= \sigma(W_f \cdot [c_{t-1}, h_{t-1}, x_t]) \\
i_t &= \sigma(W_i \cdot [c_{t-1}, h_{t-1}, x_t]) \\
o_t &= \sigma(W_o \cdot [c_t, h_{t-1}, x_t])
\end{aligned}
$$

```python
class PeepholeLSTM(nn.Module):
    """带Peephole连接的LSTM"""
    def __init__(self, input_size, hidden_size):
        super().__init__()
        self.hidden_size = hidden_size
        
        self.W_i = nn.Linear(input_size + hidden_size + hidden_size, hidden_size)
        self.W_f = nn.Linear(input_size + hidden_size + hidden_size, hidden_size)
        self.W_c = nn.Linear(input_size + hidden_size, hidden_size)
        self.W_o = nn.Linear(input_size + hidden_size + hidden_size, hidden_size)
    
    def forward(self, x, state):
        h, c = state
        
        # 注意：输入门和遗忘门可以看到c_{t-1}
        i = torch.sigmoid(self.W_i(torch.cat([c, h, x], dim=1)))
        f = torch.sigmoid(self.W_f(torch.cat([c, h, x], dim=1)))
        g = torch.tanh(self.W_c(torch.cat([h, x], dim=1)))
        
        c_next = f * c + i * g
        
        # 输出门可以看到c_t
        o = torch.sigmoid(self.W_o(torch.cat([c_next, h, x], dim=1)))
        
        h_next = o * torch.tanh(c_next)
        
        return h_next, c_next
```

### 维度说明

LSTM的参数数量计算：

对于输入维度 $d$ 和隐藏维度 $h$：
- 每个门的权重：$(d + h) \times h$
- 每个门的偏置：$h$
- 四个门总计：$4 \times (d + h + 1) \times h$

```python
def lstm_params(input_size, hidden_size):
    """计算LSTM参数数量"""
    # 四个门: 输入门、遗忘门、输出门、候选值
    weights_per_gate = (input_size + hidden_size) * hidden_size
    bias_per_gate = hidden_size
    total_per_gate = weights_per_gate + bias_per_gate
    
    return 4 * total_per_gate

# 示例：input_size=100, hidden_size=256
print(f"LSTM参数数量: {lstm_params(100, 256):,}")  # 输出: 365,568
```

## 实践技巧

### 参数初始化

```python
def init_lstm(model):
    """LSTM参数初始化"""
    for name, param in model.named_parameters():
        if 'weight_ih' in name:
            # 输入权重：使用Xavier均匀分布
            nn.init.xavier_uniform_(param)
        elif 'weight_hh' in name:
            # 隐藏权重：使用正交初始化
            nn.init.orthogonal_(param)
        elif 'bias' in name:
            # 偏置初始化：遗忘门偏置设为1
            n = param.size(0)
            param.data[n//4:n//2].fill_(1)
```

### Dropout正则化

```python
class LSTMDropout(nn.Module):
    """带Dropout的LSTM（Variational Dropout）"""
    def __init__(self, input_size, hidden_size, num_layers, dropout=0.5):
        super().__init__()
        self.lstm = nn.LSTM(input_size, hidden_size, num_layers, 
                           dropout=dropout if num_layers > 1 else 0,
                           batch_first=True)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, x, state=None):
        output, state = self.lstm(x, state)
        output = self.dropout(output)
        return output, state
```

### 双向LSTM

```python
class BidirectionalLSTM(nn.Module):
    """双向LSTM"""
    def __init__(self, input_size, hidden_size, num_layers):
        super().__init__()
        self.lstm = nn.LSTM(
            input_size=input_size,
            hidden_size=hidden_size,
            num_layers=num_layers,
            bidirectional=True,
            batch_first=True
        )
        # 输出维度是hidden_size的2倍（双向）
    
    def forward(self, x):
        output, (h, c) = self.lstm(x)
        # output: (batch, seq, hidden*2)
        # h: (num_layers*2, batch, hidden)
        return output, (h, c)
```

## 与Transformer的关系

现代NLP中，Transformer逐渐取代了LSTM成为主流架构，但LSTM仍有其价值：

| 特性 | LSTM | Transformer |
|------|------|-------------|
| 序列长度 | O(n) | O(n²) |
| 位置编码 | 隐式 | 需要显式编码 |
| 并行性 | 低（按时间展开） | 高（自注意力并行） |
| 长距离依赖 | 一般（通过门控） | 强（通过注意力） |
| 推理速度 | 快 | 较慢（注意力的二次方） |
| 显存占用 | O(n·h) | O(n²) |

**LSTM的优势场景**：
- 资源受限环境
- 实时推理需求
- 超长序列（>10k tokens）
- 增量学习/在线学习

## 参考

[^1]: Hochreiter & Schmidhuber, "Long Short-Term Memory", Neural Computation 1997
[^2]: Cho et al., "Learning Phrase Representations using RNN Encoder-Decoder for Statistical Machine Translation", EMNLP 2014
[^3]: Gers et al., "Learning Precise Timing with LSTM Recurrent Networks", JMLR 2002
[^4]: Jozefowicz et al., "An Empirical Exploration of Recurrent Network Architectures", ICML 2015
