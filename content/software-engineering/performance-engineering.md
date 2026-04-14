---
title: Performance Engineering
date: 2026-04-14
description: 性能工程：系统性能分析与优化方法论
tags:
  - performance
  - optimization
  - profiling
  - capacity-planning
draft: true
permalink:
---

# Performance Engineering

## 概述

性能工程是系统化地分析、测量和优化软件系统性能的方法论。涵盖从代码级优化到架构级扩展的完整技术栈。

## 性能指标

### 延迟（Latency）

| 指标 | 含义 | 适用场景 |
|------|------|----------|
| **P50 (中位数)** | 50% 请求的响应时间 | 典型用户体验 |
| **P90** | 90% 请求的响应时间 | 设置 SLA 基线 |
| **P95** | 95% 请求的响应时间 | 识别长尾问题 |
| **P99** | 99% 请求的响应时间 | 高要求场景 |
| **P999** | 99.9% 请求的响应时间 | 极端优化 |

```
Latency Distribution:
│
│                                    ███
│                              ████████
│                        ████████████████
│                  ████████████████████████
│            ████████████████████████████████
│      ███████████████████████████████████████████
├──────────────────────────────────────────────────────────▶
P50   P75          P90              P95        P99        P999
```

### 吞吐量（Throughput）

- **QPS**（Queries Per Second）：每秒查询数
- **TPS**（Transactions Per Second）：每秒事务数
- **RPS**（Requests Per Second）：每秒请求数
- **OPS**（Operations Per Second）：每秒操作数

### 资源利用率

- CPU 利用率（User/Sys/Idle）
- 内存使用量（RSS/Heap/Virtual）
- 磁盘 I/O（IOPS/Throughput/Latency）
- 网络带宽（bps/PPS）

## 性能分析方法

### USE 方法（Utilization-Saturation-Errors）

```bash
# 检查 CPU
vmstat 1                    # CPU 使用率
mpstat -P ALL 1            # 每核心详细
top / htop                 # 实时监控

# 检查内存
free -m                     # 内存使用
vmstat 1                   # 虚拟内存统计

# 检查 I/O
iostat -xz 1               # 磁盘 I/O
iotop                      # 按进程查看 I/O

# 检查网络
netstat -s                 # 网络统计
ss -s                      # Socket 统计
sar -n DEV 1               # 网络设备流量
```

### 性能剖析（Profiling）

#### CPU Profiling

```python
# Python: cProfile
python -m cProfile -s cumtime myscript.py

# Python: py-spy（生产环境安全）
py-spy record -o profile.svg --pid <pid>

# Go: pprof
import (
    "net/http"
    _ "net/http/pprof"
)
# 访问 http://localhost:6060/debug/pprof/
```

```bash
# Go pprof 命令行
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
(pprof) top 10
(pprof) web          # 生成火焰图
```

#### Memory Profiling

```bash
# Python: memory_profiler
@profile
def my_function():
    ...

python -m memory_profiler myscript.py

# Go: pprof heap
go tool pprof http://localhost:6060/debug/pprof/heap
```

### 追踪（Tracing）

#### 分布式追踪

```bash
# Jaeger（OpenTelemetry）
jaeger-agent --collector.zipkin-url=http://localhost:9411

# Zipkin
zipkin-server --zipkin-port=9411
```

#### 火焰图（Flame Graph）

```bash
# 使用 perf 生成
perf record -F 99 -a -g -- sleep 30
perf script | ./FlameGraph/stackcollapse-perf.pl | \
    ./FlameGraph/flamegraph.pl > perf.svg

# Go 火焰图
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/profile
```

## 数据库性能

### 查询分析

```sql
-- MySQL: EXPLAIN
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';

-- PostgreSQL: EXPLAIN ANALYZE
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) 
SELECT * FROM orders WHERE user_id = 123;

-- MongoDB: explain()
db.orders.find({user_id: 123}).explain('executionStats')
```

### 索引优化

```sql
-- 创建索引
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC);

-- 复合索引原则：最左前缀匹配
-- idx(a, b, c) 支持 a, ab, abc 查询，不支持 b, c 查询

-- 部分索引
CREATE INDEX idx_orders_active ON orders(created_at) 
WHERE status = 'active';

-- 覆盖索引（查询所有列都在索引中）
CREATE INDEX idx_users_cover ON users(email) INCLUDE (name, age);
```

### 连接池配置

```python
# SQLAlchemy 连接池
engine = create_engine(
    'postgresql://user:pass@localhost/db',
    pool_size=20,           # 基础连接数
    max_overflow=10,         # 超出 pool_size 的最大连接数
    pool_pre_ping=True,      # 连接前测试
    pool_recycle=3600,       # 连接回收时间（秒）
)
```

## 缓存优化

### 多级缓存

```
┌─────────────┐
│   Browser   │ ◀── HTTP Cache (Cache-Control, ETag)
└──────┬──────┘
       ▼
┌─────────────┐
│   CDN       │ ◀── 静态资源缓存
└──────┬──────┘
       ▼
┌─────────────┐
│ Redis/Memcached │ ◀── 应用层缓存
└──────┬──────┘
       ▼
┌─────────────┐
│ Database    │ ◀── 查询缓存（部分 DB）
└─────────────┘
```

### 缓存策略

| 策略 | 描述 | 一致性 | 复杂度 |
|------|------|--------|--------|
| **Cache-Aside** | 应用同时管 DB 和缓存 | 强 | 中 |
| **Read-Through** | 缓存自动加载 | 弱 | 低 |
| **Write-Through** | 写入 DB 和缓存 | 强 | 中 |
| **Write-Behind** | 异步写入 DB | 弱 | 高 |

### 缓存失效处理

```python
# 延迟双删（保证缓存一致性）
def update_user(user_id, data):
    # 1. 更新数据库
    db.update('users', user_id, data)
    
    # 2. 删除缓存
    redis.delete(f'user:{user_id}')
    
    # 3. 延迟一段时间后再删除（处理并发读）
    import threading
    def delayed_delete():
        time.sleep(0.5)
        redis.delete(f'user:{user_id}')
    threading.Thread(target=delayed_delete).start()
```

## 并发优化

### 异步处理

```python
# Python asyncio
import asyncio

async def fetch_data(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()

async def main():
    urls = ['http://api1.example.com', 'http://api2.example.com', ...]
    # 并发请求
    results = await asyncio.gather(*[fetch_data(url) for url in urls])

# Go goroutine
func fetchAll(urls []string) []string {
    results := make(chan string, len(urls))
    var wg sync.WaitGroup
    
    for _, url := range urls {
        wg.Add(1)
        go func(u string) {
            defer wg.Done()
            results <- fetch(u)
        }(url)
    }
    
    go func() {
        wg.Wait()
        close(results)
    }()
    
    var out []string
    for r := range results {
        out = append(out, r)
    }
    return out
}
```

### 批量处理

```python
# 批量 DB 操作
def batch_insert(items, batch_size=1000):
    for i in range(0, len(items), batch_size):
        batch = items[i:i+batch_size]
        db.bulk_insert(batch)
        db.commit()

# 批量 API 调用
def batch_get_users(user_ids):
    """N+1 问题解决：批量获取"""
    # 差：循环内单条查询
    # for uid in user_ids:
    #     fetch_user(uid)
    
    # 好：一次批量查询
    users = db.query('SELECT * FROM users WHERE id IN (?)', user_ids)
    return {u.id: u for u in users}
```

## 扩展性设计

### 水平扩展 vs 垂直扩展

| 维度 | 水平扩展 | 垂直扩展 |
|------|----------|----------|
| 扩展方式 | 增加机器 | 升级硬件 |
| 单点故障 | 无（有冗余）| 有 |
| 成本 | 线性增长 | 有上限 |
| 复杂度 | 高 | 低 |
| 瓶颈 | 无特定瓶颈 | CPU/内存/磁盘 |

### 无状态设计

```yaml
# 无状态应用：所有状态外置到缓存/数据库
application:
  instances: 100
  # 不保存任何会话状态
  # 请求可路由到任意实例

# 会话状态外置
session:
  backend: redis
  ttl: 3600
```

### 分片策略

```python
# 哈希分片
def sharding_key(user_id):
    return hash(user_id) % NUM_SHARDS

# 范围分片（按时间）
def time_sharding(created_at):
    year = created_at.year
    return (year - 2020) % NUM_SHARDS

# 一致性哈希
from consistent_hash import ConsistentHash
ch = ConsistentHash(nodes=['server1', 'server2', 'server3'])
server = ch.get_node(user_id)
```

## 容量规划

### 性能测试类型

| 类型 | 目标 | 工具 |
|------|------|------|
| **负载测试** | 验证正常负载下性能 | ab, wrk |
| **压力测试** | 确定最大承受能力 | JMeter, Gatling |
| **浸泡测试** | 长时间运行稳定性 | 自定义脚本 |
| **尖峰测试** | 突发流量响应 | 自定义脚本 |

### 性能测试工具

```bash
# Apache Bench（简单负载测试）
ab -n 10000 -c 100 http://localhost:8080/api/users

# wrk（更现代的 HTTP 基准测试）
wrk -t12 -c400 -d30s http://localhost:8080/api/users

# JMeter（复杂场景）
jmeter -n -t test_plan.jmx -l results.jtl

# locust（Python 编写测试脚本）
locust -f locustfile.py --headless -u 1000 -r 100 -t 60s
```

### 性能建模

```python
# Little's Law: L = λW
# L = 平均系统负载（请求数）
# λ = 到达率（请求/秒）
# W = 平均响应时间（秒）

# 示例：QPS=1000, P99延迟=100ms
# 平均并发 = 1000 * 0.1 = 100 个请求

# 扩展计算
current_qps = 1000
target_qps = 5000
current_latency_p99_ms = 50

# 假设延迟不随负载线性增长
# 需要服务数量 = target_qps / current_qps * 当前实例数
```

## 前端性能

### 关键指标（Core Web Vitals）

| 指标 | 含义 | 良好标准 |
|------|------|----------|
| **LCP** | 最大内容绘制 | < 2.5s |
| **FID** | 首次输入延迟 | < 100ms |
| **CLS** | 累积布局偏移 | < 0.1 |

### 前端优化

```html
<!-- 资源加载优化 -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="dns-prefetch" href="https://api.example.com">

<!-- 关键 CSS 内联 -->
<style>/* Critical CSS */</style>

<!-- 异步加载非关键资源 -->
<script defer src="analytics.js"></script>
<link rel="preload" as="image" href="hero.jpg">

<!-- 资源压缩 -->
<!-- 启用 Brotli/Gzip 压缩 -->
```

## 扩展阅读

- [Google SRE 性能章节](https://sre.google/sre-book/performance-experiment/)
- [Systems Performance](https://www.brendangregg.com-systems-performance.html)
- [Flame Graphs](https://www.brendangregg.com/flamegraphs.html)
- [WebPageTest](https://webpagetest.org/)

