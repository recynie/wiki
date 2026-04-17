---
title: 图卷积网络详解
date: 2026-04-17
description: 深入解析图卷积网络（GCN）的谱方法、空间方法及代表性架构
tags:
  - gcn
  - graph-convolutional-network
  - deep-learning
  - spectral-methods
draft: false
permalink:
---

## 概述

图卷积网络（Graph Convolutional Network，GCN）是图神经网络的重要分支，通过**图卷积操作**将传统卷积推广到图结构数据。[^1]

本文档深入讲解GCN的理论基础，包括谱域方法和空域方法。

---

## 谱域方法：图信号处理

### 图上的傅里叶变换

传统傅里叶变换将信号分解为不同频率的叠加。类似地，图上的傅里叶变换使用**拉普拉斯矩阵的特征向量**作为基。

#### 拉普拉斯矩阵的特征性质

归一化拉普拉斯矩阵 $L = I - D^{-1/2} A D^{-1/2}$：
- 是**半正定**的
- 有 $N$ 个非负特征值：$0 = \lambda_0 \leq \lambda_1 \leq \ldots \leq \lambda_{N-1}$
- 特征值 $\lambda_i$ 可以理解为图的"频率"

#### 图傅里叶变换

$$
\hat{x}(\lambda_i) = \langle x, u_i \rangle = \sum_{n=1}^{N} x_n u_{i,n}^*
$$

其中 $u_i$ 是第 $i$ 个特征向量。

**矩阵形式**：
$$
\hat{x} = U^T x
$$

其中 $U$ 是特征向量矩阵。

#### 逆图傅里叶变换

$$
x_n = \sum_{i=0}^{N-1} \hat{x}(\lambda_i) u_{i,n}
$$

**矩阵形式**：
$$
x = U \hat{x}
$$

### 图卷积的定义

图上的卷积操作定义为频域乘积的逆变换：

$$
(x * y)_G = U \left( (U^T x) \odot (U^T y) \right)
$$

其中 $\odot$ 是Hadamard乘积（元素级乘法）。

### 谱域GCN

#### 谱卷积层

Spectral GCN的第一层定义为：

$$
H^{(l+1)} = \sigma\left(U \Theta U^T H^{(l)} W^{(l)}\right)
$$

其中：
- $H^{(l)}$：第 $l$ 层的特征
- $\Theta$：可学习的对角滤波器参数
- $W^{(l)}$：特征变换矩阵

#### ChebNet：多项式滤波器

ChebNet使用Chebyshev多项式近似滤波器：[^2]

$$
\Theta(\Lambda) = \sum_{k=0}^{K-1} \theta_k T_k(\tilde{\Lambda})
$$

其中：
- $\tilde{\Lambda} = \frac{2}{\lambda_{max}} \Lambda - I$
- $T_k$ 是 $k$ 阶Chebyshev多项式

**递归形式**：
$$
x' = \sigma\left(\sum_{k=0}^{K} \theta_k T_k(\tilde{L}) x\right)
$$

其中 $\tilde{L} = \frac{2}{\lambda_{max}} L - I$。

### 切比雪夫多项式的递推

$$
T_k(x) = 2x \cdot T_{k-1}(x) - T_{k-2}(x)
$$

初始值：
- $T_0(x) = 1$
- $T_1(x) = x$

---

## 空域方法：消息传递

### GCN的空域解释

Kipf & Welling的GCN可以看作消息传递的特殊情况。

### 重新审视传播规则

$$
H^{(l+1)} = \sigma\left(\tilde{D}^{-1/2} \tilde{A} \tilde{D}^{-1/2} H^{(l)} W^{(l)}\right)
$$

#### 逐通道操作

对于单个节点 $v$：

$$
h_v^{(l+1)} = \sigma\left( \sum_{u \in \tilde{\mathcal{N}}(v)} \frac{1}{c_{uv}} h_u^{(l)} W^{(l)} \right)
$$

其中 $c_{uv} = \sqrt{d_u d_v}$ 是归一化因子。

### 与消息传递的统一

GCN的消息传递形式：

$$
m_{u \to v}^{(l)} = \frac{1}{c_{uv}} h_u^{(l-1)} W^{(l-1)}
$$
$$
m_{\mathcal{N}(v)}^{(l)} = \sum_{u \in \mathcal{N}(v)} m_{u \to v}^{(l)}
$$
$$
h_v^{(l)} = \sigma\left( m_{\mathcal{N}(v)}^{(l)} + h_v^{(l-1)} W^{(l-1)} \right)
$$

---

## GCN的深入分析

### 1. 拉普拉斯平滑

图卷积本质上是一种**拉普拉斯平滑**（Laplacian Smoothing）：

$$
L_{sym} = I - D^{-1/2} A D^{-1/2}
$$

平滑操作使相邻节点的表示趋于相似。

### 2. 过平滑问题

随着GCN层数增加，所有节点的表示会趋于相同——这就是**过平滑**（Over-smoothing）问题。[^3]

**原因**：多次平滑导致信息损失

**解决方案**：
- 残差连接（ResNet思想）
- 跳过连接（Jump Knowledge Network）
- 适当的层数（通常2-3层效果最好）

### 3. 感受野与邻居采样

对于 $K$ 层GCN，每个节点的信息来自 $K$ 跳邻居：

```
层数0: 节点自身
层数1: 直接邻居
层数2: 邻居的邻居
...
层数K: K跳邻居
```

大图中的邻居指数增长可通过**采样**控制。

---

## 改进的GCN架构

### 1. GCN with ResNet (ResGCN)

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torch_geometric.nn import GCNConv

class ResGCN(nn.Module):
    """带残差连接的GCN"""
    def __init__(self, in_channels, hidden_channels, out_channels):
        super().__init__()
        self.conv1 = GCNConv(in_channels, hidden_channels)
        self.conv2 = GCNConv(hidden_channels, out_channels)
        self.res_proj = nn.Linear(in_channels, out_channels)
    
    def forward(self, x, edge_index):
        # 保存残差
        residual = self.res_proj(x)
        
        # 第一层 + 激活
        x = self.conv1(x, edge_index)
        x = F.relu(x)
        
        # 第二层 + 残差连接
        x = self.conv2(x, edge_index)
        x = x + residual
        
        return F.relu(x)
```

### 2. GAT (Graph Attention Network)

GAT使用注意力机制自适应地为不同邻居分配权重。详见 [[./graph-neural-network|图神经网络]]。

### 3. GraphSAGE

GraphSAGE通过聚合函数的不同实现变体：

```python
from torch_geometric.nn import SAGEConv

class MeanGraphSAGE(nn.Module):
    def __init__(self, in_channels, out_channels):
        super().__init__()
        self.conv = SAGEConv(in_channels, out_channels, aggr='mean')
    
    def forward(self, x, edge_index):
        return self.conv(x, edge_index)

class MaxGraphSAGE(nn.Module):
    def __init__(self, in_channels, out_channels):
        super().__init__()
        self.conv = SAGEConv(in_channels, out_channels, aggr='max')
    
    def forward(self, x, edge_index):
        return self.conv(x, edge_index)
```

---

## GCN的应用实例

### 论文引用网络

**数据集**：Cora/Citeseer/Pubmed

| 数据集 | 节点数 | 边数 | 类别数 |
|--------|--------|------|--------|
| Cora | 2,708 | 5,429 | 7 |
| Citeseer | 3,327 | 4,732 | 6 |
| Pubmed | 19,717 | 44,338 | 3 |

### 半监督节点分类

```python
import torch
import torch.nn.functional as F
from torch_geometric.datasets import Planetoid
from torch_geometric.nn import GCNConv

# 加载数据
dataset = Planetoid(root='/tmp/Cora', name='Cora')
data = dataset[0]

# 定义模型
class Net(torch.nn.Module):
    def __init__(self):
        super(Net, self).__init__()
        self.conv1 = GCNConv(dataset.num_node_features, 16)
        self.conv2 = GCNConv(16, dataset.num_classes)
    
    def forward(self, x, edge_index):
        x = self.conv1(x, edge_index)
        x = F.relu(x)
        x = F.dropout(x, training=self.training)
        x = self.conv2(x, edge_index)
        return F.log_softmax(x, dim=1)

# 训练
model = Net()
optimizer = torch.optim.Adam(model.parameters(), lr=0.01, weight_decay=5e-4)

model.train()
for epoch in range(200):
    optimizer.zero_grad()
    out = model(data.x, data.edge_index)
    loss = F.nll_loss(out[data.train_mask], data.y[data.train_mask])
    loss.backward()
    optimizer.step()

# 测试
model.eval()
_, pred = model(data.x, data.edge_index).max(dim=1)
correct = int(pred[data.test_mask].eq(data.y[data.test_mask]).sum())
acc = correct / int(data.test_mask.sum())
print(f'准确率: {acc:.4f}')
```

---

## GCN vs 其他图模型

### 对比表

| 模型 | 聚合方式 | 表达能力 | 计算复杂度 |
|------|----------|----------|------------|
| **GCN** | 求和/归一化 | 中等 | $O(E \cdot F)$ |
| **GAT** | 注意力加权 | 较强 | $O(E \cdot F^2)$ |
| **GraphSAGE** | Mean/Max/LSTM | 较强 | $O(E \cdot F)$ |
| **ChebNet** | 多项式滤波 | 强 | $O(K \cdot E \cdot F)$ |

### 何时使用哪种模型？

| 场景 | 推荐模型 |
|------|----------|
| 节点特征重要 | GCN |
| 邻居重要性不同 | GAT |
| 需要归纳学习 | GraphSAGE |
| 需要高表达能力 | ChebNet |

---

## 深度学习中的图结构

### GNN与Transformer的关系

Transformer可以视为**全连接图**上的GNN：

| 组件 | Transformer | GNN |
|------|------------|-----|
| 图结构 | 全连接（所有token互连） | 稀疏邻接矩阵 |
| 边权重 | 自注意力分数 | 归一化邻接矩阵 |
| 聚合 | 多头注意力 | 消息传递 |

### GNN在推荐系统中的应用

现代推荐系统广泛使用图神经网络：

```
用户 → 交互 → 商品
  ↓         ↓
社交关系   相似商品
```

图神经网络能够：
- 聚合用户/商品的邻居信息
- 学习协同过滤效应
- 处理冷启动问题

---

## 参考

[^1]: Kipf & Welling, "Semi-Supervised Classification with Graph Convolutional Networks", ICLR 2017

[^2]: Defferrard et al., "Convolutional Neural Networks on Graphs with Fast Localized Spectral Filtering", NeurIPS 2016

[^3]: Li et al., "Deeper Insights into Graph Convolutional Networks for Semi-Supervised Learning", AAAI 2018
