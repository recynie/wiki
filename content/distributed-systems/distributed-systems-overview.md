---
title: 分布式系统基础
date: 2026-04-07
description: 分布式系统核心概念：CAP定理、一致性模型、共识算法、复制策略
tags:
  - distributed-systems
  - cap-theorem
  - consistency
  - consensus
  - replication
draft: false
permalink:
---

## 定义与范畴

分布式系统是由多个独立节点组成的系统，这些节点通过网络通信协调完成共同任务。与单机系统不同，分布式系统需要处理节点故障、网络分区、消息丢失等问题。[^1]

### 核心挑战

- **节点故障**：节点可能随时停止服务
- **网络分区**：节点之间可能无法通信
- **消息丢失**：网络通信不可靠
- **时钟不一致**：节点本地时钟可能不同步

---

## CAP 定理

CAP定理是分布式系统理论的基石，由Eric Brewer提出（2000年），由Gilbert和Lynch证明（2002年）。

### 定理内容

分布式系统只能同时满足以下**三个特性中的两个**：

| 特性 | 定义 | 牺牲时行为 |
|------|------|-----------|
| **Consistency（一致性）** | 每次读取获得最近一次写入或错误 | 返回错误 |
| **Availability（可用性）** | 每个非故障节点都返回响应 | 返回可能过期的数据 |
| **Partition Tolerance（分区容错）** | 网络分区时系统继续运行 | 必须选择C或A |

### 为什么必须选择 P

在现实网络中，网络分区**不可避免**。因此，实际系统必须在CP和AP之间选择：

```
CP 系统                              AP 系统
┌────────────────────────┐         ┌────────────────────────┐
│ 牺牲可用性               │         │ 牺牲一致性               │
│ 分区时阻塞/返回错误      │         │ 分区时返回本地数据       │
└────────────────────────┘         └────────────────────────┘
典型：ZooKeeper, etcd, HBase         典型：Cassandra, DynamoDB
```

### PACELC 模型

即使没有网络分区，系统也面临延迟-一致性权衡（Abadi, 2012）：

$$
\text{PACELC} = \text{if Partition} \rightarrow \text{(CP or AP)}, \text{ else Latency} \leftrightarrow \text{Consistency}
$$

---

## 一致性模型

一致性模型定义了读取操作可能返回的值的关系，从强到弱：

### 强度层次

| 一致性模型 | 描述 | AP/CP |
|-----------|------|-------|
| **线性一致性** | 读操作如同原子执行，所有节点看到相同顺序 | CP |
| **顺序一致性** | 所有节点看到相同操作顺序（不一定实时） | CP |
| **因果一致性** | 因果相关的操作顺序一致 | AP |
| **读己之所写** | 自己写的立即可见 | AP |
| **单调读** | 不会读到比之前更旧的数据 | AP |
| **最终一致性** | 所有副本最终收敛到相同值 | AP |

### 线性一致性与顺序一致性

```
线性一致性：操作如同在单个时间点原子完成
顺序一致性：所有节点看到操作顺序相同（可能不是实时顺序）
```

---

## 共识算法

共识算法用于在分布式环境中就某个值达成一致。

### Paxos 算法

Paxos是最经典的共识算法，分为两个阶段：

**准备阶段（Prepare）**：
1. 提议者选择提案编号 $n$，向多数派节点发送 `Prepare(n)`
2. 接收者承诺不再接受编号小于 $n$ 的提案

**接受阶段（Accept）**：
1. 提议者收到多数派响应后，发送 `Accept(n, v)`
2. 接收者接受提案

```python
class PaxosProposer:
    def __init__(self, node_id, peers):
        self.node_id = node_id
        self.peers = peers
        self.proposal_number = 0
    
    def prepare(self):
        n = self.proposal_number
        responses = [peer.prepare(n) for peer in self.peers]
        # 获得多数派响应，继续接受阶段
        return n
    
    def accept(self, n, value):
        accepted = [peer.accept(n, value) for peer in self.peers]
        return sum(accepted) > len(self.peers) // 2
```

### Raft 算法

Raft算法是Paxos的简化实现，通过选举Leader协调：

```
         Leader Election
    ┌───────────────────────┐
    │ Follower超时 →        │
    │ Candidate → 投票请求   │
    │ 获得多数票 → 成为Leader│
    └───────────────────────┘
              ↓
    ┌───────────────────────┐
    │ Log Replication        │
    │ Client → Leader →     │
    │ 日志条目 → 复制到       │
    │ Followers → 多数确认   │
    │ → 提交                 │
    └───────────────────────┘
```

Raft三种角色：
- **Leader**：处理客户端请求，复制日志
- **Follower**：被动响应请求
- **Candidate**：选举超时后发起选举

### 两阶段提交（2PC）

2PC用于分布式事务：

```
Phase 1: 准备阶段
Coordinator    →    所有节点: prepare？
Node          →    Coordinator: vote (commit/abort)

Phase 2: 提交阶段
Coordinator    →    所有节点: commit 或 abort
(任意节点投abort → 全部回滚)
```

缺点：协调者故障会阻塞，可能产生脑裂。

---

## 数据复制策略

### 主从复制

```python
class LeaderFollowerReplication:
    def __init__(self):
        self.leader = None
        self.followers = []
    
    def write(self, key, value):
        # 写入只在leader
        self.leader.log.append((key, value))
        # 异步复制到followers
        for follower in self.followers:
            follower.replicate(self.leader.log)
        return True
    
    def read(self, key):
        # 可以从任意副本读取（最终一致）
        follower = random.choice(self.followers)
        return follower.get(key)
```

### Quorum 机制

确保读写一致性：

- 写操作需要获得 $W$ 个节点确认
- 读操作需要查询 $R$ 个节点
- 当 $W + R > N$（总节点数）时，保证读到最新写入

```python
class QuorumStorage:
    def __init__(self, n, w=None, r=None):
        self.n = n
        self.w = w or (n // 2 + 1)
        self.r = r or (n // 2 + 1)
    
    def write(self, key, value):
        """写入W个节点"""
        versions = []
        for node in self.nodes[:self.w]:
            v = node.write(key, value)
            versions.append(v)
        return max(versions)
    
    def read(self, key):
        """读取R个节点，取最新版本"""
        results = []
        for node in self.nodes[:self.r]:
            v, data = node.read(key)
            results.append((v, data))
        return max(results)[1]
```

### CRDT（无冲突复制数据类型）

CRDT是一种保证最终一致性的数据结构，无需协调：

- **G-Counter**：只增计数器
- **PN-Counter**：可增可减计数器
- **LWW-Register**：最后写入胜出寄存器
- **OR-Set**：无冲突集合

---

## 故障类型与处理

### 故障分类

| 故障类型 | 描述 | 处理方式 |
|---------|------|----------|
| **崩溃故障** | 节点停止响应 | 心跳检测、重启 |
| **遗漏故障** | 节点无法发送/接收消息 | 超时、重传 |
| **拜占庭故障** | 节点任意行为（撒谎） | 拜占庭容错协议 |

### 故障检测

```python
class HeartbeatMonitor:
    def __init__(self, nodes, timeout=5):
        self.nodes = nodes
        self.timeout = timeout
        self.last_heartbeat = {node: time.time() for node in nodes}
    
    def check(self):
        for node in self.nodes:
            if time.time() - self.last_heartbeat[node] > self.timeout:
                # 节点可能故障
                self.handle_suspected_failure(node)
    
    def handle_suspected_failure(self, node):
        # 触发重新选举或数据恢复
        pass
```

---

## 分布式系统设计模式

### 熔断器模式（Circuit Breaker）

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func):
        if self.state == 'OPEN':
            if time.time() - self.last_failure_time > self.timeout:
                self.state = 'HALF_OPEN'
            else:
                raise Exception("Circuit is OPEN")
        
        try:
            result = func()
            self.on_success()
            return result
        except Exception:
            self.on_failure()
            raise
    
    def on_success(self):
        self.failure_count = 0
        self.state = 'CLOSED'
    
    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = 'OPEN'
```

### 限流模式

```python
class RateLimiter:
    def __init__(self, max_requests, window_seconds):
        self.max_requests = max_requests
        self.window = window_seconds
        self.requests = deque()
    
    def allow(self):
        now = time.time()
        # 清理过期的请求记录
        while self.requests and self.requests[0] < now - self.window:
            self.requests.popleft()
        
        if len(self.requests) < self.max_requests:
            self.requests.append(now)
            return True
        return False
```

### 重试模式

```python
class RetryPolicy:
    def __init__(self, max_retries=3, backoff='exponential'):
        self.max_retries = max_retries
        self.backoff = backoff
    
    def execute(self, func):
        for attempt in range(self.max_retries):
            try:
                return func()
            except Exception as e:
                if attempt == self.max_retries - 1:
                    raise
                sleep_time = 2 ** attempt if self.backoff == 'exponential' else 1
                time.sleep(sleep_time)
```

---

## 参考资料

[^1]: CAP Theorem - Distributed System Authority. https://distributedsystemauthority.com/cap-theorem.html

[^2]: The Google File System (Ghemawat et al., 2003)

[^3]: Dynamo: Amazon's Highly Available Key-value Store (DeCandia et al., 2007)
