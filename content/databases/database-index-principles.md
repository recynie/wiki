---
title: 数据库索引原理
date: 2026-04-08
description: 数据库索引原理详解：B+树、索引类型、查询优化、事务隔离级别
tags:
  - database
  - index
  - b-tree
  - query-optimization
draft: false
permalink:
---

## 概述

数据库索引是用于加速数据检索的数据结构，类似于书籍的目录。[^1]

### 索引的代价

- **空间代价**：索引占用额外存储空间
- **更新代价**：插入/删除/更新需要维护索引

---

## B+树与索引

### B+树结构

B+树是数据库索引的基石，特别是InnoDB存储引擎：

```
            [50 | 100]
           /    |    \
      [20|30] [60|70] [110|130]
        /  \      \       \
      叶    叶     叶       叶 ... (链表连接)
```

**B+树的特征**：
1. 所有数据都存储在叶子节点
2. 叶子节点通过链表连接，便于范围查询
3. 非叶子节点只存储索引（分界值）

### B+树节点结构（InnoDB）

```cpp
struct InnoDBPage {
    PageHeader header;           // 页头部
    IndexNode* elements;        // 索引目录
    Record* records;            // 数据记录
    FreeSpace* free_space;      // 空闲空间
    PageDirectory footer;       // 页目录（槽）
};
```

### B+树操作复杂度

| 操作 | 时间复杂度 |
|------|-----------|
| 查找 | $O(\log_{m}{N})$ |
| 插入 | $O(\log_{m}{N})$ |
| 删除 | $O(\log_{m}{N})$ |
| 范围查询 | $O(\log_{m}{N} + k)$ |

其中 $m$ 是树的度，$N$ 是数据量，$k$ 是结果数量。

---

## 索引类型

### 主键索引（Primary Key）

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,      -- 主键索引，唯一非空
    name VARCHAR(50)
);
```

**特点**：
- 唯一非空
- 每个表只能有一个主键
- InnoDB中主键索引即聚簇索引

### 唯一索引（Unique Index）

```sql
CREATE UNIQUE INDEX idx_email ON users(email);
```

**特点**：
- 唯一但允许NULL
- 可有多个唯一索引

### 普通索引（Secondary Index）

```sql
CREATE INDEX idx_name ON users(name);
```

### 组合索引（Composite Index）

```sql
CREATE INDEX idx_dept_salary ON employees(dept_id, salary);
```

**最左前缀原则**：组合索引按最左列开始使用。

```sql
-- 索引 idx_dept_salary 可以加速：
WHERE dept_id = 10
WHERE dept_id = 10 AND salary > 5000

-- 但不能加速：
WHERE salary > 5000
```

### 覆盖索引（Covering Index）

查询的所有列都在索引中，无需回表：

```sql
-- 假设有索引 (dept_id, salary)
SELECT dept_id, salary FROM employees WHERE dept_id = 10;
-- 只需扫描索引，无需回表
```

---

## InnoDB vs MyISAM

### InnoDB（默认存储引擎）

| 特性 | 说明 |
|------|------|
| 聚簇索引 | 主键索引即数据 |
| 事务支持 | 支持ACID |
| 行级锁 | 并发性能好 |
| 外键约束 | 支持外键 |
| 崩溃恢复 | 有redo log |

### MyISAM

| 特性 | 说明 |
|------|------|
| 非聚簇索引 | 索引与数据分离 |
| 表级锁 | 并发性能差 |
| 无事务 | 不支持 |
| 压缩表 | 支持只读压缩 |

### 聚簇索引 vs 非聚簇索引

```
聚簇索引（InnoDB）：
┌─────────────────────────────────┐
│ 主键 → 数据行（直接存储）         │
└─────────────────────────────────┘

非聚簇索引（MyISAM）：
┌────────────┐    ┌────────────┐
│ 主键 → 行号 │ →  │ 行号 → 数据 │
└────────────┘    └────────────┘
```

---

## 查询优化

### EXPLAIN分析

```sql
EXPLAIN SELECT * FROM employees WHERE dept_id = 10;
```

**关键字段**：

| 字段 | 含义 |
|------|------|
| type | 连接类型（const, eq_ref, ref, range, ALL） |
| key | 实际使用的索引 |
| rows | 预计扫描行数 |
| Extra | 额外信息（Using index, Using filesort） |

### 优化策略

#### 1. 避免全表扫描

```sql
-- 低效：全表扫描
SELECT * FROM orders WHERE quantity > 0;

-- 高效：使用索引
SELECT * FROM orders WHERE order_id > 0 AND quantity > 0;
-- 确保order_id有索引
```

#### 2. 避免函数操作

```sql
-- 无法使用索引
SELECT * FROM users WHERE YEAR(created_at) = 2024;

-- 高效
SELECT * FROM users WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
```

#### 3. 避免LIKE前导通配符

```sql
-- 无法使用索引
SELECT * FROM products WHERE name LIKE '%phone%';

-- 高效：使用后置通配符
SELECT * FROM products WHERE name LIKE 'iPhone%';
```

#### 4. 使用覆盖索引

```sql
-- 只需扫描索引
SELECT order_id, amount FROM orders WHERE customer_id = 100;
-- 如果有 idx_customer_order(customer_id, order_id, amount)
```

### 慢查询分析

```sql
-- 查看慢查询日志配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 分析查询
SHOW PROCESSLIST;
```

---

## 事务与隔离级别

### ACID特性

| 特性 | 说明 |
|------|------|
| Atomicity | 原子性：事务要么全做，要么全不做 |
| Consistency | 一致性：事务前后数据库状态一致 |
| Isolation | 隔离性：并发事务相互隔离 |
| Durability | 持久性：提交后永久生效 |

### 隔离级别

| 级别 | 脏读 | 不可重复读 | 幻读 |
|------|------|-----------|------|
| Read Uncommitted | 可能 | 可能 | 可能 |
| Read Committed | 不可能 | 可能 | 可能 |
| Repeatable Read | 不可能 | 不可能 | 可能 |
| Serializable | 不可能 | 不可能 | 不可能 |

### 隔离级别实现

#### MVCC（多版本并发控制）

```cpp
class MVCCStore {
    struct Version {
        int txn_id;        // 创建版本的事务ID
        int begin_ts;       // 开始时间戳
        int end_ts;        // 结束时间戳
        string data;       // 数据值
        bool committed;    // 是否已提交
    };
    
    map<string, vector<Version>> versions;  // key → 版本链
    
    string read(string key, int txn_id) {
        auto& version_chain = versions[key];
        for (auto it = version_chain.rbegin(); it != version_chain.rend(); ++it) {
            if (it->begin_ts <= txn_id && 
                (it->end_ts > txn_id || it->end_ts == -1) &&
                it->committed) {
                return it->data;
            }
        }
        return "";
    }
};
```

#### Next-Key Locking（行锁+间隙锁）

防止幻读：锁定索引范围+区间间隙。

```sql
-- 事务A：
SELECT * FROM orders WHERE amount > 100 FOR UPDATE;
-- 锁定 amount > 100 的行及相邻区间

-- 事务B无法插入 amount 在 100-∞ 范围内的新记录
```

---

## 索引设计原则

### 何时创建索引

1. WHERE子句经常使用的列
2. JOIN操作的连接列
3. ORDER BY/GROUP BY的列
4. SELECT中 Distinct的列

### 索引失效场景

1. 列进行函数运算
2. LIKE前导通配符
3. OR连接非索引列
4. 数据类型隐式转换
5. 使用NOT、<>操作

### 索引优化建议

```sql
-- 1. 选择性高的列优先
SELECT COUNT(DISTINCT country) / COUNT(*) FROM customers;
-- 选择性越高，索引越有效

-- 2. 前缀索引（长字符串列）
ALTER TABLE users ADD INDEX idx_email(EMAIL(10));

-- 3. 复合索引顺序
-- 考虑：区分度、查询频率、空间
```

---

## 参考资料

[^1]: 本词条参考[数据库索引 - 维基百科](https://zh.wikipedia.org/wiki/数据库索引)

---

## 相关主题

- [[database-overview|数据库系统概述]] - SQL基础、事务ACID
- [[b-tree|B树与B+树]] - 索引的底层结构
