---
title: Garbage Collection（垃圾回收算法）
date: 2026-04-13
description: 垃圾回收算法详解：标记-清除、标记-压缩、复制算法、分代回收
tags:
  - system-software
  - memory-management
  - garbage-collection
draft: false
permalink:
---

## 概述

垃圾回收（Garbage Collection，GC）是一种自动内存管理机制，通过自动回收不再使用的内存来防止内存泄漏。本文介绍经典的垃圾回收算法及其变体。

## 基本概念

### 垃圾定义

**可达对象**：从根集合（GC Roots）出发可以访问到的对象。

**垃圾对象**：无法从根集合出发访问到的对象，即程序不再需要的内存。

### GC Root

GC Root 包括：
- 栈帧中的局部变量引用
- 静态字段引用
- JNI 句柄
- 运行中的线程
- 正在等待的线程

## 追踪算法（Tracing Algorithms）

追踪算法通过遍历对象引用图来识别存活对象。

## 标记-清除（Mark-Sweep）

### 算法流程

```
1. Mark 阶段：遍历所有可达对象，标记为存活
2. Sweep 阶段：扫描堆，清除所有未标记的对象
```

### 优点

- 实现相对简单
- 不需要移动对象

### 缺点

- **内存碎片化**：清除后产生不连续的空闲内存
- **分配效率低**：需要维护空闲链表或位图
- **暂停时间长**：需要 Stop-the-World

### 内存碎片化示意

```
清除前: [使用][空闲][使用][使用][空闲][空闲][使用]
清除后: [使用][使用][使用][    ][    ][    ][    ]  ← 碎片
```

## 标记-压缩（Mark-Compact）

### 算法流程

```
1. Mark 阶段：同 Mark-Sweep
2. Compact 阶段：将所有存活对象移动到一端
```

### 压缩策略

#### 表格法（Table-based）

记录每个对象的旧地址和新地址的映射表，最后更新所有引用。

#### 滑动压缩（Sliding Compaction）

将所有存活对象滑动到堆的一端，保持原有相对顺序。

### 优点

- 消除内存碎片
- 分配简单（指针碰撞）

### 缺点

- 需要移动所有存活对象
- 需要更新所有对象引用
- 暂停时间较长

## 复制算法（Copying）

### 算法流程

将堆分为两块（From 和 To）空间，每次只使用一块：

```
1. 活动对象从 From 复制到 To
2. 交换 From 和 To 角色
3. 清空原来的 From 空间
```

### 优点

- 无内存碎片
- 分配简单（指针碰撞）
- 适合"朝生夕死"的对象

### 缺点

- 可用内存减半
- 需要复制所有存活对象

### Cheney's Algorithm

宽度优先复制算法，避免递归：

```cpp
void copy() {
    int scan = 0, free = 0;
    for (int i = 0; i < scan; i++) {
        for each field in object[i]:
            if (is_pointer(field)) {
                obj = dereference(field);
                if (!is_forwarded(obj)) {
                    forwarding[obj] = &to_space[free++];
                    copy(obj);
                }
                update_pointer(field, forwarding[obj]);
            }
    }
}
```

## 分代回收（Generational GC）

### 弱分代假说

> 大多数对象"朝生夕死"（infant mortality）

经验表明：大多数对象在创建后很快变得不可达。

### 堆分代

```
┌─────────────────────────────────────────┐
│  Old Generation (老年代)                  │
│    - 生命周期长的对象                      │
│    - 回收频率低                           │
├─────────────────────────────────────────┤
│  Survivor Spaces (幸存区) S0 | S1        │
│    - 经历 Minor GC 后存活的对象           │
├─────────────────────────────────────────┤
│  Eden (伊甸园区)                         │
│    - 新创建的对象                          │
└─────────────────────────────────────────┘
```

### 回收流程

```
Minor GC (Young GC):
  1. 回收 Eden 和 Survivor 区
  2. 存活对象复制到另一个 Survivor 或 Old 区

Major GC (Full GC):
  1. 回收整个堆
  2. 通常伴随 Stop-the-World
```

### 卡表（Card Table）

为跟踪老年代到年轻代的引用，使用卡表标记脏页：

```
Card Table: [0][0][1][0][1][0][0][1]
            ↓   ↓   ↓   ↓   ↓   ↓   ↓   ↓
         512B 512B 512B ... (每个卡对应 512B 堆区域)
```

### 写入屏障（Write Barrier）

在写操作时记录跨代引用：

```cpp
void write_barrier(object* obj, field* f, object* new_val) {
    *f = new_val;
    if (object_gen(new_val) != object_gen(obj)) {
        card_table[card_of(obj)] = DIRTY;
    }
}
```

## 三色标记算法

三色标记是增量/并发 GC 的基础框架：

| 颜色 | 含义 |
|------|------|
| 白色（White） | 尚未访问 |
| 灰色（Grey） | 已访问，但未处理完引用 |
| 黑色（Black） | 已访问，处理完所有引用 |

### 标记过程

```
初始: 所有对象标记为白色，GC Root 标记为灰色

1. 从灰色集合取出一个对象
2. 标记为黑色
3. 将其所有白色引用对象标记为灰色

重复直到灰色集合为空
结果: 白色对象为垃圾
```

### 并发标记的问题

**漏标问题**：黑色对象被错误地认为存活

解决方案：
- **SATB（Snapshot At The Beginning）**：记录标记开始时的快照
- **增量更新**：记录黑色到白色引用的变化

## 各语言 GC 策略

### Java（JVM）

| GC 类型 | 策略 |
|--------|------|
| Serial GC | 串行 Mark-Compact（年轻代 Copying） |
| Parallel GC | 并行 Mark-Compact |
| CMS | 并发 Mark-Sweep，尽量避免压缩 |
| G1 | 分区式 Mark-Compact |
| ZGC | 着色指针 + 并发压缩 |
| Shenandoah | 无暂停压缩 |

### Go

Go 使用并发三色标记和清除：
- 写屏障：插入和删除屏障组合
- 清除与标记并发执行
- 基于 Span 的内存管理

### Python

使用引用计数 + 标记-清除处理循环引用：
- 优点：实时回收
- 缺点：无法处理循环引用、暂停不确定

## 增量式与并发 GC

### 增量式 GC

将 GC 工作分成小块，与 mutator 交替执行：

```
Mutator → GC Work → Mutator → GC Work → ...
```

### 并发 GC

GC 与 mutator 同时运行：

```
Mutator:    [运行]----[运行]----[运行]
GC:      [标记]--[标记]--[清除]--[暂停]
```

需要写入屏障维持一致性。

## 性能指标

| 指标 | 含义 |
|------|------|
| 吞吐量 | Mutator 运行时间 / 总时间 |
| 暂停时间 | Stop-the-World 暂停时长 |
| 内存开销 | 元数据额外占用 |
| 延迟抖动 | 暂停时间方差 |

## 参考

- [The Garbage Collection Handbook](https://www.oracle.com/library/1911-GC.pdf)
- [GC Survey Paper](https://web.cecs.pdx.edu/~apt/cs577_2018/gcsurvey.pdf)
- [Java Generations](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/generations.html)
