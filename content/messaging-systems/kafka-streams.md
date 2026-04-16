---
title: Kafka Streams 实时处理
date: 2026-04-14
description: 深入解析 Kafka Streams 的架构、状态管理、窗口操作与容错机制
tags:
  - kafka
  - kafka-streams
  - stream-processing
  - real-time
  - distributed-systems
draft: false
permalink:
---

# Kafka Streams 实时处理

## 概述

Kafka Streams 是构建在 Kafka 生产者和消费者之上的客户端库，用于开发实时流处理应用程序[^1]。

与其他流处理框架（如 Apache Flink、Apache Spark Streaming）不同，Kafka Streams **以库的形式运行**，无需独立的集群基础设施。

```
┌─────────────────────────────────────────────────────────────┐
│                  Kafka Streams 定位                           │
│                                                              │
│  ┌──────────────────────────────────────────────────┐       │
│  │              独立流处理框架                        │       │
│  │         Flink / Spark Streaming / Storm            │       │
│  │         (需独立集群运行)                          │       │
│  └──────────────────────────────────────────────────┘       │
│                                                              │
│  ┌──────────────────────────────────────────────────┐       │
│  │              嵌入式流处理库                        │       │
│  │              Kafka Streams                          │       │
│  │         (运行在你的应用中)                          │       │
│  └──────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

**核心优势**：
- 无需独立集群，降低运维复杂度
- Exactly-Once 语义内置
- 与 Kafka 深度集成
- 支持交互式查询状态

## 核心概念

### Stream 与 Table

```
┌─────────────────────────────────────────────────────────────┐
│                    KStream vs KTable                         │
│                                                              │
│  KStream ( unbounded sequence of events)                    │
│  ┌─────┬─────┬─────┬─────┬─────┬─────┐                   │
│  │ e1  │ e2  │ e3  │ e4  │ e5  │ e6  │  →  time         │
│  └─────┴─────┴─────┴─────┴─────┴─────┘                   │
│  每个事件都是新的 fact，不可变                             │
│                                                              │
│  KTable ( changelog stream / materialized view)            │
│  ┌─────┬─────┬─────┬─────┬─────┬─────┐                   │
│  │ e1' │     │ e3' │     │ e5' │     │  →  time         │
│  └─────┴─────┴─────┴─────┴─────┴─────┘                   │
│  同一 Key 的新值覆盖旧值，表示最新状态                      │
└─────────────────────────────────────────────────────────────┘
```

**KStream**：
- 表示无界的事件流
- 每条记录都是新的 fact
- 适合：点击事件、传感器读数、交易记录

**KTable**：
- 表示 changelog 流
- 同一 Key 的新记录覆盖旧记录
- 适合：用户配置、账户余额、库存状态

### 领域特定语言（DSL）

Kafka Streams 提供高级 DSL，简化常见操作：

```java
// Java DSL 示例
StreamsBuilder builder = new StreamsBuilder();

// 1. 创建源 Stream
KStream<String, Order> orders = builder.stream("orders");

// 2. 过滤
KStream<String, Order> highValueOrders = orders
    .filter((key, order) -> order.getAmount() > 1000);

// 3. 聚合
KTable<String, Double> revenueByRegion = orders
    .groupBy((key, order) -> KeyValue.pair(order.getRegion(), order.getAmount()))
    .reduce(Double::sum);

// 4. 输出到 Sink
revenueByRegion.toStream().to("revenue-by-region");
```

## 架构原理

### Topology（拓扑）

应用的处理逻辑被建模为有向无环图（DAG）：

```
┌─────────────────────────────────────────────────────────────┐
│                    Kafka Streams Topology                     │
│                                                              │
│  Source Processor (orders)                                   │
│         │                                                    │
│         ▼                                                    │
│  ┌─────────────┐    ┌─────────────┐                         │
│  │   Filter    │───▶│    Map      │                         │
│  │ (amount>100)│    │ (extract)   │                         │
│  └─────────────┘    └──────┬──────┘                         │
│                            │                                 │
│                            ▼                                 │
│                     ┌─────────────┐                         │
│                     │  GroupBy    │                         │
│                     │  (region)   │                         │
│                     └──────┬──────┘                         │
│                            │                                 │
│                            ▼                                 │
│                     ┌─────────────┐                         │
│                     │  Aggregate  │                         │
│                     │  (sum)      │───▶ State Store         │
│                     └──────┬──────┘                         │
│                            │                                 │
│                            ▼                                 │
│                     Sink Processor (revenue)                │
└─────────────────────────────────────────────────────────────┘
```

### Task 与 Partition

Kafka Streams 的并行度基于 Kafka Topic Partition：

```java
// 输入 Topic: orders (4 partitions)
// 最大并行度 = 4

Topology topology = builder.build();

// Streams 自动创建 Task
// Task 0 → Partition 0
// Task 1 → Partition 1
// Task 2 → Partition 2
// Task 3 → Partition 3
```

**Task 的特性**：
- Task 是 Kafka Streams 的基本并行单位
- 每个 Task 处理一个 Partition
- Task 数量在运行时固定（由 Partition 数决定）

### 线程模型

```java
// 配置多个 Stream Thread
Properties props = new Properties();
props.put(StreamsConfig.NUM_STREAM_THREADS_CONFIG, 4);  // 4 个线程

KafkaStreams streams = new KafkaStreams(topology, props);
streams.start();
```

```
┌─────────────────────────────────────────────────────────────┐
│                    Kafka Streams 线程模型                      │
│                                                              │
│  ┌──────────────────────────────────────────────────┐         │
│  │           KafkaStreams 实例                       │         │
│  │                                                   │         │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐        │         │
│  │  │ Thread 1│  │ Thread 2│  │ Thread 3│  ...   │         │
│  │  │ ┌─────┐ │  │ ┌─────┐ │  │ ┌─────┐ │        │         │
│  │  │ │Task0│ │  │ │Task1│ │  │ │Task2│ │        │         │
│  │  │ └─────┘ │  │ └─────┘ │  │ └─────┘ │        │         │
│  │  │ ┌─────┐ │  │ ┌─────┐ │  │ ┌─────┐ │        │         │
│  │  │ │Task3│ │  │ │Task4│ │  │ │Task5│ │        │         │
│  │  │ └─────┘ │  │ └─────┘ │  │ └─────┘ │        │         │
│  │  └─────────┘  └─────────┘  └─────────┘        │         │
│  └──────────────────────────────────────────────────┘         │
│                                                              │
│  每个 Task 独立处理，互不干扰                                │
└─────────────────────────────────────────────────────────────┘
```

## 状态管理

### State Store

状态存储用于在流处理中维护状态：

```java
// 创建状态存储
StoreBuilder<KeyValueStore<String, Double>> storeBuilder =
    Stores.keyValueStore(
        Stores.inMemoryKeyValueStore("revenue-store")
    );

// 或持久化存储（RocksDB）
StoreBuilder<KeyValueStore<String, Double>> persistentStore =
    Stores.keyValueStore(
        Stores.persistentKeyValueStore("revenue-store")
    );

// 注册状态存储
builder.addStateStore(persistentStore);
```

### RocksDB 存储

Kafka Streams 默认使用 RocksDB 作为持久化状态后端：

```java
// RocksDB 配置
StoreBuilder<KeyValueStore<String, Long>> countStore =
    Stores.keyValueStoreBuilder(
        Stores.persistentKeyValueStore("click-counts"),
        Serdes.String(),
        Serdes.Long()
    )
    .withCachingEnabled(true)          // 开启缓存
    .withLoggingEnabled(true);        // 开启 changelog
```

**RocksDB 特性**：
- 嵌入式的 KV 存储
- 本地持久化，性能优异
- 支持大状态存储

### 交互式查询

Kafka Streams 支持从外部应用查询状态：

```java
// 启用交互式查询
props.put(StreamsConfig.APPLICATION_SERVER_CONFIG, "host:9092");

// 在处理器中使用状态存储
@ProcesssiInput
public class RegionalAggregator {

    private ProcessorContext context;
    private StateStore revenueStore;

    @Override
    public void init(ProcessorContext context) {
        this.context = context;
        this.revenueStore = context.getStateStore("revenue-store");
    }

    @Override
    public void process(String region, Double amount) {
        Double current = revenueStore.get(region);
        revenueStore.put(region, current == null ? amount : current + amount);
    }

    // REST API 暴露状态
    @GetMapping("/revenue/{region}")
    public Double getRevenue(@PathVariable String region) {
        return (Double) revenueStore.get(region);
    }
}
```

## 窗口操作

### 窗口类型

```
┌─────────────────────────────────────────────────────────────┐
│                    窗口类型对比                               │
│                                                              │
│  Tumbling Window (固定大小，不重叠)                         │
│  ├───────┤├───────┤├───────┤                               │
│  0-5min │5-10min│10-15min│                                │
│                                                              │
│  Hopping Window (固定大小，可重叠)                          │
│  ├─────┤  ├─────┤  ├─────┤                                  │
│  0-5min   5-10min  10-15min                                 │
│  advance: 5min                                                │
│                                                              │
│  Sliding Window (基于记录对齐)                               │
│  ┌─────────────────────────────────┐                        │
│  │      ┌───┐                      │                        │
│  │      │   │ ← 窗口随数据滑动                             │
│  └──────┴───┴──────────────────────┘                        │
│                                                              │
│  Session Window (基于活动会话)                               │
│  ┌──┐    ┌────┐  ┌──┐                                       │
│  │  │    │    │  │  │  ← 用户会话                          │
│  └──┘    └────┘  └──┘                                       │
└─────────────────────────────────────────────────────────────┘
```

### 时间语义

| 时间类型 | 说明 | 场景 |
|----------|------|------|
| Event Time | 事件发生时间 | 业务分析 |
| Processing Time | 处理时间 | 监控告警 |
| Ingestion Time | 消息入 Kafka 时间 | 审计日志 |

```java
// 配置时间语义
props.put(StreamsConfig.TIMESTAMP_EXTRACTOR_CLASS_CONFIG, 
    WallclockTimestampExtractor.class.getName());

// 或使用事件时间
props.put(StreamsConfig.TIMESTAMP_EXTRACTOR_CLASS_CONFIG,
    "my.custom.TimestampExtractor");
```

### 窗口聚合示例

```java
// 5分钟滚动窗口计算营收
KStream<String, Order> orders = builder.stream("orders");

KTable<Windowed<String>, Double> revenuePerRegion = orders
    .groupBy((key, order) -> order.getRegion())
    .windowedBy(TimeWindows.ofSizeAndGrace(Duration.ofMinutes(5), Duration.ofMinutes(1)))
    .reduce(
        Double::sum,
        Materialized.<String, Double, WindowStore<bytes, byte[]>>as("revenue-per-region")
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Double())
    );

// 输出窗口结果
revenuePerRegion.toStream()
    .map((windowedRegion, revenue) -> 
        KeyValue.pair(windowedRegion.key(), 
            String.format("%s: $%.2f", windowedRegion.key(), revenue)))
    .to("revenue-output");
```

## 容错机制

### Exactly-Once 语义

Kafka Streams 通过 Kafka 事务实现 Exactly-Once：

```java
// 启用 Exactly-Once
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, 
    StreamsConfig.EXACTLY_ONCE_V2);

// 等同于配置
// - enable.idempotence = true
// - acks = all
// - transactional.id = ${application.id}-${thread.id}
```

### Changelog Topic

状态变更被持久化到 Changelog Topic：

```
┌─────────────────────────────────────────────────────────────┐
│                    状态恢复机制                               │
│                                                              │
│  正常处理:                                                   │
│  ┌────────┐      ┌────────────┐      ┌────────────┐        │
│  │ Input  │ ───▶ │  Processor │ ───▶ │ State Store│       │
│  └────────┘      └────────────┘      └─────┬──────┘        │
│                                           │                 │
│                                           ▼                 │
│                                    ┌────────────┐           │
│                                    │ Changelog │           │
│                                    └────────────┘           │
│                                                              │
│  故障恢复:                                                   │
│  ┌────────────┐      ┌────────────┐      ┌────────────┐    │
│  │ Changelog  │ ───▶ │  Rebuilt   │ ───▶ │State Store │    │
│  └────────────┘      └────────────┘      └────────────┘    │
│  Processor 从 Changelog 重放状态                             │
└─────────────────────────────────────────────────────────────┘
```

### 故障恢复流程

```java
// 故障检测与自动恢复
// 1. Task 失败 → Kafka Streams 检测
// 2. 重新分配 Partition 给其他实例
// 3. 新实例从 Changelog 重建状态
// 4. 继续处理

// 配置恢复参数
props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 1000);  // 定期提交
props.put(StreamsConfig.REPALL_TIMEOUT_MS_CONFIG, 60000); // 恢复超时
```

## Processor API

对于 DSL 不支持的场景，使用 Processor API：

```java
// 自定义 Processor
public class RegionalAggregatorProcessor implements Processor<String, Order, String, Double> {

    private ProcessorContext context;
    private StateStore stateStore;

    @Override
    public void init(ProcessorContext context) {
        this.context = context;
        this.stateStore = context.getStateStore("revenue-store");
        this.context.schedule(Duration.ofSeconds(1), PunctuationType.WALL_CLOCK_TIME, 
            this::punctuate);
    }

    @Override
    public void process(String key, Order value) {
        String region = value.getRegion();
        Double current = (Double) stateStore.get(region);
        Double sum = current == null ? value.getAmount() : current + value.getAmount();
        stateStore.put(region, sum);
    }

    @Override
    public void punctuate(long timestamp) {
        // 定期输出当前状态
        KeyValueIterator<String, Double> it = stateStore.all();
        while (it.hasNext()) {
            KeyValue<String, Double> pair = it.next();
            context.forward(pair.key, pair.value);
        }
    }

    @Override
    public void close() {
        stateStore.close();
    }
}
```

### 结合 DSL 与 Processor API

```java
// 使用 DSL 过滤，用 Processor 聚合
builder.stream("orders")
    .filter((key, order) -> order.getAmount() > 100)
    .process(() -> new RegionalAggregatorProcessor(), "revenue-store")
    .to("revenue-output");
```

## 测试

### TopologyTestDriver

```java
// 使用 TopologyTestDriver 测试
TopologyTestDriver testDriver = new TopologyTestDriver(topology, props);

TestInputTopic<String, Order> inputTopic = 
    testDriver.createInputTopic("orders", 
        new StringSerde().serializer(), 
        new JsonSerde<>(Order.class).serializer());

TestOutputTopic<String, Double> outputTopic = 
    testDriver.createOutputTopic("revenue-per-region",
        new StringSerde().deserializer(),
        new DoubleSerde().deserializer());

// 发送测试数据
inputTopic.pipeInput("us", new Order("us", 100.0));
inputTopic.pipeInput("eu", new Order("eu", 50.0));
inputTopic.pipeInput("us", new Order("us", 150.0));

// 验证结果
Map<String, Double> results = outputTopic.readKeyValuesToMap();
assertEquals(250.0, results.get("us"));
assertEquals(50.0, results.get("eu"));
```

## 生产最佳实践

### 资源规划

```java
// 合理配置线程数和实例数
// 规则: 实例数 × 线程数 ≈ Topic Partition 数

int partitionCount = 12;  // 输入 Topic Partition 数
int threadsPerInstance = 4;
int instancesNeeded = partitionCount / threadsPerInstance;  // 3

props.put(StreamsConfig.NUM_STREAM_THREADS_CONFIG, threadsPerInstance);
```

### 状态存储优化

```java
// 大状态优化策略
StoreBuilder<KeyValueStore<String, Session>> sessionStore =
    Stores.keyValueStoreBuilder(
        Stores.persistentSessionStore("user-sessions", Duration.ofHours(2)),
        Serdes.String(),
        new SessionSerde<>(JsonSerde.class, JsonSerde.class)
    )
    .withCachingEnabled(true)           // 缓存减少 IO
    .withLoggingEnabled(true)            // changelog 持久化
    .withLoggingConfig(new LoggingConfig(Collections.singletonMap(
        "min.cleanable.dirty.ratio", "0.01"  // 降低清理频率
    )));
```

### 异常处理

```java
// 配置错误处理
props.put(StreamsConfig.DEFAULT_DESERIALIZATION_EXCEPTION_HANDLER_CLASS_CONFIG,
    LogAndSkipExceptionHandler.class.getName());

// 或自定义处理器
public class DeadLetterQueueExceptionHandler 
    implements DeserializationExceptionHandler<Integer, String> {

    @Override
    public DeserializationHandlerResponse handle(
            final ProcessorContext context,
            final DeserializationException<Integer, String> exception) {
        
        // 发送到 Dead Letter Topic
        context.forward(exception.topic() + "-dlq", 
            new org.apache.kafka.streams.processor.FailuresException(exception));
        return DeserializationHandlerResponse.CONTINUE;
    }
}
```

## 相关主题

- [[./kafka-internals|Apache Kafka]] — Kafka 深入解析
- [[./pulsar-vs-kafka|消息队列对比]] — Kafka、Pulsar 选型指南
- [[../distributed-systems/event-driven-architecture|事件驱动架构]] — 消息队列应用模式

---

## 参考资料

[^1]: [Kafka Streams Architecture - Apache Kafka](https://kafka.apache.org/38/streams/architecture/)

[^2]: [Introduction to Kafka Streams - Conduktor](https://conduktor.io/glossary/introduction-to-kafka-streams)

[^3]: [Kafka Streams Developer Guide - Apache Kafka](https://kafka.apache.org/25/streams/developer-guide/)
