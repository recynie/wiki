---
title: Circuit Breaker Pattern
date: 2026-04-15
description: 防止分布式系统级联故障的熔断器模式
tags:
  - circuit-breaker
  - distributed-systems
  - patterns
  - fault-tolerance
draft: false
permalink:
---

## 概述

Circuit Breaker（熔断器）模式是一种防止分布式系统中级联故障的弹性模式。通过监控调用失败次数，当故障达到阈值时"熔断"后续调用，快速失败以保护系统。[^1]

### 问题背景

在分布式系统中，一个服务的故障可能导致调用方长时间等待，最终耗尽资源，引发级联故障：

```
服务A → 服务B → 服务C（故障）
         ↑
         ├─ 等待超时...
         ├─ 等待超时...
         └─ 资源耗尽 → 服务A也故障
```

### 解决方案

熔断器在检测到下游服务故障后，**快速失败**而不是无限等待：

```
正常状态：
  请求 → 熔断器 → 服务B → 响应

熔断状态：
  请求 → 熔断器 → 直接返回失败（不调用服务B）
```

---

## 三状态机

熔断器有三种状态：[^2]

### 1. Closed（关闭状态）

正常工作，请求正常通过。

```
┌──────────────────────────────────────┐
│              CLOSED                  │
│  请求 → ● → [执行调用] → 服务B       │
│                                      │
│  失败计数 +1                         │
│  失败次数 < 阈值 → 保持 Closed       │
│  失败次数 ≥ 阈值 → 转为 Open        │
└──────────────────────────────────────┘
```

### 2. Open（打开状态）

熔断触发，所有请求快速失败。

```
┌──────────────────────────────────────┐
│               OPEN                   │
│                                      │
│  请求 → ● → [直接返回失败]           │
│                                      │
│  等待超时后 → 转为 Half-Open         │
└──────────────────────────────────────┘
```

### 3. Half-Open（半开状态）

探测服务是否恢复。

```
┌──────────────────────────────────────┐
│             HALF-OPEN                 │
│                                      │
│  限流请求通过 → 成功 → Closed        │
│  失败 → Open                         │
│                                      │
│  通常只允许 1-N 个探测请求           │
└──────────────────────────────────────┘
```

### 状态转换图

```
                    失败 ≥ 阈值
              ┌─────────────────────┐
              │                     │
              ▼                     │
         ┌────────┐                  │
         │ Closed │                  │
         └───┬────┘                  │
             │                      │
             │ 成功 < 阈值           │
             │                      │
    超时    │                      │ 失败
    之后    │                      │
             │                      │
             ▼                      │
         ┌────────┐                  │
         │  Open  │ ──────────────────┘
         └───┬────┘
             │
             │ 超时到期，允许探测
             │
             ▼
         ┌──────────┐
         │ Half-Open│
         └────┬─────┘
              │
         成功  │  失败
              │  ─────────┐
              ▼           │
         Closed           │
              │           ▼
              │         Open
              ▼
```

---

## 实现代码

### 最小实现

```python
import time
from enum import Enum
from typing import Callable, Any

class CircuitBreakerState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: int = 60,
        half_open_max_calls: int = 3
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout  # 秒
        self.half_open_max_calls = half_open_max_calls
        
        self.state = CircuitBreakerState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
        self.half_open_calls = 0
    
    def call(self, func: Callable[[], Any]) -> Any:
        """执行函数调用"""
        
        # 检查状态
        if self.state == CircuitBreakerState.OPEN:
            if self._should_attempt_reset():
                self._to_half_open()
            else:
                raise CircuitBreakerOpenError("Circuit breaker is OPEN")
        
        # Half-Open状态下限制调用数
        if self.state == CircuitBreakerState.HALF_OPEN:
            if self.half_open_calls >= self.half_open_max_calls:
                raise CircuitBreakerOpenError("Circuit breaker is HALF-OPEN, max calls reached")
            self.half_open_calls += 1
        
        # 执行调用
        try:
            result = func()
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise
    
    def _should_attempt_reset(self) -> bool:
        """检查是否应该尝试恢复"""
        if self.last_failure_time is None:
            return True
        return (time.time() - self.last_failure_time) >= self.recovery_timeout
    
    def _on_success(self):
        """处理成功"""
        if self.state == CircuitBreakerState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.half_open_max_calls:
                self._to_closed()
        else:
            # Closed状态下重置计数
            self.failure_count = 0
    
    def _on_failure(self):
        """处理失败"""
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.state == CircuitBreakerState.HALF_OPEN:
            self._to_open()
        elif self.failure_count >= self.failure_threshold:
            self._to_open()
    
    def _to_open(self):
        self.state = CircuitBreakerState.OPEN
        print(f"Circuit breaker: CLOSED → OPEN (failures: {self.failure_count})")
    
    def _to_half_open(self):
        self.state = CircuitBreakerState.HALF_OPEN
        self.half_open_calls = 0
        self.success_count = 0
        print("Circuit breaker: OPEN → HALF-OPEN")
    
    def _to_closed(self):
        self.state = CircuitBreakerState.CLOSED
        self.failure_count = 0
        self.half_open_calls = 0
        self.success_count = 0
        print("Circuit breaker: HALF-OPEN → CLOSED")


class CircuitBreakerOpenError(Exception):
    """熔断器打开时抛出的异常"""
    pass
```

### 使用示例

```python
import random

# 创建熔断器
breaker = CircuitBreaker(
    failure_threshold=3,
    recovery_timeout=5,
    half_open_max_calls=2
)

# 模拟不稳定的服务
def unstable_service():
    if random.random() < 0.7:  # 70% 失败率
        raise ConnectionError("Service unavailable")
    return "Success"

# 测试调用
for i in range(15):
    try:
        result = breaker.call(unstable_service)
        print(f"Call {i+1}: {result}")
    except CircuitBreakerOpenError as e:
        print(f"Call {i+1}: {e}")
    except ConnectionError as e:
        print(f"Call {i+1}: {e}")
```

---

## 配置参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `failure_threshold` | 触发熔断的失败次数 | 5-10 |
| `recovery_timeout` | 熔断后尝试恢复的等待时间（秒） | 30-60 |
| `half_open_max_calls` | 半开状态下允许的调用数 | 1-5 |
| `success_threshold` | 半开状态下恢复需要的成功次数 | 2-3 |

---

## 与降级策略结合

熔断器通常与降级（Fallback）策略结合使用。[^3]

### 降级模式

```python
def get_product_info(product_id: str) -> dict:
    """获取商品信息，熔断时返回缓存或默认值"""
    
    def primary_call():
        return product_service.get(product_id)
    
    def fallback():
        # 尝试从缓存获取
        cached = cache.get(f"product:{product_id}")
        if cached:
            return cached
        
        # 返回默认信息
        return {
            "id": product_id,
            "name": "Unknown Product",
            "price": 0,
            "from_cache": True
        }
    
    try:
        return breaker.call(primary_call)
    except CircuitBreakerOpenError:
        return fallback()
```

### 多级降级

```python
def get_recommendations(user_id: str) -> list:
    """多级降级策略"""
    
    # 第一级：ML推荐服务
    try:
        return ml_recommender.call(lambda: ml_service.get_recommendations(user_id))
    except CircuitBreakerOpenError:
        pass
    
    # 第二级：规则引擎
    try:
        return rule_engine.call(lambda: rule_service.get_recommendations(user_id))
    except CircuitBreakerOpenError:
        pass
    
    # 第三级：热门商品
    return popular_products.get_default()
```

---

## 与Bulkhead模式结合

熔断器和隔舱模式是互补的。隔舱模式防止资源被耗尽，熔断器快速检测和响应故障。[^2]

### 组合使用架构

```
┌─────────────────────────────────────────────────────────┐
│                    请求入口                              │
└─────────────────────┬───────────────────────────────────┘
                      │
        ┌─────────────┴─────────────┐
        │                           │
        ▼                           ▼
   ┌────────────┐            ┌────────────┐
   │  Bulkhead  │            │  Bulkhead  │
   │  (用户服务) │            │  (订单服务) │
   └─────┬──────┘            └─────┬──────┘
         │                         │
         ▼                         ▼
   ┌───────────┐            ┌───────────┐
   │Circuit Brk │            │Circuit Brk │
   │  User Svc │            │ Order Svc │
   └─────┬─────┘            └─────┬─────┘
         │                         │
         ▼                         ▼
   [用户数据库]              [订单数据库]
```

---

## 监控指标

生产环境中应该监控熔断器的状态：

```python
class CircuitBreakerMetrics:
    def __init__(self, name: str):
        self.name = name
        self.metrics = {
            'total_calls': 0,
            'successful_calls': 0,
            'failed_calls': 0,
            'circuit_open_count': 0,
            'state_changes': []
        }
    
    def record_call(self, success: bool):
        self.metrics['total_calls'] += 1
        if success:
            self.metrics['successful_calls'] += 1
        else:
            self.metrics['failed_calls'] += 1
    
    def record_state_change(self, from_state, to_state):
        self.metrics['state_changes'].append({
            'timestamp': time.time(),
            'from': from_state,
            'to': to_state
        })
    
    def get_success_rate(self) -> float:
        if self.metrics['total_calls'] == 0:
            return 0.0
        return self.metrics['successful_calls'] / self.metrics['total_calls']
    
    def export_prometheus(self):
        """导出Prometheus格式指标"""
        return f"""
# HELP circuit_breaker_calls_total Total number of calls
# TYPE circuit_breaker_calls_total counter
circuit_breaker_calls_total{{name="{self.name}"}} {self.metrics['total_calls']}

# HELP circuit_breaker_success_rate Success rate
# TYPE circuit_breaker_success_rate gauge
circuit_breaker_success_rate{{name="{self.name}"}} {self.get_success_rate()}
"""
```

---

## 常见错误

### 1. 熔断阈值设置不当

- **过小**：正常波动也会触发熔断
- **过大**：无法及时保护下游服务

### 2. 没有设置合理的恢复超时

- **过短**：服务未恢复就尝试调用
- **过长**：长时间无法恢复

### 3. 忽略幂等性

- 熔断恢复后，被中断的请求可能需要重试
- 确保服务支持幂等操作

### 4. 没有降级策略

- 熔断后应该提供有意义的响应
- 而不是简单抛出异常

---

## 主流框架实现

| 框架 | 语言 | 特性 |
|------|------|------|
| Resilience4j | Java | 轻量、功能丰富 |
| Hystrix | Java | Netflix出品，已停止维护 |
| Polly | .NET | 集成ASP.NET Core |
| pybreaker | Python | 简洁实现 |
| Golang.org/x/time/rate | Go | 限流辅助 |
| ocs-spring-circuitbreaker | Spring | Spring生态 |

---

## 相关模式

- [[bulkhead-pattern|隔舱模式]] — 资源隔离
- [[saga-pattern|Saga模式]] — 分布式事务管理
- [[cqrs-pattern|CQRS模式]] — 读写分离

---

## 参考资料

[^1]: [Essential Distributed Systems Patterns - Software Interviews](https://softwareinterviews.com/articles/distributed-systems-patterns/)
[^2]: [Components of Distributed Systems 2026 - TheLinuxCode](https://thelinuxcode.com/components-of-a-distributed-system-in-2026-a-practitioners-guide/)
[^3]: [Circuit Breaker Pattern - Martin Fowler](https://martinfowler.com/bliki/CircuitBreaker.html)