---
title: AI Agent评估体系
date: 2026-04-16
description: 生产级AI Agent评估方法论 - 多维度评估、流水线构建、回归检测
tags:
  - ai-agent
  - llmops
  - evaluation
  - testing
draft: false
permalink:
---

# AI Agent评估体系

## 1. Agent评估 vs 传统LLM评估

传统LLM评估是**输入-输出对**的评估：

```
输入 → LLM → 输出 → 评分
```

但AI Agent评估必须覆盖**多步骤会话**：

```
用户意图 → Agent决策 → 工具调用 → 状态更新 → 再次决策 → ... → 最终结果
```

### 1.1 复合错误率问题

这是Agent评估最核心的数学问题：

> 一个20步工作流，每一步可靠性95%，整体成功率仅为 **36%**

$$
P(\text{成功}) = 0.95^{20} \approx 0.36
$$

这意味着：
- 每一步的小失误会**复合放大**
- 评估必须覆盖**过程质量**，而非仅结果质量
- 需要识别**错误发生在哪一步**

### 1.2 评估维度

| 维度 | 评估内容 | 方法 |
|------|----------|------|
| Agent级 | 工具选择、推理质量 | 人工标注 + LLM-as-judge |
| 流程级 | 任务完成率、步骤数 | 端到端测试 |
| 系统级 | 延迟、成本、可用性 | 基础设施监控 |

## 2. 评估方法

### 2.1 生产数据驱动评估

**核心思想**：用真实生产数据评估，而非合成测试用例

```python
# 评估流程
1. 捕获真实用户会话trace
2. 人工标注关键决策点
3. 识别失败模式
4. 构建评估用例库
5. 持续迭代
```

**优势**：
- 覆盖真实分布（合成数据无法覆盖长尾）
- 自我更新（用户行为变化，评估也跟着变）
- 节省标注成本

### 2.2 人工标注

```python
# 标注维度
ANNOTATION_DIMENSIONS = {
    "goal_achievement": "任务是否完成",
    "tool_selection": "工具选择是否正确",
    "reasoning_quality": "推理过程是否合理",
    "safety": "是否有安全隐患",
    "efficiency": "步骤是否过于冗长"
}
```

**质量标准**：标注者需经过培训，一致性检验（Kappa > 0.7）

### 2.3 LLM-as-Judge

使用强模型评估弱模型输出：

```python
# 示例prompt
EVAL_PROMPT = """
你是一个AI助手评估员。请评估以下Agent响应的质量。

用户问题：{user_query}
Agent响应：{agent_response}
评估维度：{dimension}

评分标准：
- 5分：完全符合要求
- 4分：基本符合，有小瑕疵
- 3分：部分符合，有明显问题
- 2分：基本不符合
- 1分：完全不符合

请给出评分和简要理由。
"""
```

**注意**：LLM-as-Judge本身有偏差，需结合人工标注验证。

### 2.4 自定义评估规则

```python
# 规则示例
RULES = [
    # 安全规则
    Rule("no_pii_exposure", "不应暴露个人隐私信息"),
    Rule("no_dangerous_content", "不应生成有害内容"),
    
    # 功能规则
    Rule("tool_called_correctly", "工具调用参数正确"),
    Rule("max_steps_not_exceeded", "不超过最大步数限制"),
    Rule("no_infinite_loop", "不陷入重复模式"),
    
    # 质量规则
    Rule("response_coherent", "响应逻辑连贯"),
    Rule("addresses_user_intent", "回应用户意图")
]
```

## 3. 评估流水线

### 3.1 六步评估循环

```
Instrument → Identify → Build Eval Cases → CI Gate → Weekly Annotation → Automate
```

1. **Instrument**：捕获完整trace
2. **Identify**：人工审查50个会话，识别top 3失败类别
3. **Build Eval Cases**：将失败案例转为评估用例（20个足够起步）
4. **CI Gate**：将评估嵌入部署流程，阻塞关键回归
5. **Weekly Annotation**：每周人工标注新数据
6. **Automate**：逐步自动化评估流程

### 3.2 评估用例格式

```python
# 评估用例结构
EVAL_CASE = {
    "id": "case_001",
    "category": "tool_selection_error",
    "input": {
        "user_message": "查询北京市天气",
        "context": {"location": "北京"}
    },
    "expected": {
        "tool": "weather_api",
        "params": {"city": "北京"}
    },
    "ground_truth": {
        "success": True,
        "response": "北京今天晴，温度15-25度"
    }
}
```

### 3.3 回归检测与告警

```python
# 回归检测配置
REGRESSION_CONFIG = {
    "metrics": ["task_completion_rate", "avg_steps", "cost_per_task"],
    "thresholds": {
        "task_completion_rate": {"min": 0.85, "alert": 0.80},
        "cost_per_task": {"max": 0.50, "alert": 0.40}
    },
    "window": "7d",  # 7天滚动窗口
    "comparison": "previous_week"  # 与上周对比
}
```

## 4. 评估平台选择

### 4.1 平台对比

| 平台 | 评估能力 | 成本 | 适用场景 |
|------|----------|------|----------|
| LangSmith | 内置评估套件 | 按trace计费 | LangChain项目 |
| RAGAS | RAG专项评估 | 开源免费 | RAG系统 |
| 自建 | 完全可控 | 运维成本高 | 大型企业 |

### 4.2 选择标准

**关键问题**：这个平台是为Agent设计的，还是从LLM监控改装的？

架构差异导致：
- 专为Agent设计的平台：天然覆盖多步骤、工具调用、状态追踪
- 改装平台：需要额外配置才能捕获这些信息

## 5. 常见失败模式

### 5.1 工具描述不匹配

**问题**：工具描述与实际功能不一致，导致Agent调用错误工具

**解决**：
```python
# 严格的工具描述验证
TOOL_SCHEMA = {
    "name": "weather_api",
    "description": "查询指定城市的天气信息，返回温度、湿度、风速",
    "parameters": {
        "type": "object",
        "properties": {
            "city": {
                "type": "string",
                "description": "城市名称，必须是官方标准名称"
            }
        },
        "required": ["city"]
    }
}
```

### 5.2 循环检测

**问题**：Agent陷入重复模式，浪费Token

**解决**：
```python
def detect_loop(trace, max_identical_actions=3):
    """检测重复动作模式"""
    action_sequence = [span.action for span in trace.spans]
    for i in range(len(action_sequence) - max_identical_actions + 1):
        if len(set(action_sequence[i:i+max_identical_actions])) == 1:
            return True
    return False
```

## 6. 实施建议

### 6.1 起步路径

```
第1周：Instrument - 捕获完整trace
第2周：Identify - 人工审查50个会话
第3周：Build - 构建20个评估用例
第4周：CI Gate - 嵌入部署流程
持续：Weekly Annotation - 每周扩充评估集
```

### 6.2 评估集规模

| 阶段 | 评估集大小 | 说明 |
|------|------------|------|
| 起步 | 20个 | 覆盖主要失败模式 |
| 成熟 | 100个 | 覆盖常见场景 |
| 生产 | 500+ | 持续扩充 |

## 相关内容

- [[machine-learning/llm/ai-agent-observability|AI Agent可观测性]]
- [[machine-learning/llm/ai-agent-production-architecture|生产部署架构]]
- [[machine-learning/llm/llm-evaluation|LLM评估]]

## 参考资料

[^1]: [Complete Guide to Evaluating AI Agents in Production](https://latitude.so/blog/complete-guide-evaluating-ai-agents-production)
[^2]: [AI Agent Evaluation - Comet](https://www.comet.com/site/blog/ai-agent-evaluation/)
[^3]: [What We Learned Deploying AI Agents in Production - Viqus](https://viqus.ai/blog/ai-agents-production-lessons-2026)