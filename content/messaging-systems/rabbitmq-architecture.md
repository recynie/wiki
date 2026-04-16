---
title: RabbitMQ 架构与模式
date: 2026-04-14
description: 深入解析 RabbitMQ 的核心架构、Exchange 类型、队列模式与集群机制
tags:
  - rabbitmq
  - message-queue
  - amqp
  - architecture
draft: false
permalink:
---

# RabbitMQ 架构与模式

## 概述

RabbitMQ 是基于 AMQP（Advanced Message Queuing Protocol）协议的开源消息代理，由 Rabbit Technologies 开发，现为 VMware（后被 Broadcom 收购）维护[^1]。

与 Kafka 的**日志模型**不同，RabbitMQ 是典型的**消息代理模型**：

| 特性 | Kafka | RabbitMQ |
|------|-------|----------|
| 核心抽象 | 分布式日志 | 消息代理 |
| 消息保留 | 按时间/大小 | 按消费确认 |
| 消费模式 | Pull | Push |
| 路由能力 | 有限 | 丰富 |
| 典型吞吐量 | 百万级 msg/s | 万级 msg/s |

RabbitMQ 的核心价值在于**灵活的路由能力**和**可靠的消息传递**。

## 核心架构

### AMQP 模型

RabbitMQ 的消息流遵循以下模型：

```
┌─────────────────────────────────────────────────────────────┐
│                    RabbitMQ 消息流                           │
│                                                              │
│  Producer ──▶ Exchange ──▶ Binding ──▶ Queue ──▶ Consumer   │
│                                                              │
│  关键点：                                                     │
│  - 生产者从不直接发送消息到队列                               │
│  - Exchange 负责路由决策                                      │
│  - Binding 规则连接 Exchange 和 Queue                         │
└─────────────────────────────────────────────────────────────┘
```

### 核心组件

#### Exchange（交换机）

Exchange 是 RabbitMQ 的**路由引擎**，接收生产者消息并根据规则路由到一个或多个队列[^2]。

**Exchange 的属性**：
- **Name**：交换机名称
- **Type**：类型（direct、fanout、topic、headers）
- **Durable**：持久化，重启后保留
- **Auto-delete**：所有队列解绑后自动删除

#### Queue（队列）

队列是存储消息的缓冲区，直到被消费者取走。

**队列属性**：
```python
channel.queue_declare(
    queue='tasks',
    durable=True,              # 持久化
    arguments={
        'x-message-ttl': 3600000,        # 消息 TTL
        'x-max-length': 10000,            # 最大消息数
        'x-queue-type': 'quorum'          # 队列类型
    }
)
```

#### Binding（绑定）

Binding 是连接 Exchange 和 Queue 的规则：

```python
# 绑定规则：将 orders.exchange 的消息路由到 orders.queue
channel.queue_bind(
    exchange='orders.exchange',
    queue='orders.queue',
    routing_key='order.created'  # 绑定键
)
```

## Exchange 类型

### 1. Direct Exchange

Direct Exchange 执行**精确匹配**，路由键必须与绑定键完全相同：

```python
# 生产者
channel.basic_publish(
    exchange='orders',
    routing_key='order.created',  # 必须是 'order.created' 才能匹配
    body='{"order_id": 123}'
)

# 声明 Direct Exchange
channel.exchange_declare(
    exchange='orders',
    exchange_type='direct',
    durable=True
)

# 绑定到队列
channel.queue_bind(
    queue='orders.queue',
    exchange='orders',
    routing_key='order.created'
)
```

**适用场景**：
- 点对点消息传递
- 精确路由单一队列

### 2. Fanout Exchange

Fanout Exchange 将消息**广播**到所有绑定的队列，忽略路由键：

```python
# 声明 Fanout Exchange
channel.exchange_declare(
    exchange='notifications',
    exchange_type='fanout',
    durable=True
)

# 绑定多个队列
channel.queue_bind(queue='email.queue', exchange='notifications')
channel.queue_bind(queue='sms.queue', exchange='notifications')
channel.queue_bind(queue='push.queue', exchange='notifications')

# 发布消息，所有队列都会收到
channel.basic_publish(
    exchange='notifications',
    routing_key='',  # fanout 忽略此参数
    body='{"message": "订单已创建"}'
)
```

**适用场景**：
- 通知系统（邮件、短信、推送同时通知）
- 事件广播
- 审计日志

### 3. Topic Exchange

Topic Exchange 支持**通配符匹配**：

| 通配符 | 含义 | 示例 |
|--------|------|------|
| `*` | 匹配一个单词 | `order.*` 匹配 `order.created`、`order.cancelled` |
| `#` | 匹配零个或多个单词 | `order.#` 匹配 `order`、`order.created`、`order.created.us` |

```python
# 绑定规则
channel.queue_bind(queue='payments.processor', exchange='events', routing_key='payment.*')
channel.queue_bind(queue='orders.us', exchange='events', routing_key='order.*.us')
channel.queue_bind(queue='audit.all', exchange='events', routing_key='#')

# 消息路由示例
# 'payment.received' → payments.processor
# 'order.created.us' → orders.us + audit.all
# 'user.registered' → audit.all
```

**适用场景**：
- 复杂路由规则
- 多租户或分级事件系统

### 4. Headers Exchange

Headers Exchange 根据消息头属性进行匹配：

```python
# 声明 Headers Exchange
channel.exchange_declare(
    exchange='headers.events',
    exchange_type='headers',
    durable=True
)

# 绑定时指定 headers
channel.queue_bind(
    queue='specific.queue',
    exchange='headers.events',
    arguments={
        'x-match': 'all',      # 'all' = AND, 'any' = OR
        'format': 'json',
        'version': '2'
    }
)

# 发布带 headers 的消息
channel.basic_publish(
    exchange='headers.events',
    headers={
        'format': 'json',
        'version': '2'
    },
    body='{"data": "message"}'
)
```

**适用场景**：
- 基于消息属性的路由
- 非标准路由需求

## 队列类型

RabbitMQ 4.x 支持三种队列类型[^3]：

### Classic Queue（经典队列）

原始队列类型，灵活但 HA 能力有限：

```python
channel.queue_declare(
    queue='tasks.classic',
    durable=True,
    arguments={'x-queue-type': 'classic'}
)
```

**特性**：
- 可选镜像（已弃用，RabbitMQ 3.13+）
- 内存或磁盘持久化
- 单 leader 架构

### Quorum Queue（仲裁队列）

基于 Raft 共识协议的复制队列（推荐）：

```python
channel.queue_declare(
    queue='tasks.quorum',
    durable=True,
    arguments={
        'x-queue-type': 'quorum',
        'x-quorum-initial-group-size': 3  # 副本数
    }
)
```

**特性**：
- Raft 共识保证强一致性
- 自动故障转移
- 副本均匀分布
- 适合关键业务消息

```
┌─────────────────────────────────────────────────────────────┐
│                   Quorum Queue 复制模型                       │
│                                                              │
│  Leader ──────────────────────────────────────────────────│
│    │                   │                   │              │
│    ▼                   ▼                   ▼              │
│  Follower 1         Follower 2         Follower 3          │
│                                                              │
│  写操作需多数派同意                                          │
└─────────────────────────────────────────────────────────────┘
```

### Stream Queue（流队列）

类似 Kafka 的 Append-only 日志：

```python
channel.queue_declare(
    queue='events.stream',
    durable=True,
    arguments={'x-queue-type': 'stream'}
)
```

**特性**：
- Append-only 结构
- 支持消息回放（Consumer Replay）
- 适合事件溯源、审计日志

## 消息确认机制

### Delivery 模式

| 模式 | QoS | 说明 |
|------|-----|------|
| 自动 ack | 消费即确认 | 消息丢失风险高 |
| 手动 ack | 显式确认 | 可靠传递 |

### 手动确认示例

```python
import pika

connection = pika.BlockingConnection('amqp://guest:guest@localhost:5672/%2F')
channel = connection.channel()

def callback(ch, method, properties, body):
    try:
        process_message(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)  # 确认
    except Exception as e:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)  # 重新入队

channel.basic_qos(prefetch_count=1)  # 每次只预取一条
channel.basic_consume(queue='tasks', on_message_callback=callback)
channel.start_consuming()
```

### 消息拒绝

```python
# 拒绝并重新入队（会重新投递给消费者或下一个消费者）
channel.basic_nack(delivery_tag=delivery_tag, requeue=True)

# 拒绝但不重新入队（会被路由到 Dead Letter Exchange）
channel.basic_nack(delivery_tag=delivery_tag, requeue=False)

# 拒绝（等同于上面的 requeue=False）
channel.basic_reject(delivery_tag=delivery_tag, requeue=False)
```

## 死信队列与重试模式

### Dead Letter Exchange（DLX）

当消息被拒绝且不重新入队时，会被发送到 DLX：

```python
# 设置 DLX
channel.exchange_declare(exchange='dlx', exchange_type='direct')
channel.queue_declare(queue='dead.letters')
channel.queue_bind(queue='dead.letters', exchange='dlx', routing_key='failed')

# 在主队列上配置 DLX
channel.queue_declare(
    queue='tasks',
    arguments={
        'x-dead-letter-exchange': 'dlx',
        'x-dead-letter-routing-key': 'failed'
    }
)
```

### 重试模式

```python
def process_with_retry(channel, method, properties, body, max_retries=3):
    """带重试的消息处理"""
    headers = properties.headers or {}
    retry_count = headers.get('x-retry-count', 0)
    
    try:
        process_message(body)
        channel.basic_ack(delivery_tag=method.delivery_tag)
    except TemporaryError:
        # 临时错误，重新入队等待重试
        if retry_count < max_retries:
            # 增加重试计数后重新发布
            properties.headers['x-retry-count'] = retry_count + 1
            channel.basic_publish(
                exchange='',
                routing_key='tasks',
                body=body,
                properties=properties
            )
            channel.basic_ack(delivery_tag=method.delivery_tag)
        else:
            # 超过最大重试次数，进入 DLQ
            channel.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
    except Exception:
        # 永久错误，直接拒绝
        channel.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
```

## 集群与分布式

### 集群架构

```python
# 集群配置示例（rabbitmq.conf）
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
cluster_formation.classic_config.nodes.1 = rabbit@node1
cluster_formation.classic_config.nodes.2 = rabbit@node2
cluster_formation.classic_config.nodes.3 = rabbit@node3
```

**集群特性**：
- 节点间自动同步：用户、虚拟主机、Exchange、Binding
- 队列默认只存在于单一节点（需配置镜像）
- 客户端可连接任意节点访问所有队列

### Federation vs Shovel

| 特性 | Federation | Shovel |
|------|------------|--------|
| 用途 | 跨集群/机房的消息同步 | 单向消息迁移 |
| 协议 | AMQP | AMQP |
| 拓扑 | 双向可选 | 单向 |
| 适用 | 多站点部署 | 数据迁移 |

```python
# Federation 配置示例
rabbitmqctl set federation-upstream my-upstream \
  --uri=amqp://user:pass@remote-host:5672

# 定义 federation 策略
rabbitmqctl set federation-policy my-policy \
  --priority=1 \
  --apply-to=exchanges \
  --definition='{"upstream": "my-upstream"}'
```

## RabbitMQ 4.x 新特性

RabbitMQ 4.0（2024年9月发布）带来重大更新[^3]：

### Native AMQP 1.0 支持

AMQP 1.0 成为核心协议，不再需要插件：

```python
# 4.x 原生支持 AMQP 1.0
connection = pika.ConnectionParameters(
    protocol=pika.spec.AMQP_1_0
)
```

### Khepri 元数据存储

新的 Raft-based 元数据存储，替代 Mnesia：

- 更快的集群启动
- 改进的分区处理
- 简化的运维

### 性能优化

- 改进的消息路由性能
- 更低的内存占用
- 更好的多核利用率

## 命名规范

良好的命名规范便于运维和调试：

```text
# Exchange 命名
{domain}.{type}
  orders.direct
  events.topic
  notifications.fanout

# Queue 命名
{domain}.{function}.{optional-modifier}
  orders.processing
  orders.processing.high-priority
  payments.completed

# Routing Key 命名
{entity}.{action}.{optional-context}
  order.created
  order.shipped.us
  payment.received.stripe
```

## 相关主题

- [[./kafka-internals|Apache Kafka]] — Kafka 的架构与机制
- [[./pulsar-vs-kafka|消息队列对比]] — Kafka、Pulsar、RabbitMQ 的选型指南
- [[../distributed-systems/event-driven-architecture|事件驱动架构]] — 消息队列的应用模式

---

## 参考资料

[^1]: [RabbitMQ 官方文档](https://www.rabbitmq.com/docs)

[^2]: [AMQP 0-9-1 Model Explained - RabbitMQ](https://www.rabbitmq.com/tutorials/amqp-concepts.html)

[^3]: [RabbitMQ 4.x Release Notes](https://www.rabbitmq.com/news.html#2024-09-04)
