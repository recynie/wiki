---
title: 计算图与自动微分
date: 2026-04-17
description: 计算图概念、自动微分原理及PyTorch/JAX实现
tags:
  - deep-learning
  - autograd
  - pytorch
  - jax
draft: false
permalink:
---

# 计算图与自动微分

自动微分（Automatic Differentiation, AD）是现代深度学习框架的核心技术，使得梯度计算变得自动化。本文档介绍计算图的概念和主流框架的实现方式。

## 计算图基础

### 什么是计算图

计算图（Computational Graph）是一种表示数学表达式的方法：
- **节点（Node）**：表示变量（标量、向量、矩阵）或操作
- **边（Edge）**：表示数据依赖关系

### 示例：$f(x, y) = (x + y) \cdot \sin(x)$

```
        x ──┬───────────────────┐
            │                   │
            ├──────> + ──────────┼───────> * ────> f
            │                   │
        y ──┘                   │
                                │
        x ──────────────────────┴───> sin ────┘
```

### 计算图与梯度

计算图使得我们可以系统地应用链式法则，自动计算任意复杂表达式的梯度。

## 自动微分的数学原理

### 四种微分模式对比

| 模式 | 原理 | 复杂度 | 适用场景 |
|------|------|--------|----------|
| **数值微分** | $\frac{f(x+\epsilon) - f(x-\epsilon)}{2\epsilon}$ | $O(n)$ | 调试、简单函数 |
| **符号微分** | 解析表达式推导 | 表达式膨胀 | 闭式推导 |
| **前向模式AD** | 沿输入方向传播 | $O(n)$ | 少量输入，多输出 |
| **反向模式AD** | 沿输出方向传播 | $O(m)$ | 大量输入，少量输出 |

### 反向模式AD（反向传播的核心）

对于函数 $f: \mathbb{R}^n \rightarrow \mathbb{R}$（如神经网络损失函数），反向模式只需 $O(1)$ 次前向传播 + $O(1)$ 次反向传播即可得到全部梯度的雅可比矩阵。

设计算图节点 $v_i$ 的值为 $V_i$，我们有：

$$
\bar{v}_i \triangleq \frac{\partial f}{\partial v_i}
$$

**反向传播算法**：
1. 从输出节点开始，初始化 $\bar{v}_{final} = 1$
2. 按拓扑逆序遍历节点
3. 对每个节点应用：
   $$
   \bar{v}_i = \sum_{j \in children(i)} \bar{v}_j \cdot \frac{\partial v_j}{\partial v_i}
   $$

### 示例推导

计算 $f(x, y) = x + xy$ 的梯度。

**前向传播**：
- $v_{-1} = x, v_0 = y$
- $v_1 = v_{-1} \cdot v_0 = x \cdot y$
- $v_2 = v_{-1} + v_1 = x + xy = f$

**反向传播**：
- $\bar{v}_2 = \frac{\partial f}{\partial v_2} = 1$
- $\bar{v}_1 = \bar{v}_2 \cdot \frac{\partial v_2}{\partial v_1} = 1 \cdot 1 = 1$
- $\bar{v}_{-1} = \bar{v}_1 \cdot \frac{\partial v_1}{\partial v_{-1}} + \bar{v}_2 \cdot \frac{\partial v_2}{\partial v_{-1}} = 1 \cdot y + 1 \cdot 1 = y + 1$
- $\bar{v}_0 = \bar{v}_1 \cdot \frac{\partial v_1}{\partial v_0} = 1 \cdot x = x$

验证：$\frac{\partial f}{\partial x} = 1 + y, \frac{\partial f}{\partial y} = x$ ✓

## PyTorch autograd机制

### 核心概念

PyTorch使用**动态计算图**（Dynamic Computational Graph）：
- 图在每次前向传播时动态构建
- `torch.Tensor`支持`requires_grad`属性追踪计算历史
- `backward()`触发反向传播

### 基本使用

```python
import torch

# 创建需要梯度的张量
x = torch.tensor([1.0, 2.0], requires_grad=True)
y = torch.tensor([3.0, 4.0], requires_grad=True)

# 构建计算图
z = x * y + x.sum()  # z = x*y + sum(x)
print(f"z = {z}")  # z = tensor([4., 10.], grad_fn=<AddBackward0>)

# 反向传播
z.sum().backward()  # 对标量调用，或使用 z.sum().backward()

# 查看梯度
print(f"x.grad = {x.grad}")  # x.grad = tensor([4., 5.])
print(f"y.grad = {y.grad}")  # y.grad = tensor([1., 2.])
```

### 计算图结构

```python
# 查看计算图的详细信息
print(z.grad_fn)  # <AddBackward0 at 0x...>
print(z.grad_fn.next_functions)  # 子节点的grad_fn

# Function类
print(x.grad_fn)  # None (叶子节点)
```

### 非标量张量的反向传播

对于非标量输出，需要指定梯度初始值：

```python
# z是向量(2,)，不是标量
z = x * y  # z = [x0*y0, x1*y1]

# 需要传入相同形状的梯度
z.backward(torch.ones_like(z))  # 或 z.sum().backward()

# 或者使用Jacobian手动计算
from torch.autograd.functional import jacobian
J = jacobian(lambda x: x * y, x)
```

### 停止梯度追踪

```python
# 方法1: detach()
z = x * y
z_detached = z.detach()

# 方法2: with torch.no_grad():
with torch.no_grad():
    z_no_grad = x * y

# 方法3: .requires_grad_(False)
z.requires_grad_(False)
```

### 高阶导数

```python
# 计算二阶导数
x = torch.tensor([2.0, 3.0], requires_grad=True)
y = x ** 3

# 一阶导数：dy/dx = 3*x^2
grad1 = torch.autograd.grad(y.sum(), x, create_graph=True)
print(f"一阶导数: {grad1}")  # tensor([12., 27.])

# 二阶导数：d²y/dx² = 6*x
grad2 = torch.autograd.grad(grad1[0].sum(), x)
print(f"二阶导数: {grad2}")  # tensor([12., 18.])
```

### 训练循环中的autograd

```python
import torch.nn as nn

model = nn.Sequential(
    nn.Linear(10, 64),
    nn.ReLU(),
    nn.Linear(64, 1)
)

optimizer = torch.optim.Adam(model.parameters(), lr=0.01)
criterion = nn.MSELoss()

for epoch in range(100):
    # 前向传播（动态构建计算图）
    X_batch, y_batch = get_batch()
    predictions = model(X_batch)
    loss = criterion(predictions, y_batch)
    
    # 反向传播
    optimizer.zero_grad()  # 清零梯度
    loss.backward()         # 计算梯度
    
    # 参数更新
    optimizer.step()        # 更新参数
```

## JAX的自动微分

### JAX vs PyTorch

| 特性 | PyTorch | JAX |
|------|---------|-----|
| 计算图 | 动态 | 惰性（Lazy） |
| 编译 | 可选（torch.compile） | 默认JIT |
| 纯函数 | 否 | 是 |
| 不可变性 | 否 | 是 |

### 核心API

```python
import jax
import jax.numpy as jnp

# 定义函数（必须纯函数，无副作用）
def sum_squares(x):
    return jnp.sum(x ** 2)

# 一阶导数
grad_sum_squares = jax.grad(sum_squares)

x = jnp.array([1.0, 2.0, 3.0])
print(f"函数值: {sum_squares(x)}")       # 14.0
print(f"梯度: {grad_sum_squares(x)}")    # [2., 4., 6.]
```

### value_and_grad

```python
# 同时返回函数值和梯度
def loss_fn(params):
    W, b = params
    return jnp.sum((W @ x + b - y) ** 2)

# 返回 (loss, grads)
loss, grads = jax.value_and_grad(loss_fn)((W, b))
```

### JIT编译与微分

```python
# JIT编译的函数也可以求导
@jax.jit
def forward(x, params):
    W, b = params
    return jnp.tanh(W @ x + b)

# 编译后的函数求导
grad_forward = jax.jit(jax.grad(forward))

# 或直接组合
grad_forward_jit = jax.jit(jax.grad(forward))
```

### Jacobian和Hessian

```python
# Jacobian: 多输出函数的梯度
J = jax.jacfwd(sum_squares)(x)
print(J)  # [2., 4., 6.]

# Hessian: 二阶导数矩阵
H = jax.hessian(sum_squares)(x)
# [[2., 0., 0.],
#  [0., 4., 0.],
#  [0., 0., 6.]]
```

### Pytrees：复杂参数结构

```python
# JAX处理嵌套参数结构
params = {
    'linear1': {'W': W1, 'b': b1},
    'linear2': {'W': W2, 'b': b2}
}

# grads也是相同的结构
grads = jax.grad(loss_fn)(params)

# 更新参数
def sgd_update(params, grads, lr):
    return jax.tree_util.map(
        lambda p, g: p - lr * g,
        params, grads
    )
```

## 计算图实现：手动构建

### 反向模式AD的完整实现

```python
import numpy as np
from typing import List, Callable

class Value:
    """计算图中的节点"""
    def __init__(self, data, _children=(), _op=''):
        self.data = np.array(data)
        self.grad = np.zeros_like(self.data, dtype=float)
        self._backward = lambda: None
        self._prev = set(_children)
        self._op = _op
    
    def __add__(self, other):
        other = other if isinstance(other, Value) else Value(other)
        out = Value(self.data + other.data, (self, other), '+')
        
        def _backward():
            self.grad += out.grad * 1
            other.grad += out.grad * 1
        out._backward = _backward
        return out
    
    def __mul__(self, other):
        other = other if isinstance(other, Value) else Value(other)
        out = Value(self.data * other.data, (self, other), '*')
        
        def _backward():
            self.grad += out.grad * other.data
            other.grad += out.grad * self.data
        out._backward = _backward
        return out
    
    def tanh(self):
        out = Value(np.tanh(self.data), (self,), 'tanh')
        
        def _backward():
            self.grad += out.grad * (1 - np.tanh(self.data)**2)
        out._backward = _backward
        return out
    
    def backward(self):
        # 拓扑排序
        topo = []
        visited = set()
        def build_topo(v):
            if v not in visited:
                visited.add(v)
                for child in v._prev:
                    build_topo(child)
                topo.append(v)
        build_topo(self)
        
        # 反向传播
        self.grad = np.ones_like(self.data)
        for node in reversed(topo):
            node._backward()

# 使用示例
x = Value([1.0, 2.0])
w = Value([0.5, -0.5])
b = Value([0.1])

z = x * w + b
a = z.tanh()
a.backward()

print(f"a = {a.data}")
print(f"x.grad = {x.grad}")
```

## 性能对比

### 计算复杂度

| 框架 | 前向 | 反向 |
|------|------|------|
| 数值微分 | $O(n)$ | $O(n)$ |
| 符号微分 | 表达式膨胀 | 表达式膨胀 |
| 反向模式AD | $O(1)$ | $O(1)$ |

### PyTorch vs JAX性能

```python
# PyTorch
import torch
import time

x = torch.randn(1000, 1000, requires_grad=True)
start = time.time()
for _ in range(100):
    y = x @ x.T
    y.sum().backward()
    x.grad.zero_()
print(f"PyTorch: {time.time()-start:.2f}s")

# JAX
import jax
import jax.numpy as jnp

x = jnp.array(np.random.randn(1000, 1000))
loss_fn = lambda x: jnp.sum(x @ x.T)
grad_fn = jax.grad(loss_fn)
start = time.time()
for _ in range(100):
    grads = grad_fn(x)
print(f"JAX: {time.time()-start:.2f}s")
```

## 参考

[^1]: Baydin et al., "Automatic Differentiation in Machine Learning: A Survey"
[^2]: PyTorch Autograd Documentation. https://pytorch.org/docs/stable/autograd.html
[^3]: JAX Documentation. https://jax.readthedocs.io/
[^4]: Karpathy's "Micrograd" - A tiny autograd engine
