---
title: CQRS Pattern
date: 2026-04-15
description: 命令查询职责分离模式
tags:
  - cqrs
  - distributed-systems
  - patterns
  - architecture
draft: false
permalink:
---

## 概述

CQRS（Command Query Responsibility Segregation，命令查询职责分离）是一种架构模式，将修改状态的操作（命令）与读取状态的操作（查询）分离到不同的数据模型中。[^1]

### 核心概念

CQRS基于Bertrand Meyer在1988年提出的CQS（Command-Query Separation，命令查询分离）原则。区别在于：

- **CQS**：适用于方法级别，同一方法要么是执行操作（命令），要么是返回数据（查询）
- **CQRS**：适用于架构级别，不同的服务、存储引擎或部署单元处理各自职责

### 为什么需要CQRS

在传统CRUD应用中，同一个数据模型同时处理读写操作。这在大多数场景下是可行的，但随着系统复杂度增加，会面临：

- **读写不对称**：大多数系统的读操作比写操作多得多（常见比例10:1或更高）
- **模型冲突**：优化读性能（反规范化）会使写操作变得复杂
- **扩展困难**：无法独立扩展读和写的容量

---

## CQRS核心架构

### 命令端（Write Side）

命令端处理所有状态修改操作：

```python
# Command Handler 示例
class PlaceOrderCommand:
    def __init__(self, customer_id, items, shipping_address):
        self.customer_id = customer_id
        self.items = items
        self.shipping_address = shipping_address

class OrderCommandHandler:
    def handle(self, command: PlaceOrderCommand):
        # 1. 验证业务规则
        customer = self.customer_repository.get(command.customer_id)
        if not customer.can_place_order():
            raise OrderValidationError("Customer not eligible")
        
        # 2. 创建订单聚合根
        order = Order(customer_id=command.customer_id)
        for item in command.items:
            order.add_item(item)
        
        # 3. 保存到写模型
        self.order_repository.save(order)
        
        # 4. 发布事件
        self.event_bus.publish(OrderPlacedEvent(order.id))
        
        return order.id
```

### 查询端（Read Side）

查询端优化读取性能，使用反规范化的视图模型：

```python
# Read Model (Projection)
class OrderView:
    """查询优化的订单视图"""
    def __init__(self):
        self.order_id: str
        self.customer_name: str
        self.items_summary: str
        self.total_amount: decimal
        self.status: str
        self.created_at: datetime

# Query Handler 示例
class OrderQueryHandler:
    def get_order_summary(self, order_id: str) -> OrderView:
        return self.read_db.query(
            "SELECT * FROM order_views WHERE order_id = ?",
            order_id
        )
    
    def get_customer_orders(self, customer_id: str) -> List[OrderView]:
        return self.read_db.query(
            "SELECT * FROM order_views WHERE customer_id = ? ORDER BY created_at DESC",
            customer_id
        )
```

---

## 与Event Sourcing结合

CQRS和Event Sourcing是独立的模式，可以单独使用，也可以结合使用。结合使用时，命令端发布的事件成为查询端重建视图的输入。[^2]

### 架构图示

```
┌─────────────────────────────────────────────────────────────────┐
│                         CQRS + Event Sourcing                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐         ┌──────────────┐                      │
│  │   Commands   │         │    Queries   │                      │
│  │   (Write)   │         │    (Read)    │                      │
│  └──────┬───────┘         └──────▲───────┘                      │
│         │                        │                              │
│         ▼                        │                              │
│  ┌──────────────┐         ┌──────┴───────┐                      │
│  │   Command    │         │  Projections │                      │
│  │   Handler    │         │   (Views)    │                      │
│  └──────┬───────┘         └──────▲───────┘                      │
│         │                        │                              │
│         ▼                        │                              │
│  ┌──────────────┐         ┌──────┴───────┐                      │
│  │   Aggregate  │────Events────────────────▶│                      │
│  │   (Domain)   │         │   Read DB     │                      │
│  └──────┬───────┘         └───────────────┘                      │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────┐                                                │
│  │  Event Store │  (Source of Truth)                             │
│  └──────────────┘                                                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 结合优势

1. **完整审计日志**：事件存储包含所有状态变更的完整历史
2. **灵活的查询**：可以基于历史事件重建任意时间点的视图
3. **独立扩展**：读模型和写模型可以独立扩展
4. **解耦**：查询模型可以完全重新设计而不影响业务逻辑

---

## 实现方式

### 1. 同步更新（Polling）

命令端更新后，通过轮询或推送机制更新查询端：

```python
# 同步更新实现
class OrderService:
    async def place_order(self, command):
        order_id = await self.command_handler.handle(command)
        
        # 同步更新查询数据库
        await self.projection_rebuilder.rebuild_order_view(order_id)
        
        return order_id
```

### 2. 事件驱动更新（推荐）

命令端发布事件，查询端订阅事件并更新：

```python
# 事件驱动更新
class OrderProjection:
    async def on_order_placed(self, event: OrderPlacedEvent):
        view = OrderView(
            order_id=event.order_id,
            customer_name=await self.get_customer_name(event.customer_id),
            items_summary=self.summarize_items(event.items),
            total_amount=event.total,
            status='PLACED',
            created_at=event.timestamp
        )
        await self.read_db.insert(view)
    
    async def on_payment_received(self, event: PaymentReceivedEvent):
        await self.read_db.update(
            'order_views',
            {'status': 'PAID'},
            {'order_id': event.order_id}
        )
```

### 3. 使用Apache Kafka

Kafka作为事件流平台，实现命令端和查询端的完全解耦：

```python
# Kafka事件处理
class OrderEventConsumer:
    def __init__(self, kafka_consumer, read_db):
        self.consumer = kafka_consumer
        self.read_db = read_db
    
    async def start(self):
        async for message in self.consumer:
            event = json.loads(message.value)
            
            if event['type'] == 'OrderPlaced':
                await self.handle_order_placed(event)
            elif event['type'] == 'PaymentReceived':
                await self.handle_payment_received(event)
```

---

## 适用场景

### 适合使用CQRS的场景

| 场景 | 原因 |
|------|------|
| 读/写比例严重不对称 | 读操作远多于写操作，需要独立优化 |
| 多租户系统 | 不同租户需要不同的查询视图 |
| 复杂领域模型 | 写模型和读模型有不同的优化需求 |
| 高并发系统 | 需要独立扩展读和写容量 |
| 需要审计 | 事件日志天然提供完整的审计能力 |

### 不适合使用CQRS的场景

| 场景 | 原因 |
|------|------|
| 简单CRUD应用 | 额外复杂度不值得 |
| 低并发系统 | 增加的复杂度超过收益 |
| 强一致性要求 | 事件传播需要时间，存在短暂不一致 |

---

## 读写分离配置示例

### AWS实现

```python
# AWS Lambda + DynamoDB 实现CQRS
class CommandLambda:
    def handle(self, event, context):
        command = parse_command(event)
        
        # 写入DynamoDB（写模型）
        self.dynamodb.put_item(TableName='Orders', Item={
            'order_id': {'S': str(uuid.uuid4())},
            'customer_id': {'S': command.customer_id},
            'status': {'S': 'PENDING'},
            'created_at': {'S': datetime.now().isoformat()}
        })
        
        # DynamoDB Streams触发读模型更新
        return {'statusCode': 200}

class QueryLambda:
    def handle(self, event, context):
        order_id = event['pathParameters']['order_id']
        
        # 从读模型读取（Elasticsearch优化查询）
        result = self.es.search(
            index='order_views',
            body={'query': {'match': {'order_id': order_id}}}
        )
        
        return {'statusCode': 200, 'body': json.dumps(result)}
```

### PostgreSQL实现

```sql
-- 写模型：规范化的订单表
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);

-- 读模型：反规范化的订单视图
CREATE TABLE order_views (
    order_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    customer_name VARCHAR(255),
    items_summary TEXT,
    total_amount DECIMAL(10, 2),
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMP NOT NULL
);

-- 物化视图自动更新（PostgreSQL特性）
CREATE MATERIALIZED VIEW order_summary AS
SELECT 
    o.order_id,
    c.name as customer_name,
    SUM(oi.quantity * p.price) as total_amount,
    o.status
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
GROUP BY o.order_id, c.name, o.status;
```

---

## 最佳实践

### 1. 渐进式采用

不要在整系统范围内立即应用CQRS。首先识别读写不对称的模块，从小处着手：

```python
# 渐进式采用示例
class OrderService:
    def __init__(self):
        # 保持简单的CRUD操作
        self.crud_repository = SimpleOrderRepository()
        
        # 仅对复杂查询使用专用读模型
        self.analytics_repository = AnalyticsOrderRepository()
```

### 2. 清晰的边界

命令和查询应该有清晰的API边界：

```python
# 命令端API - 返回结果但不返回实体
POST /commands/place-order    # 返回 { order_id: "xxx" }
POST /commands/cancel-order   # 返回 { success: true }

# 查询端API - 返回视图模型
GET /queries/order-summary/123   # 返回 OrderSummaryView
GET /queries/customer-orders     # 返回 List<OrderSummaryView>
```

### 3. 处理最终一致性

CQRS天然引入最终一致性，需要优雅处理：

```python
class OrderController:
    def place_order(self, command):
        order_id = self.command_service.execute(command)
        
        # 返回订单ID，但明确说明状态需要时间同步
        return {
            'order_id': order_id,
            'message': 'Order placed. Status will update shortly.'
        }
    
    def get_order_status(self, order_id):
        view = self.query_service.get_order_view(order_id)
        
        if view is None:
            # 命令尚未传播到读模型
            return {'status': 'PROCESSING'}
        
        return {'status': view.status}
```

---

## 常见错误

### 1. 过度使用

CQRS增加了显著的系统复杂度。只在确实需要时才使用。

### 2. 忽略事件顺序

在分布式系统中，事件可能不按顺序到达。需要显式处理乱序事件。

### 3. 直接读取写模型

查询端应该始终读取查询数据库，不要绕过。

### 4. 混合使用

不应该在命令端和查询端使用相同的ORM或数据访问模式。

---

## 相关模式

- [[saga-pattern|Saga模式]] — 管理跨服务的分布式事务
- [[event-sourcing|事件溯源]] — 完整的审计日志
- [[../databases/database-index-principles|数据库索引原理]] — 读模型优化基础

---

## 参考资料

[^1]: [CQRS Pattern - Microsoft Azure](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs)
[^2]: [CQRS and Event Sourcing in Production - Md Sanwar Hossain](https://mdsanwarhossain.me/blog-cqrs-event-sourcing.html)