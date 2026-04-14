---
title: 消息队列系统
date: 2026-04-08
description: 消息队列核心概念：RabbitMQ、Kafka、ActiveMQ对比，队列模型、可靠性保证
tags:
  - message-queue
  - distributed-systems
  - middleware
  - rabbitmq
  - kafka
draft: false
permalink:
---

## 概述

消息队列（Message Queue）是一种进程间通信方式，用于解决异步处理、削峰填谷、解耦等问题。[^1]

### 应用场景

| 场景 | 说明 | 示例 |
|------|------|------|
| 异步处理 | 非核心流程异步执行 | 订单完成后发邮件 |
| 流量削峰 | 缓解突发流量 | 秒杀系统 |
| 日志处理 | 异步收集分析 | ELK日志系统 |
| 事件驱动 | 系统间事件通知 | 分布式事务 |

---

## 核心概念

### 消息模型

```
发布者 ──▶ 交换机 ──▶ 队列 ──▶ 消费者
           (Exchange)  (Queue)  (Consumer)
```

**流程**：
1. 生产者将消息发送到Exchange
2. Exchange根据路由规则将消息分发到Queue
3. 消费者从Queue获取消息并处理

### JMS模型

Java Message Service (JMS) 定义了两种消息模型：

| 模型 | 说明 | 特点 |
|------|------|------|
| Point-to-Point (P2P) | 点对点，一个消息只能被一个消费者消费 | 队列、持久化 |
| Publish/Subscribe (Pub/Sub) | 发布订阅，消息被所有订阅者接收 | Topic、持久化 |

### 可靠性保证

| 级别 | 说明 | 可靠性 |
|------|------|--------|
| At most once | 最多一次，可能丢消息 | 低 |
| At least once | 至少一次，可能重复 | 中 |
| Exactly once | 精确一次 | 高 |

---

## RabbitMQ

### 核心组件

| 组件 | 说明 |
|------|------|
| Broker | RabbitMQ服务器实例 |
| Virtual Host | 隔离的AMQP环境 |
| Exchange | 消息路由器 |
| Queue | 消息存储队列 |
| Binding | Exchange与Queue的绑定关系 |
| Routing Key | 路由关键字 |

### Exchange类型

| 类型 | 路由规则 |
|------|---------|
| Direct | 完全匹配Routing Key |
| Fanout | 广播到所有绑定队列 |
| Topic | 通配符匹配（* 或 #） |
| Headers | 根据消息头匹配 |

### RabbitMQ配置

```python
import pika

# 连接
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# 声明交换机
channel.exchange_declare(exchange='orders', exchange_type='direct', durable=True)

# 声明队列
channel.queue_declare(queue='order_queue', durable=True)

# 绑定
channel.queue_bind(exchange='orders', queue='order_queue', routing_key='new_order')

# 发送消息
channel.basic_publish(
    exchange='orders',
    routing_key='new_order',
    body='Order data...',
    properties=pika.BasicProperties(delivery_mode=2)  # 持久化
)

# 消费消息
def callback(ch, method, properties, body):
    print(f"Received: {body}")
    ch.basic_ack(delivery_tag=method.delivery_tag)  # 确认

channel.basic_consume(queue='order_queue', on_message_callback=callback)
channel.start_consuming()
```

### 消息确认机制

```python
# 手动确认
channel.basic_ack(delivery_tag=method.delivery_tag)

# 否定确认（重新入队）
channel.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

# 设置预取数量
channel.basic_qos(prefetch_count=10)  # 最多10条未确认
```

---

## Kafka

### 核心概念

| 概念 | 说明 |
|------|------|
| Topic | 消息主题 |
| Partition | 分区，实现并行处理 |
| Replica | 分区副本，保证高可用 |
| Offset | 消费者在分区中的位置 |
| Producer | 消息生产者 |
| Consumer | 消息消费者 |

### Kafka架构

```
Topic: orders
┌─────────────────────────────────────────────┐
│ Partition 0  │ Partition 1  │ Partition 2  │
│ [0,1,2,3,4]  │ [0,1,2,3]   │ [0,1,2,3,4,5]│
└─────────────────────────────────────────────┘
     ↓副本复制           ↓副本复制
Leader → Follower    Leader → Follower
```

### Kafka生产者

```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# 发送消息
future = producer.send('orders', value={'order_id': 123, 'amount': 100})
future.get(timeout=10)  # 同步等待

# 异步发送
producer.send('orders', value={'order_id': 124, 'amount': 200})
producer.flush()  # 确保所有消息发送完成
```

### Kafka消费者

```python
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['localhost:9092'],
    group_id='order_processor',  # 消费者组
    auto_offset_reset='earliest',  # 从最早开始消费
    enable_auto_commit=True  # 自动提交offset
)

for message in consumer:
    print(f"Topic: {message.topic}, Partition: {message.partition}, "
          f"Offset: {message.offset}, Value: {message.value}")
```

### 分区策略

```python
# 指定分区
producer.send('orders', value=data, partition=0)

# 手动指定分区键
producer.send('orders', value=data, key=b'user_123')

# 自定义分区器
from kafka.processor import Partitioner

class UserPartitioner(Partitioner):
    def partition(self, key, num_partitions):
        return int(key) % num_partitions
```

---

## 三大消息队列对比

| 特性 | RabbitMQ | Kafka | ActiveMQ |
|------|----------|-------|----------|
| **定位** | 企业级消息代理 | 分布式流平台 | 传统消息中间件 |
| **吞吐量** | 中等(~10万/秒) | 高(百万/秒) | 低 |
| **延迟** | 低(μs级) | 低(ms级) | 中 |
| **消息持久化** | 支持 | 支持 | 支持 |
| **事务** | 支持 | 支持 | 支持 |
| **优先级队列** | 支持 | 不支持 | 支持 |
| **延迟队列** | 插件支持 | 不支持 | 支持 |
| **集群** | 支持 | 原生支持 | 支持 |

### 选型建议

| 场景 | 推荐 |
|------|------|
| 电商订单处理 | RabbitMQ（事务支持好） |
| 日志收集分析 | Kafka（高吞吐） |
| 小型系统 | ActiveMQ（简单） |

---

## 可靠性实现

### 消息持久化

```python
# RabbitMQ持久化
channel.queue_declare(queue='persistent_queue', durable=True)
channel.basic_publish(
    body=data,
    properties=pika.BasicProperties(delivery_mode=2)  # 持久化
)

# Kafka持久化（默认）
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    acks='all'  # 等待所有副本确认
)
```

### 消费幂等

```python
# 方式1：数据库唯一键
def process_order(order):
    try:
        db.insert('orders', order)  # 唯一键冲突则忽略
    except DuplicateKeyError:
        pass  # 重复订单，忽略

# 方式2：Redis去重
def process_order(order):
    if redis.setnx(f'order:{order.id}', '1', ex=86400):
        db.process(order)
```

### 事务消息

```python
# Kafka事务（Exactly Once语义）
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    transactional_id='order_producer'  # 开启事务
)

producer.init_transactions()
producer.begin_transaction()

try:
    producer.send('orders', value=order_data)
    producer.send('inventory', value=inventory_data)
    producer.commit_transaction()
except Exception:
    producer.abort_transaction()
```

---

## 高可用设计

### RabbitMQ镜像队列

```bash
# 镜像队列策略
rabbitmqctl set_policy ha-two "^orders\." '{"ha-mode":"exactly","ha-params":2}'
```

### Kafka分区副本

```python
# Topic配置
topic_config = {
    'topic': 'orders',
    'num_partitions': 6,
    'replication_factor': 3,  # 3副本
    'config': {
        'retention.ms': '604800000',  # 保留7天
        'min.insync.replicas': '2'    # 最少2个副本确认
    }
}
```

### 消费端负载均衡

```
消费者组 A: [Consumer1, Consumer2, Consumer3]
                    ↓ 分区分配
Topic: orders [P0, P1, P2, P3, P4, P5]
                      ↓
              Consumer1: P0, P3
              Consumer2: P1, P4
              Consumer3: P2, P5
```

---

## 参考资料

[^1]: 本词条综合自 [RabbitMQ Tutorials](https://www.rabbitmq.com/getstarted.html) 和 [Kafka Documentation](https://kafka.apache.org/documentation/)

---

## 相关主题

- [[distributed-systems/distributed-systems-overview|分布式系统基础]] - 分布式架构理论
- [[distributed-systems/distributed-transactions|分布式事务]] - 2PC、TCC
- [[software-engineering/system-design|系统设计]] - 消息队列在系统设计中的应用
