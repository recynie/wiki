---
title: 图神经网络
date: 2026-04-17
description: 图神经网络（GNN）的基本概念、消息传递范式及典型架构
tags:
  - gnn
  - graph-neural-network
  - deep-learning
  - message-passing
draft: false
permalink:
---

## 概述

图神经网络（Graph Neural Network，GNN）是一类专门用于处理图结构数据的深度学习模型。[^1]

传统的神经网络（如CNN、RNN）适用于网格数据（图像）和序列数据（文本），而GNN能够处理更加通用的图结构数据。

### 什么图数据？

| 数据类型 | 示例 | 节点 | 边 |
|----------|------|------|-----|
| **社交网络** | 微信好友关系 | 用户 | 好友 |
| **分子结构** | 药物分子 | 原子 | 化学键 |
| **推荐系统** | 用户-商品网络 | 用户/商品 | 交互 |
| **知识图谱** | 实体关系网络 | 实体 | 关系 |
| **交通网络** | 道路网络 | 路口 | 道路 |

---

## 图的表示

### 基本概念

- **节点（Vertex/Node）**：图中的基本单元
- **边（Edge）**：节点之间的关系
- **邻接节点（Neighbors）**：与当前节点直接相连的节点

### 邻接矩阵

设图有 $N$ 个节点，邻接矩阵 $A \in \mathbb{R}^{N \times N}$ 定义为：

$$
A_{ij} = \begin{cases}
1 & \text{若节点 } i \text{ 与节点 } j \text{ 相连} \\
0 & \text{否则}
\end{cases}
$$

对于无向图，$A$ 是对称矩阵。

### 度矩阵

度矩阵 $D \in \mathbb{R}^{N \times N}$ 是对角矩阵：

$$
D_{ii} = \sum_{j} A_{ij}
$$

表示每个节点的邻居数量。

### 拉普拉斯矩阵

拉普拉斯矩阵 $L = D - A$ 是图信号处理的核心工具：

$$
L = D - A = \begin{pmatrix}
d_1 & -a_{12} & \cdots \\
-a_{21} & d_2 & \cdots \\
\vdots & \vdots & \ddots
\end{pmatrix}
$$

**归一化拉普拉斯矩阵**：
$$
L_{norm} = I - D^{-1/2} A D^{-1/2}
$$

---

## 消息传递范式

### 核心思想

消息传递（Message Passing）是GNN的基本操作范式，其核心思想是：**通过聚合邻居节点的信息来更新当前节点的表示**。[^2]

### 节点嵌入

每个节点 $v$ 有一个 $d$ 维特征向量 $\mathbf{x}_v \in \mathbb{R}^d$。GNN的目标是学习一个函数，将节点特征映射到嵌入空间：

$$
\mathbf{h}_v^{(k)} = f^{(k)}(\mathbf{h}_v^{(k-1)}, \{\mathbf{h}_u^{(k-1)} : u \in \mathcal{N}(v)\})
$$

其中：
- $\mathbf{h}_v^{(k)}$：节点 $v$ 在第 $k$ 层的嵌入
- $\mathcal{N}(v)$：节点 $v$ 的邻居集合

### 聚合函数

聚合函数将邻居节点的信息整合为单一向量：

#### 1. 求和聚合（Sum）
$$
\mathbf{m}_{\mathcal{N}(v)} = \sum_{u \in \mathcal{N}(v)} \mathbf{h}_u^{(k-1)}
$$

#### 2. 均值聚合（Mean）
$$
\mathbf{m}_{\mathcal{N}(v)} = \frac{1}{|\mathcal{N}(v)|} \sum_{u \in \mathcal{N}(v)} \mathbf{h}_u^{(k-1)}
$$

#### 3. 最大池化聚合（Max Pooling）
$$
\mathbf{m}_{\mathcal{N}(v)} = \max_{u \in \mathcal{N}(v)} \sigma(\mathbf{h}_u^{(k-1)} \mathbf{W})
$$

### 更新函数

更新函数结合节点自身信息和聚合的邻居信息：

$$
\mathbf{h}_v^{(k)} = \sigma\left(\mathbf{W}^{(k)} \cdot \text{CONCAT}\left(\mathbf{h}_v^{(k-1)}, \mathbf{m}_{\mathcal{N}(v)}\right) + \mathbf{b}^{(k)}\right)
$$

---

## 图卷积网络（GCN）

### Kipf & Welling 算法

2017年，Thomas Kipf和Max Welling提出了简化的图卷积网络（Semi-Supervised Classification with Graph Convolutional Networks）。[^3]

### 层间传播规则

$$
\mathbf{H}^{(l+1)} = \sigma\left(\tilde{D}^{-1/2} \tilde{A} \tilde{D}^{-1/2} \mathbf{H}^{(l)} \mathbf{W}^{(l)}\right)
$$

其中：
- $\tilde{A} = A + I$：添加自连接的邻接矩阵
- $\tilde{D}$：$\tilde{A}$ 的度矩阵
- $\mathbf{H}^{(l)}$：第 $l$ 层的节点特征
- $\mathbf{W}^{(l)}$：可学习的权重矩阵

### 简化形式

对于单层GCN，传播规则简化为：

$$
\mathbf{H}^{(1)} = \sigma\left(\hat{A} \mathbf{X} \mathbf{W}\right)
$$

其中 $\hat{A} = \tilde{D}^{-1/2} \tilde{A} \tilde{D}^{-1/2}$ 是归一化邻接矩阵。

### PyTorch实现

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torch_geometric.nn import GCNConv

class GCN(nn.Module):
    def __init__(self, in_channels, hidden_channels, out_channels):
        super().__init__()
        self.conv1 = GCNConv(in_channels, hidden_channels)
        self.conv2 = GCNConv(hidden_channels, out_channels)
    
    def forward(self, x, edge_index):
        # 第一层GCN + ReLU激活
        x = self.conv1(x, edge_index)
        x = F.relu(x)
        x = F.dropout(x, p=0.5, training=self.training)
        
        # 第二层GCN
        x = self.conv2(x, edge_index)
        return x
```

### 手写实现

```python
import torch

def gcn_layer(X, A, W):
    """
    手写GCN层
    
    参数:
        X: 节点特征 (N, in_features)
        A: 邻接矩阵 (N, N)
        W: 权重矩阵 (in_features, out_features)
    
    返回:
        H: 更新后的节点特征 (N, out_features)
    """
    N = X.shape[0]
    
    # 添加自连接
    A = A + torch.eye(N)
    
    # 计算度矩阵
    D = torch.sum(A, dim=1)
    D_inv_sqrt = torch.pow(D, -0.5)
    D_inv_sqrt[torch.isinf(D_inv_sqrt)] = 0
    
    # 归一化矩阵
    D_inv_sqrt_mat = torch.diag(D_inv_sqrt)
    A_norm = D_inv_sqrt_mat @ A @ D_inv_sqrt_mat
    
    # 图卷积操作
    H = A_norm @ X @ W
    
    return H

# 示例
N, in_feat, out_feat = 4, 8, 16
X = torch.randn(N, in_feat)        # 节点特征
A = torch.randint(0, 2, (N, N)).float()  # 邻接矩阵
W = torch.randn(in_feat, out_feat)  # 权重

H = gcn_layer(X, A, W)
print(f"输出形状: {H.shape}")  # (4, 16)
```

---

## GraphSAGE

### 归纳学习

GraphSAGE的核心贡献是**归纳学习**（Inductive Learning）——能够泛化到未见过的节点和图。[^4]

### 邻居采样

为了处理大图，GraphSAGE使用邻居采样：

```python
from torch_geometric.nn import SAGEConv

class GraphSAGE(nn.Module):
    def __init__(self, in_channels, hidden_channels, out_channels):
        super().__init__()
        self.conv1 = SAGEConv(in_channels, hidden_channels)
        self.conv2 = SAGEConv(hidden_channels, out_channels)
    
    def forward(self, x, edge_index):
        x = self.conv1(x, edge_index)
        x = F.relu(x)
        x = self.conv2(x, edge_index)
        return x
```

### 聚合器设计

GraphSAGE支持多种聚合器：

| 聚合器 | 特点 |
|--------|------|
| **Mean** | 简单平均，类似于GCN |
| **LSTM** | 使用双向LSTM捕获序列信息 |
| **Pooling** | 先做线性变换再做最大池化 |

---

## 图注意力网络（GAT）

### 注意力机制

GAT（Graph Attention Network）将注意力机制引入图神经网络。[^5]

### 注意力系数

$$
e_{ij} = \alpha(\mathbf{W}\mathbf{h}_i, \mathbf{W}\mathbf{h}_j) = \text{LeakyReLU}\left(\mathbf{a}^T[\mathbf{W}\mathbf{h}_i \Vert \mathbf{W}\mathbf{h}_j]\right)
$$

### 归一化注意力权重

$$
\alpha_{ij} = \frac{\exp(e_{ij})}{\sum_{k \in \mathcal{N}(i)} \exp(e_{ik})}
$$

### 最终输出

$$
\mathbf{h}_i' = \sigma\left(\sum_{j \in \mathcal{N}(i)} \alpha_{ij} \mathbf{W}\mathbf{h}_j\right)
$$

### 多头注意力

使用多个注意力头并行计算，增强模型表达能力：

$$
\mathbf{h}_i^{(l)} = \Vert_{k=1}^{K} \sigma\left(\sum_{j \in \mathcal{N}(i)} \alpha_{ij}^{(k)} \mathbf{W}^{(k)}\mathbf{h}_j\right)
$$

### PyTorch实现

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torch_geometric.nn import GATConv

class GAT(nn.Module):
    def __init__(self, in_channels, hidden_channels, out_channels, heads=8):
        super().__init__()
        self.gat1 = GATConv(in_channels, hidden_channels, heads=heads, dropout=0.6)
        self.gat2 = GATConv(hidden_channels * heads, out_channels, heads=1, concat=False, dropout=0.6)
    
    def forward(self, x, edge_index):
        x = F.dropout(x, p=0.6, training=self.training)
        x = self.gat1(x, edge_index)
        x = F.elu(x)
        x = F.dropout(x, p=0.6, training=self.training)
        x = self.gat2(x, edge_index)
        return x
```

---

## GNN的应用场景

### 节点分类

根据节点特征和图结构，预测节点的标签。

**典型任务**：论文引用网络中的主题分类

### 链接预测

预测图中可能存在但尚未被观测到的边。

**典型任务**：推荐系统中的商品推荐

### 图分类

将整个图作为输入，预测图的属性。

**典型任务**：分子性质预测（药物发现）

### 图生成

学习图的分布，生成新的图结构。

**典型任务**：新分子生成

---

## GNN的表达能力

### 与 Weisfeiler-Lehman 测试的关系

GNN的表达能力与图同构测试（Weisfeiler-Lehman 1-WL测试）密切相关。

**定理**：如果两层GNN的聚合函数满足特定条件，则它能够区分的图与1-WL测试相同。

### GNN的局限性

1. **过平滑问题**：随着层数增加，节点表示趋于相似
2. **表达能力有限**：无法区分某些非同构的图
3. **计算复杂度**：大规模图的计算开销大

---

## 与其他模型的关系

### GNN vs CNN

| 维度 | CNN | GNN |
|------|-----|-----|
| 数据结构 | 网格/欧式空间 | 图/非欧空间 |
| 邻居 | 固定大小 | 可变大小 |
| 聚合 | 卷积核 | 消息传递 |
| 平移不变性 | 有 | 无 |

### GNN vs Transformer

Transformer本质上可以视为一种**全连接的GNN**：

- Transformer中的自注意力 = GNN中的消息传递
- 所有token互为邻居
- 无需预先定义图结构

详见 [[./transformer-and-attention|Transformer与注意力机制]]。

---

## 参考

[^1]: Zhou et al., "Graph Neural Networks: A Survey of Methods and Applications", arXiv 2018

[^2]: Gilmer et al., "Neural Message Passing for Quantum Chemistry", ICML 2017

[^3]: Kipf & Welling, "Semi-Supervised Classification with Graph Convolutional Networks", ICLR 2017

[^4]: Hamilton et al., "Inductive Representation Learning on Large Graphs", NeurIPS 2017

[^5]: Veličković et al., "Graph Attention Networks", ICLR 2018
