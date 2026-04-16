---
title: AI对齐技术详解
date: 2026-04-16
description: RLHF、Constitutional AI、DPO等AI对齐技术的深入解析与实践
tags:
  - alignment
  - rlhf
  - constitutional-ai
  - dpo
  - llm
  - training
draft: false
permalink:
---

# AI对齐技术详解

## 概述

AI对齐（Alignment）技术的核心目标是确保AI系统追求人类真正意图，而非被字面指令误导或找到奖励函数的漏洞。随着LLM规模的增大，对齐技术从学术研究走向工业实践，成为2026年企业AI部署的必备要素。

## 对齐技术谱系

```
对齐技术演进：

RLHF (2020-2022)
    │
    ├── RLAIF (2022-2023) ─── 用AI反馈替代人类反馈
    │
    └── Constitutional AI (2022-) ─── 基于原则的自我批判
            │
            └── DPO (2023-) ─── 直接偏好优化，绕过强化学习
```

## RLHF：人类反馈强化学习

### 核心思想

RLHF（Reinforcement Learning from Human Feedback）通过人类反馈信号训练奖励模型，再用强化学习优化LLM行为。

### 三阶段流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    RLHF 三阶段流程                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  阶段1: SFT (Supervised Fine-Tuning)                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  人类示范 ──▶ 预测下一个token ──▶ 行为模仿               │   │
│  │  (Demonstration)      (Next Token Prediction)           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                            │                                    │
│                            ▼                                    │
│  阶段2: 奖励模型训练                                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  比较数据 ──▶ 奖励模型 ──▶ 预测人类偏好                  │   │
│  │  (Comparison)     (Reward Model)    (rθ(x,y))          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                            │                                    │
│                            ▼                                    │
│  阶段3: RL优化 (PPO)                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  策略梯度 ──▶ KL散度约束 ──▶ 平衡效果与偏离              │   │
│  │  (Policy Gradient)  (KL Penalty)                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 数学推导

**阶段1：SFT**

给定输入 $x$ 和目标输出 $y$，SFT通过最大化条件似然进行学习：

$$
\mathcal{L}_{\text{SFT}} = -\sum_{t=1}^{T} \log P_\theta(y_t | x, y_{<t})
$$

**阶段2：奖励模型**

对于人类偏好对 $(y^+, y^-)$，奖励模型学习区分：

$$
\mathcal{L}_r = -\mathbb{E}_{(x, y^+, y^-) \sim \mathcal{D}} \log \sigma(r_\phi(x, y^+) - r_\phi(x, y^-))
$$

其中 $\sigma$ 是sigmoid函数，$r_\phi$ 是奖励模型。

**阶段3：PPO优化**

使用近端策略优化（PPO）最大化期望奖励，同时约束策略偏离参考模型：

$$
\mathcal{L}_{\text{PPO}} = -\mathbb{E}_{x \sim \mathcal{D}, y \sim \pi_\theta} \left[ r_\phi(x, y) - \beta \cdot \text{KL}(\pi_\theta || \pi_{\text{ref}}) \right]
$$

其中 $\beta$ 控制KL惩罚强度，$\pi_{\text{ref}}$ 是SFT阶段训练的参考模型。

### 代码实现

```python
import torch
import torch.nn as nn
from typing import List, Tuple, Dict

class RewardModel(nn.Module):
    """奖励模型：输入 prompt + response，输出标量奖励"""
    def __init__(self, base_model):
        super().__init__()
        self.base_model = base_model
        # 奖励预测头
        self.reward_head = nn.Linear(base_model.config.hidden_size, 1)
    
    def forward(
        self, 
        input_ids: torch.Tensor, 
        attention_mask: torch.Tensor
    ) -> torch.Tensor:
        outputs = self.base_model(
            input_ids=input_ids,
            attention_mask=attention_mask
        )
        # 使用最后一个token的隐藏状态预测奖励
        last_hidden = outputs.last_hidden_state[:, -1, :]
        reward = self.reward_head(last_hidden)
        return reward.squeeze(-1)


class PPOTrainer:
    """PPO训练器"""
    def __init__(
        self,
        policy_model,
        ref_model,
        reward_model,
        kl_coef: float = 0.1
    ):
        self.policy = policy_model
        self.ref_model = ref_model
        self.reward_model = reward_model
        self.kl_coef = kl_coef
    
    def compute_kl_divergence(
        self,
        log_probs: torch.Tensor,
        ref_log_probs: torch.Tensor
    ) -> torch.Tensor:
        """计算策略与参考模型的KL散度"""
        return (log_probs - ref_log_probs).mean()
    
    def ppo_loss(
        self,
        log_probs: torch.Tensor,
        ref_log_probs: torch.Tensor,
        rewards: torch.Tensor,
        epsilon: float = 0.2
    ) -> torch.Tensor:
        """
        PPO剪辑损失
        
        Args:
            log_probs: 新策略的对数概率
            ref_log_probs: 参考策略的对数概率
            rewards: 奖励模型输出的奖励
            epsilon: 剪辑参数
        """
        # 计算比率
        ratio = torch.exp(log_probs - ref_log_probs)
        
        # PPO剪辑
        clipped_ratio = ratio.clamp(1 - epsilon, 1 + epsilon)
        
        # 最小化（负号因为是最小化损失）
        loss = -torch.min(
            ratio * rewards,
            clipped_ratio * rewards
        ).mean()
        
        # 添加KL惩罚
        kl_penalty = self.compute_kl_divergence(log_probs, ref_log_probs)
        
        return loss + self.kl_coef * kl_penalty
```

### RLHF的挑战

| 挑战 | 描述 | 解决方案 |
|------|------|----------|
| 标注成本 | 人类偏好标注耗时且昂贵 | RLAIF、用AI辅助标注 |
| 奖励黑客 | 模型找到奖励函数漏洞 | 多样本投票、混合奖励 |
| 分布偏移 | RL导致分布大幅偏离预训练 | KL惩罚、PPO剪辑 |
| 人类偏见 | 标注者偏见影响模型 | 多样化标注者、偏见检测 |

## Constitutional AI

### 核心理念

Constitutional AI（宪法AI）由Anthropic提出，核心思想是：

1. 定义一套**宪法原则**（Constitutional Principles）描述期望行为
2. 让AI基于这些原则进行**自我批判**和**修订**
3. 用修订后的结果训练模型

```
┌─────────────────────────────────────────────────────────────────┐
│                   Constitutional AI 流程                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  初始响应 ←── AI生成                                             │
│       │                                                        │
│       ▼                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 批判提示 (Critique Prompt)                               │   │
│  │ "请检查以下回复是否违反宪法原则：                        │   │
│  │ 1. 有帮助且无害                                          │   │
│  │ 2. 诚实且透明                                           │   │
│  │ 3. 不误导用户                                           │   │
│  │ ..."                                                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│       │                                                        │
│       ▼                                                        │
│  批判结果 ←── AI评估                                             │
│       │                                                        │
│       ▼                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 修订提示 (Revision Prompt)                              │   │
│  │ "请根据上述批判修订回复..."                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│       │                                                        │
│       ▼                                                        │
│  修订后响应 ←── AI生成                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 宪法原则设计

有效的宪法原则应涵盖：

```python
CONSTITUTIONAL_PRINCIPLES = """
请根据以下宪法原则评估AI回复：

1. 有帮助性 (Helpfulness)
   - 回复应对用户有实际价值
   - 避免过度冗长或无关内容

2. 无害性 (Harmlessness)
   - 不助长非法活动
   - 不包含暴力或令人不安的内容
   - 不侵犯他人权利和隐私

3. 诚实性 (Honesty)
   - 准确陈述事实
   - 承认不确定性和局限性
   - 不误导或欺骗用户

4. 公正性 (Fairness)
   - 不歧视特定群体
   - 提供平衡的观点

5. 透明性 (Transparency)
   - 清楚说明AI身份
   - 解释推理过程
"""
```

### CAI vs RLHF

| 维度 | RLHF | Constitutional AI |
|------|------|------------------|
| 反馈来源 | 人类标注 | AI自我评估 |
| 原则表达 | 隐式（偏好数据） | 显式（宪法文本） |
| 可扩展性 | 受限（标注成本） | 高（自我改进） |
| 可解释性 | 低 | 高（原则明确） |
| 效果 | 成熟稳定 | 持续改进 |

## DPO：直接偏好优化

### 问题背景

RLHF的主要问题是复杂性和不稳定性：

- 需要训练奖励模型
- 需要使用PPO进行强化学习优化
- KL约束和剪辑超参数敏感

DPO（Direct Preference Optimization）提出绕过强化学习，直接用偏好数据优化。

### 核心公式

DPO的损失函数：

$$
\mathcal{L}_{\text{DPO}} = -\mathbb{E}_{(x, y^+, y^-) \sim \mathcal{D}} \left[ \log \sigma \left( \beta \log \frac{P_\theta(y^+|x)}{P_{\text{ref}}(y^+|x)} - \beta \log \frac{P_\theta(y^-|x)}{P_{\text{ref}}(y^-|x)} \right) \right]
$$

其中：
- $y^+$ 是偏好（chosen）响应
- $y^-$ 是不偏好（rejected）响应
- $\beta$ 是温度参数，控制偏离参考模型的强度
- $\sigma$ 是sigmoid函数

### 直观理解

将公式变形：

$$
\log \frac{P_\theta(y^+|x)}{P_\theta(y^-|x)} = \log \frac{P_{\text{ref}}(y^+|x)}{P_{\text{ref}}(y^-|x)} + \beta \left( \log \frac{P_\theta(y^+|x)}{P_{\text{ref}}(y^+|x)} - \log \frac{P_\theta(y^-|x)}{P_{\text{ref}}(y^-|x)} \right)
$$

这可以理解为：**偏好比 = 参考模型的偏好比 × 策略偏离程度**

### 代码实现

```python
import torch
import torch.nn.functional as F

class DPO Trainer:
    """直接偏好优化训练器"""
    
    def __init__(
        self,
        policy_model,
        ref_model,
        beta: float = 0.1
    ):
        self.policy = policy_model
        self.ref_model = ref_model
        self.beta = beta
    
    def compute_log_probs(
        self,
        model,
        input_ids: torch.Tensor,
        attention_mask: torch.Tensor,
        labels: torch.Tensor
    ) -> torch.Tensor:
        """计算序列对数似然"""
        outputs = model(
            input_ids=input_ids,
            attention_mask=attention_mask,
            labels=labels
        )
        # 返回每个token的负对数似然之和
        return -outputs.loss
    
    def dpo_loss(
        self,
        # Chosen 序列
        chosen_input_ids: torch.Tensor,
        chosen_attention_mask: torch.Tensor,
        chosen_labels: torch.Tensor,
        # Rejected 序列
        rejected_input_ids: torch.Tensor,
        rejected_attention_mask: torch.Tensor,
        rejected_labels: torch.Tensor,
    ) -> torch.Tensor:
        """
        计算DPO损失
        
        Args:
            chosen_input_ids: 偏好响应的input_ids
            rejected_input_ids: 不偏好响应的input_ids
        """
        # 计算策略模型的对数概率
        policy_chosen_log_probs = self.compute_log_probs(
            self.policy, chosen_input_ids, chosen_attention_mask, chosen_labels
        )
        policy_rejected_log_probs = self.compute_log_probs(
            self.policy, rejected_input_ids, rejected_attention_mask, rejected_labels
        )
        
        # 计算参考模型的对数概率
        with torch.no_grad():
            ref_chosen_log_probs = self.compute_log_probs(
                self.ref_model, chosen_input_ids, chosen_attention_mask, chosen_labels
            )
            ref_rejected_log_probs = self.compute_log_probs(
                self.ref_model, rejected_input_ids, rejected_attention_mask, rejected_labels
            )
        
        # 计算偏好比
        chosen_rewards = self.beta * (policy_chosen_log_probs - ref_chosen_log_probs)
        rejected_rewards = self.beta * (policy_rejected_log_probs - ref_rejected_log_probs)
        
        # DPO损失：偏好比通过sigmoid压缩
        loss = -F.logsigmoid(chosen_rewards - rejected_rewards).mean()
        
        return loss
```

### DPO的优势与局限

| 优势 | 局限 |
|------|------|
| 简单：无须训练奖励模型 | 理论基础不如RLHF成熟 |
| 高效：无须PPO采样 | 对噪声偏好数据更敏感 |
| 稳定：无KL约束调参 | 复杂任务效果可能不如RLHF |
| 可解释：偏好比直接可见 | 需要足够的偏好数据 |

## RLAIF：AI反馈强化学习

### 核心思想

用AI（通常是更大的模型）替代人类进行反馈标注：

```
┌─────────────────────────────────────────────────────────────────┐
│                      RLAIF 流程                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  人类指令 ──▶ SFT模型 ──▶ 候选响应1, 候选响应2                   │
│                       │                                          │
│                       ▼                                          │
│              ┌────────────────┐                                  │
│              │  反馈模型      │  (通常是GPT-4级别)                │
│              │  (Feedback     │                                  │
│              │   Model)        │                                  │
│              └────────────────┘                                  │
│                       │                                          │
│                       ▼                                          │
│              偏好标签: Response1 > Response2                      │
│                       │                                          │
│                       ▼                                          │
│              标准RLHF/DPO训练                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 反馈提示设计

```python
RLAIF_FEEDBACK_PROMPT = """
你是一位AI反馈评估员。请对以下两个AI回复进行比较评估。

任务: {task_description}

回复A:
{response_a}

回复B:
{response_b}

评估维度:
1. 有帮助性：哪个回复更有助于解决用户问题？
2. 无害性：哪个回复更安全、更不会造成潜在伤害？
3. 诚实性：哪个回复更准确、更不会误导用户？

请直接输出比较结果，格式如下：
选择：A（或B）
原因：[简短解释]
"""
```

## 高级对齐技术

### 1. 迭代扩展监督

当任务复杂度超过人类直接评估能力时：

```python
class RecursiveOversight:
    """递归监督：当AI能力超过人类时"""
    
    def __init__(self, ai_agent, human_overseer):
        self.ai = ai_agent
        self.overseer = human_overseer
    
    def evaluate_complex_task(self, task: str, max_depth: int = 5):
        """递归分解直到人类可评估"""
        if max_depth == 0 or self.is_human_evaluable(task):
            return self.overseer.evaluate(task)
        
        # 分解为子任务
        subtasks = self.ai.decompose(task)
        
        # 递归评估子任务
        subtask_results = []
        for subtask in subtasks:
            result = self.evaluate_complex_task(subtask, max_depth - 1)
            subtask_results.append(result)
        
        # 聚合结果
        return self.ai.aggregate(subtask_results)
```

### 2. Debate（辩论）

让AI互相辩论，人类判断：

```python
class AIDebate:
    """AI辩论：两个AI互相论证"""
    
    def __init__(self, ai_1, ai_2):
        self.ai_1 = ai_1
        self.ai_2 = ai_2
    
    def debate(self, statement: str, rounds: int = 4):
        """进行多轮辩论"""
        context = {"statement": statement, "rounds": []}
        
        for round_num in range(rounds):
            # AI1论证
            argument_1 = self.ai_1.argue(
                context=context,
                role="support",
                round_num=round_num
            )
            context["rounds"].append({"speaker": "ai1", "argument": argument_1})
            
            # AI2反驳
            argument_2 = self.ai_2.argue(
                context=context,
                role="oppose",
                round_num=round_num
            )
            context["rounds"].append({"speaker": "ai2", "argument": argument_2})
        
        # 人类最终裁决
        return human_judge.decide(context)
```

## 相关主题

- [[ai-safety-introduction|AI安全概述]] — 对齐在AI安全中的位置
- [[../machine-learning/llm/llm-theory|LLM理论]] — 对齐技术的理论基础
- [[llm-security-guardrails|LLM安全护栏]] — 护栏是对齐的补充

## 参考资料

[^1]: Ouyang, L., et al. "Training language models to follow instructions with human feedback." *NeurIPS* 2022.

[^2]: Bai, Y., et al. "Constitutional AI: Harmlessness from AI Feedback." *arXiv* 2022.

[^3]: Rafailov, R., et al. "Direct Preference Optimization: Your Language Model is Secretly a Reward Model." *NeurIPS* 2023.

[^4]: Glaese, A., et al. "Improving alignment of dialogue agents via targeted human judgements." *arXiv* 2022.

[^5]: Anthropic. "Claude's Constitutional AI". 2022.
