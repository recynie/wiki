---
title: Score Matching与SDE视角下的扩散模型
date: 2026-04-21
description: Score Matching理论基础、SDE统一框架、与DDPM的联系
tags:
  - score-matching
  - sde
  - diffusion-model
  - score-based
draft: false
permalink:
---

## 概述

Score Matching提供了一种无需计算归一化常数即可学习能量模型的方法，与扩散模型有着深刻的数学联系。Song等人(2021)将这一理论扩展到随机微分方程（SDE）框架，统一了离散扩散模型的各种变体。[^1]

---

## Score Function

### 定义

对于概率密度 $p(\mathbf{x})$，其**Score函数**定义为对数密度的梯度：

$$\nabla_{\mathbf{x}} \log p(\mathbf{x})$$

几何意义上，Score函数指向概率密度增长最快的方向：

$$\mathbf{s}(\mathbf{x}) = \nabla_{\mathbf{x}} \log p(\mathbf{x}) = \frac{\nabla_{\mathbf{x}} p(\mathbf{x})}{p(\mathbf{x})}$$

### 物理直觉

考虑一维情况，Score函数有直观解释：

$$p(x) = \frac{e^{-V(x)}}{Z}$$

其中 $V(x)$ 是能量函数，$Z$ 是配分函数。则：

$$\nabla \log p(x) = -\nabla V(x)$$

即Score函数是能量梯度的负方向，指向低能量（高概率）区域。

### 与朗之万动力学的联系

已知Score函数后，可通过**朗之万动力学（Langevin Dynamics）**采样：

$$\mathbf{x}_{t+1} = \mathbf{x}_t + \eta \nabla \log p(\mathbf{x}_t) + \sqrt{2\eta}\boldsymbol{\epsilon}_t$$

其中 $\boldsymbol{\epsilon}_t \sim \mathcal{N}(\mathbf{0}, \mathbf{I})$，$\eta$ 是步长。当 $\eta \to 0$ 且 $t \to \infty$ 时，$\mathbf{x}_t$ 收敛到真实分布 $p(\mathbf{x})$。

---

## Score Matching

### 问题设定

能量模型定义为：

$$p_\theta(\mathbf{x}) = \frac{e^{-E_\theta(\mathbf{x})}}{Z_\theta}$$

其中 $E_\theta(\mathbf{x})$ 是能量函数，$Z_\theta = \int e^{-E_\theta(\mathbf{x})} d\mathbf{x}$ 是** intractable**的配分函数。

直接最大化似然 $\log p_\theta(\mathbf{x})$ 需要计算 $Z_\theta$，这通常不可行。

### 显式Score Matching (ESM)

**目标函数**直接最小化预测Score与真实Score的差异：

$$\mathcal{J}_{ESM}(\theta) = \frac{1}{2} \mathbb{E}_{p_{\text{data}}}\left[\|\mathbf{s}_\theta(\mathbf{x}) - \nabla \log p_{\text{data}}(\mathbf{x})\|^2\right]$$

展开后：

$$\mathcal{J}_{ESM} = \mathbb{E}_{p_{\text{data}}}\left[\frac{1}{2}\|\mathbf{s}_\theta(\mathbf{x})\|^2 + \text{tr}(\nabla \mathbf{s}_\theta(\mathbf{x}))\right] + C$$

其中 $C$ 是与 $\theta$ 无关的常数。

**问题**：需要计算 $\text{tr}(\nabla \mathbf{s}_\theta)$（Hessian的迹），对于神经网络计算量巨大。

### 切片Score Matching (SSM)

**核心思想**：用随机投影降低计算复杂度。

对于随机向量 $\mathbf{v} \sim \mathcal{N}(\mathbf{0}, \mathbf{I})$：

$$\mathbf{v}^T \nabla \mathbf{s}_\theta(\mathbf{x}) \mathbf{v} = \text{tr}(\nabla \mathbf{s}_\theta(\mathbf{x}))$$

切片Score Matching目标：

$$\mathcal{J}_{SSM}(\theta) = \mathbb{E}_{p_{\text{data}}, \mathbf{v}}\left[\frac{1}{2}(\mathbf{v}^T \mathbf{s}_\theta(\mathbf{x}))^2 + \mathbf{v}^T \nabla \mathbf{s}_\theta(\mathbf{x}) \mathbf{v}\right]$$

只需计算Jacobian-vector products，避免显式计算Hessian。

### 去噪Score Matching (DSM)

**关键洞察**：对于特定形式的噪声分布，可以得到Score的闭式表达式。

设 $q_\sigma(\tilde{\mathbf{x}} \mid \mathbf{x}) = \mathcal{N}(\tilde{\mathbf{x}}; \mathbf{x}, \sigma^2 \mathbf{I})$，则：

$$\nabla_{\tilde{\mathbf{x}}} \log q_\sigma(\tilde{\mathbf{x}} \mid \mathbf{x}) = \frac{\mathbf{x} - \tilde{\mathbf{x}}}{\sigma^2}$$

去噪Score Matching目标：

$$\mathcal{J}_{DSM}(\theta) = \mathbb{E}_{p_{\text{data}}, q_\sigma}\left[\frac{1}{2}\left\|\mathbf{s}_\theta(\tilde{\mathbf{x}}; \sigma) - \frac{\mathbf{x} - \tilde{\mathbf{x}}}{\sigma^2}\right\|^2\right]$$

这正是DDPM训练目标的数学基础。

---

## SDE统一框架

### 从离散到连续

DDPM的离散前向过程：
$$q(\mathbf{x}_t \mid \mathbf{x}_{t-1}) = \mathcal{N}(\mathbf{x}_t; \mathbf{x}_{t-1}, \beta_t \mathbf{I})$$

当步数 $T \to \infty$，步长 $\Delta t \to 0$ 时，离散过程收敛为连续SDE：

$$d\mathbf{x}_t = \mathbf{f}(\mathbf{x}_t, t)dt + g(t)d\mathbf{w}_t$$

其中 $\mathbf{w}_t$ 是维纳过程（Wiener Process）。

### 前向SDE

一般形式的前向SDE：

$$d\mathbf{x}_t = \underbrace{\mathbf{f}(\mathbf{x}_t, t)}_{\text{drift}} dt + \underbrace{g(t)}_{\text{diffusion}} d\mathbf{w}_t$$

| 类型 | 漂移项 $f(t)$ | 扩散项 $g(t)$ | 特点 |
|------|---------------|---------------|------|
| **VP** (Variance Preserving) | $-\frac{1}{2}\beta(t)\mathbf{x}_t$ | $\sqrt{\beta(t)}$ | $q_t(\mathbf{x}_t)$ 方差保持 |
| **VE** (Variance Exploding) | $0$ | $\sqrt{\frac{d\sigma^2(t)}{dt}}$ | 方差随时间爆炸 |
| **subVP** | $-\frac{1}{2}\beta(t)\mathbf{x}_t$ | $\sqrt{\beta(t)(1-e^{-2\int_0^t \beta(s)ds})}$ | VP的改进 |

### 逆向SDE

从时间 $T$ 到 $0$ 反向求解，需要知道反向时间的SDE：

$$d\mathbf{x}_t = \left[-\mathbf{f}(\mathbf{x}_t, t) + g(t)^2 \nabla \log p_t(\mathbf{x}_t)\right]dt + g(t)d\bar{\mathbf{w}}_t$$

其中 $\bar{\mathbf{w}}_t$ 是反向维纳过程，$\nabla \log p_t(\mathbf{x}_t)$ 正是我们需要学习的Score函数。

### 统一视角

```
离散DDPM ←─────────────→ 连续SDE
    ↓                        ↓
预测噪声 ε_θ          预测Score s_θ
    ↓                        ↓
等价变换 ───────────→ 完全等价
```

**核心结论**：学习预测噪声 $\boldsymbol{\epsilon}_\theta(\mathbf{x}_t, t)$ 等价于学习Score函数 $\nabla \log p_t(\mathbf{x}_t)$：

$$\boldsymbol{\epsilon}_\theta(\mathbf{x}_t, t) = -\sqrt{1-\bar{\alpha}_t} \cdot \mathbf{s}_\theta(\mathbf{x}_t, t)$$

---

## SDE求解与采样

### Euler-Maruyama方法

最简单的SDE数值求解：

```python
def euler_maruyama(score_fn, xT, T, dt=1e-5, device='cuda'):
    """Euler-Maruyama求解逆向SDE"""
    x = xT
    for t in reversed(range(0, int(T), int(1/dt))):
        t_norm = t / T
        drift = ...  # 漂移项
        diffusion = ...  # 扩散项
        x = x + drift * dt + diffusion * np.sqrt(dt) * np.random.randn()
    return x
```

### Predictor-Corrector方法

**Predictor**：使用数值SDE求解器预测下一步
**Corrector**：使用朗之万动力学校正

```python
def predictor_corrector_sampling(score_fn, xT, T, n_steps=1000, corrector_steps=10, eps=1e-5):
    """Predictor-Corrector采样"""
    dt = T / n_steps
    x = xT
    
    for i in range(n_steps):
        t = T - i * dt
        
        # Predictor: Euler-Maruyama
        drift = -0.5 * beta(t) * x - beta(t) * score_fn(x, t)
        diffusion = np.sqrt(beta(t))
        x_pred = x + drift * dt + diffusion * np.sqrt(dt) * np.random.randn()
        
        # Corrector: 朗之万动力学
        for _ in range(corrector_steps):
            grad = score_fn(x_pred, t - dt)
            x_pred = x_pred + corrector_lr * grad + np.sqrt(2 * corrector_lr) * np.random.randn()
        
        x = x_pred
    
    return x
```

### 概率流ODE

SDE对应的ODE形式（确定性采样）：

$$\frac{d\mathbf{x}_t}{dt} = -\frac{1}{2}\beta(t)\mathbf{x}_t - \beta(t)\nabla \log p_t(\mathbf{x}_t)$$

使用ODE求解器（如Runge-Kutta）可实现**确定性**采样，路径可逆。

---

## 理论分析

### 收敛性保证

对于VP-SDE，可以证明以下收敛性：

**定理**：设Score估计误差为 $\epsilon$，则使用Euler-Maruyama采样的误差满足：

$$\mathbb{E}[\|\mathbf{x}_0 - \hat{\mathbf{x}}_0\|] \leq O(\epsilon^{1/2})$$

当 $T \to \infty$ 且步长 $\Delta t \to 0$ 时，误差趋于零。

### 与DDPM的关系

| 方面 | DDPM | Score SDE |
|------|------|-----------|
| 时间 | 离散 $t = 1, 2, ..., T$ | 连续 $t \in [0, T]$ |
| 采样 | 固定步数 $T$ | 可变步长 |
| 理论基础 | 变分推断 | Score Matching + SDE |
| 加速 | DDIM | ODE/SDE求解器 |

### 统一损失函数

从Score Matching视角，DDPM的简化损失可写为：

$$\mathcal{L} = \mathbb{E}_t\left[\lambda(t) \cdot \mathbb{E}_{\mathbf{x}_0, \mathbf{x}_t}\left[\left\|\mathbf{s}_\theta(\mathbf{x}_t, t) - \nabla \log q_t(\mathbf{x}_t \mid \mathbf{x}_0)\right\|^2\right]\right]$$

其中权重 $\lambda(t)$ 与噪声调度相关。

---

## 实践技巧

### 条件Score估计

对于条件生成 $p(\mathbf{x} \mid y)$，Classifier-Free Guidance扩展：

$$\tilde{\mathbf{s}}(\mathbf{x}, t, y) = (1 + w)\mathbf{s}_\theta(\mathbf{x}, t, y) - w \cdot \mathbf{s}_\theta(\mathbf{x}, t, \emptyset)$$

### 混合时间步训练

为提高低噪声时间步的Score估计质量：

```python
def mixed_time_loss(model, x0, t_strategy='uniform'):
    # 策略1: 均匀采样
    if t_strategy == 'uniform':
        t = torch.randint(0, T, (batch_size,))
    
    # 策略2: 重要性采样（低t权重更高）
    elif t_strategy == 'importance':
        snr = alphas_bar / (1 - alphas_bar)
        weights = snr / snr.sum()
        t = torch.multinomial(weights, batch_size, replacement=True)
    
    # 策略3: 最小化SNR损失
    elif t_strategy == 'snr':
        t = torch.randint(1, T, (batch_size,))  # 跳过t=0
        snr_t = alphas_bar[t] / (1 - alphas_bar[t])
        loss = snr_t * ((1 + alphas_bar[t]) * eps_pred - eps_true).square()
```

### 架构选择

- **U-Net + Attention**：标准选择，适合图像
- **Transformer**：DiT架构，更好的可扩展性
- **EDM框架**：Karras等人的统一实现框架

---

## 代码实现

### 完整Score SDE框架

```python
import torch
import torch.nn as nn
import numpy as np

class ScoreSDE(nn.Module):
    """基于SDE的统一Score模型"""
    
    def __init__(self, sde_type='vp', beta_min=0.1, beta_max=20.0):
        super().__init__()
        self.sde_type = sde_type
        self.beta_min = beta_min
        self.beta_max = beta_max
        
        # 神经网络：预测Score
        self.score_net = ScoreNetwork()
    
    def beta(self, t):
        """噪声调度"""
        return self.beta_min + t * (self.beta_max - self.beta_min)
    
    def drift(self, x, t):
        """漂移项 f(x, t)"""
        if self.sde_type == 'vp':
            return -0.5 * self.beta(t) * x
        elif self.sde_type == 've':
            return torch.zeros_like(x)
    
    def diffusion(self, t):
        """扩散项 g(t)"""
        if self.sde_type == 'vp':
            return np.sqrt(self.beta(t))
        elif self.sde_type == 've':
            return self.beta(t)
    
    def forward(self, x, t):
        """预测Score函数"""
        return self.score_net(x, t)
    
    @torch.no_grad()
    def euler_maruyama_sampling(self, shape, n_steps=1000, device='cuda'):
        """Euler-Maruyama采样"""
        x = torch.randn(shape, device=device)
        dt = 1.0 / n_steps
        
        for i in range(n_steps):
            t = 1.0 - i * dt
            t_tensor = torch.full((shape[0],), t, device=device)
            
            score = self.score_net(x, t_tensor)
            drift = self.drift(x, t)
            diff = self.diffusion(t)
            
            # 逆向SDE
            x = x - (drift - diff**2 * score) * dt + diff * np.sqrt(dt) * torch.randn_like(x)
        
        return x
    
    @torch.no_grad()
    def ode_sampling(self, shape, n_steps=1000, device='cuda'):
        """概率流ODE采样（确定性）"""
        x = torch.randn(shape, device=device)
        dt = 1.0 / n_steps
        
        for i in range(n_steps):
            t = 1.0 - i * dt
            t_tensor = torch.full((shape[0],), t, device=device)
            
            score = self.score_net(x, t_tensor)
            drift = self.drift(x, t)
            diff = self.diffusion(t)
            
            # ODE
            x = x - (drift - 0.5 * diff**2 * score) * dt
        
        return x

class ScoreNetwork(nn.Module):
    """Score估计网络"""
    
    def __init__(self, dim=64, time_dim=128):
        super().__init__()
        self.time_mlp = SinusoidalPosEmb(time_dim)
        
        # U-Net风格编码器
        self.enc1 = nn.Sequential(
            nn.Conv2d(3, dim, 3, padding=1),
            nn.GroupNorm(8, dim),
            nn.SiLU()
        )
        self.enc2 = nn.Sequential(
            nn.Conv2d(dim, dim*2, 3, stride=2, padding=1),
            nn.GroupNorm(8, dim*2),
            nn.SiLU()
        )
        
        # 中间层
        self.mid = nn.Sequential(
            nn.Conv2d(dim*2, dim*2, 3, padding=1),
            nn.GroupNorm(8, dim*2),
            nn.SiLU()
        )
        
        # 解码器
        self.dec2 = nn.Sequential(
            nn.ConvTranspose2d(dim*2, dim, 4, stride=2, padding=1),
            nn.GroupNorm(8, dim),
            nn.SiLU()
        )
        self.dec1 = nn.Sequential(
            nn.Conv2d(dim, dim, 3, padding=1),
            nn.GroupNorm(8, dim),
            nn.SiLU(),
            nn.Conv2d(dim, 3, 3, padding=1)
        )
    
    def forward(self, x, t):
        t_emb = self.time_mlp(t)
        
        h1 = self.enc1(x)
        h2 = self.enc2(h1)
        h_mid = self.mid(h2)
        h2_out = self.dec2(h_mid)
        out = self.dec1(h2_out + h1)
        
        # 返回Score（而非噪声）
        return out

class SinusoidalPosEmb(nn.Module):
    def __init__(self, dim):
        super().__init__()
        self.dim = dim
    
    def forward(self, t):
        half = self.dim // 2
        emb = torch.log(torch.tensor(10000.0)) / (half - 1)
        emb = torch.exp(torch.arange(half, device=t.device) * -emb)
        emb = t[:, None] * emb[None, :]
        emb = torch.cat([emb.sin(), emb.cos()], dim=-1)
        return emb
```

---

## 延伸阅读

### 重要论文

| 论文 | 年份 | 贡献 |
|------|------|------|
| [Score Matching](https://www.jmlr.org/papers/volume12/hyvarinen12a/hyvarinen12a.pdf) | 2005 | Score Matching基础理论 |
| [NCSN](https://arxiv.org/abs/1907.05600) | 2019 | 噪声条件Score网络 |
| [Score SDE](https://arxiv.org/abs/2011.13456) | 2021 | SDE统一框架 |
| [EDM](https://arxiv.org/abs/2206.00364) | 2022 | 改进的训练与采样框架 |

### 学习资源

- [Score-Based Generative Modeling through SDE (NeurIPS 2021 Talk)](https://slideslive.com/38952166/scorebased-generative-modeling-through-stochastic-differential-equations)
- [Yang Song's Blog](https://yang-song.net/) - Score Matching的详细教程

---

## 参考资料

[^1]: Song, Y., Sohl-Dickstein, J., Kingma, D. P., Kumar, A., Ermon, S., & Poole, B. (2021). Score-Based Generative Modeling through Stochastic Differential Equations. *ICLR 2021*. https://arxiv.org/abs/2011.13456

[^2]: Hyvärinen, A., & Dayan, P. (2005). Estimation of non-normalized statistical models by score matching. *JMLR*, 6(4).

[^3]: Song, Y., & Ermon, S. (2019). Generative modeling by estimating gradients of the data distribution. *NeurIPS 2019*.
