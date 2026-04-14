---
title: LSM Tree（Log-Structured Merge-Tree）
date: 2026-04-13
description: LSM树是一种写优化的数据结构，广泛应用于RocksDB、LevelDB、Cassandra等数据库
tags:
  - databases
  - lsm-tree
  - storage-engine
draft: false
permalink:
---

## 概述

LSM 树（Log-Structured Merge-Tree）是一种针对写密集型工作负载优化的数据结构。与 B 树不同，LSM 树不进行原地更新，而是将所有修改顺序写入内存，然后异步合并到磁盘。

**核心思想**：将随机写转化为顺序写，最大化写吞吐量的同时接受读和空间放大作为权衡。

## 架构组件

### 1. MemTable（内存表）

- 有序内存结构（通常为 SkipList 或 Red-Black Tree）
- 接收所有写操作
- 达到阈值后 flush 到磁盘

### 2. Immutable MemTable（只读内存表）

- MemTable flush 期间的过渡结构
- 正在被序列化到磁盘

### 3. SSTable（Sorted String Table）

- 磁盘上的不可变有序文件
- 包含数据块、索引块、布隆过滤器

### 4. WAL（Write-Ahead Log）

- 保证 MemTable 数据的持久性
- 崩溃恢复的依据

## 写入流程

```
┌──────────────────────────────────────────────────────────────┐
│                         写入流程                             │
└──────────────────────────────────────────────────────────────┘

1. 写入请求 ──► 写入 WAL（保证持久性）
              │
              ▼
2. 写入 MemTable（SkipList，按 key 排序）
              │
              ▼
3. 返回成功给客户端
              │
              ▼ (MemTable 达到阈值)
4. 转换为 Immutable MemTable
              │
              ▼
5. 后台线程 flush 到 L0 层 SSTable
```

### 代码示例（RocksDB 风格）

```cpp
// 写入操作
void Put(const Slice& key, const Slice& value) {
    // 1. 写入 WAL
    log->AddRecord({key, value});
    
    // 2. 写入 MemTable
    memtable->Add(key, value);
}

// MemTable flush
void FlushMemTable() {
    ImmutableMemTable* imm = memtable_->RefAsImmutable();
    SSTFileWriter writer;
    
    // 按 key 排序写入 SSTable
    for (auto& kv : *imm) {
        writer.Add(kv.key, kv.value);
    }
    writer.Finish();
    
    // 添加到 L0
    AddSSTableToL0(writer.filename());
}
```

## 读取流程

读取需要从最新层到最旧层逐层查找：

```
┌─────────────────────────────────────────────┐
│              读取流程                        │
└─────────────────────────────────────────────┘

Get(key):
  1. 检查 MemTable（最新数据）
  2. 检查 Immutable MemTables
  3. 从 L0 到 Ln 按层查找 SSTable
     ├── 检查 Bloom Filter（快速判断是否存在）
     ├── 二分查找索引块定位数据块
     └── 在数据块中查找
  4. 返回最新版本（时间戳最大的）
```

### 时间复杂度

最坏情况下需要检查所有层：**$O(N)$**

但实际中：
- Bloom Filter 可以 $O(1)$ 判断不存在
- 通常只需检查少数几层

## 合并策略（Compaction）

### Leveled Compaction

```
L0: [f1] [f2] [f3]     ← 新 flush 的文件，可能重叠
L1: [a-k] [l-z]        ← 按 key 范围分区，无重叠
L2: [a-f][g-m][n-t][u-z]
L3: ...
```

**特点**：
- 每层大小约为前一层的 10 倍
- 90% 数据存储在最后一层
- 空间放大约 10x

### Tiered Compaction（Universal）

```
Level 0: [f1] [f2] [f3] [f4]  ← 多个重叠文件
Level 1: [a-z]                  ← 单个排序run
Level 2: [a-z]                  ← 更大的排序run
```

**特点**：
- 写放大更低
- 读可能需要合并更多文件
- 空间放大可能较高

## LSM vs B-Tree

| 特性 | LSM Tree | B-Tree |
|------|----------|--------|
| 写放大 | 低（顺序写） | 高（随机读写） |
| 读放大 | 高（多层查找） | 低（单次查找） |
| 空间放大 | 1.5-2x | 1.1-1.2x |
| 写吞吐 | 高 | 中 |
| 读延迟 | 不稳定 | 稳定 |
| 内存 | 较少 | 较多（指针对象） |

## 删除与 Tombstone

LSM 树不支持原地删除，而是使用 **Tombstone** 标记删除：

```
写入: key="foo", value="bar"
删除: key="foo", type=Tombstone  ← 墓碑标记

Compaction 时:
  key="foo"(tombstone) + key="foo"(v1) → 都丢弃（真正的删除）
```

**问题**：墓碑可能长期存在直到被压缩掉

## 变体与实现

### LevelDB

- Google 开发
- 简单、高效
- RocksDB 的前身

### RocksDB

- Facebook 开发，LevelDB 的优化版本
- 多核 CPU 优化
- 支持列族、事务、备份

### Cassandra

- 分布式 NoSQL
- 使用 Leveled Compaction
- 适合写入密集型

### TiKV

- 使用 RocksDB 作为存储引擎
- 存储 Raft 日志和用户数据

## 配置参数（RocksDB）

```cpp
Options options;

// MemTable 配置
options.write_buffer_size = 64 * 1024 * 1024;  // 64MB
options.max_write_buffer_number = 3;

// Compaction 配置
options.level_compaction_dynamic_level_bytes = true;
options.level0_file_num_compaction_trigger = 4;
options.level0_slowdown_writes_trigger = 20;
options.max_bytes_for_level_base = 256 * 1024 * 1024;  // 256MB

// Bloom Filter
options.filter_policy = NewBloomFilterPolicy(10);
```

## 优缺点总结

### 优点

- ✅ 极快的顺序写入
- ✅ 高写入吞吐
- ✅ 优秀的磁盘空间利用
- ✅ 崩溃恢复简单（WAL）

### 缺点

- ❌ 读取需要合并多个源
- ❌ 后台合并 I/O 开销
- ❌ 合并期间延迟抖动
- ❌ 写放大（同一数据被重写多次）

## 适用场景

- 写入密集型负载（日志、时序数据、消息队列）
- 需要高吞吐的键值存储
- SSD/Flash 存储优化

## 参考

- [LSM Trees - DEV Community](https://dev.to/godofgeeks/lsm-trees-log-structured-merge-trees-3me4)
- [RocksDB Documentation](https://rocksdb.org/docs/getting-started.html)
- [LSM Tree Paper](https://www.cs.umass.edu/~emery/memory/papers/wils94/paper.pdf)
- [Leveled Compaction - RocksDB Wiki](https://github.com/facebook/rocksdb/wiki/Leveled-Compaction)
