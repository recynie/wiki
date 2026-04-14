---
title: 分布式事务
date: 2026-04-08
description: 分布式事务：2PC、TCC、Saga、本地消息表、Seata框架
tags:
  - distributed-systems
  - distributed-transactions
  - 2pc
  - tcc
  - saga
draft: false
permalink:
---

## 分布式事务基础

在分布式系统中，数据分布在多个节点，事务需要跨多个节点保证ACID特性。[^1]

### CAP 定理下的权衡

| 特性 | 说明 |
|------|------|
| **原子性** | 所有节点要么全部提交，要么全部回滚 |
| **一致性** | 事务前后系统状态一致 |
| **隔离性** | 并发事务相互隔离 |
| **持久性** | 事务结果永久保存 |

CAP定理指出：一致性、可用性、分区容错只能同时满足两个。

### 分布式事务模式

| 模式 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| **2PC** | 强一致 | 阻塞、 coordinator单点 | 低并发、短事务 |
| **TCC** | 性能较好 | 业务侵入性强 | 高并发 |
| **Saga** | 异步、性能好 | 无隔离性 | 长事务 |
| **本地消息表** | 简单 | 依赖本地数据库 | 最终一致 |

---

## 2PC（两阶段提交）

2PC是最基础的分布式事务协议。

### 两阶段

**准备阶段（Prepare）**：

```
Coordinator → prepare() → 所有参与者
     ↑                      ↑
     │                      │
     └──── vote (commit/abort) ←─┘
```

**提交阶段（Commit）**：

```
Coordinator → commit/abort → 所有参与者
     ↑                      ↑
     │                      │
     └──── ack ←────────────┘
```

### 流程图

```
┌─────────────┐          ┌─────────────┐          ┌─────────────┐
│ Coordinator │          │Participant 1│          │Participant 2│
└──────┬──────┘          └──────┬──────┘          └──────┬──────┘
       │                        │                        │
       │   1. prepare()         │                        │
       ├────────────────────────►│                        │
       │                        │   OK (locked)          │
       ├────────────────────────┤◄───────────────────────┤
       │                        │                        │
       │   2. prepare()         │                        │
       ├────────────────────────────────────────────────►│
       │                        │                        │
       │   3. OK (locked)       │                        │
       ├─────────────────────────────────────────────────┤◄──────
       │                        │                        │
       │   4. commit()           │                        │
       ├────────────────────────►│                        │
       │                        │   committed            │
       │◄───────────────────────┤                        │
       │                        │                        │
       │   5. commit()           │                        │
       ├────────────────────────────────────────────────►│
       │                        │                        │
       │   6. committed         │                        │
       ├─────────────────────────────────────────────────┤◄──────
       │                        │                        │
```

### Python 实现

```python
class Coordinator:
    def __init__(self, participants):
        self.participants = participants
        self.transaction_log = []  # 持久化日志
    
    def execute(self, action_fn):
        try:
            # Phase 1: Prepare
            votes = []
            for participant in self.participants:
                vote = participant.prepare()
                votes.append(vote)
            
            # 判断是否全部提交
            if all(v == 'commit' for v in votes):
                # Phase 2: Commit
                for participant in self.participants:
                    participant.commit()
                return True
            else:
                # Phase 2: Abort
                for participant in self.participants:
                    participant.abort()
                return False
        except Exception as e:
            # 异常处理
            for participant in self.participants:
                participant.abort()
            raise

class Participant:
    def __init__(self, name):
        self.name = name
        self.state = 'INIT'
        self.locked_resources = []
    
    def prepare(self):
        try:
            # 锁定资源，准备提交
            self.locked_resources = self.acquire_locks()
            self.state = 'PREPARED'
            return 'commit'
        except Exception:
            self.state = 'ABORT'
            return 'abort'
    
    def commit(self):
        self.execute_transaction()
        self.release_locks()
        self.state = 'COMMITTED'
    
    def abort(self):
        self.release_locks()
        self.state = 'ABORTED'
    
    def acquire_locks(self):
        # 获取数据库锁
        pass
    
    def release_locks(self):
        # 释放数据库锁
        pass
    
    def execute_transaction(self):
        # 执行实际业务
        pass
```

### 2PC 的问题

| 问题 | 说明 |
|------|------|
| **同步阻塞** | Prepare阶段锁定资源，其他事务等待 |
| **Coordinator单点** | Coordinator故障导致阻塞 |
| **数据不一致** | Commit阶段部分失败 |

### 3PC（三阶段提交）

```
CanCommit → PreCommit → DoCommit

- 加入超时机制
- 减少阻塞时间
- 但仍无法完全避免数据不一致
```

---

## TCC（Try-Confirm-Cancel）

TCC是应用层的两阶段提交，通过业务代码实现补偿。

### 三阶段

| 阶段 | 说明 |
|------|------|
| **Try** | 预留资源，检证业务可行性 |
| **Confirm** | 确认执行，使用预留资源 |
| **Cancel** | 取消执行，释放预留资源 |

### 流程图

```
┌─────────────┐          ┌─────────────┐
│   Service   │          │  Inventory  │
│  (TCC调用方) │          │  (TCC参与者) │
└──────┬──────┘          └──────┬──────┘
       │                        │
       │   1. try()             │
       ├────────────────────────►
       │   2. 预留资源成功       │
       ├────────────────────────┤
       │                        │
       │   3. confirm()          │
       ├────────────────────────►
       │   4. 使用预留资源       │
       ├────────────────────────┤
       │                        │
```

### 账户服务 TCC 示例

```python
class AccountService:
    def try_deduct(self, account_id, amount):
        """Try：检查并冻结金额"""
        account = self.db.get_account(account_id)
        
        if account.balance < amount:
            raise Exception("余额不足")
        
        # 冻结金额（减少可用余额，增加冻结金额）
        account.frozen_balance += amount
        account.balance -= amount
        self.db.save(account)
        
        return True
    
    def confirm_deduct(self, account_id, amount):
        """Confirm：确认扣减，释放冻结"""
        account = self.db.get_account(account_id)
        
        # 减少冻结金额
        account.frozen_balance -= amount
        self.db.save(account)
        
        return True
    
    def cancel_deduct(self, account_id, amount):
        """Cancel：回滚，返还可用余额"""
        account = self.db.get_account(account_id)
        
        # 返还可用余额
        account.balance += amount
        account.frozen_balance -= amount
        self.db.save(account)
        
        return True

class OrderService:
    def create_order(self, user_id, items):
        """创建订单"""
        # 扣减库存（调用TCC）
        for item in items:
            self.tcc_service.execute(
                try_fn=lambda: inventory.try_deduct(item.id, item.count),
                confirm_fn=lambda: inventory.confirm_deduct(item.id, item.count),
                cancel_fn=lambda: inventory.cancel_deduct(item.id, item.count)
            )
        
        # 创建订单
        order = self.create_order_record(user_id, items)
        return order
```

### TCC 注意事项

1. **幂等性**：Confirm/Cancel操作必须幂等
2. **悬挂**：Cancel在Try之前执行
3. **空回滚**：Try未执行但收到Cancel

```python
class TCCParticipant:
    def __init__(self):
        self.tried = False
    
    def try_phase(self):
        self.tried = True
        # 预留资源
    
    def confirm_phase(self):
        if not self.tried:
            return  # 避免空确认
        # 确认操作
    
    def cancel_phase(self):
        if not self.tried:
            return  # 避免空回滚（但需记录）
        # 补偿操作
```

---

## Saga 模式

Saga将长事务拆分为多个短事务，通过补偿达到最终一致。

### 编排方式

**顺序执行**：
```
A → B → C → D → E
  ↙︎   ↙︎   ↙︎
  ✕    ✕    ✕  (失败则逆向补偿)
```

**补偿定义**：

| 正向操作 | 补偿操作 |
|---------|---------|
| 创建订单 | 取消订单 |
| 扣减余额 | 返还余额 |
| 扣减库存 | 返还库存 |
| 发送短信 | - |

### Python 实现

```python
class SagaOrchestrator:
    def __init__(self, steps):
        self.steps = steps  # [(action, compensation), ...]
        self.executed_steps = []
    
    def execute(self):
        for i, (action, compensation) in enumerate(self.steps):
            try:
                result = action()
                self.executed_steps.append((i, result))
            except Exception as e:
                # 执行补偿
                self.compensate(i - 1)
                raise SagaExecutionError(f"Step {i} failed: {e}")
        
        return True
    
    def compensate(self, last_successful_index):
        """逆向补偿"""
        for i in range(last_successful_index, -1, -1):
            _, compensation = self.steps[i]
            try:
                compensation()
            except Exception:
                # 记录补偿失败，需要人工处理
                log_error(f"Compensation failed for step {i}")

# 使用示例
def create_order_flow():
    saga = SagaOrchestrator([
        # [(正向操作), (补偿操作)]
        (lambda: inventory.reduce(item_id, count),
         lambda: inventory.restore(item_id, count)),
        
        (lambda: account.deduct(user_id, total),
         lambda: account.refund(user_id, total)),
        
        (lambda: order.create(user_id, items),
         lambda: order.cancel(order_id)),
    ])
    
    saga.execute()
```

### Saga vs TCC

| 特性 | Saga | TCC |
|------|------|-----|
| 业务侵入 | 低（定义补偿即可） | 高（Try/Confirm/Cancel） |
| 性能 | 好（无锁） | 较好（资源预占） |
| 一致性 | 最终一致 | 强一致 |
| 隔离性 | 无 | 可通过预留实现 |
| 适用场景 | 长事务 | 短事务 |

---

## 本地消息表

本地消息表将消息写入本地数据库，结合MQ实现最终一致。

### 流程

```
1. 业务操作与消息写入同一事务
                    ↓
2. 定时任务扫描未发送消息
                    ↓
3. 发送消息到MQ
                    ↓
4. 消费方处理，ACK
                    ↓
5. 更新消息状态为已发送
```

### Python 实现

```python
class LocalMessageTable:
    def __init__(self, db, mq):
        self.db = db
        self.mq = mq
    
    def send_message(self, topic, payload):
        """发送消息（与业务操作原子）"""
        with self.db.transaction():
            # 执行业务
            self.db.execute(payload['business_sql'])
            
            # 写入消息表
            self.db.execute("""
                INSERT INTO local_messages (topic, payload, status, created_at)
                VALUES (?, ?, 'pending', NOW())
            """, [topic, json.dumps(payload)])
    
    def scan_and_send(self):
        """扫描并发送待发送消息"""
        messages = self.db.query("""
            SELECT * FROM local_messages 
            WHERE status = 'pending' 
            AND retry_count < 3
            ORDER BY created_at
            LIMIT 100
        """)
        
        for msg in messages:
            try:
                # 发送消息
                self.mq.send(msg['topic'], msg['payload'])
                
                # 更新状态
                self.db.execute("""
                    UPDATE local_messages 
                    SET status = 'sent', sent_at = NOW()
                    WHERE id = ?
                """, [msg['id']])
            except Exception:
                # 重试计数
                self.db.execute("""
                    UPDATE local_messages 
                    SET retry_count = retry_count + 1
                    WHERE id = ?
                """, [msg['id']])

# 消费方
class MessageConsumer:
    def handle(self, message):
        try:
            # 幂等处理
            if self.processed(message.id):
                return
            
            # 业务处理
            self.process(message.payload)
            
            # 标记已处理
            self.mark_processed(message.id)
        except Exception:
            # 失败，MQ会重试
            raise
```

---

## 最大努力通知

最大努力通知通过定期检查确保最终一致。

```python
class BestEffortNotification:
    def notify(self, order_id):
        """最大努力通知"""
        for attempt in range(3):
            try:
                payment_service.notify(order_id)
                return True
            except Exception:
                time.sleep(5 * (attempt + 1))  # 递增等待
        return False
    
    def check_status(self, order_id):
        """定期检查状态"""
        local_status = self.order_db.get(order_id)
        remote_status = self.payment_service.query(order_id)
        
        if local_status != remote_status:
            self.order_db.update(order_id, remote_status)
```

---

## Seata 框架

Seata（Simple Extensible Autonomous Transaction Architecture）是阿里巴巴开源的分布式事务解决方案。

### 四种模式

| 模式 | 说明 | 资源管理 |
|------|------|---------|
| **AT** | 自动补偿，对业务无侵入 | Seata |
| **TCC** | 用户定义补偿 | 用户 |
| **Saga** | 状态机编排 | 用户 |
| **XA** | XA协议，强一致 | 数据库 |

### AT 模式

```yaml
# seata-server (TC)
server:
  port: 8091

# application.yml
seata:
  tx-service-group: my_test_tx_group
  service:
    vgroup-mapping:
      my_test_tx_group: default
    enable-degrade: false
```

```java
// AT模式示例
@GlobalTransactional
public void createOrder(OrderDTO order) {
    // 1. 创建订单（自动开启全局事务）
    orderService.create(order);
    
    // 2. 扣减库存（自动注册分支事务）
    inventoryService.deduct(order.getItemId(), order.getCount());
    
    // 3. 扣减余额
    accountService.deduct(order.getUserId(), order.getTotal());
    
    // 异常时自动回滚
    if (order.getCount() < 0) {
        throw new RuntimeException("库存不足");
    }
}
```

### TCC 模式

```java
@LocalTCC
public interface InventoryTccService {
    
    @TwoPhaseBusinessAction(
        name = "inventoryTcc",
        commitMethod = "confirm",
        rollbackMethod = "cancel"
    )
    boolean tryDeduct(
        @BusinessActionContextParameter(paramName = "itemId") Long itemId,
        @BusinessActionContextParameter(paramName = "count") Integer count
    );
    
    boolean confirm(BusinessActionContext context);
    
    boolean cancel(BusinessActionContext context);
}

@Service
public class InventoryTccServiceImpl implements InventoryTccService {
    
    @Override
    public boolean tryDeduct(Long itemId, Integer count) {
        // Try阶段：检查并预留
        return inventoryMapper.reduceStock(itemId, count) > 0;
    }
    
    @Override
    public boolean confirm(BusinessActionContext context) {
        // Confirm阶段：确认使用预留资源
        // 通常为空（已扣减）
        return true;
    }
    
    @Override
    public boolean cancel(BusinessActionContext context) {
        // Cancel阶段：释放预留
        Long itemId = context.getActionContext("itemId", Long.class);
        Integer count = context.getActionContext("count", Integer.class);
        inventoryMapper.restoreStock(itemId, count);
        return true;
    }
}
```

---

## 方案选型

| 场景 | 推荐方案 |
|------|---------|
| **强一致、短事务、低并发** | 2PC/XA |
| **强一致、高并发** | TCC |
| **最终一致、长事务** | Saga |
| **简单集成** | 本地消息表 |
| ** Alibaba生态** | Seata AT |

---

## 参考资料

[^1]: Distributed Transactions - Distributed System Authority. https://distributedsystemauthority.com/distributed-transactions.html

[^2]: Seata Documentation. https://seata.io/zh-cn/

[^3]: Saga Pattern - Microsoft Azure. https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/saga/saga
