---
title: 反向传播算法详解
date: 2026-04-17
description: 反向传播算法的数学推导与实现
tags:
  - deep-learning
  - backpropagation
  - neural-network
draft: false
permalink:
---

# 反向传播算法详解

反向传播（Backpropagation）是训练神经网络的核心算法，通过链式法则高效计算损失函数对参数的梯度。本文档详细推导反向传播的数学原理。

## 链式法则基础

### 回顾：一元链式法则

对于复合函数 $y = f(g(x))$：

$$
\frac{dy}{dx} = \frac{dy}{dg} \cdot \frac{dg}{dx}
$$

### 多元链式法则

对于多变量函数，设 $z = f(u_1, u_2, ..., u_n)$，每个 $u_i = g_i(x_1, ..., x_m)$：

$$
\frac{\partial z}{\partial x_j} = \sum_{i=1}^{n} \frac{\partial z}{\partial u_i} \cdot \frac{\partial u_i}{\partial x_j}
$$

## 单神经元梯度推导

### 网络结构

考虑单个神经元：
- 输入：$x_1, x_2, ..., x_n$
- 权重：$w_1, w_2, ..., w_n$
- 偏置：$b$
- 加权和：$z = \sum_i w_i x_i + b$
- 输出：$\hat{y} = \sigma(z)$

### 前向传播

```python
import numpy as np

def sigmoid(z):
    return 1 / (1 + np.exp(-z))

def forward(x, w, b):
    z = np.dot(x, w) + b  # x: (n,), w: (n,), z: scalar
    y_hat = sigmoid(z)
    return y_hat
```

### 反向传播推导

设损失函数为均方误差：$C = \frac{1}{2}(y - \hat{y})^2$

**步骤1：输出层梯度**

$$
\frac{\partial C}{\partial \hat{y}} = \hat{y} - y
$$

**步骤2：激活函数梯度**

$$
\frac{\partial \hat{y}}{\partial z} = \sigma'(z) = \sigma(z)(1 - \sigma(z))
$$

**步骤3：应用链式法则**

$$
\frac{\partial C}{\partial z} = \frac{\partial C}{\partial \hat{y}} \cdot \frac{\partial \hat{y}}{\partial z} = (\hat{y} - y) \cdot \sigma'(z)
$$

**步骤4：权重梯度**

$$
\frac{\partial C}{\partial w_i} = \frac{\partial C}{\partial z} \cdot \frac{\partial z}{\partial w_i} = \frac{\partial C}{\partial z} \cdot x_i
$$

$$
\frac{\partial C}{\partial b} = \frac{\partial C}{\partial z} \cdot \frac{\partial z}{\partial b} = \frac{\partial C}{\partial z}
$$

### 完整梯度计算代码

```python
def backward(x, y, y_hat):
    # 前向传播值
    z = np.dot(x, w) + b
    sigma_z = sigmoid(z)
    
    # dC/d(y_hat)
    d_loss_d_yhat = y_hat - y
    
    # d(y_hat)/dz = sigmoid'(z)
    d_sigmoid = sigma_z * (1 - sigma_z)
    
    # dC/dz
    d_loss_d_z = d_loss_d_yhat * d_sigmoid
    
    # dC/dw_i = dC/dz * dz/dw_i = dC/dz * x_i
    d_loss_d_w = d_loss_d_z * x
    
    # dC/db = dC/dz * dz/db = dC/dz * 1
    d_loss_d_b = d_loss_d_z
    
    return d_loss_d_w, d_loss_d_b
```

## 多层感知机完整推导

### 网络结构

考虑 $L$ 层神经网络：
- 层 $l$ 有 $n_l$ 个神经元
- $W^l \in \mathbb{R}^{n_{l-1} \times n_l}$：权重矩阵
- $b^l \in \mathbb{R}^{n_l}$：偏置向量
- $z^l = a^{l-1} W^l + b^l$：层输入的线性变换
- $a^l = \sigma(z^l)$：层输出（激活值）

### Nielsen四个核心方程

#### BP1：输出层误差

定义层 $l$ 的误差 $\delta^l_j$：

$$
\delta^l_j = \frac{\partial C}{\partial z^l_j}
$$

对于输出层 $L$：

$$
\delta^L_j = \frac{\partial C}{\partial a^L_j} \cdot \sigma'(z^L_j)
\tag{BP1}
$$

向量形式：

$$
\delta^L = \nabla_a C \odot \sigma'(z^L)
$$

**MSE损失的特例**：
若 $C = \frac{1}{2}\|y - a^L\|^2$，则 $\nabla_a C = a^L - y$。

#### BP2：误差反向传播

$$
\delta^l = (W^{l+1})^T \delta^{l+1} \odot \sigma'(z^l)
\tag{BP2}
$$

推导：
$$
\delta^l_j = \frac{\partial C}{\partial z^l_j}
= \sum_k \frac{\partial C}{\partial z^{l+1}_k} \cdot \frac{\partial z^{l+1}_k}{\partial a^l_j} \cdot \frac{\partial a^l_j}{\partial z^l_j}
= \sum_k \delta^{l+1}_k \cdot W^l_{jk} \cdot \sigma'(z^l_j)
$$

#### BP3：偏置梯度

$$
\frac{\partial C}{\partial b^l_j} = \delta^l_j
\tag{BP3}
$$

#### BP4：权重梯度

$$
\frac{\partial C}{\partial W^l_{jk}} = a^{l-1}_j \delta^l_k
\tag{BP4}
$$

### 梯度计算示例

#### 示例网络

```
输入(2) → 隐藏层(3) → 输出(2)
```

```python
import numpy as np

class MLP:
    def __init__(self, layer_sizes):
        """layer_sizes: [input_dim, hidden1, ..., output_dim]"""
        self.L = len(layer_sizes) - 1
        self.W = []
        self.b = []
        
        for i in range(self.L):
            # Xavier初始化
            scale = np.sqrt(2.0 / layer_sizes[i])
            self.W.append(np.random.randn(layer_sizes[i], layer_sizes[i+1]) * scale)
            self.b.append(np.zeros((1, layer_sizes[i+1])))
    
    def sigmoid(self, z):
        return 1 / (1 + np.exp(-np.clip(z, -500, 500)))
    
    def sigmoid_prime(self, z):
        s = self.sigmoid(z)
        return s * (1 - s)
    
    def forward(self, X):
        self.z = []  # 存储每层的线性输出
        self.a = [X]  # 存储每层的激活输出
        
        for i in range(self.L):
            z = self.a[-1] @ self.W[i] + self.b[i]
            self.z.append(z)
            a = self.sigmoid(z) if i < self.L - 1 else z  # 输出层不用激活
            self.a.append(a)
        
        return self.a[-1]
    
    def backward(self, y, loss='mse'):
        m = y.shape[0]
        self.grad_W = []
        self.grad_b = []
        
        # 输出层误差 (BP1)
        if loss == 'mse':
            delta = (self.a[-1] - y) * self.sigmoid_prime(self.z[-1])
        elif loss == 'cross_entropy':
            delta = self.a[-1] - y  # softmax + CE 的简化形式
        
        # 从输出层向输入层反向传播
        for l in range(self.L - 1, -1, -1):
            # BP3: 偏置梯度
            self.grad_b.append(delta.sum(axis=0, keepdims=True) / m)
            
            # BP4: 权重梯度
            self.grad_W.append(self.a[l].T @ delta / m)
            
            # BP2: 传播到下一层（如果不是输入层）
            if l > 0:
                delta = (delta @ self.W[l].T) * self.sigmoid_prime(self.z[l-1])
        
        # 反转，因为我们是反向遍历的
        self.grad_W = self.grad_W[::-1]
        self.grad_b = self.grad_b[::-1]
    
    def update_params(self, learning_rate):
        for i in range(self.L):
            self.W[i] -= learning_rate * self.grad_W[i]
            self.b[i] -= learning_rate * self.grad_b[i]
```

### 训练循环

```python
def train(model, X_train, y_train, epochs, lr):
    losses = []
    
    for epoch in range(epochs):
        # 前向传播
        y_pred = model.forward(X_train)
        
        # 计算损失
        loss = 0.5 * np.sum((y_pred - y_train) ** 2) / X_train.shape[0]
        losses.append(loss)
        
        # 反向传播
        model.backward(y_train, loss='mse')
        
        # 更新参数
        model.update_params(lr)
        
        if epoch % 100 == 0:
            print(f"Epoch {epoch}, Loss: {loss:.4f}")
    
    return losses
```

## 梯度消失与梯度爆炸

### 问题分析

### 梯度消失

在深层网络中，梯度逐层指数衰减：

$$
\frac{\partial C}{\partial W^1} = \frac{\partial C}{\partial a^L} \cdot \sigma'(z^L) \cdot W^2 \cdot \sigma'(z^2) \cdots W^L \cdot \sigma'(z^1)
$$

若 $|\sigma'(z) \cdot W| < 1$，梯度逐层衰减至接近0。

### 梯度爆炸

若 $|W \cdot \sigma'(z)| > 1$，梯度指数增长。

### 解决方案

| 方法 | 原理 |
|------|------|
| **ReLU激活** | $\sigma'(z) = 1$ 当 $z > 0$，缓解消失 |
| **残差连接** | 梯度可直接传递 |
| **BatchNorm** | 稳定每层输入分布 |
| **梯度裁剪** | $\\|\nabla\| \leftarrow \min(\\|\nabla\\|, threshold)$ |
| **合适的初始化** | Xavier/He初始化 |

```python
# 梯度裁剪
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)

# He初始化
W = np.random.randn(n_in, n_out) * np.sqrt(2.0 / n_in)
```

## 计算复杂度分析

| 操作 | 前向传播 | 反向传播 |
|------|----------|----------|
| 每层乘法 | $O(m \cdot n_{l-1} \cdot n_l)$ | $O(m \cdot n_{l-1} \cdot n_l)$ |
| 总复杂度 | $O(\sum_l m \cdot n_{l-1} \cdot n_l)$ | $O(\sum_l m \cdot n_{l-1} \cdot n_l)$ |

反向传播与前向传播的计算量基本相同，这使得梯度计算非常高效。

## 常见激活函数的梯度

| 激活函数 | 正向 $\sigma(z)$ | 梯度 $\sigma'(z)$ |
|----------|------------------|-------------------|
| Sigmoid | $\frac{1}{1+e^{-z}}$ | $\sigma(z)(1-\sigma(z))$ |
| Tanh | $\frac{e^z - e^{-z}}{e^z + e^{-z}}$ | $1 - \tanh^2(z)$ |
| ReLU | $\max(0, z)$ | $1$ if $z > 0$, else $0$ |
| Leaky ReLU | $\max(\alpha z, z)$ | $1$ if $z > 0$, else $\alpha$ |

## 参考

[^1]: Nielsen, Michael. "Neural Networks and Deep Learning." Determination Press, 2015.
[^2]: Goodfellow, Bengio, Courville. "Deep Learning." MIT Press, 2016.
[^3]: CS231n Convolutional Neural Networks for Visual Recognition. Stanford University.
