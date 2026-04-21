---
title: 扩散模型（Diffusion Model）
date: 2026-04-21
description: 扩散模型（DDPM）基础理论：前向过程、反向过程、训练目标与采样算法
tags:
  - diffusion-model
  - generative-model
  - ddpm
  - score-based
draft: false
permalink:
---

## 概述

扩散模型（Diffusion Model）是一类基于马尔可夫链的生成模型，通过逐步添加噪声（正向过程）和学习逆过程（逆向过程）来生成数据。2020年Ho等人提出DDPM（Denoising Diffusion Probabilistic Models）后，扩散模型在图像生成、音频合成、分子设计等领域取得突破性进展。[^1]

### 生成式模型家族对比

| 模型 | 隐变量 | 训练目标 | 采样速度 | 生成质量 |
|------|--------|----------|----------|----------|
| **VAE** | 连续 | ELBO | 快 | 模糊 |
| **GAN** | 无 | 对抗训练 | 快 | 锐利（模式塌陷） |
| **Flow** | 可逆 | 负对数似然 | 快 | 精确但受限 |
| **Diffusion** | 多步渐进 | 变分下界/去噪 | 慢（但可加速） | 高质量、多样 |

扩散模型的核心优势在于：
- **稳定的训练**：无需对抗训练，避免模式塌陷
- **统一的似然优化**：训练目标为精确的对数似然下界
- **可组合性**：各步骤独立，可灵活设计网络结构

---

## 前向过程（Forward Process）

前向过程 $q(\mathbf{x}_{1:T} \mid \mathbf{x}_0)$ 是一个预先定义的马可夫链，逐步向数据 $\mathbf{x}_0$ 添加高斯噪声，最终将分布转换为标准正态分布。

### 定义

$$q(\mathbf{x}_t \mid \mathbf{x}_{t-1}) = \mathcal{N}\left(\mathbf{x}_t; \sqrt{1-\beta_t}\mathbf{x}_{t-1}, \beta_t\mathbf{I}\right)$$

其中 $\{\beta_t\}_{t=1}^T$ 是噪声调度（noise schedule），通常随 $t$ 递增。

### 闭式解

由于高斯分布的可组合性，可以直接计算任意时间步 $t$ 的分布：

$$q(\mathbf{x}_t \mid \mathbf{x}_0) = \mathcal{N}\left(\mathbf{x}_t; \sqrt{\bar{\alpha}_t}\mathbf{x}_0, (1-\bar{\alpha}_t)\mathbf{I}\right)$$

其中：
$$\alpha_t = 1 - \beta_t, \quad \bar{\alpha}_t = \prod_{i=1}^t \alpha_i$$

**实用采样形式**：
$$\mathbf{x}_t = \sqrt{\bar{\alpha}_t}\mathbf{x}_0 + \sqrt{1-\bar{\alpha}_t}\boldsymbol{\epsilon}, \quad \boldsymbol{\epsilon} \sim \mathcal{N}(\mathbf{0}, \mathbf{I})$$

### 噪声调度

常见的噪声调度策略：

```python
import numpy as np

def linear_schedule(T, beta_start=1e-4, beta_end=0.02):
    """线性调度"""
    betas = np.linspace(beta_start, beta_end, T)
    alphas = 1 - betas
    alphas_bar = np.cumprod(alphas)
    return betas, alphas, alphas_bar

def cosine_schedule(T, s=0.008):
    """余弦调度（更平滑）"""
    t = np.arange(T + 1)
    alphas_bar = np.cos(((t / T) + s) / (1 + s) * np.pi / 2) ** 2
    alphas_bar = alphas_bar / alphas_bar[0]  # 归一化
    betas = 1 - (alphas_bar[1:] / alphas_bar[:-1])
    return np.clip(betas, 0, 0.999), 1 - betas, alphas_bar[:-1]

def quadratic_schedule(T, n=2):
    """二次调度"""
    t = np.arange(T)
    betas = np.linspace(0.0001 ** (1/n), 0.01 ** (1/n), T) ** n
    alphas = 1 - betas
    alphas_bar = np.cumprod(alphas)
    return betas, alphas, alphas_bar
```

---

## 反向过程（Reverse Process）

反向过程 $p_\theta(\mathbf{x}_{0:T})$ 是学习的马尔可夫链，从纯噪声 $\mathbf{x}_T \sim \mathcal{N}(\mathbf{0}, \mathbf{I})$ 开始，逐步去噪生成数据。

### 定义

$$p_\theta(\mathbf{x}_{t-1} \mid \mathbf{x}_t) = \mathcal{N}\left(\mathbf{x}_{t-1}; \boldsymbol{\mu}_\theta(\mathbf{x}_t, t), \boldsymbol{\Sigma}_\theta(\mathbf{x}_t, t)\right)$$

### 重参数化

DDPM使用**重参数化技巧**简化逆向分布。给定 $q(\mathbf{x}_t \mid \mathbf{x}_0)$：

$$\mathbf{x}_t(\mathbf{x}_0, \boldsymbol{\epsilon}) = \sqrt{\bar{\alpha}_t}\mathbf{x}_0 + \sqrt{1-\bar{\alpha}_t}\boldsymbol{\epsilon}$$

反向过程均值可表示为：

$$\boldsymbol{\mu}_t(\mathbf{x}_t, \mathbf{x}_0) = \frac{1}{\sqrt{\alpha_t}}\left(\mathbf{x}_t - \frac{1-\alpha_t}{\sqrt{1-\bar{\alpha}_t}}\boldsymbol{\epsilon}\right)$$

### 简化参数化

DDPM的核心洞察：直接预测噪声 $\boldsymbol{\epsilon}_\theta(\mathbf{x}_t, t)$，而非预测均值或数据：

$$\hat{\mathbf{x}}_0 = \frac{1}{\sqrt{\bar{\alpha}_t}}\left(\mathbf{x}_t - \sqrt{1-\bar{\alpha}_t}\boldsymbol{\epsilon}_\theta(\mathbf{x}_t, t)\right)$$

$$\boldsymbol{\mu}_\theta(\mathbf{x}_t, t) = \frac{1}{\sqrt{\alpha_t}}\left(\mathbf{x}_t - \frac{\beta_t}{\sqrt{1-\bar{\alpha}_t}}\boldsymbol{\epsilon}_\theta(\mathbf{x}_t, t)\right)$$

---

## 训练目标

### 变分下界（VLB）

扩散模型的训练目标为负对数似然的变分下界：

$$-\log p_\theta(\mathbf{x}_0) \leq \underbrace{D_{KL}(q(\mathbf{x}_T \mid \mathbf{x}_0) \parallel p(\mathbf{x}_T))}_{L_T} + \sum_{t=1}^{T-1} \underbrace{D_{KL}(q(\mathbf{x}_t \mid \mathbf{x}_{t+1}, \mathbf{x}_0) \parallel p_\theta(\mathbf{x}_t \mid \mathbf{x}_{t+1}))}_{L_t} + \underbrace{-\log p_\theta(\mathbf{x}_0 \mid \mathbf{x}_1)}_{L_0}$$

### 简化损失

DDPM证明VLB可简化为简单的MSE损失：

$$L_{\text{simple}} = \mathbb{E}_{t, \mathbf{x}_0, \boldsymbol{\epsilon}}\left[\|\boldsymbol{\epsilon} - \boldsymbol{\epsilon}_\theta(\mathbf{x}_t, t)\|^2\right]$$

其中 $t \sim \mathcal{U}(1, T)$。

### 不同预测目标

| 预测目标 | 表达式 | 特点 |
|----------|--------|------|
| **噪声预测** | $\boldsymbol{\epsilon}_\theta$ | DDPM默认，简单有效 |
| **数据预测** | $\mathbf{x}_\theta$ | 直接，但训练不稳定 |
| **速度预测** | $\mathbf{v}_\theta = \alpha_t\boldsymbol{\epsilon} - \beta_t\mathbf{x}_0$ | 平衡方案 |

### SNT（信噪比）视角

Kingma等人从信噪比角度分析损失函数：

$$L_{\text{snr}}(\mathbf{x}_0) = \mathbb{E}_t\left[\text{SNR}(t) \cdot \|\mathbf{x}_0 - \hat{\mathbf{x}}_0\|^2\right]$$

其中 $\text{SNR}(t) = \frac{\alpha_t^2}{1-\alpha_t^2}$。

---

## 采样算法

### DDPM采样

标准DDPM采样需要 $T$ 步迭代：

```python
def ddpm_sampling(model, T, betas, device='cuda'):
    """DDPM反向采样"""
    alphas = 1 - betas
    alphas_bar = np.cumprod(alphas)
    
    # 从纯噪声开始
    x_t = torch.randn(1, 3, 64, 64).to(device)
    
    for t in reversed(range(T)):
        t_tensor = torch.full((1,), t, device=device, dtype=torch.long)
        
        # 预测噪声
        eps = model(x_t, t_tensor)
        
        # 计算均值
        mean = (x_t - betas[t] / np.sqrt(1 - alphas_bar[t]) * eps) / np.sqrt(alphas[t])
        
        # 添加噪声（最后一步除外）
        if t > 0:
            noise = torch.randn_like(x_t)
            x_t = mean + np.sqrt(betas[t]) * noise
        else:
            x_t = mean
    
    return x_t
```

### DDIM加速采样

DDIM（Denoising Diffusion Implicit Models）通过调整噪声调度实现更少的采样步数：

$$p_{\theta,\sigma}^{(c)}(\mathbf{x}_{t-1} \mid \mathbf{x}_t) = \boldsymbol{\mu}_\theta^{(c)}(\mathbf{x}_t, t) + \sigma_t \boldsymbol{\epsilon}$$

```python
def ddim_sampling(model, T, eta=0.0, skip=10):
    """DDIM加速采样"""
    # ...
    for t in list(range(1, T + 1, skip))[::-1]:
        t_prev = max(1, t - skip)
        # 使用隐式采样
        pred_x0 = predict_x0(model, x_t, t)
        pred_eps = (x_t - np.sqrt(alphas_bar[t]) * pred_x0) / np.sqrt(1 - alphas_bar[t])
        
        # 非确定性采样
        var = eta * (1 - alphas_bar[t_prev]) / (1 - alphas_bar[t]) * (1 - alphas[t]/alphas_bar[t])
        x_t = np.sqrt(alphas_bar[t_prev]) * pred_x0 + np.sqrt(1 - alphas_bar[t_prev] - var) * pred_eps
        x_t += np.sqrt(var) * noise
```

### Classifier-Free Guidance

Classifier-Free Guidance (CFG) 通过无条件与条件预测的线性组合提升生成质量：

$$\tilde{\boldsymbol{\epsilon}}(x_t, t, c) = (1 + w)\boldsymbol{\epsilon}_\theta(x_t, t, c) - w \cdot \boldsymbol{\epsilon}_\theta(x_t, t, \emptyset)$$

其中 $w$ 是引导强度（通常 $w \in [5, 10]$），$\emptyset$ 表示无条件预测。

---

## 代码实现

### 完整DDPM模型

```python
import torch
import torch.nn as nn
import math

class SinusoidalPosEmb(nn.Module):
    """时间步位置编码"""
    def __init__(self, dim):
        super().__init__()
        self.dim = dim
    
    def forward(self, t):
        device = t.device
        half_dim = self.dim // 2
        emb = math.log(10000) / (half_dim - 1)
        emb = torch.exp(torch.arange(half_dim, device=device) * -emb)
        emb = t[:, None] * emb[None, :]
        emb = torch.cat((emb.sin(), emb.cos()), dim=-1)
        return emb

class ResBlock(nn.Module):
    """ResNet残差块"""
    def __init__(self, dim, time_dim):
        super().__init__()
        self.conv1 = nn.Conv2d(dim, dim, 3, padding=1)
        self.conv2 = nn.Conv2d(dim, dim, 3, padding=1)
        self.time_mlp = nn.Sequential(
            nn.SiLU(),
            nn.Linear(time_dim, dim * 2)
        )
        self.norm = nn.GroupNorm(8, dim)
    
    def forward(self, x, t):
        h = self.norm(x)
        h = self.conv1(h)
        
        t_emb = self.time_mlp(t)
        scale, shift = t_emb.chunk(2, dim=1)
        h = h * (1 + scale.unsqueeze(-1).unsqueeze(-1))
        h = h + shift.unsqueeze(-1).unsqueeze(-1)
        
        h = self.conv2(torch.nn.functional.silu(h))
        return h + x

class UNet(nn.Module):
    """U-Net噪声预测网络"""
    def __init__(self, dim=64, time_dim=128):
        super().__init__()
        self.time_mlp = SinusoidalPosEmb(time_dim)
        
        self.conv1 = nn.Conv2d(3, dim, 3, padding=1)
        self.down1 = nn.Sequential(ResBlock(dim, time_dim), ResBlock(dim, time_dim))
        self.downsample1 = nn.Conv2d(dim, dim * 2, 3, stride=2, padding=1)
        
        self.down2 = nn.Sequential(ResBlock(dim * 2, time_dim), ResBlock(dim * 2, time_dim))
        self.downsample2 = nn.Conv2d(dim * 2, dim * 4, 3, stride=2, padding=1)
        
        self.mid = nn.Sequential(ResBlock(dim * 4, time_dim), ResBlock(dim * 4, time_dim))
        
        self.upsample2 = nn.ConvTranspose2d(dim * 4, dim * 2, 4, stride=2, padding=1)
        self.up2 = nn.Sequential(ResBlock(dim * 4, time_dim), ResBlock(dim * 4, time_dim))
        
        self.upsample1 = nn.ConvTranspose2d(dim * 2, dim, 4, stride=2, padding=1)
        self.up1 = nn.Sequential(ResBlock(dim * 2, time_dim), ResBlock(dim * 2, time_dim))
        
        self.conv_out = nn.Conv2d(dim, 3, 3, padding=1)
    
    def forward(self, x, t):
        t_emb = self.time_mlp(t)
        
        x1 = self.conv1(x)
        x1 = self.down1(x1)
        x1_down = self.downsample1(x1)
        
        x2 = self.down2(x1_down)
        x2_down = self.downsample2(x2)
        
        x_mid = self.mid(x2_down)
        
        x2_up = self.upsample2(x_mid)
        x2_up = torch.cat([x2_up, x2], dim=1)
        x2_up = self.up2(x2_up)
        
        x1_up = self.upsample1(x2_up)
        x1_up = torch.cat([x1_up, x1], dim=1)
        x1_up = self.up1(x1_up)
        
        return self.conv_out(x1_up)

class DiffusionModel(nn.Module):
    """完整扩散模型"""
    def __init__(self, T=1000, beta_start=1e-4, beta_end=0.02):
        super().__init__()
        self.T = T
        self.network = UNet()
        
        # 注册buffers存储调度参数
        betas = torch.linspace(beta_start, beta_end, T)
        alphas = 1 - betas
        alphas_bar = torch.cumprod(alphas, dim=0)
        
        self.register_buffer('betas', betas)
        self.register_buffer('alphas', alphas)
        self.register_buffer('alphas_bar', alphas_bar)
    
    def forward_diffusion(self, x0, t):
        """前向过程：添加噪声"""
        eps = torch.randn_like(x0)
        xt = torch.sqrt(self.alphas_bar[t]) * x0 + torch.sqrt(1 - self.alphas_bar[t]) * eps
        return xt, eps
    
    def training_loss(self, x0):
        """训练损失"""
        batch_size = x0.shape[0]
        t = torch.randint(0, self.T, (batch_size,), device=x0.device)
        xt, eps = self.forward_diffusion(x0, t)
        eps_pred = self.network(xt, t)
        return (eps_pred - eps).square().mean()
    
    @torch.no_grad()
    def sampling(self, shape, cfg_scale=7.0):
        """无条件采样"""
        device = next(self.parameters()).device
        xt = torch.randn(shape, device=device)
        
        for t in reversed(range(self.T)):
            t_batch = torch.full((shape[0],), t, device=device)
            eps = self.network(xt, t_batch)
            
            # CFG（简化版：无条件）
            if cfg_scale > 1.0:
                eps_uncond = self.network(xt, t_batch)  # 实际中需要无条件预测
                eps = (1 + cfg_scale) * eps - cfg_scale * eps_uncond
            
            mean = (xt - self.betas[t] / torch.sqrt(1 - self.alphas_bar[t]) * eps) / torch.sqrt(self.alphas[t])
            if t > 0:
                xt = mean + torch.sqrt(self.betas[t]) * torch.randn_like(xt)
            else:
                xt = mean
        
        return xt
```

---

## 与其他生成模型的关系

### VAE视角

扩散模型可以视为无限深层的VAE，其中：
- 前向过程 = 变分编码器（固定）
- 反向过程 = 变分解码器（学习）
- $T \to \infty$ 时，$q(\mathbf{x}_T \mid \mathbf{x}_0) \to \mathcal{N}(\mathbf{0}, \mathbf{I})$

### Flow视角

当 $T \to \infty$ 且步长 $\Delta t \to 0$ 时，DDPM的前向过程退化为常微分方程（ODE），与可逆Flow模型统一：

$$\frac{d\mathbf{x}_t}{dt} = -\frac{1}{2}\beta(t)\mathbf{x}_t - \beta(t)\nabla_{\mathbf{x}}\log p_t(\mathbf{x})$$

### Score-Based视角

扩散模型的训练等价于学习Score函数：

$$\boldsymbol{\epsilon}_\theta(\mathbf{x}_t, t) \approx -\sqrt{1-\bar{\alpha}_t}\nabla_{\mathbf{x}}\log p_t(\mathbf{x}_t)$$

详见 [[score-matching-sde]]。

---

## 应用场景

### 图像生成

- **DALL-E 2/3**：基于CLIP引导的扩散模型
- **Stable Diffusion**：潜在空间扩散（Latent Diffusion）
- **Imagen**：级联扩散，超分辨率增强

### 视频生成

- **Sora**：基于Diffusion Transformer的长时间视频生成
- **VideoLDM**：时序一致的扩散模型

### 音频合成

- **AudioLM**：语音/音乐的扩散生成
- **DiffWave**：波形级音频扩散

### 科学应用

- **分子设计**：Drug Discovery中的分子生成
- **材料科学**：晶体结构生成

---

## 参考资料

[^1]: Ho, J., Jain, A., & Abbeel, P. (2020). Denoising Diffusion Probabilistic Models. *NeurIPS 2020*. https://arxiv.org/abs/2006.11239

[^2]: Ho, J., & Salimans, T. (2022). Classifier-Free Diffusion Guidance. *NeurIPS 2022 Workshop*. https://arxiv.org/abs/2207.12598

[^3]: Song, Y., Sohl-Dickstein, J., Kingma, D. P., Kumar, A., Ermon, S., & Poole, B. (2021). Score-Based Generative Modeling through Stochastic Differential Equations. *ICLR 2021*. https://arxiv.org/abs/2011.13456

[^4]: Nichol, A. Q., & Dhariwal, P. (2021). Improved Denoising Diffusion Probabilistic Models. *ICML 2021*. https://arxiv.org/abs/2102.09672
