---
title: RAG（检索增强生成）
date: 2026-04-14
description: 检索增强生成（Retrieval-Augmented Generation, RAG）— 让LLM访问外部知识库，提升答案准确性和时效性
tags:
  - rag
  - llm
  - ai-engineering
  - retrieval
draft: false
permalink:
---

# RAG（检索增强生成）

## 概述

检索增强生成（Retrieval-Augmented Generation, RAG）是一种将大型语言模型（LLM）与外部知识检索系统结合的技术架构。RAG 通过先从知识库中检索相关文档，再将检索结果作为上下文提供给 LLM 生成答案，从而克服 LLM 固有的知识截止日期限制和幻觉问题。

RAG 在 2026 年已成为企业 AI 应用的标准架构模式。根据行业统计，93% 的企业 AI 项目采用 RAG 架构来提供可信的问答能力。

## 核心原理

### RAG 流程

```
用户查询 → 检索阶段 → 增强阶段 → 生成阶段 → 回答
           ↓
      向量数据库
      检索Top-K相关文档
```

**三阶段流程：**

1. **检索阶段（Retrieval）**：将用户问题转换为向量，在知识库中检索最相关的 Top-K 文档
2. **增强阶段（Augmentation）**：将检索到的文档与原始问题组合成提示词（Prompt）
3. **生成阶段（Generation）**：LLM 基于增强后的上下文生成最终答案

### 为什么需要 RAG

| LLM 局限性 | RAG 解决方案 |
|-----------|-------------|
| 知识截止日期 | 实时检索最新文档 |
| 幻觉问题 | 答案基于检索到的真实文档 |
| 无法访问私有数据 | 检索私有知识库 |
| 领域知识不足 | 检索专业领域文档 |
| 答案不可溯源 | 提供答案的文档来源 |

## 技术架构

### 分层架构

```
┌─────────────────────────────────────────────────────┐
│                   应用层                             │
│   (Chatbot、问答系统、文档助手)                      │
├─────────────────────────────────────────────────────┤
│                   生成层                             │
│   (LLM: GPT-4、Claude、Qwen)                       │
├─────────────────────────────────────────────────────┤
│                   检索层                             │
│   (向量检索、关键词检索、混合检索、重排序)            │
├─────────────────────────────────────────────────────┤
│                   索引层                             │
│   (Embedding模型、分块策略、元数据)                  │
├─────────────────────────────────────────────────────┤
│                   数据层                             │
│   (向量数据库、知识文档存储)                         │
└─────────────────────────────────────────────────────┘
```

### RAG 流水线类型

#### 简单 RAG（Naive RAG）

最基本的架构：分块 → Embedding → 存储 → 检索 → 生成。

**局限性：**
- 精确度有限：语义相似性不等于实际相关性
- 上下文长度限制：无法处理大量检索文档
- 缺少重排序：Top-K 结果可能包含噪声

#### 高级 RAG（Advanced RAG）

针对简单 RAG 的改进：

1. **混合检索（Hybrid Search）**：结合向量检索 + BM25 关键词检索
2. **重排序（Reranking）**：使用 Cross-Encoder 对检索结果进行二次排序
3. **查询转换（Query Transformation）**：HyDE、查询扩展、查询改写

#### 模块化 RAG（Modular RAG）

将检索、排序、生成解耦，可灵活组合：

- **检索模块**：向量检索、BM25、SQL 检索
- **排序模块**：CCCR、RRF、LM-based Reranker
- **记忆模块**：对话历史、对话摘要
- **融合模块**：多查询扩展、多文档组合

## 关键技术细节

### 文档分块（Chunking）

分块策略直接影响检索质量，是 RAG 系统最重要的超参数。

#### 固定大小分块

```
chunk_size = 512 tokens
chunk_overlap = 50 tokens
```

**优点**：简单、可重复
**缺点**：可能切断语义单元

#### 结构化分块

根据文档结构（标题、段落、表格）进行分块：

```python
def structural_chunking(document):
    chunks = []
    for section in document.sections:
        if section.type == "heading":
            # 作为新块的开始
            chunks.append(create_chunk(section))
        elif section.type == "table":
            # 表格作为独立块
            chunks.append(create_chunk(section, preserve_structure=True))
    return chunks
```

**保留表格结构**：表格块应保持表格格式，避免将单元格内容混入普通文本。

#### 语义分块

使用轻量级模型识别自然语义断点：

```python
def semantic_chunking(text, model="bge-small"):
    embeddings = model.encode(text, split_by="sentence")
    # 在语义断点处分割
    return find_semantic_boundaries(embeddings)
```

### 嵌入模型选择

| 模型 | 维度 | 成本 | 适用场景 |
|------|------|------|----------|
| text-embedding-3-large | 3072 | $0.13/1M | 通用、高精度 |
| text-embedding-3-small | 1536 | $0.02/1M | 成本敏感 |
| voyage-3 | 1024 | $0.06/1M | 代码/技术内容 |
| BGE-large-en-v1.5 | 1024 | 免费自托管 | 隐私敏感 |

**选择建议**：
- 通用场景：`text-embedding-3-small` 性价比最高
- 代码密集型：使用 `voyage-3` 或 CodeBERT
- 隐私敏感：使用开源模型自托管

### 检索策略

#### 向量检索（Dense Retrieval）

基于语义相似性：

```python
results = vector_db.search(
    query_vector=query_embedding,
    top_k=20,
    filter={"source": "documentation"}
)
```

#### 关键词检索（BM25/Sparse Retrieval）

基于词频和逆文档频率：

```python
results = bm25_index.search(query, top_k=20)
```

**适用场景**：
- 精确匹配（产品 ID、专业术语）
- 用户明确知道要查找的内容

#### 混合检索（Hybrid Search）

结合向量和关键词检索：

```python
# 1. 向量检索
vector_results = vector_search(query, top_k=20)

# 2. BM25 检索
bm25_results = bm25_search(query, top_k=20)

# 3. 分数融合（RRF）
combined = reciprocal_rank_fusion(vector_results, bm25_results, k=60)
```

**RRF（Reciprocal Rank Fusion）公式**：

$$
\text{RRF}(d) = \sum_{i} \frac{1}{k + \text{rank}_i(d)}
$$

其中 $k$ 通常取 60。

### 重排序（Reranking）

使用 Cross-Encoder 对初始检索结果进行精细排序：

```python
reranker = CohereRerank(model="rerank-multilingual-v3.0")

reranked = reranker rerank(
    query=query,
    documents=initial_results,
    top_n=5  # 只保留最相关的5个
)
```

**重排序收益**：
- 延迟增加 50-200ms
- 精度提升 15-30%

### 上下文压缩

减少输入 token，降低成本：

```python
compressor = ContextualCompressor(model="gpt-4")

compressed = compressor.compress(
    document=retrieved_chunk,
    query=user_question
)
```

## 生产级架构

### 三层架构

```
┌──────────────────────────────────────────────────────────┐
│                    摄取层 (Ingestion)                    │
│  文档解析 → 分块 → Embedding → 向量存储                  │
│  异步批处理，支持 PDF/DOCX/HTML/Markdown                 │
├──────────────────────────────────────────────────────────┤
│                    检索层 (Retrieval)                    │
│  混合检索 → 重排序 → 上下文压缩                         │
│  QPS > 1000 的低延迟检索                                │
├──────────────────────────────────────────────────────────┤
│                    生成层 (Generation)                   │
│  Prompt 构建 → LLM 调用 → 响应验证                     │
│  支持流式输出、Function Calling                         │
└──────────────────────────────────────────────────────────┘
```

### 配置示例

```yaml
# docker-compose.yml
version: '3.8'
services:
  ingestion:
    image: rag-ingestion-service
    environment:
      BATCH_SIZE: 50
      WORKERS: 4

  vector-db:
    image: qdrant/qdrant
    ports:
      - "6333:6333"

  reranker:
    image: cohere/rerank
    environment:
      MODEL: rerank-multilingual-v3.0
```

### 评估指标

使用 RAGAS 框架评估：

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy

result = evaluate(
    dataset=eval_dataset,
    metrics=[faithfulness, answer_relevancy]
)

# 生产级目标：
# - faithfulness > 0.85（答案基于检索上下文）
# - answer_relevancy > 0.80（答案真正回答问题）
```

## 生产最佳实践

### 1. 数据质量优先

> "Garbage in, garbage out" — 数据质量比检索算法更重要

- 清洗源文档，移除噪音
- 保留文档结构（元数据、标题、层级）
- 提取表格为独立块

### 2. 监控分离

| 指标 | 监控目标 |
|------|----------|
| 检索精度 | Hit Rate、MRR |
| 生成质量 | RAGAS 各维度得分 |
| 系统延迟 | P50/P95/P99 |
| 成本 | $ per query |

### 3. 缓存策略

- **Embedding 缓存**：相同文档不重复 embedding
- **Query 缓存**：常见问题缓存完整响应
- **语义缓存**：相似 query 复用结果

```python
# 语义缓存示例
cache = SemanticCache(vector_db, threshold=0.95)
cached_result = cache.get(query)
if cached_result:
    return cached_result  # 命中缓存
```

### 4. 知识截止管理

```python
def get_knowledge_cutoff(prompt: str) -> str:
    # 在 Prompt 中明确知识截止日期
    return f"""已知知识截止到 2024-06。
    
    如果以下上下文包含相关信息，请基于它回答。
    如果不包含，请说明你不知道。"""
```

## 常见问题与解决方案

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 答案不准确 | 检索质量差 | 改进分块策略、使用混合检索 |
| 检索不到相关内容 | Embedding 不匹配领域 | 微调 Embedding 模型 |
| 表格信息丢失 | 分块破坏表格结构 | 使用表格专用分块器 |
| 延迟过高 | 向量数据库未优化 | 优化 HNSW 参数、增加缓存 |
| 成本过高 | Token 消耗大 | 上下文压缩、缓存、选择更小模型 |

## 相关主题

- [[machine-learning/llm/vector-database|向量数据库]] — RAG 的存储基础设施
- [[machine-learning/llm/llm-evaluation|LLM 评估]] — RAG 系统质量评估
- [[machine-learning/transformer-and-attention|Transformer 与注意力机制]] — LLM 基础架构
- [[machine-learning/nlp-basics|NLP 基础]] — 文本处理前置知识

---

## 参考资料

[^1]: [RAG Best Practices: Building Production-Ready Systems in 2026](https://devstarsj.github.io/2026/03/22/rag-retrieval-augmented-generation-production-best-practices-2026/)
[^2]: [Building Production RAG Systems: Complete Guide](https://zenvanriel.com/ai-engineer-blog/building-production-rag-systems-complete-guide/)
[^3]: [AWS: Writing best practices to optimize RAG applications](https://docs.aws.amazon.com/pdfs/prescriptive-guidance/latest/writing-best-practices-rag/writing-best-practices-rag.pdf)
[^4]: [Architecting Enterprise RAG: Vector DBs & Retrieval Strategy](https://www.developers.dev/tech-talk/architecting-enterprise-grade-rag-engineering-trade-offs-in-vector-databases-and-retrieval-pipelines.html)
