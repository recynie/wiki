---
title: 深度学习优化器
date: 2026-04-20
description: SGD、Adam、AdamW等优化算法的理论与实践
tags:
  - deep-learning
  - optimization
  - sgd
  - adam
draft: false
permalink:
---

# 深度学习优化器

深度学习模型的训练本质上是一个优化问题：我们需要在巨大的参数空间中找到一组参数，使得损失函数最小化。优化器（Optimizer）决定了如何更新参数，直接影响模型的收敛速度、最终性能和泛化能力。

## 一阶梯度优化基础

深度学习中使用的几乎所有优化器都是**一阶优化器**，即利用损失函数的一阶导数（梯度）来指导参数更新。与二阶优化器（如牛顿法）相比，一阶方法的计算开销更低，更适合大规模深度学习模型。

### 梯度下降

最基本的参数更新规则：

$$
\theta_{t+1} = \theta_t - \eta \cdot \nabla_\theta \mathcal{L}(\theta_t)
$$

其中 $\eta$ 是学习率，$\nabla_\theta \mathcal{L}$ 是损失函数关于参数的梯度。

```python
# 简单的梯度下降示例
theta = theta - learning_rate * gradient
```

### 随机梯度下降（SGD）

传统梯度下降需要计算整个数据集的梯度，在大数据场景下不可行。**随机梯度下降**（SGD）每次使用单个样本或小批量样本来估计梯度：

```python
def sgd_step(model, batch_x, batch_y, lr):
    """SGD单步更新"""
    optimizer = torch.optim.SGD(model.parameters(), lr=lr)
    optimizer.zero_grad()
    loss = criterion(model(batch_x), batch_y)
    loss.backward()
    optimizer.step()
```

## 带动量的SGD

### 动量法原理

动量法（Momentum）模拟物理中的动量概念，加速收敛并在相关方向上累积速度：

$$
\begin{aligned}
v_t &= \gamma v_{t-1} + \eta \nabla_\theta \mathcal{L}(\theta_t) \\
\theta_{t+1} &= \theta_t - v_t
\end{aligned}
$$

其中 $\gamma \in [0, 1)$ 是动量系数，通常取 $0.9$。

**直观理解**：
- 当梯度方向一致时，动量累积使更新幅度增大
- 当梯度方向变化时，动量提供惯性，平滑更新路径
- 动量可以理解为对历史梯度的指数加权平均

```python
class SGDWithMomentum:
    def __init__(self, params, lr=0.01, momentum=0.9):
        self.params = list(params)
        self.lr = lr
        self.momentum = momentum
        self.velocity = [torch.zeros_like(p) for p in self.params]
    
    def step(self, gradients):
        for i, (param, grad, vel) in enumerate(zip(self.params, gradients, self.velocity)):
            # 更新速度
            vel.mul_(self.momentum).add_(grad)
            # 更新参数
            param.sub_(self.lr * vel)
    
    def zero_grad(self):
        pass
```

### PyTorch实现

```python
optimizer = torch.optim.SGD(
    model.parameters(), 
    lr=0.01,           # 学习率
    momentum=0.9,       # 动量系数
    weight_decay=0     # L2正则化系数（可选）
)
```

### Nesterov动量

Nesterov动量是动量法的改进版本，在计算梯度前先看一步：

$$
\begin{aligned}
v_t &= \gamma v_{t-1} + \eta \nabla_\theta \mathcal{L}(\theta_t - \gamma v_{t-1}) \\
\theta_{t+1} &= \theta_t - v_t
\end{aligned}
$$

```python
optimizer = torch.optim.SGD(
    model.parameters(), 
    lr=0.01,
    momentum=0.9,
    nesterov=True      # 启用Nesterov动量
)
```

## 自适应学习率方法

传统的SGD使用固定学习率，需要精心调参。自适应方法为每个参数维护独立的学习率。

### AdaGrad

AdaGrad对稀疏特征的自适应学习率效果较好：

$$
\theta_{t+1,i} = \theta_{t,i} - \frac{\eta}{\sqrt{G_{t,ii} + \epsilon}} \cdot g_{t,i}
$$

其中 $G_t$ 是历史梯度平方的累积和，$\epsilon$ 防止除零。

```python
optimizer = torch.optim.Adagrad(model.parameters(), lr=0.01)
```

**问题**：学习率会随时间单调递减，可能导致训练过早停止。

### RMSProp

RMSProp通过指数移动平均来解决AdaGrad的问题：

$$
\begin{aligned}
v_t &= \rho v_{t-1} + (1-\rho) g_t^2 \\
\theta_{t+1} &= \theta_t - \frac{\eta}{\sqrt{v_t + \epsilon}} \cdot g_t
\end{aligned}
$$

其中 $\rho$ 通常取 $0.9$。

```python
optimizer = torch.optim.RMSprop(model.parameters(), lr=0.01, rho=0.9)
```

### Adam

Adam（Adaptive Moment Estimation）结合了动量和RMSProp的思想：

$$
\begin{aligned}
m_t &= \beta_1 m_{t-1} + (1-\beta_1) g_t \quad \text{（一阶矩，类比动量）}\\
v_t &= \beta_2 v_{t-1} + (1-\beta_2) g_t^2 \quad \text{（二阶矩，类比RMSProp）}\\
\hat{m}_t &= \frac{m_t}{1-\beta_1^t} \quad \text{（偏差校正）}\\
\hat{v}_t &= \frac{v_t}{1-\beta_2^t} \quad \text{（偏差校正）}\\
\theta_{t+1} &= \theta_t - \frac{\eta}{\sqrt{\hat{v}_t} + \epsilon} \hat{m}_t
\end{aligned}
$$

偏差校正的原因：$m_t$ 和 $v_t$ 的初始值接近0，在训练早期会产生偏差。

```python
def adam_update(params, grads, m, v, t, lr, beta1=0.9, beta2=0.999, eps=1e-8):
    """Adam优化器的核心更新公式"""
    for i, (p, g) in enumerate(zip(params, grads)):
        # 更新一阶矩估计
        m[i] = beta1 * m[i] + (1 - beta1) * g
        # 更新二阶矩估计
        v[i] = beta2 * v[i] + (1 - beta2) * (g ** 2)
        
        # 偏差校正
        m_hat = m[i] / (1 - beta1 ** t)
        v_hat = v[i] / (1 - beta2 ** t)
        
        # 参数更新
        p.data -= lr * m_hat / (torch.sqrt(v_hat) + eps)
```

```python
optimizer = torch.optim.Adam(
    model.parameters(),
    lr=0.001,            # 默认学习率
    betas=(0.9, 0.999),  # 动量和RMSProp系数
    eps=1e-8,            # 防止除零
    weight_decay=0       # 权重衰减
)
```

### AdamW

AdamW（Weight Decay Decoupled）由Loshchilov和Hutter在2017年提出，解决了Adam中L2正则化与权重衰减不等价的问题。[^1]

**核心发现**：在Adam中，L2正则化（$L2\_loss = \frac{\lambda}{2}\sum_i \theta_i^2$）的梯度是 $\lambda \theta$，但由于Adam对梯度做了归一化，这个项的实际效果取决于参数的历史梯度统计量，与真正的权重衰减（直接 $\theta \leftarrow (1-\eta\lambda)\theta$）效果不同。

```python
def adamw_update(params, grads, m, v, t, lr, beta1, beta2, eps, weight_decay):
    """AdamW优化器"""
    for i, (p, g) in enumerate(zip(params, grads)):
        m[i] = beta1 * m[i] + (1 - beta1) * g
        v[i] = beta2 * v[i] + (1 - beta2) * (g ** 2)
        
        m_hat = m[i] / (1 - beta1 ** t)
        v_hat = v[i] / (1 - beta2 ** t)
        
        # 关键区别：权重衰减与梯度归一化解耦
        p.data -= lr * (m_hat / (torch.sqrt(v_hat) + eps) + weight_decay * p)
```

```python
# PyTorch内置AdamW
optimizer = torch.optim.AdamW(
    model.parameters(),
    lr=0.001,
    weight_decay=0.01     # 真正的权重衰减
)
```

**AdamW vs Adam + L2正则化**：

| 特性 | Adam + L2 | AdamW |
|------|-----------|-------|
| 正则化项 | $\lambda \sum \theta_i^2$ 加到损失 | 直接应用于参数更新 |
| 实际效果 | 取决于梯度统计量 | 与参数尺度无关 |
| 推荐场景 | 简单场景 | 现代大模型训练 |

## 学习率调度

学习率是深度学习最重要的超参数之一。学习率调度（Learning Rate Scheduling）可以在训练过程中动态调整学习率。

### 常见调度策略

#### Step Decay（阶梯衰减）

每隔固定epoch降低学习率：

```python
def step_decay(epoch, initial_lr, drop_rate=0.5, epochs_drop=10):
    """每隔epochs_drop个epoch，学习率乘以drop_rate"""
    return initial_lr * (drop_rate ** (epoch // epochs_drop))
```

```python
scheduler = torch.optim.lr_scheduler.StepLR(
    optimizer,
    step_size=30,      # 每30个epoch
    gamma=0.1           # 学习率乘以0.1
)
```

#### Cosine Annealing（余弦退火）

学习率按照余弦曲线从最大值降到最小值：

$$
\eta_t = \eta_{\min} + \frac{1}{2}(\eta_{\max} - \eta_{\min})\left(1 + \cos\left(\frac{t\pi}{T}\right)\right)
$$

```python
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(
    optimizer,
    T_max=100,         # 最大迭代次数
    eta_min=1e-6       # 最小学习率
)
```

#### Warmup + Cosine

训练初期逐渐增加学习率，然后按余弦曲线衰减：

```python
class WarmupCosineScheduler:
    def __init__(self, optimizer, warmup_epochs, total_epochs, 
                 base_lr, min_lr=0):
        self.optimizer = optimizer
        self.warmup_epochs = warmup_epochs
        self.total_epochs = total_epochs
        self.base_lr = base_lr
        self.min_lr = min_lr
    
    def step(self, epoch):
        if epoch < self.warmup_epochs:
            # 线性warmup
            lr = self.base_lr * (epoch + 1) / self.warmup_epochs
        else:
            # 余弦退火
            progress = (epoch - self.warmup_epochs) / \
                       (self.total_epochs - self.warmup_epochs)
            lr = self.min_lr + 0.5 * (self.base_lr - self.min_lr) * \
                 (1 + np.cos(np.pi * progress))
        
        for param_group in self.optimizer.param_groups:
            param_group['lr'] = lr
```

```python
# PyTorch内置
scheduler = torch.optim.lr_scheduler.SequentialLR(
    optimizer,
    schedulers=[
        torch.optim.lr_scheduler.LinearLR(
            optimizer, start_factor=0.1, total_iters=5
        ),  # Warmup
        torch.optim.lr_scheduler.CosineAnnealingLR(
            optimizer, T_max=95
        )   # Cosine decay
    ],
    milestones=[5]
)
```

#### OneCycleLR

由Leslie Smith提出，训练分为三个阶段：[^2]
1. **Warmup阶段**：学习率从很低线性增加到最大学习率
2. **Annealing阶段**：学习率从最大值退火到最小值
3. **微调阶段**：最终学习率略微上升

```python
scheduler = torch.optim.lr_scheduler.OneCycleLR(
    optimizer,
    max_lr=0.01,           # 最大学习率
    epochs=100,
    steps_per_epoch=len(train_loader),
    pct_start=0.3,         # warmup占前30%
    anneal_strategy='cos'  # 余弦退火
)
```

## 优化器比较与实践

### 理论与实践的差异

研究[^3]发现，自适应优化器（如Adam）在测试集上的泛化能力通常不如SGD+Momentum：

| 特性 | SGD+Momentum | Adam |
|------|-------------|------|
| 收敛速度 | 较慢 | 快 |
| 最终性能 | 通常更好 | 有时更差 |
| 调参难度 | 需要仔细调学习率 | 更鲁棒 |
| 泛化能力 | 找到更平坦的极小值 | 找到更尖锐的极小值 |

**原因分析**：
- Adam的自适应学习率使得在尖锐极小值处也能稳定收敛
- SGD的固定/慢速学习率天然倾向于平坦极小值
- 平坦极小值通常泛化能力更强

### 现代训练实践

#### 预训练大模型

```python
# BERT/Transformer预训练常用配置
optimizer = torch.optim.AdamW(
    model.parameters(),
    lr=1e-4,
    betas=(0.9, 0.999),
    weight_decay=0.01
)

scheduler = torch.optim.lr_scheduler.LinearLR(
    optimizer, start_factor=0.1, total_iters=num_warmup_steps
)
scheduler = torch.optim.lr_scheduler.ChainedScheduler([
    warmup_scheduler,
    torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=num_training_steps)
])
```

#### ViT/视觉模型

```python
# 常用配置
optimizer = torch.optim.AdamW(
    model.parameters(),
    lr=3e-3,           # ViT通常用较大学习率
    weight_decay=0.05, # 强权重衰减
    betas=(0.9, 0.999)
)

scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(
    optimizer, T_max=num_epochs
)
```

### 优化器选择指南

```
┌─────────────────────────────────────────────────────────────┐
│                     优化器选择决策树                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  快速原型/小数据集                                           │
│      ↓                                                      │
│  Adam/AdamW（lr=1e-3 ~ 1e-4）                               │
│                                                             │
│  大规模预训练/生成模型                                       │
│      ↓                                                      │
│  AdamW + Warmup + Cosine（lr=1e-4 ~ 3e-4）                  │
│                                                             │
│  追求最佳最终性能                                            │
│      ↓                                                      │
│  SGD + Momentum + Cosine Annealing                         │
│      (需要仔细调学习率和动量)                                │
│                                                             │
│  稀疏特征学习                                               │
│      ↓                                                      │
│  AdaGrad / Sparse Adam                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 训练技巧

1. **梯度裁剪**：防止梯度爆炸
   ```python
   torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
   ```

2. **混合精度训练**：加速 + 节省显存
   ```python
   scaler = torch.cuda.amp.GradScaler()
   with torch.cuda.amp.autocast():
       outputs = model(inputs)
       loss = criterion(outputs, targets)
   scaler.scale(loss).backward()
   scaler.step(optimizer)
   scaler.update()
   ```

3. **梯度累积**：模拟大批量训练
   ```python
   accumulation_steps = 4
   for i, (inputs, targets) in enumerate(dataloader):
       outputs = model(inputs)
       loss = criterion(outputs, targets) / accumulation_steps
       loss.backward()
       if (i + 1) % accumulation_steps == 0:
           optimizer.step()
           optimizer.zero_grad()
   ```

## 优化器发展前沿

### 2024-2025年新进展

#### Muon

Muon优化器由Nicholas J. F. Rose提出，专门针对神经网络训练设计：

```python
# Muon核心更新
def muon_step(params, grads, m, epoch):
    for p, g in zip(params, grads):
        # 牛顿法风格的更新
        p.data -= lr * (g @ M(grads)) / (torch.norm(g) + eps)
```

Muon在某些架构上展现出比Adam更快的收敛速度。

#### Sophia

Sophia（Second-order Clipped Stochastic Optimization）使用轻量级Hessian估计进行梯度裁剪，比Adam更快。

#### AdamW变体

- **AdamP**：在AdamW基础上添加逐参数投影，防止梯度爆炸
- **LAMB**：Layer-wise Adaptive Moments and Bias correction，适合大batch训练

## 参考

[^1]: Loshchilov & Hutter, "Decoupled Weight Decay Regularization", ICLR 2019
[^2]: Smith, "Super-Convergence: Very Fast Training of Neural Networks Using Large Learning Rates", arXiv 2018
[^3]: Wilson et al., "The Marginal Value of Adaptive Gradient Methods in Machine Learning", NeurIPS 2017
[^4]: He et al., "Delving Deep into Rectifiers: Surpassing Human-Level Performance on ImageNet Classification", ICCV 2015
[^5]: Loshchilov & Hutter, "SGDR: Stochastic Gradient Descent with Warm Restarts", ICLR 2017
