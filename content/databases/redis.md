---
title: Redis Deep Dive
date: 2026-04-14
description: Redis 深度解析：数据结构、持久化与集群
tags:
  - redis
  - database
  - caching
draft: true
permalink:
---

# Redis Deep Dive

## 概述

Redis（REmote DIctionary Server）是由 Salvatore Sanfilippo 开发的高级键值存储，以高性能、丰富的数据结构和多样的持久化选项著称。广泛应用于缓存、消息队列、会话存储、实时分析等场景。

## 数据结构

Redis 支持五种基础数据结构，以及在此基础上扩展的多种特殊类型。

### 字符串（String）

最基本的类型，字节序列存储。

```bash
SET key "value"              # 设置值
GET key                      # 获取值
SETNX key "value"            # 仅当 key 不存在时设置（原子）
SETEX key 10 "value"        # 设置值并指定过期时间（秒）
PSETEX key 10000 "value"     # 设置值并指定过期时间（毫秒）
APPEND key " appended"       # 追加内容
STRLEN key                   # 获取字符串长度

# 数值操作
INCR counter                 # 原子递增
INCRBY counter 5             # 增加指定数值
DECR counter                 # 原子递减
INCRBYFLOAT counter 0.5      # 浮点数递增

# 批量操作
MSET key1 "v1" key2 "v2"     # 批量设置
MGET key1 key2               # 批量获取
```

### 列表（List）

有序字符串序列，基于双向链表实现。

```bash
LPUSH mylist "one"           # 左侧插入
RPUSH mylist "two"           # 右侧插入
LPOP mylist                  # 左侧弹出
RPOP mylist                  # 右侧弹出
LRANGE mylist 0 -1            # 获取范围元素（0 到 -1 表示全部）
LREM mylist 2 "hello"        # 移除指定元素（2 次）
LTRIM mylist 0 9             # 截断列表保留指定范围
BLPOP mylist 0               # 阻塞式左侧弹出（等待数据）
```

### 集合（Set）

无序唯一字符串集合，支持交集、并集、差集运算。

```bash
SADD myset "member1"         # 添加成员
SREM myset "member1"         # 移除成员
SMEMBERS myset               # 获取所有成员
SISMEMBER myset "member1"    # 检查成员是否存在
SCARD myset                  # 获取集合基数（成员数量）
SRANDMEMBER myset 2          # 随机获取 2 个成员（不删除）
SPOP myset                   # 随机弹出成员

# 集合运算
SINTER set1 set2             # 交集
SUNION set1 set2              # 并集
SDIFF set1 set2               # 差集（set1 有而 set2 没有的）
SINTERSTORE result set1 set2  # 交集并存储到 result
```

### 有序集合（Sorted Set / ZSet）

每个成员关联一个分数，按分数排序。

```bash
ZADD leaderboard 100 "Alice"  # 添加成员及分数
ZINCRBY leaderboard 50 "Alice" # 增加分数
ZSCORE leaderboard "Alice"     # 获取成员分数
ZRANK leaderboard "Alice"       # 获取排名（从低到高）
ZREVRANK leaderboard "Alice"   # 获取排名（从高到低）
ZRANGE leaderboard 0 9          # 获取排名 0-9 的成员
ZREVRANGE leaderboard 0 9       # 获取排名最高的前 10 名

# 按分数范围
ZRANGEBYSCORE leaderboard 0 100   # 获取 0-100 分的成员
ZCOUNT leaderboard 90 100         # 统计 90-100 分的成员数

# 集合运算
ZUNIONSTORE result 2 zset1 zset2   # 合并有序集合
ZINTERSTORE result 2 zset1 zset2  # 交集（分数相加）
```

### 哈希（Hash）

键值对集合，适合存储对象。

```bash
HSET user:1000 name "Alice" age "30"  # 设置字段
HGET user:1000 name                    # 获取字段值
HMGET user:1000 name age               # 批量获取字段
HGETALL user:1000                      # 获取所有字段值
HDEL user:1000 age                     # 删除字段
HEXISTS user:1000 name                 # 检查字段是否存在
HLEN user:1000                         # 获取字段数量
HINCRBY user:1000 login_count 1       # 字段数值递增

# 扫描操作
HSCAN user:1000 0 MATCH name*         # 渐进式扫描字段
```

## 特殊数据类型

### Bitmap

位图，适合存储布尔状态和布隆过滤器。

```bash
SETBIT user:login:2026:04:14 1001 1   # 设置第 1001 位为 1
GETBIT user:login:2026:04:14 1001     # 获取第 1001 位的值
BITCOUNT user:login:2026:04:14        # 统计设置为 1 的位数
BITOP AND result key1 key2            # 位运算 AND
```

### HyperLogLog

基数估算算法，用于统计唯一元素数量（内存极小）。

```bash
PFADD visits "2026-04-14:user1"       # 添加元素
PFCOUNT visits                        # 估算唯一元素数量
PFMERGE result hll1 hll2              # 合并多个 HyperLogLog
```

### Geospatial (GEO)

地理位置存储和计算。

```bash
GEOADD cities 116.40 39.90 "Beijing"    # 添加城市坐标
GEOPOS cities "Beijing"                 # 获取城市坐标
GEODIST cities "Beijing" "Shanghai" km  # 计算距离（公里）
GEORADIUS cities 116.40 39.90 100 km    # 查找 100km 范围内的城市
GEOSEARCH cities FROMLONLAT 116.40 39.90 BYRADIUS 100 km  # 替代 GEORADIUS
```

### Stream

事件流，支持消息队列和消费者组模式。

```bash
XADD mystream * sensor-id 1 temperature 25.5  # 添加消息（* 自动生成 ID）
XRANGE mystream - +                           # 读取所有消息
XREAD STREAMS mystream $                      # 读取新消息（$ 之后）
XREADGROUP GROUP g1 consumer1 STREAMS mystream ">"  # 消费者组读取新消息
XACK mystream g1 message-id                   # 确认消息已处理
```

## 持久化机制

### RDB（Redis Database）

定时生成数据快照，文件紧凑、恢复快速。

```bash
# 配置（redis.conf）
save 900 1        # 900 秒内 ≥1 个 key 变化则保存
save 300 10       # 300 秒内 ≥10 个 key 变化则保存
save 60 10000     # 60 秒内 ≥10000 个 key 变化则保存

# 手动触发
BGSAVE            # 后台异步保存
SAVE              # 同步保存（阻塞）

# 恢复
# Redis 启动时自动加载 dump.rdb
```

### AOF（Append Only File）

记录每次写操作命令，以日志形式持久化。

```bash
# 配置
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec   # everysec（每秒同步，推荐）/ always（每次写）/ no（操作系统决定）

# AOF 重写（压缩历史命令）
BGREWRITEAOF
```

### RDB vs AOF

| 特性 | RDB | AOF |
|------|-----|-----|
| 文件体积 | 小（紧凑）| 大（记录所有命令）|
| 恢复速度 | 快（直接加载）| 慢（重放命令）|
| 数据安全性 | 可能丢失部分数据 | 可配置（everysec/always）|
| 性能影响 | fork() 子进程 | 主线程记录，后台同步 |
| 内存占用 | 复制内存（fork 时）| 最小缓冲区 |

### 混合持久化（Redis 4.0+）

```bash
aof-use-rdb-preamble yes  # AOF 重写时先写 RDB 再追加增量
```

## 集群模式

### 主从复制（Master-Slave）

一主多从配置，提供数据冗余和读写分离。

```bash
# 从节点配置
REPLICAOF master-host master-port   # 或在 redis.conf
REPLICAOF NO ONE                      # 取消复制，变为主节点

# 复制原理
# 1. 主节点 fork 后台 RDB 快照
# 2. 主节点继续处理命令，同时将新命令写入 replication buffer
# 3. 从节点加载 RDB 后，接收增量命令
```

### 哨兵（Sentinel）

监控主从节点，自动故障转移。

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │ 获取 master 地址
       ▼
┌─────────────┐
│   Sentinel  │◀──────┐ 监控/选举
│   (Leader)  │       │
└──────┬──────┘       │
       │              │
   ┌───┴───┐         │
   │       │         │
   ▼       ▼         │
┌─────┐ ┌─────┐       │
│ M   │ │ S   │       │
└─────┘ └──┬──┘       │
           │ 心跳/复制 │
           ▼          │
        ┌─────┐       │
        │ S   │───────┘
        └─────┘
```

```bash
# 典型配置（sentinel.conf）
sentinel monitor mymaster 127.0.0.1 6379 2   # 监控主节点，2 个 sentinel 同意则故障转移
sentinel down-after-milliseconds mymaster 5000  # 5 秒无响应则认为主观下线
sentinel parallel-syncs mymaster 1              # 故障转移后同时同步的从节点数
```

### Redis Cluster

数据分片集群，支持自动分片和故障转移。

```
        ┌─────────┐
        │ Client  │
        └────┬────┘
             │
        ┌────┴────┐
        ▼         ▼
   ┌────────┐ ┌────────┐
   │ Slot 0  │ │Slot 5461│ ← 节点 1 负责 0-5460
   │ -5460  │ │  -10922 │
   └────┬────┘ └────────┘
        │
        ▼
   ┌────────┐ ┌────────┐
   │ Node 2  │ │ Node 3  │
   │ Slot    │ │ Slot    │
   │10923-   │ │16383   │
   │16383    │ │         │
   └────────┘ └────────┘
```

```bash
# 创建集群（6 节点，3 主 3 从）
redis-cli --cluster create 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 \
                           127.0.0.1:7004 127.0.0.1:7005 127.0.0.1:7006 \
                           --cluster-replicas 1

# 集群操作
CLUSTER INFO                       # 集群状态信息
CLUSTER SLOTS                      # 槽分配信息
CLUSTER NODES                      # 节点列表
CLUSTER FAILOVER                   # 手动故障转移
```

## 内存管理

### 内存淘汰策略（maxmemory-policy）

```bash
maxmemory 2gb                    # 最大内存 2GB
maxmemory-policy allkeys-lru      # 最近最少使用淘汰

# 策略类型
# - noeviction: 不淘汰，返回错误（默认）
# - allkeys-lru: 所有 key 中 LRU 淘汰
# - allkeys-random: 随机淘汰所有 key
# - volatile-lru: 有过期时间的 key 中 LRU 淘汰
# - volatile-random: 有过期时间的 key 中随机淘汰
# - volatile-ttl: 有过期时间的 key 中 TTL 最短的优先淘汰
# - allkeys-lfu: 所有 key 中最不经常使用淘汰
# - volatile-lfu: 有过期时间的 key 中 LFU 淘汰
```

### 内存碎片

```bash
MEMORY STATS                         # 详细内存统计
MEMORY DOCTOR                         # 内存诊断
MEMORY PURGE                          # 清理内存碎片（需要 Jemalloc）
```

## 缓存策略

### Cache Aside

```
读取：
1. 应用先查 Redis
2. 命中则返回
3. 未命中则查 DB
4. 结果写入 Redis
5. 返回数据

写入：
1. 更新 DB
2. 删除 Redis（而非更新）
```

### 缓存问题及解决方案

| 问题 | 描述 | 解决方案 |
|------|------|----------|
| 缓存穿透 | 查询不存在的数据 | BloomFilter / 缓存空值 |
| 缓存击穿 | 热点 key 过期导致大量请求 | 互斥锁 / 逻辑过期 |
| 缓存雪崩 | 大量 key 同时过期 | 过期时间随机 / 预热 |

```python
# 互斥锁实现
import redis
import uuid

def get_with_lock(redis_client, key, db_getter, expire=10):
    """带互斥锁的缓存读取"""
    lock_key = f"lock:{key}"
    token = str(uuid.uuid4())
    
    # 尝试获取锁
    if redis_client.set(lock_key, token, nx=True, ex=expire):
        try:
            # 获取数据
            value = db_getter()
            redis_client.setex(key, 3600, value)
            return value
        finally:
            # 释放锁
            if redis_client.get(lock_key) == token:
                redis_client.delete(lock_key)
    else:
        # 等待后重试
        import time
        time.sleep(0.1)
        return redis_client.get(key)
```

## 性能优化

### 慢查询分析

```bash
SLOWLOG GET 10              # 获取最近 10 条慢查询
SLOWLOG LEN                 # 获取慢查询队列长度
SLOWLOG RESET               # 清空慢查询队列

# 配置（redis.conf）
slowlog-log-slower-than 10000   # 超过 10ms 记录
slowlog-max-len 128              # 最多保存 128 条
```

### Pipeline

减少网络往返次数。

```python
# 普通方式：N 次网络往返
for key in keys:
    redis_client.get(key)

# Pipeline：1 次网络往返
pipe = redis_client.pipeline()
for key in keys:
    pipe.get(key)
results = pipe.execute()
```

### Lua 脚本

原子执行多条命令。

```lua
-- 库存扣减脚本
local stock = tonumber(redis.call('GET', KEYS[1]))
if stock >= tonumber(ARGV[1]) then
    redis.call('DECRBY', KEYS[1], ARGV[1])
    return stock - tonumber(ARGV[1])
else
    return -1
end
```

## 扩展阅读

- [Redis 官方文档](https://redis.io/docs/)
- [Redis 命令参考](https://redis.io/commands/)
- [Redis 设计与实现](http://redisbook.com/)
- [Redis 持久化详解](https://redis.io/topics/persistence)
- [Redis Cluster 教程](https://redis.io/topics/cluster-tutorial)

