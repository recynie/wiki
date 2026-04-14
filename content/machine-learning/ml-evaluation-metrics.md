---
title: 机器学习评估指标
date: 2026-04-13
description: 机器学习模型评估的核心指标与选择指南
tags:
  - machine-learning
  - evaluation
  - metrics
draft: false
permalink:
---

# 机器学习评估指标

模型评估是机器学习工作流中最关键的环节之一。选择合适的评估指标直接影响模型优化方向和最终性能。

## 回归任务指标

### 均方误差（MSE）

$$
MSE = \frac{1}{n}\sum_{i=1}^{n}(y_i - \hat{y}_i)^2
$$

**优点**：对大误差惩罚更重，梯度计算友好
**缺点**：对异常值敏感，单位依赖原始数据

```python
import numpy as np
from sklearn.metrics import mean_squared_error

y_true = np.array([3.0, -0.5, 2.0, 7.0])
y_pred = np.array([2.5, 0.0, 2.1, 8.0])
mse = mean_squared_error(y_true, y_pred)
```

### 均方根误差（RMSE）

$$
RMSE = \sqrt{MSE}
$$

与MSE相同单位，更易解释。

### 平均绝对误差（MAE）

$$
MAE = \frac{1}{n}\sum_{i=1}^{n}|y_i - \hat{y}_i|
$$

**优点**：对异常值鲁棒，梯度恒定
**缺点**：在零点不可导

### R² 决定系数

$$
R^2 = 1 - \frac{\sum(y_i - \hat{y}_i)^2}{\sum(y_i - \bar{y})^2}
$$

取值范围通常为（-∞, 1]，1表示完美拟合。

## 分类任务指标

### 准确率（Accuracy）

$$
Accuracy = \frac{TP + TN}{TP + TN + FP + FN}
$$

**适用场景**：类别均衡数据集
**局限性**：类别不均衡时会产生误导

### 精确率（Precision）

$$
Precision = \frac{TP}{TP + FP}
$$

**适用场景**：假阳性代价高的场景（垃圾邮件检测、医疗诊断）

### 召回率（Recall）

$$
Recall = \frac{TP}{TP + FN}
$$

**适用场景**：假阴性代价高的场景（欺诈检测、癌症筛查）

### F1 分数

$$
F1 = 2 \times \frac{Precision \times Recall}{Precision + Recall}
$$

精确率和召回率的调和平均，适用于类别不均衡场景。

```python
from sklearn.metrics import precision_score, recall_score, f1_score

y_true = [0, 1, 1, 1, 0, 1]
y_pred = [0, 0, 1, 1, 0, 1]

precision = precision_score(y_true, y_pred)
recall = recall_score(y_true, y_pred)
f1 = f1_score(y_true, y_pred)
```

### AUC-ROC

ROC曲线下面积，衡量分类器区分能力，取值范围[0, 1]。

- AUC = 0.5：随机猜测
- AUC = 1.0：完美分类
- AUC > 0.9：优秀性能

```python
from sklearn.metrics import roc_auc_score

y_true = [0, 1, 1, 0, 1]
y_scores = [0.1, 0.4, 0.35, 0.8, 0.7]
auc = roc_auc_score(y_true, y_scores)
```

### PR曲线

精确率-召回率曲线，适用于正例稀少的数据集（推荐系统、异常检测）。

## 指标选择指南

| 场景 | 推荐指标 | 原因 |
|------|---------|------|
| 类别均衡分类 | Accuracy | 各类别同等重要 |
| 类别不均衡 | F1 / AUC | 避免多数类主导 |
| 医学诊断 | Recall | 漏检代价高 |
| 垃圾邮件过滤 | Precision | 误杀代价高 |
| 推荐系统 | AUC / MAP | 排序质量优先 |
| 异常检测 | F1 / AUC-PR | 正例稀少 |

## 交叉验证

### K折交叉验证

```python
from sklearn.model_selection import cross_val_score
from sklearn.ensemble import RandomForestClassifier

model = RandomForestClassifier(n_estimators=100)
scores = cross_val_score(model, X, y, cv=5, scoring='f1')
print(f"F1 scores: {scores}")
print(f"Mean: {scores.mean():.3f} (+/- {scores.std() * 2:.3f})")
```

### 分层K折

保持每折中类别比例与整体一致：

```python
from sklearn.model_selection import StratifiedKFold

skf = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
```

## 过拟合与欠拟合检测

- **训练集表现好，测试集表现差**：过拟合 → 增加正则化、数据增强、简化模型
- **训练集和测试集表现都差**：欠拟合 → 增加模型复杂度、添加特征

## 参考

[^1]: scikit-learn.metrics文档 — https://scikit-learn.org/stable/modules/model_evaluation.html
[^2]: Google Machine Learning Crash Course — https://developers.google.com/machine-learning

