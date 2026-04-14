---
title: 深度学习基础
date: 2026-04-07
description: 深度学习与神经网络核心概念：感知机、激活函数、反向传播、常见网络结构
tags:
  - deep-learning
  - neural-network
  - backpropagation
  - cnn
  - rnn
draft: false
permalink:
---

## 概述

深度学习（Deep Learning）是机器学习的子领域，基于具有多个隐藏层的人工神经网络（ANN）。深度学习通过层次化表示学习自动从原始数据中提取特征，在计算机视觉、自然语言处理、语音识别等领域取得突破性进展。[^1]

---

## 神经网络基础

### 神经元模型

神经元是神经网络的基本单元：

$$
y = \sigma\left(\sum_{i=1}^{n} w_i x_i + b\right)
$$

其中 $w_i$ 是权重，$b$ 是偏置，$\sigma$ 是激活函数。

### 激活函数

| 激活函数 | 公式 | 输出范围 | 特点 |
|----------|------|----------|------|
| **Sigmoid** | $\frac{1}{1+e^{-x}}$ | $(0, 1)$ | 易梯度消失 |
| **Tanh** | $\frac{e^x - e^{-x}}{e^x + e^{-x}}$ | $(-1, 1)$ | 零中心化 |
| **ReLU** | $\max(0, x)$ | $[0, +\infty)$ | 计算高效，主导 |
| **Leaky ReLU** | $\max(0.01x, x)$ | $(-\infty, +\infty)$ | 避免死神经元 |
| **Softmax** | $\frac{e^{x_i}}{\sum_j e^{x_j}}$ | $(0, 1)^n$ | 多分类 |

```python
import numpy as np

def sigmoid(x):
    return 1 / (1 + np.exp(-x))

def relu(x):
    return np.maximum(0, x)

def leaky_relu(x, alpha=0.01):
    return np.where(x > 0, x, alpha * x)

def softmax(x):
    exp_x = np.exp(x - np.max(x, axis=-1, keepdims=True))
    return exp_x / np.sum(exp_x, axis=-1, keepdims=True)
```

---

## 多层感知机（MLP）

MLP由输入层、若干隐藏层、输出层组成：

```
输入层 → [隐藏层 → ReLU] × N → 输出层 → Softmax
```

### 前向传播

```python
class MLP:
    def __init__(self, layer_sizes):
        """layer_sizes: [input_dim, hidden1, hidden2, ..., output_dim]"""
        self.weights = []
        self.biases = []
        
        for i in range(len(layer_sizes) - 1):
            # Xavier 初始化
            w = np.random.randn(layer_sizes[i], layer_sizes[i+1]) * np.sqrt(2.0 / layer_sizes[i])
            b = np.zeros((1, layer_sizes[i+1]))
            self.weights.append(w)
            self.biases.append(b)
    
    def forward(self, X):
        self.activations = [X]
        A = X
        
        for i, (w, b) in enumerate(zip(self.weights, self.biases)):
            Z = A @ w + b
            # 输出层用 softmax，其余用 ReLU
            A = softmax(Z) if i == len(self.weights) - 1 else relu(Z)
            self.activations.append(A)
        
        return A
```

---

## 反向传播算法

反向传播（Backpropagation）通过链式法则计算损失函数对参数的梯度。

### 链式法则

对于复合函数 $f(g(x))$：

$$
\frac{\partial f}{\partial x} = \frac{\partial f}{\partial g} \cdot \frac{\partial g}{\partial x}
$$

### 计算流程

1. **前向传播**：计算每层输出和损失 $L$
2. **反向传播**：从输出层向输入层计算梯度
3. **参数更新**：$w \leftarrow w - \alpha \cdot \frac{\partial L}{\partial w}$

```python
class MLPTrainer:
    def __init__(self, model, learning_rate=0.01):
        self.model = model
        self.lr = learning_rate
    
    def train_step(self, X, y_onehot):
        # 1. 前向传播
        output = self.model.forward(X)
        
        # 2. 计算损失梯度（交叉熵）
        dZ = output - y_onehot  # shape: (batch, num_classes)
        
        # 3. 反向传播
        for i in range(len(self.model.weights) - 1, -1, -1):
            dW = self.model.activations[i].T @ dZ / X.shape[0]
            db = np.sum(dZ, axis=0, keepdims=True) / X.shape[0]
            
            # 传播到下一层
            if i > 0:
                dA = dZ @ self.model.weights[i].T
                dZ = dA * (self.model.activations[i] > 0)  # ReLU 梯度
            
            # 4. 更新参数
            self.model.weights[i] -= self.lr * dW
            self.model.biases[i] -= self.lr * db
```

---

## 梯度问题

### 梯度消失

深层网络中，梯度逐层指数衰减，前层参数几乎不更新。

**解决方案**：
- ReLU 激活函数
- 残差连接（ResNet）
- LSTM/GRU 门控机制
- Batch Normalization

### 梯度爆炸

梯度逐层指数增长，参数更新过大。

**解决方案**：
- 梯度裁剪：`np.clip(gradient, -threshold, threshold)`
- 合适的权重初始化
- 学习率调度

---

## 常见网络结构

### 卷积神经网络（CNN）

适合处理图像，通过卷积层提取空间特征：

```
输入 → [Conv → BN → ReLU → Pool] × N → FC → 输出
```

**核心组件**：

| 组件 | 作用 |
|------|------|
| 卷积层 | 可学习滤波器提取局部特征 |
| 池化层 | 下采样，减少参数 |
| BatchNorm | 稳定梯度，加速收敛 |
| Dropout | 正则化，防止过拟合 |

### 循环神经网络（RNN）

适合序列数据，通过隐藏状态传递历史信息：

```python
def rnn_step(x, h_prev, wx, wh, b):
    h = np.tanh(x @ wx + h_prev @ wh + b)
    return h

# RNN 长期依赖问题：梯度随序列长度指数衰减/爆炸
# 解决：LSTM、GRU
```

**LSTM（长短期记忆网络）**：

$$
\begin{aligned}
f_t &= \sigma(W_f \cdot [h_{t-1}, x_t]) \quad \text{遗忘门} \\
i_t &= \sigma(W_i \cdot [h_{t-1}, x_t]) \quad \text{输入门} \\
\tilde{C}_t &= \tanh(W_C \cdot [h_{t-1}, x_t]) \quad \text{新记忆} \\
C_t &= f_t \cdot C_{t-1} + i_t \cdot \tilde{C}_t \quad \text{细胞状态} \\
o_t &= \sigma(W_o \cdot [h_{t-1}, x_t]) \quad \text{输出门} \\
h_t &= o_t \cdot \tanh(C_t)
\end{aligned}
$$

### Transformer

基于自注意力机制（Self-Attention），适合并行处理长序列：

$$
\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

核心组件：
- **多头注意力**：并行多个注意力
- **位置编码**：注入序列位置信息
- **前馈网络**：逐位置非线性变换

---

## 训练技巧

### 优化器

| 优化器 | 更新规则 | 特点 |
|--------|----------|------|
| **SGD** | $w \leftarrow w - \alpha \nabla L$ | 简单稳定 |
| **Momentum** | $v \leftarrow \beta v + \alpha \nabla L$；$w \leftarrow w - v$ | 加速收敛 |
| **Adam** | 自适应学习率 | 默认首选 |
| **AdamW** | Adam + 权重衰减 | 学习率衰减更合理 |

### 正则化

- **Dropout**：训练时随机丢弃神经元
- **L2正则化**：$\mathcal{L}_{reg} = \mathcal{L}_0 + \lambda \|w\|^2$
- **Early Stopping**：验证集性能下降时停止
- **数据增强**：随机变换扩充数据

```python
# Dropout 实现
def dropout(x, rate=0.5):
    mask = np.random.binomial(1, 1-rate, x.shape) / (1-rate)
    return x * mask
```

---

## PyTorch 示例

```python
import torch
import torch.nn as nn
import torch.optim as optim

class Net(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(784, 128)
        self.fc2 = nn.Linear(128, 64)
        self.fc3 = nn.Linear(64, 10)
        self.dropout = nn.Dropout(0.2)
    
    def forward(self, x):
        x = torch.relu(self.fc1(x))
        x = self.dropout(x)
        x = torch.relu(self.fc2(x))
        x = self.dropout(x)
        x = self.fc3(x)
        return x

# 训练
model = Net()
criterion = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=0.001)

for epoch in range(10):
    for batch_x, batch_y in dataloader:
        optimizer.zero_grad()
        output = model(batch_x)
        loss = criterion(output, batch_y)
        loss.backward()
        optimizer.step()
```

---

## 参考资料

[^1]: Deep Learning and Neural Networks - Computer Science Authority. https://computerscienceauthority.com/deep-learning-and-neural-networks.html
