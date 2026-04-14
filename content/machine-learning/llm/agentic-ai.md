---
title: Agentic AI（代理式人工智能）
date: 2026-04-14
description: Agentic AI 架构、工具调用、多智能体系统与生产实践
tags:
  - agentic-ai
  - llm
  - ai-engineering
  - autonomous-agents
draft: false
permalink:
---

# Agentic AI（代理式人工智能）

## 概述

Agentic AI（代理式人工智能）是指能够自主追求目标、执行多步骤工作流的 AI 系统。与简单的问答式 Chatbot 不同，Agentic AI 能够：

- **感知环境**：通过工具获取外部信息
- **推理决策**：基于上下文制定行动计划
- **执行动作**：调用工具完成具体任务
- **观察反馈**：根据执行结果调整下一步行动

> 2026 年，64% 的 CIO 计划在 24 个月内部署代理式 AI（来源：Gartner 2026 CIO Agenda）

## 核心概念

### Agentic AI vs 传统 Chatbot

| 特性 | 传统 Chatbot | Agentic AI |
|------|-------------|-----------|
| 交互模式 | 单轮问答 | 多轮推理执行 |
| 工具使用 | 无 | 主动调用工具 |
| 自主程度 | 低 | 高（目标导向） |
| 适用场景 | FAQ、客服 | 复杂任务自动化 |

### Agent Loop（智能体循环）

```
┌─────────────────────────────────────────────────────┐
│                      Agent Loop                      │
├─────────────────────────────────────────────────────┤
│                                                      │
│   ┌─────────┐    ┌──────────┐    ┌─────────┐       │
│   │Perceive │───▶│  Reason  │───▶│  Act    │       │
│   │  (感知) │    │  (推理)  │    │ (执行)  │       │
│   └─────────┘    └──────────┘    └─────────┘       │
│        ▲              │              │               │
│        │              │              ▼               │
│        │         ┌──────────┐  ┌─────────┐          │
│        └─────────│Observe   │◀─│Result   │          │
│                  │(观察反馈)│  │(结果)   │          │
│                  └──────────┘  └─────────┘          │
└─────────────────────────────────────────────────────┘
```

**循环终止条件**：
- 达成目标
- 达到最大迭代次数
- 检测到不可恢复的错误

## 架构模式

### 1. ReAct（Reasoning + Acting）

将推理和执行交织进行，适合需要灵活调整策略的任务：

```python
def react_agent(query: str, tools: list[Tool]) -> str:
    thought = "我需要分析这个问题..."
    action = select_tool(thought, tools)
    observation = execute(action)
    
    while not finished:
        thought = reason(observation, query)
        if "answer" in thought:
            return extract_answer(thought)
        action = select_tool(thought, tools)
        observation = execute(action)
```

### 2. Plan-and-Execute

先生成完整计划，再按顺序执行，适合可预测的任务：

```python
def plan_and_execute(query: str, tools: list[Tool]) -> str:
    # 1. 规划阶段
    plan = planner.create_plan(query)  # ["步骤1", "步骤2", ...]
    
    # 2. 执行阶段
    for step in plan:
        result = execute_step(step, tools)
        if not validate(result):
            plan = replan(query, result)  # 重新规划
            break
    
    return aggregate_results(results)
```

### 3. Supervisor（多智能体协作）

一个协调者智能体将任务分配给专业子智能体：

```
┌──────────────────────────────────────────────┐
│              Supervisor Agent                │
│  ┌────────────────────────────────────────┐ │
│  │  任务：分析代码库并生成文档             │ │
│  └────────────────────────────────────────┘ │
│                    │                         │
│         ┌──────────┼──────────┐             │
│         ▼          ▼          ▼             │
│   ┌───────────┐ ┌───────────┐ ┌───────────┐ │
│   │  Code     │ │  API      │ │  Doc      │ │
│   │  Reader   │ │  Analyzer │ │  Writer   │ │
│   │  Agent    │ │  Agent    │ │  Agent    │ │
│   └───────────┘ └───────────┘ └───────────┘ │
└──────────────────────────────────────────────┘
```

## 工具调用

### 工具定义架构

```python
from typing import Annotated, Any
from pydantic import BaseModel, Field

class SearchTool(BaseModel):
    """网络搜索工具"""
    query: str = Field(description="搜索查询词")
    max_results: int = Field(default=5, description="最大结果数")

class CalculatorTool(BaseModel):
    """计算器工具"""
    expression: str = Field(description="数学表达式")

# 工具 Schema 用于 LLM 函数调用
tools = [
    {
        "type": "function",
        "function": {
            "name": "search",
            "description": "搜索互联网获取最新信息",
            "parameters": SearchTool.model_json_schema()
        }
    },
    {
        "type": "function", 
        "function": {
            "name": "calculate",
            "description": "执行数学计算",
            "parameters": CalculatorTool.model_json_schema()
        }
    }
]
```

### 工具调用执行器

```python
import json
from typing import Callable

class ToolExecutor:
    def __init__(self):
        self.tools: dict[str, Callable] = {}
    
    def register(self, name: str, func: Callable):
        self.tools[name] = func
    
    async def execute(self, tool_call: dict) -> Any:
        name = tool_call["function"]["name"]
        args = json.loads(tool_call["function"]["arguments"])
        
        if name not in self.tools:
            return {"error": f"Unknown tool: {name}"}
        
        try:
            result = await self.tools[name](**args)
            return result
        except Exception as e:
            return {"error": str(e)}
```

### 顺序执行 vs 并行执行

```python
# 顺序执行：结果影响下一步决策
async def sequential_execution(steps: list[ToolCall]) -> list[Result]:
    results = []
    for step in steps:
        result = await executor.execute(step)
        results.append(result)
        if is_critical_error(result):
            break  # 提前终止
    return results

# 并行执行：独立任务加速完成
async def parallel_execution(steps: list[ToolCall]) -> list[Result]:
    tasks = [executor.execute(step) for step in steps]
    return await asyncio.gather(*tasks)
```

## 记忆系统

### 短期记忆（Conversation Memory）

```python
class ConversationMemory:
    def __init__(self, max_turns: int = 10):
        self.messages: list[Message] = []
        self.max_turns = max_turns
    
    def add(self, role: str, content: str):
        self.messages.append(Message(role=role, content=content))
        # 滑动窗口，保持最近 N 轮对话
        if len(self.messages) > self.max_turns:
            self.messages = self.messages[-self.max_turns:]
    
    def get_context(self) -> list[dict]:
        return [{"role": m.role, "content": m.content} 
                for m in self.messages]
```

### 长期记忆（Semantic Memory）

```python
class SemanticMemory:
    def __init__(self, vector_db: VectorDB):
        self.vector_db = vector_db
        self.collection = "long_term_memory"
    
    async def store(self, experience: str, metadata: dict):
        embedding = await embed_model.encode(experience)
        self.vector_db.upsert(
            collection=self.collection,
            points=[{
                "id": str(uuid.uuid4()),
                "vector": embedding,
                "payload": {"content": experience, **metadata}
            }]
        )
    
    async def recall(self, query: str, top_k: int = 5) -> list[dict]:
        query_embedding = await embed_model.encode(query)
        results = self.vector_db.search(
            collection=self.collection,
            vector=query_embedding,
            limit=top_k
        )
        return [r["payload"] for r in results]
```

## 多智能体系统

### 智能体协作模式

```python
class MultiAgentOrchestrator:
    def __init__(self):
        self.agents: dict[str, Agent] = {}
        self.registered_tools: dict[str, Tool] = {}
    
    def register(self, name: str, agent: Agent, tools: list[Tool]):
        self.agents[name] = agent
        for tool in tools:
            self.registered_tools[f"{name}.{tool.name}"] = tool
    
    async def run_task(self, task: str, agent_roles: list[str]) -> str:
        # 1. 任务分解
        subtasks = await self.decompose(task, agent_roles)
        
        # 2. 并行执行独立子任务
        coroutines = [self.agents[role].execute(subtask) 
                      for role, subtask in subtasks.items()]
        results = await asyncio.gather(*coroutines)
        
        # 3. 结果聚合
        return self.aggregate(results)
```

### 通信协议

```python
from enum import Enum

class MessageType(Enum):
    TASK = "task"           # 分配任务
    RESULT = "result"       # 返回结果
    ERROR = "error"         # 错误报告
    QUERY = "query"         # 查询信息
    RESPONSE = "response"   # 响应信息

@dataclass
class AgentMessage:
    from_agent: str
    to_agent: str
    type: MessageType
    content: Any
    conversation_id: str
    timestamp: datetime
```

## 生产实践

### 评估框架

```python
from dataclasses import dataclass

@dataclass
class AgentEvaluation:
    task_success: float      # 任务成功率
    avg_steps: float         # 平均步数
    tool_usage_accuracy: float  # 工具使用准确率
    response_time_ms: float  # 响应时间
    cost_usd: float          # 成本

# 评估指标
def evaluate_agent(agent: Agent, tasks: list[Task]) -> AgentEvaluation:
    successes = 0
    total_steps = 0
    correct_tools = 0
    total_tools = 0
    
    for task in tasks:
        trace = agent.run(task)
        successes += trace.success
        total_steps += len(trace.steps)
        
        for step in trace.steps:
            total_tools += 1
            if step.tool == step.expected_tool:
                correct_tools += 1
    
    return AgentEvaluation(
        task_success=successes / len(tasks),
        avg_steps=total_steps / len(tasks),
        tool_usage_accuracy=correct_tools / total_tools,
        response_time_ms=trace.total_time_ms,
        cost_usd=trace.total_cost
    )
```

### 安全护栏（Guardrails）

```python
class SafetyGuardrails:
    def __init__(self):
        self.allowed_tools: set[str] = set()
        self.deny_patterns: list[re.Pattern] = []
        self.max_iterations: int = 20
    
    def check_tool(self, tool_name: str) -> bool:
        if tool_name not in self.allowed_tools:
            return False
        return True
    
    def check_content(self, content: str) -> bool:
        for pattern in self.deny_patterns:
            if pattern.search(content):
                return False
        return True
    
    def check_iterations(self, count: int) -> bool:
        return count < self.max_iterations

# 使用示例
guardrails = SafetyGuardrails()
guardrails.allowed_tools = {"search", "calculate", "read_file", "write_file"}
guardrails.deny_patterns = [re.compile(r"rm\s+-rf\s+/"), re.compile(r"DROP\s+TABLE")]
```

### 成本管理

```python
class CostTracker:
    def __init__(self, budget_usd: float):
        self.budget = budget_usd
        self.spent = 0.0
        self.requests = 0
    
    def estimate_cost(self, model: str, input_tokens: int, 
                     output_tokens: int) -> float:
        pricing = {
            "gpt-4o": (0.005, 0.015),   # ($/1K input, $/1K output)
            "claude-3-5-sonnet": (0.003, 0.015),
            "gpt-4o-mini": (0.00015, 0.0006),
        }
        if model not in pricing:
            return 0.0
        input_cost = input_tokens / 1000 * pricing[model][0]
        output_cost = output_tokens / 1000 * pricing[model][1]
        return input_cost + output_cost
    
    def check_budget(self, estimated_cost: float) -> bool:
        if self.spent + estimated_cost > self.budget:
            raise BudgetExceededError(f"Budget exceeded: ${self.spent:.2f}")
        return True
```

## 最佳实践

### 1. 工具设计原则

| 原则 | 说明 | 示例 |
|------|------|------|
| 单一职责 | 每个工具做一件事 | `search_web` 而非 `search_and_summarize` |
| 清晰命名 | 动宾短语 | `get_user_orders` 而非 `user_orders` |
| 完整描述 | 说明输入输出和副作用 | 文档字符串包含示例 |
| 幂等性 | 多次调用结果一致 | `get_current_time` vs `send_email` |

### 2. 错误处理策略

```python
async def robust_execute(tool_call: ToolCall, 
                        max_retries: int = 3) -> Result:
    for attempt in range(max_retries):
        try:
            return await executor.execute(tool_call)
        except ToolUnavailableError:
            # 等待后重试（指数退避）
            await asyncio.sleep(2 ** attempt)
        except RateLimitError:
            # 切换到备用工具
            tool_call = fallback_tool(tool_call)
        except Exception as e:
            if attempt == max_retries - 1:
                return Result(success=False, error=str(e))
    
    return Result(success=False, error="Max retries exceeded")
```

### 3. 调试与可观测性

```python
class AgentTracer:
    def __init__(self, service_name: str):
        self.logger = structured_logger(service_name)
    
    def trace_step(self, step: AgentStep):
        self.logger.info(
            "agent_step",
            step=step.number,
            tool=step.tool_used,
            input_tokens=step.input_tokens,
            output_tokens=step.output_tokens,
            latency_ms=step.latency_ms,
            reasoning=step.thought[:200]  # 截断推理过程
        )
    
    def trace_error(self, error: Exception, context: dict):
        self.logger.error(
            "agent_error",
            error_type=type(error).__name__,
            error_message=str(error),
            **context
        )
```

## 相关主题

- [[machine-learning/llm/rag|RAG 检索增强生成]] — Agentic AI 的重要组成部分
- [[machine-learning/llm/vector-database|向量数据库]] — 记忆系统的存储基础设施
- [[machine-learning/llm/llm-evaluation|LLM 评估]] — Agent 系统质量评估
- [[distributed-systems/distributed-tracing|分布式追踪]] — Agent 执行的可观测性

---

## 参考资料

[^1]: [Agentic Engineering Roadmap 2026 - Towards Agentic AI](https://towardsagenticai.com/agentic-engineering-roadmap-skills-tools-resources-2026/)
[^2]: [The Future of Software Engineering with AI - Pragmatic Engineer](https://newsletter.pragmaticengineer.com/p/the-future-of-software-engineering-with-ai)
[^3]: [AI Development Trends in 2026 - Cozcore](https://www.cozcore.com/blog/ai-development-trends-2026)
[^4]: [Harness Engineering: Codex in an Agent-First World - OpenAI](https://openai.com/index/harness-engineering)
