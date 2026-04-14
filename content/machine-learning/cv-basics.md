---
title: 计算机视觉基础
date: 2026-04-08
description: 计算机视觉（CV）基础：CNN、目标检测、图像分割、经典架构
tags:
  - computer-vision
  - cnn
  - object-detection
  - image-segmentation
  - deep-learning
draft: false
permalink:
---

## 定义与范畴

计算机视觉（Computer Vision，CV）旨在让计算机理解和处理图像/视频内容。[^1]

### 主要任务

| 任务 | 输入 | 输出 | 经典网络 |
|------|------|------|---------|
| **图像分类** | 图像 | 类别标签 | ResNet, VGG |
| **目标检测** | 图像 | 边界框+类别 | YOLO, Faster R-CNN |
| **语义分割** | 图像 | 像素级类别掩码 | FCN, U-Net |
| **实例分割** | 图像 | 像素级实例掩码 | Mask R-CNN |
| **目标跟踪** | 视频 | 边界框序列 | SORT, DeepSort |

---

## CNN 基础

卷积神经网络（Convolutional Neural Network）是CV的基础架构。

### 卷积层

```python
import torch
import torch.nn as nn

# 单通道卷积
conv = nn.Conv2d(in_channels=1, out_channels=1, kernel_size=3, padding=1)
input_tensor = torch.randn(1, 1, 32, 32)  # (batch, channel, H, W)
output = conv(input_tensor)  # (1, 1, 32, 32)
```

### 卷积操作细节

对于输入 $H \times W$ 的图像，使用 $K \times K$ 卷积核，padding=$P$，stride=$S$：

$$
\text{输出尺寸} = \left\lfloor \frac{H + 2P - K}{S} \right\rfloor + 1
$$

### 池化层

**Max Pooling**：

```python
pool = nn.MaxPool2d(kernel_size=2, stride=2)
# 将 $H \times W$ 缩小为 $H/2 \times W/2$
```

**Average Pooling**：

```python
pool = nn.AvgPool2d(kernel_size=2, stride=2)
```

### 激活函数

```python
# ReLU（最常用）
relu = nn.ReLU()

# Leaky ReLU（避免神经元死亡）
lrelu = nn.LeakyReLU(0.1)

# GELU（Transformer风格）
gelu = nn.GELU()
```

### 经典CNN结构

```python
class SimpleCNN(nn.Module):
    def __init__(self, num_classes=10):
        super().__init__()
        self.features = nn.Sequential(
            # Block 1: 32x32 -> 16x16
            nn.Conv2d(3, 32, kernel_size=3, padding=1),
            nn.BatchNorm2d(32),
            nn.ReLU(),
            nn.MaxPool2d(2),
            
            # Block 2: 16x16 -> 8x8
            nn.Conv2d(32, 64, kernel_size=3, padding=1),
            nn.BatchNorm2d(64),
            nn.ReLU(),
            nn.MaxPool2d(2),
            
            # Block 3: 8x8 -> 4x4
            nn.Conv2d(64, 128, kernel_size=3, padding=1),
            nn.BatchNorm2d(128),
            nn.ReLU(),
            nn.MaxPool2d(2),
        )
        
        self.classifier = nn.Sequential(
            nn.Flatten(),
            nn.Linear(128 * 4 * 4, 256),
            nn.ReLU(),
            nn.Dropout(0.5),
            nn.Linear(256, num_classes)
        )
    
    def forward(self, x):
        x = self.features(x)
        x = self.classifier(x)
        return x
```

---

## 经典架构

### LeNet（1998）

首个成功的CNN，用于手写数字识别：

```
Input(32x32) → Conv(6,5x5) → AvgPool(2x2) → Conv(16,5x5) → AvgPool(2x2) 
→ Conv(120,5x5) → FC(84) → FC(10)
```

### AlexNet（2012）

ImageNet竞赛冠军，引入ReLU和Dropout：

| 层类型 | 输出通道数 |
|--------|-----------|
| Conv + ReLU + MaxPool | 96 |
| Conv + ReLU + MaxPool | 256 |
| Conv + ReLU | 384 |
| Conv + ReLU | 384 |
| Conv + ReLU + MaxPool | 256 |
| FC + ReLU + Dropout | 4096 |
| FC + ReLU + Dropout | 4096 |
| FC | 1000 |

### VGG（2014）

使用更小的3x3卷积堆叠：

```python
# VGG16 结构
vgg16 = nn.Sequential(
    # Block 1
    nn.Conv2d(3, 64, 3, padding=1), nn.ReLU(),
    nn.Conv2d(64, 64, 3, padding=1), nn.ReLU(),
    nn.MaxPool2d(2),  # 224->112
    
    # Block 2
    nn.Conv2d(64, 128, 3, padding=1), nn.ReLU(),
    nn.Conv2d(128, 128, 3, padding=1), nn.ReLU(),
    nn.MaxPool2d(2),  # 112->56
    
    # Block 3
    nn.Conv2d(128, 256, 3, padding=1), nn.ReLU(),
    nn.Conv2d(256, 256, 3, padding=1), nn.ReLU(),
    nn.Conv2d(256, 256, 3, padding=1), nn.ReLU(),
    nn.MaxPool2d(2),  # 56->28
    
    # ... 更多Blocks
)
```

### ResNet（2015）

引入残差连接解决深层网络退化：

$$
y = F(x) + x
$$

```python
class ResidualBlock(nn.Module):
    def __init__(self, in_channels, out_channels, stride=1):
        super().__init__()
        self.conv1 = nn.Conv2d(in_channels, out_channels, 3, stride, 1, bias=False)
        self.bn1 = nn.BatchNorm2d(out_channels)
        self.conv2 = nn.Conv2d(out_channels, out_channels, 3, 1, 1, bias=False)
        self.bn2 = nn.BatchNorm2d(out_channels)
        
        self.shortcut = nn.Sequential()
        if stride != 1 or in_channels != out_channels:
            self.shortcut = nn.Sequential(
                nn.Conv2d(in_channels, out_channels, 1, stride, bias=False),
                nn.BatchNorm2d(out_channels)
            )
    
    def forward(self, x):
        out = torch.relu(self.bn1(self.conv1(x)))
        out = self.bn2(self.conv2(out))
        out += self.shortcut(x)
        return torch.relu(out)
```

---

## 目标检测

### Two-Stage 检测器

#### R-CNN 系列

```python
# R-CNN 流程
def rcnn_detect(image):
    # 1. Selective Search 提取候选框（约2000个）
    proposals = selective_search(image)
    
    # 2. Warp每个候选框为固定大小
    warped_proposals = [warp(p) for p in proposals]
    
    # 3. CNN特征提取
    features = [cnn_extract(p) for p in warped_proposals]
    
    # 4. SVM分类 + 边界框回归
    classes = svm_classify(features)
    boxes = bbox_regress(features, proposals)
    
    return boxes, classes
```

#### Fast R-CNN

改进：整图只过一次CNN：

```python
def fast_rcnn_detect(image):
    # 1. 整图CNN特征提取
    feature_map = cnn(image)  # 共享特征
    
    # 2. ROI Pooling
    rois = selective_search(image)
    roi_features = roi_pooling(feature_map, rois)
    
    # 3. 分类+回归
    class_logits = classifier(roi_features)
    box_deltas = regressor(roi_features)
    
    return class_logits, box_deltas
```

#### Faster R-CNN

引入RPN（Region Proposal Network）：

```python
class RPN(nn.Module):
    def __init__(self):
        super().__init__()
        self.conv = nn.Conv2d(512, 512, 3, padding=1)
        # 2k个分类分数（前景/背景）
        self.cls_logits = nn.Conv2d(512, 18, 1)
        # 4k个边界框回归
        self.bbox_pred = nn.Conv2d(512, 36, 1)
    
    def forward(self, feature_map):
        x = torch.relu(self.conv(feature_map))
        cls_logits = self.cls_logits(x)  # (batch, 18, H, W)
        bbox_pred = self.bbox_pred(x)      # (batch, 36, H, W)
        return cls_logits, bbox_pred
```

### One-Stage 检测器

#### YOLO（You Only Look Once）

将检测问题转化为回归问题：

```python
class YOLOv1(nn.Module):
    def __init__(self, S=7, B=2, C=20):
        super().__init__()
        self.S = S  # 网格大小
        self.B = B  # 每个网格的边界框数
        self.C = C  # 类别数
        
        self.backbone = nn.Sequential(
            nn.Conv2d(3, 64, 7, stride=2, padding=3),
            nn.MaxPool2d(2),
            # ... 更多卷积层
        )
        
        # 输出: S x S x (B*5 + C)
        self.head = nn.Sequential(
            nn.Flatten(),
            nn.Linear(1024 * 7 * 7, 4096),
            nn.ReLU(),
            nn.Dropout(0.5),
            nn.Linear(4096, S * S * (B * 5 + C))
        )
```

#### SSD（Single Shot MultiBox Detector）

多尺度特征图检测：

```python
# 不同尺度的特征图用于检测
feature_maps = [
    conv4_3,   # 38x38 小物体
    conv7,     # 19x19
    conv8_2,   # 10x10
    conv9_2,   # 5x5
    conv10_2,  # 3x3
    conv11_2,  # 1x1 大物体
]
```

### 检测器对比

| 检测器 | mAP | FPS | 特点 |
|--------|-----|-----|------|
| Faster R-CNN | 高 | 低 | 精度高，two-stage |
| YOLOv5 | 中高 | 高 | 实时性好 |
| SSD | 中 | 高 | 多尺度 |
| RetinaNet | 高 | 中 | Focal Loss处理类别不平衡 |

---

## 图像分割

### 语义分割

#### FCN（Fully Convolutional Network）

将分类网络的FC层替换为卷积层：

```python
class FCN8s(nn.Module):
    def __init__(self, num_classes=21):
        super().__init__()
        # VGG16 backbone
        self.conv1 = nn.Sequential(...)
        self.conv2 = nn.Sequential(...)
        self.conv3 = nn.Sequential(...)
        self.conv4 = nn.Sequential(...)
        self.conv5 = nn.Sequential(...)
        
        # FCN特定层
        self.conv6 = nn.Conv2d(4096, 4096, 1)
        self.conv7 = nn.Conv2d(4096, num_classes, 1)
        
        # 上采样
        self.upscore2 = nn.ConvTranspose2d(num_classes, num_classes, 4, stride=2)
        self.upscore8 = nn.ConvTranspose2d(num_classes, num_classes, 16, stride=8)
    
    def forward(self, x):
        # 编码器
        feat3 = self.conv3(x)  # 1/8
        feat4 = self.conv4(feat3)  # 1/16
        feat5 = self.conv5(feat4)  # 1/32
        
        # 解码器
        score = self.conv7(torch.relu(self.conv6(feat5)))
        upscore2 = self.upscore2(score)  # 1/16
        fuse = upscore2 + feat4
        upscore8 = self.upscore8(fuse)  # 1/8
        
        return upscore8
```

### U-Net

编码器-解码器架构，带跳跃连接：

```
   Encoder                Decoder
Input → Conv → Conv → Pool → ...
                    ↕          ↕
                    ... → UpConv → UpConv → Output
```

```python
class UNet(nn.Module):
    def __init__(self, in_channels=1, out_channels=2):
        super().__init__()
        
        # Encoder
        self.enc1 = self._block(in_channels, 64)
        self.enc2 = self._block(64, 128)
        self.enc3 = self._block(128, 256)
        self.enc4 = self._block(256, 512)
        
        # Bottleneck
        self.bottleneck = self._block(512, 1024)
        
        # Decoder
        self.up4 = nn.ConvTranspose2d(1024, 512, 2, stride=2)
        self.dec4 = self._block(1024, 512)
        self.up3 = nn.ConvTranspose2d(512, 256, 2, stride=2)
        self.dec3 = self._block(512, 256)
        
        # Output
        self.out = nn.Conv2d(256, out_channels, 1)
    
    def forward(self, x):
        # Encoder
        e1 = self.enc1(x)
        e2 = self.enc2(nn.MaxPool2d(2)(e1))
        e3 = self.enc3(nn.MaxPool2d(2)(e2))
        e4 = self.enc4(nn.MaxPool2d(2)(e3))
        
        # Bottleneck
        b = self.bottleneck(nn.MaxPool2d(2)(e4))
        
        # Decoder with skip connections
        d4 = self.dec4(torch.cat([self.up4(b), e4], dim=1))
        d3 = self.dec3(torch.cat([self.up3(d4), e3], dim=1))
        
        return self.out(d3)
```

### 实例分割

#### Mask R-CNN

在Faster R-CNN基础上添加分割头：

```python
class MaskRCNN(nn.Module):
    def __init__(self):
        super().__init__()
        # Backbone: FPN
        self.backbone = FPN()
        
        # RPN
        self.rpn = RPN()
        
        # RoI Align
        self.roi_align = RoIAlign(7, 7, 2.0)
        
        # 检测头
        self.box_head = TwoMLPHead(1024, 1024)
        self.box_classifier = Linear(1024, num_classes)
        self.box_regressor = Linear(1024, num_classes * 4)
        
        # 分割头
        self.mask_head = MaskHead()
        self.mask_predictor = MaskPredictor(256, 256, num_classes)
    
    def forward(self, images, targets=None):
        features = self.backbone(images)
        proposals = self.rpn(features)
        
        if self.training:
            losses = {}
            # ... 计算各项损失
            return losses
        
        # 推理
        class_logits, box_regression = self.box_head(proposals)
        masks = self.mask_predictor(mask_features)
        return proposals, class_logits, box_regression, masks
```

### 分割任务对比

| 方法 | 精度 | 速度 | 实例区分 |
|------|------|------|---------|
| FCN | 中 | 快 | 否 |
| U-Net | 高 | 中 | 否 |
| DeepLab | 高 | 中 | 否 |
| Mask R-CNN | 最高 | 慢 | 是 |

---

## 目标跟踪

### 卡尔曼滤波

```python
class KalmanFilter:
    def __init__(self, dt=1.0):
        # 状态: [x, y, vx, vy]
        self.F = np.array([[1, 0, dt, 0],
                           [0, 1, 0, dt],
                           [0, 0, 1, 0],
                           [0, 0, 0, 1]])
        
        self.H = np.array([[1, 0, 0, 0],
                           [0, 1, 0, 0]])
        
        self.Q = np.eye(4) * 0.01  # 过程噪声
        self.R = np.eye(2) * 0.1   # 测量噪声
    
    def predict(self, x, P):
        x = self.F @ x
        P = self.F @ P @ self.F.T + self.Q
        return x, P
    
    def update(self, x, P, z):
        S = self.H @ P @ self.H.T + self.R
        K = P @ self.H.T @ np.linalg.inv(S)
        x = x + K @ (z - self.H @ x)
        P = (np.eye(4) - K @ self.H) @ P
        return x, P
```

### SORT（Simple Online and Realtime Tracking）

```python
def sort_update(trackers, detections, features):
    # 1. 预测所有跟踪器状态
    for track in trackers:
        track.predict()
    
    # 2. 匈牙利匹配
    cost_matrix = compute_iou_cost(trackers, detections)
    matches, unmatched_tracks, unmatched_detections = hungarian_matching(cost_matrix)
    
    # 3. 更新匹配的跟踪器
    for track_idx, det_idx in matches:
        trackers[track_idx].update(detections[det_idx])
    
    # 4. 创建新跟踪器
    for det_idx in unmatched_detections:
        create_new_tracker(detections[det_idx])
    
    return trackers
```

---

## 应用场景

### 人脸识别

```python
# FaceNet 嵌入
class FaceNet(nn.Module):
    def __init__(self):
        super().__init__()
        self.backbone = InceptionResNetV1()
    
    def forward(self, x):
        # L2归一化
        x = self.backbone(x)
        return F.normalize(x, p=2, dim=1)

# 人脸验证
def verify_face(img1, img2, model, threshold=0.7):
    emb1 = model(img1)
    emb2 = model(img2)
    similarity = F.cosine_similarity(emb1, emb2)
    return similarity > threshold
```

### 医疗影像

- **X光/CT分析**：肺炎检测、骨折检测
- **MRI分析**：脑肿瘤分割
- **病理切片**：癌细胞检测

### 自动驾驶

- 车道线检测
- 交通标志识别
- 行人/车辆检测
- 语义分割（道路、可行驶区域）

---

## 参考资料

[^1]: Deep Learning for Computer Vision - Adrian Rosebrock. https://pyimagesearch.com/

[^2]: Mask R-CNN - He et al. (2017). https://arxiv.org/abs/1703.06870

[^3]: U-Net: Convolutional Networks for Biomedical Image Segmentation - Ronneberger et al. (2015). https://arxiv.org/abs/1505.04597
