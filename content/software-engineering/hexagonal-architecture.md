---
title: Hexagonal Architecture
date: 2026-04-14
description: 六边形架构（端口与适配器）深度解析
tags:
  - architecture
  - hexagonal
  - clean-architecture
  - design-patterns
draft: true
permalink:
---

# Hexagonal Architecture

## 概述

六边形架构（Hexagonal Architecture），又称端口与适配器（Ports and Adapters），由 Alistair Cockburn 于 2005 年提出。核心思想是将应用的核心业务逻辑与外部依赖（数据库、UI、外部服务）完全解耦。

## 核心概念

### 架构图示

```
                           ┌─────────────────────────────┐
                           │                             │
    ┌──────────┐          │     ┌─────────────────┐      │
    │   UI     │──────────│     │                 │      │
    └──────────┘          │     │   Application   │      │
                          │     │    Core /       │      │
    ┌──────────┐          │     │   Domain        │      │
    │   DB     │──────────│     │   (Business     │      │
    └──────────┘          │     │    Logic)        │      │
                          │     │                 │      │
    ┌──────────┐          │     └─────────────────┘      │
    │ External │──────────│            │                │
    │ Services │          │            │                │
    └──────────┘          │            ▼                │
                          │     ┌─────────────────┐      │
                           │     │   Ports         │      │
                           │     │   (Interfaces)  │      │
                           └─────┴─────────────────┘─────┘
```

### 关键术语

| 术语 | 含义 |
|------|------|
| **Core/Domain** | 应用核心业务逻辑，完全不依赖外部 |
| **Port** | 接口定义（抽象），描述 Core 能做什么 |
| **Adapter** | 适配器实现，连接 Core 与外部系统 |
| **Driving Adapter** | 主动触发 Core 的适配器（API、CLI） |
| **Driven Adapter** | 被 Core 调用的适配器（DB、MQ） |

## 分层结构

### 经典三层（扩展版）

```
┌─────────────────────────────────────────────────────────────┐
│                    Driving Adapters                          │
│         (Controllers, API Handlers, CLI commands)           │
├─────────────────────────────────────────────────────────────┤
│                    Application Layer                         │
│              (Use Cases, Application Services)              │
├─────────────────────────────────────────────────────────────┤
│                      Domain Layer                            │
│           (Entities, Value Objects, Domain Services)        │
├─────────────────────────────────────────────────────────────┤
│                    Infrastructure Layer                      │
│              (Repositories, External Services)              │
└─────────────────────────────────────────────────────────────┘
```

### 四层架构变体

```
┌─────────────────────────────────────────────────────────────┐
│                    Primary / Driving                         │
│           UI, API Gateway, Controllers                       │
├─────────────────────────────────────────────────────────────┤
│                   Secondary / Driven                        │
│         Databases, Message Queues, External APIs            │
├─────────────────────────────────────────────────────────────┤
│                         Ports                               │
│            (IUserRepository, IPaymentGateway)              │
├─────────────────────────────────────────────────────────────┤
│                        Core                                 │
│         Domain Logic (无任何外部依赖)                         │
└─────────────────────────────────────────────────────────────┘
```

## 核心实现

### 领域层（Domain）

```python
# domain/entities.py
from dataclasses import dataclass
from datetime import datetime
from typing import List, Optional

@dataclass
class Order:
    id: str
    customer_id: str
    items: List[OrderItem]
    status: OrderStatus
    created_at: datetime
    
    @property
    def total_amount(self) -> Money:
        return sum(item.subtotal for item in self.items)
    
    def can_cancel(self) -> bool:
        return self.status == OrderStatus.PENDING
    
    def cancel(self) -> None:
        if not self.can_cancel():
            raise OrderCannotBeCancelledError(self.id)
        self.status = OrderStatus.CANCELLED

@dataclass
class OrderItem:
    product_id: str
    quantity: int
    unit_price: Money
    # ...
```

### 端口定义（Ports）

```python
# ports/repositories.py
from abc import ABC, abstractmethod
from typing import List, Optional

class OrderRepository(ABC):
    """Port - 定义仓储接口"""
    
    @abstractmethod
    def find_by_id(self, order_id: str) -> Optional[Order]:
        pass
    
    @abstractmethod
    def find_by_customer(self, customer_id: str) -> List[Order]:
        pass
    
    @abstractmethod
    def save(self, order: Order) -> None:
        pass
    
    @abstractmethod
    def delete(self, order_id: str) -> None:
        pass

class PaymentGateway(ABC):
    """Port - 支付网关接口"""
    
    @abstractmethod
    def charge(self, amount: Money, payment_method: PaymentMethod) -> ChargeResult:
        pass
    
    @abstractmethod
    def refund(self, charge_id: str) -> RefundResult:
        pass

class NotificationService(ABC):
    """Port - 通知服务接口"""
    
    @abstractmethod
    def send_order_confirmation(self, order: Order) -> None:
        pass
    
    @abstractmethod
    def send_shipping_notification(self, order: Order) -> None:
        pass
```

### 应用服务（Application）

```python
# application/order_service.py
from dataclasses import inject

class OrderService:
    """Application Service - 编排业务用例"""
    
    @inject
    def __init__(
        self,
        order_repository: OrderRepository,
        payment_gateway: PaymentGateway,
        notification_service: NotificationService
    ):
        self._order_repository = order_repository
        self._payment_gateway = payment_gateway
        self._notification_service = notification_service
    
    def create_order(self, command: CreateOrderCommand) -> Order:
        # 1. 创建订单实体
        order = Order(
            id=self._generate_id(),
            customer_id=command.customer_id,
            items=self._build_items(command.items),
            status=OrderStatus.PENDING,
            created_at=datetime.now()
        )
        
        # 2. 持久化
        self._order_repository.save(order)
        
        # 3. 返回
        return order
    
    def cancel_order(self, command: CancelOrderCommand) -> None:
        order = self._order_repository.find_by_id(command.order_id)
        
        if order is None:
            raise OrderNotFoundError(command.order_id)
        
        order.cancel()
        self._order_repository.save(order)
```

### 适配器实现（Adapters）

```python
# adapters/persistence/sqlalchemy_order_repository.py
from sqlalchemy.orm import Session

class SQLAlchemyOrderRepository(OrderRepository):
    """Driven Adapter - SQLAlchemy 实现"""
    
    def __init__(self, session: Session):
        self._session = session
    
    def find_by_id(self, order_id: str) -> Optional[Order]:
        row = self._session.query(OrderRow).filter_by(id=order_id).first()
        return self._to_domain(row) if row else None
    
    def find_by_customer(self, customer_id: str) -> List[Order]:
        rows = self._session.query(OrderRow).filter_by(
            customer_id=customer_id
        ).all()
        return [self._to_domain(row) for row in rows]
    
    def save(self, order: Order) -> None:
        row = self._to_row(order)
        self._session.merge(row)
        self._session.commit()
    
    def _to_domain(self, row: OrderRow) -> Order:
        # SQLAlchemy Row -> Domain Entity
        ...
    
    def _to_row(self, order: Order) -> OrderRow:
        # Domain Entity -> SQLAlchemy Row
        ...
```

```python
# adapters/api/fastapi_controller.py
from fastapi import FastAPI, Depends, HTTPException

app = FastAPI()

@inject
async def create_order(
    command: CreateOrderCommand,
    service: OrderService
):
    try:
        order = service.create_order(command)
        return OrderResponse.from_domain(order)
    except OrderCannotBeCancelledError as e:
        raise HTTPException(status_code=400, detail=str(e))
```

## 依赖注入

```python
# dependency_injection.py
from wire import Injector, Module

class OrderModule(Module):
    def configure(self, binder):
        binder.bind(OrderRepository, to=SQLAlchemyOrderRepository)
        binder.bind(PaymentGateway, to=StripePaymentGateway)
        binder.bind(NotificationService, to=SendGridNotificationService)

# 或使用 inject 装饰器
injector = Injector([OrderModule()])
service = injector.create(OrderService)
```

## 六边形 vs 分层架构

| 维度 | 分层架构 | 六边形架构 |
|------|----------|------------|
| 依赖方向 | 上层依赖下层 | 外层依赖内层 |
| 核心逻辑 | 依赖基础设施 | 完全独立 |
| 可测试性 | 难以单元测试核心 | 核心逻辑无需外部依赖 |
| 可替换性 | 替换 DB 需改核心 | 仅替换适配器 |
| 学习曲线 | 直观 | 需要理解端口/适配器概念 |

## 六边形 vs Clean Architecture

两者高度相似，核心区别在于：

| 方面 | 六边形架构 | Clean Architecture |
|------|------------|-------------------|
| 起源 | Alistair Cockburn (2005) | Robert Martin (2012) |
| 核心概念 | 端口与适配器 | 依赖规则 + 实体 |
| 分层命名 | Ports/Adapters | Entities/Use Cases/Interfaces |
| 适用场景 | 强调外部系统解耦 | 强调业务规则封装 |

## 优势

1. **核心业务可测试**：无需依赖数据库或外部服务
2. **高度可替换**：更换数据库仅改适配器
3. **清晰边界**：依赖方向单一明确
4. **独立开发**：前端/后端可并行开发
5. **技术选型灵活**：随时切换技术栈

## 挑战

1. **初始复杂度**：需要额外抽象
2. **学习曲线**：端口/适配器概念需要理解
3. **过度设计风险**：小项目可能过于复杂
4. **映射复杂性**：领域对象与持久化对象转换

## 实践建议

- **从简单开始**：单模块应用可先用简单分层
- **渐进式演进**：逐步提取端口和适配器
- **专注核心**：保持 Domain 层纯净
- **使用框架辅助**：如 Spring (Java)、Django (Python) 配合使用

## 扩展阅读

- [Hexagonal Architecture 文章](https://alistair.cockburn.us/architecture+reading+binary/)
- [Ports and Adapters](https://alistair.cockburn.us/hexagonal+architecture/)
- [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Hands-On Domain-Driven Design](https://www.amazon.com/Hands-Domain-Driven-Design-Lembeck/dp/1838984542)

