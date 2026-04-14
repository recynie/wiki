---
title: 进程调度算法
date: 2026-04-13
description: 深入理解操作系统CPU调度算法，从FIFO到CFS的演进
tags:
  - operating-systems
  - scheduling
  - process
  - cpu
draft: false
permalink:
---

## 概述

CPU调度是操作系统核心理论之一。当多个进程竞争CPU时，调度器决定哪个进程运行、何时运行、运行多久。现代操作系统需要同时满足高吞吐量、低延迟、公平性和效率等目标。[^1]

## 调度目标

调度算法追求以下目标的平衡：

| 目标 | 说明 |
|------|------|
| **CPU利用率** | 保持CPU忙碌，最大化资源利用 |
| **吞吐量** | 单位时间完成的进程数 |
| **周转时间** | 进程从提交到完成的总时间 |
| **等待时间** | 进程在就绪队列中的总等待时间 |
| **响应时间** | 从提交到首次响应的时间 |

> 没有任何调度算法能在所有指标上同时最优——这是调度领域的根本性权衡（fundamental trade-off）。

## 基本概念

### 非抢占式 vs 抢占式

- **非抢占式（Non-preemptive）**：进程运行直到阻塞（I/O）或主动让出CPU
- **抢占式（Preemptive）**：调度器可以在任意时刻打断进程，强制切换

### 调度时机

调度决策发生在以下时刻：
1. 进程从运行态切换到阻塞态（如等待I/O）
2. 进程从运行态切换到就绪态（时间片用完）
3. 进程从阻塞态切换到就绪态（如I/O完成）
4. 进程终止

## 经典调度算法

### 1. FIFO（先进先出）

最简单的调度策略：

```
进程到达顺序: P1(24ms), P2(3ms), P3(3ms)
执行顺序: P1 → P2 → P3

时间线:
|-------P1-------||P2|-P3-|
0              24   27  30

平均周转时间: (24 + 3 + 6) / 3 = 11ms
平均等待时间: (0 + 24 + 27) / 3 = 17ms
```

**特点**：
- 简单易实现
- 对长作业友好
- 短作业可能等待很久（护航效应）

### 2. SJF（最短作业优先）

优先调度最短的进程：

```
同一例子:
执行顺序: P2(3ms) → P3(3ms) → P1(24ms)

平均周转时间: (3 + 6 + 30) / 3 = 13ms
平均等待时间: (0 + 3 + 6) / 3 = 3ms
```

**最优性**：SJF是**最小化平均等待时间**的最优算法。

**问题**：需要事先知道作业长度，实际难以实现。

### 3. STCF（最短剩余时间优先）

SJF的抢占式版本：

```
P1(24ms)在0时刻到达并开始执行
P2(3ms)在3时刻到达 → 抢占P1
P2执行完毕后，P1继续执行

时间线:
|--P1--|P2|-P1----------|
0      3   6            30

等待时间 P1: 3ms (被P2抢占)
等待时间 P2: 0ms
```

### 4. Round Robin（时间片轮转）

每个进程执行一个时间片（time quantum），然后放回队列：

```
时间片 = 4ms
执行顺序: P1 → P2 → P3 → P1 → P1...

时间线:
|P1|P2|P3|P1|P1|P1|P1|P1|
0  4  7 10 14 18 22 26 30

平均响应时间: 远优于SJF（首次响应快速）
平均周转时间: 可能较差（上下文切换开销）
```

**关键参数**：时间片大小
- 太短：上下文切换开销大
- 太长：响应时间变差

### 响应时间 vs 周转时间

这是调度的根本矛盾：

| 算法 | 响应时间 | 周转时间 |
|------|---------|---------|
| SJF/STCF | 较差 | 最优 |
| RR | 最优 | 较差 |

## Linux调度器

### 调度类（Scheduling Classes）

Linux采用模块化调度框架，按优先级从高到低：

```
1. Stop Scheduler     ← 最高优先级，用于进程停止/迁移
2. RT Scheduler      ← 实时进程（SCHED_FIFO, SCHED_RR）
3. Fair Scheduler    ← 普通进程（CFS）
4. Idle Scheduler    ← 空闲进程
```

### 实时调度策略

#### SCHED_FIFO

- 无时间片限制
- 相同优先级按FIFO顺序
- 被更高优先级进程抢占

#### SCHED_RR

- 有固定时间片
- 相同优先级轮转执行

```bash
# 设置实时调度
chrt -f 99 ./realtime_program  # FIFO，优先级99
chrt -r 50 ./rt_program        # RR，优先级50

# 查看进程调度信息
ps -eo pid,ni,cls,pri,cmd
```

### CFS（完全公平调度器）

Linux 2.6.23引入的默认调度器，目标是给每个任务**等比例的CPU时间**。[^2]

#### 虚拟运行时间（vruntime）

```c
// CFS核心概念
struct sched_entity {
    u64 vruntime;  // 虚拟运行时间
    u64 sum_exec_runtime;
    // ...
};
```

调度决策：选择vruntime最小的进程执行

#### 红黑树

CFS使用红黑树管理就绪进程：

```
        进程C (vruntime=10)
              │
    ┌─────────┴─────────┐
    │                    │
  进程A(5)            进程B(15)
```

新进程插入，vruntime最小的在最左侧，被优先调度。

#### 权重（Weight）

Nice值影响权重：

```
nice值    权重      相对CPU份额
 -20     1024     ~50%
 -10      205     ~10%
   0      102     ~5%
  +10      15     ~0.8%
  +19        1     ~0.05%
```

### 调度策略选择

```bash
# 普通进程（默认）
chrt -o 0 ./program   # SCHED_OTHER

# 批处理
chrt -b 0 ./program   # SCHED_BATCH

# 实时（需root）
chrt -f 50 ./program  # SCHED_FIFO
```

## 多级反馈队列（MLFQ）

结合多种策略的实用调度器：

```
┌──────────────────────────────────────┐
│  Queue 0: RR (time quantum = 8ms)   │ ← 最高优先级
├──────────────────────────────────────┤
│  Queue 1: RR (time quantum = 16ms)  │
├──────────────────────────────────────┤
│  Queue 2: RR (time quantum = 32ms)  │
├──────────────────────────────────────┤
│  Queue 3: FCFS                      │ ← 最低优先级
└──────────────────────────────────────┘
```

**规则**：
1. 新进程进入Queue 0
2. 用完时间片 → 降到下一队列
3. 被阻塞 → 保持当前队列
4. 超过队列阈值 → 进入下一队列

## 优先级反转与PI Mutex

### 问题

```
T1（高优先级）需要资源X
T2（中优先级）正在运行
T3（低优先级）持有资源X

T1等待T3释放X
T3被T2抢占无法运行
T1被间接阻塞 → 优先级反转
```

### 解决方案：优先级继承

低优先级进程继承等待其资源的最高优先级进程的优先级。

```c
// Linux PI Mutex实现
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
```

## 调度器配置与调优

### 常用命令

```bash
# 查看调度信息
cat /proc/schedstat
cat /proc/sched_debug

# 实时监控
top → 按r查看调度信息
htop → F5查看进程树

# CPU亲和性
taskset -c 0,1 ./program  # 绑定到特定CPU
```

### 内核参数

```bash
# 查看
sysctl kernel.sched_latency_ns
sysctl kernel.sched_min_granularity_ns

# 临时修改
sysctl -w kernel.sched_latency_ns=10000000
```

### 延迟敏感应用

```bash
# 使用实时调度
chrt -f 99 -p $$   # 当前shell使用FIFO优先级99

# 绑核运行
taskset -c 0 ./latency_sensitive_app
```

## 调度指标分析

### 调度延迟

- **调度延迟**：从事件发生到新进程开始运行的时间
- **目标**：< 1ms（高优先级实时任务）

### 吞吐量

```bash
# 计算吞吐量
vmstat 1
# procs
# r: 运行队列长度
```

### CPU利用率

```
理想状态: CPU利用率 ≈ 100%（排除空闲任务）
```

## 最佳实践

1. **识别工作负载类型**
   - CPU密集型：计算优先
   - I/O密集型：快速响应优先

2. **使用合适的调度策略**
   - 数据库：CPU密集 → SCHED_BATCH
   - 实时控制：确定性 → SCHED_FIFO
   - 通用应用：SCHED_OTHER（CFS）

3. **避免优先级反转**
   - 使用PI Mutex保护共享资源
   - 减少锁持有时间

4. **监控与调优**
   - 关注调度延迟和上下文切换次数
   - 过高context switch可能表示调度问题

## 参考资料

[^1]: [Operating Systems: Three Easy Pieces - CPU Scheduling](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched.pdf)
[^2]: [Linux Kernel Scheduling Explained - Medium](https://medium.com/@majidbasharat21/linux-kernel-scheduling-explained-how-processes-are-prioritized-6c086ddb096a)
