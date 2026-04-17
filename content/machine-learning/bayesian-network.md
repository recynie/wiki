---
title: 贝叶斯网络
date: 2026-04-17
description: 概率图模型：贝叶斯网络的原理与实现
tags:
  - machine-learning
  - bayesian-network
  - probabilistic-graphical-model
draft: false
permalink:
---

# 贝叶斯网络

贝叶斯网络（Bayesian Network），又称信念网络（Belief Network）或有向无环图模型（Directed Acyclic Graph, DAG），是一种用图形化方式表示和推理随机变量之间条件依赖关系的概率图模型。

## 基本概念

### 什么是贝叶斯网络

贝叶斯网络由两部分组成：
1. **有向无环图（DAG）**：节点表示随机变量，有向边表示条件依赖
2. **条件概率表（CPT）**：每个节点的概率分布，以其父节点为条件

### 示例：学生成绩模型

```
        [智力] ──────→ [成绩]
           │              ▲
           │              │
        [难度] ───→ [推荐]─┘
           │
           ▼
        [SAT]
```

这个网络表示：
- 智力（Intelligence）影响成绩和SAT分数
- 难度（Difficulty）影响成绩
- 成绩（Grade）影响推荐信（Letter）

## 联合概率分解

### 马尔可夫条件

贝叶斯网络利用**条件独立性**：给定父节点，节点条件独立于其非后代节点。

这使得联合概率可以分解为：

$$
P(X_1, X_2, ..., X_n) = \prod_{i=1}^{n} P(X_i | Pa(X_i))
$$

其中 $Pa(X_i)$ 是 $X_i$ 的父节点集合。

### 示例分解

对于学生成绩网络：

$$
P(I, D, G, S, L) = P(I) \cdot P(D) \cdot P(G|I,D) \cdot P(S|I) \cdot P(L|G)
$$

这比直接存储联合分布（需要 $2^5 = 32$ 个参数）节省大量空间。

## 条件独立性检验

### D-分离（D-Separation）

在贝叶斯网络中，判断两个节点集合是否条件独立：

1. **链式结构**（Chain）：$X \rightarrow Z \rightarrow Y$
   - $X \perp Y | Z$ ✓（给定Z，X和Y独立）

2. **分叉结构**（Fork）：$X \leftarrow Z \rightarrow Y$
   - $X \perp Y | Z$ ✓

3. **碰撞结构**（Collider）：$X \rightarrow Z \leftarrow Y$
   - $X \perp Y$（在Z未观测时）
   - $X \not\perp Y | Z$（在Z被观测时）

### 示例

```
I → G ← D
```

- 给定G之前：I和D独立
- 给定G之后：I和D不独立（"解释效应"）

## 推理方法

### 精确推理

#### 变量消除算法

核心思想：按顺序消去变量，每次只处理一个因子的乘积。

```python
import numpy as np

class VariableElimination:
    def __init__(self, network):
        self.network = network
    
    def eliminate(self, factors, variable, evidence):
        """变量消除算法"""
        # 1. 收集所有包含该变量的因子
        relevant_factors = [f for f in factors if variable in f.vars]
        
        # 2. 乘积
        product = relevant_factors[0]
        for f in relevant_factors[1:]:
            product = product * f
        
        # 3. 边缘化
        result = product.sum_over(variable)
        
        return result
    
    def query(self, query_var, evidence):
        """
        计算后验概率 P(query_var | evidence)
        """
        factors = list(self.network.factors.values())
        
        # 加入证据因子
        for var, val in evidence.items():
            factors.append(self.network.get_evidence_factor(var, val))
        
        # 消除所有非查询变量
        elimination_order = [v for v in self.network.variables 
                           if v != query_var and v not in evidence]
        
        for var in elimination_order:
            factors = [self.eliminate(factors, var, evidence)]
        
        # 归一化
        result = factors[0]
        result.values /= result.values.sum()
        
        return result
```

#### 置信传播（Belief Propagation）

对于树结构的贝叶斯网络，使用消息传递算法：

```python
class BeliefPropagation:
    def __init__(self, network):
        self.network = network
        self.messages = {}  # (from, to) -> message
    
    def message(self, node, parent):
        """计算从node到parent的消息"""
        if (node, parent) in self.messages:
            return self.messages[(node, parent)]
        
        # 收集来自其他子节点的消息
        message = self.network.get_cpt(node)
        for child in node.children:
            if child != parent:
                child_msg = self.message(child, node)
                message = message * child_msg
        
        # 边缘化
        message = message.sum_over(node.name)
        
        self.messages[(node, parent)] = message
        return message
    
    def belief(self, node):
        """计算节点的边缘分布"""
        # 收集来自所有父节点的消息
        prior = self.network.get_prior(node)
        
        for parent in node.parents:
            parent_msg = self.message(parent, node)
            prior = prior * parent_msg
        
        # 归一化
        prior.values /= prior.values.sum()
        
        return prior
```

### 近似推理

#### 吉布斯采样

```python
class GibbsSampling:
    def __init__(self, network, num_samples=10000):
        self.network = network
        self.num_samples = num_samples
    
    def sample(self, query_vars, evidence):
        """吉布斯采样"""
        # 初始化所有变量
        state = {}
        for var in self.network.variables:
            if var in evidence:
                state[var] = evidence[var]
            else:
                state[var] = np.random.choice(self.network.get_states(var))
        
        # 采样计数
        counts = {var: np.zeros(self.network.get_states(var)) 
                 for var in query_vars}
        
        # 吉布斯采样循环
        for _ in range(self.num_samples):
            for var in self.network.variables:
                if var in evidence:
                    continue
                
                # 计算条件分布 P(var | 其他变量)
                probs = self.network.get_conditional(var, state)
                
                # 采样
                state[var] = np.random.choice(len(probs), p=probs)
            
            # 更新计数
            for var in query_vars:
                counts[var][state[var]] += 1
        
        # 归一化
        beliefs = {var: counts[var] / counts[var].sum() for var in query_vars}
        
        return beliefs
```

## 参数学习

### 极大似然估计（MLE）

给定数据 $D = \{x^{(1)}, ..., x^{(m)}\}$，学习参数 $\theta$：

$$
\hat{\theta}_{MLE} = \arg\max_\theta P(D | \theta)
$$

对于贝叶斯网络：

$$
\hat{\theta}_{ijk} = \frac{N_{ijk}}{\sum_k N_{ijk}}
$$

其中 $N_{ijk}$ 是在数据中父节点取值为 $i$，节点取值为 $j$ 的次数。

### 贝叶斯估计

引入先验分布 $P(\theta)$：

$$
P(\theta | D) = \frac{P(D | \theta) P(\theta)}{P(D)}
$$

共轭先验使得后验分布与先验有相同形式。

```python
class BayesianNetworkLearner:
    def __init__(self, network, prior_alpha=1.0):
        self.network = network
        self.prior_alpha = prior_alpha  # Dirichlet先验参数
    
    def fit(self, data):
        """
        data: DataFrame, 每列是一个变量
        """
        for node in self.network.nodes:
            parents = node.parents
            
            # 计数
            if parents:
                counts = data.groupby([node.name] + [p.name for p in parents]).size()
            else:
                counts = data[node.name].value_counts()
            
            # 计算CPT参数
            self.network.set_cpt(node, self._compute_cpt(counts))
    
    def _compute_cpt(self, counts):
        """计算条件概率表（使用贝叶斯估计）"""
        # 添加伪计数（先验）
        # ...
        pass
```

### EM算法（处理隐变量）

```python
class EMAlgorithm:
    def __init__(self, model, max_iter=100, tol=1e-6):
        self.model = model
        self.max_iter = max_iter
        self.tol = tol
    
    def fit(self, data, hidden_vars):
        """
        data: 观测数据
        hidden_vars: 隐变量列表
        """
        # E步：给定当前参数，计算隐变量的后验
        def e_step(params):
            # 使用置信传播计算 P(hidden | observed, params)
            pass
        
        # M步：最大化期望似然
        def m_step(expected_counts):
            # 重新估计参数
            pass
        
        # 迭代
        for iteration in range(self.max_iter):
            # E步
            expected_hidden = e_step(self.params)
            
            # M步
            new_params = m_step(expected_hidden)
            
            # 检查收敛
            if self._converged(self.params, new_params, self.tol):
                break
            
            self.params = new_params
        
        return self.params
```

## 使用pgmpy库

### 安装与基础使用

```bash
pip install pgmpy
```

```python
from pgmpy.models import BayesianNetwork
from pgmpy.estimators import MaximumLikelihoodEstimator, BayesianEstimator
from pgmpy.inference import VariableElimination
import pandas as pd

# 定义网络结构
model = BayesianNetwork([
    ('Intelligence', 'Grade'),
    ('Difficulty', 'Grade'),
    ('Intelligence', 'SAT'),
    ('Grade', 'Letter')
])

# 定义CPT（简化）
cpt_intelligence = pd.DataFrame({
    'Intelligence': ['High', 'Low'],
    'p': [0.3, 0.7]
})

cpt_difficulty = pd.DataFrame({
    'Difficulty': ['Easy', 'Hard'],
    'p': [0.6, 0.4]
})

# 设置CPT
model.fit()
```

### 推理示例

```python
from pgmpy.inference import VariableElimination

# 创建推理器
infer = VariableElimination(model)

# 查询：给定某学生SAT高分，求其智力高的概率
result = infer.query(['Intelligence'], evidence={'SAT': 'High'})
print(result)

# 查询：给定某学生成绩好且难度高，求其智力高的概率
result = infer.query(['Intelligence'], 
                     evidence={'Grade': 'Good', 'Difficulty': 'Hard'})
print(result)
```

## 应用场景

### 疾病诊断

```
症状 → 疾病
```

根据症状推断疾病概率，是医疗诊断系统的基础。

### 故障检测

```
设备状态 → 传感器读数
```

根据传感器异常推断设备故障。

### 文本分类

```python
class NaiveBayesTextClassifier:
    """朴素贝叶斯文本分类器（贝叶斯网络特例）"""
    
    def __init__(self, alpha=1.0):
        self.alpha = alpha  # 平滑参数
        self.class_probs = {}
        self.word_probs = {}
    
    def fit(self, X, y):
        # P(class)
        self.classes, counts = np.unique(y, return_counts=True)
        self.class_probs = dict(zip(self.classes, counts / len(y)))
        
        # P(word | class)
        vocab = set()
        for doc in X:
            vocab.update(doc)
        
        self.vocab_size = len(vocab)
        
        for c in self.classes:
            docs_c = [X[i] for i in range(len(y)) if y[i] == c]
            word_counts = {}
            total_words = 0
            
            for doc in docs_c:
                for word in doc:
                    word_counts[word] = word_counts.get(word, 0) + 1
                    total_words += 1
            
            self.word_probs[c] = {
                word: (count + self.alpha) / (total_words + self.alpha * self.vocab_size)
                for word, count in word_counts.items()
            }
    
    def predict(self, X):
        predictions = []
        for doc in X:
            probs = {}
            for c in self.classes:
                log_prob = np.log(self.class_probs[c])
                for word in doc:
                    if word in self.word_probs[c]:
                        log_prob += np.log(self.word_probs[c][word])
                probs[c] = log_prob
            
            predictions.append(max(probs, key=probs.get))
        
        return np.array(predictions)
```

## 与其他模型的关系

| 模型 | 类型 | 特点 |
|------|------|------|
| 朴素贝叶斯 | 生成模型 | 条件独立假设（无向边） |
| HMM | 序列模型 | 时间序列上的贝叶斯网络 |
| CRF | 判别模型 | 无向图，直接建模条件概率 |
| 贝叶斯网络 | 生成模型 | 有向图，建模联合分布 |

## 参考

[^1]: Koller & Friedman, "Probabilistic Graphical Models: Principles and Techniques"
[^2]: Murphy, K., "Machine Learning: A Probabilistic Perspective"
[^3]: Pearl, J., "Probabilistic Reasoning in Intelligent Systems"
[^4]: pgmpy Documentation. https://pgmpy.org/
