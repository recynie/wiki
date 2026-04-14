---
title: 软件架构模式
date: 2026-04-13
description: 常见架构模式：微服务、事件驱动、CQRS、六边形架构
tags:
  - software-engineering
  - architecture
  - microservices
  - patterns
draft: false
permalink:
---

# 软件架构模式

软件架构模式是经过验证的解决方案模板，用于解决常见的系统设计问题。

## 分层架构（Layered Architecture）

最传统的架构模式，将系统分为多个层级：

```
┌─────────────────────────┐
│     Presentation        │ ← 用户界面、API网关
├─────────────────────────┤
│      Application        │ ← 用例编排、事务管理
├─────────────────────────┤
│        Domain           │ ← 业务逻辑、领域模型
├─────────────────────────┤
│      Infrastructure     │ ← 数据库、外部服务
└─────────────────────────┘
```

### Python实现示例

```python
# Domain层（核心业务逻辑）
class Order:
    def __init__(self, order_id, customer_id):
        self.order_id = order_id
        self.customer_id = customer_id
        self.items = []
        self.status = "pending"
    
    def add_item(self, product_id, quantity, price):
        self.items.append({
            "product_id": product_id,
            "quantity": quantity,
            "price": price
        })
    
    def calculate_total(self):
        return sum(item["quantity"] * item["price"] for item in self.items)
    
    def confirm(self):
        if not self.items:
            raise ValueError("Cannot confirm empty order")
        self.status = "confirmed"

# Application层（用例编排）
class OrderService:
    def __init__(self, order_repository, payment_service, notification_service):
        self.order_repository = order_repository
        self.payment_service = payment_service
        self.notification_service = notification_service
    
    def create_order(self, customer_id, items):
        order = Order(order_id=None, customer_id=customer_id)
        
        for item in items:
            order.add_item(item["product_id"], item["quantity"], item["price"])
        
        saved_order = self.order_repository.save(order)
        
        # 触发后续流程
        self._send_order_created_event(saved_order)
        
        return saved_order
    
    def confirm_order(self, order_id):
        order = self.order_repository.find_by_id(order_id)
        order.confirm()
        self.order_repository.save(order)
        
        self.payment_service.request_payment(order)
        
        return order

# Infrastructure层
class SQLAlchemyOrderRepository(OrderRepository):
    def __init__(self, session):
        self.session = session
    
    def save(self, order):
        db_order = OrderModel(order)
        self.session.add(db_order)
        self.session.commit()
        return db_order.to_domain()
```

## 六边形架构（Hexagonal Architecture）

核心业务逻辑与外部依赖完全解耦：

```
                    ┌────────────────────────┐
                    │      Application      │
                    │       Core            │
                    │   ┌──────────────┐   │
                    │   │   Domain     │   │
                    │   │   Model      │   │
                    │   └──────────────┘   │
                    │          ▲          │
                    │          │          │
                    │   ┌──────────────┐   │
                    │   │   Ports      │   │ ← 接口定义
                    │   │  (输入/输出)  │   │
                    │   └──────────────┘   │
                    └──────────┬───────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
┌────────┴────────┐   ┌────────┴────────┐   ┌────────┴────────┐
│   Adapters     │   │   Adapters      │   │   Adapters     │
│  (REST API)    │   │  (Database)     │   │  (Message Q)   │
└────────────────┘   └─────────────────┘   └────────────────┘
```

```python
# Ports（接口定义）
class OrderRepository(Protocol):
    """输出端口：订单仓储接口"""
    def save(self, order: Order) -> Order: ...
    def find_by_id(self, order_id: str) -> Order: ...

class PaymentGateway(Protocol):
    """输出端口：支付网关接口"""
    def charge(self, amount: float, payment_method: str) -> PaymentResult: ...

class NotificationPort(Protocol):
    """输出端口：通知接口"""
    def send(self, recipient: str, message: str) -> None: ...

# Application Service
class OrderApplicationService:
    def __init__(
        self,
        order_repository: OrderRepository,
        payment_gateway: PaymentGateway,
        notification: NotificationPort
    ):
        self.order_repository = order_repository
        self.payment_gateway = payment_gateway
        self.notification = notification
    
    def place_order(self, customer_id: str, items: List[OrderItem]) -> Order:
        order = Order.create(customer_id, items)
        saved_order = self.order_repository.save(order)
        
        # 通过端口与外部交互，完全解耦
        result = self.payment_gateway.charge(
            amount=saved_order.total,
            payment_method="credit_card"
        )
        
        if result.success:
            saved_order.mark_paid()
            self.order_repository.save(saved_order)
            self.notification.send(
                saved_order.customer_id,
                f"Order {saved_order.id} confirmed!"
            )
        
        return saved_order

# Adapters（适配器实现）
class DjangoORMOrderRepository:
    """数据库适配器"""
    def __init__(self, model_class):
        self.model_class = model_class
    
    def save(self, order: Order) -> Order:
        obj = self.model_class(
            id=order.id,
            customer_id=order.customer_id,
            status=order.status
        )
        obj.save()
        return Order.from_dict(obj.to_dict())

class StripePaymentAdapter:
    """支付网关适配器"""
    def __init__(self, api_key: str):
        self.stripe = Stripe(api_key)
    
    def charge(self, amount: float, payment_method: str) -> PaymentResult:
        try:
            charge = self.stripe.Charge.create(
                amount=int(amount * 100),
                currency="usd",
                source=payment_method
            )
            return PaymentResult(success=True, transaction_id=charge.id)
        except Exception as e:
            return PaymentResult(success=False, error=str(e))
```

## 事件驱动架构（Event-Driven Architecture）

```
┌──────────┐    Event    ┌──────────┐    Event    ┌──────────┐
│ Producer │ ──────────► │  Event   │ ──────────► │ Consumer │
│   App    │             │   Bus    │             │   App    │
└──────────┘             └──────────┘             └──────────┘
     │                                               │
     │                                               │
     ▼                                               ▼
┌──────────┐             ┌──────────┐            ┌──────────┐
│  Event   │             │  Event   │            │  Event   │
│  Store   │             │  Broker  │            │ Handler  │
└──────────┘             └──────────┘            └──────────┘
```

### 事件溯源（Event Sourcing）

```python
from dataclasses import dataclass
from typing import List
import uuid
from datetime import datetime

@dataclass
class Event:
    event_id: str
    event_type: str
    timestamp: datetime
    payload: dict

class EventStore:
    def __init__(self):
        self.events: List[Event] = []
    
    def append(self, event_type: str, payload: dict):
        event = Event(
            event_id=str(uuid.uuid4()),
            event_type=event_type,
            timestamp=datetime.now(),
            payload=payload
        )
        self.events.append(event)
        return event
    
    def get_events_for_aggregate(self, aggregate_id: str) -> List[Event]:
        return [e for e in self.events if e.payload.get("id") == aggregate_id]
    
    def replay_events(self, aggregate_id: str, reducer):
        events = self.get_events_for_aggregate(aggregate_id)
        return reduce(reducer, events, None)

# 聚合根
class Order:
    def __init__(self, order_id: str):
        self.order_id = order_id
        self.status = "created"
        self.items = []
    
    @classmethod
    def from_events(cls, events: List[Event]):
        order = cls(events[0].payload["id"])
        for event in events:
            order._apply(event)
        return order
    
    def _apply(self, event: Event):
        if event.event_type == "OrderItemAdded":
            self.items.append(event.payload["item"])
        elif event.event_type == "OrderConfirmed":
            self.status = "confirmed"
    
    def add_item(self, item: dict, event_store: EventStore):
        event_store.append("OrderItemAdded", {
            "id": self.order_id,
            "item": item
        })
        self.items.append(item)
    
    def confirm(self, event_store: EventStore):
        event_store.append("OrderConfirmed", {"id": self.order_id})
        self.status = "confirmed"
```

### CQRS（命令查询职责分离）

```python
# 命令端（写入）
class OrderCommandHandler:
    def __init__(self, event_store: EventStore):
        self.event_store = event_store
    
    def handle_create_order(self, command: CreateOrderCommand):
        order_id = str(uuid.uuid4())
        
        # 创建订单事件
        self.event_store.append("OrderCreated", {
            "id": order_id,
            "customer_id": command.customer_id,
            "items": command.items,
            "timestamp": datetime.now().isoformat()
        })
        
        return order_id

# 查询端（读取）
class OrderQueryHandler:
    def __init__(self, read_db: ReadDatabase):
        self.read_db = read_db
    
    def get_order_summary(self, order_id: str) -> OrderSummaryDTO:
        # 从物化视图读取
        return self.read_db.queries.get_order_summary(order_id)
    
    def get_customer_orders(self, customer_id: str) -> List[OrderSummaryDTO]:
        return self.read_db.queries.get_orders_by_customer(customer_id)

# 物化视图更新
class ReadModelProjector:
    def __init__(self, read_db: WriteDatabase):
        self.read_db = read_db
    
    def project(self, event: Event):
        if event.event_type == "OrderCreated":
            self.read_db.execute("""
                INSERT INTO order_summaries (id, customer_id, status, total)
                VALUES (?, ?, ?, ?)
            """, [
                event.payload["id"],
                event.payload["customer_id"],
                "created",
                sum(item["price"] * item["quantity"] for item in event.payload["items"])
            ])
```

## 微服务架构模式

### 服务发现

```python
# 服务注册中心
class ServiceRegistry:
    def __init__(self):
        self.services = {}
    
    def register(self, service_name: str, instance_id: str, host: str, port: int):
        key = f"{service_name}/{instance_id}"
        self.services[key] = {
            "service_name": service_name,
            "instance_id": instance_id,
            "host": host,
            "port": port,
            "status": "UP",
            "last_heartbeat": time.time()
        }
    
    def deregister(self, service_name: str, instance_id: str):
        key = f"{service_name}/{instance_id}"
        if key in self.services:
            del self.services[key]
    
    def find_instances(self, service_name: str) -> List[Dict]:
        return [
            inst for key, inst in self.services.items()
            if inst["service_name"] == service_name and inst["status"] == "UP"
        ]

# 客户端负载均衡
class ClientSideLoadBalancer:
    def __init__(self, registry: ServiceRegistry):
        self.registry = registry
        self.strategy = RoundRobinStrategy()
    
    def call_service(self, service_name: str, path: str):
        instances = self.registry.find_instances(service_name)
        
        if not instances:
            raise ServiceUnavailable(f"No instances of {service_name}")
        
        selected = self.strategy.select(instances)
        
        url = f"http://{selected['host']}:{selected['port']}{path}"
        return requests.get(url)
```

### 熔断器（Circuit Breaker）

```python
import time
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"      # 正常
    OPEN = "open"          # 熔断
    HALF_OPEN = "half_open"  # 半开

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60, success_threshold=2):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.success_threshold = success_threshold
        
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
    
    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.timeout:
                self.state = CircuitState.HALF_OPEN
            else:
                raise CircuitOpenException("Circuit is OPEN")
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise
    
    def _on_success(self):
        self.failure_count = 0
        
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.success_threshold:
                self.state = CircuitState.CLOSED
                self.success_count = 0
    
    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
```

## 参考

[^1]: Patterns of Enterprise Application Architecture - Martin Fowler
[^2]: Building Microservices - Sam Newman
[^3]: Domain-Driven Design - Eric Evans

