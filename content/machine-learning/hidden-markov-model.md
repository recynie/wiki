---
title: 隐马尔可夫模型
date: 2026-04-17
description: 隐马尔可夫模型（HMM）的三个基本问题、算法及Python实现
tags:
  - hmm
  - hidden-markov-model
  - sequence-modeling
  - machine-learning
draft: false
permalink:
---

## 概述

隐马尔可夫模型（Hidden Markov Model，HMM）是一种统计模型，用于描述一个含有隐含未知参数的马尔可夫过程。[^1]

HMM的核心思想是：系统状态不可直接观测，只能通过观测序列间接推断。

### 经典应用场景

| 领域 | 应用 |
|------|------|
| **语音识别** | 将声学信号转为文字 |
| **自然语言处理** | 词性标注、命名实体识别 |
| **生物信息学** | 基因序列分析 |
| **金融** | 市场状态预测（牛市/熊市） |
| **手势识别** | 从视频序列中识别动作 |

---

## HMM的基本结构

### 一个直观的例子

**问题**：根据多年树木年轮的尺寸（观察值），推断过去的气温状态（隐藏状态）。

```
隐藏状态（不可观测）:  炎热(H) → 炎热(H) → 寒冷(C) → 炎热(H)
                          ↓         ↓         ↓         ↓
观察值（可观测）:      小(S)     中(M)     小(S)     大(L)
```

### 五元组定义

| 符号 | 含义 | 说明 |
|------|------|------|
| $N$ | 状态数 | 隐藏状态的个数 |
| $M$ | 观测符号数 | 可能的观测值个数 |
| $A$ | 转移概率矩阵 | $a_{ij} = P(q_{t+1}=j \mid q_t=i)$ |
| $B$ | 观测概率矩阵 | $b_j(k) = P(o_t=k \mid q_t=j)$ |
| $\pi$ | 初始状态分布 | $\pi_i = P(q_1=i)$ |

HMM通常记作 $\lambda = (A, B, \pi)$。

---

## HMM的三个基本问题

### 问题一：评估问题（Evaluation）

**给定**：模型 $\lambda = (A, B, \pi)$ 和观测序列 $O = (o_1, o_2, \ldots, o_T)$

**求解**：计算 $P(O \mid \lambda)$，即观测序列出现的概率

### 问题二：解码问题（Decoding）

**给定**：模型 $\lambda = (A, B, \pi)$ 和观测序列 $O = (o_1, o_2, \ldots, o_T)$

**求解**：找到最可能的状态序列 $Q = (q_1, q_2, \ldots, q_T)$

### 问题三：学习问题（Learning）

**给定**：观测序列 $O = (o_1, o_2, \ldots, o_T)$

**求解**：调整参数 $\lambda = (A, B, \pi)$ 使得 $P(O \mid \lambda)$ 最大

---

## 问题一：Forward算法

### 朴素方法的困境

直接计算 $P(O \mid \lambda)$ 需要对所有可能的状态序列求和：

$$
P(O \mid \lambda) = \sum_{Q} P(O \mid Q, \lambda) P(Q \mid \lambda)
$$

对于长度为 $T$ 的序列，状态数为 $N$，则需要计算 $N^T$ 种可能，这在实际中是不可行的。

### α-pass递归

定义**前向变量**：

$$
\alpha_t(i) = P(o_1, o_2, \ldots, o_t, q_t = i \mid \lambda)
$$

**递归步骤**：

1. **初始化**（$t=1$）：
$$
\alpha_1(i) = \pi_i b_i(o_1), \quad i = 1, 2, \ldots, N
$$

2. **递归**（$t = 2, 3, \ldots, T$）：
$$
\alpha_t(i) = \left[\sum_{j=1}^{N} \alpha_{t-1}(j) a_{ji}\right] b_i(o_t)
$$

3. **终止**：
$$
P(O \mid \lambda) = \sum_{i=1}^{N} \alpha_T(i)
$$

### 时间复杂度

| 方法 | 时间复杂度 |
|------|-----------|
| 朴素方法 | $O(N^T)$ |
| Forward算法 | $O(N^2 T)$ |

---

## 问题二：Viterbi算法

### 动态规划思想

Viterbi算法使用动态规划找到最可能的状态序列。核心思想是：保留到达每个状态的最优路径。

### δ-ψ变量

定义：
- $\delta_t(i)$：时刻 $t$，状态为 $i$ 的所有路径中，最大概率值
- $\psi_t(i)$：时刻 $t$，状态为 $i$ 的最优路径的前驱状态

### 算法步骤

1. **初始化**：
$$
\delta_1(i) = \pi_i b_i(o_1), \quad \psi_1(i) = 0
$$

2. **递归**（$t = 2, 3, \ldots, T$）：
$$
\delta_t(i) = \max_{1 \leq j \leq N} [\delta_{t-1}(j) a_{ji}] b_i(o_t)
$$
$$
\psi_t(i) = \arg\max_{1 \leq j \leq N} [\delta_{t-1}(j) a_{ji}]
$$

3. **终止**：
$$
P^* = \max_{1 \leq i \leq N} \delta_T(i)
$$
$$
q_T^* = \arg\max_{1 \leq i \leq N} \delta_T(i)
$$

4. **回溯**：
$$
q_{t-1}^* = \psi_t(q_t^*), \quad t = T, T-1, \ldots, 2
$$

### 下溢问题

由于概率连乘会导致数值下溢，实际中通常使用**对数概率**：

$$
\widehat{\delta}_t(i) = \log \delta_t(i)
$$
$$
\widehat{\delta}_t(i) = \max_j [\widehat{\delta}_{t-1}(j) + \log a_{ji} + \log b_i(o_t)]
$$

---

## 问题三：Baum-Welch算法（EM算法）

### 后向变量

定义**后向变量**：
$$
\beta_t(i) = P(o_{t+1}, o_{t+2}, \ldots, o_T \mid q_t = i, \lambda)
$$

**递归步骤**：

1. **初始化**（$t = T$）：
$$
\beta_T(i) = 1
$$

2. **递归**（$t = T-1, T-2, \ldots, 1$）：
$$
\beta_t(i) = \sum_{j=1}^{N} a_{ij} b_j(o_{t+1}) \beta_{t+1}(j)
$$

### γ和ξ变量

**γ变量**（单状态概率）：
$$
\gamma_t(i) = P(q_t = i \mid O, \lambda) = \frac{\alpha_t(i) \beta_t(i)}{\sum_{j=1}^{N} \alpha_t(j) \beta_t(j)}
$$

**ξ变量**（双状态概率）：
$$
\xi_t(i, j) = P(q_t = i, q_{t+1} = j \mid O, \lambda) = \frac{\alpha_t(i) a_{ij} b_j(o_{t+1}) \beta_{t+1}(j)}{\sum_{i=1}^{N}\sum_{j=1}^{N} \alpha_t(i) a_{ij} b_j(o_{t+1}) \beta_{t+1}(j)}
$$

### 参数重估计

1. **更新初始分布**：
$$
\pi_i = \gamma_1(i)
$$

2. **更新转移概率**：
$$
\widehat{a}_{ij} = \frac{\sum_{t=1}^{T-1} \xi_t(i, j)}{\sum_{t=1}^{T-1} \gamma_t(i)}
$$

3. **更新观测概率**：
$$
\widehat{b}_j(k) = \frac{\sum_{t=1}^{T} \mathbf{1}(o_t = v_k) \gamma_t(j)}{\sum_{t=1}^{T} \gamma_t(j)}
$$

### 算法流程

```
输入: 观测序列 O, 初始参数估计
输出: 最优参数 λ*

重复:
    1. E步: 计算 γ_t(i) 和 ξ_t(i,j)
    2. M步: 用公式更新 A, B, π
    3. 计算 log P(O|λ)
直到 log P(O|λ) 收敛
```

---

## Python实现

### Forward算法实现

```python
import numpy as np

def forward(obs, A, B, pi):
    """
    Forward算法计算观测概率
    
    参数:
        obs: 观测序列 (T,)
        A: 转移概率矩阵 (N, N)
        B: 观测概率矩阵 (N, M)
        pi: 初始分布 (N,)
    
    返回:
        prob: 观测序列的概率
        alpha: 前向变量 (T, N)
    """
    T = len(obs)
    N = len(pi)
    
    # 缩放因子，防止下溢
    scale = np.zeros(T)
    
    # 初始化
    alpha = np.zeros((T, N))
    alpha[0] = pi * B[:, obs[0]]
    scale[0] = np.sum(alpha[0])
    alpha[0] /= scale[0]
    
    # 递归
    for t in range(1, T):
        for j in range(N):
            alpha[t, j] = np.dot(alpha[t-1], A[:, j]) * B[j, obs[t]]
        scale[t] = np.sum(alpha[t])
        alpha[t] /= scale[t]
    
    # 计算对数概率
    prob = -np.sum(np.log(scale))
    
    return np.exp(prob), alpha

# 示例：天气模型
# 状态：0=晴, 1=雨
# 观测：0=散步, 1=购物, 2=打扫

A = np.array([[0.7, 0.3],
              [0.4, 0.6]])  # 转移概率

B = np.array([[0.5, 0.2, 0.3],
              [0.1, 0.6, 0.3]])  # 观测概率

pi = np.array([0.6, 0.4])  # 初始分布

obs = np.array([0, 2, 1])  # 观测序列：散步→打扫→购物
prob, alpha = forward(obs, A, B, pi)
print(f"观测概率: {prob:.6f}")
```

### Viterbi算法实现

```python
def viterbi(obs, A, B, pi):
    """
    Viterbi算法求解最优状态序列
    
    返回:
        path: 最优状态序列
        prob: 最优路径的概率
    """
    T = len(obs)
    N = len(pi)
    
    # 初始化
    delta = np.zeros((T, N))
    psi = np.zeros((T, N), dtype=int)
    
    delta[0] = np.log(pi) + np.log(B[:, obs[0]])
    psi[0] = 0
    
    # 递归
    for t in range(1, T):
        for j in range(N):
            prob = delta[t-1] + np.log(A[:, j])
            psi[t, j] = np.argmax(prob)
            delta[t, j] = np.max(prob) + np.log(B[j, obs[t]])
    
    # 回溯
    path = np.zeros(T, dtype=int)
    path[T-1] = np.argmax(delta[T-1])
    for t in range(T-2, -1, -1):
        path[t] = psi[t+1, path[t+1]]
    
    return path, np.exp(np.max(delta[T-1]))

# 测试
path, prob = viterbi(obs, A, B, pi)
state_names = ['晴', '雨']
print(f"最优状态序列: {[state_names[i] for i in path]}")
print(f"概率: {prob:.6f}")
```

---

## 实际应用：词性标注

### 问题背景

给定一个句子，为每个词标注词性（如名词、动词、形容词等）。

### HMM建模

- **隐藏状态**：词性标签（NN, VB, DT等）
- **观测序列**：句子中的词
- **转移概率**：$P(\text{词性}_t \mid \text{词性}_{t-1})$
- **发射概率**：$P(\text{词}_t \mid \text{词性}_t)$

### 示例

```
句子: "The cat sits on the mat"
观测:  The  cat  sits  on   the  mat
标签:  DT   NN   VBZ   IN   DT   NN
```

使用Viterbi算法找到最可能的标签序列。

---

## HMM的局限性与扩展

### HMM的局限

1. **观测独立性假设**：假设观测只与当前状态有关
2. **一阶马尔可夫假设**：假设状态只与前一状态有关
3. **表达能力有限**：难以建模复杂的序列依赖

### 扩展方向

| 模型 | 改进 |
|------|------|
| **高阶HMM** | 状态依赖前多个状态 |
| **条件随机场(CRF)** | 放松观测独立性假设 |
| **双向LSTM** | 建模长距离依赖 |

详见 [[./transformer-and-attention|Transformer与注意力机制]]。

---

## 参考

[^1]: L. R. Rabiner, "A Tutorial on Hidden Markov Models and Selected Applications in Speech Recognition", Proceedings of the IEEE, 1989

[^2]: Mark Stamp, "A Revealing Introduction to Hidden Markov Models", San Jose State University
