---
title: 分布式追踪：OpenTelemetry 与 Jaeger 实践指南
date: 2026-04-14
description: 深入解析分布式追踪技术栈：OpenTelemetry SDK、Collector、OTLP 协议，以及 Jaeger 的部署与最佳实践
tags:
  - distributed-tracing
  - opentelemetry
  - jaeger
  - observability
  - microservices
draft: false
permalink:
---

## 概述

### 什么是分布式追踪

分布式追踪（Distributed Tracing）是一种用于监控和追踪分布式系统中请求流转路径的技术手段。在微服务架构下，一个用户请求可能跨越数十个服务节点，传统日志和监控手段难以追踪完整请求链路。分布式追踪通过为每个请求分配唯一标识，将分散在各个服务中的调用串联成一条完整的「追踪链」。

假设有一个电商系统，用户下单流程涉及：网关 → 鉴权服务 → 商品服务 → 库存服务 → 支付服务 → 订单服务。如果支付失败，没有分布式追踪时，开发者需要在每个服务的日志中逐一搜索，耗时且低效。有了分布式追踪后，可以在 Jaeger UI 中直观看到整个调用链路，快速定位问题发生在库存服务还是支付服务。

### 为什么需要分布式追踪

单体架构时代，应用运行在单一进程内，调用栈清晰可见。微服务架构下，问题定位变得复杂：

| 场景 | 无追踪 | 有追踪 |
|------|--------|--------|
| 慢请求定位 | 逐个服务查日志 | 链路可视化，快速定位瓶颈 |
| 跨服务调用错误 | 猜测调用顺序 | 精准还原调用链 |
| 系统架构梳理 | 依赖文档或人工梳理 | 自动生成拓扑图 |
| 性能优化 | 盲目优化 | 有据可依的热点分析 |

分布式追踪是**可观测性（Observability）**的三大支柱之一，与**指标（Metrics）**和**日志（Logs）**共同构成完整的监控系统。[^1]

---

## 核心概念

### Trace 与 Span

**Trace**（追踪）是请求在分布式系统中的完整流转路径，每个请求对应一个 Trace。Trace 由多个 **Span**（跨度）组成。

**Span** 是追踪的基本单元，代表一个逻辑操作单元。每个 Span 包含：

- **Span Name**：操作名称，如 `HTTP GET /api/orders`
- **Start Time / End Time**：操作耗时
- **SpanContext**：跨服务传递的上下文信息
- **Attributes**：键值对形式的元数据
- **Events**：时间点事件
- **Status**：操作状态（OK / ERROR）

```
Trace: [Span A] → [Span B] → [Span C]
                 ↘ [Span D] ↗
```

### SpanContext 与 Trace ID

**SpanContext** 包含在服务间传递所需的上下文信息：

- **Trace ID**：全局唯一标识整个追踪链路，长度为 128 位（32 字节十六进制字符串）
- **Span ID**：当前 Span 的唯一标识，长度为 64 位
- **Trace Flags**：采样相关标志
- **Trace State**：用于多租户场景的键值对

```
TraceID: 4bf92f3577b34da6a3ce929d0e0e4736
SpanID:  00f067aa0ba902b7
```

### Parent-Child 关系

Span 之间通过 Parent-Span 关系构成树状结构。根 Span（Root Span）代表整个请求的入口，通常由网关或入口服务创建。子 Span 由子服务创建，通过 SpanContext 传递建立关联。

```
[Root Span]
    ├── [Child Span: 库存服务]
    │       └── [Grandchild Span: 库存扣减]
    └── [Child Span: 支付服务]
            └── [Grandchild Span: 支付扣款]
```

---

## OpenTelemetry

### 架构概述

OpenTelemetry（OTel）是 CNCF 旗下的开源可观测性框架，提供了统一的 API、SDK 和 Collector，用于收集 traces、metrics 和 logs。其设计目标是 vendor-neutral（供应商无关），支持多种后端导出。

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Application  │ OTLP │   Collector │     │   Backend   │
│   (SDK)      │─────▶│   (Agent)   │────▶│  (Jaeger)   │
└─────────────┘     └─────────────┘     └─────────────┘
```

### 核心组件

**OpenTelemetry SDK** 提供各语言的实现：

- **API**：定义 Tracer、Span、SpanContext 等接口
- **SDK**：实现 API，提供配置、采样、上下文传播等功能
- **Instrumentation Libraries**：自动埋点库（HTTP、gRPC、数据库客户端等）

**OpenTelemetry Collector** 是一个独立的进程，接收、处理和导出遥测数据。它支持：

- OTLP 协议（gRPC/HTTP）
- Jaeger、Zipkin、Prometheus 等多种导出器
- 数据转换和过滤

### OTLP 协议

OTLP（OpenTelemetry Protocol）是 OTel 的标准传输协议，基于 Protobuf 编码：

```protobuf
message ExportTraceServiceRequest {
  repeated ResourceSpans resource_spans = 1;
}

message ResourceSpans {
  Resource resource = 1;
  repeated ScopeSpans scope_spans = 2;
}
```

OTLP 支持两种传输方式：

| 传输方式 | 端口 | 特点 |
|----------|------|------|
| gRPC | 4317 | 高效，支持双向流 |
| HTTP/JSON | 4318 | 穿透性强，兼容性好 |

### 自动埋点与手动埋点

**自动埋点（Auto Instrumentation）** 通过拦截常见库实现，无需修改业务代码：

```python
# Python 自动埋点示例
from opentelemetry import trace
from opentelemetry.instrumentation.flask import FlaskInstrumentor

app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)  # 自动拦截 Flask 请求
```

**手动埋点（Manual Instrumentation）** 通过 API 精确控制 Span 的创建：

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

def query_order(order_id):
    with tracer.start_as_current_span("query_order") as span:
        span.set_attribute("order.id", order_id)
        span.set_attribute("service.name", "order-service")
        # 业务逻辑
        result = db.query(order_id)
        if result is None:
            span.set_status(trace.Status(trace.StatusCode.ERROR, "Order not found"))
        return result
```

---

## Jaeger

### 架构组件

Jaeger 是 Uber 开源的分布式追踪系统，由以下组件构成：

```
┌─────────┐     ┌────────────┐     ┌───────────┐     ┌─────────┐
│  Agent  │────▶│  Collector  │────▶│   Storage  │◀────│  Query  │
└─────────┘     └────────────┘     └───────────┘     └─────────┘
   ▲                                                    │
   │                                                    ▼
┌─────────┐                                       ┌─────────┐
│ Client  │                                       │   UI    │
└─────────┘                                       └─────────┘
```

| 组件 | 功能 |
|------|------|
| **Agent** | 部署在每个节点上的守护进程，接收客户端数据，通过 UDP 发送到 Collector |
| **Collector** | 接收并处理 Trace 数据，验证并存储到后端存储 |
| **Storage** | 支持 Cassandra、Elasticsearch、PostgreSQL 等 |
| **Query** | 提供 API 供 UI 查询 |

### 部署模式

**All-in-One 模式** 适用于开发测试：

```yaml
# docker-compose all-in-one
version: '3.8'
services:
  jaeger:
    image: jaegertracing/all-in-one:1.52
    ports:
      - "16686:16686"  # UI
      - "4317:4317"    # OTLP gRPC
      - "4318:4318"    # OTLP HTTP
    environment:
      - COLLECTOR_OTLP_ENABLED=true
```

**Production 模式** 各组件独立部署，支持高可用：

```yaml
# production-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger-collector
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: collector
        image: jaegertracing/jaeger-collector:1.52
        ports:
        - containerPort: 14267
        env:
        - name: SPAN_STORAGE_TYPE
          value: elasticsearch
        - name: ES_SERVER_URLS
          value: http://elasticsearch:9200
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-collector
spec:
  ports:
  - port: 14267
    targetPort: 14267
  selector:
    app: jaeger-collector
```

### UI 功能

Jaeger UI 提供以下核心功能：

- **Search Trace**：按 Trace ID、服务名、时间范围搜索
- **Trace Timeline**：可视化 Span 时间线
- **Trace Graph**：调用关系图
- **Statistics**：延迟百分位、QPS 等统计
- **System Topology**：服务依赖拓扑图

---

## 实战：完整配置示例

### Python 服务接入 OpenTelemetry

```python
# order_service.py
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource, SERVICE_NAME
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.pymongo import PymongoInstrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor

def setup_tracing(service_name: str, endpoint: str):
    # 创建资源
    resource = Resource.create({
        SERVICE_NAME: service_name,
        "service.version": "1.0.0",
        "deployment.environment": "production"
    })

    # 创建 TracerProvider
    provider = TracerProvider(resource=resource)

    # 配置 OTLP 导出器
    otlp_exporter = OTLPSpanExporter(
        endpoint=endpoint,  # 例如：http://jaeger-collector:4317
        insecure=True
    )

    # 添加批处理器
    provider.add_span_processor(BatchSpanProcessor(otlp_exporter))

    # 设置为全局 provider
    trace.set_tracer_provider(provider)

    return trace.get_tracer(__name__)

# 初始化
tracer = setup_tracing("order-service", "http://jaeger-collector:4317")

# Flask 自动埋点
FlaskInstrumentor().instrument_app(app)
PymongoInstrumentor().instrument()
RedisInstrumentor().instrument()
```

### Go 服务接入

```go
// main.go
package main

import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/trace"
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
    "google.golang.org/grpc"
)

func initTracer(endpoint string) (func(), error) {
    exporter, err := otlptracegrpc.New(
        context.Background(),
        otlptracegrpc.WithEndpoint(endpoint),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }

    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithResource(resource.NewWithAttributes(
            []attribute.KeyValue{
                attribute.String("service.name", "payment-service"),
            },
        )),
    )

    otel.SetTracerProvider(tp)

    // gRPC 拦截器自动埋点
    grpcServer := grpc.NewServer(
        grpc.UnaryInterceptor(otelgrpc.UnaryServerInterceptor()),
    )

    return func() {
        tp.Shutdown(context.Background())
    }, nil
}
```

### Collector 配置

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

  jaeger:
    protocols:
      thrift_binary:
        endpoint: 0.0.0.0:6832
      thrift_http:
        endpoint: 0.0.0.0:14268

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024

  memory_limiter:
    check_interval: 1s
    limit_mib: 1000

  # 过滤测试环境数据
  filter:
    spans:
      exclude:
        match_type: strict
        attributes:
          - key: deployment.environment
            value: test

exporters:
  jaeger:
    endpoint: jaeger-agent:14250
    tls:
      insecure: false

  prometheus:
    endpoint: 0.0.0.0:8889

service:
  pipelines:
    traces:
      receivers: [otlp, jaeger]
      processors: [batch, memory_limiter]
      exporters: [jaeger]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
```

### Kubernetes 部署

```yaml
# otel-collector-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  labels:
    app: otel-collector
spec:
  replicas: 2
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector:0.96.0
        args: ["--config=/etc/otel-collector-config.yaml"]
        ports:
        - name: otlp-grpc
          containerPort: 4317
        - name: otlp-http
          containerPort: 4318
        - name: jaeger-thrift
          containerPort: 14268
        volumeMounts:
        - name: config
          mountPath: /etc
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: config
        configMap:
          name: otel-collector-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
data:
  otel-collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
    exporters:
      jaeger:
        endpoint: jaeger-collector:14250
    service:
      pipelines:
        traces:
          receivers: [otlp]
          exporters: [jaeger]
```

```bash
# 部署命令
kubectl apply -f otel-collector-configmap.yaml
kubectl apply -f otel-collector-deployment.yaml

# 验证部署
kubectl get pods -l app=otel-collector
kubectl logs -l app=otel-collector
```

---

## 最佳实践

### 采样策略

采样是控制追踪数据量的关键策略。选择不当会导致关键请求丢失或系统负载过高。

**头部采样（Head-based Sampling）** 在请求入口处决定是否采样：

```yaml
# OpenTelemetry Collector 采样配置
processors:
  tail_sampling:
    decision_wait: 10s
    policies:
      # 始终采样错误请求
      - name: errors-policy
        type: status_code
        status_code: { status_codes: [ERROR] }
      # 按比例采样
      - name: probabilistic-policy
        type: probabilistic
        probabilistic: { sampling_percentage: 10 }
      # 采样慢请求
      - name: slow-traces-policy
        type: latency
        latency: { threshold_ms: 1000 }
```

| 策略 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| Always On | 不丢失任何追踪 | 资源消耗大 | 调试环境 |
| Probabilistic | 资源可控 | 可能丢失关键请求 | 生产环境基线 |
| Tail-based | 按条件灵活采样 | 实现复杂 | 精准采样 |

**尾部采样（Tail-based Sampling）** 在请求结束后根据结果决定采样，需要 Collector 支持。上述配置中的 `tail_sampling` 即为此类。

### Span 属性规范

添加有意义的 Span 属性能极大提升排查效率：

```python
# 推荐：为每个关键操作添加属性
span.set_attribute("db.system", "postgresql")
span.set_attribute("db.name", "orders_db")
span.set_attribute("db.statement", "SELECT * FROM orders WHERE id=?")  # 脱敏后
span.set_attribute("db.operation", "SELECT")
span.set_attribute("db.sql.table", "orders")

# HTTP 场景
span.set_attribute("http.method", "POST")
span.set_attribute("http.url", "/api/v1/orders")
span.set_attribute("http.status_code", 201)
span.set_attribute("http.response_content_length", 1024)

# gRPC 场景
span.set_attribute("rpc.system", "grpc")
span.set_attribute("rpc.method", "/order.OrderService/Create")
span.set_attribute("rpc.grpc.status_code", 0)
```

### 保留策略

根据业务需求和数据价值设置合理的保留期：

| 环境 | 保留期 | 存储建议 |
|------|--------|----------|
| 开发/测试 | 1-3 天 | 本地存储 |
| 生产 | 7-30 天 | Elasticsearch + 冷存储归档 |
| 合规要求 | 1 年+ | 对象存储 + Analytics |

---

## 相关工具

### Zipkin 兼容

OpenTelemetry 原生支持 Zipkin 数据导出，只需修改 Collector 配置：

```yaml
exporters:
  zipkin:
    endpoint: http://zipkin:9411/api/v2/spans
    format: json
```

已有 Zipkin 部署的团队可以通过 Collector 进行协议转换，逐步迁移到 OpenTelemetry 生态。

### Prometheus 指标集成

将追踪数据与指标结合可以获得更全面的视图：

```yaml
# 在 Collector 中启用 prometheus 指标导出
exporters:
  prometheus:
    endpoint: 0.0.0.0:8889
    namespace: otel
    const_labels:
      service: jaeger
```

Prometheus 可以从 Collector 拉取以下指标：

- `otel_colExporter_sent_spans` - 已导出的 Span 数量
- `otel_colExporter_dropped_spans` - 丢弃的 Span 数量
- `otel_processor_batch_size` - 批处理大小

### Grafana 集成

```bash
# docker-compose 配置
services:
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
    volumes:
      - ./provisioning:/etc/grafana/provisioning

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
```

Jaeger 数据源配置：

```json
{
  "name": "Jaeger",
  "type": "jaeger",
  "url": "http://jaeger:16686",
  "access": "proxy",
  "jsonData": {
    "tracesToMetrics": true,
    "serviceMapping": [
      {"name": "order-service", "displayName": "Order Service"}
    ]
  }
}
```

---

## 故障排查

### 常见问题

**问题 1：Trace 数据丢失**

可能原因及排查步骤：

1. 检查网络连通性：`telnet jaeger-collector 4317`
2. 验证 Collector 日志：`kubectl logs -f deployment/otel-collector`
3. 检查Exporter配置是否正确
4. 确认采样率不为 0

**问题 2：Span 父子关系断裂**

通常由上下文传播失败导致。检查：

1. 客户端是否正确注入 SpanContext 到 HTTP Headers
2. 服务间通信是否使用了统一的上下文传播器

```python
# 确保使用 W3C TraceContext 传播
from opentelemetry.propagate import set_global_textmap
from opentelemetry.propagation.trace_context import TraceContextTextMapPropagator

set_global_textmap(TraceContextTextMapPropagator())
```

**问题 3：性能开销过高**

追踪本身会引入延迟和资源消耗。建议：

1. 使用异步导出（BatchSpanProcessor）
2. 合理配置采样率（生产环境通常 10%-30%）
3. 避免在 Span 属性中存储大对象

### 调试技巧

```bash
# 使用 all-in-one 镜像的探针功能
kubectl run debug-jaeger --image=jaegertracing/all-in-one:1.52 \
  --rm -it --entrypoint /bin/sh

# 检查 Collector 健康状态
curl http://jaeger-collector:14267/info

# 查看 Trace 统计
curl http://jaeger-query:16686/api/traces?service=order-service&limit=100
```

---

## 参考资料

[^1]: OpenTelemetry 官方文档 - https://opentelemetry.io/docs/
[^2]: Jaeger 官方文档 - https://www.jaegertracing.io/docs/latest/
[^3]: W3C Trace Context 规范 - https://www.w3.org/TR/trace-context/
[^4]: OpenTelemetry Go SDK - https://github.com/open-telemetry/opentelemetry-go
[^5]: OpenTelemetry Python SDK - https://github.com/open-telemetry/opentelemetry-python

---

## 相关主题

[[distributed-systems-overview|分布式系统概述]] | [[service-mesh|服务网格]] | [[message-queue|消息队列]]