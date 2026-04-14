---
title: 自然语言处理基础
date: 2026-04-08
description: 自然语言处理（NLP）基础：分词、词嵌入、Seq2Seq、Attention、Transformer
tags:
  - nlp
  - tokenization
  - word-embedding
  - seq2seq
  - attention
  - transformer
draft: false
permalink:
---

## 定义与范畴

自然语言处理（Natural Language Processing，NLP）是人工智能与语言学的交叉领域，旨在让计算机理解、生成和交互自然语言。[^1]

### NLP 任务分类

| 任务类型 | 示例 | 特点 |
|---------|------|------|
| **文本分类** | 垃圾邮件检测、情感分析 | 句子级别输出 |
| **序列标注** | 分词、词性标注、命名实体识别 | 每个token输出标签 |
| **序列生成** | 机器翻译、文本摘要 | 变长输出序列 |
| **文本匹配** | 问答系统、语义相似度 | 两段文本的关系 |

---

## 分词（Tokenization）

分词是将文本切分为最小语义单元的过程。

### 词级别分词

```python
# 简单空格分词
text = "The quick brown fox jumps"
tokens = text.split()
# ['The', 'quick', 'brown', 'fox', 'jumps']
```

### 子词级别分词

**BPE（Byte Pair Encoding）** 是最常用的子词算法：

```python
# BPE训练过程（伪代码）
def train_bpe(corpus, vocab_size):
    vocab = set(characters)
    pairs = count_adjacent_pairs(corpus)
    
    while len(vocab) < vocab_size:
        most_frequent = find_most_common_pair(pairs)
        new_token = merge_pair(most_frequent)
        vocab.add(new_token)
        update_pairs(corpus, most_frequent, new_token)
    
    return vocab
```

### 主流分词器对比

| 分词器 | 算法 | 特点 |
|-------|------|------|
| **WordPiece** | 贪心构建 | Google BERT使用 |
| **SentencePiece** | 统一框架 | 支持BPE/UNIGRAM |
| **BPE** | 频率合并 | GPT-2使用 |
| **Character-level** | 字符为单位 | 鲁棒性强，序列长 |

---

## 词嵌入（Word Embedding）

词嵌入将离散的词映射到连续的向量空间。

### One-Hot 编码

```python
import numpy as np

vocab = ["cat", "dog", "bird"]
vocab_size = len(vocab)

# One-Hot编码
cat = np.array([1, 0, 0])
dog = np.array([0, 1, 0])
bird = np.array([0, 0, 1])
```

问题：维度高（=|V|）、稀疏、无语义关系。

### Word2Vec

Word2Vec通过上下文学习词向量，包含两种架构：

**CBOW（Continuous Bag-of-Words）**：根据上下文预测中心词

$$
\mathcal{L} = -\log P(w_c | w_{c-m}, \dots, w_{c-1}, w_{c+1}, \dots, w_{c+m})
$$

**Skip-gram**：根据中心词预测上下文

$$
\mathcal{L} = -\sum_{-m \leq j \leq m, j \neq 0} \log P(w_{c+j} | w_c)
$$

```python
from gensim.models import Word2Vec

sentences = [["cat", "say", "meow"], ["dog", "say", "woof"]]
model = Word2Vec(sentences, vector_size=100, window=3, min_count=1)

# 词向量
vector = model.wv["cat"]  # 100维向量

# 语义相似度
similarity = model.wv.similarity("cat", "dog")
```

### GloVe

GloVe（Global Vectors）结合全局矩阵分解与局部上下文：

$$
J = \sum_{i,j} f(X_{ij})(w_i^T \tilde{w}_j + b_i + \tilde{b}_j - \log X_{ij})^2
$$

其中 $X_{ij}$ 是词 $i$ 在词 $j$ 上下文中出现的次数。

### Embedding Layer

深度学习中的可训练词嵌入：

```python
import torch
import torch.nn as nn

vocab_size = 10000
embed_dim = 256

embedding = nn.Embedding(vocab_size, embed_dim)
input_ids = torch.tensor([1, 5, 3, 2])  # batch=1, seq_len=4

embedded = embedding(input_ids)  # (1, 4, 256)
```

---

## Seq2Seq 与 Attention

Seq2Seq解决变长序列到变长序列的映射问题。

### Encoder-Decoder 架构

```
Encoder:                    Decoder:
h_t = f(x_t, h_{t-1})       y_t = g(y_{<t}, s_{t-1})
                              s_t = f(s_{t-1}, y_{t-1}, c_t)
```

### Attention 机制

Attention解决了Encoder信息压缩的问题：

$$
\alpha_{ti} = \frac{\exp(e_{ti})}{\sum_{j=1}^{T} \exp(e_{tj})}, \quad e_{ti} = v^T \tanh(W_h h_i + W_s s_{t-1})
$$

$$
c_t = \sum_{i=1}^{T} \alpha_{ti} h_i
$$

```python
class Attention(nn.Module):
    def __init__(self, hidden_dim):
        super().__init__()
        self.attn = nn.Linear(hidden_dim * 2, hidden_dim)
        self.v = nn.Linear(hidden_dim, 1, bias=False)
    
    def forward(self, decoder_hidden, encoder_outputs):
        # decoder_hidden: (1, hidden_dim)
        # encoder_outputs: (seq_len, hidden_dim)
        
        seq_len = encoder_outputs.size(0)
        decoder_hidden = decoder_hidden.repeat(seq_len, 1, 1)
        
        energy = torch.tanh(self.attn(
            torch.cat([decoder_hidden, encoder_outputs], dim=2)
        ))
        attention = self.v(energy).squeeze(2)
        
        return torch.softmax(attention, dim=0)
```

### Bahdanau Attention vs Luong Attention

| 特性 | Bahdanau | Luong |
|------|----------|-------|
| 注意力位置 | Decoder之前 | Decoder之后 |
| 拼接方式 | $W_a [s_{t-1}; h_i]$ | $W_a [s_t; h_i]$ |
| 输出 | $c_t$ 作为输入 | 直接用于输出层 |

---

## Transformer

Transformer完全基于Attention机制，摒弃了RNN。[^2]

### Self-Attention

Self-Attention计算序列内部的关系：

$$
\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

```python
import torch
import torch.nn.functional as F
import math

def self_attention(Q, K, V, mask=None):
    d_k = Q.size(-1)
    scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(d_k)
    
    if mask is not None:
        scores = scores.masked_fill(mask == 0, -1e9)
    
    attention_weights = F.softmax(scores, dim=-1)
    return torch.matmul(attention_weights, V), attention_weights
```

### Multi-Head Attention

多头注意力并行计算多个注意力：

$$
\text{MultiHead}(Q, K, V) = \text{Concat}(\text{head}_1, \ldots, \text{head}_h)W^O
$$

$$
\text{head}_i = \text{Attention}(QW_i^Q, KW_i^K, VW_i^V)
$$

```python
class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, num_heads):
        super().__init__()
        self.num_heads = num_heads
        self.d_model = d_model
        self.d_k = d_model // num_heads
        
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
    
    def split_heads(self, x):
        batch_size, seq_len = x.size(0), x.size(1)
        return x.view(batch_size, seq_len, self.num_heads, self.d_k).transpose(1, 2)
    
    def forward(self, query, key, value, mask=None):
        Q = self.split_heads(self.W_q(query))
        K = self.split_heads(self.W_k(key))
        V = self.split_heads(self.W_v(value))
        
        attn_output, attn_weights = self_attention(Q, K, V, mask)
        
        attn_output = attn_output.transpose(1, 2).contiguous()
        attn_output = attn_output.view(attn_output.size(0), -1, self.d_model)
        
        return self.W_o(attn_output)
```

### Positional Encoding

由于Transformer没有循环结构，需要添加位置信息：

$$
PE_{(pos, 2i)} = \sin\left(\frac{pos}{10000^{2i/d_{model}}}\right)
$$

$$
PE_{(pos, 2i+1)} = \cos\left(\frac{pos}{10000^{2i/d_{model}}}\right)
$$

```python
class PositionalEncoding(nn.Module):
    def __init__(self, d_model, max_len=5000):
        super().__init__()
        
        pe = torch.zeros(max_len, d_model)
        position = torch.arange(0, max_len).unsqueeze(1).float()
        div_term = torch.exp(
            torch.arange(0, d_model, 2).float() * (-math.log(10000.0) / d_model)
        )
        
        pe[:, 0::2] = torch.sin(position * div_term)
        pe[:, 1::2] = torch.cos(position * div_term)
        
        self.register_buffer('pe', pe.unsqueeze(0))
    
    def forward(self, x):
        return x + self.pe[:, :x.size(1)]
```

### Transformer 完整结构

```
Input Embedding + Positional Encoding
           ↓
     [Encoder Layer] × N
           ↓
     [Decoder Layer] × N
           ↓
        Linear + Softmax
```

---

## 预训练模型

### BERT（Bidirectional Encoder Representations from Transformers）

BERT使用MLM（Masked Language Model）进行训练：

```python
# MLM: 随机遮盖15%的token
input_ids = torch.tensor([101, 2003, 2305, 2023, 2154, 102])
labels = torch.tensor([-100, -100, 2305, -100, -100, -100])  # 仅计算遮盖位置

# BERT输出
outputs = bert_model(input_ids)
sequence_output = outputs.last_hidden_state  # (batch, seq_len, hidden)
pooled_output = outputs.pooler_output        # (batch, hidden)
```

### GPT（Generative Pre-trained Transformer）

GPT采用单向语言模型：

$$
\mathcal{L} = -\sum_i \log P(x_i | x_{<i}; \theta)
$$

### T5（Text-to-Text Transfer Transformer）

将所有NLP任务统一为文本到文本格式：

| 任务 | 输入 | 输出 |
|------|------|------|
| 翻译 | "translate English to French: hello" | "bonjour" |
| 摘要 | "summarize: article text..." | "summary..." |
| 问答 | "question: ... context: ..." | "answer" |

---

## 常用NLP工具

### Hugging Face Transformers

```python
from transformers import pipeline, AutoTokenizer, AutoModel

# 预训练pipeline
classifier = pipeline("sentiment-analysis")
result = classifier("I love this movie!")
# [{'label': 'POSITIVE', 'score': 0.9998}]

# 加载模型
model_name = "bert-base-chinese"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModel.from_pretrained(model_name)
```

###jieba分词

```python
import jieba

text = "北京大学的学生在研究自然语言处理"

# 精确模式
words = jieba.cut(text)
print("/".join(words))
# 北京大学/的/学生/在/研究/自然语言处理

# 全模式
words = jieba.cut(text, cut_all=True)
print("/".join(words))
# 北京/北京大字/大学/的/学生/在/研究/自然/自然语言/语言/处理
```

---

## 参考资料

[^1]: Speech and Language Processing - Daniel Jurafsky & James H. Martin. https://web.stanford.edu/~jurafsky/slp3/

[^2]: Attention Is All You Need - Vaswani et al. (2017). https://arxiv.org/abs/1706.03762

[^3]: BERT: Pre-training of Deep Bidirectional Transformers - Devlin et al. (2018). https://arxiv.org/abs/1810.04805
