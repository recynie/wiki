---
title: DDPM实现细节与Latent Diffusion Models
date: 2026-04-21
description: DDPM训练与采样实现、Classifier-Free Guidance、Latent Diffusion Model原理
tags:
  - ddpm
  - implementation
  - latent-diffusion
  - classifier-free-guidance
draft: false
permalink:
---

## 概述

本文介绍DDPM的实际实现细节，包括训练流程、采样策略、Classifier-Free Guidance技术，以及在实践中广泛应用的Latent Diffusion Model（LDM）架构。[^1]

---

## DDPM完整训练流程

### 伪代码

```python
# 算法1: DDPM训练

def trainDDPM(dataset, model, T=1000):
    """
    输入: 训练数据集, 噪声预测模型, 扩散步数T
    输出: 训练好的模型
    """
    
    # 1. 定义噪声调度
    beta = linear_beta_schedule(T)  # 线性调度
    
    # 2. 预计算系数
    alpha = 1 - beta
    alpha_bar = cumprod(alpha)  # 累积乘积
    
    optimizer = Adam(model.parameters())
    
    while training_continue:
        # 3. 从数据集采样
        x0 = sample_batch(dataset)
        
        # 4. 随机选择时间步
        t = uniform_sample(T)  # t ~ U{1, ..., T}
        
        # 5. 采样噪声
        eps = normal_sample_like(x0)
        
        # 6. 添加噪声到数据
        # x_t = sqrt(alpha_bar[t]) * x0 + sqrt(1 - alpha_bar[t]) * eps
        xt = sqrt(alpha_bar[t]) * x0 + sqrt(1 - alpha_bar[t]) * eps
        
        # 7. 预测噪声
        eps_theta = model(xt, t)
        
        # 8. 计算损失并更新
        loss = mse_loss(eps_theta, eps)
        optimizer.step()
    
    return model
```

### PyTorch实现

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torch.utils.data import DataLoader
from torchvision import datasets, transforms

class DDPM(nn.Module):
    """完整的DDPM实现"""
    
    def __init__(self, T=1000, beta_start=1e-4, beta_end=0.02, device='cuda'):
        super().__init__()
        self.T = T
        self.device = device
        
        # 噪声调度
        self.register_buffer('betas', self.linear_beta_schedule(T, beta_start, beta_end))
        self.register_buffer('alphas', 1. - self.betas)
        self.register_buffer('alphas_bar', torch.cumprod(self.alphas, dim=0))
        self.register_buffer('alphas_bar_prev', F.pad(self.alphas_bar[:-1], (1, 0), value=1.0))
        
        # 计算方差
        self.register_buffer('sqrt_alphas_bar', torch.sqrt(self.alphas_bar))
        self.register_buffer('sqrt_one_minus_alphas_bar', torch.sqrt(1. - self.alphas_bar))
        self.register_buffer('log_one_minus_alphas_bar', torch.log(1. - self.alphas_bar))
        self.register_buffer('sqrt_recip_alphas', torch.sqrt(1. / self.alphas))
        self.register_buffer('sqrt_recipm1_alphas_bar', torch.sqrt(1. / self.alphas_bar - 1))
        
        # 后验方差（用于采样）
        self.register_buffer('posterior_variance', 
            self.betas * (1. - self.alphas_bar_prev) / (1. - self.alphas_bar))
        self.register_buffer('posterior_log_variance_clipped',
            torch.log(torch.clamp(self.posterior_variance, min=1e-20)))
        self.register_buffer('posterior_mean_coef1',
            self.betas * torch.sqrt(self.alphas_bar_prev) / (1. - self.alphas_bar))
        self.register_buffer('posterior_mean_coef2',
            (1. - self.alphas_bar_prev) * torch.sqrt(self.alphas) / (1. - self.alphas_bar))
        
        # UNet模型
        self.model = UNet()
    
    @staticmethod
    def linear_beta_schedule(T, beta_start=1e-4, beta_end=0.02):
        return torch.linspace(beta_start, beta_end, T)
    
    @staticmethod
    def cosine_beta_schedule(T, s=0.008):
        """余弦调度：更平滑的噪声添加"""
        t = torch.arange(T + 1)
        alphas_bar = torch.cos(((t / T) + s) / (1 + s) * torch.pi / 2) ** 2
        alphas_bar = alphas_bar / alphas_bar[0]
        betas = 1 - (alphas_bar[1:] / alphas_bar[:-1])
        return torch.clamp(betas, 0.0001, 0.9999)
    
    def q_sample(self, x0, t, noise=None):
        """前向过程：添加噪声"""
        if noise is None:
            noise = torch.randn_like(x0)
        
        return (
            self.sqrt_alphas_bar[t][:, None, None, None] * x0 +
            self.sqrt_one_minus_alphas_bar[t][:, None, None, None] * noise
        ), noise
    
    def p_mean_variance(self, xt, t, clip_denoised=True):
        """逆向过程：计算均值和方差"""
        # 预测噪声
        eps_pred = self.model(xt, t)
        
        # 预测原始数据
        x0_pred = (
            xt - self.sqrt_one_minus_alphas_bar[t][:, None, None, None] * eps_pred
        ) / self.sqrt_alphas_bar[t][:, None, None, None]
        
        if clip_denoised:
            x0_pred = torch.clamp(x0_pred, -1, 1)
        
        model_mean = (
            self.posterior_mean_coef1[t][:, None, None, None] * x0_pred +
            self.posterior_mean_coef2[t][:, None, None, None] * xt
        )
        
        return model_mean, self.posterior_variance[t], x0_pred
    
    @torch.no_grad()
    def p_sample(self, xt, t, clip_denoised=True):
        """单步采样"""
        mean, variance, _ = self.p_mean_variance(xt, t, clip_denoised)
        
        noise = torch.randn_like(xt) if t > 0 else 0
        return mean + torch.sqrt(variance) * noise
    
    @torch.no_grad()
    def p_sample_loop(self, shape, cfg_scale=None):
        """完整采样循环"""
        device = self.device
        batch_size = shape[0]
        
        # 从纯噪声开始
        xt = torch.randn(shape, device=device)
        
        for t in reversed(range(self.T)):
            t_batch = torch.full((batch_size,), t, device=device, dtype=torch.long)
            
            if cfg_scale is not None and cfg_scale > 1.0:
                # Classifier-Free Guidance
                # 1. 有条件预测
                eps_cond = self.model(xt, t_batch)
                # 2. 无条件预测（使用空条件）
                eps_uncond = self.model(xt, t_batch)  # 实际需要模型支持条件输入
                # 3. CFG组合
                eps_pred = (1 + cfg_scale) * eps_cond - cfg_scale * eps_uncond
                
                # 手动计算下一步
                x0_pred = (
                    xt - self.sqrt_one_minus_alphas_bar[t][:, None, None, None] * eps_pred
                ) / self.sqrt_alphas_bar[t][:, None, None, None]
                x0_pred = torch.clamp(x0_pred, -1, 1)
                
                mean = (
                    self.posterior_mean_coef1[t][:, None, None, None] * x0_pred +
                    self.posterior_mean_coef2[t][:, None, None, None] * xt
                )
                
                if t > 0:
                    noise = torch.randn_like(xt)
                    xt = mean + torch.sqrt(self.posterior_variance[t]) * noise
                else:
                    xt = mean
            else:
                xt = self.p_sample(xt, t_batch)
        
        return xt
    
    def training_loss(self, x0):
        """计算训练损失"""
        batch_size = x0.shape[0]
        
        # 随机时间步
        t = torch.randint(0, self.T, (batch_size,), device=x0.device, dtype=torch.long)
        
        # 前向过程
        xt, noise = self.q_sample(x0, t)
        
        # 预测噪声
        noise_pred = self.model(xt, t)
        
        # MSE损失
        return F.mse_loss(noise_pred, noise)
```

---

## UNet架构实现

```python
import math
import torch
import torch.nn as nn
import torch.nn.functional as F

class Attention(nn.Module):
    """自注意力层"""
    def __init__(self, channels, num_heads=8):
        super().__init__()
        self.num_heads = num_heads
        self.channels = channels
        self.norm = nn.GroupNorm(32, channels)
        self.qkv = nn.Linear(channels, channels * 3)
        self.proj = nn.Linear(channels, channels)
    
    def forward(self, x):
        B, C, H, W = x.shape
        x_norm = self.norm(x)
        x_flat = x_norm.flatten(2).transpose(1, 2)
        
        qkv = self.qkv(x_flat).reshape(B, -1, 3, self.num_heads, C // self.num_heads)
        q, k, v = qkv[..., 0], qkv[..., 1], qkv[..., 2]
        
        attn = (q @ k.transpose(-2, -1)) / math.sqrt(C // self.num_heads)
        attn = attn.softmax(-1)
        
        out = (attn @ v).reshape(B, -1, C)
        out = self.proj(out).transpose(1, 2).reshape(B, C, H, W)
        return x + out

class ResBlock(nn.Module):
    """残差块"""
    def __init__(self, in_ch, out_ch, time_emb_dim, groups=32):
        super().__init__()
        self.conv1 = nn.Conv2d(in_ch, out_ch, 3, padding=1)
        self.conv2 = nn.Conv2d(out_ch, out_ch, 3, padding=1)
        self.norm1 = nn.GroupNorm(groups, in_ch)
        self.norm2 = nn.GroupNorm(groups, out_ch)
        self.act = nn.SiLU()
        
        # 时间嵌入MLP
        self.time_mlp = nn.Sequential(
            nn.SiLU(),
            nn.Linear(time_emb_dim, out_ch * 2)
        )
        
        # 跳跃连接
        self.skip = nn.Conv2d(in_ch, out_ch, 1) if in_ch != out_ch else nn.Identity()
    
    def forward(self, x, t_emb):
        h = self.norm1(x)
        h = self.act(h)
        h = self.conv1(h)
        
        # AdaGN风格的时间调制
        t = self.time_mlp(t_emb)
        scale, shift = t.chunk(2, dim=1)
        h = h * (scale[:, :, None, None] + 1) + shift[:, :, None, None]
        
        h = self.norm2(h)
        h = self.act(h)
        h = self.conv2(h)
        
        return h + self.skip(x)

class UNet(nn.Module):
    """DDPM使用的U-Net架构"""
    
    def __init__(self, in_channels=3, out_channels=3, base_channels=128, 
                 channel_mults=(1, 2, 4, 4), num_res_blocks=2, 
                 attention_resolutions=(4,), time_dim=256):
        super().__init__()
        
        # 时间嵌入
        self.time_mlp = nn.Sequential(
            SinusoidalEmbedding(time_dim),
            nn.Linear(time_dim, time_dim * 4),
            nn.SiLU(),
            nn.Linear(time_dim * 4, time_dim)
        )
        
        # 编码器
        self.conv_in = nn.Conv2d(in_channels, base_channels, 3, padding=1)
        
        channels = [base_channels]
        in_ch = base_channels
        for mult in channel_mults:
            out_ch = base_channels * mult
            for _ in range(num_res_blocks):
                self.append_res_block(in_ch, out_ch, time_dim)
                channels.append(out_ch)
                in_ch = out_ch
                if out_ch in attention_resolutions:
                    self.append_attention(out_ch)
            
            if mult != channel_mults[-1]:
                self.append_downsample(out_ch)
                channels.append(out_ch)
        
        # 中间层
        self.mid = nn.ModuleList([
            ResBlock(in_ch, in_ch, time_dim),
            Attention(in_ch) if in_ch in attention_resolutions else nn.Identity(),
            ResBlock(in_ch, in_ch, time_dim)
        ])
        
        # 解码器
        self.up_blocks = nn.ModuleList()
        for i, mult in enumerate(reversed(channel_mults)):
            out_ch = base_channels * mult
            for j in range(num_res_blocks + 1):
                self.up_blocks.append(ResBlock(in_ch + channels.pop(), out_ch, time_dim))
                in_ch = out_ch
                if out_ch in attention_resolutions:
                    self.up_blocks.append(Attention(out_ch))
            
            if i != len(channel_mults) - 1:
                self.up_blocks.append(Upsample(out_ch))
        
        self.conv_out = nn.Sequential(
            nn.GroupNorm(32, base_channels),
            nn.SiLU(),
            nn.Conv2d(base_channels, out_channels, 3, padding=1)
        )
    
    def append_res_block(self, in_ch, out_ch, time_dim):
        setattr(self, f'down_res_{in_ch}_{out_ch}', ResBlock(in_ch, out_ch, time_dim))
    
    def append_attention(self, channels):
        setattr(self, f'attention_{channels}', Attention(channels))
    
    def append_downsample(self, channels):
        setattr(self, f'downsample_{channels}', nn.Conv2d(channels, channels, 3, stride=2, padding=1))
    
    def forward(self, x, t):
        t_emb = self.time_mlp(t)
        
        # 下采样路径
        hs = [self.conv_in(x)]
        for module in self.down_blocks:
            if isinstance(module, ResBlock):
                h = module(hs[-1], t_emb)
            else:
                h = module(hs[-1])
            hs.append(h)
        
        # 中间层
        h = hs[-1]
        for module in self.mid:
            if isinstance(module, ResBlock):
                h = module(h, t_emb)
            else:
                h = module(h)
        
        # 上采样路径
        for module in self.up_blocks:
            if isinstance(module, ResBlock):
                h = module(torch.cat([h, hs.pop()], dim=1), t_emb)
            else:
                h = module(h)
        
        return self.conv_out(h)

class SinusoidalEmbedding(nn.Module):
    def __init__(self, dim):
        super().__init__()
        self.dim = dim
    
    def forward(self, t):
        device = t.device
        half_dim = self.dim // 2
        emb = math.log(10000) / (half_dim - 1)
        emb = torch.exp(torch.arange(half_dim, device=device) * -emb)
        emb = t[:, None] * emb[None, :]
        emb = torch.cat([emb.sin(), emb.cos()], dim=-1)
        return emb

class Upsample(nn.Module):
    def __init__(self, channels):
        super().__init__()
        self.conv = nn.ConvTranspose2d(channels, channels, 4, stride=2, padding=1)
    
    def forward(self, x):
        return self.conv(x)
```

---

## Classifier-Free Guidance

### 原理

Classifier-Free Guidance（CFG）通过组合条件和无条件预测来引导生成，无需训练单独的分类器。

**数学推导**：

已知条件Score：$\nabla \log p(\mathbf{x}_t \mid y)$

无条件Score：$\nabla \log p(\mathbf{x}_t)$

CFG Score：

$$\tilde{\nabla} \log p(\mathbf{x}_t \mid y) = (1 + w) \nabla \log p(\mathbf{x}_t \mid y) - w \nabla \log p(\mathbf{x}_t)$$

展开：

$$= \nabla \log p(\mathbf{x}_t) + w\left(\nabla \log p(\mathbf{x}_t \mid y) - \nabla \log p(\mathbf{x}_t)\right)$$

对于噪声预测（DDPM）：

$$\tilde{\boldsymbol{\epsilon}}(\mathbf{x}_t, y) = (1 + w)\boldsymbol{\epsilon}_\theta(\mathbf{x}_t, y) - w\boldsymbol{\epsilon}_\theta(\mathbf{x}_t, \emptyset)$$

### 实现技巧

```python
class CFGDiffusionModel(nn.Module):
    """支持Classifier-Free Guidance的扩散模型"""
    
    def __init__(self, model, p_uncond=0.1):
        super().__init__()
        self.model = model  # 接受条件输入的模型
        self.p_uncond = p_uncond
        self.model.requires_grad_(False)  # 模型参数冻结
    
    def forward(self, x0, y=None, cfg_scale=7.0):
        """
        Args:
            x0: 原始图像
            y: 条件（如类别标签或文本嵌入）
            cfg_scale: CFG引导强度
        """
        batch_size = x0.shape[0]
        
        # 随机丢弃条件（10%概率）
        mask = torch.rand(batch_size) > self.p_uncond
        y_dropped = torch.where(mask, y, None)
        
        # 标准训练
        t = torch.randint(0, self.T, (batch_size,), device=x0.device)
        noise = torch.randn_like(x0)
        xt = self.q_sample(x0, t, noise)
        
        eps_cond = self.model(xt, t, y_dropped)
        
        if self.training:
            # 训练模式：正常MSE
            return (eps_cond - noise).square().mean()
        else:
            # 生成模式：使用CFG
            eps_uncond = self.model(xt, t, None)  # 无条件预测
            eps_cfg = (1 + cfg_scale) * eps_cond - cfg_scale * eps_uncond
            return eps_cfg
```

### CFG调度

研究发现固定的CFG强度并非最优：

```python
def cfg_schedule(epoch, max_epochs, max_w=7.0, min_w=1.0):
    """CFG强度调度：从高到低"""
    progress = epoch / max_epochs
    
    # 余弦衰减
    w = min_w + 0.5 * (max_w - min_w) * (1 + math.cos(math.pi * progress))
    return w

# 或：低噪声时降低引导强度
def adaptive_cfg(noise_level, w_base=7.0, noise_threshold=0.5):
    """自适应CFG"""
    if noise_level < noise_threshold:
        return w_base * (noise_level / noise_threshold)
    return w_base
```

---

## Latent Diffusion Model

### 核心思想

Latent Diffusion Model（LDM）通过在压缩的潜在空间中进行扩散，大幅降低计算成本。[^1]

```
原始空间: H × W × C  →  潜在空间: H/8 × W/8 × 4
                          ↓
        VAE编码 ──────────────────→ VAE解码
                          ↓
                   扩散过程在潜在空间进行
```

### VAE架构

```python
class VAE(nn.Module):
    """变分自编码器"""
    
    def __init__(self, latent_dim=4):
        super().__init__()
        
        # 编码器
        self.encoder = nn.Sequential(
            nn.Conv2d(3, 64, 4, stride=2, padding=1),
            nn.SiLU(),
            ResBlock(64, 128), Downsample(128),
            ResBlock(128, 256), Downsample(256),
            ResBlock(256, 512), Downsample(512),
            ResBlock(512, 512),
            nn.GroupNorm(32, 512),
            nn.SiLU(),
            nn.Conv2d(512, latent_dim * 2, 3, padding=1)  # 均值+对数方差
        )
        
        # 解码器
        self.decoder = nn.Sequential(
            nn.Conv2d(latent_dim, 512, 3, padding=1),
            ResBlock(512, 512), Upsample(512),
            ResBlock(512, 256), Upsample(256),
            ResBlock(256, 128), Upsample(128),
            ResBlock(128, 64), Upsample(64),
            nn.GroupNorm(32, 64),
            nn.SiLU(),
            nn.Conv2d(64, 3, 3, padding=1)
        )
    
    def encode(self, x):
        h = self.encoder(x)
        mean, logvar = h.chunk(2, dim=1)
        logvar = torch.clamp(logvar, -30, 20)
        std = torch.exp(0.5 * logvar)
        z = mean + std * torch.randn_like(std)
        return z
    
    def decode(self, z):
        return self.decoder(z)
    
    def forward(self, x):
        z = self.encode(x)
        recon = self.decode(z)
        return recon, z
```

### LDM实现

```python
class LatentDiffusionModel(nn.Module):
    """潜在扩散模型"""
    
    def __init__(self, latent_channels=4, T=1000, device='cuda'):
        super().__init__()
        self.latent_channels = latent_channels
        self.T = T
        self.device = device
        
        # VAE（冻结）
        self.vae = VAE(latent_channels).to(device)
        for p in self.vae.parameters():
            p.requires_grad = False
        
        # 扩散模型（在潜在空间）
        self.diffusion = LatentUNet(latent_channels)
        
        # 文本编码器（可选）
        self.text_encoder = CLIPTextEncoder()
    
    def encode_image(self, x):
        """图像编码到潜在空间"""
        with torch.no_grad():
            z = self.vae.encode(x)
        # 下采样因子通常为8
        z = F.avg_pool2d(z, 2)  # 额外下采样
        return z
    
    def decode_latent(self, z):
        """潜在空间解码到图像"""
        z = F.interpolate(z, scale_factor=2)  # 上采样回原尺寸
        with torch.no_grad():
            x = self.vae.decode(z)
        return x
    
    @torch.no_grad()
    def generate(self, prompt, num_images=1, cfg_scale=7.5, num_steps=50):
        """文本到图像生成"""
        # 编码文本
        text_emb = self.text_encoder(prompt)
        
        # 初始化潜在空间噪声
        shape = (num_images, self.latent_channels, 64, 64)
        latents = torch.randn(shape, device=self.device)
        
        # DDIM采样
        for i, t in enumerate(reversed(range(0, self.T, self.T // num_steps))):
            t_batch = torch.full((num_images,), t, device=self.device)
            
            # 预测噪声（使用CFG）
            eps_cond = self.diffusion(latents, t_batch, text_emb)
            eps_uncond = self.diffusion(latents, t_batch, None)
            eps = (1 + cfg_scale) * eps_cond - cfg_scale * eps_uncond
            
            # DDIM步骤
            alpha_bar = self.get_alpha_bar(t)
            alpha_bar_prev = self.get_alpha_bar(max(0, t - self.T // num_steps))
            
            # 预测x0
            x0_pred = (latents - torch.sqrt(1 - alpha_bar) * eps) / torch.sqrt(alpha_bar)
            x0_pred = torch.clamp(x0_pred, -1, 1)
            
            # 隐式轨迹
            pred = torch.sqrt(alpha_bar_prev) * x0_pred + torch.sqrt(1 - alpha_bar_prev) * eps
            latents = pred
        
        # 解码到图像空间
        images = self.decode_latent(latents)
        return images
```

---

## 采样加速技术

### DDIM

```python
def ddim_step(xt, t, t_prev, eps, alpha_bar, alpha_bar_prev, eta=0.0):
    """
    DDIM单步采样
    
    Args:
        xt: 当前噪声
        t, t_prev: 当前和上一步的时间步
        eps: 预测的噪声
        eta: 随机性控制（0=确定性, 1=完全随机）
    """
    # 预测x0
    x0_pred = (xt - torch.sqrt(1 - alpha_bar) * eps) / torch.sqrt(alpha_bar)
    x0_pred = torch.clamp(x0_pred, -1, 1)
    
    # 系数
    c1 = torch.sqrt(alpha_bar_prev) * (1 - alpha_bar) / (1 - alpha_bar_prev)
    c2 = torch.sqrt(alpha_bar_prev)
    
    # 均值
    pred = c1 * eps + c2 * x0_pred
    
    # 方差
    var = eta * (1 - alpha_bar_prev) / (1 - alpha_bar) * (1 - alpha_bar / alpha_bar_prev)
    std = torch.sqrt(torch.clamp(var, min=1e-20))
    
    # 添加噪声
    noise = torch.randn_like(xt) if eta > 0 else 0
    return pred + std * noise
```

### DPM-Solver

```python
class DPMSolver:
    """DPM-Solver: 高阶ODE求解器"""
    
    def __init__(self, model, alpha_bar_fn):
        self.model = model
        self.alpha_bar = alpha_bar_fn
    
    def dpm_solver_first_order(self, xt, t, t_next):
        """一阶DPM-Solver"""
        lambda_t = torch.log(t) - torch.log(1 - self.alpha_bar(t))
        lambda_s = torch.log(s) - torch.log(1 - self.alpha_bar(s))
        
        h = lambda_s - lambda_t
        eps = self.model(xt, t)
        
        return xt - (1 - self.alpha_bar(s)) * eps
    
    def dpm_solver_second_order(self, xt, t, s):
        """二阶DPM-Solver"""
        lambda_t = torch.log(t) - torch.log(1 - self.alpha_bar(t))
        lambda_s = torch.log(s) - torch.log(1 - self.alpha_bar(s))
        
        h = lambda_s - lambda_t
        eps_theta_1 = self.model(xt, t)
        eps_theta_2 = self.model(xt - h * eps_theta_1, s)
        
        return xt - (1 - self.alpha_bar(s)) * (eps_theta_1 + 0.5 * h * (eps_theta_2 - eps_theta_1))
```

---

## 实践建议

### 训练技巧

| 技巧 | 描述 | 效果 |
|------|------|------|
| **指数移动平均(EMA)** | 权重指数滑动平均 | 稳定生成 |
| **梯度裁剪** | 限制梯度范数 | 稳定训练 |
| **混合精度** | FP16/BF16训练 | 加速+省显存 |
| **渐进式退火** | 损失权重随训练调整 | 加速收敛 |
| **数据增强** | 随机裁剪、翻转 | 防止过拟合 |

```python
# EMA实现
class EMA:
    def __init__(self, model, decay=0.9999):
        self.model = model
        self.decay = decay
        self.shadow = {}
        self.backup = {}
        
        for name, param in model.named_parameters():
            if param.requires_grad:
                self.shadow[name] = param.data.clone()
    
    def update(self):
        for name, param in self.model.named_parameters():
            if param.requires_grad:
                new_avg = (1 - self.decay) * param.data + self.decay * self.shadow[name]
                self.shadow[name] = new_avg.clone()
    
    def apply_shadow(self):
        for name, param in self.model.named_parameters():
            if param.requires_grad:
                self.backup[name] = param.data
                param.data = self.shadow[name]
    
    def restore(self):
        for name, param in self.model.named_parameters():
            if param.requires_grad:
                param.data = self.backup[name]
        self.backup = {}
```

### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 采样颜色过饱和 | 损失权重不当 | 调整SNR权重 |
| 模式塌陷 | 训练不稳定 | 降低学习率 |
| 伪影 | 模型容量不足 | 增大模型 |
| 速度慢 | T过大 | DDIM/DPM-Solver加速 |

---

## 参考资料

[^1]: Rombach, R., Blattmann, A., Lorenz, D., Esser, P., & Ommer, B. (2022). High-Resolution Image Synthesis with Latent Diffusion Models. *CVPR 2022*. https://arxiv.org/abs/2112.10752

[^2]: Ho, J., & Salimans, T. (2022). Classifier-Free Diffusion Guidance. *NeurIPS 2022 Workshop*. https://arxiv.org/abs/2207.12598

[^3]: Song, J., Meng, C., & Ermon, S. (2021). Denoising Diffusion Implicit Models. *ICLR 2021*. https://arxiv.org/abs/2010.02502
