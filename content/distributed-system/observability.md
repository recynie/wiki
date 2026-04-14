---
title: Observability
date: 2026-04-13
description: 可观测性：四黄金信号、SLO/SLI、Prometheus与Grafana、燃烧率告警
tags:
  - observability
  - prometheus
  - grafana
  - slo
draft: true
permalink:
---

## 四黄金信号（Four Golden Signals）

四黄金信号是 Google SRE 团队提出的可观测性核心指标体系，用于快速评估系统健康状态。[^1]

### Latency（延迟）

延迟是请求从发起到收到响应的时间。关键点在于**区分正常请求与慢速请求**：

- 延迟分布：p50、p95、p99、p999
- 关注异常值：慢速请求可能暗示系统瓶颈
- 区分错误与慢响应：超时≠错误，但都需关注

```promql
# 计算 HTTP 请求延迟的 p99 分位数
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

### Traffic（流量）

流量反映系统处理的请求量，通常用 **QPS（Queries Per Second）** 或 **TPS（Transactions Per Second）** 衡量：

- 帮助判断系统负载水平
- 识别流量异常（突增或骤降）
- 容量规划的重要依据

```promql
# 计算每秒 HTTP 请求数
rate(http_requests_total[1m])

# 按状态码分组统计
rate(http_requests_total{status=~"5.."}[5m])
```

### Errors（错误）

错误率是系统健康程度的直接指标，需区分不同类型：

| 错误类型 | 说明 | 严重程度 |
|---------|------|---------|
| 5xx | 服务器端错误 | 高 |
| 4xx | 客户端错误 | 中（需分析） |
| 超时 | 请求超时 | 高 |
| 自定义错误码 | 业务定义错误 | 视业务而定 |

```promql
# 计算 5xx 错误率
sum(rate(http_requests_total{status=~"5.."}[5m])) 
/ 
sum(rate(http_requests_total[5m]))

# 自定义业务错误率
sum(rate(myapp_errors_total{type="business"}[5m]))
```

### Saturation（饱和度）

饱和度衡量资源的利用程度，当资源接近极限时系统性能会急剧下降：

- **CPU 使用率**：长期高于 80% 需关注
- **内存使用率**：内存泄漏会导致 OOM
- **磁盘 IO**：IO 瓶颈会导致请求排队
- **连接数**：数据库连接池、线程池满会导致拒绝服务

```promql
# CPU 使用率
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 内存使用率
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) 
/ node_memory_MemTotal_bytes * 100

# 磁盘 IO 利用率
rate(node_disk_io_time_seconds_total[5m]) * 100
```

---

## SLO / SLI / SLA 概念

这三个概念定义了服务质量的衡量标准。[^2]

### SLI（Service Level Indicator）

SLI 是**量化指标**，直接测量服务质量：

| 常见 SLI | 测量方式 |
|---------|---------|
| 延迟 | 请求响应时间（p99 < 200ms） |
| 可用性 | 成功请求比例（> 99.9%） |
| 吞吐量 | QPS（> 1000 req/s） |
| 质量 | 错误率（< 0.1%） |

### SLO（Service Level Objective）

SLO 是 SLI 的**目标值**，是团队承诺的内部标准：

> "我们的 API 服务可用性目标是 99.9%，即每月允许的不可用时间约 8.7 分钟。"

SLO 示例：
- 延迟：p99 < 500ms
- 可用性：月度 99.95%
- 吞吐量：支持 10,000 QPS

### SLA（Service Level Agreement）

SLA 是**合同承诺**，通常包含：

- 与客户/用户签订的正式协议
- 未达标的赔偿条款（退款、优惠券等）
- SLA 通常比 SLO 更宽松（留有缓冲）

### Error Budget（错误预算）

错误预算 = `1 - SLO`，用于量化剩余的容错空间：

| SLO | 错误预算（每月） |
|-----|----------------|
| 99% | 7.3 小时 |
| 99.9% | 43.8 分钟 |
| 99.99% | 4.4 分钟 |

**错误预算的作用**：
- 判断系统稳定性：预算消耗速度
- 指导发布决策：预算充足时可以更激进发布
- 避免过度优化：预算充足时无需投入大量精力

```promql
# 计算当前月的错误预算消耗
# 假设 SLO 为 99.9%（错误预算 0.1%）

# 实际错误率
error_rate = sum(rate(http_requests_total{status=~"5.."}[1h])) 
             / sum(rate(http_requests_total[1h]))

# 错误预算剩余比例（以月为单位）
error_budget_remaining = 1 - (error_rate * 30 * 24)
```

---

## Prometheus + Grafana 监控体系

### Prometheus 核心概念

Prometheus 是 CNCF 毕业项目，采用 **Pull 模型**收集指标。[^3]

**核心架构**：
```
┌─────────────┐     scrape      ┌────────────┐
│  Exporter   │ ──────────────→ │ Prometheus  │
│  (node_exporter)             │            │
└─────────────┘                │  ┌──────┐   │
                               │  │ TSDB │   │
┌─────────────┐                │  └──────┘   │
│ PushGateway │ ─── push ─────→ │            │
│ (batch jobs)│                └──────┬─────┘
└─────────────┘                       │ query
                                      ▼
                               ┌────────────┐
                               │  Grafana   │
                               │ Dashboard  │
                               └────────────┘
```

**关键组件**：

| 组件 | 用途 |
|-----|------|
| Exporters | 暴露系统/应用指标（node_exporter, mysqld_exporter） |
| PushGateway | 接收短期任务的指标推送 |
| AlertManager | 处理和发送告警 |
| Recording Rules | 预计算常用查询 |

### 常用 Exporter

| Exporter | 监控目标 |
|----------|---------|
| node_exporter | CPU、内存、磁盘、网络 |
| mysqld_exporter | MySQL 查询、连接、慢查询 |
| redis_exporter | Redis 内存、命令统计 |
| blackbox_exporter | HTTP/TCP/ICMP 探测 |
| cadvisor | Docker 容器指标 |

### 常用 PromQL 查询

```promql
# --- 请求率与错误率 ---

# QPS（每秒请求数）
sum(rate(http_requests_total[5m]))

# 按服务维度的 QPS
sum by (service) (rate(http_requests_total[5m]))

# 5xx 错误率
sum(rate(http_requests_total{status=~"5.."}[5m])) 
/ sum(rate(http_requests_total[5m]))

# --- 延迟统计 ---

# 平均延迟
rate(http_request_duration_seconds_sum[5m]) 
/ rate(http_request_duration_seconds_count[5m])

# P99 延迟
histogram_quantile(0.99, 
    rate(http_request_duration_seconds_bucket[5m]))

# --- 资源利用率 ---

# Pod CPU 使用率（Kubernetes 环境）
sum(rate(container_cpu_usage_seconds_total{pod!=""}[5m])) by (pod)
/
sum(container_spec_cpu_quota{pod!=""}/container_spec_cpu_period{pod!=""}) by (pod) * 100

# Pod 内存使用率
sum(container_memory_working_set_bytes{pod!=""}) by (pod)
/
sum(container_spec_memory_limit_bytes{pod!=""}) by (pod) * 100
```

### Recording Rules（记录规则）

记录规则用于预计算常用查询，提升查询性能：

```yaml
groups:
  - name: http_slo
    interval: 30s
    rules:
      # 预计算 SLO 指标
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
      
      - record: job:http_request_errors:rate5m
        expr: sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
      
      - record: job:http_request_latency_p99:rate5m
        expr: histogram_quantile(0.99, 
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m])))
```

### Grafana 告警规则

```yaml
# Grafana Alerting Rule 示例
groups:
  - name: service_health
    rules:
      # 高错误率告警
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) 
          / sum(rate(http_requests_total[5m])) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "服务错误率超过 1%"
          description: "{{ $labels.job }} 错误率: {{ $value | humanizePercentage }}"

      # 高延迟告警
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, 
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99 延迟超过 1 秒"
```

---

## Burn Rate 告警

Burn Rate（燃烧率）是一种基于错误预算的告警方法，比传统阈值告警更敏感于长期趋势。[^4]

### 核心概念

**Burn Rate** 定义为错误预算的消耗速度：

$$
\text{burn\_rate} = \frac{\text{error\_budget\_consumed}}{\text{time\_elapsed}}
$$

### 多窗口 Burn Rate

传统单窗口告警难以同时检测快速和缓慢的预算消耗，因此采用**双窗口策略**：

| 窗口类型 | 短期窗口 | 长期窗口 | 用途 |
|---------|---------|---------|-----|
| 快燃 | 1 小时 | 1 天 | 检测快速故障（如服务崩溃） |
| 中燃 | 6 小时 | 3 天 | 检测中速故障 |
| 慢燃 | 1 小时 | 5 天 | 检测缓慢退化（如内存泄漏） |

### 告警公式

对于给定窗口，burn rate 计算如下：

```promql
# 短期窗口（如 1h）的错误预算消耗
short_window_error_budget_consumed = 
    sum(increase(http_requests_total{status=~"5.."}[1h]))
    /
    (sum(increase(http_requests_total[1h])) * (1 - 0.999))

# Burn Rate = 短期消耗 / 长期消耗比例
# 如果 1h 内消耗了 5% 的月度错误预算，burn rate = 30x
# （即以当前速度，30 天才能消耗完的预算在 1 小时内消耗了）
```

### 实际告警规则示例

```yaml
groups:
  - name: burn_rate_alerts
    interval: 1m
    rules:
      # 快燃告警 - 1 小时内消耗 10% 错误预算（burn rate > 60x）
      - alert: FastBurn
        expr: |
          (
            sum(increase(http_requests_total{status=~"5.."}[1h]))
            /
            (sum(increase(http_requests_total[1h])) * 0.001)
          ) > 60
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "错误预算快速消耗！"
          description: "1 小时内已消耗 {{ $value | humanize }}% 的月度错误预算"

      # 慢燃告警 - 5 天内消耗 100% 错误预算（burn rate > 10x）
      - alert: SlowBurn
        expr: |
          (
            sum(increase(http_requests_total{status=~"5.."}[5d]))
            /
            (sum(increase(http_requests_total[5d])) * 0.001)
          ) > 10
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "错误预算缓慢消耗中"
          description: "5 天内已消耗 {{ $value | humanize }}% 的月度错误预算"

      # 综合告警 - 结合多窗口
      - alert: MultiWindowBurn
        expr: |
          # 短期：1 小时窗口，burn rate > 15（消耗 10% 预算）
          (sum(increase(http_requests_total{status=~"5.."}[1h]))
           / (sum(increase(http_requests_total[1h])) * 0.001) > 15)
          or
          # 长期：5 天窗口，burn rate > 10（消耗 100% 预算）
          (sum(increase(http_requests_total{status=~"5.."}[5d]))
           / (sum(increase(http_requests_total[5d])) * 0.001) > 10)
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "多窗口检测到错误预算燃烧"
```

### Burn Rate 与 SLO 对照表

| Burn Rate | 1h 窗口消耗 | 5d 窗口消耗 | 告警级别 |
|-----------|-----------|------------|---------|
| 1x | 0.14% | 6.7% | 正常 |
| 10x | 1.4% | 67% | 警告 |
| 60x | 8.3% | 100%+ | 严重 |
| 360x | 50%+ | 100%+ | 紧急 |

### 最佳实践

1. **多窗口组合**：不要只用单一窗口，短窗口检测快速故障，长窗口检测缓慢退化
2. **分级告警**：根据 burn rate 阈值设置不同严重级别
3. **消除噪音**：设置合理的 `for`  duration，避免瞬时抖动触发告警
4. **定期复盘**：分析告警触发原因，持续优化 SLO 目标

---

## 参考资料

[^1]: Google SRE Book - [Four Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/)
[^2]: Google SRE Book - [Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
[^3]: Prometheus Documentation - [Architecture](https://prometheus.io/docs/introduction/overview/)
[^4]: Google SRE Workbook - [Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
