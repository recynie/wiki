---
title: 事件溯源与CQRS
date: 2026-04-13
description: 事件溯源（Event Sourcing）与CQRS架构模式详解
tags:
  - event-sourcing
  - cqrs
  - distributed-systems
  - architecture-patterns
draft: false
permalink:
---

## 概述

事件溯源（Event Sourcing）和CQRS（Command Query Responsibility Segregation，命令查询职责分离）是两种互补的架构模式，广泛应用于分布式系统、微服务架构和事件驱动系统中。

**核心思想**：不只保存系统当前状态，而是保存导致状态变更的完整事件序列。

## 事件溯源（Event Sourcing）

### 定义

事件溯源的核心思想：**确保应用状态的每一次变更都被捕获为一个事件对象，这些事件对象按照应用顺序存储，与应用状态本身具有相同的生命周期。**[^1]

### 与传统模式对比

```
传统模式（只保存当前状态）：
┌─────────────┐
│   当前状态   │  ← 只知道当前位置
│  King Roy   │
│ → Hong Kong │
└─────────────┘

事件溯源模式：
┌─────────────────────────────┐
│  King Roy 事件日志          │
│  1. 离开 San Francisco     │
│  2. 抵达 Hong Kong         │
└─────────────────────────────┘
     ↓ 同时保存
┌─────────────────────────────┐
│  Ship对象 + Event Store    │
└─────────────────────────────┘
```

### 核心机制

#### Event Store（事件存储）

事件被持久化，与应用对象存储方式相同。每个事件包含：
- 事件类型
- 发生时间
- 相关数据
- 序列号/版本

#### Complete Rebuild（完全重建）

可以丢弃应用状态，通过在空应用上重新运行事件日志来重建：

```csharp
class EventProcessor {
    IList<Event> log = new ArrayList();
    
    public void Process(DomainEvent e) {
        e.Process();
        log.Add(e);
    }
}
```

#### Event Replay（事件重放）

发现过去事件有误时，通过**撤销该事件及其后续事件，然后重放新事件及其后续事件**来修正：

```csharp
abstract class DomainEvent {
    DateTime _recorded, _occurred;
    
    abstract internal void Process();
    virtual internal void Reverse() { }  // 支持撤销
}
```

#### Temporal Query（时序查询）

可以在任意时间点确定应用状态——从空白状态开始重放事件到特定时间或事件：

```
起点 ──→ 事件1 ──→ 事件2 ──→ ... ──→ T时刻状态
```

## CQRS（命令查询职责分离）

### 定义

CQRS将系统操作分为两类：
- **命令（Command）**：修改状态的操作，返回void或操作结果
- **查询（Query）**：读取状态的操作，返回数据

### 架构图

```
命令路径：
Client → Command Handler → Domain Model → Event Bus → Event Store
                                      ↓
                              更新聚合根

查询路径：
Client → Query Handler → Materialized View (Read Model)
```

### 与事件溯源的关系

事件溯源的架构天然支持CQRS模式：
- **写路径**：通过事件日志记录所有变更
- **读路径**：可以构建多种物化视图（不同系统可能使用不同的模型）

## 优势

| 优势 | 说明 |
|------|------|
| **完整审计跟踪** | 完整记录所有变更，可用作审计日志 |
| **时序查询** | 可查询任意历史时间点的状态 |
| **事件重放** | 支持回溯、撤销、重放，利于调试 |
| **完全重建** | 可完全丢弃状态后重建 |
| **事件驱动架构** | 支持水平扩展，适合高并发读、低并发写的场景 |
| **事件日志** | 可作为消息总线，分发到多个下游系统 |

## 劣势与挑战

| 劣势 | 说明 |
|------|------|
| **事件schema演化** | 事件结构随代码演化变得复杂；需要处理"旧事件用新代码重放"的问题 |
| **最终一致性** | 读者系统与主系统会因事件传播时序不同而暂时不同步 |
| **外部系统交互** | 外部查询和外部更新需要特殊处理（Gateway模式） |
| **双重复杂性** | 需要同时维护应用状态和事件日志两套数据 |
| **查询复杂度** | 对于简单查询，事件溯源可能过度设计 |

## 典型应用场景

### 适合使用事件溯源

- **需要完整审计跟踪**：金融系统、订单系统
- **需要时序查询**：版本控制系统（如Git、Subversion）
- **需要事件驱动**：微服务间的事件通知
- **需要业务历史**：保险理赔、审批流程

### 典型案例

| 系统 | 事件溯源应用 |
|------|-------------|
| Git | 每次提交是一个事件，HEAD状态可通过重放重建 |
| 银行账户 | 所有交易记录保存，可查任意时间点余额 |
| 电商订单 | 订单状态变更事件：创建、支付、发货、收货 |

## 实现要点

### 事件设计原则

1. **事件是不可变的**：只追加，不修改
2. **使用有意义的动词**：Created, Updated, Deleted, Shipped
3. **包含足够上下文**：让事件能自描述
4. **版本化**：预留扩展字段

```json
{
  "eventType": "OrderShipped",
  "occurredAt": "2026-04-13T10:30:00Z",
  "aggregateId": "order-123",
  "version": 1,
  "data": {
    "shipmentId": "ship-456",
    "carrier": "SF Express",
    "trackingNumber": "SF123456789"
  }
}
```

### 投影（Projection）

投影用于从事件构建读模型（物化视图）：

```python
class OrderReadModel:
    def project(self, event):
        if event.type == "OrderCreated":
            self.orders[event.orderId] = {"status": "created", ...}
        elif event.type == "OrderShipped":
            self.orders[event.orderId]["status"] = "shipped"
```

## 与其他模式的关系

- **Event Sourcing + CQRS**：天然互补，ES提供命令侧数据，投影提供查询侧数据
- **Event Sourcing + Saga**：处理跨服务的分布式事务
- **Event Sourcing + CDC**：将数据库变更捕获为事件

## 参考资料

[^1]: Martin Fowler, *Event Sourcing* (https://martinfowler.com/eaaDev/EventSourcing.html)

[^2]: Greg Young, *CQRS and Event Sourcing* - CQRS先驱的演讲和论文

[^3]: Event Store documentation (https://eventstore.org/)
