---
title: 变分推断
date: 2026-04-17
description: 变分推断（Variational Inference）的原理、与信息论的联系，以及在机器学习中的应用
tags:
  - probability
  - variational-inference
  - bayesian
  - information-theory
draft: false
permalink:
---

# 变分推断

**变分推断**（Variational Inference, VI）是贝叶斯统计中一种高效的近似推断方法，通过优化一个近似分布来逼近真实后验分布。[^1] 在现代机器学习中，变分推断是变分自编码器（VAE）、变分信息瓶颈（VIB）等模型的理论基础。

## 问题背景

### 贝叶斯推断

在贝叶斯框架下，我们关心后验分布：

$$
p(z \mid x) = \frac{p(x \mid z) p(z)}{p(x)}
$$

其中：
- $p(z)$ 是**先验分布**（Prior）
- $p(x \mid z)$ 是**似然函数**（Likelihood）
- $p(z \mid x)$ 是**后验分布**（Posterior）
- $p(x) = \int p(x \mid z) p(z) dz$ 是**边缘似然**（Evidence）

### 计算挑战

边缘似然 $p(x) = \int p(x \mid z) p(z) dz$ 通常**难以计算**（积分无解析解），这导致：
- 无法直接计算后验 $p(z \mid x)$
- 无法使用最大似然估计（MLE）

**变分推断的核心思想**：用一个简单的分布 $q(z)$ 来近似复杂的真实后验 $p(z \mid x)$。

---

## 变分推断原理

### 优化目标

寻找最优的近似分布 $q^*(z)$：

$$
q^*(z) = \arg\min_q \; D_{KL}(q(z) \| p(z \mid x))
$$

这等价于最小化两个分布之间的 KL 散度。

### 分解与推导

对 KL 散度进行展开：

$$
D_{KL}(q(z) \| p(z \mid x)) = \int q(z) \log \frac{q(z)}{p(z \mid x)} dz
$$

代入贝叶斯公式 $p(z \mid x) = \frac{p(x \mid z) p(z)}{p(x)}$：

$$
\begin{aligned}
D_{KL}(q(z) \| p(z \mid x)) &= \int q(z) \log \frac{q(z) p(x)}{p(x \mid z) p(z)} dz \\
&= \int q(z) \log \frac{q(z)}{p(z)} dz + \int q(z) \log p(x) dz - \int q(z) \log p(x \mid z) dz \\
&= D_{KL}(q(z) \| p(z)) + \log p(x) - \mathbb{E}_q[\log p(x \mid z)]
\end{aligned}
$$

### 证据下界（ELBO）

重新整理得到：

$$
\log p(x) = D_{KL}(q(z) \| p(z \mid x)) + \underbrace{\mathbb{E}_q[\log p(x \mid z)] - D_{KL}(q(z) \| p(z))}_{\text{ELBO}}
$$

由于 KL 散度非负：

$$
\log p(x) \geq \mathcal{L}(q) = \mathbb{E}_q[\log p(x \mid z)] - D_{KL}(q(z) \| p(z))
$$

**$\mathcal{L}(q)$ 就是证据下界**（Evidence Lower Bound, ELBO）。

---

## ELBO 的信息论解释

### 两种视角

#### 视角一：重构 + 正则化

$$
\mathcal{L}(q) = \underbrace{\mathbb{E}_q[\log p(x \mid z)]}_{\text{重构项}} - \underbrace{D_{KL}(q(z) \| p(z))}_{\text{正则项}}
$$

| 项 | 含义 |
|----|------|
| $\mathbb{E}_q[\log p(x \mid z)]$ | 重构损失：$q$ 应该能重建 $x$ |
| $D_{KL}(q(z) \| p(z))$ | 正则项：$q$ 不应偏离先验太远 |

#### 视角二：信息瓶颈

重新审视 ELBO：

$$
\mathcal{L} = I(X; Z) - D_{KL}(q(z) \| p(z))
$$

- **$I(X; Z)$**：输入与表示之间的互信息（信息保留）
- **$D_{KL}(q(z) \| p(z))$**：与先验的偏离（压缩）

这与[[../math/information-bottleneck|信息瓶颈理论]]的目标高度一致！

### 直观理解

```
┌──────────────────────────────────────────────────────┐
│                                                      │
│    log p(x)                                          │
│      │                    ╭────────── ELBO           │
│      │                   ╱                           │
│      │                  ╱                            │
│      │                 ╱   ════════════════         │
│      │                ╱                             │
│      │               ╱                              │
│      │              ╱                                │
│      │             ╱    KL(q || p(z|x))             │
│      │            ╱                                  │
│      └────────────╰─────────────────────────────────→ q 的复杂度
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## 平均场变分推断

### 分解假设

**平均场假设**（Mean Field Assumption）将近似分布分解为独立因子的乘积：

$$
q(z) = \prod_{i=1}^{K} q_i(z_i)
$$

其中每个 $q_i(z_i)$ 只依赖对应的隐变量 $z_i$。

### 坐标上升更新

对于平均场变分推断，可以通过**坐标上升法**（Coordinate Ascent）求解：

**最优解**：

$$
q_j^*(z_j) \propto p(z_j) \exp\left(\mathbb{E}_{-q_j}[\log p(x, z)]\right)
$$

### 变分混合模型示例

```python
import numpy as np
import scipy.special as sp

class VariationalGaussianMixture:
    """
    使用变分推断的高斯混合模型
    
    假设数据由 K 个高斯分布生成：
    - z ~ Cat(π): 混合系数
    - x|z=k ~ N(μ_k, Σ_k): 观测分布
    """
    def __init__(self, n_components, max_iter=100, tol=1e-6):
        self.K = n_components
        self.max_iter = max_iter
        self.tol = tol
        
    def _init_params(self, X):
        N, D = X.shape
        self.pi = np.ones(self.K) / self.K  # 混合系数
        self.mu = X[np.random.choice(N, self.K, False)]  # 均值
        self.var = np.ones((self.K, D))  # 方差
        
    def fit(self, X):
        self._init_params(X)
        N = X.shape[0]
        
        for it in range(self.max_iter):
            # E-step: 计算隐变量的后验
            log_rho = np.zeros((N, self.K))
            for k in range(self.K):
                log_rho[:, k] = np.log(self.pi[k]) + \
                                self._log_gaussian_pdf(X, self.mu[k], self.var[k])
            
            # 归一化（log-sum-exp 技巧）
            log_rho_max = np.max(log_rho, axis=1, keepdims=True)
            rho = np.exp(log_rho - log_rho_max)
            rho = rho / np.sum(rho, axis=1, keepdims=True)
            
            # 计算 ELBO
            elbo = self._compute_elbo(X, rho)
            
            # M-step: 更新参数
            Nk = np.sum(rho, axis=0)  # effective counts
            self.pi = Nk / N
            
            for k in range(self.K):
                # 更新均值
                self.mu[k] = np.sum(rho[:, k:k+1] * X, axis=0) / Nk[k]
                # 更新方差
                diff = X - self.mu[k]
                self.var[k] = np.sum(rho[:, k:k+1] * diff**2, axis=0) / Nk[k]
            
            # 检查收敛
            if it > 0 and abs(elbo - prev_elbo) < self.tol:
                break
            prev_elbo = elbo
                
        return self
    
    def _log_gaussian_pdf(self, X, mu, var):
        D = X.shape[1]
        return -0.5 * D * np.log(2*np.pi) - 0.5 * np.sum(np.log(var)) - \
               0.5 * np.sum((X - mu)**2 / var, axis=1)
    
    def _compute_elbo(self, X, rho):
        """计算 ELBO 的各部分"""
        N = X.shape[0]
        
        # 重构项
        recon = 0
        for k in range(self.K):
            recon += np.sum(rho[:, k] * self._log_gaussian_pdf(X, self.mu[k], self.var[k]))
        
        # KL 项 (pi)
        pi_kl = np.sum(rho * (np.log(self.pi + 1e-10) - np.log(rho + 1e-10)))
        
        return recon - pi_kl
```

---

## 变分推断 vs MCMC

| 特性 | 变分推断 | MCMC |
|------|----------|------|
| **速度** | 快（优化问题） | 慢（采样） |
| **精度** | 有偏近似 | 无偏（渐近） |
| **扩展性** | 易扩展到大数据 | 困难 |
| **收敛诊断** | 直接 | 困难 |
| **适用场景** | 大规模、快速原型 | 高精度需求 |

---

## 在机器学习中的应用

### 1. 变分自编码器（VAE）

VAE 使用变分推断来学习隐变量表示：[^2]

```python
class VAE(nn.Module):
    """
    变分自编码器
    
    ELBO = E_q[log p(x|z)] - D_KL(q(z|x) || p(z))
    """
    def __init__(self, input_dim, latent_dim):
        super().__init__()
        # 编码器：q(z|x)
        self.encoder = nn.Sequential(
            nn.Linear(input_dim, 512),
            nn.ReLU(),
            nn.Linear(512, 256),
            nn.ReLU(),
            nn.Linear(256, 2 * latent_dim)  # mu and log_var
        )
        # 解码器：p(x|z)
        self.decoder = nn.Sequential(
            nn.Linear(latent_dim, 256),
            nn.ReLU(),
            nn.Linear(256, 512),
            nn.ReLU(),
            nn.Linear(512, input_dim)
        )
        
    def reparameterize(self, mu, log_var):
        std = torch.exp(0.5 * log_var)
        eps = torch.randn_like(std)
        return mu + eps * std
    
    def forward(self, x):
        # 编码
        h = self.encoder(x)
        mu, log_var = h.chunk(2, dim=-1)
        z = self.reparameterize(mu, log_var)
        
        # 解码
        x_recon = self.decoder(z)
        
        return x_recon, mu, log_var
    
    def loss(self, x):
        x_recon, mu, log_var = self.forward(x)
        
        # 重构损失（二次误差或 BCE）
        recon_loss = F.mse_loss(x_recon, x, reduction='sum')
        
        # KL 散度：D_KL(N(mu, sigma) || N(0, I))
        kl_loss = -0.5 * torch.sum(1 + log_var - mu.pow(2) - log_var.exp())
        
        # ELBO
        elbo = recon_loss + kl_loss
        
        return elbo / x.size(0), recon_loss / x.size(0), kl_loss / x.size(0)
```

### 2. 变分信息瓶颈（VIB）

见 [[../math/information-bottleneck|信息瓶颈理论]]。

### 3. 贝叶斯神经网络

使用变分推断进行贝叶斯后验近似：

```python
class BayesianLinear(nn.Module):
    """
    贝叶斯线性层
    
    使用变分推断近似权重后验 q(w|θ)
    """
    def __init__(self, in_features, out_features):
        super().__init__()
        self.in_features = in_features
        self.out_features = out_features
        
        # 变分参数
        self.weight_mu = nn.Parameter(torch.randn(out_features, in_features) * 0.1)
        self.weight_log_var = nn.Parameter(torch.zeros(out_features, in_features))
        
        # 先验（标准高斯）
        self.prior_mu = torch.zeros(out_features, in_features)
        self.prior_log_var = torch.zeros(out_features, in_features)
        
    def forward(self, x):
        # 从近似后验采样权重
        weight = self.weight_mu + torch.randn_like(self.weight_mu) * \
                 torch.exp(0.5 * self.weight_log_var)
        return F.linear(x, weight)
    
    def kl_loss(self):
        """计算与先验的 KL 散度"""
        q_mean, q_log_var = self.weight_mu, self.weight_log_var
        p_mean, p_log_var = self.prior_mu, self.prior_log_var
        
        kl = 0.5 * torch.sum(
            p_log_var - q_log_var + 
            (q_log_var.exp() + (q_mean - p_mean).pow(2)) / p_log_var.exp() - 1
        )
        return kl
```

---

## 进阶主题

### 黑盒变分推断（BBVI）

当期望无法解析计算时，使用随机梯度变分推断（SGVI）：

$$
\nabla_\phi \mathcal{L} \approx \mathbb{E}_{q(z;\phi)}\left[\nabla_\phi \log q(z;\phi) \cdot \left(\log p(x,z) - \log q(z;\phi)\right)\right]
$$

### 共轭性

| 模型 | 先验 | 似然 | 后验 |
|------|------|------|------|
| Beta-Binomial | Beta | Binomial | Beta |
| Dirichlet-Multinomial | Dirichlet | Multinomial | Dirichlet |
| Gaussian-Gaussian | Normal | Normal | Normal |

共轭先验允许解析计算后验，简化变分推断。

---

## 核心公式速查

| 概念 | 公式 |
|------|------|
| **KL 散度目标** | $\min_q D_{KL}(q(z) \| p(z \mid x))$ |
| **ELBO 定义** | $\mathcal{L}(q) = \mathbb{E}_q[\log p(x \mid z)] - D_{KL}(q(z) \| p(z))$ |
| **ELBO 等价形式** | $\log p(x) = D_{KL}(q(z) \| p(z \mid x)) + \mathcal{L}(q)$ |
| **KL 散度展开** | $D_{KL}(q \| p) = \mathbb{E}_q[\log q] - \mathbb{E}_q[\log p]$ |
| **平均场最优解** | $q_j^*(z_j) \propto p(z_j) \exp(\mathbb{E}_{-q_j}[\log p(x,z)])$ |

---

## 参考

[^1]: Blei, D.M., Kucukelbir, A., & McAuliffe, J.D. (2017). "Variational Inference: A Review for Statisticians". *Journal of the American Statistical Association*, 112(518), 859-877.
[^2]: Kingma, D.P., & Welling, M. (2014). "Auto-Encoding Variational Bayes". *ICLR*.
[^3]: Rezende, D.J., Mohamed, S., & Wierstra, D. (2014). "Stochastic Backpropagation and Approximate Inference in Deep Generative Models". *ICML*.

## 相关文章

- [[../math/information-theory|信息论基础]] — 熵、互信息、KL散度
- [[../math/information-bottleneck|信息瓶颈理论]] — IB 目标与深度学习
- [[../machine-learning/deep-vib|Deep VIB 实现]] — 变分信息瓶颈实战
