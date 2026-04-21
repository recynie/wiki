---
title: 矩阵微积分与深度学习
date: 2026-04-21
description: 神经网络反向传播的矩阵形式推导与自动微分原理
tags:
  - deep-learning
  - linear-algebra
  - backpropagation
  - matrix-calculus
  - automatic-differentiation
draft: false
permalink:
---

# 矩阵微积分与深度学习

深度学习的核心是优化海量参数以最小化损失函数。理解矩阵微积分是掌握反向传播算法和神经网络训练机制的关键。本章将从矩阵微分基础出发，系统推导神经网络中的梯度计算。

## 矩阵微分基础

### 符号约定

在神经网络中，我们使用以下约定：

| 符号 | 含义 | 形状 |
|------|------|------|
| $\mathbf{X} \in \mathbb{R}^{m \times n}$ | 输入矩阵 | $m \times n$ |
| $\mathbf{W} \in \mathbb{R}^{n \times k}$ | 权重矩阵 | $n \times k$ |
| $\mathbf{y} \in \mathbb{R}^{m \times k}$ | 输出矩阵 | $m \times k$ |
| $\mathcal{L}$ | 损失函数 | 标量 |

### 梯度定义

对于标量函数 $f: \mathbb{R}^{m \times n} \to \mathbb{R}$，梯度定义为：

$$
\nabla_{\mathbf{X}} f = \begin{pmatrix}
\frac{\partial f}{\partial X_{11}} & \frac{\partial f}{\partial X_{12}} & \cdots & \frac{\partial f}{\partial X_{1n}} \\
\frac{\partial f}{\partial X_{21}} & \frac{\partial f}{\partial X_{22}} & \cdots & \frac{\partial f}{\partial X_{2n}} \\
\vdots & \vdots & \ddots & \vdots \\
\frac{\partial f}{\partial X_{m1}} & \frac{\partial f}{\partial X_{m2}} & \cdots & \frac{\partial f}{\partial X_{mn}}
\end{pmatrix}
$$

**注意**：梯度与原矩阵形状相同。

### 雅可比矩阵

对于向量函数 $\mathbf{f}: \mathbb{R}^n \to \mathbb{R}^m$，雅可比矩阵为：

$$
\mathbf{J} = \frac{\partial \mathbf{f}}{\partial \mathbf{x}} = \begin{pmatrix}
\frac{\partial f_1}{\partial x_1} & \frac{\partial f_1}{\partial x_2} & \cdots & \frac{\partial f_1}{\partial x_n} \\
\frac{\partial f_2}{\partial x_1} & \frac{\partial f_2}{\partial x_2} & \cdots & \frac{\partial f_2}{\partial x_n} \\
\vdots & \vdots & \ddots & \vdots \\
\frac{\partial f_m}{\partial x_1} & \frac{\partial f_m}{\partial x_2} & \cdots & \frac{\partial f_m}{\partial x_n}
\end{pmatrix} \in \mathbb{R}^{m \times n}
$$

### 链式法则

深度学习中最核心的工具是链式法则。设 $\mathbf{z} = \mathbf{g}(\mathbf{y})$，$\mathbf{y} = \mathbf{f}(\mathbf{x})$，则：

$$
\frac{\partial \mathbf{z}}{\partial \mathbf{x}} = \frac{\partial \mathbf{z}}{\partial \mathbf{y}} \cdot \frac{\partial \mathbf{y}}{\partial \mathbf{x}}
$$

其中乘积为**矩阵乘法**。

## 神经网络层求导

### 全连接层

全连接层前向传播：

$$
\mathbf{Z} = \mathbf{X}\mathbf{W} + \mathbf{b}
$$

**对输入 $\mathbf{X}$ 求导**：

$$
\frac{\partial \mathcal{L}}{\partial \mathbf{X}} = \frac{\partial \mathcal{L}}{\partial \mathbf{Z}} \cdot \mathbf{W}^T
$$

**对权重 $\mathbf{W}$ 求导**：

$$
\frac{\partial \mathcal{L}}{\partial \mathbf{W}} = \mathbf{X}^T \cdot \frac{\partial \mathcal{L}}{\partial \mathbf{Z}}
$$

**对偏置 $\mathbf{b}$ 求导**：

$$
\frac{\partial \mathcal{L}}{\partial \mathbf{b}} = \text{sum}\left(\frac{\partial \mathcal{L}}{\partial \mathbf{Z}}\right)
$$

其中 $\text{sum}$ 沿行维度求和。

### 逐元素激活函数

设 $\mathbf{Y} = \sigma(\mathbf{Z})$，其中 $\sigma$ 逐元素作用。

$$
\frac{\partial \mathcal{L}}{\partial \mathbf{Z}} = \frac{\partial \mathcal{L}}{\partial \mathbf{Y}} \odot \sigma'(\mathbf{Z})
$$

其中 $\odot$ 为哈达玛积（元素对应乘积）。

### Softmax层

对于多分类问题，Softmax输出为：

$$
\hat{y}_i = \text{softmax}(\mathbf{z})_i = \frac{e^{z_i}}{\sum_j e^{z_j}}
$$

**交叉熵损失下的梯度**：

设真实标签为 $\mathbf{y}$（one-hot编码），交叉熵损失为 $\mathcal{L} = -\sum_i y_i \log \hat{y}_i$，则：

$$
\frac{\partial \mathcal{L}}{\partial \mathbf{z}} = \hat{\mathbf{y}} - \mathbf{y}
$$

这个简洁的结论是深度学习中最重要的公式之一。[^1]

### 矩阵乘法的梯度汇总

对于损失函数 $\mathcal{L}$，以下矩阵恒等式在反向传播中反复出现：

| 前向传播 | $\partial \mathcal{L} / \partial \mathbf{X}$ | $\partial \mathcal{L} / \partial \mathbf{W}$ |
|----------|-----------------------------------------|---------------------------------------------|
| $\mathbf{Y} = \mathbf{X}\mathbf{W}$ | $\mathbf{G} \mathbf{W}^T$ | $\mathbf{X}^T \mathbf{G}$ |
| $\mathbf{Y} = \mathbf{W}\mathbf{X}$ | $\mathbf{W}^T \mathbf{G}$ | $\mathbf{G} \mathbf{X}^T$ |

其中 $\mathbf{G} = \partial \mathcal{L} / \partial \mathbf{Y}$ 是下游梯度。

## 反向传播算法矩阵形式

### 两层全连接网络

考虑最简单的两层网络：

$$
\mathbf{h} = \sigma(\mathbf{X}\mathbf{W}_1 + \mathbf{b}_1), \quad \mathbf{y} = \mathbf{h}\mathbf{W}_2 + \mathbf{b}_2
$$

**反向传播推导**：

**Step 1：输出层梯度**

$$
\mathbf{G}_y = \frac{\partial \mathcal{L}}{\partial \mathbf{y}} = \hat{\mathbf{y}} - \mathbf{y} \quad \text{（交叉熵）}
$$

**Step 2：第二层权重梯度**

$$
\frac{\partial \mathcal{L}}{\partial \mathbf{W}_2} = \mathbf{h}^T \mathbf{G}_y, \quad 
\frac{\partial \mathcal{L}}{\partial \mathbf{b}_2} = \text{sum}(\mathbf{G}_y)
$$

**Step 3：传播到隐藏层**

$$
\mathbf{G}_h = \mathbf{G}_y \mathbf{W}_2^T \odot \sigma'(\mathbf{h})
$$

**Step 4：第一层权重梯度**

$$
\frac{\partial \mathcal{L}}{\partial \mathbf{W}_1} = \mathbf{X}^T \mathbf{G}_h, \quad
\frac{\partial \mathcal{L}}{\partial \mathbf{b}_1} = \text{sum}(\mathbf{G}_h)
$$

### 批量处理的矩阵形式

设批量大小为 $B$，输入维度为 $D_{in}$，隐藏维度为 $D_{hidden}$，输出维度为 $D_{out}$：

$$
\mathbf{X} \in \mathbb{R}^{B \times D_{in}}, \quad \mathbf{W}_1 \in \mathbb{R}^{D_{in} \times D_{hidden}}, \quad \mathbf{W}_2 \in \mathbb{R}^{D_{hidden} \times D_{out}}
$$

反向传播时，所有样本的梯度可以**批量计算**，充分利用GPU的并行能力。

### PyTorch实现

```python
import torch
import torch.nn as nn

class TwoLayerNet(nn.Module):
    def __init__(self, input_dim, hidden_dim, output_dim):
        super().__init__()
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.fc2 = nn.Linear(hidden_dim, output_dim)
    
    def forward(self, x):
        x = torch.relu(self.fc1(x))
        x = self.fc2(x)
        return x

# 手动验证梯度计算
model = TwoLayerNet(784, 256, 10)
x = torch.randn(32, 784)  # batch_size=32
y = torch.randn(32, 10)

output = model(x)
loss = torch.mean((output - y) ** 2)  # MSE损失
loss.backward()

# 检查梯度
for name, param in model.named_parameters():
    print(f"{name}: grad.shape = {param.grad.shape}")
```

## 自动微分原理

### 计算图

自动微分的核心是构建**计算图**（Computational Graph）。每个节点表示一个变量，每条边表示一个运算。

**前向传播**：

$$
x_1 \xrightarrow{\text{op}_1} x_2 \xrightarrow{\text{op}_2} x_3 \xrightarrow{\text{op}_3} \cdots \xrightarrow{\text{op}_n} \mathcal{L}
$$

**反向传播**：

$$
\frac{\partial \mathcal{L}}{\partial x_{n-1}} \xleftarrow{\text{op}_n} \frac{\partial \mathcal{L}}{\partial x_n} \xleftarrow{\text{op}_{n-1}} \cdots \xleftarrow{\text{op}_2} \frac{\partial \mathcal{L}}{\partial x_1}
$$

### PyTorch autograd机制

PyTorch使用**反向模式自动微分**（Reverse Mode AD），适合计算图输入多、输出少的场景（如神经网络的单个损失值）。

```python
import torch

# 创建需要梯度的张量
x = torch.tensor([1.0, 2.0, 3.0], requires_grad=True)
y = torch.tensor([4.0, 5.0, 6.0], requires_grad=True)

# 构建计算图
z = (x * 2 + y ** 2).sum()
z.backward()

# 自动得到梯度
print(x.grad)  # dz/dx = 2
print(y.grad)  # dz/dy = 2*y = [8, 10, 12]
```

### JAX vs PyTorch

| 特性 | PyTorch | JAX |
|------|---------|-----|
| 梯度函数 | `loss.backward()` | `jax.grad(loss_fn)` |
| 函数式 | 命令式 | 函数式 |
| 即时编译 | `torch.compile` | `jax.jit` |
| 纯函数 | 不要求 | 严格要求 |

```python
# JAX版本
import jax
import jax.numpy as jnp

def loss_fn(params, x, y):
    w1, b1, w2, b2 = params
    h = jnp.maximum(0, x @ w1 + b1)  # ReLU
    return jnp.mean((h @ w2 + b2 - y) ** 2)

# 自动梯度
grad_fn = jax.grad(loss_fn)
grads = grad_fn(params, x, y)
```

### 雅可比矩阵的PyTorch计算

```python
# 计算完整雅可比矩阵
x = torch.randn(3, requires_grad=True)
y = x ** 2  # 向量输出

# 方法1：逐元素backward
jacobian = torch.zeros(3, 3)
for i in range(3):
    y[i].backward(retain_graph=True)
    jacobian[i] = x.grad
    x.grad.zero_()

# 方法2：使用torch.autograd.functional.jacobian
from torch.autograd.functional import jacobian
jacobian = jacobian(lambda x: x ** 2, x)
```

## 梯度问题与数值稳定性

### 梯度消失与爆炸

在深层网络中，梯度是多个雅可比矩阵的连乘：

$$
\frac{\partial \mathcal{L}}{\partial \mathbf{W}^{(1)}} = \mathbf{J}^{(L)} \cdots \mathbf{J}^{(2)} \mathbf{G}^{(1)}
$$

**梯度消失**：当 $| \lambda_{\max}(\mathbf{J}) | < 1$ 时，梯度指数衰减。

**梯度爆炸**：当 $| \lambda_{\max}(\mathbf{J}) | > 1$ 时，梯度指数增长。

### 梯度裁剪

```python
# 梯度裁剪
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
```

### 数值精度问题

```python
# 避免log(0)导致-inf
log_prob = torch.log_softmax(logits, dim=-1)
# 等价于在softmax内部进行数值稳定化处理
```

## 高级主题：二阶优化

### Hessian矩阵

Hessian矩阵是损失函数的二阶导数：

$$
\mathbf{H} = \nabla^2_{\mathbf{W}} \mathcal{L} = \frac{\partial^2 \mathcal{L}}{\partial \mathbf{W} \partial \mathbf{W}^T}
$$

**牛顿法更新**：

$$
\mathbf{W}_{new} = \mathbf{W} - \mathbf{H}^{-1} \nabla \mathcal{L}
$$

### 自然梯度

自然梯度考虑了参数空间黎曼几何：

$$
\tilde{\nabla} \mathcal{L} = \mathbf{F}^{-1} \nabla \mathcal{L}
$$

其中 $\mathbf{F}$ 是Fisher信息矩阵。

## 参考

[^1]: Goodfellow, Bengio, Courville. "Deep Learning". MIT Press, 2016. Chapter 6.
[^2]: Parr, Howard. "The Matrix Calculus You Need For Deep Learning". 2018.
[^3]: Paszke et al. "Automatic Differentiation in PyTorch". NIPS-W 2017.

---

*相关词条：[[backpropagation|反向传播算法]]，[[computational-graph|计算图与自动微分]]，[[optimizers|神经网络优化器]]*
