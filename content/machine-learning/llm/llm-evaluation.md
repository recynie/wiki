---
title: LLM评估
date: 2026-04-14
description: 大语言模型（LLM）评估的核心维度、主流框架与生产级实践指南
tags:
  - llm
  - evaluation
  - rag
  - ai-engineering
  - metrics
draft: false
permalink:
---

# LLM评估

## 概述

大型语言模型（LLM）评估是 AI 工程中最具挑战性的课题之一。与传统机器学习模型不同，LLM 的输出具有开放性和多样性，难以用简单的准确率指标衡量。在 2026 年的生产环境中，LLM 评估已从简单的 BLEU/ROUGE 分数演变为涵盖 faithfulness、answer relevancy、幻觉检测等多个维度的综合评估体系。

评估的核心挑战在于：**什么才算是一个"好"的回答？** 这个问题没有标准答案，因为答案的质量取决于用户意图、上下文相关性、事实准确性等多个因素。

本文聚焦于 RAG 系统和 LLM 应用评估，涵盖评估维度、主流框架、基准数据集以及生产级实践。

## 评估维度

### Faithfulness（忠实度）

Faithfulness 衡量答案是否基于检索到的上下文，而非模型的内部知识。计算方式如下：

$$
\text{Faithfulness} = \frac{\text{答案中可归因于上下文的陈述数}}{\text{答案中总陈述数}}
$$

**低忠实度的典型表现**：
- 答案包含上下文未提及的细节
- 模型用自己的话重述，但引入错误
- 过度推理，从少量事实推断出不存在的关系

**提升方法**：
- 约束 Prompt：「基于以下上下文回答，不要添加外部知识」
- 使用引用标注，明确标注答案来源
- 引入事实核查（Fact Checking）环节

### Answer Relevancy（答案相关性）

答案相关性衡量回答是否真正回应了用户问题，而非表面相关。评估指标：

$$
\text{AnswerRelevancy} = \text{语义相似度}(Q, A_{\text{rephrased}})
$$

其中 $A_{\text{rephrased}}$ 是根据答案反向生成的问题，与原始问题越相似越好。

**低相关性的典型表现**：
- 答非所问：问题问 A，答案讲 B
- 信息冗余：包含大量无关细节
- 不完整：只回答了问题的部分子集

### Context Precision（上下文精确度）

Context Precision 衡量检索到的 Top-K 文档中，与问题真正相关的文档所占的比例和排名质量：

$$
\text{Context Precision} = \frac{\sum_{i=1}^{k} \left( \text{Rel}_i \times \frac{1}{i} \right)}{\text{相关文档总数}}
$$

其中 $\text{Rel}_i$ 表示第 $i$ 个文档是否相关（1 或 0）。

**优化方向**：
- 改进 Embedding 模型，使用领域适配版本
- 采用混合检索（向量 + BM25）
- 引入重排序（Reranking）模块

### Context Recall（上下文召回率）

Context Recall 衡量检索系统是否找到了所有相关文档：

$$
\text{Context Recall} = \frac{\text{检索到的相关文档数}}{\text{所有相关文档数}}
$$

**挑战**：召回率的上限难以确定，因为很难枚举"所有相关文档"。

### Hallucination Detection（幻觉检测）

幻觉是 LLM 最严重的问题之一，指模型生成看似合理但实际错误的内容。检测方法分为两类：

#### 基于规则的方法

```python
def detect_hallucination_fuzzy(answer, context, threshold=0.7):
    """基于语义相似度的幻觉检测"""
    claim_embeddings = extract_claims(answer)
    context_embedding = embed(context)
    
    for claim in claim_embeddings:
        similarity = cosine_similarity(claim, context_embedding)
        if similarity < threshold:
            yield HallucinationFlag(claim, similarity)
```

#### 基于模型的方法

使用专门训练的幻觉检测模型（如 SELF-RAG 的归因模型）来判断每个陈述是否可归因于上下文。

**幻觉类型分类**：

| 类型 | 描述 | 例子 |
|------|------|------|
| 事实性幻觉 | 与真实世界事实不符 | "2024年巴黎奥运会金牌榜第一是中国" |
| 上下文幻觉 | 与给定上下文不符 | 上下文未提及某公司，答案却提到 |
| 语义幻觉 | 陈述逻辑上不连贯 | "A 导致 B，且 B 导致 A" |

## 评估框架

### RAGAS

RAGAS（RAG Assessment）是专为 RAG 系统设计的评估框架，核心指标包括：

- **Faithfulness**：答案对上下文的忠实程度
- **Answer Relevancy**：答案与问题的相关程度
- **Context Precision**：检索结果的相关文档排名质量
- **Context Recall**：检索系统的召回能力

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall
)

# 评估数据集格式
eval_dataset = [
    {
        "user_input": "什么是 RAG？",
        "retrieved_contexts": [context1, context2],
        "response": "RAG 是检索增强生成...",
        "reference": "RAG = Retrieval Augmented Generation"
    }
]

result = evaluate(
    dataset=eval_dataset,
    metrics=[
        faithfulness,
        answer_relevancy,
        context_precision,
        context_recall
    ]
)

print(result)
# {
#   'faithfulness': 0.92,
#   'answer_relevancy': 0.88,
#   'context_precision': 0.85,
#   'context_recall': 0.78
# }
```

**生产阈值建议**：

| 指标 | 目标值 | 说明 |
|------|--------|------|
| Faithfulness | > 0.85 | 答案必须基于上下文 |
| Answer Relevancy | > 0.80 | 答案需真正回答问题 |
| Context Precision | > 0.75 | 检索结果需精准 |

### TruLens

TruLens 专注于深度学习模型的可观测性和评估，提供更细粒度的反馈：

```python
from trulens_eval import TruCustomApp
from trulens_eval.feedback import Feedback, Groundedness

# 定义评估反馈函数
groundedness = Feedback(Groundedness().groundedness_score).on(
    context=Trace.context,
    response=Trace.response
)

# 评估 LLM 应用
tru = TruCustomApp(my_rag_app, feedbacks=[groundedness])
result = tru.run(user_input="如何优化 RAG 系统？")
```

**TruLens 特色**：

- **多维度反馈**：可自定义任意评估维度
- **溯源追踪**：精确到每个 Prompt 的评估结果
- **可视化分析**：识别低分样本的模式

### G-Eval

G-Eval 使用 GPT-4 作为评估器，通过思维链（Chain of Thought）生成评估维度的评分：

```python
from g_eval import G.Eval

evaluator = G.Eval(
    model="gpt-4-turbo",
    dimensions=["relevance", "coherence", "factual_accuracy"]
)

# 评估单个回答
score = evaluator.evaluate(
    prompt="解释量子计算的基本原理",
    response="量子计算是一种使用量子比特...",
    reference="量子计算使用量子叠加态..."
)

print(f"G-Eval 分数: {score}")
# {'relevance': 4.2/5, 'coherence': 4.5/5, 'factual_accuracy': 3.8/5}
```

**优点**：

- 高评估质量，GPT-4 的判断接近人类
- 可评估创意性、连贯性等软维度
- 支持自定义评估维度

**缺点**：

- 成本高，每次评估需要消耗 API 调用
- 延迟高，不适合实时评估
- 存在位置偏差（Position Bias）

## 基准数据集

### MTEB

MTEB（Massive Text Embedding Benchmark）是目前最全面的 Embedding 模型评估基准，涵盖 58 个数据集、12 个任务类别：

| 任务类型 | 数据集 | 代表性指标 |
|----------|--------|----------|
| Classification | Amazon-Polarity, MR | Accuracy |
| Clustering | Arxiv, Biorxiv | V-Measure |
| Pair Classification | Quora-Pair, SMS | AP |
| Reranking | MSMARCO, NFCorpus | MRR@10 |
| Retrieval | BEIR, Mmarco | NDCG@10 |
| STS | STS-B, SICK-R | Spearman |

**评估方法**：

```python
from mteb import get_tasks, get_model_results

tasks = get_tasks(languages=["zh", "en"])
model_results = get_model_results(model, tasks)
print(model_results["zh"]["average"])
```

### BEIR

BEIR（Benchmark for Information Retrieval）专注于检索系统评估，是 RAG 检索层评估的标准基准：

```python
from beir.datasets import GenericDataLoader
from beir.retrieval import QPP

# 加载数据集
dataset = GenericDataLoader.load("msmarco")
evaluator = QPP(dataset)

# 评估检索模型
results = evaluator.evaluate(retriever)
```

### 其他基准数据集

| 数据集 | 用途 | 规模 |
|--------|------|------|
| HaluEval | 幻觉检测 | 35K 样本 |
| TruthfulQA | 事实准确性 | 817 问答对 |
| BIG-Bench | 综合能力 | 200+ 任务 |
| HELM | 语言模型全貌 | 42 场景 |

## 生产级实践

### 自动化评估流水线

生产环境中的评估流水线需要持续运行，检测模型变更或数据漂移：

```python
class EvaluationPipeline:
    def __init__(self, rag_app, eval_dataset, schedule="0 */6 * * *"):
        self.rag_app = rag_app
        self.eval_dataset = eval_dataset
        self.schedule = schedule  # 每6小时运行一次
        self.metrics = [faithfulness, answer_relevancy]
    
    def run_batch_evaluation(self):
        """批量评估"""
        results = []
        for sample in tqdm(self.eval_dataset):
            response = self.rag_app.query(sample["user_input"])
            result = evaluate(
                [{"user_input": sample["user_input"],
                  "retrieved_contexts": response.contexts,
                  "response": response.text}],
                self.metrics
            )
            results.append(result)
        
        return self.aggregate_results(results)
    
    def aggregate_results(self, results):
        """聚合结果"""
        return {
            "faithfulness_mean": np.mean([r["faithfulness"] for r in results]),
            "faithfulness_p95": np.percentile([r["faithfulness"] for r in results], 95),
            "answer_relevancy_mean": np.mean([r["answer_relevancy"] for r in results]),
            "drift_detected": self.detect_drift(results)
        }
```

### 人类评估集成

自动化指标无法完全替代人类判断。以下场景需要人工评估：

- 创意性评估（诗歌、故事生成）
- 敏感内容检测（政治、宗教）
- 复杂推理验证（数学证明、逻辑推导）

```python
def human_eval_sample(responses, sample_size=50, random_seed=42):
    """随机采样进行人类评估"""
    samples = random.sample(responses, sample_size)
    
    for idx, response in enumerate(samples):
        rating = input(f"样本 {idx}: {response.text[:100]}...\n评分(1-5): ")
        save_to_db({"response_id": response.id, "rating": int(rating)})
    
    return calculate_interannotator_agreement()
```

### A/B 测试框架

在生产环境中部署 LLM 应用时，A/B 测试是评估模型效果的金标准：

```python
class LLMABTest:
    def __init__(self, experiment_id, traffic_split=0.5):
        self.experiment_id = experiment_id
        self.traffic_split = traffic_split
        self.variants = {}
    
    def assign_variant(self, user_id: str) -> str:
        """基于用户 ID 分配实验组"""
        hash_value = hash(user_id) % 100
        if hash_value < self.traffic_split * 100:
            return "control"  # 当前版本
        return "treatment"  # 新版本
    
    def track_conversion(self, user_id: str, metric: str, value: float):
        """追踪转化指标"""
        variant = self.assign_variant(user_id)
        log_event({
            "experiment_id": self.experiment_id,
            "variant": variant,
            "metric": metric,
            "value": value,
            "timestamp": datetime.now()
        })
```

**关键指标**：

| 指标类型 | 指标名 | 说明 |
|----------|--------|------|
| 参与度 | 用户停留时长、对话轮数 | 用户是否觉得有用 |
| 转化率 | 任务完成率、咨询解决率 | 是否帮助用户达成目标 |
| 负面指标 | 投诉率、退出率 | 是否有问题 |

## 成本与质量权衡

### 评估方法对比

| 方法 | 成本 | 质量 | 适用场景 |
|------|------|------|----------|
| 规则匹配 | $0.001/query | 中等 | 基础指标检查 |
| RAGAS | $0.01/query | 高 | 生产监控 |
| G-Eval | $0.10/query | 很高 | 重要版本发布 |
| 人类评估 | $5-20/样本 | 最高 | 关键决策 |

### 选择决策树

```
评估场景
    │
    ├── 日常监控（每天运行）
    │   └── 选择规则匹配 + RAGAS（低成本）
    │
    ├── 模型迭代评估
    │   ├── 小版本改进 → RAGAS
    │   └── 大版本发布 → G-Eval + 人类抽检
    │
    └── 关键业务决策
        └── 人类评估 + A/B 测试
```

### 成本优化策略

1. **采样评估**：不必评估全部数据，抽样 5-10% 可获得统计显著性
2. **分层评估**：高频场景用规则，低频场景用 LLM 评估
3. **缓存评估结果**：相同 Prompt 的评估结果可复用
4. **离线批量评估**：避免实时 API 调用，降低成本

## 总结

LLM 评估是一个多维度、系统化的工程问题。核心要点：

1. **评估维度**：根据业务场景选择合适的指标组合，Faithfulness 和 Answer Relevancy 是 RAG 系统的基础
2. **框架选择**：RAGAS 适合日常监控，TruLens 适合深度调试，G-Eval 适合重要决策
3. **基准数据集**：MTEB 和 BEIR 是评估 Embedding 和检索系统的事实标准
4. **生产实践**：自动化流水线 + 人工抽检 + A/B 测试的组合是最优解
5. **成本控制**：分层评估、采样评估、结果缓存是成本优化的关键

随着 LLM 技术的发展，评估方法和框架也在持续演进。建议团队建立评估文化，将评估作为开发流程的核心环节，而非事后的补救措施。

## 参考资料

[^1]: [RAGAS: Evaluation Framework for RAG](https://github.com/explodinggradients/ragas)
[^2]: [TruLens: Evaluable Deep Learning](https://github.com/truefoundry/trulens)
[^3]: [G-Eval: GPT-4 Evaluation Framework](https://github.com/G-Eval/G-Eval)
[^4]: [MTEB: Massive Text Embedding Benchmark](https://huggingface.co/datasets/llm-collection/mteb)
[^5]: [BEIR: Benchmark for Information Retrieval](https://github.com/beir-cellar/beir)
