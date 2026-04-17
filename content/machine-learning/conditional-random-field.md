---
title: 条件随机场
date: 2026-04-17
description: 条件随机场（CRF）的原理与序列标注应用
tags:
  - machine-learning
  - crf
  - sequence-labeling
  - nlp
draft: false
permalink:
---

# 条件随机场

条件随机场（Conditional Random Field, CRF）是一种判别式概率无向图模型，常用于序列标注任务，如词性标注、命名实体识别等。

## 生成模型 vs 判别模型

### 生成模型

建模联合分布 $P(X, Y)$，然后通过贝叶斯推断得到 $P(Y|X)$。

代表：朴素贝叶斯、隐马尔可夫模型（HMM）

**优点**：可以利用先验知识，学习数据分布
**缺点**：需要建模 $P(X|Y)$，当特征复杂时难以处理

### 判别模型

直接建模条件分布 $P(Y|X)$，关注如何区分不同类别。

代表：逻辑回归、条件随机场

**优点**：直接优化分类边界，更灵活
**缺点**：需要大量标注数据

### 为什么需要CRF

HMM的局限性：
1. 强假设：观测独立、状态转移仅依赖前一状态
2. 难以利用丰富的特征（如词形、后缀、大小写）

CRF优势：
1. 可以利用任意特征
2. 考虑标签之间的依赖关系
3. 避免标签偏置（Label Bias）问题

## CRF的数学定义

### 无向图模型

设输入序列 $X = (x_1, x_2, ..., x_n)$，输出序列 $Y = (y_1, y_2, ..., y_n)$。

CRF是定义在 $X$ 和 $Y$ 上的条件概率分布：

$$
P(Y | X) = \frac{1}{Z(X)} \exp\left(\sum_{i, k} \lambda_k t_k(y_{i-1}, y_i, X, i) + \sum_{i, l} \mu_l s_l(y_i, X, i)\right)
$$

其中：
- $t_k$：转移特征函数（描述标签对之间的关系）
- $s_l$：状态特征函数（描述标签与观测的关系）
- $\lambda_k, \mu_l$：对应的权重
- $Z(X)$：归一化因子（配分函数）

### 线性链CRF

假设标签只与相邻标签有关：

$$
P(y | x) = \frac{1}{Z(x)} \exp\left(\sum_{i=1}^{n} \sum_{k} \lambda_k f_k(y_{i-1}, y_i, x, i)\right)
$$

简化形式：

$$
P(y | x) = \frac{1}{Z(x)} \exp\left(\sum_{i=1}^{n} \theta \cdot \phi(x, i, y_{i-1}, y_i)\right)
$$

## 特征函数设计

### 状态特征函数

描述当前标签与观测的关系：

```python
def state_features(x, i, y_i):
    """状态特征：捕捉标签与观测的关系"""
    features = {
        # 词性特征
        f'word_is_upper_{y_i}': x[i][0].isupper(),
        f'word_ends_with_ing_{y_i}': x[i].endswith('ing'),
        f'word_is_digit_{y_i}': x[i].isdigit(),
        f'word_length_{y_i}': len(x[i]) > 10,
        
        # 上下文特征
        f'prev_word_{y_i}': x[i-1] if i > 0 else '<START>',
        f'next_word_{y_i}': x[i+1] if i < len(x)-1 else '<END>',
        
        # 位置特征
        f'position_start_{y_i}': i == 0,
        f'position_end_{y_i}': i == len(x) - 1,
    }
    return features
```

### 转移特征函数

描述标签之间的转移关系：

```python
def transition_features(y_prev, y_curr):
    """转移特征：捕捉标签之间的依赖"""
    features = {
        # 常见转移模式
        f'trans_{y_prev}_to_{y_curr}': True,
        
        # 特定转移约束
        f'det_noun': (y_prev == 'DET' and y_curr == 'NOUN'),
        f'verb_adj': (y_prev == 'VERB' and y_curr == 'ADJ'),
    }
    return features
```

### 特征向量表示

```python
def extract_features(x, i, y_prev, y_curr):
    """提取完整特征"""
    features = {}
    
    # 状态特征
    features.update(state_features(x, i, y_curr))
    
    # 转移特征
    if y_prev is not None:
        features.update(transition_features(y_prev, y_curr))
    
    return features
```

## Viterbi解码

### 问题定义

给定观测序列 $x$，找到最优标签序列：

$$
y^* = \arg\max_{y} P(y | x)
$$

### 动态规划求解

```python
import numpy as np

class ViterbiDecoder:
    def __init__(self, tagset):
        self.tagset = tagset
        self.num_tags = len(tagset)
        self.tag_to_idx = {tag: i for i, tag in enumerate(tagset)}
    
    def decode(self, x, start_scores, trans_scores, emit_scores):
        """
        x: 输入序列
        start_scores: 初始发射分数 [num_tags]
        trans_scores: 转移分数 [num_tags, num_tags]
        emit_scores: 发射分数 [seq_len, num_tags]
        """
        T = len(x)
        num_tags = self.num_tags
        
        # Viterbi表
        viterbi = np.zeros((T, num_tags))
        # 回溯指针
        backpointer = np.zeros((T, num_tags), dtype=int)
        
        # 初始化
        viterbi[0] = start_scores + emit_scores[0]
        
        # 递推
        for t in range(1, T):
            for j in range(num_tags):
                # 从所有前驱状态转移到当前状态
                scores = viterbi[t-1] + trans_scores[:, j] + emit_scores[t, j]
                viterbi[t, j] = np.max(scores)
                backpointer[t, j] = np.argmax(scores)
        
        # 回溯
        best_path = np.zeros(T, dtype=int)
        best_path[T-1] = np.argmax(viterbi[T-1])
        
        for t in range(T-2, -1, -1):
            best_path[t] = backpointer[t+1, best_path[t+1]]
        
        # 转换为标签
        idx_to_tag = {i: tag for tag, i in self.tag_to_idx.items()}
        return [idx_to_tag[i] for i in best_path]
```

## 参数学习

### 极大似然估计

目标：最大化条件似然

$$
\mathcal{L}(\theta) = \sum_{i=1}^{M} \log P(y^{(i)} | x^{(i)}; \theta)
$$

对数似然函数：

$$
\ell(\theta) = \sum_{i=1}^{M} \left[ \sum_{k} \theta_k f_k(y^{(i)}, x^{(i)}) - \log Z(x^{(i)}) \right]
$$

### 梯度计算

$$
\frac{\partial \ell}{\partial \theta_k} = \sum_{i=1}^{M} \left[ f_k(y^{(i)}, x^{(i)}) - \mathbb{E}_{P(y|x^{(i)})}[f_k]\right]
$$

其中 $\mathbb{E}_{P(y|x)}[f_k]$ 是期望特征数，需要通过动态规划计算。

### L-BFGS优化

```python
from scipy.optimize import minimize
import numpy as np

class LinearChainCRF:
    def __init__(self, tagset, feature_extractor):
        self.tagset = tagset
        self.feature_extractor = feature_extractor
        self.num_tags = len(tagset)
        self.num_features = None
        self.weights = None
    
    def _compute_scores(self, x, y):
        """计算给定序列的特征分数"""
        score = 0.0
        for i in range(len(x)):
            if i == 0:
                # 起始标签
                features = self.feature_extractor(x, i, '<START>', y[i])
            else:
                features = self.feature_extractor(x, i, y[i-1], y[i])
            
            for feat_id, feat_val in features.items():
                score += self.weights[feat_id] * feat_val
        
        return score
    
    def _forward(self, x, weights):
        """前向算法计算Z(x)"""
        T = len(x)
        alpha = np.zeros((T, self.num_tags))
        
        # 初始化
        alpha[0] = self._get_emission_scores(x, 0, weights)
        
        # 递推
        for t in range(1, T):
            for j in range(self.num_tags):
                scores = alpha[t-1] + self._get_transition_scores(x, t, j, weights)
                alpha[t, j] = np.max(scores) + np.log(np.sum(np.exp(scores - np.max(scores))))
        
        return np.sum(alpha[T-1])
    
    def fit(self, X, Y):
        """训练CRF"""
        # 构建特征映射
        feature_map = {}
        feature_id = 0
        
        for x, y in zip(X, Y):
            for i in range(len(x)):
                y_prev = y[i-1] if i > 0 else '<START>'
                y_curr = y[i]
                features = self.feature_extractor(x, i, y_prev, y_curr)
                
                for feat in features:
                    if feat not in feature_map:
                        feature_map[feat] = feature_id
                        feature_id += 1
        
        self.num_features = feature_id
        self.weights = np.zeros(self.num_features)
        
        # 优化
        def objective(weights):
            return -self._pseudo_log_likelihood(X, Y, weights, feature_map)
        
        def gradient(weights):
            return -self._pseudo_gradient(X, Y, weights, feature_map)
        
        result = minimize(objective, self.weights, method='L-BFGS-B', jac=gradient)
        self.weights = result.x
        self.feature_map = feature_map
    
    def predict(self, x):
        """预测最优标签序列"""
        return self._viterbi(x)
```

## PyTorch-CRF实现

### 安装

```bash
pip install torchcrf
```

### 基本使用

```python
from torchcrf import CRF
import torch

# 参数
num_tags = 5  # 标签数量
batch_size = 2
seq_length = 3

# 创建CRF层
crf = CRF(num_tags, batch_first=True)

# 模拟数据
emissions = torch.randn(batch_size, seq_length, num_tags)
tags = torch.randint(0, num_tags, (batch_size, seq_length))

# 计算损失（负对数似然）
loss = -crf(emissions, tags)
print(f"Loss: {loss.item():.4f}")

# 解码（Viterbi）
best_paths = crf.decode(emissions)
print(f"Best paths: {best_paths}")

# 发射分数和标签掩码
mask = torch.tensor([[True, True, True], [True, True, False]])
loss_masked = -crf(emissions, tags, mask=mask, reduction='mean')
```

### BiLSTM-CRF模型

```python
import torch
import torch.nn as nn
from torchcrf import CRF

class BiLSTM_CRF(nn.Module):
    def __init__(self, vocab_size, tag_to_idx, embedding_dim=128, hidden_dim=256):
        super().__init__()
        
        self.embedding = nn.Embedding(vocab_size, embedding_dim, padding_idx=0)
        self.lstm = nn.LSTM(embedding_dim, hidden_dim // 2, 
                           batch_first=True, bidirectional=True)
        self.hidden2tag = nn.Linear(hidden_dim, len(tag_to_idx))
        self.crf = CRF(len(tag_to_idx), batch_first=True)
        
        self.tag_to_idx = tag_to_idx
        self.idx_to_tag = {i: t for t, i in tag_to_idx.items()}
    
    def forward(self, sentences, tags=None, mask=None):
        """
        sentences: (batch_size, seq_len)
        tags: (batch_size, seq_len)
        mask: (batch_size, seq_len)
        """
        # 嵌入
        embeddings = self.embedding(sentences)
        
        # BiLSTM
        lstm_out, _ = self.lstm(embeddings)  # (batch, seq_len, hidden_dim)
        
        # 发射分数
        emissions = self.hidden2tag(lstm_out)
        
        if tags is not None:
            # 训练模式：返回负对数似然
            loss = -self.crf(emissions, tags, mask=mask)
            return loss
        else:
            # 推理模式：返回解码结果
            return self.crf.decode(emissions, mask=mask)
```

### 训练循环

```python
def train_epoch(model, dataloader, optimizer, device):
    model.train()
    total_loss = 0
    
    for batch in dataloader:
        sentences, tags, mask = batch
        sentences = sentences.to(device)
        tags = tags.to(device)
        mask = mask.to(device)
        
        optimizer.zero_grad()
        loss = model(sentences, tags, mask)
        loss.backward()
        
        # 梯度裁剪
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=5.0)
        
        optimizer.step()
        total_loss += loss.item()
    
    return total_loss / len(dataloader)

def evaluate(model, dataloader, tag_to_idx, device):
    model.eval()
    correct = 0
    total = 0
    
    with torch.no_grad():
        for batch in dataloader:
            sentences, tags, mask = batch
            sentences = sentences.to(device)
            tags = tags.to(device)
            mask = mask.to(device)
            
            predictions = model(sentences, mask=mask)
            
            # 计算准确率
            for pred, gold, m in zip(predictions, tags, mask):
                for p, g, valid in zip(pred, gold, m):
                    if valid.item():
                        total += 1
                        if p == g.item():
                            correct += 1
    
    return correct / total if total > 0 else 0
```

## 应用场景

### POS词性标注

| 输入 | The | cat | sits | on | the | mat |
|------|-----|-----|------|----|----|-----|
| 输出 | DT | NN | VBZ | IN | DT | NN |

### 命名实体识别（NER）

```python
# NER标签体系（BIO标注）
B-PER, I-PER, B-LOC, I-LOC, B-ORG, I-ORG, O

# 示例
# Tom works at Microsoft .
# B-PER I-PER O B-ORG I-ORG O
```

### 分词

```python
# 分词标注（BMES）
# 微软/是/一家/科技/公司/
# B M E S B M E S

# 或简单的IO标注
# 微软/是/一家/科技/公司/
# I I O I I I
```

## CRF vs 其他序列标注方法

| 方法 | 标签依赖 | 全局最优 | 计算复杂度 |
|------|----------|----------|------------|
| 朴素分类 | 无 | 否 | O(n) |
| HMM | 相邻标签 | 是 | O(n) |
| MEMM | 相邻标签 | 否 | O(n) |
| **CRF** | 相邻标签 | 是 | O(n) |
| BiLSTM-CRF | 通过CRF层建模 | 是 | O(n) |

**CRF的优势**：
1. 全局归一化，避免MEMM的标签偏置问题
2. 可以学习任意特征
3. 解码算法高效（Viterbi）

## 参考

[^1]: Lafferty et al., "Conditional Random Fields: Probabilistic Models for Segmenting and Labeling Sequence Data", ICML 2001
[^2]: Sutton & McCallum, "An Introduction to Conditional Random Fields for Relational Learning"
[^3]: PyTorch-CRF Documentation. https://pytorch-crf.readthedocs.io/
[^4]: Lample et al., "Neural Architectures for Named Entity Recognition", NAACL 2016
