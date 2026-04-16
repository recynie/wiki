---
title: LLM 理论与机制
date: 2026-04-14
description: 大语言模型的核心理论与工作机制
tags:
  - llm
  - theory
  - attention
  - transformer
draft: false
permalink:
---

## 概述

大语言模型（Large Language Model, LLM）是一类基于[[../deep-learning/transformer|Transformer]]架构的深度神经网络模型，通过在大规模文本语料上进行无监督预训练，学习语言的统计规律和语义表示。LLM 的核心目标是**建模语言的可能性分布**，即给定一个文本序列，预测下一个最可能的 token。

LLM 的关键特征包括：

- **大规模参数**：通常具有数十亿到数千亿个参数
- **大规模预训练**：在互联网级别的文本语料上训练
- **通用能力**：通过prompt即可完成多种下游任务
- **涌现现象**：随着模型规模增大，出现超出预期的能力

## Next-Token Prediction

### 自回归语言模型的本质

自回归语言模型的核心任务是：**给定前 $t$ 个 token，预测第 $t+1$ 个 token**。形式化地，给定文本序列 $x_1, x_2, \ldots, x_T$，模型学习条件概率分布：

$$
P(x_{t+1} | x_1, x_2, \ldots, x_t; \theta)
$$

其中 $\theta$ 是模型参数。整个序列的联合概率可以分解为：

$$
P(x_1, x_2, \ldots, x_T) = \prod_{t=1}^{T} P(x_t | x_1, \ldots, x_{t-1}; \theta)
$$

### 概率分布与 softmax

模型的输出通常是一个向量 $h_t \in \mathbb{R}^V$，其中 $V$ 是词表大小。对应的 logit 通过 softmax 转化为概率分布：

$$
P_{\theta}(x_{t+1} = v | x_{<t}) = \frac{\exp(h_t^\top w_v)}{\sum_{v' \in V} \exp(h_t^\top w_{v'})}
$$

其中 $w_v$ 是词表中第 $v$ 个 token 对应的词嵌入向量。

```cpp
#include <bits/stdc++.h>
using namespace std;

// 简化的 softmax 实现
vector<double> softmax(const vector<double>& logits) {
    vector<double> probs(logits.size());
    double max_logit = *max_element(logits.begin(), logits.end());
    double sum = 0.0;
    for (size_t i = 0; i < logits.size(); i++) {
        probs[i] = exp(logits[i] - max_logit);
        sum += probs[i];
    }
    for (size_t i = 0; i < probs.size(); i++) {
        probs[i] /= sum;
    }
    return probs;
}
```

### 温度 (Temperature) 与 top-p 采样

在生成时，通过调整采样策略控制输出的多样性和质量：

- **Temperature $T$**：在 softmax 前对 logit 进行缩放
  $$
  P_T(v) = \frac{\exp(\text{logit}_v / T)}{\sum_{v'} \exp(\text{logit}_{v'} / T)}
  $$
  - $T \to 0$：近似贪婪采样，输出确定性强
  - $T \to 1$：保持原始分布
  - $T > 1$：增加随机性，输出更多样

- **Top-p (Nucleus) 采样**：动态选择累积概率达到 $p$ 的最小 token 集合进行采样，比固定 top-k 更灵活

```python
import numpy as np

def sample_with_temperature(logits, temperature=1.0):
    """带温度的采样"""
    logits = np.array(logits, dtype=np.float64)
    logits /= temperature
    max_logit = np.max(logits)
    logits = logits - max_logit  # 数值稳定
    probs = np.exp(logits)
    probs /= np.sum(probs)
    return np.random.choice(len(probs), p=probs)

def top_p_sample(logits, p=0.9):
    """Top-p (Nucleus) 采样"""
    sorted_indices = np.argsort(logits)[::-1]
    sorted_logits = logits[sorted_indices]
    cumsum = np.cumsum(np.exp(sorted_logits - np.max(sorted_logits)))
    cumsum /= cumsum[-1]  # 归一化
    
    # 找到累积概率超过 p 的最小集合
    cutoff_idx = np.searchsorted(cumsum, p) + 1
    top_indices = sorted_indices[:cutoff_idx]
    top_logits = logits[top_indices]
    
    # 归一化并采样
    top_probs = np.exp(top_logits - np.max(top_logits))
    top_probs /= np.sum(top_probs)
    return top_indices[np.random.choice(len(top_probs), p=top_probs)]
```

## Transformer 架构基础

### Tokenization: Subword (BPE, WordPiece)

Tokenization 是将原始文本转换为模型可处理的 token 序列的过程。现代 LLM 主要使用**子词（subword）**分词方法：

- **BPE (Byte Pair Encoding)**[^1]：通过合并最高频的字节对，逐步构建词表。GPT 系列使用。
- **WordPiece**：基于语言学动机，优先保留完整词。BERT 使用。
- **SentencePiece**：在训练时直接从原始文本学习，脱离语言假设。

BPE 的核心算法：

```python
from collections import Counter, defaultdict

def learn_bpe(vocab, num_merges):
    """学习 BPE 词表"""
    # vocab: {(token_ids): frequency}
    vocab = {tuple(ids): freq for ids, freq in vocab.items()}
    
    for _ in range(num_merges):
        # 统计所有相邻字节对频率
        pairs = Counter()
        for token_ids, freq in vocab.items():
            for i in range(len(token_ids) - 1):
                pairs[(token_ids[i], token_ids[i+1])] += freq
        
        if not pairs:
            break
        # 找到最频繁的字节对
        best_pair = max(pairs, key=pairs.get)
        
        # 合并所有出现的 best_pair
        new_token_id = max(max(ids) for ids in vocab) + 1
        new_vocab = {}
        for token_ids, freq in vocab.items():
            new_ids = []
            i = 0
            while i < len(token_ids):
                if i < len(token_ids) - 1 and (token_ids[i], token_ids[i+1]) == best_pair:
                    new_ids.append(new_token_id)
                    i += 2
                else:
                    new_ids.append(token_ids[i])
                    i += 1
            new_vocab[tuple(new_ids)] = freq
        vocab = new_vocab
    
    return vocab
```

### Embedding + Positional Encoding

Transformer 的输入由两部分组成：

- **Token Embedding**：将 token ID 映射为 $d_{\text{model}}$ 维向量
- **Positional Encoding (PE)**：为序列中的每个位置添加位置信息

标准正弦/余弦位置编码[^2]：

$$
\text{PE}(pos, 2i) = \sin\left(\frac{pos}{10000^{2i/d_{\text{model}}}}\right), \quad \text{PE}(pos, 2i+1) = \cos\left(\frac{pos}{10000^{2i/d_{\text{model}}}}\right)
$$

这种编码方式允许模型学习相对位置关系，因为 $\text{PE}(pos+k)$ 可以表示为 $\text{PE}(pos)$ 的线性函数。

```python
import numpy as np

def positional_encoding(max_seq_len, d_model):
    """生成正弦/余弦位置编码"""
    pe = np.zeros((max_seq_len, d_model))
    position = np.arange(max_seq_len)[:, np.newaxis]
    div_term = np.exp(np.arange(0, d_model, 2) * -(np.log(10000.0) / d_model))
    
    pe[:, 0::2] = np.sin(position * div_term)
    pe[:, 1::2] = np.cos(position * div_term)
    return pe
```

> **注意**：现代 LLM（如 [[llm-gpt|GPT]]、Llama）常用**旋转位置编码 (RoPE)**[^3] 替代绝对位置编码，在计算 attention score 时注入相对位置信息。

### Self-Attention 计算: Q, K, V 矩阵

Self-Attention 是 Transformer 的核心，其计算过程为：

1. **线性投影**：将输入 $X \in \mathbb{R}^{n \times d_{\text{model}}}$ 投影为 Query、Key、Value：
   $$
   Q = X W_Q,\quad K = X W_K,\quad V = X W_V
   $$
   其中 $W_Q, W_K, W_V \in \mathbb{R}^{d_{\text{model}} \times d_k}$

2. **Attention Score**：计算 Query 和 Key 的相似度
   $$
   \text{Attention}(Q, K, V) = \text{softmax}\left(\frac{Q K^\top}{\sqrt{d_k}}\right) V
   $$

3. **缩放因子 $\sqrt{d_k}$**：防止点积过大导致 softmax 梯度消失

```cpp
#include <bits/stdc++.h>
using namespace std;

vector<vector<double>> self_attention(
    const vector<vector<double>>& Q,
    const vector<vector<double>>& K,
    const vector<vector<double>>& V,
    double scale = 1.0) {
    
    int n = Q.size();      // 序列长度
    int d_k = Q[0].size(); // 维度
    
    // 计算 QK^T
    vector<vector<double>> scores(n, vector<double>(n, 0));
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            for (int k = 0; k < d_k; k++) {
                scores[i][j] += Q[i][k] * K[j][k];
            }
            scores[i][j] /= scale;
        }
    }
    
    // Softmax
    vector<vector<double>> attn(n, vector<double>(n, 0));
    for (int i = 0; i < n; i++) {
        double max_score = *max_element(scores[i].begin(), scores[i].end());
        double sum = 0;
        for (int j = 0; j < n; j++) {
            scores[i][j] = exp(scores[i][j] - max_score);
            sum += scores[i][j];
        }
        for (int j = 0; j < n; j++) {
            attn[i][j] = scores[i][j] / sum;
        }
    }
    
    // 乘以 V
    vector<vector<double>> output(n, vector<double>(d_k, 0));
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            for (int k = 0; k < d_k; k++) {
                output[i][k] += attn[i][j] * V[j][k];
            }
        }
    }
    return output;
}
```

### Multi-Head Attention 的作用

Multi-Head Attention (MHA)[^2] 在多个"头"上并行计算 attention，每个头学习不同的注意力模式：

$$
\text{MultiHead}(Q, K, V) = \text{Concat}(\text{head}_1, \ldots, \text{head}_h) W_O
$$

其中 $\text{head}_i = \text{Attention}(Q W_Q^i, K W_K^i, V W_V^i)$。

**为什么使用多头的几个解释**：

1. **注意力模式多样化**：不同头可能关注不同的语义关系（语法、语义、指代等）
2. **集成学习**：每个头可视为一个弱分类器，concat 后通过 $W_O$ 整合
3. **表示分裂**：将 $d_{\text{model}}$ 维空间分解为 $h$ 个子空间，增强表达能力

```python
import numpy as np

class MultiHeadAttention:
    def __init__(self, d_model, num_heads):
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        # 初始化投影矩阵
        self.W_Q = np.random.randn(d_model, d_model) * 0.02
        self.W_K = np.random.randn(d_model, d_model) * 0.02
        self.W_V = np.random.randn(d_model, d_model) * 0.02
        self.W_O = np.random.randn(d_model, d_model) * 0.02
    
    def split_heads(self, X):
        """将 d_model 分割为 num_heads 个头"""
        batch_size, seq_len, _ = X.shape
        X = X.reshape(batch_size, seq_len, self.num_heads, self.d_k)
        return X.transpose(0, 2, 1, 3)  # (batch, heads, seq_len, d_k)
    
    def forward(self, Q, K, V):
        batch_size = Q.shape[0]
        
        # 线性投影
        Q = Q @ self.W_Q
        K = K @ self.W_K
        V = V @ self.W_V
        
        # 分裂为多头
        Q = self.split_heads(Q)
        K = self.split_heads(K)
        V = self.split_heads(V)
        
        # Scaled dot-product attention
        scale = np.sqrt(self.d_k)
        scores = Q @ K.transpose(0, 1, 3, 2) / scale
        attn_weights = softmax(scores, axis=-1)
        
        # 合并多头
        attn_output = attn_weights @ V
        attn_output = attn_output.transpose(0, 2, 1, 3).reshape(batch_size, -1, self.d_model)
        
        return attn_output @ self.W_O
```

### FFN (Feed-Forward Network) 作为 Key-Value Memory

FFN 通常采用两层全连接结构：

$$
\text{FFN}(x) = \sigma(x W_1 + b_1) W_2 + b_2
$$

其中 $\sigma$ 通常为 ReLU 或 GELU。

**FFN 的隐式存储功能**[^4]：研究表明 FFN 层可以视为 key-value memory：
- 第一层 $W_1$ 的行向量可视为"键"（patterns）
- 第二层 $W_2$ 的列向量可视为"值"（facts）

这解释了为什么 FFN 占据了 Transformer 约 2/3 的参数。

## 预训练与微调

### Pre-training: 无监督学习大规模文本

预训练阶段采用**自监督学习**，主要有两种范式：

1. **仅解码器 (Decoder-only)**：如 GPT 系列，使用 Next-Token Prediction
2. **编码器-解码器 (Encoder-Decoder)**：如 T5，使用 Span Corruption（随机遮盖一段文本，预测原内容）

预训练的核心是最大化似然：

$$
\mathcal{L}_{\text{pretrain}} = \sum_{t=1}^{T} -\log P_\theta(x_t | x_{<t})
$$

### Supervised Fine-Tuning (SFT)

SFT 是指在特定任务的标注数据上微调预训练模型：

$$
\mathcal{L}_{\text{SFT}} = \sum_{(x, y)} -\log P_\theta(y | x)
$$

典型应用包括：
- 对话系统微调
- 特定领域知识注入
- 输出格式控制

### RLHF (Reinforcement Learning from Human Feedback)

RLHF[^5] 通过人类反馈信号优化模型行为，分为三步：

1. **收集人类偏好数据**：对模型输出进行排序，训练 Reward Model $r_\phi(x, y)$
2. **训练 Reward Model**：最大化偏好差异
   $$
   \mathcal{L}_r = -\mathbb{E}_{(x, y^+, y^-)} \log \sigma(r_\phi(x, y^+) - r_\phi(x, y^-))
   $$
3. **强化学习优化**：使用 PPO 算法最大化 reward
   $$
   \mathcal{L}_{\text{PPO}} = \mathbb{E}_x \mathbb{E}_{y \sim \pi_\theta} \left[ r_\phi(x, y) - \beta \cdot \text{KL}(\pi_\theta || \pi_{\text{ref}}) \right]
   $$

### DPO (Direct Preference Optimization)

DPO[^6] 绕过强化学习，直接在偏好数据上优化：

$$
\mathcal{L}_{\text{DPO}} = -\mathbb{E}_{(x, y^+, y^-)} \log \sigma \left( \beta \log \frac{P_\theta(y^+|x)}{P_{\text{ref}}(y^+|x)} - \beta \log \frac{P_\theta(y^-|x)}{P_{\text{ref}}(y^-|x)} \right)
$$

DPO 避免了 RLHF 中复杂的强化学习训练过程，同时保持类似的效果。

## 涌现现象 (Emergent Abilities)

### 定义与例子

**涌现能力 (Emergent Ability)**[^7]是指模型在规模较小时不存在，但随着规模增大突然出现的能力。例如：

- **多步推理**：Chain-of-Thought
- **数学运算**：多位数加法、乘法
- **代码生成**：复杂编程任务
- **知识推理**：基于逻辑的问答

### 缩放定律 (Scaling Laws): Kaplan et al. 2020

Kaplan et al. (2020)[^8] 发现模型的性能（困惑度）与模型参数量 $N$、数据集大小 $D$、计算量 $C$ 存在幂律关系：

$$
L(N) \approx \left(\frac{N_0}{N}\right)^{\alpha_N}, \quad L(D) \approx \left(\frac{D_0}{D}\right)^{\alpha_D}, \quad L(C) \approx \left(\frac{C_0}{C}\right)^{\alpha_C}
$$

**Chinchilla 缩放定律**[^9]指出：对于给定的计算预算，最优的模型参数量和训练 token 数量应该线性缩放，即：

$$
N_{\text{opt}} \approx G^{1/3} C^{1/3}, \quad D_{\text{opt}} \approx G^{-1/3} C^{1/3}
$$

其中 $G$ 是每个 token 的计算量常数。

### 通过缩放涌现的能力

| 能力 | 涌现规模（估计） |
|------|-----------------|
| 3位数加法 | ~10B 参数 |
| 单词释义 | ~100M 参数 |
| Chain-of-Thought | ~10B 参数 |
| 复杂代码生成 | ~100B 参数 |

涌现现象与评估指标的选择也有关[^7]——使用离散的"任务完成率"比连续的困惑度更容易观察到涌现。

## In-Context Learning (ICL)

### 什么是 ICL

In-Context Learning (ICL)[^10] 是指 LLM 在不更新参数的情况下，仅通过给定 prompt 中的示例即可学习新任务的能力。例如：

```
输入: "狗 -> 动物
     猫 -> 动物
     苹果 -> ?"
输出: "水果"
```

ICL 的形式化定义：给定一个测试输入 $x_{\text{test}}$ 和 $k$ 个示例 $\{(x_i, y_i)\}_{i=1}^k$，LLM 预测：

$$
P(y_{\text{test}} | x_{\text{test}}, \{(x_i, y_i)\}_{i=1}^k)
$$

### 梯度 vs 非梯度更新

ICL 的核心特点是**无梯度更新**（gradient-free），与传统的微调形成对比：

| 特性 | ICL | Fine-tuning |
|------|-----|-------------|
| 参数更新 | ❌ | ✅ |
| 推理时学习 | ✅ | ❌ |
| 适配新任务 | 只需改 prompt | 需要重新训练 |
| 计算成本 | 低 | 高 |

### Demonstration 的作用

ICL 中示例的选择对性能影响显著。研究表明：

1. **示例数量**：通常 $k \in [4, 32]$ 效果较好，过多可能引入噪声
2. **示例顺序**：格式一致性比内容一致性更重要
3. **标注质量**：错误的示例会导致性能下降

ICL 的机制尚在研究中，主要假说包括：

- **贝叶斯推断**：LLM 隐式进行贝叶斯推理，根据示例推断任务分布
- **隐式梯度下降**[^11]：attention 机制模拟了梯度下降的优化过程

## Chain-of-Thought Reasoning

### 思维链提示 (Chain-of-Thought Prompting)

Chain-of-Thought (CoT) Prompting[^12] 通过让模型输出中间推理步骤来提升复杂推理能力：

**标准 Prompt：**
```
问题：小明有5个苹果，小红给了他3个，小明吃掉了2个。小明现在有多少苹果？
答案：6
```

**CoT Prompt：**
```
问题：小明有5个苹果，小红给了他3个，小明吃掉了2个。小明现在有多少苹果？
思考：先算小明最后收到的苹果：5 + 3 = 8。
      然后减去吃掉的：8 - 2 = 6。
答案：6
```

CoT 在数学题、逻辑推理、代码生成等任务上效果显著。

### Self-Consistency

Self-Consistency[^13] 是 CoT 的改进，通过采样多个推理路径并投票选择最一致的答案：

1. 对同一问题进行 $k$ 次采样（temperature > 0），得到 $k$ 个推理路径
2. 统计最终答案的频率
3. 选择出现频率最高的答案作为最终输出

### 内在的推理机制

关于 CoT 为什么有效，存在多种解释：

1. **计算扩展**：思维链提供了额外的"计算预算"，允许模型进行更多参数更新
2. **符号化推理**：将高层推理分解为可验证的步骤
3. **对齐效应**：强制模型"思考"而非直接"猜测"

## 上下文窗口与扩展

### 上下文窗口的重要性

上下文窗口定义了 LLM 单次输入可以处理的最大 token 数量。扩展上下文窗口的意义：

- **长文档理解**：书籍、论文、代码仓库
- **多轮对话**：保持长程记忆
- **复杂推理**：多步骤任务的中间状态存储

### 位置插值 (Position Interpolation)

标准位置编码在扩展窗口时面临**外推问题**（extrapolation）：训练时的位置范围 $[0, L_{\text{train}}]$ 与推理时的 $[0, L_{\text{test}}]$ 不一致。

**位置插值 (PI)**[^14] 的核心思想是将超出训练范围的位置索引压缩到训练范围内：

$$
\text{PI}(\text{pos}) = \text{pos} \cdot \frac{L_{\text{train}}}{L_{\text{test}}}
$$

例如，将 4096 位置插值到 2048 训练范围，每个位置除以 2。

**RoPE 的扩展**：对于 RoPE[^3]，通过缩放旋转角度实现位置插值。

### 稀疏注意力与线性注意力

标准 Self-Attention 的计算复杂度为 $O(n^2)$，对于长序列成为瓶颈。

**稀疏注意力**：只计算部分位置对的 attention，如：
- **滑窗注意力**：只关注局部上下文
- **膨胀注意力**：像膨胀卷积一样跳跃采样
- **全局注意力**：特定 token（如 [CLS]）与所有位置交互

**线性注意力**[^15]：将 $O(n^2)$ 的 softmax 操作近似为线性复杂度：

$$
\text{Attention}(Q, K, V) = \frac{\phi(Q) (\phi(K)^\top V)}{\phi(Q) \phi(K)^\top}
$$

其中 $\phi$ 是特征映射函数。核心是利用矩阵乘法的结合律：

$$
(\phi(Q) \phi(K)^\top) V = \phi(Q) (\phi(K)^\top V)
$$

从而将计算顺序调整为 $O(n)$。

---

## 参考资料

[^1]: Sennrich, R., Haddow, B., & Birch, A. (2015). Neural Machine Translation of Rare Words with Subword Units. *ACL*.

[^2]: Vaswani, A., et al. (2017). Attention Is All You Need. *NeurIPS*.

[^3]: Su, J., et al. (2024). RoFormer: Enhanced Transformer with Rotary Position Embedding. *ACL*.

[^4]: Geva, M., et al. (2020). Transformer Feed-Forward Layers Are Key-Value Memories. *EMNLP*.

[^5]: Ouyang, L., et al. (2022). Training language models to follow instructions with human feedback. *NeurIPS*.

[^6]: Rafailov, R., et al. (2023). Direct Preference Optimization: Your Language Model is Secretly a Reward Model. *NeurIPS*.

[^7]: Wei, J., et al. (2022). Emergent Abilities of Large Language Models. *TMLR*.

[^8]: Kaplan, J., et al. (2020). Scaling Laws for Neural Language Models. *arXiv*.

[^9]: Hoffmann, J., et al. (2022). Training Compute-Optimal Large Language Models. *NeurIPS*.

[^10]: Brown, T., et al. (2020). Language Models are Few-Shot Learners. *NeurIPS*.

[^11]: von Oswald, J., et al. (2023). Transformers as Meta-Learners for In-Context Learning. *NeurIPS*.

[^12]: Wei, J., et al. (2022). Chain-of-Thought Prompting Elicits Reasoning in Large Language Models. *NeurIPS*.

[^13]: Wang, X., et al. (2022). Self-Consistency Improves Chain of Thought Reasoning in Language Models. *ICLR*.

[^14]: Chen, S., et al. (2023). Extending Context Is Hard But Not Impossible. *arXiv*.

[^15]: Katharopoulos, A., et al. (2020). Transformers are RNNs: Fast Autoregressive Transformers with Linear Attention. *ICML*.
