---
title: 向量数据库
date: 2026-04-14
description: 向量数据库（Vector Database）— 存储和检索高维向量嵌入的核心基础设施，支撑现代 AI 应用的语义搜索与 RAG 系统
tags:
  - vector-database
  - llm
  - ai-engineering
  - embedding
  - ann
draft: false
permalink:
---

# 向量数据库

## 概述

向量数据库是专门用于存储、索引和检索高维向量（Vector）的数据库系统。在 AI 和大语言模型（LLM）应用场景中，向量通常由 embedding 模型生成，能够将文本、图像、音频等非结构化数据转换为稠密向量表示，从而支持语义级别的相似性搜索。

随着 RAG（检索增强生成）架构在 2026 年成为企业 AI 应用的标准范式，向量数据库已从"nice-to-have"演变为不可或缺的核心基础设施。据统计，超过 85% 的生产级 RAG 系统依赖向量数据库提供低延迟、高精度的语义检索能力。

**核心价值**：传统关系型数据库擅长精确匹配（WHERE id = 1），而向量数据库擅长语义相似性搜索（FIND TOP-K NEAREST vectors），这正是 AI 应用所需的核心能力。

## 核心概念

### 向量嵌入

向量嵌入（Vector Embedding）是将离散的非结构化数据映射到连续向量空间的技术。嵌入后的向量具有"语义相似性"特性：含义相近的内容在向量空间中距离更近。

**文本嵌入的数学表示**：

设文本 $t$ 经过 embedding 模型 $f$ 映射为 $d$ 维向量：

$$
\mathbf{v} = f(t) \in \mathbb{R}^d
$$

其中 $d$ 通常为 768、1024、1536 或 3072。向量范数通常归一化为 1，此时向量点积等价于余弦相似度。

**主流 embedding 模型**：

| 模型 | 维度 | 特点 |
|------|------|------|
| text-embedding-3-large | 3072 | OpenAI 最新，高精度 |
| text-embedding-3-small | 1536 | 高性价比 |
| BGE-large-en-v1.5 | 1024 | 开源自托管 |
| voyage-3 | 1024 | 代码/技术内容优化 |

### ANN 算法

精确的最近邻搜索（Exact Nearest Neighbor）在高维空间中时间复杂度为 $O(N \cdot d)$，无法满足大规模数据的需求。近似最近邻搜索（Approximate Nearest Neighbor, ANN）通过牺牲少量精度换取数量级的性能提升。

#### HNSW（Hierarchical Navigable Small World）

HNSW 是一种基于图的多层近似最近邻算法，检索质量高、延迟低，是当前最流行的 ANN 算法。

**核心思想**：构建多层图结构，上层稀疏、下层稠密，搜索时从顶层快速定位局部区域，再逐层精细搜索。

**关键参数**：

- `M`：每层连接数，通常 16-64
- `efConstruction`：构建时的动态列表大小，通常 100-400
- `efSearch`：搜索时的动态列表大小，通常 50-200

```python
# Qdrant HNSW 配置示例
client.create_collection(
    collection_name="documents",
    vector_size=1536,
    hnsw_config={
        "m": 32,           # M 参数
        "ef_construct": 200,  # efConstruction
    }
)
```

**复杂度分析**：

- 构建时间：$O(N \log N)$
- 检索时间：$O(\log N)$ 到 $O(\sqrt{N})$，取决于召回率要求
- 内存占用：$O(N \cdot d \cdot \text{overhead})$

#### IVF（Inverted File Index）

IVF 是一种基于聚类的索引方法，将向量空间划分为 $K$ 个聚类（Voroni cells），搜索时只检索最相关的少数聚类。

**工作流程**：

1. 使用 K-Means 将 $N$ 个向量聚类为 $K$ 个中心
2. 构建倒排索引：每个聚类中心关联其所属的向量列表
3. 搜索时，先找到最近的 $nprobe$ 个聚类中心，再在对应列表中精确搜索

```python
# Milvus IVF-Flat 配置示例
collection.schema = {
    "fields": [
        {"name": "id", "type": "int64"},
        {"name": "embedding", "type": "float32", "dim": 1536}
    ],
    "indexes": [{
        "field": "embedding",
        "index_type": "IVF_FLAT",
        "params": {"nlist": 1024, "nprobe": 32}
    }]
}
```

#### PQ（Product Quantization）

PQ 通过将高维向量压缩为低维码书表示，大幅降低内存占用。适用于超大规模数据集（数亿级别）。

**压缩原理**：

将 $d$ 维向量划分为 $m$ 个子向量，每个子向量独立量化：

$$
\mathbf{v} = [\mathbf{v}_1, \mathbf{v}_2, ..., \mathbf{v}_m] \xrightarrow{\text{量化}} [q_1, q_2, ..., q_m]
$$

压缩比可达 $4 \times$ 到 $64 \times$，检索时通过查表快速计算近似距离。

**适用场景**：内存受限的超大规模检索（$N > 1000$ 万向量）

### 距离度量

向量数据库使用多种距离度量来评估向量间的相似性：

| 度量方式 | 公式 | 适用场景 |
|----------|------|----------|
| **余弦相似度** | $\frac{\mathbf{a} \cdot \mathbf{b}}{\|\mathbf{a}\| \|\mathbf{b}\|}$ | 文本相似性、推荐系统 |
| **点积（Dot Product）** | $\mathbf{a} \cdot \mathbf{b}$ | 归一化向量的精确相似性 |
| **欧氏距离（L2）** | $\sqrt{\sum_i (a_i - b_i)^2}$ | 图像检索、聚类分析 |

**选择建议**：

- **余弦相似度**：最通用的选择，对向量范数不敏感
- **点积**：计算最快，但需确保向量已归一化；未归一化时偏向检索长向量
- **欧氏距离**：适合几何距离有物理意义的场景

**归一化与点积的关系**：若 $\|\mathbf{a}\| = \|\mathbf{b}\| = 1$，则：

$$
\text{cosine}(\mathbf{a}, \mathbf{b}) = \mathbf{a} \cdot \mathbf{b}
$$

因此，很多场景用归一化后的点积代替余弦相似度以提升性能。

## 主流解决方案

### 专用向量数据库

#### Pinecone

Pinecone 是云原生的全托管向量数据库，以易用性和稳定性著称。

**特点**：

- 完全托管，无需运维
- 支持元数据过滤和混合搜索
- 强一致性保证
- 延迟：P95 < 50ms（百万级数据）

**定价**：按向量数和查询次数计费，适合中小规模（< 1 亿向量）。

#### Weaviate

Weaviate 是开源的向量搜索引擎，支持 GraphQL 和 RESTful API。

**特点**：

- 原生支持混合搜索（向量 + BM25）
- 模块化架构，可连接多种 embedding 服务
- 支持 GraphQL 懒得查询
- 延迟：P95 < 30ms

**适用场景**：需要快速迭代的中小团队，混合搜索需求强。

#### Milvus

Milvus 是 LF AI & Data 基金会下的开源项目，专为超大规模向量设计。

**特点**：

- 支持数十亿向量规模
- 多种索引类型（HNSW、IVF、PQ、DiskANN）
- 混合标量过滤能力强
- 可部署为分布式集群

**适用场景**：大型企业，超大规模（> 1 亿向量）需求。

#### Qdrant

Qdrant 是 Rust 实现的高性能向量数据库，以延迟极低著称。

**特点**：

- Rust 实现，内存安全、性能优异
- 支持 Payload 过滤和复杂查询条件
- 接近 HNSW 实现，召回率高
- 延迟：P99 < 10ms

**适用场景**：对延迟敏感的业务场景，中大规模（< 10 亿向量）。

### 数据库扩展方案

#### pgvector（PostgreSQL）

pgvector 是 PostgreSQL 的向量搜索扩展，将向量数据存储在关系型数据库中。

**特点**：

- 零运维，集成到现有 PostgreSQL 生态
- 支持 HNSW 和 IVF 两种索引
- 可与其他数据联合查询
- 延迟：P95 < 100ms（百万级）

**适用场景**：

- 数据量 < 1000 万向量
- 需要与结构化数据联合查询
- 团队缺乏运维专用向量数据库的能力

```sql
-- pgvector 示例
CREATE EXTENSION vector;

CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,
    content TEXT,
    embedding vector(1536)
);

-- 创建 HNSW 索引
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);

-- 检索
SELECT content, 1 - (embedding <=> $1) AS similarity
FROM documents
WHERE metadata->>'source' = 'blog'
ORDER BY embedding <=> $1
LIMIT 5;
```

#### ChromaDB

ChromaDB 是为 AI 应用设计的开源向量数据库，Python-first，API 简洁。

**特点**：

- 专为 LLM 应用设计
- 内置 embedding 功能
- 支持持久化和临时模式
- 轻量级，易于原型开发

**适用场景**：AI 应用原型开发、小规模应用（< 100 万向量）。

```python
import chromadb

client = chromadb.Client()
collection = client.create_collection("documents")

collection.add(
    documents=["文档内容1", "文档内容2"],
    embeddings=[[0.1, 0.2, ...], [0.3, 0.4, ...]],
    metadatas=[{"source": "blog"}, {"source": "wiki"}],
    ids=["doc1", "doc2"]
)

results = collection.query(
    query_texts=["查询内容"],
    n_results=5
)
```

### 方案对比

| 方案 | 规模上限 | P95 延迟 | 运维复杂度 | 月成本估算（参考） | 适用规模 |
|------|----------|---------|------------|-------------------|----------|
| **Pinecone** | ~10 亿 | < 50ms | 零（托管） | $500+（百万向量） | 中等 |
| **Weaviate** | ~10 亿 | < 30ms | 中（需部署） | $300+（自托管） | 中等 |
| **Milvus** | 100 亿+ | < 100ms | 高（分布式） | $500+（集群） | 大规模 |
| **Qdrant** | ~10 亿 | < 10ms | 中 | $400+（集群） | 中大规模 |
| **pgvector** | 1000 万 | < 100ms | 低 | $100+（RDS） | 小规模 |
| **ChromaDB** | 100 万 | < 50ms | 极低 | $0（本地） | 原型/小规模 |

**选型建议**：

- **原型/MVP**：ChromaDB 或 pgvector
- **中小规模（< 1000 万向量）**：pgvector 或 Qdrant
- **中等规模（1000 万-10 亿）**：Pinecone、Weaviate、Qdrant
- **超大规模（> 10 亿）**：Milvus

## 架构模式

### 单层与分层架构

#### 单层集合架构

所有向量存储在单一集合（Collection）中，通过分区（Partition）实现逻辑隔离。

**优点**：

- 架构简单，易于维护
- 跨分区检索延迟低
- 资源利用率高

**缺点**：

- 全局索引膨胀，检索性能随数据量下降
- 无法针对不同数据类型优化索引参数

```
┌─────────────────────────────────────────┐
│            Collection: All              │
│  ┌──────────┬──────────┬──────────┐    │
│  │ Partition│ Partition│ Partition│    │
│  │  (Blog)  │  (Wiki)  │  (Docs) │    │
│  └──────────┴──────────┴──────────┘    │
└─────────────────────────────────────────┘
```

#### 分层集合架构

按数据类型或用途拆分为多个独立集合，通过统一网关路由查询。

**优点**：

- 可针对不同数据类型优化索引参数
- 支持不同 SLA 级别
- 故障隔离性好

**缺点**：

- 跨集合检索需要多次查询和合并
- 架构复杂度增加

```
┌─────────────┐   ┌─────────────┐
│ Collection  │   │ Collection  │
│   (Blog)    │   │   (Wiki)    │
│  HNSW m=64  │   │  HNSW m=32  │
└─────────────┘   └─────────────┘
```

### 分片与复制

#### 分片（Sharding）

水平扩展的核心手段，将数据分散到多个节点：

- **基于哈希的分片**：按向量 ID 哈希分布，适合均匀访问模式
- **基于距离的分片**：将相似向量分配到同一分片，减少跨分片查询

```
┌─────────────┐     ┌─────────────┐
│   Shard 1   │     │   Shard 2   │
│  0-5000万   │     │ 5000万-1亿  │
│  Node A     │     │  Node B     │
└─────────────┘     └─────────────┘
```

#### 复制（Replication）

数据复制到多个节点，提供：

- **高可用**：单节点故障不影响服务
- **读写分离**：写主节点，读可路由到副本
- **地理分布**：多地域部署降低访问延迟

**一致性级别**：

| 级别 | 描述 | 适用场景 |
|------|------|----------|
| 强一致性 | 所有副本同步确认写入 | 数据一致性优先 |
| 最终一致性 | 异步复制，允许短暂不一致 | 延迟敏感场景 |
| 读-your-writes | 保证自己能读到自己的写入 | 用户数据 |

### 索引类型与权衡

| 索引类型 | 召回率 | 延迟 | 内存占用 | 适用场景 |
|----------|--------|------|----------|----------|
| **HNSW** | 95-99% | 低 | 高（~2×原始） | 高精度需求 |
| **IVF-Flat** | 90-95% | 中 | 中（~1.2×原始） | 平衡场景 |
| **IVF-PQ** | 80-90% | 低 | 低（~0.1×原始） | 超大规模 |
| **DiskANN** | 90-95% | 中 | 低（磁盘存储） | 超大规模、内存受限 |

**HNSW 参数调优**：

```python
# 高召回率配置（适合 RAG）
hnsw_config = {
    "m": 64,              # 增加连接数，提升召回率
    "ef_construct": 400, # 增加构建质量
    "ef_search": 200     # 增加搜索宽度
}

# 低延迟配置（适合实时搜索）
hnsw_config = {
    "m": 16,              # 减少连接数
    "ef_construct": 100,
    "ef_search": 50
}
```

## 生产环境考量

### 规模阈值与迁移决策

pgvector 适合早期阶段，但随数据量增长会面临性能瓶颈。以下是经验性的迁移阈值：

| 数据规模 | 推荐方案 | 迁移触发信号 |
|----------|----------|--------------|
| < 10 万 | pgvector/ChromaDB | — |
| 10 万 - 100 万 | pgvector | 单次查询 > 50ms |
| 100 万 - 1000 万 | pgvector/Qdrant | P99 > 100ms |
| 1000 万 - 1 亿 | Qdrant/Pinecone | 内存 > 64GB |
| > 1 亿 | Milvus | 水平扩展需求 |

**迁移注意事项**：

1. **验证索引参数**：不同数据库的 HNSW 参数含义可能不同
2. **重新评估召回率**：迁移后需验证召回率不低于 95%
3. **双写过渡**：新旧系统并行写入，渐进切换流量

### 元数据过滤

生产环境中，几乎所有向量检索都需要结合元数据过滤（如按时间、来源、用户 ID 等条件筛选）。

**过滤执行时机**：

- **Pre-filtering**：先过滤再检索，可能丢失相关结果
- **Post-filtering**：先检索再过滤，可能返回结果不足
- **混合过滤**：索引时构建元数据倒排，检索时合并

```python
# Qdrant 元数据过滤示例
from qdrant_client.models import Filter, FieldCondition, MatchValue, Range

results = client.search(
    collection_name="documents",
    query_vector=query_embedding,
    query_filter=Filter(
        must=[
            FieldCondition(
                key="category",
                match=MatchValue(value="technology")
            ),
            FieldCondition(
                key="timestamp",
                range=Range(gte="2026-01-01")
            )
        ]
    ),
    limit=10
)
```

**性能影响**：复杂的多条件过滤可能使延迟增加 3-10 倍，需在过滤精度和性能间权衡。

### 一致性与性能权衡

分布式向量数据库面临 CAP 定理的权衡：

**强一致性场景**（如金融、医疗）：

- 写操作需同步到所有副本
- 延迟增加，但保证数据不丢失
- 推荐：Pinecone（托管）、Milvus（强一致性模式）

**最终一致性场景**（如社交 Feed、推荐系统）：

- 允许异步复制，读取可能略有过期
- 延迟更低，吞吐量更高
- 推荐：Qdrant、Weaviate

**读写分离模式**：

```python
# 写入主节点
client.insert(collection_name="docs", vector=embedding, wait=True)

# 读取可路由到副本，降低延迟
results = client.search(
    collection_name="docs",
    vector=embedding,
    consistency="eventual"  # 允许最终一致性
)
```

### 运维复杂度

各方案的运维要求差异显著：

| 方案 | 运维要点 | 团队要求 |
|------|----------|----------|
| **Pinecone** | 无（完全托管） | 极低 |
| **Qdrant Cloud** | 最小（托管+监控） | 低 |
| **Milvus 集群** | 部署、扩缩容、监控、备份 | 需要专职 SRE |
| **pgvector** | 集成到 PostgreSQL 运维 | 中等 |
| **自托管 Qdrant** | 部署、监控、备份 | 中等 |

**自托管最低配置**（开发/小规模生产）：

```yaml
# docker-compose.yml 最小生产配置
services:
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - qdrant_storage:/qdrant/storage
    environment:
      - QDRANT__SERVICE__GRPC_PORT=6334
    deploy:
      resources:
        limits:
          memory: 4G
        reservations:
          memory: 2G
```

## 选型框架

### 决策矩阵

根据业务需求选择向量数据库的决策框架：

```
                    高一致性需求
                         │
          ┌──────────────┼──────────────┐
          │              │              │
    小规模数据 ───────────────────── 大规模数据
          │              │              │
          │         中等规模            │
          │              │              │
          └──────────────┼──────────────┘
                         │
                    低一致性需求
```

| 维度 | 低要求 | 中等要求 | 高要求 |
|------|--------|----------|--------|
| **数据规模** | < 100 万 | 100 万 - 1 亿 | > 1 亿 |
| **延迟敏感度** | 秒级可接受 | P95 < 100ms | P99 < 20ms |
| **运维能力** | 无专职 DBA | 有 DBA | 有 SRE 团队 |
| **预算** | < $200/月 | $200 - 1000/月 | > $1000/月 |
| **一致性** | 最终一致 | 可调 | 强一致 |

### 推荐路径

**路径 1：快速启动（< 2 周上线）**

```
ChromaDB → pgvector → Pinecone
     ↑          ↑          ↑
  原型验证   中小规模    中等规模
```

**路径 2：企业级（需要长期运营）**

```
pgvector(QA环境) + Qdrant(生产) → Milvus(超大规模)
```

**路径 3：超大规模（> 10 亿向量）**

```
Milvus 集群 + DiskANN 索引
```

### 关键决策问题

在选型时，请逐个回答以下问题：

1. **数据规模是多少？** 决定基本选型范围
2. **延迟要求是多少？** P50/P95/P99 各是多少？
3. **需要元数据过滤吗？** 过滤复杂度如何？
4. **团队运维能力如何？** 有无专职 DBA/SRE？
5. **预算范围是多少？** 托管 vs 自托管成本差异？
6. **有一致性要求吗？** 金融/医疗类数据不容许丢失？

回答这些问题后，结合上文的决策矩阵即可得出适合的方案。

## 相关主题

- [[machine-learning/llm/rag|RAG（检索增强生成）]] — 向量数据库的核心应用场景
- [[machine-learning/llm/llm-evaluation|LLM 评估]] — RAG 系统质量评估
- [[machine-learning/transformer-and-attention|Transformer 与注意力机制]] — LLM 基础架构

---

## 参考资料

[^1]: [Pinecone: Vector Database Guide](https://www.pinecone.io/learn/vector-database/)
[^2]: [Qdrant Documentation](https://qdrant.tech/documentation/)
[^3]: [Milvus Architecture Overview](https://milvus.io/docs/architecture_overview.md)
[^4]: [pgvector: Vector Similarity Search](https://github.com/pgvector/pgvector)
[^5]: [HNSW Algorithm Explained](https://www.pinecone.io/learn/series/ann-hnsw/)
