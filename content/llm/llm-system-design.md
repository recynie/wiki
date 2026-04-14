---
title: LLM System Design
date: 2026-04-13
description: LLM系统设计：RAG架构、分块策略、Embedding模型与向量数据库
tags:
  - llm
  - rag
  - vector-database
  - system-design
draft: true
permalink:
---

## 1. RAG 架构概述

RAG（Retrieval-Augmented Generation，检索增强生成）是一种将外部知识检索与LLM生成结合的架构，旨在解决LLM的幻觉问题和知识时效性问题。

### 1.1 Naive RAG

Naive RAG 是最基础的RAG范式，流程简洁：

```
用户查询 → 向量检索 → 获取Top-K文档 → 拼接到Prompt → LLM生成
```

**优点**：实现简单，延迟低
**缺点**：检索与生成解耦不紧密，容易出现文档与查询不匹配的问题

### 1.2 Advanced RAG

Advanced RAG 在 Naive RAG 的基础上引入多个优化环节：

```
┌─────────────────────────────────────────────────────────────┐
│                      Advanced RAG 流程                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  用户查询                                                    │
│     │                                                       │
│     ▼                                                       │
│  ┌──────────┐    ┌───────────┐    ┌──────────┐              │
│  │Query     │───▶│ Retrieval │───▶│  Rerank  │              │
│  │Rewriting │    │           │    │          │              │
│  └──────────┘    └───────────┘    └──────────┘              │
│                      │                   │                   │
│                      │  Hybrid Search    │                   │
│                      ▼                   │                   │
│               ┌──────────────┐          │                   │
│               │ Vector Store │          │                   │
│               │   + BM25     │          │                   │
│               └──────────────┘          │                   │
│                                          ▼                   │
│                                   ┌──────────────┐          │
│                                   │   Generate   │          │
│                                   │     (LLM)    │          │
│                                   └──────────────┘          │
│                                          │                   │
│                                          ▼                   │
│                                    最终回复                   │
└─────────────────────────────────────────────────────────────┘
```

**关键技术环节**：

| 环节 | 作用 | 常见方法 |
|------|------|----------|
| Query Rewriting | 改写查询以获得更好的检索效果 | HyDE、query expansion |
| Hybrid Search | 结合稠密和稀疏检索 | Dense + BM25 |
| Reranking | 对检索结果重排序 | Cross-encoder、LLM-based |

### 1.3 RAG 完整流程

RAG 系统可分为三个主要阶段：

#### Indexing（索引阶段）

```
文档 → 文本提取 → 分块(Chunking) → Embedding → 向量数据库存储
```

此阶段是离线阶段，决定了后续检索的质量上限。

#### Retrieval（检索阶段）

```
用户查询 → 向量化 → 向量相似度搜索 → 召回Top-K → 重排序
```

#### Generation（生成阶段）

```
原始查询 + 检索结果 → Prompt组装 → LLM生成 → 返回结果
```

## 2. 分块策略

分块（Chunking）是将文档切分为适合检索的最小单元，直接影响检索精度和生成质量。

### 2.1 固定大小分块

最简单的分块方式，按指定token数或字符数均匀切分：

```python
def fixed_size_chunk(text: str, chunk_size: int = 500, overlap: int = 50) -> list[str]:
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end])
        start = end - overlap  # 保留重叠区域
    return chunks
```

**优点**：实现简单，一致性好
**缺点**：可能切断句子、段落等语义单元

### 2.2 递归字符分块

按层级结构递归切分：段落 → 句子 → 单词：

```
文档
  │
  ▼
┌─────────┐    ┌─────────┐
│ 段落1   │───▶│ 句子1.1 │
│ 段落2   │    │ 句子1.2 │───▶ 递归切分
│ ...     │    │ ...     │
└─────────┘    └─────────┘
```

常用分隔符：`["\n\n", "\n", ". ", " ", ""]`

```python
def recursive_chunk(text: str, separators: list[str] = ["\n\n", "\n", ". ", " "]) -> list[str]:
    # 递归地将文本按分隔符列表由粗到细切分
    pass
```

### 2.3 语义分块

基于句子级别的语义相似度进行分块：

```python
def semantic_chunk(sentences: list[str], threshold: float = 0.7) -> list[str]:
    """将语义相近的连续句子归为同一块"""
    embeddings = embed_model.encode(sentences)
    chunks = []
    current_chunk = [sentences[0]]
    
    for i in range(1, len(sentences)):
        similarity = cosine_sim(embeddings[i-1], embeddings[i])
        if similarity > threshold:
            current_chunk.append(sentences[i])
        else:
            chunks.append(" ".join(current_chunk))
            current_chunk = [sentences[i]]
    
    if current_chunk:
        chunks.append(" ".join(current_chunk))
    return chunks
```

### 2.4 Chunk Size 选择

选择合适的 chunk size 需要权衡以下因素：

| 因素 | 小Chunk | 大Chunk |
|------|---------|---------|
| 精确度 | 高（噪声少） | 低（可能包含无关内容） |
| 完整性 | 低（可能丢失上下文） | 高（保留完整语义） |
| 检索速度 | 快 | 慢 |
| 内存占用 | 低 | 高 |

**一般建议**：

- **通用场景**：512 tokens
- **长文档问答**：256-512 tokens
- **代码检索**：128-256 tokens（保持函数级别）
- **对话场景**：128-256 tokens

**注意**：chunk size 应小于 embedding 模型的 context window（通常 512 或 1024 tokens），保留 overlap（15-25%）以维持上下文连贯性。

## 3. Embedding 模型

Embedding 模型负责将文本映射为稠密向量，是 RAG 系统中连接检索与生成的关键组件。

### 3.1 常见模型

#### Encoder-only 模型（BERT 系列）

| 模型 | 维度 | 特点 |
|------|------|------|
| bert-base-uncased | 768 | 通用，英文为主 |
| bert-multilingual | 768 | 多语言支持 |
| text2vec-large-chinese | 1024 | 中文优化 |

#### OpenAI 模型

| 模型 | 维度 | context | MTEB 得分 |
|------|------|---------|-----------|
| text-embedding-ada-002 | 1536 | 8191 | ~60 |
| text-embedding-3-small | 1536 | 8191 | ~65 |
| text-embedding-3-large | 3072 | 8191 | ~67 |

#### 开源模型

| 模型 | 维度 | MTEB | 说明 |
|------|------|------|------|
| MPNet | 768 | ~65 | 通用，高效 |
| E5-small | 384 | ~63 | 轻量级 |
| BGE-large-zh | 1024 | ~68 | 中文优化 |
| GTE-large | 1024 | ~67 | 华为开源 |

### 3.2 MTEB Benchmark

MTEB（Massive Text Embedding Benchmark）是评估 Embedding 模型的标准基准，涵盖 58 个数据集、112 种任务类型。

**常见任务类型**：

- **Retrieval**：信息检索
- **Clustering**：文本聚类
- **Classification**：文本分类
- **Pairwise Classification**：句子对分类
- **Reranking**：重排序
- **STS**：语义相似度
- **Summarization**：摘要评估

### 3.3 Domain-Specific Fine-tuning

通用 Embedding 在特定领域可能表现不佳，需要微调：

```python
# 对比学习微调框架
def contrastive_loss(anchor, positive, negative, temperature: float = 0.01):
    # anchor: 查询
    # positive: 相关文档
    # negative: 不相关文档
    pos_sim = cos_sim(anchor, positive) / temperature
    neg_sim = cos_sim(anchor, negative) / temperature
    return softmax_cross_entropy([pos_sim, neg_sim])
```

**微调策略**：

1. **数据收集**：领域相关查询-文档对
2. **硬负例挖掘**：使用通用模型召回的"近似负例"
3. **混合预训练**：先通用再领域

## 4. 向量数据库

向量数据库用于存储 Embedding 向量并支持高效相似度检索。

### 4.1 索引类型

#### HNSW（Hierarchical Navigable Small World）

```
层3:  ●────────────●           (最稀疏)
      │             │
层2:  ●────●──●────●──●         (中层)
      │    │  │    │  │
层1:  ●────●──●──●─●──●──●      (最稠密)
```

- **原理**：构建多层图结构，上层稀疏、下层稠密
- **优点**：高召回率（~95%），低延迟（对数级）
- **缺点**：内存占用高，构建慢
- **适用场景**：追求高召回的在线服务

#### IVF（Inverted File Index）

- **原理**：将向量空间聚类为N个簇，检索时只搜索最近的K个簇
- **优点**：内存占用低，可控召回率
- **缺点**：需要额外配置 nprobe 参数
- **适用场景**：大规模数据

#### PQ（Product Quantization）

- **原理**：将高维向量分割为多段，每段独立量化
- **优点**：极大压缩存储（可压缩 90%+）
- **缺点**：召回率略有下降
- **适用场景**：超大规模数据

**组合索引**：生产环境常用 `HNSW + IVF` 或 `HNSW + PQ` 组合。

### 4.2 常见向量数据库选型

| 数据库 | 类型 | 优势 | 适用场景 |
|--------|------|------|----------|
| Pinecone | 云服务 | 全托管，易用 | 快速上线 |
| Weaviate | 自部署/云 | 混合搜索强 | 需要BM25 |
| Milvus | 自部署 | 成熟，功能全 | 大规模部署 |
| Qdrant | 自部署 | Rust，高性能 | 低延迟需求 |
| pgvector | PostgreSQL扩展 | 兼容现有DB | 轻量级/中小规模 |
| Chroma | 嵌入式 | 轻量，易用 | POC/原型 |

### 4.3 关键指标

| 指标 | 说明 | 目标值（参考） |
|------|------|---------------|
| 召回率@K | Top-K结果中相关文档比例 | >95%（在线服务） |
| QPS | 每秒查询数 | 根据业务峰值设计 |
| Latency | P99 延迟 | <100ms（在线） |
| 索引构建时间 | 批量导入耗时 | 视数据量而定 |

## 5. 混合搜索

混合搜索（Hybrid Search）结合稠密检索（Dense Retrieval）和稀疏检索（Sparse Retrieval），以平衡精确性和语义理解能力。

### 5.1 Dense vs Sparse Retrieval

| 维度 | Dense Retrieval | Sparse Retrieval (BM25) |
|------|-----------------|-------------------------|
| 表示方式 | 稠密向量 | 高维稀疏向量（词袋） |
| 语义理解 | 强 | 弱 |
| 关键词匹配 | 弱 | 强 |
| 冷启动 | 需要训练 | 无需训练 |
| 内存占用 | 高 | 低 |

### 5.2 Reciprocal Rank Fusion (RRF)

RRF 是一种简单有效的多路召回融合算法：

$$
\text{RRF}(d) = \sum_{i=1}^{n} \frac{1}{k + \text{rank}_i(d)}
$$

其中 $k$ 通常取 60，$\text{rank}_i(d)$ 是文档 $d$ 在第 $i$ 路召回中的排名。

```python
def reciprocal_rank_fusion(results_list: list[list[str]], k: int = 60) -> list[str]:
    """融合多路检索结果"""
    scores = defaultdict(float)
    
    for results in results_list:
        for rank, doc_id in enumerate(results):
            scores[doc_id] += 1.0 / (k + rank + 1)
    
    return sorted(scores.keys(), key=lambda x: scores[x], reverse=True)
```

### 5.3 何时使用混合搜索

**推荐使用**：

- 文档包含大量专有名词、术语（如法律、医疗）
- 用户查询常为短查询，语义信息不足
- 需要同时捕获关键词匹配和语义相似度
- 对召回率要求 >90%

**可不使用**：

- 文档集合小（<10万）
- 查询以长句为主，语义信息充足
- 对延迟要求极高（混合搜索增加复杂度）

## 参考资料

[^1]: RAG（Retrieval-Augmented Generation） - Hugging Face: https://huggingface.co/docs/transformers/model_doc/rag
[^2]: MTEB: Massive Text Embedding Benchmark: https://arxiv.org/abs/2305.14251
[^3]: BGE Embedding Model: https://github.com/FlagOpen/FlagEmbedding
[^4]: Hybrid Search — Weaviate Documentation: https://weaviate.io/developers/weaviate/search/hybrid
