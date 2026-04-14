---
title: 数据库事务隔离级别
date: 2026-04-13
description: 数据库事务隔离级别详解：读未提交、读已提交、可重复读、串行化，以及MVCC实现原理
tags:
  - database
  - transaction
  - mvcc
  - isolation-level
draft: false
permalink:
---

## 基本概念

**事务**是数据库操作的最小逻辑单元，具有 ACID 四大特性：

| 特性 | 说明 |
|------|------|
| **原子性（Atomicity）** | 事务要么全部成功，要么全部失败回滚 |
| **一致性（Consistency）** | 事务执行前后，数据库状态保持一致 |
| **隔离性（Isolation）** | 并发事务相互隔离，不互相干扰 |
| **持久性（Durability）** | 事务提交后，其结果永久保存 |

隔离性是本篇讨论的重点。

## 四种隔离级别

SQL 标准定义了四种隔离级别，从低到高：

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
|----------|------|-----------|------|
| **读未提交（Read Uncommitted）** | 可能 | 可能 | 可能 |
| **读已提交（Read Committed）** | 不可能 | 可能 | 可能 |
| **可重复读（Repeatable Read）** | 不可能 | 不可能 | 可能 |
| **串行化（Serializable）** | 不可能 | 不可能 | 不可能 |

### 1. 读未提交（Read Uncommitted）

最低级别，事务可以读取其他事务**未提交**的数据。

```sql
-- 事务A
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- 未提交

-- 事务B（读未提交）
SELECT balance FROM accounts WHERE id = 1;  -- 可能读到 900（脏读）
```

**问题**：脏读（Dirty Read）—— 读取到未确认的数据。

### 2. 读已提交（Read Committed）

只能读取已提交事务的数据。Oracle、PostgreSQL 默认级别。

```sql
-- 事务A
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;  -- 提交

-- 事务B（读已提交）
SELECT balance FROM accounts WHERE id = 1;  -- 读到 900
```

**问题**：不可重复读（Non-repeatable Read）—— 同一事务内两次读取同一行，结果可能不同。

### 3. 可重复读（Repeatable Read）

MySQL InnoDB 默认级别。同一事务内多次读取同一行，结果始终相同。

```sql
-- 事务B
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- 读到 1000
-- 事务A此时修改了 id=1 的 balance 并提交
SELECT balance FROM accounts WHERE id = 1;  -- 仍读到 1000（同一快照）
COMMIT;
```

**问题**：幻读（Phantom Read）—— 同一事务内，两次查询返回的**行数**不同。

### 4. 串行化（Serializable）

最高级别，事务顺序执行，完全避免并发问题。但性能最差。

---

## 并发问题详解

### 脏读（Dirty Read）

读取到其他事务未提交的数据：

| 时间 | 事务A | 事务B |
|------|--------|--------|
| T1 | BEGIN | |
| T2 | | BEGIN |
| T3 | UPDATE SET balance=900 | |
| T4 | | SELECT balance=**900**（脏读） |
| T5 | ROLLBACK | |
| T6 | | COMMIT（基于错误数据） |

### 不可重复读（Non-repeatable Read）

同一事务内，两次读取同一行数据不同：

| 时间 | 事务A | 事务B |
|------|--------|--------|
| T1 | BEGIN; SELECT balance=**1000** | |
| T2 | | BEGIN |
| T3 | | UPDATE SET balance=900; COMMIT |
| T4 | SELECT balance=**900**（不一致） | |
| T5 | COMMIT | |

### 幻读（Phantom Read）

同一事务内，两次查询返回的**记录数**不同：

| 时间 | 事务A | 事务B |
|------|--------|--------|
| T1 | BEGIN; SELECT COUNT(*) FROM orders=**5** | |
| T2 | | BEGIN |
| T3 | | INSERT INTO orders...; COMMIT |
| T4 | SELECT COUNT(*) FROM orders=**6**（幻读） | |
| T5 | COMMIT | |

---

## MVCC 多版本并发控制

### 基本原理

MVCC（Multi-Version Concurrency Control）通过保存数据的多个版本来实现高并发：

- **写操作**：不直接覆盖旧数据，而是创建新版本
- **读操作**：根据事务时间戳，读取合适的历史版本
- **垃圾回收**：定期清理不再需要的旧版本

### InnoDB 的 MVCC 实现

InnoDB 通过以下字段实现 MVCC：

```sql
-- 隐藏列（对用户不可见）
每行数据包含：
- DB_TRX_ID: 最近修改该行的事务ID
- DB_ROLL_PTR: 指向 undo log 的指针
- DB_ROW_ID: 行ID（聚簇索引）
```

### Read View

Read View 是快照的核心，包含：
- `m_ids`：活跃事务ID列表
- `min_trx_id`：最小活跃事务ID
- `max_trx_id`：创建Read View时最大事务ID
- `creator_trx_id`：当前事务ID

### 可见性判断规则

```
if (row.trx_id == creator_trx_id) {
    // 自己修改的，当前事务可见
    return true;
}
if (row.trx_id < min_trx_id) {
    // 事务已提交，当前事务可见
    return true;
}
if (row.trx_id in m_ids) {
    // 有活跃事务修改过，当前事务不可见
    return false;
}
return true;  // 已提交事务修改的，可见
```

### 快照读 vs 当前读

| 类型 | 说明 | 示例 |
|------|------|------|
| **快照读（Snapshot Read）** | 读取历史版本，不加锁 | `SELECT * FROM t` |
| **当前读（Current Read）** | 读取最新数据，加锁 | `SELECT * FROM t FOR UPDATE` |

```sql
-- 快照读：不加锁，读取历史版本
SELECT * FROM accounts WHERE id = 1;

-- 当前读：加锁，读取最新数据
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
```

---

## 隔离级别实现（MySQL InnoDB）

### 读未提交

直接读取最新版本，无任何控制。

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```

### 读已提交

每次读取都创建新的 Read View：

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

### 可重复读

事务开始时创建 Read View，整个事务期间使用同一个快照：

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

### 串行化

所有读取都加锁，转化为串行执行：

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

---

## Next-Key Lock 解决幻读

InnoDB 在可重复读级别下使用 **Next-Key Lock** 来防止幻读：

### 记录锁（Record Lock）

锁定索引记录本身：

```sql
SELECT * FROM orders WHERE id = 10 FOR UPDATE;
-- 锁定 id=10 的索引记录
```

### 间隙锁（Gap Lock）

锁定索引记录之间的间隙：

```sql
SELECT * FROM orders WHERE id BETWEEN 5 AND 15 FOR UPDATE;
-- 锁定 (5, 15) 区间，不允许插入
```

### Next-Key Lock

记录锁 + 间隙锁的组合：

```
索引 1, 3, 5, 7, 9

对 id=5 加锁时，Next-Key Lock 锁定：
- 记录锁：锁定 id=5
- 间隙锁：锁定 (3, 5) 和 (5, 7)
```

---

## 实战建议

### 选择隔离级别

| 场景 | 推荐级别 | 原因 |
|------|---------|------|
| 金融交易 | Serializable | 数据一致性最高 |
| 普通业务 | Repeatable Read | MySQL默认，平衡性能与一致性 |
| 日志分析 | Read Committed | 只关心已提交数据 |
| 高并发读取 | Read Committed | 减少锁竞争 |

### 避免长事务

```sql
-- 监控长事务
SELECT * FROM information_schema.INNODB_TRX 
WHERE trx_started < NOW() - INTERVAL 60 SECOND;

-- 优化建议
1. 减少事务内操作数量
2. 及时提交/回滚
3. 使用批量操作减少交互次数
```

### 理解 MVCC 的代价

```sql
-- MVCC 维护多版本，有以下开销：
1. 存储空间：旧版本占用额外空间
2. 垃圾回收：定期清理过期版本
3. 查找开销：定位正确版本需要判断

-- 监控表大小
SELECT TABLE_NAME, 
       (DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 AS 'MB'
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database';
```

---

## 参考资料

[^1]: InnoDB and MVCC. https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html

[^2]:_transaction_isolation. https://dev.mysql.com/doc/refman/8.0/en/set-transaction.html

[^3]: MySQL 文档 - InnoDB Locking. https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html

