---
title: WAL (Write-Ahead Log)
date: 2026-04-15
description: 预写日志机制（WAL）确保数据持久性的核心原理
tags:
  - database
  - storage-engine
  - wal
  - innodb
draft: false
permalink:
---

## 简介

Write-Ahead Logging（WAL，预写日志）是数据库存储引擎确保数据持久性（Durability）的核心机制。其核心思想是：**在修改数据页之前，必须先将更改写入日志并刷盘**。如果数据库发生崩溃，可以通过重放日志恢复未完成的事务。

WAL 解决了数据库崩溃后可能丢失已提交事务的问题，实现了 ACID 中的 **Durability** 保证。

## Redo Log vs Undo Log

| 特性 | Redo Log | Undo Log |
|------|----------|----------|
| **用途** | 崩溃恢复时重放已提交事务的更改 | 事务回滚和 MVCC 快照读取 |
| **记录内容** | 修改后的数据（物理修改） | 修改前的数据（旧值） |
| **生命周期** | 事务提交后仍需保留至 checkpoint | MVCC 引用结束后由 purge 清理 |
| **存储位置** | ib_logfile0, ib_logfile1（InnoDB） | undo tablespace（InnoDB） |

## InnoDB Redo Log 实现

### 日志文件结构

MySQL 5.7 及之前版本使用 `ib_logfile0`、`ib_logfile1` 两个循环使用的日志文件：

```sql
-- 查看 redo log 配置
SHOW VARIABLES LIKE 'innodb_log_file_size';       -- 每个文件大小
SHOW VARIABLES LIKE 'innodb_log_files_in_group';  -- 文件数量（默认 2）
-- Total redo log = innodb_log_file_size × innodb_log_files_in_group
```

MySQL 8.0.30+ 使用单个 `#innodb_redo` 目录下的多个文件，自动管理：

```sql
-- MySQL 8.0.30+ 查看 redo log 状态
SHOW VARIABLES LIKE 'innodb_redo_log_capacity';  -- 总容量（默认 100MB）
```

### Log Sequence Number（LSN）

LSN 是一个单调递增的序列号，标识 redo log 中的每个字节位置：

```sql
-- 查看当前 LSN 状态
SHOW ENGINE INNODB STATUS\G
-- Log sequence number:      当前写入位置
-- Log flushed up to:       已刷盘位置
-- Last checkpoint at:       最后 checkpoint 位置
```

## Checkpoint 机制

### 什么是 Checkpoint

Checkpoint（检查点）是 redo log 中的一个位置，表示**在此之前的脏页已全部刷盘**。其作用是：

1. **缩短崩溃恢复时间**：只需重放 checkpoint 之后的日志
2. **回收日志空间**：checkpoint 之前的日志可以被覆盖

### InnoDB 模糊检查点（Fuzzy Checkpointing）

InnoDB 使用**模糊检查点**，分批小量地刷新脏页，不会阻塞用户请求：

```sql
-- 控制刷新速度
SHOW VARIABLES LIKE 'innodb_io_capacity';      -- 后台刷新线程每秒 I/O 操作数
SHOW VARIABLES LIKE 'innodb_max_dirty_pages_pct';  -- 脏页比例阈值
```

## 崩溃恢复流程

当 MySQL 意外宕机后重启，InnoDB 按以下步骤恢复：

### 步骤一：确定起始点

读取 redo log 中的最后一个有效 checkpoint LSN。

### 步骤二：前滚（Roll Forward）

从 checkpoint 位置开始，重放所有 redo log 记录，将数据页恢复到崩溃前的状态。

### 步骤三：回滚（Roll Back）

对于未提交的事务，通过 undo log 回滚其更改。

### 示例

假设崩溃前：
- T1（事务ID=100）已提交
- T2（事务ID=101）未提交

恢复过程：
1. 读取 checkpoint LSN = 5000
2. 重放 LSN > 5000 的所有更改（T1 和 T2 的修改都被重放）
3. 检测到 T2 未提交，读取 undo log 回滚 T2 的更改
4. 最终数据状态与 T1 提交后、T2 回滚后一致

## 日志持久化策略

`innodb_flush_log_at_trx_commit` 控制事务提交时日志的刷盘行为：

| 值 | 含义 | 性能 | 安全性 |
|----|------|------|--------|
| **1** | 每次提交时强制刷盘（默认） | 最慢 | 最安全 |
| **2** | 提交时刷到 OS 缓存，定期刷盘 | 中等 | 可能丢失最近 1 秒的事务 |
| **0** | 完全依赖 OS 刷盘 | 最快 | 崩溃可能丢失大量事务 |

```sql
-- 高安全要求场景（金融、订单）
SET GLOBAL innodb_flush_log_at_trx_commit = 1;

-- 极高写入性能场景（日志收集）
SET GLOBAL innodb_flush_log_at_trx_commit = 2;
```

## Doublewrite Buffer

Doublewrite Buffer 是防止**页撕裂（Torn Page）**的机制：

1. 脏页先写入 doublewrite buffer（连续 128 页）
2. 再从 doublewrite buffer 写入数据文件
3. 如果刷盘过程崩溃，doublewrite 中仍有完整副本

```sql
-- 查看 doublewrite buffer 状态
SHOW VARIABLES LIKE 'innodb_doublewrite%';
```

## 实战调优建议

### Redo Log 大小规划

```sql
-- 高写入负载建议：redo log 总大小 = buffer pool 大小或更大
-- 太小会导致频繁 checkpoint，产生写抖动
SHOW VARIABLES LIKE 'innodb_log_file_size';
```

### 监控指标

```sql
-- 关键监控指标
SELECT 
    variable_name,
    variable_value
FROM performance_schema.global_status 
WHERE variable_name IN (
    'Innodb_log_write_requests',     -- 日志写请求数
    'Innodb_os_log_written',         -- 已写入日志字节数
    'Innodb_log_writes'              -- 物理日志写次数
);
```

### 常见问题处理

1. **频繁的 checkpoint 导致写入抖动**
   - 增大 redo log 文件总大小
   - 提高 `innodb_io_capacity`

2. **日志空间不足**
   - MySQL 8.0+：自动管理 `innodb_redo_log_capacity`
   - 早期版本：增加 `innodb_log_files_in_group`

## PostgreSQL WAL 简介

PostgreSQL 同样采用 WAL 机制，但实现有所不同：

- **WAL 文件位置**：`$PGDATA/pg_wal/`
- **Checkpoint 触发**：`checkpoint_timeout`（默认 5 分钟）或 `max_wal_size`
- **归档模式**：支持 WAL 归档用于 Point-in-Time Recovery（PITR）

## 参考资料

[^1]: [MySQL 8.4 Reference Manual - Redo Log](https://dev.mysql.com/doc/refman/en/innodb-redo-log.html)

[^2]: [MySQL 8.4 Reference Manual - InnoDB Checkpoints](https://dev.mysql.com/doc/refman/en/innodb-checkpoints.html)

[^3]: [How MySQL InnoDB Crash Recovery Works - OneUptime](https://oneuptime.com/blog/post/2026-03-31-mysql-how-mysql-innodb-crash-recovery-works/view)
