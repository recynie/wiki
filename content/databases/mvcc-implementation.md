---
title: MVCC Implementation
date: 2026-04-15
description: 多版本并发控制（MVCC）实现原理
tags:
  - database
  - storage-engine
  - mvcc
  - concurrency
  - innodb
  - postgresql
draft: false
permalink:
---

## 简介

MVCC（Multi-Version Concurrency Control，多版本并发控制）是现代数据库解决**读写冲突**的核心机制。其核心思想是：

> 读事务看到的是某个时间点的**一致快照**，而非当前的实时数据

**核心优势**：
- **读者不阻塞写者**：读取快照无需等待
- **写者不阻塞读者**：写入创建新版本，不影响旧版本读取
- **无脏读**：每个事务看到一致的数据库视图

## MVCC 解决的问题

### 无 MVCC 的困境

假设事务 T1 正在修改某行：

| 方案 | 读者行为 | 问题 |
|------|---------|------|
| 阻塞等待 | T2 等待 T1 提交 | 性能差 |
| 读取未提交 | T2 读到 T1 的中间结果 | 脏读 |

### MVCC 的解决

T2 看到的是 T1 修改**之前**的版本，T1 继续修改不影响 T2。

## InnoDB 中的行版本管理

### 隐藏列

InnoDB 每行有两个隐藏系统列：

| 列名 | 类型 | 说明 |
|------|------|------|
| `DB_TRX_ID` | 6 字节 | 最近一次插入或修改此行的事务ID |
| `DB_ROLL_PTR` | 7 字节 | 指向 undo log 中前一版本的指针 |

### 版本链（Version Chain）

当一行被 UPDATE 时：

1. InnoDB 将**新版本**写入数据页
2. **旧版本**复制到 undo log
3. 通过 `DB_ROLL_PTR` 构成链表

```
最新版本（数据页）→ 旧版本1（undo log）→ 旧版本2（undo log）→ ...
     DB_ROLL_PTR ──────────┬───────────
```

### UPDATE 操作的 MVCC 流程

```sql
-- 事务 T1 执行
START TRANSACTION;
UPDATE accounts SET balance = 500 WHERE id = 1;
-- 此时：
--   新版本 balance=500 在数据页，DB_TRX_ID=T1
--   旧版本 balance=100 在 undo log，DB_ROLL_PTR 指向它
COMMIT;
```

## PostgreSQL 中的行版本管理

### 隐藏列

PostgreSQL 每行（tuple）有三个关键系统列：

| 列名 | 说明 |
|------|------|
| `xmin` | 创建此版本的事务ID（insert transaction） |
| `xmax` | 删除/更新此版本的事务ID（delete/update transaction） |
| `ctid` | 指向堆中物理位置的元组标识符 |

### 版本存储

PostgreSQL 的所有版本都存在**堆（heap）**中，而非单独的 undo log：

```
Index → ctid → Tuple v1 (xmax=已删除)
              → Tuple v2 (当前版本)
```

### HOT 链（Heap-Only Tuple）

当索引列未更改时，更新后的新版本与旧版本在同一个 heap page 内，通过 `ctid` 链接成 HOT 链，避免更新索引。

## Read View 快照机制

### 快照内容

当事务执行快照读时，创建一个 Read View：

| 字段 | 说明 |
|------|------|
| `trx_ids` | 活跃（未提交）事务ID列表 |
| `up_limit_id` | `trx_ids` 中的最小值 |
| `low_limit_id` | 下一个将分配的事务ID |
| `creator_trx_id` | 创建此快照的事务ID |

### 可见性判断规则

对于每行版本，比较其 `DB_TRX_ID` 与 Read View：

```
┌─────────────────────────────────────────────────────────────┐
│                    可见性判断流程                            │
├─────────────────────────────────────────────────────────────┤
│  DB_TRX_ID = creator_trx_id?                                │
│    → 是：当前事务自己修改的，可见                              │
│    → 否：继续判断                                            │
│                                                             │
│  DB_TRX_ID < up_limit_id?                                   │
│    → 是：事务在快照创建前已提交，可见                          │
│    → 否：继续判断                                            │
│                                                             │
│  DB_TRX_ID >= low_limit_id?                                 │
│    → 是：事务在快照创建后开始，不可见                          │
│    → 否：继续判断                                            │
│                                                             │
│  DB_TRX_ID in trx_ids?                                      │
│    → 是：快照创建时事务未提交，不可见                          │
│    → 否：事务在快照创建前已提交，可见                          │
└─────────────────────────────────────────────────────────────┘
```

### 不可见时的处理

如果当前版本不可见，沿 `DB_ROLL_PTR` 读取**前一个版本**，重复可见性判断，直到找到可见版本或到达链表末端。

## 隔离级别与 MVCC

### MySQL InnoDB

| 隔离级别 | 快照时机 | 说明 |
|----------|---------|------|
| READ UNCOMMITTED | 不使用 MVCC | 读取最新版本，包括未提交 |
| READ COMMITTED | 每个 SELECT 语句 | 每次查询重新创建 Read View |
| REPEATABLE READ（默认）| 事务第一条 SELECT | 整个事务使用相同 Read View |
| SERIALIZABLE | - | 使用锁机制，MVCC 退化 |

### PostgreSQL

| 隔离级别 | 快照时机 | 说明 |
|----------|---------|------|
| READ COMMITTED | 每个语句 | 语句级别快照 |
| REPEATABLE READ | 事务开始 | 事务级别快照 |
| SERIALIZABLE | 事务开始 | 使用 SSI 串行化 |

## Undo Log 与版本链

### InnoDB Undo Log 类型

| 类型 | 用途 | 清理时机 |
|------|------|---------|
| **Insert Undo** | 仅用于事务回滚 | 事务提交后立即删除 |
| **Update Undo** | 事务回滚 + MVCC 读取 | 无活跃事务引用时由 purge 清理 |

### Undo Log 段结构

Undo log 存储在专门的 undo tablespace 中：

```sql
-- 查看 undo 表空间
SHOW VARIABLES LIKE 'innodb_undo_tablespaces';
```

## PostgreSQL VACUUM 机制

### 为什么需要 VACUUM

PostgreSQL 的旧版本存储在堆中，如果不清理会不断积累，造成**表膨胀（bloat）**。

### VACUUM 工作原理

1. 扫描堆，标记已删除的元组为可用空间
2. 更新空闲空间映射（Free Space Map）
3. 可选：重建索引移除指向已删除元组的索引条目

### Autovacuum

PostgreSQL 自动运行 autovacuum 守护进程：

```sql
-- 配置参数
SHOW autovacuum_max_workers;     -- 最大工作进程数
SHOW autovacuum_naptime;         -- 两次运行之间的最小间隔
```

### VACUUM 与性能

VACUUM 不是"免费"的：
- 大量死元组时可能很慢
- 可能产生大量 I/O
- 需要合理配置阈值和调度

## MySQL vs PostgreSQL 实现对比

| 特性 | InnoDB (MySQL) | PostgreSQL |
|------|---------------|------------|
| **旧版本存储位置** | Undo Log（独立空间） | Heap（表中） |
| **最新版本位置** | B+Tree 叶子节点 | Heap |
| **行版本标识** | DB_TRX_ID + DB_ROLL_PTR | xmin + xmax + ctid |
| **可见性判断** | 单次 TRX_ID 比较 | 两次判断（xmin AND xmax） |
| **版本链方向** | 新 → 旧 | 旧 → 新 |
| **清理机制** | Purge 线程 | VACUUM |
| **空间效率** | 表本身紧凑 | 可能产生表膨胀 |

### 设计哲学差异

**MySQL/InnoDB**：
- 表中只存储最新版本，历史版本在 undo log
- 优点：表本身始终紧凑
- 缺点：读取旧版本需要通过 undo log 重建

**PostgreSQL**：
- 所有版本都在堆中
- 优点：读取任何版本都很快
- 缺点：需要定期 VACUUM 清理死元组

## 实战观察 MVCC

### MySQL 观察方法

```sql
-- 启动两个会话

-- 会话1：
START TRANSACTION;
UPDATE sbtest1 SET k=999 WHERE id=1;
-- 此时查看 undo log
SELECT * FROM information_schema.innodb_trx;  -- 查看活动事务
SELECT * FROM information_schema.innodb_undo_locks;  -- 查看 undo 锁

-- 会话2：会看到旧值，直到会话1提交
SELECT * FROM sbtest1 WHERE id=1;
```

### PostgreSQL 观察方法

```sql
-- 查看事务状态
SELECT txid_current(), txid_current_snapshot();

-- 查看元组版本信息（需要安装 pageinspect）
CREATE EXTENSION IF NOT EXISTS pageinspect;
SELECT xmin, xmax, ctid, * FROM sbtest1 WHERE id=1;
```

## 长事务的问题

### MySQL

长事务持有 undo log 引用，导致 undo log 无法 purge：

```sql
-- 查看长时间运行的事务
SELECT trx_id, trx_state, trx_started, trx_rows_modified
FROM information_schema.innodb_trx
ORDER BY trx_started;
```

### PostgreSQL

长事务阻止 VACUUM 清理死元组：

```sql
-- 查看活跃事务
SELECT datname, pid, usesysid, usename, application_name, 
       state, backend_xmin
FROM pg_stat_activity 
WHERE backend_xmin IS NOT NULL;
```

## 参考资料

[^1]: [MySQL MVCC - OneUptime](https://oneuptime.com/blog/post/2026-03-31-mysql-mvcc-multi-version-concurrency-control/view)

[^2]: [MySQL vs PostgreSQL Internals: MVCC - LinkedIn](https://www.linkedin.com/pulse/mysql-vs-postgresql-internals-part-2-mvcc-concurrency-zhao-song-qleqf)

[^3]: [PostgreSQL Documentation: MVCC Introduction](https://www.postgresql.org/docs/current/mvcc-intro.html)
