---
title: 分布式缓存
date: 2026-04-09
description: Redis集群、缓存策略、高可用架构、缓存问题解决方案
tags:
  - redis
  - distributed-cache
  - cache
draft: false
permalink:
---

# 分布式缓存

分布式缓存是提升系统性能的核心手段。本节深入讲解Redis缓存架构、集群方案、缓存策略及常见问题解决方案。

## 1. 缓存架构设计原则

### 1.1 缓存适用场景

| 场景 | 特征 | 缓存效果 |
|------|------|----------|
| 高并发读 | 读多写少，热数据集中 | 降低DB压力90%+ |
| 热点数据 | 符合二八定律 | 显著提升响应速度 |
| 低延迟需求 | 毫秒级响应要求 | 绕过DB直连缓存 |

### 1.2 多级缓存架构

```
┌─────────────────────────────────────────────────────┐
│                     应用服务器                       │
│  ┌─────────────┐                                   │
│  │ L1本地缓存   │ ◀── 微秒级访问                    │
│  │ (Caffeine)  │                                   │
│  └──────┬──────┘                                   │
│         │ 未命中                                    │
│  ┌──────▼──────┐                                   │
│  │ L2 Redis    │ ◀── 毫秒级访问                     │
│  │   Cluster   │                                   │
│  └──────┬──────┘                                   │
│         │ 未命中                                    │
│  ┌──────▼──────┐                                   │
│  │   数据库     │                                   │
│  └─────────────┘                                   │
└─────────────────────────────────────────────────────┘
```

**设计要点**：
- L1缓存：存储热点数据，容量小（MB级），响应极快
- L2缓存：存储更大量数据，容量大（GB级），共享访问
- 读取策略：L1 → L2 → DB

---

## 2. 缓存策略

### 2.1 Cache-Aside（旁路缓存）

**最常用的缓存模式**。

```python
def get_user(user_id):
    # 1. 先查缓存
    user = redis.get(f"user:{user_id}")
    if user:
        return json.loads(user)
    
    # 2. 缓存未命中，查数据库
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # 3. 写入缓存
    if user:
        redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    
    return user

def update_user(user_id, data):
    # 1. 先更新数据库
    db.execute("UPDATE users SET ... WHERE id = ?", user_id, data)
    
    # 2. 删除缓存（而非更新）
    redis.delete(f"user:{user_id}")
```

**为什么删除而非更新？**
- 避免并发时的脏数据问题
- 更新缓存时如果线程B也更新了数据库，线程A再更新缓存会导致数据不一致

### 2.2 缓存更新策略对比

| 策略 | 写操作 | 读操作 | 一致性 | 性能 | 适用场景 |
|------|--------|--------|--------|------|----------|
| Cache-Aside | 删除缓存 | 先缓存后DB | 最终一致 | 高 | 读多写少 |
| Write-Through | 同步写缓存和DB | 直接读缓存 | 强一致 | 中 | 写多读少 |
| Write-Behind | 先写缓存异步写DB | 直接读缓存 | 最终一致 | 最高 | 高并发写入 |
| Read-Through | 缓存代理自动加载 | 缓存未命中自动查DB | 中 | 高 | 简化代码逻辑 |

### 2.3 缓存粒度控制

```python
# 粒度太粗：缓存整个用户对象
redis.setex(f"user:{user_id}", 3600, json.dumps(user_object))

# 粒度适中：只缓存需要的字段
redis.hset(f"user:{user_id}", mapping={
    "name": user.name,
    "email": user.email
})
redis.expire(f"user:{user_id}", 3600)

# 粒度设计原则
# 1. 最小可用原则：只缓存业务需要的字段
# 2. 读写匹配原则：缓存粒度与业务读写频率匹配
# 3. 避免大对象：单个Key不宜过大
```

---

## 3. Redis集群方案

### 3.1 主从复制

```bash
# 配置主从复制
replicaof master_ip 6379

# 或者在redis.conf中配置
replicaof <masterip> <masterport>
replica-read-only yes
```

**读写分离策略**：
- 主节点：写操作
- 从节点：读操作（需注意复制延迟）

### 3.2 Redis Sentinel（哨兵模式）

```bash
# 哨兵配置
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs mymaster 1
```

**哨兵职责**：
- 监控主从节点健康状态
- 自动故障转移（主节点宕机时将从节点升级）
- 通知客户端新的主节点地址

### 3.3 Redis Cluster

```
┌─────────────────────────────────────────────────────┐
│                   Redis Cluster                      │
│                                                      │
│   槽位范围: 0-5460      槽位范围: 5461-10922        │
│   ┌─────────┐         ┌─────────┐                  │
│   │ Master1 │────────│ Master2 │                  │
│   └────┬────┘         └────┬────┘                  │
│        │                    │                       │
│   ┌────▼────┐         ┌────▼────┐                  │
│   │ Replica1│         │ Replica2│                  │
│   └─────────┘         └─────────┘                  │
│                                                      │
│   槽位范围: 10923-16383                              │
│   ┌─────────┐                                       │
│   │ Master3 │                                       │
│   └────┬────┘                                       │
│   ┌────▼────┐                                       │
│   │ Replica3│                                       │
│   └─────────┘                                       │
└─────────────────────────────────────────────────────┘
```

**特点**：
- 数据分片：16384个槽位，自动分配到不同节点
- 水平扩展：增加节点即可扩容
- 高可用：主从自动故障转移

```bash
# 创建集群
redis-cli --cluster create 192.168.1.10:7000 192.168.1.10:7001 \
  192.168.1.11:7002 192.168.1.11:7003 \
  192.168.1.12:7004 192.168.1.12:7005 \
  --cluster-replicas 1

# 查看集群状态
redis-cli --cluster check 192.168.1.10:7000
```

---

## 4. 缓存三大问题

### 4.1 缓存穿透

**定义**：查询不存在的数据，绕过缓存直接打穿数据库。

```
请求 → Redis（未命中） → DB（未命中） → 返回空
              ↑
         大量无效请求穿透
```

**解决方案**：

**方案1：布隆过滤器**
```python
from bloom_filter import BloomFilter

bf = BloomFilter(capacity=1000000, error_rate=0.01)

# 添加存在的Key
bf.add("user:1")
bf.add("user:2")

# 查询
if "user:123" in bf:
    # 可能存在，继续查Redis/DB
    pass
else:
    # 一定不存在，直接返回
    return None
```

**方案2：缓存空值**
```python
def get_user(user_id):
    user = redis.get(f"user:{user_id}")
    if user is None:
        user = db.query("SELECT * FROM users WHERE id = ?", user_id)
        if user:
            redis.setex(f"user:{user_id}", 3600, json.dumps(user))
        else:
            # 缓存空值，短过期时间
            redis.setex(f"user:{user_id}", 60, "NULL")
    elif user == "NULL":
        return None
    return user
```

### 4.2 缓存击穿

**定义**：热点Key过期瞬间，大量请求同时击穿到数据库。

```
热点Key: "product:hot:1001" (已过期)
         ↓
    大量请求同时查Redis (未命中)
         ↓
    同时打向数据库 → 数据库崩溃
```

**解决方案**：

**方案1：互斥锁**
```python
import redis
import uuid

lock_key = f"lock:user:{user_id}"
lock_value = str(uuid.uuid4())

# 尝试获取锁
if redis.set(lock_key, lock_value, nx=True, ex=10):
    try:
        # 成功获取锁，查询数据库
        user = db.query(...)
        redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    finally:
        # 释放锁
        if redis.get(lock_key) == lock_value:
            redis.delete(lock_key)
else:
    # 获取锁失败，短暂等待后重试
    time.sleep(0.1)
    return redis.get(f"user:{user_id}")
```

**方案2：逻辑过期**
```python
def get_user(user_id):
    user_json = redis.get(f"user:{user_id}")
    if not user_json:
        return None
    
    user = json.loads(user_json)
    
    # 检查逻辑过期时间
    if user['logic_expire_time'] < time.time():
        # 已过期，但返回旧数据
        # 启动异步线程更新缓存
        asyncio.create_task(refresh_cache(user_id))
        return user
    
    return user
```

### 4.3 缓存雪崩

**定义**：大量Key同时过期或Redis宕机，导致数据库压力骤增。

**解决方案**：

**方案1：过期时间随机化**
```python
# 设置过期时间时加上随机偏移
base_timeout = 3600
random_offset = random.randint(0, 300)
redis.setex(key, base_timeout + random_offset, value)
```

**方案2：多级缓存**
```python
# 本地缓存作为Redis的备份
local_cache = CaffeineCache(max_size=10000)

def get_user(user_id):
    # 先查本地缓存
    user = local_cache.get(user_id)
    if user:
        return user
    
    # 再查Redis
    user = redis.get(f"user:{user_id}")
    if user:
        local_cache.set(user_id, user)
        return user
    
    # 最后查数据库
    user = db.query(...)
    if user:
        redis.setex(f"user:{user_id}", 3600, json.dumps(user))
        local_cache.set(user_id, user)
    return user
```

**方案3：限流降级**
```python
from sentinel import CircuitBreaker

breaker = CircuitBreaker(failure_threshold=5, recovery_timeout=30)

@breaker
def get_user_circuit(user_id):
    # 当Redis/DB错误率过高时，返回降级数据
    return get_user_fallback(user_id)
```

---

## 5. Redis 内存优化

### 5.1 内存淘汰策略

当Redis内存达到上限时，根据策略删除Key。

| 策略 | 作用范围 | 算法 | 适用场景 |
|------|----------|------|----------|
| noeviction | 无 | 不淘汰，返回错误 | 数据不能丢失 |
| allkeys-lru | 所有Key | LRU | 通用缓存 |
| allkeys-lfu | 所有Key | LFU | 有明显热点 |
| allkeys-random | 所有Key | 随机 | 无规律访问 |
| volatile-lru | 有TTL的Key | LRU | 缓存+持久混合 |
| volatile-lfu | 有TTL的Key | LFU | TTL数据有热点 |
| volatile-random | 有TTL的Key | 随机 | - |
| volatile-ttl | 有TTL的Key | TTL最小 | 优先删除快过期 |

```bash
# 配置
maxmemory 4gb
maxmemory-policy allkeys-lfu
```

### 5.2 Big Key处理

```bash
# 查找Big Key
redis-cli --bigkeys

# 扫描大Key
redis-cli --scan --pattern '*' | while read key; do
    size=$(redis-cli memory usage "$key")
    if [ $size -gt 10485760 ]; then  # > 10MB
        echo "$key: $size bytes"
    fi
done
```

### 5.3 内存碎片优化

```bash
# 内存碎片率
redis-cli info memory | grep mem_fragmentation_ratio

# 碎片率 > 1.5 时需要优化
# 方案1：重启Redis
# 方案2：设置activedefrag yes
```

---

## 6. Redis持久化

### 6.1 RDB（快照）

```bash
# 配置
save 3600 1      # 1小时内有1次写入
save 300 100     # 5分钟内有100次写入
save 60 10000    # 1分钟内有10000次写入

# 手动触发
redis-cli save  # 同步（阻塞）
redis-cli bgsave  # 异步（非阻塞）
```

**优点**：恢复速度快
**缺点**：可能丢失最后一次快照后的数据

### 6.2 AOF（追加日志）

```bash
# 配置
appendonly yes
appendfsync everysec  # 每秒刷盘，性能与安全性平衡
# appendfsync always  # 每次写都刷盘，最安全但最慢
# appendfsync no       # 由OS决定何时刷盘
```

**优点**：数据安全性更高
**缺点**：文件较大，恢复较慢

### 6.3 混合持久化（Redis 4.0+）

```bash
# 开启混合持久化
aof-use-rdb-preamble yes
```

**原理**：AOF重写时使用RDB格式开头，之后用AOF格式追加

---

## 7. 高可用架构实践

### 7.1 生产环境Redis架构

```
                    ┌─────────────────┐
                    │   负载均衡器     │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────▼─────┐        ┌────▼─────┐        ┌────▼─────┐
   │ 应用节点1 │        │ 应用节点2 │        │ 应用节点3 │
   └────┬─────┘        └────┬─────┘        └────┬─────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
         ┌────▼────┐   ┌────▼────┐   ┌────▼────┐
         │ Master1 │   │ Master2 │   │ Master3 │
         └────┬────┘   └────┬────┘   └────┬────┘
              │             │             │
         ┌────▼────┐   ┌────▼────┐   ┌────▼────┐
         │ Replica1│   │ Replica2│   │ Replica3│
         └─────────┘   └─────────┘   └─────────┘
              │             │             │
         ┌────▼────┐   ┌────▼────┐   ┌────▼────┐
         │ Sentinel│   │ Sentinel│   │ Sentinel│
         └─────────┘   └─────────┘   └─────────┘
```

### 7.2 监控指标

| 指标 | 含义 | 健康阈值 |
|------|------|----------|
| used_memory | Redis已分配内存 | < maxmemory的90% |
| keyspace_hits | 缓存命中次数 | 命中率 > 80% |
| keyspace_misses | 缓存未命中次数 | - |
| connected_clients | 客户端连接数 | < maxclients的80% |
| mem_fragmentation_ratio | 内存碎片率 | < 1.5 |
| replication_lag | 主从复制延迟 | < 1秒 |

```bash
# 查看关键指标
redis-cli info stats | grep -E "keyspace|cmdstat"
redis-cli info memory | grep -E "used|peak|maxmemory"
```

---

## 8. 参考资料

[^1]: [Redis Official Documentation](https://redis.io/documentation)
[^2]: [Redis Cluster Specification](https://redis.io/topics/cluster-spec)
[^3]: [阿里云Redis开发规范](https://developer.aliyun.com/article/1702414)
[^4]: 《Redis深度历险》- 钱文品

---

## 相关主题

- [[../distributed-systems/distributed-systems-overview|分布式系统基础]]
- [[../distributed-systems/message-queue|消息队列系统]]
- [[../tools/redis|Redis快速入门]]
