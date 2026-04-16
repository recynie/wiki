---
title: Saga Pattern
date: 2026-04-15
description: 分布式系统中管理长事务的Saga模式
tags:
  - saga
  - distributed-systems
  - patterns
  - microservices
draft: false
permalink:
---

## 概述

Saga模式是分布式系统中管理跨服务长事务的核心模式。在微服务架构中，每个服务拥有独立的数据库，传统的分布式ACID事务（如2PC）难以实施。Saga模式通过将长事务分解为一系列本地事务，每个步骤附带补偿事务来实现最终一致性。[^1]

### 问题背景

在单体应用中，一个数据库事务可以原子性地跨越多个操作。但在微服务架构中：

- 每个服务拥有独立的数据库
- 跨服务的原子性事务无法直接实现
- 2PC（两阶段提交）在微服务中存在性能瓶颈和单点故障问题

Saga模式应运而生，成为分布式事务的事实标准。

### 与2PC对比

| 特性 | 2PC | Saga |
|------|-----|------|
| 一致性类型 | 强一致性 | 最终一致性 |
| 协调方式 | 集中式协调器 | 分散式 |
| 性能开销 | 高（全局锁） | 低 |
| 故障恢复 | 复杂 | 通过补偿事务 |
| 适用场景 | 紧密耦合系统 | 松耦合微服务 |

---

## 两种实现方式

Saga模式有两种主要的协调方式：编排（Orchestration）和 choreography（事件驱动）。[^2]

### 1. 编排式Saga（Orchestration）

编排式Saga使用一个中心协调器来管理整个工作流。协调器显式调用每个服务，跟踪 saga 状态，并根据响应决定下一步。

**架构图示**：

```
┌─────────────────────────────────────────────────────────┐
│                   Saga Orchestrator                      │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────┐ │
│  │ Step 1  │───▶│ Step 2  │───▶│ Step 3  │───▶│ End │ │
│  │(Inventory)│  │(Payment)│   │(Shipment)│   │     │ │
│  └────┬────┘    └────┬────┘    └────┬────┘    └─────┘ │
└───────┼─────────────┼─────────────┼──────────────────┘
        │             │             │
        ▼             ▼             ▼
   [Reserve]      [Charge]     [Arrange]
   Inventory      Payment      Shipping
```

**优点**：
- 完整的工作流在单一位置可见
- 便于调试和监控
- 易于添加条件逻辑和重试策略
- 中心化错误处理

**缺点**：
- 协调器可能成为单点故障
- 增加架构复杂度

**适用场景**：
- 复杂的多步骤工作流
- 需要条件分支的流程
- 需要审计和可追溯性的业务

### 2. 事件驱动式Saga（Choreography）

在事件驱动式Saga中，每个服务监听事件并发布自己的事件来触发下一步。没有中心协调器，每个服务自主响应。

**架构图示**：

```
┌─────────┐    OrderCreated     ┌─────────┐
│  Order  │────────────────────▶│Inventory│
│ Service │                    │ Service │
└────┬────┘                    └────┬────┘
     │                               │
     │ InventoryReserved             │ PaymentProcessed
     │                               │
     ▼                               ▼
┌─────────┐    ShipmentCreated   ┌─────────┐
│ Payment │─────────────────────▶│ Shipping│
│ Service │                    │ Service │
└─────────┘                    └─────────┘
```

**优点**：
- 服务间松耦合
- 无需额外的协调服务
- 适合简单的2-3步流程

**缺点**：
- 工作流分散，难以追踪
- 添加新步骤时可能变得混乱
- 服务间可能存在循环依赖

**适用场景**：
- 简单的2-3步流程
- 不同团队拥有的服务
- 最大化解耦的场景

---

## 补偿事务

补偿事务是Saga模式的核心。每个前进步骤都必须有对应的补偿事务，用于在后续步骤失败时撤销之前的操作。[^3]

### 补偿事务的设计原则

1. **幂等性**：补偿操作应该是幂等的，可以安全地重试
2. **可逆性**：每个操作都有明确的逆向操作
3. **独立性**：补偿事务独立执行，不依赖其他补偿

### 补偿示例

以电商订单处理为例：

| 步骤 | 正向操作 | 补偿操作 |
|------|----------|----------|
| 1 | 预留库存 | 释放库存 |
| 2 | 扣款 | 退款 |
| 3 | 创建发货 | 取消发货 |
| 4 | 发送确认 | 发送取消通知 |

```python
# 补偿事务示例
class OrderSaga:
    async def execute(self):
        try:
            # Step 1: Reserve inventory
            await self.inventory_service.reserve(self.order_id, self.items)
            
            # Step 2: Process payment
            await self.payment_service.charge(self.order_id, self.amount)
            
            # Step 3: Create shipment
            await self.shipping_service.create(self.order_id)
            
            # Step 4: Send confirmation
            await self.notification_service.confirm(self.order_id)
            
        except PaymentFailed:
            # Compensate step 1
            await self.inventory_service.release(self.order_id, self.items)
            raise
        
        except ShippingFailed:
            # Compensate steps 2 and 1
            await self.payment_service.refund(self.order_id, self.amount)
            await self.inventory_service.release(self.order_id, self.items)
            raise
```

---

## Outbox模式：确保可靠的事件发布

Saga模式要求服务原子性地更新数据库和发布事件。Outbox模式解决了这一问题。[^2]

### 问题

如果服务在更新数据库后、发布事件前崩溃，会导致数据不一致。

### 解决方案

Outbox模式将事件存储在本地数据库表中，作为业务事务的一部分。另一个进程定期读取Outbox表并发布事件。

```python
# Outbox模式实现
class OrderService:
    async def create_order(self, order_data):
        async with self.db.transaction():
            # 1. Create order
            order = await self.orders.create(order_data)
            
            # 2. Write to outbox (same transaction)
            await self.outbox.create({
                'aggregate_id': order.id,
                'event_type': 'OrderCreated',
                'payload': {'order_id': order.id, 'amount': order.amount}
            })
        
        # Outbox Relay publishes events asynchronously
```

### 推荐工具

- **Axon Framework**：支持Saga和Outbox
- **Eventuate Tram**：提供可靠的消息传递
- **Temporal**：工作流引擎，内置重试和补偿
- **Amazon Step Functions**：托管式工作流服务

---

## 实战案例：订单处理系统

### 场景描述

用户下单，涉及以下服务：
- Order Service（订单服务）
- Inventory Service（库存服务）
- Payment Service（支付服务）
- Shipping Service（发货服务）

### 编排式实现

```python
# Saga Orchestrator
class CreateOrderSaga:
    def __init__(self, order_id, items, payment_info):
        self.order_id = order_id
        self.items = items
        self.payment_info = payment_info
        self.steps = [
            SagaStep('reserve_inventory', 
                      self.reserve_inventory, 
                      self.release_inventory),
            SagaStep('charge_payment',
                      self.charge_payment,
                      self.refund_payment),
            SagaStep('create_shipment',
                      self.create_shipment,
                      self.cancel_shipment),
        ]
    
    async def execute(self):
        completed = []
        try:
            for step in self.steps:
                await step.execute()
                completed.append(step)
        except Exception as e:
            # Execute compensations in reverse order
            for step in reversed(completed):
                await step.compensate()
            raise SagaExecutionFailed(e)
```

### 事件驱动实现

```python
# Event-driven Saga Participant (Payment Service)
class PaymentService:
    async def handle_inventory_reserved(self, event):
        """Listener for InventoryReserved event"""
        try:
            await self.payment_service.charge(
                order_id=event['order_id'],
                amount=event['amount']
            )
            # Publish success event
            await self.event_bus.publish(
                PaymentProcessed(order_id=event['order_id'])
            )
        except PaymentDeclined:
            # Publish failure event - triggers compensation
            await self.event_bus.publish(
                PaymentFailed(order_id=event['order_id'], reason='declined')
            )
```

---

## 选择指南

| 因素 | 编排式 | 事件驱动式 |
|------|--------|------------|
| 服务数量 | 多（>3） | 少（2-3） |
| 流程复杂度 | 复杂、有条件分支 | 简单、线性 |
| 团队组织 | 同一团队 | 不同团队 |
| 可观测性需求 | 高 | 一般 |
| 耦合度 | 依赖协调器 | 松耦合 |

---

## 常见错误与避免

### 1. 滥用 choreography

**错误**：将复杂的多步骤流程也使用 choreography，导致难以追踪。

**建议**：超过3个步骤的流程使用编排式。

### 2. 补偿事务设计不当

**错误**：补偿事务不是真正的逆向操作，导致数据不一致。

**建议**：每个补偿事务必须完全撤销正向操作的影响。

### 3. 缺少幂等性

**错误**：补偿事务无法安全重试。

**建议**：使用幂等键或幂等操作确保补偿可以安全重试。

### 4. 忽略 Outbox 模式

**错误**：直接发布事件可能导致事件丢失。

**建议**：使用 Outbox 模式确保可靠的事件发布。

---

## 相关模式

- [[../distributed-systems/cqrs-pattern|CQRS模式]] — 常与Saga结合使用
- [[../distributed-systems/event-sourcing|事件溯源]] — 提供可靠的事件存储
- [[circuit-breaker|熔断器模式]] — 防止故障级联
- [[bulkhead-pattern|隔舱模式]] — 资源隔离

---

## 参考资料

[^1]: [Pattern: Saga - Microservices.io](https://microservices.io/patterns/data/saga)
[^2]: [Saga Pattern for Distributed Transactions - Md Sanwar Hossain](https://mdsanwarhossain.me/blog-saga-pattern.html)
[^3]: [Saga Design Pattern - Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/saga/saga)