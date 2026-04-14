---
title: 系统设计基础
date: 2026-04-06
description: 系统设计的核心概念、架构模式与实践指南
tags:
  - system-design
  - architecture
  - scalability
draft: false
permalink:
---

# 系统设计基础

系统设计是软件工程中用于构建可靠、高效、可扩展软件系统的核心学科。本文将介绍系统设计的基础知识、核心概念和实用模式。

## 1. 系统设计基础

### 1.1 什么是系统设计

系统设计（System Design）是指在构建软件系统之前，对系统的架构、组件、数据流和交互方式进行规划和设计的过程。良好的系统设计能够：

- **支撑业务增长**：系统能够应对用户量和数据量的增长
- **保障服务稳定**：具备容错能力，减少单点故障影响
- **优化资源效率**：合理分配计算、存储和网络资源
- **降低维护成本**：模块化设计便于后续迭代和扩展

系统设计通常发生在项目早期，需要在**性能**、**成本**、**复杂度**和**可维护性**之间取得平衡。

### 1.2 性能 vs 可扩展性 vs 可用性

| 指标 | 定义 | 关注点 |
|------|------|--------|
| **性能（Performance）** | 系统响应速度和吞吐量 | 延迟、QPS、TPS |
| **可扩展性（Scalability）** | 系统处理增长负载的能力 | 水平扩展、垂直扩展 |
| **可用性（Availability）** | 系统正常运行的时间比例 | 99.9%、99.99%、99.999% |

**三者之间的权衡**：

```
可用性 99.9%  = 每年停机约 8.7 小时
可用性 99.99% = 每年停机约 52.6 分钟
可用性 99.999% = 每年停机约 5.3 分钟
```

在实际设计中，我们通常无法同时最大化这三个指标，需要根据业务场景做出权衡。例如：金融系统更注重一致性（Consistency），而社交媒体可能更注重可用性（Availability）。

### 1.3 一致性模型

一致性模型定义了分布式系统中数据同步和读取的规则。

#### 强一致性（Strong Consistency）

**定义**：任何读取操作都会返回最近一次写入的结果。

**特点**：
- 所有节点的数据完全一致
- 写入后立即可读
- 通常通过分布式锁或共识协议实现

**适用场景**：金融交易、库存管理、订单处理

**示意图**：

```
客户端A --写入--> 节点1 --同步--> 节点2 --同步--> 节点3
        |                              |                              |
        v                              v                              v
     读取返回                    读取返回                      读取返回
     相同结果                    相同结果                      相同结果
```

#### 最终一致性（Eventual Consistency）

**定义**：写入操作后，数据最终会在所有节点上达到一致状态，但短期内可能存在不一致。

**特点**：
- 写入后可立即读取，但可能读到旧值
- 系统会异步同步数据
- 冲突解决策略（如最后写入获胜）

**适用场景**：社交媒体帖子、评论、缓存更新

#### 弱一致性（Weak Consistency）

**定义**：读取操作不保证能读取到最新的写入结果。

**特点**：
- 无法预测何时会读到新值
- 性能最高但一致性最弱
- 通常用于实时性要求高的场景

**示例**：DNS 解析、视频直播弹幕

**一致性模型对比**：

| 特性 | 强一致性 | 最终一致性 | 弱一致性 |
|------|---------|-----------|---------|
| 读取延迟 | 高 | 中 | 低 |
| 写入延迟 | 高 | 中 | 低 |
| 数据新鲜度 | 始终最新 | 最终最新 | 不保证 |
| 实现复杂度 | 高 | 中 | 低 |

---

## 2. 核心概念

### 2.1 负载均衡（Load Balancing）

负载均衡将传入的网络流量分配到多个服务器，以优化资源使用和提高吞吐量。

#### 负载均衡算法

**1. 轮询（Round Robin）**

```cpp
// 简单轮询实现
int current_index = 0;
vector<Server> servers = {server1, server2, server3};

Server get_next_server() {
    Server selected = servers[current_index];
    current_index = (current_index + 1) % servers.size();
    return selected;
}
```

**特点**：
- 适用于服务器性能相近的场景
- 无状态，实现简单
- 无法感知服务器实际负载

**2. 最少连接（Least Connections）**

```cpp
// 最少连接算法
struct Server {
    string address;
    int active_connections;
    // ...
};

Server select_server(vector<Server>& servers) {
    // 选择活动连接数最少的服务器
    auto it = min_element(servers.begin(), servers.end(),
        [](const Server& a, const Server& b) {
            return a.active_connections < b.active_connections;
        });
    return *it;
}
```

**特点**：
- 动态感知服务器负载
- 适合处理请求耗时差异大的场景
- 需要维护连接状态

**3. IP 哈希（IP Hash）**

```cpp
// IP哈希算法
Server select_server(vector<Server>& servers, const string& client_ip) {
    // 对客户端IP进行哈希
    hash<string> hasher;
    size_t hash_value = hasher(client_ip);
    
    // 取模选择服务器
    int index = hash_value % servers.size();
    return servers[index];
}
```

**特点**：
- 同一客户端IP始终路由到同一服务器
- 有利于会话保持（Session Persistence）
- 适合需要用户亲和性的场景

#### 负载均衡器类型

| 类型 | 工作层 | 示例 |
|------|--------|------|
| **L4 负载均衡** | 传输层（TCP/UDP） | HAProxy、LVS |
| **L7 负载均衡** | 应用层（HTTP/HTTPS） | Nginx、AWS ALB |

### 2.2 缓存（Caching）

缓存是提高系统性能的核心技术，通过存储热点数据减少数据库访问。

#### 缓存策略

**1. Cache-Aside（旁路缓存）**

```
读操作：
1. 应用先查缓存
2. 缓存命中则直接返回
3. 缓存未命中，查数据库
4. 将结果写入缓存，返回

写操作：
1. 先更新数据库
2. 再删除（而非更新）缓存
```

```python
# Cache-Aside 模式伪代码
def get_user(user_id):
    # 1. 尝试从缓存获取
    user = redis.get(f"user:{user_id}")
    if user:
        return user
    
    # 2. 缓存未命中，从数据库获取
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # 3. 写入缓存，设置过期时间
    redis.setex(f"user:{user_id}", 3600, user)
    return user

def update_user(user_id, data):
    # 1. 先更新数据库
    db.execute("UPDATE users SET ... WHERE id = ?", user_id, data)
    
    # 2. 删除缓存（而非更新）
    redis.delete(f"user:{user_id}")
```

**2. Write-Through（写穿透）**

数据同时写入缓存和数据库，保证强一致性，但写入性能较低。

**3. Write-Behind（写回）**

先写入缓存，异步批量写入数据库。性能高但存在数据丢失风险。

#### 缓存淘汰策略

| 策略 | 原理 | 适用场景 |
|------|------|----------|
| **LRU** | 最近最少使用 | 通用场景 |
| **LFU** | 最不频繁使用 | 访问频率差异大 |
| **FIFO** | 先进先出 | 简单场景 |

#### Redis vs Memcached

| 特性 | Redis | Memcached |
|------|-------|-----------|
| 数据结构 | String, Hash, List, Set, Sorted Set | 仅 String |
| 持久化 | 支持 RDB、AOF | 不支持 |
| 集群 | 原生支持集群模式 | 需要客户端分片 |
| 内存管理 | 支持虚拟内存 | 纯内存 |

### 2.3 数据库分片（Database Sharding）

分片是将数据水平拆分到多个数据库/表的技术。

```
┌─────────────────────────────────────────┐
│              用户表 (10亿条)              │
└─────────────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐
   │ Shard 1 │ │ Shard 2 │ │ Shard 3 │
   │ user_id │ │ user_id │ │ user_id │
   │ % 3 = 0 │ │ % 3 = 1 │ │ % 3 = 2 │
   └─────────┘ └─────────┘ └─────────┘
```

#### 分片策略

**1. 哈希分片（Hash Sharding）**

```cpp
// 哈希分片
int get_shard(long user_id, int num_shards) {
    return user_id % num_shards;
}
```

**优点**：数据分布均匀
**缺点**：扩容困难（需要重新哈希所有数据）

**2. 范围分片（Range Sharding）**

```cpp
// 范围分片（按用户ID范围）
int get_shard(long user_id) {
    if (user_id < 1000000000) return 0;  // 1-10亿
    if (user_id < 2000000000) return 1;  // 10-20亿
    return 2;
}
```

**优点**：适合范围查询
**缺点**：可能产生热点

### 2.4 索引（Indexing）

索引是加速数据库查询的数据结构。

```sql
-- 创建索引
CREATE INDEX idx_user_email ON users(email);
CREATE INDEX idx_order_date_status ON orders(create_date, status);

-- 复合索引遵循最左前缀原则
-- 索引 (a, b, c) 可以加速：
-- - 查询 a
-- - 查询 a, b
-- - 查询 a, b, c
-- 但无法加速仅查询 b 或 c 的查询
```

**B+树 vs 哈希索引**：

| 特性 | B+树索引 | 哈希索引 |
|------|---------|---------|
| 查询类型 | 范围查询 (>, <, BETWEEN) | 等值查询 (=) |
| 排序 | 支持 | 不支持 |
| 内存占用 | 较高 | 较低 |
| 适用场景 | 大多数场景 | 简单等值查询 |

### 2.5 CDN（Content Delivery Network）

CDN 通过在全球部署边缘节点，加速内容分发。

```
用户请求 → CDN边缘节点 → 缓存命中返回
                │
                └→ 缓存未命中 → 源站获取 → 缓存 → 返回
```

**CDN 的优势**：
- **低延迟**：内容就近返回
- **减轻源站压力**：静态资源由CDN承担
- **提高可用性**：分布式架构避免单点故障
- **安全防护**：提供DDoS防护、WAF等功能

### 2.6 DNS（Domain Name System）

DNS 将域名解析为IP地址，是互联网的基础设施。

```
用户浏览器 → DNS解析请求 → DNS服务器 → 返回IP地址
```

**DNS 记录类型**：

| 类型 | 用途 | 示例 |
|------|------|------|
| A | IPv4地址 | example.com → 93.184.216.34 |
| AAAA | IPv6地址 | example.com → 2606:2800:220:1:: |
| CNAME | 域名别名 | www.example.com → example.com |
| MX | 邮件服务器 | example.com → mail.example.com |

**DNS 负载均衡**：
- 轮询返回多个IP
- 地理位置感知解析
- 动态健康检查

---

## 3. 架构模式

### 3.1 单体架构 vs 微服务

#### 单体架构（Monolithic Architecture）

```
┌──────────────────────────────┐
│         单体应用               │
│  ┌─────┐ ┌─────┐ ┌─────┐    │
│  │ 用户 │ │ 订单 │ │ 支付 │    │
│  │ 服务 │ │ 服务 │ │ 服务 │    │
│  └─────┘ └─────┘ └─────┘    │
│  ┌─────────────────────┐    │
│  │      数据库          │    │
│  └─────────────────────┘    │
└──────────────────────────────┘
```

**优点**：
- 开发、部署简单
- 团队协作门槛低
- 测试容易

**缺点**：
- 代码耦合度高
- 部署粒度粗
- 技术栈单一

#### 微服务架构（Microservices Architecture）

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│ 用户服务  │  │ 订单服务  │  │ 支付服务  │
│  :8001   │  │  :8002   │  │  :8003   │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │              │              │
     └──────────────┼──────────────┘
                    │
             ┌──────┴──────┐
             │  API Gateway │
             └─────────────┘
```

**优点**：
- 服务独立部署
- 技术栈灵活
- 故障隔离

**缺点**：
- 复杂度高（服务间通信、分布式事务）
- 运维挑战大
- 数据一致性困难

### 3.2 事件驱动架构（Event-Driven Architecture）

事件驱动架构通过事件的产生、消费和处理来实现系统解耦。

```
┌─────────┐    事件    ┌─────────┐    事件    ┌─────────┐
│ 生产者  │ ────────→ │  消息队列 │ ←─────── │ 消费者  │
└─────────┘            └─────────┘            └─────────┘
```

```cpp
// 事件定义示例
struct OrderCreatedEvent {
    string order_id;
    string user_id;
    double amount;
    long timestamp;
};

// 生产者：发布事件
void publish_order_created(const OrderCreatedEvent& event) {
    string payload = json_encode(event);
    mq.publish("order.events", payload);
}

// 消费者：订阅并处理事件
void on_order_created(const string& payload) {
    auto event = json_decode<OrderCreatedEvent>(payload);
    // 发送通知、更新库存、记录日志等
}
```

### 3.3 CQRS（Command Query Responsibility Segregation）

CQRS 将读操作和写操作分离到不同的模型。

```
┌─────────────┐         ┌─────────────┐
│   Commands  │         │    Queries   │
│   (写操作)   │         │   (读操作)    │
└──────┬──────┘         └──────┬──────┘
       │                       │
       ▼                       ▼
┌─────────────┐         ┌─────────────┐
│    Command  │         │    Query    │
│    Model    │ ─同步──→│   Model     │
│  (写入优化)  │         │  (读取优化)  │
└─────────────┘         └─────────────┘
```

**适用场景**：
- 读写负载差异大
- 需要独立优化读写性能
- 复杂业务域

### 3.4 Saga 模式

Saga 用于管理分布式事务，将大事务拆分为多个本地事务。

#### 编排式 Saga（Choreography）

```
      ┌──────┐       ┌──────┐       ┌──────┐
      │订单  │       │库存  │       │支付  │
      │服务  │       │服务  │       │服务  │
      └──┬───┘       └──┬───┘       └──┬───┘
         │              │              │
         │ 创建订单 ──→ │              │
         │              │ 扣减库存 ──→ │
         │              │              │ 扣款
         │              │              │
         │ ←────────────┘              │
         │     失败回滚                │
```

#### 编排器 Saga（Orchestration）

```
┌─────────────────────────────────────┐
│           Saga 编排器                │
│                                     │
│  1. 调用订单服务创建订单              │
│  2. 调用库存服务扣减库存              │
│  3. 调用支付服务扣款                 │
│  4. 若任一步骤失败，执行补偿事务      │
└─────────────────────────────────────┘
```

---

## 4. 可扩展性设计

### 4.1 水平扩展 vs 垂直扩展

| 扩展方式 | 原理 | 优点 | 缺点 |
|----------|------|------|------|
| **垂直扩展（Scale Up）** | 增加单机硬件资源 | 简单、无代码改动 | 有物理上限、成本非线性 |
| **水平扩展（Scale Out）** | 增加机器数量 | 无理论上限 | 复杂度高、需要数据分片 |

**实际建议**：
- 优先垂直扩展，成本效益高
- 达到瓶颈后转向水平扩展
- 数据库通常先垂直扩展，后考虑分片

### 4.2 数据分区（Partitioning）

数据分区将数据拆分到多个物理存储单元。

**按键分区（Key-based Partitioning）**

```cpp
// 哈希分区
int partition(const string& key, int num_partitions) {
    return hash(key) % num_partitions;
}
```

**按范围分区（Range Partitioning）**

```cpp
// 按时间范围分区
string partition_by_date(long timestamp) {
    time_t t = timestamp;
    struct tm* tm_info = localtime(&t);
    char buffer[32];
    strftime(buffer, 32, "%Y%m", tm_info);
    return string(buffer);  // "202604"
}
```

### 4.3 复制（Replication）

复制将数据同步到多个节点，提高可用性和读取性能。

```
┌─────────────────────────────────────────┐
│              主从复制                     │
├─────────────────────────────────────────┤
│                                         │
│    写入 ──→ 主库 ──→ 从库1                │
│                  └──→ 从库2                │
│                  └──→ 从库3                │
│                                         │
└─────────────────────────────────────────┘
```

**复制模式**：

| 模式 | 同步方式 | 一致性 | 延迟 |
|------|----------|--------|------|
| **同步复制** | 写入所有副本后返回 | 强一致 | 高 |
| **异步复制** | 写入主库后立即返回 | 最终一致 | 低 |
| **半同步** | 写入主库+至少一个从库 | 弱强一致 | 中 |

### 4.4 一致性哈希（Consistent Hashing）

一致性哈希用于在分布式系统中均匀分配数据，同时最小化数据迁移。

```
                    ┌─────────┐
              ┌────→│  节点A  │←────┐
              │     └─────────┘     │
              │                     │
    ┌─────────┼─────────┐           │
    │         │         │           │
    ▼         ▼         ▼           │
┌───────┐ ┌───────┐ ┌───────┐     ┌─────────┐
│  0-90° │ │90-180°│ │180-270│ ... │330-360° │
└───────┘ └───────┘ └───────┘     └─────────┘
```

**虚拟节点**：为每个物理节点创建多个虚拟节点，提高负载均衡性。

```cpp
// 一致性哈希实现
class ConsistentHash {
    // 物理节点
    vector<string> physical_nodes;
    // 虚拟节点数量
    int virtual_nodes_per_node;
    // 哈希环
    map<size_t, string> hash_ring;

public:
    ConsistentHash(const vector<string>& nodes, int vnodes = 150) {
        physical_nodes = nodes;
        virtual_nodes_per_node = vnodes;
        
        for (const auto& node : nodes) {
            add_node(node);
        }
    }
    
    void add_node(const string& node) {
        for (int i = 0; i < virtual_nodes_per_node; i++) {
            string vnode = node + "#" + to_string(i);
            size_t hash_val = hash(vnode);
            hash_ring[hash_val] = node;
        }
    }
    
    string get_node(const string& key) {
        if (hash_ring.empty()) return "";
        
        size_t hash_val = hash(key);
        auto it = hash_ring.lower_bound(hash_val);
        
        if (it == hash_ring.end()) {
            it = hash_ring.begin();  // 环绕到开头
        }
        
        return it->second;
    }
};
```

---

## 5. 高可用设计

### 5.1 冗余与故障转移

冗余通过部署多个副本确保单点故障不影响整体服务。

```
┌─────────────┐      ┌─────────────┐
│   主节点    │ ←──→ │   备节点    │
│  (Active)   │ 心跳  │  (Standby)  │
└──────┬──────┘      └─────────────┘
       │
       ▼
  故障检测 → 自动切换 → 流量切换
```

**故障转移类型**：

| 类型 | 原理 | RTO | 数据丢失 |
|------|------|-----|---------|
| **自动故障转移** | 系统自动检测并切换 | 短 | 取决于复制模式 |
| **手动故障转移** | 人工介入决策 | 较长 | 可能更少 |

### 5.2 降级（Degradation）

降级是在系统压力过大时，主动放弃部分功能以保证核心功能可用。

**降级策略示例**：

```
正常情况：
┌──────────────────────────────────┐
│         完整的商品详情页           │
│  标题、价格、库存、评论、推荐、评分  │
└──────────────────────────────────┘

降级情况：
┌──────────────────────────────────┐
│         降级的商品详情页           │
│       标题、价格、库存              │
│       (评论、推荐、评分暂时关闭)     │
└──────────────────────────────────┘
```

**降级实现**：

```cpp
// 功能降级装饰器
class FeatureFlag {
public:
    static bool is_enabled(const string& feature) {
        // 检查开关配置
        return config.get_bool("feature." + feature, true);
    }
};

string get_product_detail(long product_id) {
    ProductDetail detail;
    
    // 基础功能始终可用
    detail.basic_info = get_basic_info(product_id);
    
    // 可选功能根据开关决定
    if (FeatureFlag::is_enabled("product.comments")) {
        detail.comments = get_comments(product_id);
    }
    
    if (FeatureFlag::is_enabled("product.recommendations")) {
        detail.recommendations = get_recommendations(product_id);
    }
    
    return detail;
}
```

### 5.3 熔断器（Circuit Breaker）

熔断器防止级联故障，当下游服务持续失败时快速失败。

```
        正常           半开            打开
      ┌──────┐       ┌──────┐       ┌──────┐
      │  ●   │       │  ◐   │       │  ○   │
      │ 闭合  │ ───→ │ 半开  │ ───→ │ 打开  │
      └──────┘ 失败   └──────┘ 超时  └──────┘
        │              │              │
        │              ↓              │
        │           尝试请求           │
        │              │              │
        │              ↓              │
        └──────── 成功 ────────────────┘
```

```cpp
class CircuitBreaker {
    enum State { CLOSED, OPEN, HALF_OPEN };
    
    State state = CLOSED;
    int failure_count = 0;
    int success_count = 0;
    const int threshold = 5;  // 失败阈值
    const int recovery_timeout = 60;  // 恢复尝试间隔(秒)
    time_t last_failure_time;
    
public:
    bool allow_request() {
        switch (state) {
            case CLOSED:
                return true;
            case OPEN:
                if (time(nullptr) - last_failure_time > recovery_timeout) {
                    state = HALF_OPEN;
                    return true;
                }
                return false;
            case HALF_OPEN:
                return true;  // 允许单个请求测试
        }
        return false;
    }
    
    void record_success() {
        if (state == HALF_OPEN) {
            success_count++;
            if (success_count >= 2) {
                state = CLOSED;
                failure_count = 0;
                success_count = 0;
            }
        } else {
            failure_count = 0;
        }
    }
    
    void record_failure() {
        last_failure_time = time(nullptr);
        failure_count++;
        
        if (state == HALF_OPEN) {
            state = OPEN;  // 回到打开状态
        } else if (failure_count >= threshold) {
            state = OPEN;  // 熔断打开
        }
    }
};
```

### 5.4 健康检查

健康检查用于监控服务状态，及时发现和处理故障。

```yaml
# Kubernetes 健康检查配置
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 15
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

**健康检查端点实现**：

```cpp
// 健康检查端点
void handle_health_check(evhtp_request_t *req) {
    HealthStatus status;
    
    // 检查依赖服务
    status.database = check_database() ? "healthy" : "unhealthy";
    status.cache = check_redis() ? "healthy" : "unhealthy";
    status.external_api = check_external_api() ? "healthy" : "unhealthy";
    
    // 聚合健康状态
    bool overall_healthy = (status.database == "healthy") &&
                           (status.cache == "healthy");
    
    string response = json_encode(status);
    
    if (overall_healthy) {
        evbuffer_add_printf(req->buffer_out, "%s", response.c_str());
        evhtp_send_reply(req, EVHTP_RES_OK);
    } else {
        evbuffer_add_printf(req->buffer_out, "%s", response.c_str());
        evhtp_send_reply(req, EVHTP_RES_SVCUNAVAIL);
    }
}
```

---

## 6. 案例分析

### 6.1 设计短链接系统

#### 需求分析

| 功能 | 要求 |
|------|------|
| 生成短链接 | 接受原始URL，返回短码 |
| 访问跳转 | 访问短链接时重定向到原始URL |
| 容量 | 支持1000亿URL |
| 性能 | 延迟 < 10ms |

#### 架构设计

```
用户请求短链接
      │
      ▼
┌─────────────┐     ┌─────────────┐
│   API服务   │ ──→ │   缓存层    │
└──────┬──────┘     │  (Redis)    │
       │            └─────────────┘
       │ yes 缓存命中
       │      │
       │      ▼
       │   直接返回
       │
       │ no 缓存未命中
       │      │
       ▼      ▼
┌─────────────────┐
│   存储层        │
│ (分布式数据库)  │
└─────────────────┘
```

#### 短码生成算法

```cpp
// 62进制编码
class ShortCodeGenerator {
    const string chars = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    
    string to_base62(long n) {
        string result;
        while (n > 0) {
            result = chars[n % 62] + result;
            n /= 62;
        }
        return result;
    }
    
    // 使用分布式ID生成器（如Snowflake）的ID
    string generate(long distributed_id) {
        return to_base62(distributed_id);
    }
};
```

#### 存储设计

```sql
-- 短链接表
CREATE TABLE short_urls (
    short_code VARCHAR(12) PRIMARY KEY,
    original_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    click_count BIGINT DEFAULT 0,
    INDEX idx_created_at (created_at),
    INDEX idx_expires_at (expires_at)
) ENGINE=InnoDB;

-- 分片策略：按short_code哈希分片
-- 扩展时：一致性哈希重新分配
```

### 6.2 设计分布式锁

#### 分布式锁的要求

| 要求 | 说明 |
|------|------|
| **互斥** | 任意时刻只有一个客户端能持有锁 |
| **死锁避免** | 锁最终可释放（TTL机制） |
| **容错** | 节点故障时锁能自动释放 |
| **公平** | 按请求顺序获取锁（可选） |

#### Redis 分布式锁实现

```cpp
class DistributedLock {
    RedisClient& redis;
    string lock_key;
    string lock_value;  // 唯一标识，用于安全释放
    int ttl_seconds;
    
public:
    DistributedLock(RedisClient& r, const string& key, int ttl = 30)
        : redis(r), lock_key(key), ttl_seconds(ttl) {
        // 生成唯一值（UUID + 时间戳）
        lock_value = generate_unique_id();
    }
    
    // 获取锁
    bool acquire() {
        // SET key value NX EX seconds
        string result = redis.command(
            "SET %s %s NX EX %d",
            lock_key.c_str(), lock_value.c_str(), ttl_seconds
        );
        return result == "OK";
    }
    
    // 释放锁（Lua脚本保证原子性）
    bool release() {
        // 只有持有者才能释放
        const char* lua_script = R"(
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
        )";
        string result = redis.eval(lua_script, 1, lock_key, lock_value);
        return result == "1";
    }
    
    // 续期（ watchdog 机制）
    bool extend(int additional_seconds) {
        const char* lua_script = R"(
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('expire', KEYS[1], ARGV[2])
            else
                return 0
            end
        )";
        string result = redis.eval(lua_script, 1, lock_key, lock_value, additional_seconds);
        return result == "1";
    }
};
```

#### Redlock 算法

```
获取当前时间戳(ms)
for 每个Redis节点:
    尝试获取锁，记录耗时
    若成功且耗时 < 超时时间，则锁获取成功

若成功节点数 > N/2 + 1:
    计算锁的有效期 = 初始TTL - 获取耗时
    返回成功
否则:
    释放所有节点的锁
    返回失败
```

### 6.3 设计消息队列

#### 消息队列的核心功能

| 功能 | 说明 |
|------|------|
| **异步通信** | 生产者和消费者解耦 |
| **削峰填谷** | 缓解流量洪峰 |
| **可靠传输** | 消息不丢失 |
| **顺序保证** | 按序处理（可选） |

#### 架构设计

```
┌─────────┐    生产消息     ┌─────────┐
│ 生产者1 │ ─────────────→ │         │
├─────────┤                 │         │
│ 生产者2 │ ─────────────→ │  Broker  │
├─────────┤                 │         │
│ 生产者3 │ ─────────────→ │         │
└─────────┘                 └────┬────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
        ┌──────────┐       ┌──────────┐       ┌──────────┐
        │ 消费者1  │       │ 消费者2  │       │ 消费者3  │
        │ Topic A  │       │ Topic B  │       │ Topic C  │
        └──────────┘       └──────────┘       └──────────┘
```

#### 消息持久化

```cpp
// 消息结构
struct Message {
    string id;           // 全局唯一ID
    string topic;         // 主题
    string body;          // 消息内容
    int partition;        // 分区
    long timestamp;       // 时间戳
    map<string, string> headers;  // 扩展头
};

// 消息存储（追加写 WAL）
class MessageStore {
    vector<Partition> partitions;
    
    bool append(const Message& msg) {
        int partition = select_partition(msg.topic, msg.key);
        Partition& p = partitions[partition];
        
        // 写入内存缓冲
        p.buffer.push_back(msg);
        
        // 异步刷盘
        if (should_flush()) {
            flush_to_disk(p);
        }
        
        // 复制到副本
        replicate_to_followers(p, msg);
        
        return true;
    }
};
```

#### 消费语义

| 语义 | 说明 | 实现难度 |
|------|------|---------|
| **At Most Once** | 消息最多投递一次 | 简单 |
| **At Least Once** | 消息至少投递一次 | 中等 |
| **Exactly Once** | 消息恰好投递一次 | 困难 |

---

## 参考资料

[^1]: 《Designing Data-Intensive Applications》 - Martin Kleppmann
[^2]: 《System Design Interview》 - Alex Xu
[^3]: [CQRS Pattern - Microsoft](https://learn.microsoft.com/en-us/architecture/patterns/cqrs)
[^4]: [Redis Distributed Lock](https://redis.io/topics/distlock)
[^5]: [Kafka Documentation](https://kafka.apache.org/documentation/)

---

## 相关主题

- [[microservices|微服务架构]]
- [[distributed-systems|分布式系统]]
- [[database-sharding|数据库分片]]
- [[../database/mysql-performance|MySQL性能优化]]
