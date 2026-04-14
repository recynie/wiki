---
title: NoSQL数据库
date: 2026-04-11
description: NoSQL数据库分类、CAP定理、MongoDB/Redis/Cassandra对比
tags:
  - nosql
  - database
  - distributed-systems
draft: false
permalink:
---

## 概述

NoSQL（Not Only SQL）数据库是为了解决关系型数据库在**大规模数据扩展**、**灵活schema**和**高性能**方面的局限性而设计的数据库管理系统。它们放弃了传统RDBMS的ACID事务模型，转而追求更高的可用性和可扩展性。[^1]

NoSQL数据库通常具有以下特征：
- **灵活的Schema**：无需预定义表结构
- **水平扩展**：支持数据分片和分布式集群
- **BASE语义**：Basically Available, Soft state, Eventually consistent
- **高性能**：针对特定访问模式优化

## CAP定理

CAP定理是NoSQL数据库设计的理论基础，由Eric Brewer提出。[^2]

### 定理内容

CAP代表三个性质：

| 性质 | 含义 |
|------|------|
| **Consistency（一致性）** | 所有节点在同一时刻看到相同的数据 |
| **Availability（可用性）** | 每个请求都能得到响应（成功或失败） |
| **Partition Tolerance（分区容错）** | 系统在网络分区时仍能继续运行 |

**核心结论**：在分布式系统中，网络分区不可避免，因此**必须在一致性和可用性之间做出选择**。

### 经典分类

```
            CA
           /  \
          /    \
        CP      PA
       /          \
      /            \
   MongoDB      Cassandra
   HBase         DynamoDB
   Redis         CouchDB
```

| 类型 | 描述 | 代表数据库 |
|------|------|-----------|
| **CP（一致+分区容错）** | 网络分区时牺牲可用性 | MongoDB、HBase、Redis |
| **AP（可用+分区容错）** | 网络分区时牺牲一致性 | Cassandra、DynamoDB、Couchbase |
| **CA（一致+可用）** | 仅在无分区时成立（单节点RDBMS） | MySQL、PostgreSQL |

### 一致性级别

现代NoSQL数据库通常提供**可调一致性**：

```sql
-- Cassandra 可调一致性
CONSISTENCY ONE;   -- 最低延迟，任一副本确认即可
CONSISTENCY QUORUM; -- 多数节点确认
CONSISTENCY ALL;    -- 全部副本确认，最高一致性
```

```javascript
// MongoDB 读写关注点
db.collection.findOne(
    { _id: 1 },
    { readConcern: "majority" }  // 读取已复制到多数节点的数据
)
db.collection.update(
    { _id: 1 },
    { $set: { status: "active" } },
    { writeConcern: { w: "majority" } }  // 写入多数节点确认
)
```

## NoSQL数据库分类

### 1. 文档数据库（Document Store）

以**JSON/BSON文档**为存储单位，适合灵活结构的数据。

#### MongoDB

MongoDB是最流行的文档数据库，使用BSON格式存储文档。[^3]

**核心概念**：
- **Database** → **Collection** → **Document**
- Document等价于JSON对象，支持嵌套结构
- 集合不需要预定义schema

**应用场景**：
- 内容管理系统（CMS）
- 用户配置文件存储
- 实时分析数据

**CRUD操作**：

```javascript
// 插入文档
db.users.insertOne({
    name: "Alice",
    email: "alice@example.com",
    tags: ["python", "mongodb"],
    address: {
        city: "Beijing",
        district: "Haidian"
    }
})

// 查询文档
db.users.find({
    tags: "python",
    "address.city": "Beijing"
}).sort({ created: -1 }).limit(10)

// 更新文档
db.users.updateOne(
    { _id: ObjectId("...") },
    { 
        $set: { status: "active" },
        $push: { tags: "developer" }
    }
)

// 聚合管道
db.orders.aggregate([
    { $match: { year: 2024 } },
    { $group: { _id: "$category", total: { $sum: "$amount" } } },
    { $sort: { total: -1 } }
])
```

### 2. 键值数据库（Key-Value Store）

最简单的NoSQL类型，以键值对形式存储数据，提供极高的读写性能。

#### Redis

Redis是内存键值数据库，支持多种数据结构。[^4]

**支持的数据结构**：

| 类型 | 示例 | 应用场景 |
|------|------|----------|
| String | `SET key value` | 缓存、计数器 |
| Hash | `HSET user:1 name Alice` | 对象存储 |
| List | `LPUSH queue item` | 消息队列、任务列表 |
| Set | `SADD tags python` | 标签系统、去重 |
| Sorted Set | `ZADD leaderboard 100 player1` | 排行榜 |
| Bitmap | `SETBIT visit:2024 100 1` | 活跃用户统计 |
| HyperLogLog | `PFADD page:views url` | UV统计 |

**应用场景**：
- 会话缓存（Session Store）
- 消息队列（List实现）
- 实时排行榜（Sorted Set）
- 分布式锁

**高级特性**：

```python
import redis

r = redis.Redis(host='localhost', port=6379, db=0)

# 分布式锁
def acquire_lock(lock_name, timeout=10):
    lock_key = f"lock:{lock_name}"
    if r.set(lock_key, "1", nx=True, ex=timeout):
        return True
    return False

def release_lock(lock_name):
    lock_key = f"lock:{lock_name}"
    r.delete(lock_key)

# 管道批量操作
pipe = r.pipeline()
pipe.incr("page:views:home")
pipe.incr("page:views:about")
pipe.zincrby("leaderboard", 10, "player1")
pipe.execute()

# 发布/订阅
pubsub = r.pubsub()
pubsub.subscribe('channel:updates')
for message in pubsub.listen():
    print(f"收到消息: {message['data']}")
```

### 3. 宽列数据库（Wide-Column Store）

基于Google BigTable模型，以**列族**组织数据，适合海量数据的稀疏存储。

#### Apache Cassandra

Cassandra采用**无主分布式架构**，每个节点都是平等的。[^5]

**核心概念**：
- **Keyspace** → **Table** → **Row**
- 使用CQL（Cassandra Query Language）查询
- 副本策略：NetworkTopologyStrategy、SimpleStrategy
- 一致性级别可调

**数据模型**：

```sql
CREATE KEYSPACE IF NOT EXISTS social_media 
WITH REPLICATION = { 'class': 'NetworkTopologyStrategy', 'dc1': 3 };

CREATE TABLE social_media.posts (
    user_id UUID,
    post_id TIMEUUID,
    content TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, post_id)
) WITH CLUSTERING ORDER BY (post_id DESC);

-- 写入数据
INSERT INTO social_media.posts (user_id, post_id, content, created_at)
VALUES (
    uuid(), 
    now(), 
    'Hello Cassandra!',
    toTimestamp(now())
);

-- 查询用户的所有帖子
SELECT * FROM social_media.posts 
WHERE user_id = aaaaaaaa-0000-0000-0000-000000000000;
```

**架构特点**：
- **线性扩展**：新增节点自动平衡数据
- **多数据中心支持**：跨数据中心复制
- **高写入吞吐量**：针对写入优化

### 4. 图数据库（Graph Database）

以**节点和边**组织数据，专门处理复杂的关系查询。

#### Neo4j

Neo4j是主流图数据库，使用Cypher查询语言。[^6]

**核心概念**：
- **Node（节点）**：表示实体
- **Relationship（关系）**：表示实体间的联系
- **Property（属性）**：节点和关系的特征
- **Label（标签）**：节点的类型

**Cypher查询**：

```cypher
// 创建节点和关系
CREATE (alice:Person {name: 'Alice', age: 30})
CREATE (bob:Person {name: 'Bob', age: 28})
CREATE (project:Project {title: 'GraphDB'})
CREATE (alice)-[:KNOWS {since: 2020}]->(bob)
CREATE (alice)-[:WORKS_ON]->(project)

// 查询：Alice认识的人参与的项目
MATCH (alice:Person {name: 'Alice'})-[:KNOWS]->(person)-[:WORKS_ON]->(proj)
RETURN person.name, proj.title

// 最短路径查询
MATCH path = shortestPath(
    (a:Person {name: 'Alice'})-[*]-(b:Person {name: 'Charlie'})
)
RETURN path
```

**应用场景**：
- 社交网络（好友推荐）
- 知识图谱
- 欺诈检测
- 推荐系统

## NoSQL vs 关系型数据库

| 维度 | 关系型数据库 | NoSQL数据库 |
|------|-------------|-------------|
| **数据模型** | 严格的表结构 | 灵活（文档/键值/列族/图） |
| **事务** | ACID（强一致性） | BASE（最终一致性） |
| **扩展方式** | 垂直扩展（升级硬件） | 水平扩展（增加节点） |
| **查询能力** | SQL通用查询 | 特定查询语言 |
| **JOIN** | 原生支持 | 通常不支持，需应用层处理 |
| **适用场景** | 强一致性OLTP | 高扩展性需求、灵活模型 |

## 选择指南

```
选择NoSQL数据库时考虑：

┌─────────────────────────────────────────────────────┐
│ 1. 数据模型需求                                       │
│    文档型 ← 需要灵活schema → MongoDB/Couchbase       │
│    键值型 ← 极致简单/缓存 → Redis/DynamoDB           │
│    宽列型 ← 海量数据/写入密集 → Cassandra/ScyllaDB  │
│    图关系 ← 复杂关系查询 → Neo4j/ArangoDB            │
├─────────────────────────────────────────────────────┤
│ 2. 一致性需求                                        │
│    强一致性（CP）→ MongoDB/HBase                     │
│    可调一致性 → Cassandra/DynamoDB                  │
│    最终一致性 → 所有AP系统                           │
├─────────────────────────────────────────────────────┤
│ 3. 运维复杂度                                        │
│    托管服务 → MongoDB Atlas/DynamoDB                  │
│    自托管 → Cassandra/Redis                         │
└─────────────────────────────────────────────────────┘
```

## BASE语义

与RDBMS的ACID对应，NoSQL采用BASE语义：

| 性质 | 含义 |
|------|------|
| **Basically Available** | 系统保证可用性，但可能返回过期数据 |
| **Soft state** | 系统状态可能随时间变化，无需实时一致 |
| **Eventually consistent** | 系统经过一段时间后最终达到一致状态 |

这意味着在分布式系统中，**暂时的不一致是正常的**，应用需要容忍短期的不一致状态。

[^1]: 本段参考[A Primer on NoSQL Databases for Enterprise Architects](https://scholarspace.manoa.hawaii.edu/bitstreams/b27f47de-e936-4a9e-a2a2-0bd548d0850d/download)
[^2]: 本段参考[Mastering the CAP Theorem](https://www.mongodb.com/resources/basics/databases/cap-theorem)
[^3]: 本段参考[CSci 340: NoSQL - MongoDB Section](https://cburch.com/cs/340/reading/nosql/index.html)
[^4]: 本段参考[Top 10 NoSQL Databases in 2026](https://www.tasrieit.com/blog/top-10-nosql-databases-2026)
[^5]: 本段参考[MongoDB vs Cassandra vs Redis](https://databasesample.com/blog/mongodb-vs.-cassandra-vs.-redis:-a-nosql-database-comparison)
[^6]: 本段参考[Graph databases model data as nodes and relationships](https://www.tasrieit.com/blog/top-10-nosql-databases-2026)
