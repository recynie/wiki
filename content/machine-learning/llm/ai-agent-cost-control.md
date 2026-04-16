---
title: AI Agent成本控制
date: 2026-04-16
description: 生产环境AI Agent成本控制策略 - Token预算、重试限制、模型分层、缓存优化
tags:
  - ai-agent
  - llmops
  - cost-optimization
  - engineering-economics
draft: false
permalink:
---

# AI Agent成本控制

## 1. 原型成本 vs 生产成本

AI Agent的成本放大效应远超直觉：

| 成本因子 | 原型环境 | 生产环境 |
|----------|----------|----------|
| 重试次数 | 0-1次 | 3-5次（网络抖动、限流） |
| Fan-out | 串行 | 并行调用多个工具 |
| 上下文 | 短prompt | 长历史、检索结果 |
| 步骤数 | 3-5步 | 10-20步复杂任务 |

**结果**：原型$0.02/请求 → 生产可能$0.20/请求（10倍放大）

## 2. 五大成本控制杠杆

### 2.1 Token预算（硬限制）

```python
# 设置硬性Token上限
AGENT_CONFIG = {
    "max_input_tokens": 8000,
    "max_output_tokens": 2000,
    "max_total_tokens": 10000,
    "hard_cutoff": True  # 超限直接返回错误
}

# 监控示例
def check_token_budget(messages):
    total_tokens = count_tokens(messages)
    if total_tokens > AGENT_CONFIG["max_total_tokens"]:
        raise TokenBudgetExceeded(
            f"Token {total_tokens} exceeds budget {AGENT_CONFIG['max_total_tokens']}"
        )
```

**关键原则**：
- 设置**硬截止**（Hard Cutoff），而非软限制
- 软限制只会导致部分响应被截断，用户体验差

### 2.2 重试限制

```python
# 重试配置
RETRY_CONFIG = {
    "max_retries": 3,  # 最多3次，不是无限
    "backoff_factor": 1.5,  # 指数退避
    "retry_on": ["rate_limit", "timeout", "server_error"],
    "do_not_retry": ["invalid_request", "context_overflow"]
}

# 正确做法：有限重试 + 优雅降级
def call_with_retry(agent, user_input):
    for attempt in range(RETRY_CONFIG["max_retries"]):
        try:
            return agent.execute(user_input)
        except RetryableError as e:
            if attempt == RETRY_CONFIG["max_retries"] - 1:
                return fallback_response()  # 降级策略
            wait(exponential_backoff(attempt))
```

**为什么限制重试**：
- 每次重试都是完整的LLM调用（费用全价）
- 无限重试可能在限流时指数级浪费成本

### 2.3 Fan-out限制

```python
# 并行工具调用的成本风险
async def dangerous_fan_out(agent, user_intent):
    # ❌ 无限制并行：成本可能爆炸
    tools = await select_all_possible_tools(user_intent)  # 可能返回10个
    results = await asyncio.gather(*[
        agent.call_tool(tool) for tool in tools
    ])  # 10个并行LLM调用

# ✅ 限制并发
CONCURRENCY_CONFIG = {
    "max_parallel_tools": 3,
    "tool_timeout_seconds": 30
}

async def safe_fan_out(agent, user_intent):
    tools = await select_top_k_tools(user_intent, k=3)  # 只选top 3
    semaphore = asyncio.Semaphore(CONCURRENCY_CONFIG["max_parallel_tools"])
    
    async def bounded_call(tool):
        async with semaphore:
            return await asyncio.wait_for(
                agent.call_tool(tool),
                timeout=CONCURRENCY_CONFIG["tool_timeout_seconds"]
            )
    
    results = await asyncio.gather(*[bounded_call(t) for t in tools])
```

### 2.4 模型分层

```
模型分层策略：

Router Model (便宜快)     → 意图分类、简单问答
    ↓ 不满足条件
Mid-tier Model (中等)     → 常规生成、工具选择
    ↓ 更复杂需求
Frontier Model (贵但强)   → 复杂推理、创意任务
```

```python
MODEL_TIER_CONFIG = {
    "routing": {
        "intent_classification": {"model": "gpt-3.5-turbo", "cost_factor": 1},
        "simple_qa": {"model": "gpt-3.5-turbo", "cost_factor": 1},
        "tool_selection": {"model": "gpt-4o-mini", "cost_factor": 5},
        "complex_reasoning": {"model": "gpt-4-turbo", "cost_factor": 20},
    },
    "default": "gpt-4o-mini"  # 默认用便宜的
}

def route_to_model(task_type):
    return MODEL_TIER_CONFIG["routing"].get(
        task_type, 
        MODEL_TIER_CONFIG["default"]
    )
```

### 2.5 任务级计费

```python
# ❌ 传统方式：按API调用计费
# 问题：5次重试的成功请求计费5次

# ✅ 正确方式：按任务成功计费
class TaskCostTracker:
    def __init__(self):
        self.costs = defaultdict(float)
    
    def record_task(self, task_id, calls, successful):
        # 只有成功的任务计费
        if successful:
            self.costs[task_id] = sum(call.cost for call in calls)
    
    def cost_per_successful_task(self):
        successful_tasks = [t for t in self.tasks.values() if t.successful]
        return sum(t.cost for t in successful_tasks) / len(successful_tasks)
```

## 3. 成本监控

### 3.1 关键成本指标

```python
COST_METRICS = {
    "cost_per_request": "每请求平均成本",
    "cost_per_successful_task": "每成功任务成本（推荐）",
    "token_efficiency": "有效Token / 总Token（缓存优化依据）",
    "retry_rate": "重试率（成本浪费指标）",
    "tool_call_cost": "工具调用成本分布"
}
```

### 3.2 成本异常检测

```python
# 异常检测配置
COST_ALERT_CONFIG = {
    "cost_per_task_threshold": 0.50,  # 单任务成本上限
    "daily_budget": 1000,  # 日预算
    "anomaly_multiplier": 3,  # 3倍均值触发告警
    
    "alerts": [
        {"metric": "cost_per_task", "threshold": 0.50, "severity": "high"},
        {"metric": "retry_rate", "threshold": 0.20, "severity": "medium"},
        {"metric": "tool_call_count", "threshold": 20, "severity": "high"}
    ]
}
```

## 4. 缓存策略

### 4.1 确定性与非确定性缓存

```python
# 确定性缓存：相同输入 → 相同输出
@cache(ttl=3600, key_func=lambda ctx: hash(ctx["user_query"]))
def cached_classification(user_query):
    return llm.classify(user_query)

# 非确定性缓存：需要验证结果
async def cached_generation(user_query, temp=0.7):
    cache_key = hash((user_query, temp))
    cached = await redis.get(cache_key)
    if cached:
        # 验证缓存结果质量
        if await validate_cached_result(cached):
            return cached
    return await llm.generate(user_query, temperature=temp)
```

### 4.2 缓存键设计

```python
# 好的缓存键设计
CACHE_KEY_RULES = {
    "normalize_input": True,  # "北京" vs "北京市" → 归一化
    "strip_whitespace": True,
    "lowercase": False,  # 区分大小写时不要
    "include_model_version": True,  # 模型更新时失效旧缓存
    "include_temperature": True
}

def make_cache_key(prompt, model, temperature):
    normalized = normalize(prompt)
    return f"{model}:{temperature}:{hash(normalized)}"
```

### 4.3 缓存命中优化

```python
# 提升缓存命中率的方法
CACHE_OPTIMIZATION = {
    "prompt_normalization": [
        "移除无关空格",
        "统一日期格式",
        "标准化城市/公司名称"
    ],
    "semantic_cache": [
        "embedding相似度匹配",
        "阈值0.95以上命中"
    ],
    "分层缓存": {
        "L1": "内存缓存（极快）",
        "L2": "Redis（分布式）",
        "L3": "数据库（持久化）"
    }
}
```

## 5. 成本优化实战清单

```
AI Agent成本控制检查清单：

[ ] Token硬限制配置
[ ] 重试次数上限（2-3次）
[ ] Fan-out并发限制
[ ] 模型分层路由
[ ] 任务级成本追踪
[ ] 成本异常告警
[ ] 缓存策略实施
[ ] 成本Dashboard

成本优化优先级：
1. 高：Token预算 + 重试限制（立竿见影）
2. 中：模型分层（需评估质量影响）
3. 低：缓存（效果因场景而异）
```

## 相关内容

- [[machine-learning/llm/ai-agent-observability|AI Agent可观测性]]
- [[machine-learning/llm/ai-agent-evaluation|AI Agent评估体系]]
- [[machine-learning/llm/ai-agent-production-architecture|生产部署架构]]

## 参考资料

[^1]: [The Production AI Agent Playbook - Kevin Tan](https://blog.jztan.com/production-ai-agent-playbook/)
[^2]: [What We Learned Deploying AI Agents in Production - Viqus](https://viqus.ai/blog/ai-agents-production-lessons-2026)