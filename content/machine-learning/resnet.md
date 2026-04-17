---
title: ResNet与残差学习
date: 2026-04-17
description: 深度残差网络ResNet的理论分析与实现
tags:
  - deep-learning
  - cnn
  - resnet
  - skip-connection
draft: false
permalink:
---

# ResNet与残差学习

ResNet（Deep Residual Learning Network）是2015年ImageNet比赛的冠军，通过残差连接解决了深层网络的训练难题。这一创新使得训练数百甚至上千层的神经网络成为可能。

## 问题背景

### 梯度消失/爆炸

深层网络中，梯度在反向传播时会逐层指数衰减（消失）或增长（爆炸），导致前层参数几乎无法更新。

设网络有 $L$ 层，每层梯度传递为 $\prod_{i=1}^{L} \sigma'(z_i) W^i$。当层数很深时：
- 若 $|W\sigma'(z)| < 1$，梯度趋于0
- 若 $|W\sigma'(z)| > 1$，梯度爆炸

### 网络退化问题

更关键的问题是：**随着网络加深，训练误差和测试误差反而上升**。

这说明深层网络难以学习恒等映射（identity mapping）。当浅层网络已经足够好时，深层网络应该至少能达到浅层网络的性能。

## 残差学习核心思想

### 残差块设计

传统网络学习映射：$H(x) = \text{Desired Mapping}$

残差网络学习残差：$\mathcal{F}(x) = H(x) - x$

最终输出：$H(x) = \mathcal{F}(x) + x$

### 为什么残差学习更容易

直觉上，如果恒等映射是最优解，那么将残差推向零比通过非线性层学习恒等映射更容易：

- 学习 $\mathcal{F}(x) = 0$：只需将所有权重设为0（相对简单）
- 学习 $H(x) = x$：需要非线性层拟合恒等映射（可能很复杂）

### 数学分析

假设我们要学习的目标映射为 $H^*(x)$，实际学习 $\mathcal{F}(x) = H^*(x) - x$。

若 $H^*(x) \approx x$（接近恒等映射），则：
- 传统网络：需要通过非线性层学习 $H^*(x)$
- 残差网络：$\mathcal{F}(x) \approx 0$，学习小残差更容易

## 残差块结构

### 基本形式

$$
y = \mathcal{F}(x, \{W_i\}) + x
$$

其中 $\mathcal{F}(x) = W_2 \cdot \sigma(W_1 \cdot x)$ 是可学习的残差函数。

### PyTorch实现

```python
import torch
import torch.nn as nn

class ResidualBlock(nn.Module):
    """
    标准的残差块
    - 两次3x3卷积
    - 残差连接
    - 后期激活（post-activation）
    """
    expansion = 1  # 通道数扩展因子
    
    def __init__(self, in_channels, out_channels, stride=1):
        super().__init__()
        
        # 第一个卷积层
        self.conv1 = nn.Conv2d(in_channels, out_channels, 
                               kernel_size=3, stride=stride, padding=1, bias=False)
        self.bn1 = nn.BatchNorm2d(out_channels)
        
        # 第二个卷积层
        self.conv2 = nn.Conv2d(out_channels, out_channels,
                               kernel_size=3, stride=1, padding=1, bias=False)
        self.bn2 = nn.BatchNorm2d(out_channels)
        
        # 激活函数
        self.relu = nn.ReLU(inplace=True)
        
        # Shortcut连接
        self.shortcut = nn.Sequential()
        if stride != 1 or in_channels != out_channels * self.expansion:
            # 需要投影的shortcut
            self.shortcut = nn.Sequential(
                nn.Conv2d(in_channels, out_channels * self.expansion,
                         kernel_size=1, stride=stride, bias=False),
                nn.BatchNorm2d(out_channels * self.expansion)
            )
    
    def forward(self, x):
        out = self.relu(self.bn1(self.conv1(x)))  # 第一个卷积 + BN + ReLU
        out = self.bn2(self.conv2(out))            # 第二个卷积 + BN
        
        out += self.shortcut(x)  # 残差连接
        out = self.relu(out)     # 最后激活
        
        return out
```

### Bottleneck残差块

对于更深的网络（如ResNet-50+），使用bottleneck设计：

```python
class BottleneckBlock(nn.Module):
    """
    Bottleneck残差块
    - 1x1卷积降维
    - 3x3卷积
    - 1x1卷积升维
    """
    expansion = 4  # 输出通道数是中间层的4倍
    
    def __init__(self, in_channels, out_channels, stride=1):
        super().__init__()
        
        # 1x1降维
        self.conv1 = nn.Conv2d(in_channels, out_channels, 1, bias=False)
        self.bn1 = nn.BatchNorm2d(out_channels)
        
        # 3x3卷积
        self.conv2 = nn.Conv2d(out_channels, out_channels, 3, 
                               stride=stride, padding=1, bias=False)
        self.bn2 = nn.BatchNorm2d(out_channels)
        
        # 1x1升维
        self.conv3 = nn.Conv2d(out_channels, out_channels * self.expansion, 1, bias=False)
        self.bn3 = nn.BatchNorm2d(out_channels * self.expansion)
        
        self.relu = nn.ReLU(inplace=True)
        
        # Shortcut
        self.shortcut = nn.Sequential()
        if stride != 1 or in_channels != out_channels * self.expansion:
            self.shortcut = nn.Sequential(
                nn.Conv2d(in_channels, out_channels * self.expansion,
                         kernel_size=1, stride=stride, bias=False),
                nn.BatchNorm2d(out_channels * self.expansion)
            )
    
    def forward(self, x):
        out = self.relu(self.bn1(self.conv1(x)))
        out = self.relu(self.bn2(self.conv2(out)))
        out = self.bn3(self.conv3(out))
        
        out += self.shortcut(x)
        out = self.relu(out)
        
        return out
```

## 梯度流分析

### 为什么残差连接能缓解梯度消失

考虑第 $l$ 层到第 $L$ 层的梯度：

$$
\frac{\partial \mathcal{L}}{\partial x_l} = \frac{\partial \mathcal{L}}{\partial x_L} \cdot \frac{\partial x_L}{\partial x_l}
$$

对于残差网络，信号可以绕过非线性层直接传递。设第 $l$ 层输入为 $x_l$，输出为：

$$
x_{l+1} = x_l + \mathcal{F}(x_l)
$$

则：

$$
\frac{\partial x_{l+1}}{\partial x_l} = I + \frac{\partial \mathcal{F}}{\partial x_l}
$$

**关键洞察**：即使 $\frac{\partial \mathcal{F}}{\partial x_l}$ 很小，梯度仍然包含 $I$（单位矩阵），保证了底层梯度不会完全消失！

### 形式化证明

设残差单元 $l$ 到 $l+1$ 的映射为：

$$
x_{l+1} = x_l + \mathcal{F}(x_l, W_l)
$$

反向传播时，从深层 $L$ 到浅层 $l$ 的梯度：

$$
\frac{\partial \mathcal{L}}{\partial x_l} = \frac{\partial \mathcal{L}}{\partial x_L} \cdot \prod_{i=l}^{L-1} \frac{\partial x_{i+1}}{\partial x_i}
= \frac{\partial \mathcal{L}}{\partial x_L} \cdot \prod_{i=l}^{L-1} (I + J_i)
$$

其中 $J_i = \frac{\partial \mathcal{F}}{\partial x_i}$ 是雅可比矩阵。

即使 $J_i$ 很小，$I + J_i$ 的特征值仍然接近1，梯度得以稳定传播。

## Identity Mapping改进

### 问题

原始ResNet的残差单元使用 post-activation（ReLU在加法之后），2016年的论文分析了这种设计并提出改进。

### 改进的残差单元

论文提出了"clean"的身份映射设计：

```
原始 (post-activation):
y_l = h(x_l) + F(f(x_l), W_l)
x_{l+1} = f(y_l)

改进 (pre-activation):
x_{l+1} = x_l + F(W_{l-1} * BN(x_{l-1}) + W_l * BN(x_l))
```

```python
class PreActivationResidualBlock(nn.Module):
    """
    改进的残差块：pre-activation
    BN-ReLU-Conv-BN-ReLU-Conv-Add
    """
    def __init__(self, in_channels, out_channels, stride=1):
        super().__init__()
        
        # Pre-activation顺序
        self.bn1 = nn.BatchNorm2d(in_channels)
        self.relu1 = nn.ReLU(inplace=True)
        self.conv1 = nn.Conv2d(in_channels, out_channels, 3, stride=stride, padding=1, bias=False)
        
        self.bn2 = nn.BatchNorm2d(out_channels)
        self.relu2 = nn.ReLU(inplace=True)
        self.conv2 = nn.Conv2d(out_channels, out_channels, 3, padding=1, bias=False)
        
        self.shortcut = nn.Sequential()
        if stride != 1 or in_channels != out_channels:
            self.shortcut = nn.Sequential(
                nn.Conv2d(in_channels, out_channels, 1, stride=stride, bias=False)
            )
    
    def forward(self, x):
        out = self.relu1(self.bn1(x))
        shortcut = self.shortcut(out) if len(self.shortcut) > 0 else x
        out = self.conv1(out)
        
        out = self.relu2(self.bn2(out))
        out = self.conv2(out)
        
        return out + shortcut
```

### Pre-activation的优势

1. **更好的梯度流**：BN在卷积之前，规范化更稳定
2. **减少过拟合**：实验显示更好的泛化能力
3. **更简洁的计算图**：反向传播更直接

## ResNet架构

### 标准ResNet配置

| 层 | 输出尺寸 | ResNet-18 | ResNet-34 | ResNet-50 | ResNet-101 |
|---|---------|-----------|-----------|------------|------------|
| conv1 | 112×112 | 7×7, 64, stride 2 |
| conv2_x | 56×56 | 3×3 max pool, stride 2 |
| | | 3×3, 64×2 | 3×3, 64×3 | 1×1,64×3<br>3×3,64×3<br>1×1,256×3 | ... |
| conv3_x | 28×28 | 3×3, 128×2 | 3×3, 128×4 | 1×1,128×4<br>3×3,128×4<br>1×1,512×4 | ... |
| conv4_x | 14×14 | 3×3, 256×2 | 3×3, 256×6 | 1×1,256×6<br>3×3,256×6<br>1×1,1024×6 | ... |
| conv5_x | 7×7 | 3×3, 512×2 | 3×3, 512×3 | 1×1,512×3<br>3×3,512×3<br>1×1,2048×3 | ... |
| | 1×1 | avg pool, 1000-d fc, softmax |

### 完整ResNet实现

```python
class ResNet(nn.Module):
    def __init__(self, block, num_blocks, num_classes=1000):
        super().__init__()
        self.in_channels = 64
        
        # 初始卷积层
        self.conv1 = nn.Conv2d(3, 64, kernel_size=7, stride=2, padding=3, bias=False)
        self.bn1 = nn.BatchNorm2d(64)
        self.relu = nn.ReLU(inplace=True)
        self.maxpool = nn.MaxPool2d(kernel_size=3, stride=2, padding=1)
        
        # 残差层
        self.layer1 = self._make_layer(block, 64, num_blocks[0], stride=1)
        self.layer2 = self._make_layer(block, 128, num_blocks[1], stride=2)
        self.layer3 = self._make_layer(block, 256, num_blocks[2], stride=2)
        self.layer4 = self._make_layer(block, 512, num_blocks[3], stride=2)
        
        # 分类头
        self.avgpool = nn.AdaptiveAvgPool2d((1, 1))
        self.fc = nn.Linear(512 * block.expansion, num_classes)
        
        # 参数初始化
        self._initialize_weights()
    
    def _make_layer(self, block, out_channels, num_blocks, stride):
        strides = [stride] + [1] * (num_blocks - 1)
        layers = []
        
        for stride in strides:
            layers.append(block(self.in_channels, out_channels, stride))
            self.in_channels = out_channels * block.expansion
        
        return nn.Sequential(*layers)
    
    def _initialize_weights(self):
        for m in self.modules():
            if isinstance(m, nn.Conv2d):
                nn.init.kaiming_normal_(m.weight, mode='fan_out', nonlinearity='relu')
            elif isinstance(m, nn.BatchNorm2d):
                nn.init.constant_(m.weight, 1)
                nn.init.constant_(m.bias, 0)
    
    def forward(self, x):
        x = self.relu(self.bn1(self.conv1(x)))
        x = self.maxpool(x)
        
        x = self.layer1(x)
        x = self.layer2(x)
        x = self.layer3(x)
        x = self.layer4(x)
        
        x = self.avgpool(x)
        x = torch.flatten(x, 1)
        x = self.fc(x)
        
        return x

# 实例化
def resnet18():
    return ResNet(ResidualBlock, [2, 2, 2, 2])

def resnet50():
    return ResNet(BottleneckBlock, [3, 4, 6, 3])
```

## ResNet变体

### ResNeXt：分组卷积

```python
class ResNeXtBlock(nn.Module):
    """
    ResNeXt残差块：使用分组卷积
    cardinality: 分组数
    width: 每组宽度
    """
    def __init__(self, in_channels, out_channels, cardinality=32, width=4, stride=1):
        super().__init__()
        mid_channels = width * out_channels // 64 * cardinality
        
        self.conv1 = nn.Conv2d(in_channels, mid_channels, 1, bias=False)
        self.bn1 = nn.BatchNorm2d(mid_channels)
        
        self.conv2 = nn.Conv2d(mid_channels, mid_channels, 3, 
                               stride=stride, padding=1, groups=cardinality, bias=False)
        self.bn2 = nn.BatchNorm2d(mid_channels)
        
        self.conv3 = nn.Conv2d(mid_channels, out_channels * 4, 1, bias=False)
        self.bn3 = nn.BatchNorm2d(out_channels * 4)
        
        self.relu = nn.ReLU(inplace=True)
        self.shortcut = self._shortcut(in_channels, out_channels * 4, stride)
    
    def forward(self, x):
        out = self.relu(self.bn1(self.conv1(x)))
        out = self.relu(self.bn2(self.conv2(out)))
        out = self.bn3(self.conv3(out))
        out += self.shortcut(x)
        return self.relu(out)
```

### DenseNet：密集连接

每层接收所有前面层的特征作为输入：

```python
class DenseBlock(nn.Module):
    def __init__(self, in_channels, growth_rate, num_layers):
        super().__init__()
        self.layers = nn.ModuleList()
        
        for i in range(num_layers):
            layer = nn.Sequential(
                nn.BatchNorm2d(in_channels + i * growth_rate),
                nn.ReLU(inplace=True),
                nn.Conv2d(in_channels + i * growth_rate, growth_rate, 3, padding=1, bias=False)
            )
            self.layers.append(layer)
    
    def forward(self, x):
        features = [x]
        for layer in self.layers:
            new_feature = layer(torch.cat(features, dim=1))
            features.append(new_feature)
        return torch.cat(features, dim=1)
```

## 迁移学习使用

```python
from torchvision.models import resnet50, ResNet50_Weights

# 加载预训练模型
model = resnet50(weights=ResNet50_Weights.DEFAULT)

# 冻结底层参数
for param in model.parameters():
    param.requires_grad = False

# 替换最后的分类层
num_features = model.fc.in_features
model.fc = nn.Linear(num_features, num_classes)

# 只训练新添加的层
optimizer = torch.optim.Adam(model.fc.parameters(), lr=0.001)
```

## 参考

[^1]: He et al., "Deep Residual Learning for Image Recognition", CVPR 2016
[^2]: He et al., "Identity Mappings in Deep Residual Networks", ECCV 2016
[^3]: Xie et al., "Aggregated Residual Transformations for Deep Neural Networks", CVPR 2017
[^4]: Huang et al., "Densely Connected Convolutional Networks", CVPR 2017
