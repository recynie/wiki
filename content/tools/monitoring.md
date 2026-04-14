---
title: Prometheus与Grafana监控
date: 2026-04-13
description: 介绍Prometheus监控系统和Grafana可视化平台的原理、架构与实践
tags:
  - prometheus
  - grafana
  - monitoring
  - devops
draft: false
permalink:
---

## 监控概述

监控系统是生产环境运维的重要基础设施，其核心目标是**及早发现生产问题**，避免影响用户体验。监控不仅仅是「看到服务器还活着」，而是要能够回答「系统是否正常运转，性能是否达标」。

### 可观测性三要素

现代分布式系统强调可观测性（Observability），主要由三部分组成：

| 要素 | 描述 | 典型工具 |
|------|------|----------|
| **指标（Metrics）** | 数值型时序数据，反映系统状态 | Prometheus、InfluxDB |
| **日志（Logs）** | 事件记录，包含时间戳和上下文 | ELK、Loki |
| **链路追踪（Traces）** | 请求在分布式系统中的调用路径 | Jaeger、Zipkin |

这三个要素相辅相成：指标告诉你「出了问题」，日志告诉你「哪里出了问题」，链路追踪告诉你「问题是怎么发生的」。[^1]

### Prometheus + Grafana组合的优势

- **开源免费**：均为CNCF毕业项目，社区活跃
- **无缝集成**：Prometheus作为数据源，Grafana作为可视化层
- **云原生友好**：天然支持容器化、微服务架构
- **生态丰富**：大量预置 exporters 和 Dashboard

---

## Prometheus基础

### 简介

Prometheus 是一个开源的监控系统，最初由 SoundCloud 开发，现已成为 **CNCF（云原生计算基金会）毕业项目**。它采用独特的 Pull-based 模型，被广泛应用于 Kubernetes 生态系统中。

### Pull-based模型

与传统的 Push-based 系统不同，Prometheus 采用**主动拉取（Pull）**的方式收集指标：

```
┌─────────────┐      Pull       ┌─────────────┐
│  Prometheus │  ────────────→  │   Target    │
│   Server    │    HTTP请求     │ (Exporter)  │
└─────────────┘                 └─────────────┘
```

**优势**：
- 无需在应用端安装代理，Prometheus 统一管理抓取
- 更容易实现高可用，多 Prometheus 实例抓取相同目标
- 指标数据更干净，避免 push 带来的数据丢失问题[^2]

### 数据模型

Prometheus 以**时间序列（Time Series）**存储数据，每条记录由以下部分组成：

```
metric_name{label_name="label_value"} value
```

**示例**：

```
node_cpu_seconds_total{cpu="0",mode="user"} 12345.67
```

组成部分：
- **metric name**：指标名称，描述测量内容（如 `node_cpu_seconds_total`）
- **labels**：键值对标签，用于区分同一指标的不同维度（如 `cpu="0", mode="user"`）
- **value**：指标值
- **timestamp**：时间戳（毫秒级）

### PromQL查询语言

PromQL（Prometheus Query Language）是 Prometheus 的核心查询语言，支持多种表达式类型：

```promql
# 瞬时向量查询：返回当前时刻的样本
node_cpu_seconds_total{mode="user"}

# 区间向量查询：返回一段时间内的样本
rate(node_cpu_seconds_total{mode="user"}[5m])

# 聚合查询
sum(rate(http_requests_total[5m])) by (job)

# 计算CPU使用率
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### 关键组件

| 组件 | 作用 |
|------|------|
| **Prometheus Server** | 核心组件，负责抓取、存储和查询指标 |
| **Node Exporter** | 采集 Linux/Unix 系统指标（CPU、内存、磁盘、网络） |
| **Alertmanager** | 处理来自 Prometheus 的告警，发送通知 |
| **Exporters** | 各类应用的指标导出器（MySQL、Redis、Nginx 等） |

---

## 核心指标类型

Prometheus 定义了四种核心指标类型[^3]：

### Counter（计数器）

**特点**：只增不减的累积值，适用于记录累计发生的事件次数。

**典型应用**：
- HTTP 请求总数
- 数据库查询次数
- 服务重启次数

```promql
# HTTP请求总数
http_requests_total{method="GET", status="200"}

# 计算QPS（每秒请求数）
rate(http_requests_total[5m])
```

### Gauge（仪表值）

**特点**：可增可减的瞬时值，适用于描述当前状态。

**典型应用**：
- 当前 CPU 使用率
- 内存使用量
- 活跃连接数

```promql
# 当前内存使用量（字节）
node_memory_MemTotal_bytes - node_memory_MemFree_bytes

# 磁盘可用空间
node_filesystem_avail_bytes{mountpoint="/"}
```

### Histogram（直方图）

**特点**：将指标值划分到多个桶（bucket）中，适用于计算分位数和分布统计。

**典型应用**：
- 请求延迟分布
- 响应大小分布
- 计算百分位数

```promql
# Histogram 自动生成的桶指标
http_request_duration_seconds_bucket{le="0.1"}

# 计算平均延迟
rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m])
```

Prometheus 会为每个 Histogram 生成以下指标：
- `<name>_bucket{le="<buckets>"}`：各桶的累积计数
- `<name>_sum`：所有观测值的总和
- `<name>_count`：观测值总数

### Summary（分位数统计）

**特点**：在客户端直接计算分位数，服务端存储最终结果。

**典型应用**：需要精确分位数的场景（如 P99 延迟）。

```promql
# Summary 生成的指标
http_request_duration_seconds{quantile="0.5"}  # 中位数
http_request_duration_seconds{quantile="0.9"}  # P90
http_request_duration_seconds{quantile="0.99"} # P99
```

> **注意**：Histogram 可以在服务端计算任意分位数，而 Summary 的分位数在客户端计算后就不能再更改。

---

## Grafana基础

### 简介

Grafana 是一个开源的**可视化平台**，支持多种数据源，可用于创建、查看和分享监控仪表盘。

### 数据源支持

Grafana 原生支持众多数据源：

| 数据源 | 用途 |
|--------|------|
| **Prometheus** | 指标存储和查询 |
| **Elasticsearch** | 日志分析 |
| **Loki** | 日志聚合（由 Grafana 实验室开发） |
| **InfluxDB** | 时序数据库 |
| **MySQL/PostgreSQL** | 关系型数据库 |

### Dashboard创建

Grafana Dashboard 由多个 **Panel（面板）** 组成，每个面板可以：

- 展示折线图、柱状图、仪表盘等
- 设置告警规则
- 支持变量（Variables）实现动态过滤

### Alerting告警功能

Grafana 8.0 后统一了告警系统，支持：

- 基于查询结果的条件触发
- 多种通知渠道（Email、Slack、PagerDuty、webhook 等）
- 告警状态管理（Pending、Firing、Resolved）

---

## 监控架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         监控架构                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌────────────────┐                      ┌──────────────────┐   │
│   │  Node Exporter │                      │   其他 Exporters │   │
│   │    (:9100)     │                      │    (MySQL等)     │   │
│   └───────┬────────┘                      └────────┬─────────┘   │
│           │                                         │             │
│           │            Pull 抓取                    │             │
│           └──────────────────┬──────────────────────┘             │
│                              ↓                                    │
│                    ┌─────────────────┐                          │
│                    │    Prometheus    │                          │
│                    │     (:9090)      │                          │
│                    └────────┬─────────┘                          │
│                             │                                    │
│           ┌─────────────────┼─────────────────┐                 │
│           ↓                 ↓                 ↓                 │
│   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐        │
│   │ Alertmanager   │  │    Grafana    │  │   其他消费者   │        │
│   │   (:9093)      │  │   (:3000)     │  │               │        │
│   └───────┬───────┘  └───────────────┘  └───────────────┘        │
│           │                                                       │
│           ↓                                                       │
│   ┌───────────────┐                                               │
│   │  通知渠道      │  (Email, Slack, PagerDuty, Webhook等)         │
│   └───────────────┘                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**端口说明**：
- `:9100` — Node Exporter 系统指标
- `:9090` — Prometheus Web UI 和 API
- `:9093` — Alertmanager Web UI
- `:3000` — Grafana Web UI

---

## 配置示例

### Prometheus配置

`prometheus.yml` 是 Prometheus 的主配置文件：

```yaml
global:
  # 全局抓取间隔（默认15秒）
  scrape_interval: 15s
  # 评估规则间隔
  evaluation_interval: 15s

# 告警规则配置
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - localhost:9093

# 规则文件
rule_files:
  - "alert_rules.yml"

# 抓取目标配置
scrape_configs:
  # Prometheus 自身监控
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node Exporter 系统监控
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
    relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):9100'
        replacement: '${1}'
        target_label: instance

  # 自定义应用监控
  - job_name: 'my-app'
    static_configs:
      - targets: ['my-app:8080']
    metrics_path: '/metrics'
    scrape_interval: 10s
```

### 告警规则配置

`alert_rules.yml` 定义告警规则：

```yaml
groups:
  - name: node_alerts
    rules:
      # CPU使用率过高
      - alert: HighCPUUsage
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "实例 {{ $labels.instance }} CPU使用率过高"
          description: "CPU使用率已超过80%（当前值：{{ $value }}%）"

      # 内存使用率过高
      - alert: HighMemoryUsage
        expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "实例 {{ $labels.instance }} 内存使用率过高"
          description: "内存使用率已超过85%（当前值：{{ $value }}%）"

      # 磁盘空间不足
      - alert: DiskSpaceLow
        expr: node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100 < 15
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "实例 {{ $labels.instance }} 磁盘空间不足"
          description: "根分区可用空间低于15%"

      # 服务不可用
      - alert: TargetDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "目标 {{ $labels.job }} 不可用"
          description: "目标已连续1分钟无法抓取指标"
```

---

## Grafana Dashboard配置

### 数据源配置

通过 Grafana API 添加 Prometheus 数据源：

```bash
curl -X POST http://localhost:3000/api/datasources \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-api-key>" \
  -d '{
    "name": "Prometheus",
    "type": "prometheus",
    "url": "http://localhost:9090",
    "access": "proxy",
    "isDefault": true
  }'
```

### 常用PromQL模板

```promql
# CPU使用率趋势
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 内存使用率趋势
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# 网络流量（入方向）
rate(node_network_receive_bytes_total[5m])

# 磁盘IOPS
rate(node_disk_io_now[5m])

# HTTP请求QPS
sum(rate(http_requests_total[5m])) by (job, status)
```

---

## 常见命令

### 检查指标端点

```bash
# 查看Node Exporter指标
curl http://localhost:9100/metrics | head -50

# 查看Prometheus指标
curl http://localhost:9090/metrics

# 测试PromQL查询
curl -s 'http://localhost:9090/api/v1/query?query=up'
```

### Prometheus配置验证

```bash
# 验证配置文件语法
promtool check config /etc/prometheus/prometheus.yml

# 验证告警规则
promtool check rules /etc/prometheus/alert_rules.yml

# 测试规则文件
promtool test rules /etc/prometheus/alert_rules.yml
```

### Grafana CLI

```bash
# 重置管理员密码
grafana-cli admin reset-admin-password <new-password>

# 安装插件
grafana-cli plugins install <plugin-id>

# 导出Dashboard配置
curl -s http://localhost:3000/api/dashboards/uid/<uid> \
  -H "Authorization: Bearer <api-key>" | jq '.dashboard' > dashboard.json
```

---

## 最佳实践

### 标签设计（Labels）的重要性

良好的标签设计能显著提升查询效率：

```promql
# 推荐：为每种服务添加明确标签
http_requests_total{service="api-gateway", method="GET", status="200"}

# 避免：滥用标签值
http_requests_total{service="api-gateway", instance="10.0.1.5:8080", method="GET", status="200"}
```

**原则**：
- 标签值不宜过多（高基数），会导致 Prometheus 内存膨胀
- 避免使用 `instance` 作为查询维度时再添加 `host`、`ip` 等同类标签
- 标签名称统一小写

### 告警阈值设置

| 场景 | 建议阈值 | 说明 |
|------|---------|------|
| CPU 使用率 | Warning: 70%, Critical: 85% | 预留处理缓冲 |
| 内存使用率 | Warning: 80%, Critical: 90% | 避免OOM |
| 磁盘使用率 | Warning: 80%, Critical: 90% | 根分区需更保守 |
| 请求延迟 P99 | Warning: 500ms, Critical: 1s | 根据业务SLA调整 |
| 错误率 | Warning: 1%, Critical: 5% | 考虑自动扩容 |

### 监控覆盖层级

完整的监控应覆盖以下层级：

```
┌─────────────────────────────────────────┐
│           应用层（Application）           │
│    业务指标、错误率、延迟分布、吞吐量       │
├─────────────────────────────────────────┤
│           服务层（Service）              │
│    HTTP状态、数据库连接、缓存命中率        │
├─────────────────────────────────────────┤
│           系统层（System）                │
│    CPU、内存、磁盘、网络、负载            │
├─────────────────────────────────────────┤
           基础设施层（Infrastructure）            
│    容器、Pod、网络、存储卷                │
└─────────────────────────────────────────┘
```

---

## 参考资料

[^1]: 本文参考了 Prometheus 官方文档关于架构的介绍：[Prometheus Architecture](https://prometheus.io/docs/introduction/architecture/)

[^2]: 关于 Pull vs Push 模型的优势讨论，参见：[Monitoring Systems: Pull vs Push](https://prometheus.io/docs/introduction/faq/#how-does-prometheus-compare-to-other-monitoring-systems)

[^3]: Prometheus 指标类型官方文档：[Prometheus Metric Types](https://prometheus.io/docs/concepts/metric_types/)

---

## 相关主题

- [[../devops/container-orchestration|容器编排]] — Kubernetes 与监控的结合
- [[../devops/logging|日志收集]] — EFK日志栈与Prometheus的集成
- [[../distributed-systems/observability|可观测性]] — 分布式系统可观测性最佳实践
- [[../tools/docker|Docker]] — 容器化环境下的监控部署
