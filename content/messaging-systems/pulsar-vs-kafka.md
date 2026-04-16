---
title: Pulsar 与 Kafka 对比
date: 2026-04-14
description: Apache Pulsar 与 Kafka 的架构差异、适用场景与选型指南
tags:
  - kafka
  - pulsar
  - message-queue
  - comparison
  - architecture
draft: false
permalink:
---

# Pulsar 与 Kafka 对比

## 概述

Apache Kafka 和 Apache Pulsar 都是主流的分布式消息流平台，但它们采用了根本不同的架构哲学[^1][^2]。

| 维度 | Apache Kafka | Apache Pulsar |
|------|--------------|---------------|
| 架构模型 | 计算存储耦合 | 计算存储分离 |
| 存储层 | Broker 本地磁盘 | Apache BookKeeper |
| 协调组件 | KRaft/ZooKeeper | ZooKeeper + BookKeeper |
| 多租户 | 有限 | 原生支持 |
| 消息模型 | Pub/Sub | Pub/Sub + Queue |

## 架构差异

### Kafka 架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Kafka 架构                                 │
│                                                              │
│  Producers ────────────────────────────────────────────────│
│                                                              │
│  ┌─────────┐      ┌─────────┐      ┌─────────┐           │
│  │ Broker  │      │ Broker  │      │ Broker  │           │
│  │  (计算+  │      │  (计算+  │      │  (计算+  │           │
│  │   存储)  │      │   存储)  │      │   存储)  │           │
│  └────┬────┘      └────┬────┘      └────┬────┘           │
│       │                │                │                  │
│       └────────────────┴────────────────┘                  │
│                      │                                   │
│                  Kafka Topic                             │
│              (Partition on Disk)                          │
│                                                              │
│  Consumers ────────────────────────────────────────────────│
│                                                              │
│  ┌──────────────────────────────────────────────────┐     │
│  │              ZooKeeper / KRaft                     │     │
│  │           (元数据协调)                            │     │
│  └──────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

**特点**：
- 每个 Broker 同时负责数据服务和数据存储
- 扩展时需要重新均衡分区
- Partition 作为数据分布的基本单位

### Pulsar 架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Pulsar 架构                               │
│                                                              │
│  Producers ────────────────────────────────────────────────│
│                                                              │
│  ┌─────────┐      ┌─────────┐      ┌─────────┐           │
│  │  Broker  │      │  Broker  │      │  Broker  │           │
│  │  (仅计算) │      │  (仅计算) │      │  (仅计算) │           │
│  │  无状态   │      │  无状态   │      │  无状态   │           │
│  └────┬────┘      └────┬────┘      └────┬────┘           │
│       │                │                │                  │
│       └────────────────┴────────────────┘                  │
│                      │                                   │
│                      ▼                                   │
│  ┌──────────────────────────────────────────────────┐     │
│  │              BookKeeper (Bookies)                 │     │
│  │         (分布式存储层 - Segments)                │     │
│  └──────────────────────────────────────────────────┘     │
│                                                              │
│  ┌──────────────────────────────────────────────────┐     │
│  │              ZooKeeper                            │     │
│  │           (元数据协调)                            │     │
│  └──────────────────────────────────────────────────┘     │
│                                                              │
│  Consumers ────────────────────────────────────────────────│
└─────────────────────────────────────────────────────────────┘
```

**特点**：
- Broker 无状态，仅负责请求路由
- 数据存储委托给 BookKeeper
- 计算和存储独立扩展

### 核心架构哲学差异

| 维度 | Kafka | Pulsar |
|------|-------|--------|
| 核心理念 | 简洁、统一的架构 | 分离关注点 |
| 扩展方式 | 计算存储联动扩展 | 独立扩展 |
| 故障影响 | Broker 故障影响数据和请求 | 仅影响请求路由 |
| 运维复杂度 | 较低 | 较高（多组件） |

## 存储机制对比

### Kafka 分区存储

```
┌─────────────────────────────────────────────────────────────┐
│                  Kafka Partition                              │
│                                                              │
│  Broker Disk:                                               │
│  ┌─────────────────────────────────────────────────┐       │
│  │ Partition-0                                      │       │
│  │ [Segment 1] [Segment 2] [Segment 3]              │       │
│  │  (本地磁盘)                                      │       │
│  └─────────────────────────────────────────────────┘       │
│                                                              │
│  特点：分区作为连续日志段存储在单 Broker                    │
└─────────────────────────────────────────────────────────────┘
```

### Pulsar Segment 存储

```
┌─────────────────────────────────────────────────────────────┐
│                  Pulsar Tiered Storage                       │
│                                                              │
│  ┌─────────────────────────────────────────────────┐       │
│  │ Partition                                         │       │
│  │  ├─ Segment 1 → Bookie-1, Bookie-3            │       │
│  │  ├─ Segment 2 → Bookie-2, Bookie-4            │       │
│  │  └─ Segment 3 → Bookie-1, Bookie-5            │       │
│  └─────────────────────────────────────────────────┘       │
│                                                              │
│  特点：Segment 分布式存储在多个 Bookie                     │
└─────────────────────────────────────────────────────────────┘
```

### BookKeeper 简介

Apache BookKeeper 是分布式日志存储系统，提供：

- **高持久性**：多副本、强一致
- **IO 隔离**：不同 Segment 可写入不同磁盘
- **弹性扩展**：添加 Bookie 自动均衡

```yaml
# BookKeeper 配置示例
bookies: 4
zkServers: "zookeeper:2181"
journalDirectory: "/mnt/journal"
ledgerDirectories: "/data/ledger1,/data/ledger2"
```

## 多租户能力

### Kafka 多租户（有限）

```properties
# Kafka ACL 配置实现多租户
allow.everyone.if.no.acl.found=false
super.users=User:admin

# Topic 级别配额
quota.producer.default=10MB
quota.consumer.default=50MB
```

**限制**：
- 主要依赖 ACL 和命名约定
- 通常需要为每个租户创建独立集群

### Pulsar 原生多租户

```
┌─────────────────────────────────────────────────────────────┐
│                   Pulsar 多租户架构                          │
│                                                              │
│  ┌──────────────────────────────────────────────────┐       │
│  │                    Tenant: ecommerce            │       │
│  │  ┌────────────────────────────────────────────┐ │       │
│  │  │  Namespace: orders                         │ │       │
│  │  │  ├─ Topic: orders.created                 │ │       │
│  │  │  ├─ Topic: orders.shipped                │ │       │
│  │  │  └─ Topic: orders.completed              │ │       │
│  │  └────────────────────────────────────────────┘ │       │
│  │  ┌────────────────────────────────────────────┐ │       │
│  │  │  Namespace: inventory                     │ │       │
│  │  │  └─ Topic: stock.update                  │ │       │
│  │  └────────────────────────────────────────────┘ │       │
│  └──────────────────────────────────────────────────┘       │
│                                                              │
│  ┌──────────────────────────────────────────────────┐       │
│  │                  Tenant: finance                 │       │
│  │  ...                                              │       │
│  └──────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

**Tenant/Namespace 特性**：
- 资源隔离：CPU、带宽、存储配额
- 独立认证授权
- 灵活的消息保留策略

```java
// Pulsar 多租户配置
admin.namespaces().setNamespaceResourceQuota(
    "ecommerce/orders",
    new ResourceQuota()
        .setMsgRateIn(10000)
        .setMsgRateOut(20000)
        .setBandwidthIn(100000000)
        .setBandwidthOut(200000000)
);
```

## 消息模型

### Kafka 消息模型

```
Topic: orders (6 partitions)
┌─────┬─────┬─────┬─────┬─────┬─────┐
│ P0  │ P1  │ P2  │ P3  │ P4  │ P5  │
└──┬──┴──┬──┴──┬──┴──┬──┴──┬──┴──┬──┘
   │     │     │     │     │     │
 Consumer Group A (每个 Partition 一个 Consumer)
```

- **消费模式**：基于 Partition 的负载均衡
- **有序性**：Partition 内有序
- **Replay**：通过 Offset 支持

### Pulsar 订阅模型

Pulsar 支持四种订阅模式[^3]：

```java
// 1. Exclusive（独占）- 仅一个消费者
admin.topics().createSubscription("orders", "sub-1", SubscriptionMode.Exclusive);

// 2. Failover（故障转移）- 主备消费者
admin.topics().createSubscription("orders", "sub-2", SubscriptionMode.Failover);

// 3. Shared（共享）- 多个消费者，消息轮询
admin.topics().createSubscription("orders", "sub-3", SubscriptionMode.Shared);

// 4. Key_Shared（键共享）- 相同键的消息发给同一消费者
admin.topics().createSubscription("orders", "sub-4", SubscriptionMode.Key_Shared);
```

| 订阅模式 | 特性 | 有序性 | 适用场景 |
|----------|------|--------|----------|
| Exclusive | 单一消费者 | 是 | 顺序处理 |
| Failover | 主备切换 | 是 | 高可用 |
| Shared | 轮询分发 | 否 | 负载均衡 |
| Key_Shared | 键哈希 | 是 | 按用户处理 |

## 性能对比

### 吞吐量

| 指标 | Kafka | Pulsar |
|------|-------|--------|
| 写入吞吐 | 极高 | 高 |
| 读取吞吐 | 极高 | 高 |
| 扩展性 | 好 | 极好 |

### 延迟

| 指标 | Kafka | Pulsar |
|------|-------|--------|
| P50 延迟 | ~1ms | ~2ms |
| P99 延迟 | ~5ms | ~10ms |
| P999 延迟 | 差异大 | 更稳定 |

**Pulsar 延迟更稳定的原因**：
- IO 隔离：不同 Topic 可写入不同磁盘
- 无分区重均衡延迟

### 扩展性对比

```
┌─────────────────────────────────────────────────────────────┐
│                    扩展性对比                                │
│                                                              │
│  Kafka:                                                     │
│  - 添加 Broker → 需要重均衡 Partition                       │
│  - 重均衡期间：IO 密集、延迟抖动                            │
│  - 时间：数小时（取决于数据量）                             │
│                                                              │
│  Pulsar:                                                    │
│  - 添加 Broker：立即接管新请求                              │
│  - 无数据迁移延迟                                           │
│  - Bookie 自动均衡                                          │
└─────────────────────────────────────────────────────────────┘
```

## Geo-replication

### Kafka MirrorMaker 2

```yaml
# Kafka Connect MirrorMaker 2 配置
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaMirrorMaker2
metadata:
  name: my-mirror-maker
spec:
  mirrors:
    - sourceCluster: "primary"
      targetCluster: "secondary"
      sourceConnector:
        tasksMax: 10
        config:
          replication.factor: 3
      topicsPattern: ".*"
      groupsPattern: ".*"
```

**特点**：
- 外部工具，需单独部署
- 配置复杂
- 无自动故障转移

### Pulsar 内置 Geo-replication

```java
// 启用跨集群复制
admin.tenants().create(
    "ecommerce",
    new TenantInfo()
        .setAllowedClusters(List.of("us-east", "us-west", "eu"))
);

// 创建跨集群 Topic
admin.topics().createNonPartitionedTopic(
    "persistent://ecommerce/orders/global-notifications"
);

// 自动复制，无需额外配置
```

**特点**：
- 内置于 Pulsar
- Namespace 级别配置
- 自动同步
- 延迟复制/同步复制可选

## 选型指南

### 选择 Kafka 的场景

| 场景 | 原因 |
|------|------|
| 超高吞吐量（100K+ msg/s） | 生态最成熟，优化充分 |
| 简单消费者组模型 | Partition + Consumer Group 足够 |
| 成熟生态需求 | Kafka Connect、Streams、ksqlDB |
| 事件流平台 | Event Streaming 场景首选 |

### 选择 Pulsar 的场景

| 场景 | 原因 |
|------|------|
| 多租户 SaaS | 原生 Namespace/租户隔离 |
| 强多租户隔离需求 | 配额、限流内置 |
| 独立扩展计算/存储 | BookKeeper 分离架构 |
| 内置 Geo-replication | 无需额外组件 |
| 混合消息+流场景 | Queue + Pub/Sub 一体化 |

### 不适合两者的场景

| 场景 | 替代方案 |
|------|----------|
| 低延迟 RPC | gRPC、NATS |
| 简单任务队列 | Redis Queue、Sidekiq |
| 极高消息频率 | 定制协议 |

## 迁移考虑

### Kafka → Pulsar

```python
# 使用 Pulsar Kafka Sink/Source
admin.topics().create(
    "persistent://public/default/migrated-topic",
    # 配置从 Kafka 消费
)

# 或使用 StreamNative Offloader
```

### Pulsar → Kafka

```python
# 使用标准 Kafka Consumer
consumer = KafkaConsumer(
    'migrated-topic',
    bootstrap_servers=['pulsar-broker:9092'],
    # Pulsar 兼容 Kafka 协议
)
```

## 相关主题

- [[./kafka-internals|Apache Kafka]] — Kafka 深入解析
- [[./rabbitmq-architecture|RabbitMQ 架构]] — RabbitMQ 详解
- [[../distributed-systems/event-driven-architecture|事件驱动架构]] — 消息队列应用模式

---

## 参考资料

[^1]: [Kafka vs Pulsar: A Modern Messaging Deep Dive - sanj.dev](https://sanj.dev/post/pulsar-vs-kafka-deep-dive)

[^2]: [Kafka vs Pulsar: Streaming Platform Comparison - OneUptime](https://oneuptime.com/blog/post/2026-01-21-kafka-vs-pulsar/view)

[^3]: [Apache Pulsar Documentation](https://pulsar.apache.org/docs/)
