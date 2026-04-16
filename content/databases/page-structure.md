---
title: Page Structure
date: 2026-04-15
description: 数据库页结构与存储格式详解
tags:
  - database
  - storage-engine
  - page-structure
  - innodb
draft: false
permalink:
---

## 简介

数据库以**页（Page）**作为基本存储单位。InnoDB 默认页大小为 **16KB**，是数据读写、缓存、刷盘的最小单位。

## InnoDB Page 类型

InnoDB 使用不同页类型管理不同数据：

| Page Type | 说明 |
|-----------|------|
| `FIL_PAGE_INDEX` | B-tree 索引页（数据 + 二级索引） |
| `FIL_PAGE_UNDO_LOG` | Undo 日志页（MVCC 回滚历史） |
| `FIL_PAGE_INODE` | 段描述符页（管理 extent） |
| `FIL_PAGE_IBUF_FREE_LIST` | Insert Buffer 空闲列表 |
| `FIL_PAGE_TYPE_SYS` | 系统页 |
| `FIL_PAGE_TYPE_BLOB` | 溢出页（大字段存储） |

## InnoDB Index Page 布局

一个完整的 B-tree 索引页（`FIL_PAGE_INDEX`）布局如下：

```
┌──────────────────────────────────────────────────────────────────┐
│                        File Header (38 bytes)                     │
│  页号、Space ID、校验和、LSN、页类型                                │
├──────────────────────────────────────────────────────────────────┤
│                        Page Header (56 bytes)                     │
│  Heap top、Free list、Record count、Direction                      │
├──────────────────────────────────────────────────────────────────┤
│                    Infimum Record (13 bytes)                      │
│  虚拟最小记录，作为链表哨兵                                         │
├──────────────────────────────────────────────────────────────────┤
│                    Supremum Record (13 bytes)                     │
│  虚拟最大记录，作为链表哨兵                                         │
├──────────────────────────────────────────────────────────────────┤
│                         User Records                               │
│  实际数据记录（从顶部向下增长）                                      │
│  通过 next_record 指针构成单向链表                                  │
├──────────────────────────────────────────────────────────────────┤
│                          Free Space                                │
│  空闲空间（中间区域）                                               │
├──────────────────────────────────────────────────────────────────┤
│                       Page Directory                              │
│  槽数组，每槽 2 字节，指向 records（从底部向上增长）                  │
├──────────────────────────────────────────────────────────────────┤
│                      File Trailer (8 bytes)                       │
│  校验和 + LSN 末尾，用于检测页损坏                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 各区域详解

#### File Header（偏移 0-38）

| 字段 | 大小 | 说明 |
|------|------|------|
| `FIL_PAGE_SPACE_OR_CHKSUM` | 4 | 页校验和 |
| `FIL_PAGE_OFFSET` | 4 | 页号 |
| `FIL_PAGE_PREV` | 4 | 前一页（用于 B-tree 叶子节点链表） |
| `FIL_PAGE_NEXT` | 4 | 下一页（用于 B-tree 叶子节点链表） |
| `FIL_PAGE_LSN` | 8 | 最后修改的日志序列号 |
| `FIL_PAGE_TYPE` | 2 | 页类型 |
| `FIL_PAGE_FILE_FLUSH_LSN` | 8 | 仅在 SYSTEM 表空间中使用的 LSN |
| `FIL_PAGE_ARCH_LOG_NO` | 4 | 归档日志号 |

#### Page Header（偏移 38-94）

| 字段 | 大小 | 说明 |
|------|------|------|
| `PAGE_N_HEAP` | 2 | 堆中的记录数（包括 infimum/supremum） |
| `PAGE_FREE` | 2 | 空闲空间首地址 |
| `PAGE_GARBAGE` | 2 | 已删除记录占用的字节数 |
| `PAGE_LAST_INSERT` | 2 | 最后插入位置 |
| `PAGE_DIRECTION` | 2 | 最后插入方向 |
| `PAGE_N_DIRECTION` | 2 | 同方向连续插入次数 |
| `PAGE_N_RECS` | 2 | 用户记录数（不含 infimum/supremum） |
| `PAGE_MAX_TRX_ID` | 8 | 最大事务ID（仅叶子节点） |
| `PAGE_LEVEL` | 2 | B-tree 中的层级（叶子节点 = 0） |
| `PAGE_INDEX_ID` | 8 | 索引 ID |

#### Infimum 和 Supremum

每个页包含两个**虚拟记录**作为哨兵：
- **Infimum**：比任何用户记录都小，链表的起始点
- **Supremum**：比任何用户记录都大，链表的终点

用户记录通过 `next_record` 指针构成单向链表：`infimum → record1 → record2 → ... → supremum`

## Record 格式（COMPACT Row Format）

COMPACT 是 MySQL 5.0+ 引入的记录格式，相比 REDUNDANT 节省约 20% 空间。

### 记录头（5 字节）

```
┌─────────┬─────────┬──────────┬─────────────┬─────────────────┐
│ offset  │ offset  │          │             │                 │
│ 1-2     │ 3-4     │ bit 5-8  │ bit 1-4     │ bit 5-8         │
│ next_record │ heap_no |   flags   │ owned_count | record_type    │
└─────────┴─────────┴──────────┴─────────────┴─────────────────┘
```

- **next_record**：到下一记录的偏移
- **heap_no**：在堆中的位置序号
- **flags**：`deleted_flag`、`min_rec_flag`
- **owned_count**：此记录拥有的 page directory 槽数
- **record_type**：`0=普通记录`, `1=B-tree 非叶子节点`, `2=infimum`, `3=supremum`

### 变长列长度数组

每个变长列（VARCHAR, VARBINARY）占用 1-2 字节：
- 长度 ≤ 255：1 字节
- 长度 > 255：2 字节

### NULL Bitmap

允许 NULL 的列，每列对应 1 位。

## Row Format 对比

| 特性 | REDUNDANT | COMPACT | DYNAMIC | COMPRESSED |
|------|-----------|---------|---------|------------|
| 引入版本 | MySQL 4.1 | MySQL 5.0 | MySQL 5.7 | MySQL 5.7 |
| 空间节省 | 基准 | ~20% | 与 COMPACT 相同 | 最大 |
| 变长列存储 | 768字节+指针 | 768字节+指针 | 完全 off-page | 完全 off-page |
| 前缀索引 | 不支持 | 不支持 | 支持 3072 字节 | 支持 3072 字节 |
| 压缩 | 否 | 否 | 否 | 是 |

### DYNAMIC/COMPACT 的 Off-page 策略

当列值过长无法放入 B-tree 节点时：

1. **COMPACT/DYNAMIC**：存储前 768 字节 + 20 字节指针
2. **完全 off-page**：仅存储 20 字节指针，实际数据在独立的溢出页链表

```sql
-- 查看表使用的 row format
SELECT row_format 
FROM information_schema.tables 
WHERE table_schema = 'mydb' AND table_name = 'orders';
```

## Overflow Page（溢出页）

### 何时使用

当 BLOB、TEXT、VARCHAR 等变长列无法fit入页时，使用溢出页存储：

```sql
-- TEXT/BLOB <= 40 字节总是 in-row 存储
-- 超过限制则 off-page 存储
```

### 溢出页链表

每个 off-page 列有一个单向链表：
```
B-tree 记录 → Overflow Page 1 → Overflow Page 2 → ... → Overflow Page N
  (20字节指针)
```

## Page Directory（二分查找）

### 结构

页底部的 Page Directory 是**有序的 2 字节槽数组**，每个槽指向一条记录。

### 查找过程

1. 二分查找定位槽范围
2. 线性扫描槽内记录链表

这使得页内查找时间从 O(n) 降为 O(log n + k)。

## Checksum 与数据完整性

### 校验机制

- **File Header**：存储页校验和
- **File Trailer**：存储校验和 + LSN 末尾

### 校验和算法

```sql
SHOW VARIABLES LIKE 'innodb_checksum_algorithm';
-- crc32（默认，硬件加速）
-- strict_* 变体会拒绝无效校验和的页
```

读取页时重新计算校验和并与存储值比较，不一致则报告错误。

## B+Tree 中的 Page 结构

### 叶子节点

- 存储完整的行数据（聚簇索引）
- 通过 `FIL_PAGE_PREV/NEXT` 构成双向链表
- 包含 `DB_TRX_ID` 和 `DB_ROLL_PTR`（MVCC 用）

### 非叶子节点

- 存储索引键 + 子页号（而不是行数据）
- 不包含 MVCC 字段
- 通过子页号导航 B-tree

## 实战注意事项

### 行大小限制

单行数据必须能够放入一个页：
- 所有 in-row 列的总和 ≤ 页大小的一半（约 8KB）
- 超出的列必须 off-page 存储

### 页碎片

频繁更新会导致页内碎片：
- 删除记录后空间进入 `PAGE_GARBAGE`
- 新记录优先复用 `PAGE_FREE` 空间

## 参考资料

[^1]: [MySQL 8.4 Reference Manual - InnoDB Row Formats](https://dev.mysql.com/doc/refman/en/innodb-row-format.html)

[^2]: [The physical structure of InnoDB index pages - Jeremy Cole](https://blog.jcole.us/2013/01/07/)

[^3]: [How to Understand InnoDB Page Structure - OneUptime](https://oneuptime.com/blog/post/2026-03-31-mysql-innodb-page-structure/view)
