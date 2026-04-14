---
title: 机器学习基础
date: 2026-04-07
description: 机器学习（Machine Learning）三大范式：监督学习、无监督学习、强化学习
tags:
  - machine-learning
  - supervised-learning
  - unsupervised-learning
  - reinforcement-learning
draft: false
permalink:
---

## 定义与范畴

机器学习（Machine Learning，ML）是人工智能的一个子领域，关注于通过数据学习模式并做出预测或决策的算法。[^1]

根据训练数据的结构和学习过程中反馈信号的不同，机器学习主要分为三大范式：

| 范式 | 训练数据 | 反馈信号 | 典型任务 |
|------|----------|----------|----------|
| **监督学习** | 标注的输入-输出对 $(x, y)$ | 预测误差（损失函数） | 分类、回归 |
| **无监督学习** | 仅输入 $x$，无标签 | 内部结构指标 | 聚类、降维 |
| **强化学习** | 状态-动作-奖励序列 | 标量奖励信号 | 序列决策、控制 |

### 第四范式：半监督与自监督学习

- **半监督学习**：结合少量标注数据和大量无标注数据
- **自监督学习**：从无标注数据中生成监督信号（如预测被遮挡的部分）

## 监督学习

监督学习使用标注的训练数据，算法学习从输入到输出的映射函数。

### 核心概念

设训练数据集为 $D = \{(x_1, y_1), (x_2, y_2), \dots, (x_n, y_n)\}$，算法通过最小化损失函数 $\mathcal{L}$ 来调整模型参数：

$$
\theta^* = \arg\min_\theta \sum_{i=1}^{n} \mathcal{L}(f(x_i; \theta), y_i)
$$

### 经典算法

| 算法 | 类型 | 适用场景 |
|------|------|----------|
| 线性/逻辑回归 | 回归/分类 | 基线模型、线性关系 |
| 决策树 | 分类/回归 | 可解释性要求高 |
| 支持向量机 | 分类 | 高维空间、小样本 |
| 神经网络 | 分类/回归 | 复杂非线性关系 |

### 分类问题示例：鸢尾花分类

```python
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier

# 加载数据
iris = load_iris()
X, y = iris.data, iris.target

# 划分训练集和测试集
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# 训练随机森林
clf = RandomForestClassifier(n_estimators=100)
clf.fit(X_train, y_train)

# 评估
accuracy = clf.score(X_test, y_test)
print(f"准确率: {accuracy:.2f}")
```

---

## 无监督学习

无监督学习在无标签数据上发现潜在结构。

### 聚类

**K-means 算法**是最经典的聚类方法：

```python
import numpy as np
from sklearn.cluster import KMeans

# 生成示例数据
np.random.seed(42)
X = np.concatenate([
    np.random.randn(50, 2) + [2, 2],
    np.random.randn(50, 2) + [-2, -2]
])

# K-means 聚类
kmeans = KMeans(n_clusters=2, random_state=42)
labels = kmeans.fit_predict(X)

# 聚类中心
centers = kmeans.cluster_centers_
```

### 降维

**主成分分析（PCA）**是最常用的线性降维方法：

```python
from sklearn.decomposition import PCA
from sklearn.preprocessing import StandardScaler

# 标准化
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)

# PCA降维到2维
pca = PCA(n_components=2)
X_pca = pca.fit_transform(X_scaled)

print(f"解释方差比例: {pca.explained_variance_ratio_}")
```

---

## 强化学习

强化学习中，智能体（Agent）通过与环境交互学习策略。

### 马尔可夫决策过程（MDP）

MDP由四元组 $(S, A, P, R)$ 定义：

- $S$：状态空间
- $A$：动作空间
- $P$：状态转移概率 $P(s'|s, a)$
- $R$：奖励函数 $R(s, a, s')$

### Q-learning 算法

```python
import numpy as np

class QLearningAgent:
    def __init__(self, n_states, n_actions, learning_rate=0.1, gamma=0.99):
        self.q_table = np.zeros((n_states, n_actions))
        self.lr = learning_rate
        self.gamma = gamma
        self.n_actions = n_actions
    
    def select_action(self, state, epsilon=0.1):
        if np.random.random() < epsilon:
            return np.random.randint(self.n_actions)
        return np.argmax(self.q_table[state])
    
    def update(self, state, action, reward, next_state):
        best_next_q = np.max(self.q_table[next_state])
        td_target = reward + self.gamma * best_next_q
        td_error = td_target - self.q_table[state, action]
        self.q_table[state, action] += self.lr * td_error
```

### 探索与利用的权衡

- **探索（Exploration）**：尝试新的动作以发现更好的策略
- **利用（Exploitation）**：使用当前已知的最优策略

常用方法：$\epsilon$-贪心、Thompson采样、UCB。

---

## 机器学习项目流程

### Phase 1：问题定义

- 确定预测目标（分类/回归/排序）
- 定义评估指标（准确率、F1、AUC、RMSE）
- 建立基线模型

### Phase 2：数据采集与探索

- 收集原始数据
- 分析分布、缺失值、类别不平衡
- 识别特征类型

### Phase 3：数据预处理

```python
# 常见预处理步骤
from sklearn.preprocessing import StandardScaler, LabelEncoder
from sklearn.impute import SimpleImputer

# 缺失值处理
imputer = SimpleImputer(strategy='mean')
X_imputed = imputer.fit_transform(X)

# 特征缩放（对SVM、神经网络、KNN重要，对树模型不重要）
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X_imputed)

# 类别编码
encoder = LabelEncoder()
y_encoded = encoder.fit_transform(y)
```

### Phase 4：模型选择与训练

- 划分训练集、验证集、测试集
- 使用交叉验证调参
- 防止数据泄露

### Phase 5：评估

```python
from sklearn.metrics import classification_report, confusion_matrix

# 分类报告
print(classification_report(y_test, y_pred))

# 混淆矩阵
cm = confusion_matrix(y_test, y_pred)
```

### Phase 6：部署与监控

- 设置数据漂移检测
- 定义性能下降阈值
- 建立重训练触发机制

---

## 常见误区

### 1. 深度学习 ≠ 机器学习

深度学习是机器学习的一个子集，由多层神经网络组成。机器学习还包括线性回归、决策树、SVM等没有"深度"概念的方法。

### 2. 无监督学习不只在无标签时使用

无监督技术也用于探索性数据分析、异常检测、特征工程。

### 3. 过拟合与欠拟合

- **过拟合**：模型记忆训练数据，在测试集表现差
- **欠拟合**：模型容量不足，连训练数据都拟合不好

```python
# 使用交叉验证检测过拟合
from sklearn.model_selection import cross_val_score

scores = cross_val_score(model, X, y, cv=5)
print(f"交叉验证分数: {scores.mean():.3f} (+/- {scores.std()*2:.3f})")
```

### 4. 模型并非客观

训练数据反映其收集过程，可能包含历史偏差。NIST AI 100-1明确将偏差列为需要测量和缓解的风险类别。

---

## 参考资料

[^1]: Machine Learning Fundamentals - Computer Science Authority. https://computerscienceauthority.com/machine-learning-fundamentals.html
