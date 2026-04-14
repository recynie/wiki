---
title: 共识算法深入
date: 2026-04-08
description: 分布式系统共识算法：Paxos、Raft、ZAB协议与拜占庭容错
tags:
  - distributed-systems
  - consensus
  - paxos
  - raft
  - zab
  - byzantine
draft: false
permalink:
---

## 共识算法基础

共识算法用于在分布式环境中就某个值或某序列值达成一致。[^1]

### FLP不可能定理

FLP定理（Fisher, Lynch, Paterson, 1985）证明：

> 在异步分布式系统中，如果存在至少一个进程可能发生故障，则不存在算法能始终保证共识。

这意味着在真实系统中，我们必须在**活性**（liveness）和**安全性**（safety）之间权衡。

### 共识的定义

一个共识算法必须满足：

| 性质 | 说明 |
|------|------|
| **终止性** | 所有正确进程最终做出决定 |
| **一致性** | 所有正确进程决定相同的值 |
| **有效性** | 如果所有进程输入相同，则决定该值 |

---

## Paxos 算法

Paxos是Leslie Lamport提出的经典共识算法。

### 角色

- **提议者（Proposer）**：提出值
- **接受者（Acceptor）**：投票
- **学习者（Learner）**：学习达成共识的值

### Basic Paxos

#### 两阶段执行

**准备阶段（Prepare）**：

```
Proposer → Prepare(n) → 多数派Acceptors
           其中 n 为提案编号（递增）
```

Acceptor承诺不再接受编号小于 $n$ 的Prepare请求。

**接受阶段（Accept）**：

```
Proposer → Accept(n, v) → 多数派Acceptors
           其中 v 为提议的值
```

Acceptor接受提案并通知Learner。

#### Python 伪代码

```python
class Acceptor:
    def __init__(self):
        self.promised_id = 0      # 已承诺的最大提案编号
        self.accepted_id = 0      # 已接受的最大提案编号
        self.accepted_value = None
    
    def prepare(self, n):
        if n > self.promised_id:
            self.promised_id = n
            # 返回已接受的提案（如果有）
            return {
                'ok': True,
                'promised_id': n,
                'accepted_id': self.accepted_id,
                'accepted_value': self.accepted_value
            }
        return {'ok': False}
    
    def accept(self, n, v):
        if n >= self.promised_id:
            self.promised_id = n
            self.accepted_id = n
            self.accepted_value = v
            return {'ok': True}
        return {'ok': False}

class Proposer:
    def __init__(self, node_id, acceptors):
        self.node_id = node_id
        self.acceptors = acceptors
        self.proposal_number = 0
    
    def propose(self, value):
        # 生成全局唯一递增提案编号
        self.proposal_number += 1
        n = self.proposal_number
        
        # 准备阶段：向所有Acceptor发送Prepare
        promises = []
        for acceptor in self.acceptors:
            response = acceptor.prepare(n)
            if response['ok']:
                promises.append(response)
        
        # 获得多数派响应
        if len(promises) > len(self.acceptors) // 2:
            # 如果有已接受的值，选择编号最大的
            accepted = [p for p in promises if p['accepted_value'] is not None]
            if accepted:
                max_accepted = max(accepted, key=lambda x: x['accepted_id'])
                value = max_accepted['accepted_value']
            
            # 接受阶段：向所有Acceptor发送Accept
            accepts = []
            for acceptor in self.acceptors:
                response = acceptor.accept(n, value)
                if response['ok']:
                    accepts.append(response)
            
            return len(accepts) > len(self.acceptors) // 2
        
        return False
```

### Multi-Paxos

Basic Paxos每次共识需要两轮通信，Multi-Paxos优化连续值：

```
            ┌─────── Leader ───────┐
            │                       │
Prepare(n) →│ ←────────────────── Promise
            │
Accept(n,v)→│ ←────────────────── Accepted (多数派)
            │                       │
(省略Prepare，直接Accept)
```

### Paxos 的问题

1. **难以实现**：算法细节复杂
2. **两轮通信**：延迟高
3. **活锁**：多个Proposer可能交替提高编号导致无法达成共识

---

## Raft 算法

Raft是Diego Ongaro提出的 Paxos 替代算法，更易于理解和实现。[^2]

### 三种角色

| 角色 | 说明 |
|------|------|
| **Leader** | 处理客户端请求，管理日志复制 |
| **Follower** | 被动响应请求，参与选举 |
| **Candidate** | 选举超时后发起选举 |

### Leader 选举

```
                    选举超时
         ┌──────────────────────────┐
         │                          │
    Follower                      Follower
         │                          │
         ▼                          ▼
   成为Candidate ──────────── 成为Candidate
         │                          │
         │   RequestVote(n) ────────→│
         │←─────── Vote ────────────│
         │   获得多数票，成为Leader  │
         │                          │
         ▼                          ▼
```

```python
class RaftNode:
    def __init__(self, node_id, peers):
        self.node_id = node_id
        self.peers = peers
        
        # 持久状态（故障恢复需要）
        self.current_term = 0
        self.voted_for = None
        self.log = []  # [(term, value), ...]
        
        # -volatile状态
        self.role = 'follower'
        self.commit_index = 0
        self.last_applied = 0
        
        # Leader专属
        self.next_index = {}   # 每个Follower的下一个日志索引
        self.match_index = {}  # 每个Follower已复制的最高日志索引
        
        # 选举超时
        self.election_timeout = random.uniform(150, 300)  # ms
    
    def run_election_timer(self):
        """选举定时器"""
        while True:
            elapsed = time_elapsed()
            if elapsed > self.election_timeout:
                if self.role != 'leader':
                    self.start_election()
                self.reset_election_timer()
    
    def start_election(self):
        """开始选举"""
        self.role = 'candidate'
        self.current_term += 1
        self.voted_for = self.node_id  # 给自己投票
        votes = 1
        
        # 向所有节点发送RequestVote
        for peer in self.peers:
            granted = peer.request_vote(
                term=self.current_term,
                candidate_id=self.node_id,
                last_log_index=len(self.log),
                last_log_term=self.log[-1][0] if self.log else 0
            )
            if granted:
                votes += 1
        
        # 获得多数票
        if votes > (len(self.peers) + 1) // 2:
            self.become_leader()
    
    def become_leader(self):
        self.role = 'leader'
        # 初始化next_index和match_index
        for peer in self.peers:
            self.next_index[peer] = len(self.log) + 1
            self.match_index[peer] = 0
```

### 日志复制

```
Client Request → Leader → 本地追加日志
                        ↓
              AppendEntries RPC → Followers
                        ↓
              多数派确认 → 提交 → 回复Client
```

```python
def append_entries(self, peer, prev_log_index, entries):
    """复制日志到Follower"""
    # 检查是否包含前一条日志
    if prev_log_index > 0:
        if prev_log_index > len(self.log):
            return False
        if self.log[prev_log_index - 1].term != prev_log_term:
            return False
    
    # 追加新日志
    for i, entry in enumerate(entries):
        if prev_log_index + i < len(self.log):
            if self.log[prev_log_index + i].term != entry.term:
                self.log = self.log[:prev_log_index + i]  # 冲突，截断
                self.log.append(entry)
        else:
            self.log.append(entry)
    
    # 更新match_index
    self.match_index[peer] = prev_log_index + len(entries)
    self.next_index[peer] = self.match_index[peer] + 1
    
    return True

def replicate_to_majority(self):
    """检查是否复制到多数派"""
    for i in range(self.commit_index + 1, len(self.log) + 1):
        count = 1  # Leader自身
        for peer in self.peers:
            if self.match_index[peer] >= i:
                count += 1
        if count > (len(self.peers) + 1) // 2:
            self.commit_index = i
```

### 成员变更

Raft支持在线成员变更：

```yaml
# 配置变更示例（联合共识）
old_config: { servers: [A, B, C] }
new_config: { servers: [A, B, C, D, E] }

# 两阶段：
# 1. Cold,new：所有操作需要Cold∪Cnew多数派确认
# 2. Cnew：只需要Cnew多数派确认
```

---

## ZAB 协议

ZAB（ZooKeeper Atomic Broadcast）是ZooKeeper使用的协议。

### 两种模式

| 模式 | 说明 |
|------|------|
| **恢复模式** | Leader崩溃后，选举新Leader并同步数据 |
| **广播模式** | 正常运行时，处理客户端请求 |

### 恢复模式

```python
class ZABRecovery:
    def __init__(self):
        self.epoch = 0           # 选举周期
        self.zxid = 0            # 事务ID
        self.leader = None
    
    def find_leader(self, peers):
        """选举新Leader"""
        votes = {}
        
        for peer in peers:
            peer_state = peer.get_state()
            votes[peer.id] = {
                'zxid': peer_state.last_zxid,
                'epoch': peer_state.epoch,
                'id': peer.id
            }
        
        # 优先选择zxid最大的（数据最新）
        # 次选epoch最大的
        leader = max(votes.values(), 
                    key=lambda x: (x['zxid'], x['epoch']))
        return leader['id']
    
    def sync_with_leader(self, leader):
        """与新Leader同步数据"""
        # 获取Leader的事务日志
        leader_history = leader.get_history()
        
        # 同步缺失的事务
        for tx in leaderHistory:
            if tx.zxid > self.zxid:
                self.apply(tx)
        
        self.leader = leader
        self.zxid = leader.zxid
```

### 广播模式

ZAB使用类似两阶段的原子广播：

```python
def broadcast(self, proposal):
    """原子广播"""
    # 1. Leader发送proposal给所有Followers
    for follower in self.followers:
        follower.receive_proposal(proposal)
    
    # 2. 等待多数派ACK
    acks = self.wait_for_acks(majority=len(self.followers)//2 + 1)
    
    # 3. 发送COMMIT
    self.send_commit(proposal)
```

### ZAB vs Raft

| 特性 | ZAB | Raft |
|------|-----|------|
| 选主依据 | zxid（事务ID） | term + 日志完整性 |
| 日志复制 | 相同 | 相同 |
| 成员变更 | 单阶段 | 两阶段（联合共识） |
| 应用 | ZooKeeper | etcd, CockroachDB |

---

## Paxos vs Raft vs ZAB

| 特性 | Paxos | Raft | ZAB |
|------|-------|------|-----|
| **提出者** | Lamport (1998) | Ongaro (2014) | ZooKeeper (2007) |
| **复杂度** | 高 | 中 | 中 |
| **可理解性** | 难 | 较易 | 较易 |
| **线性一致性** | 支持 | 支持 | 支持 |
| **成员变更** | 需额外处理 | 联合共识 | 单阶段 |
| **日志完整性** | 不保证 | Leader保证 | zxid保证 |
| **应用** | 理论基石 | etcd, Consul | ZooKeeper |

### 选型建议

- **小型团队**：选择 Raft（文档丰富，易于实现）
- **已有ZooKeeper**：继续使用 ZAB
- **理论验证**：参考 Paxos

---

## 拜占庭容错

拜占庭容错（BFT）处理节点可能发送错误/恶意消息的情况。

### PBFT（实用拜占庭容错）

PBFT（Practical Byzantine Fault Tolerance）可在 $f$ 个恶意节点下正常工作：

$$
\text{节点总数} \geq 3f + 1
$$

#### 三阶段协议

```
Client → Preprepare → [n-f] → Prepare → [n-f] → Commit → Client
              │                          │                      │
              ↓                          ↓                      ↓
         Leader发送                 节点广播                 等待[n-f]
         提案给所有                Prepare到所有             Commit后
         Followers                                          执行
```

```python
class PBFTNode:
    def __init__(self, node_id, peers, f):
        self.node_id = node_id
        self.peers = peers
        self.f = f  # 最大恶意节点数
        self.messages = {}  # 消息日志
    
    def preprepare(self, view, seq_num, digest):
        """Pre-prepare阶段"""
        msg = {
            'type': 'PREPREPARE',
            'view': view,
            'seq_num': seq_num,
            'digest': digest,
            'id': self.node_id
        }
        self.messages[seq_num] = {'preprepare': msg}
        self.broadcast(msg)
    
    def prepare(self, view, seq_num, digest):
        """Prepare阶段：等待n-f个PREPREPARE后，广播PREPARE"""
        if seq_num not in self.messages:
            return
        
        if len(self.messages[seq_num]['preprepare']) >= self.f + 1:
            msg = {
                'type': 'PREPARE',
                'view': view,
                'seq_num': seq_num,
                'digest': digest
            }
            self.broadcast(msg)
    
    def commit(self, view, seq_num, digest):
        """Commit阶段：等待n-f个PREPARE后，广播COMMIT"""
        if len([m for m in self.messages[seq_num]['prepare'] 
                if m['digest'] == digest]) >= 2 * self.f + 1:
            msg = {
                'type': 'COMMIT',
                'view': view,
                'seq_num': seq_num
            }
            self.broadcast(msg)
```

### BFT-SMaRt

Java实现的BFT库，支持状态机复制。

### Tendermint

基于BFT的权益证明区块链共识引擎。

---

## 参考资料

[^1]: Consensus Algorithms in Distributed Systems. https://distributedsystemauthority.com/consensus-algorithms.html

[^2]: In Search of an Understandable Consensus Algorithm. https://raft.github.io/raft.pdf

[^3]: Paxos vs Raft vs ZAB. https://www.designgurus.io/answers/detail/paxos-vs-raft-vs-zab-choosing-a-consensus-protocol
