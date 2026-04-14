---
title: 形式化验证入门
date: 2026-04-13
description: 形式化验证（Formal Verification）— 使用数学方法证明系统正确性，包括TLA+、模型检验、Coq证明助手等工具和技术
tags:
  - formal-verification
  - tla+
  - model-checking
  - distributed-systems
draft: false
permalink:
---

## 概述

形式化验证是使用数学方法证明系统正确性的技术。与传统测试不同，形式化验证能够**穷尽检查**所有可能的系统行为，发现传统测试难以发现的边界情况错误。

### 核心问题

> 如何数学地证明一个系统永远不会出错？

### 形式化验证 vs 测试

| 方面 | 传统测试 | 形式化验证 |
|------|----------|------------|
| 覆盖度 | 有限测试用例 | 所有可能状态 |
| 正确性 | 发现bug | 证明无bug |
| 成本 | 相对较低 | 较高（需要规范） |
| 适用场景 | 通用 | 关键系统 |

## TLA+ 概述

TLA+（Temporal Logic of Actions）是由 Leslie Lamport 开发的形式化规范语言，用于描述并发和分布式系统的行为。

### 核心概念

1. **状态机模型**：系统行为被建模为状态转换
2. **时序逻辑**：描述系统随时间变化的性质
3. **模型检验**：自动穷举检查所有可能状态

### TLA+ 的应用场景

- 分布式系统（Paxos、Raft）
- 并发算法（互斥、死锁检测）
- 协议规范（TLS、TCP）
- 存储系统（文件系统、数据库）

## TLA+ 基础语法

### 模块结构

```tla
---- MODULE AtomicSnapshot ----
EXTENDS Naturals, Sequences, FiniteSets

VARIABLES
    (\* 变量声明 \*)
    x, y,
    procState

(\* 初始状态 \*)
Init ==
    /\ x = 0
    /\ y = 0
    /\ procState \in [Proc -> {"init", "active", "done"}]

(\* 下一步动作 \*)
Next ==
    \E i \in Proc:
        /\ procState[i] = "active"
        /\ procState' = [procState EXCEPT ![i] = "done"]
        /\ UNCHANGED <<x, y>>

(\* 规范 \*)
Spec == Init /\ [][Next]_vars

==== END MODULE ====
```

### 关键语法元素

| 语法 | 含义 |
|------|------|
| `VARIABLES` | 声明状态变量 |
| `Init` | 初始状态谓词 |
| `Next` | 下一状态关系 |
| `[]` | 总是（always）时序操作符 |
| `\E` | 存在量词 |
| `\A` | 全称量词 |
| `UNCHANGED` | 变量未变 |

### 常用操作符

```tla
(\* 逻辑操作符 \*)
/\  (\* 合取 ∧ \*)
/\  (\* 析取 ∨ \*)
~   (\* 否定 ¬ \*)
=>  (\* 蕴含 → \*)

(\* 集合操作符 \*)
\in  (\* 属于 \*)
\subseteq  (\* 子集 \*)
\cup  (\* 并集 \*)
\cap  (\* 交集 \*)

(\* 序数操作符 \*)
<  (\* 小于 \*)
<=  (\* 小于等于 \*)
+  (\* 加 \*)
-  (\* 减 \*)
```

## 实践示例：互斥

### 问题描述

设计一个互斥算法，确保：
1. **安全性**：最多一个进程进入临界区
2. **活跃性**：所有想进入临界区的进程最终都能进入

### TLA+ 规范

```tla
---- MODULE Mutex ----
EXTENDS Naturals

CONSTANTS N  (\* 进程数量 \*)

VARIABLES
    inCS,    (\* 正在临界区的进程集合 \*)
    want,    (\* 想进入临界区的进程集合 \*)
    pc       (\* 程序计数器 \*)

Proc == 1..N

Init ==
    /\ inCS = {}
    /\ want = {}
    /\ pc = [i \in Proc |-> "idle"]

(\* 进程i尝试进入临界区 \*)
TryEnter(i) ==
    /\ pc[i] = "idle"
    /\ pc' = [pc EXCEPT ![i] = "trying"]
    /\ want' = want \cup {i}
    /\ UNCHANGED inCS

(\* 进入临界区条件检查 \*)
CanEnter(i) ==
    /\ i \in want
    /\ inCS = {}
    /\ \A j \in want \ {i}: j < i

(\* 进入临界区 \*)
Enter(i) ==
    /\ pc[i] = "trying"
    /\ CanEnter(i)
    /\ pc' = [pc EXCEPT ![i] = "cs"]
    /\ inCS' = inCS \cup {i}
    /\ want' = want \ {i}

(\* 退出临界区 \*)
Exit(i) ==
    /\ pc[i] = "cs"
    /\ pc' = [pc EXCEPT ![i] = "idle"]
    /\ inCS' = inCS \ {i}
    /\ UNCHANGED want

(\* 下一状态动作 \*)
Next ==
    \E i \in Proc:
        \/ TryEnter(i)
        \/ Enter(i)
        \/ Exit(i)

(\* 安全性：最多一个进程在临界区 \*)
Safety ==
    Cardinality(inCS) <= 1

(\* 活跃性：如果想进入，最终会进入 \*)
Liveness ==
    \A i \in Proc:
        (pc[i] = "trying") ~> (pc[i] = "cs")

Spec == Init /\ [][Next]_vars
==== END MODULE ====
```

## 模型检验

### TLC 模型检验器

TLA+ Toolbox 包含 TLC 模型检验器，能够穷举检查所有状态：

```bash
# 下载 TLA+ Tools
wget https://github.com/tlaplus/tlaplus/releases/download/v1.7.1/tla2tools.jar

# 运行模型检验
java -cp tla2tools.jar tlc2.TLC Mutex.tla
```

### 常见检查

| 检查类型 | TLA+ 语法 | 含义 |
|----------|-----------|------|
| 不变性 | `INVARIANT Name` | 某性质在所有状态成立 |
| 性质 | `PROPERTY Name` | 某时序性质成立 |
| 死后检验 | `[]_vars` | 每个状态都有后继 |

### 反例

当模型检验发现违反时，会输出**反例轨迹**：

```
Error: Invariant Safety is violated.
State 1: inCS = {}, want = {1}, pc = [1|->"trying", 2|->"idle"]
State 2: inCS = {1}, want = {}, pc = [1|->"cs", 2|->"idle"]
State 3: inCS = {1,2}, want = {}, pc = [1|->"cs", 2|->"cs"]
                                                    ^^^^^^^^
                                          违反：两个进程同时在临界区
```

## PlusCal 算法语言

PlusCal 是 TLA+ 的算法伪代码语法，比纯 TLA+ 更易读：

```tla
---- MODULE Philosopher ----
EXTENDS Naturals

CONSTANT N

(*--algorithm DiningPhilosophers
variables
    forks = [i \in 1..N |-> "dirty"];
    
define
    Left(i) == (i - 1) \mod N + 1
    Right(i) == i
end define;

fair process (Phil \in 1..N)
variables
    left, right;
begin
    while TRUE do
        think:
            skip;
        acquire:
            either
                left := Left(self);
                right := Right(self);
            or
                left := Right(self);
                right := Left(self);
            end either;
        eat:
            skip;
        release:
            forks[left] := "dirty";
            forks[right] := "dirty";
    end while;
end process;
end algorithm; *)
==== END MODULE ====
```

PlusCal 代码会被翻译成等价的 TLA+ 规范。

## 分布式系统验证实例

### 两阶段提交（2PC）

```tla
---- MODULE TwoPhaseCommit ----
EXTENDS Naturals, FiniteSets

CONSTANT RM  (\* 资源管理器数量 \*)

VARIABLES
    rmState,    (\* 各RM的状态 \*)
    coordinator, (\* 协调者状态 \*)
    msgs        (\* 消息队列 \*)

Types == {"working", "prepared", "committed", "aborted"}
CoordStates == {"init", "commit", "abort"}

Message == [type : {"prepare"}, rm : RM]
        \cup [type : {"vote-abort"}, rm : RM]
        \cup [type : {"vote-commit"}, rm : RM]
        \cup [type : {"global-abort"}, rm : RM]
        \cup [type : {"global-commit"}, rm : RM]

Init ==
    /\ rmState \in [RM -> {"working", "prepared", "committed", "aborted"}]
    /\ coordinator = "init"
    /\ msgs = {}

(\* RM投票 \*)
RMPrepare(r) ==
    /\ rmState[r] = "working"
    /\ rmState' = [rmState EXCEPT ![r] = "prepared"]
    /\ msgs' = msgs \cup {[type |-> "vote-commit", rm |-> r]}

(\* 协调者收集投票 \*)
CollectVotes ==
    /\ coordinator = "init"
    /\ \A r \in RM: [type |-> "vote-commit", rm |-> r] \in msgs
    /\ coordinator' = "commit"
    /\ msgs' = msgs \cup {[type |-> "global-commit"]}

Abort ==
    /\ coordinator = "init"
    /\ \E r \in RM: [type |-> "vote-abort", rm |-> r] \in msgs
    /\ coordinator' = "abort"
    /\ msgs' = msgs \cup {[type |-> "global-abort"]}

Next ==
    \E r \in RM: RMPrepare(r)
    \/ CollectVotes
    \/ Abort

Spec == Init /\ [][Next]_vars
==== END MODULE ====
```

### 验证的性质

```tla
(\* 原子性：所有RM最终都提交或中止 \*)
Atomicity ==
    \A r1, r2 \in RM:
        (<> (rmState[r1] = "committed")) /\ (<> (rmState[r2] = "aborted")) => FALSE

(\* 一致性：如果任一RM提交，所有RM最终都提交 \*)
Consistency ==
    (\E r \in RM: <> (rmState[r] = "committed"))
        => (\A r \in RM: <> (rmState[r] = "committed"))
```

## TLA+ 生态系统

| 工具 | 用途 |
|------|------|
| **TLC** | 模型检验器 |
| **TLAPS** | 机器可检验的证明系统 |
| **Apalache** | 符号分析模型检验器 |
| **PlusCal** | 算法伪代码语言 |
| **SMT插件** | SMT求解器集成 |

## 何时使用形式化验证

### 适合的场景

1. **关键系统**：航空航天、医疗设备、金融系统
2. **并发/分布式系统**：难以通过测试发现所有竞态条件
3. **协议设计**：在实现前发现设计缺陷
4. **维护遗留系统**：理解复杂逻辑

### 成本考量

- **学习成本**：TLA+ 有一定学习曲线
- **规范成本**：需要详细描述系统行为
- **维护成本**：规范需要随系统更新

## 参考文献

[^1]: Leslie Lamport - Temporal Logic of Actions (TLA+)
[^2]: TLA+ Hyperbook - Leslie Lamport
[^3]: Practical TLA+ - Hillel Wayne
[^4]: TLA+ Tutorial - https://learn.tla.best/

---

*本页面内容基于TLA+官方文档和经典论文整理*
