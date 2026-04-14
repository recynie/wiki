---
title: 卷积神经网络与图像分类
date: 2026-04-13
description: CNN原理、经典架构与图像分类实战
tags:
  - machine-learning
  - deep-learning
  - cnn
  - computer-vision
draft: false
permalink:
---

# 卷积神经网络与图像分类

卷积神经网络（CNN）是深度学习在计算机视觉领域的基石，通过局部感受野和权重共享高效处理图像数据。

## CNN核心组件

### 卷积层

卷积操作通过滤波器（卷积核）在输入图像上滑动提取特征：

```python
import torch
import torch.nn as nn

class Conv2d(nn.Module):
    def __init__(self, in_channels, out_channels, kernel_size=3, stride=1, padding=1):
        super().__init__()
        self.conv = nn.Conv2d(in_channels, out_channels, kernel_size, stride, padding)
    
    def forward(self, x):
        return self.conv(x)
```

**关键参数**：
- **卷积核大小**：3×3、5×5、7×7，更大的核提供更大感受野
- **步长（stride）**：控制滑动间隔
- **填充（padding）**：保持空间尺寸，边缘信息不丢失

### 激活函数

```python
# ReLU是最常用的激活函数
x = torch.relu(conv_output)

# Leaky ReLU避免神经元死亡
x = torch.nn.functional.leaky_relu(conv_output, negative_slope=0.01)
```

### 池化层

**最大池化（Max Pooling）**：保留显著特征，减小尺寸

```python
# 2×2最大池化，步长2
pool = nn.MaxPool2d(kernel_size=2, stride=2)
```

**平均池化（Average Pooling）**：平滑特征，减少噪声

### 全连接层

将特征图展平后进行分类：

```python
fc = nn.Linear(feature_dim, num_classes)
```

## 经典CNN架构

### LeNet-5（1998）

首个成功的手写数字识别网络：2个卷积层 + 2个池化层 + 2个全连接层。

### AlexNet（2012）

ImageNet比赛冠军，引入ReLU激活函数和Dropout正则化。

```python
# AlexNet结构简化
alexnet = nn.Sequential(
    nn.Conv2d(3, 64, kernel_size=11, stride=4, padding=2),
    nn.ReLU(),
    nn.MaxPool2d(kernel_size=3, stride=2),
    
    nn.Conv2d(64, 192, kernel_size=5, padding=2),
    nn.ReLU(),
    nn.MaxPool2d(kernel_size=3, stride=2),
    
    nn.Conv2d(192, 384, kernel_size=3, padding=1),
    nn.Conv2d(384, 256, kernel_size=3, padding=1),
    nn.Conv2d(256, 256, kernel_size=3, padding=1),
    nn.ReLU(),
    nn.MaxPool2d(kernel_size=3, stride=2),
    
    nn.AdaptiveAvgPool2d((6, 6)),  # 全局平均池化
    nn.Flatten(),
    nn.Linear(256 * 6 * 6, 4096),
    nn.ReLU(),
    nn.Dropout(0.5),
    nn.Linear(4096, num_classes)
)
```

### VGGNet（2014）

使用更小的3×3卷积核堆叠加深网络（16-19层），证明网络深度重要性。

### ResNet（2015）

引入残差连接（Skip Connection）解决深层网络梯度消失问题：

```python
class ResidualBlock(nn.Module):
    def __init__(self, channels):
        super().__init__()
        self.conv1 = nn.Conv2d(channels, channels, 3, padding=1)
        self.conv2 = nn.Conv2d(channels, channels, 3, padding=1)
    
    def forward(self, x):
        residual = x
        out = torch.relu(self.conv1(x))
        out = self.conv2(out)
        out += residual  # 残差连接
        return torch.relu(out)
```

## 图像分类实战

### 数据预处理

```python
from torchvision import transforms

train_transform = transforms.Compose([
    transforms.RandomResizedCrop(224),
    transforms.RandomHorizontalFlip(),
    transforms.RandomRotation(15),
    transforms.ColorJitter(brightness=0.2, contrast=0.2),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], 
                         std=[0.229, 0.224, 0.225])
])

test_transform = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406],
                         std=[0.229, 0.224, 0.225])
])
```

### 训练循环

```python
def train_epoch(model, dataloader, criterion, optimizer, device):
    model.train()
    total_loss = 0
    correct = 0
    
    for images, labels in dataloader:
        images, labels = images.to(device), labels.to(device)
        
        optimizer.zero_grad()
        outputs = model(images)
        loss = criterion(outputs, labels)
        loss.backward()
        optimizer.step()
        
        total_loss += loss.item()
        correct += (outputs.argmax(1) == labels).sum().item()
    
    return total_loss / len(dataloader), correct / len(dataloader.dataset)
```

### 使用预训练模型

```python
from torchvision.models import resnet50, ResNet50_Weights

# 加载预训练权重
model = resnet50(weights=ResNet50_Weights.DEFAULT)

# 迁移学习：冻结底层权重
for param in model.parameters():
    param.requires_grad = False

# 替换分类头
model.fc = nn.Linear(model.fc.in_features, num_classes)
for param in model.fc.parameters():
    param.requires_grad = True
```

## 经典数据集

| 数据集 | 规模 | 类别数 | 主要用途 |
|--------|------|--------|----------|
| MNIST | 70,000 | 10 | 入门基准 |
| CIFAR-10 | 60,000 | 10 | 物体分类 |
| ImageNet | 14,000,000 | 21,841 | 大规模识别 |

## 常见问题与解决方案

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 过拟合 | 数据不足，网络过深 | 数据增强，Dropout，正则化 |
| 收敛慢 | 学习率不合适 | 学习率衰减，warmup |
| 梯度消失 | 网络过深 | ResNet残差连接，BatchNorm |

## 参考

[^1]: LeCun et al., Gradient-Based Learning Applied to Document Recognition
[^2]: He et al., Deep Residual Learning for Image Recognition

