---
title: 计算机体系结构基础
date: 2026-04-08
description: 计算机体系结构核心概念：CPU组成、内存层次、指令系统、总线与I/O
tags:
  - computer-architecture
  - cpu
  - memory-hierarchy
  - instruction
draft: false
permalink:
---

## 概述

计算机体系结构（Computer Architecture）是研究计算机硬件组成、各部件功能及其相互关系的学科。[^1]

### 冯·诺依曼架构

```
        ┌─────────────┐
        │    CPU      │
        │  ┌───────┐  │
        │  │  ALU  │  │
        │  └───────┘  │
        │  ┌───────┐  │
        │  │寄存器 │  │
        │  └───────┘  │
        │  ┌───────┐  │
        │  │  CU   │  │
        │  └───────┘  │
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │    内存      │
        │ (RAM/ROM)   │
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │  输入/输出   │
        │   (I/O)     │
        └─────────────┘
```

特点：**存储程序原理** - 程序和数据都存储在内存中。

---

## CPU组成

### 算术逻辑单元（ALU）

ALU是CPU的核心执行部件，负责算术运算和逻辑运算：

| 运算类型 | 示例 |
|---------|------|
| 算术运算 | 加、减、乘、除 |
| 逻辑运算 | AND、OR、NOT、XOR |
| 移位运算 | 左移、右移、算术移位 |

### 寄存器

寄存器是CPU内部的高速存储单元：

| 类型 | 用途 | 示例 |
|------|------|------|
| 通用寄存器 | 存储操作数 | EAX, EBX (x86) |
| 程序计数器(PC) | 指向下一条指令 | IP (x86) |
| 指令寄存器(IR) | 存储当前指令 | - |
| 状态寄存器 | 存储运算结果状态 | FLAGS (x86) |
| 栈指针(SP) | 指向栈顶 | ESP (x86) |
| 基址指针(BP) | 栈帧基址 | EBP (x86) |

### 控制单元（CU）

控制单元负责指令的取指、解码和执行控制：

```
取指(Fetch) → 解码(Decode) → 执行(Execute) → 存储(Store)
```

### 指令周期

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ 取指令   │ → │ 译码    │ → │ 执行    │ → │ 写回    │
│ (Fetch) │    │(Decode) │   │(Execute)│   │(Store)  │
└─────────┘    └─────────┘    └─────────┘    └─────────┘
```

---

## 指令系统

### 指令格式

```
┌─────────┬─────────┬─────────┐
│ 操作码  │  源操作数 │ 目标操作数│
│ (Opcode)│  (Src)   │  (Dst)   │
└─────────┴─────────┴─────────┘
```

### 寻址模式

| 模式 | 示例 | 说明 |
|------|------|------|
| 立即数寻址 | `MOV EAX, 5` | 操作数在指令中 |
| 寄存器寻址 | `MOV EAX, EBX` | 操作数在寄存器中 |
| 直接寻址 | `MOV EAX, [1000h]` | 操作数在内存地址中 |
| 间接寻址 | `MOV EAX, [EBX]` | EA在EBX指定的内存中 |
| 基址寻址 | `MOV EAX, [EBX+4]` | EA=EBX+偏移量 |
| 变址寻址 | `MOV EAX, [EBX+ESI]` | EA=EBX+ESI |

### 常见指令类型

```cpp
// 数据传送指令
MOV EAX, EBX        // 寄存器到寄存器
MOV [1000h], EAX    // 寄存器到内存
MOV EAX, [1000h]    // 内存到寄存器

// 算术指令
ADD EAX, EBX        // 加法
SUB EAX, 1          // 减法
IMUL EAX, EBX       // 有符号乘法
IDIV EBX            // 有符号除法

// 逻辑指令
AND EAX, EBX        // 与
OR EAX, 0Fh         // 或
XOR EAX, EAX        // 异或（常用于寄存器清零）
NOT EAX             // 非

// 控制转移
JMP label           // 无条件跳转
JE label            // 相等则跳转
CALL func           // 函数调用
RET                 // 返回
```

---

## 内存层次结构

### 层次划分

```
┌────────────────────────────────────────┐
│           CPU寄存器                     │ ← 最快，<1ns
│         (几 Words)                      │
├────────────────────────────────────────┤
│           L1 Cache                     │ ← 1-2ns, ~64KB
├────────────────────────────────────────┤
│           L2 Cache                     │ ← 3-10ns, ~256KB
├────────────────────────────────────────┤
│           L3 Cache                     │ ← 10-20ns, ~数MB
├────────────────────────────────────────┤
│           主存 (RAM)                   │ ← 50-100ns, GB级
├────────────────────────────────────────┤
│           磁盘 (SSD/HDD)              │ ← ms级, TB级
└────────────────────────────────────────┘
                    ↑
              容量增大，速度减慢
```

### 缓存（Cache）

#### 缓存行

Cache与内存之间以**缓存行（Cache Line）**为单位传输，典型大小为64字节。

```
内存块 (64B) → 缓存行 → CPU
```

#### 映射方式

| 方式 | 原理 | 特点 |
|------|------|------|
| 直接映射 | 每个块只能放在一个特定位置 | 简单，冲突多 |
| 组相联 | 每个块放在N个位置的集合中 | 常用(N=2,4,8) |
| 全相联 | 每个块可以放在任意位置 | 复杂，适用于小Cache |

#### 缓存策略

```cpp
struct CacheLine {
    bool valid;
    int tag;
    char data[64];
};

class DirectMappedCache {
    static const int CACHE_SIZE = 1024;  // 1KB
    static const int LINE_SIZE = 64;      // 64B
    static const int NUM_LINES = CACHE_SIZE / LINE_SIZE;
    
    CacheLine lines[NUM_LINES];
    
    int getLineIndex(int addr) {
        return (addr / LINE_SIZE) % NUM_LINES;
    }
    
    int getTag(int addr) {
        return addr / (LINE_SIZE * NUM_LINES);
    }
};
```

### 虚拟内存

虚拟内存通过页表实现地址转换：

```
虚拟地址 → MMU(内存管理单元) → 物理地址
              ↓
         页表查找
              ↓
         TLB(快表)缓存
```

#### 页表结构

```cpp
struct PageTableEntry {
    bool present;       // 页面是否在内存中
    int frame;          // 物理页框号
    bool dirty;         // 是否被修改过
    int accessed;       // 访问位
};
```

#### 页面置换算法

- **LRU（最近最少使用）**：淘汰最长时间未使用的页面
- **FIFO（先进先出）**：淘汰最早进入的页面
- **Clock算法**：环形检查，第二次 chance 淘汰

---

## 总线与I/O

### 总线类型

| 总线类型 | 说明 | 示例 |
|---------|------|------|
| 数据总线 | 传输数据 | 64位数据总线 |
| 地址总线 | 传输地址 | 32位地址总线 |
| 控制总线 | 传输控制信号 | 时钟、读写控制 |

### I/O控制方式

| 方式 | CPU介入 | 特点 |
|------|--------|------|
| 程序查询 | 高 | 简单，效率低 |
| 中断驱动 | 中 | I/O完成时通知CPU |
| DMA(直接存储器访问) | 低 | 大数据量传输 |

### DMA控制器

```cpp
struct DMAController {
    int src_addr;      // 源地址
    int dst_addr;      // 目标地址
    int count;          // 传输字节数
    int mode;           // 传输模式
    
    void start() {
        // 开始DMA传输
        // 传输过程中CPU可以执行其他任务
    }
};
```

---

## 流水线技术

### 指令流水线

将指令执行分为多个阶段并行处理：

```
时间 →
指令: [IF] [ID] [EX] [MEM] [WB]
      [IF] [ID] [EX] [MEM] [WB]  (并行)
          [IF] [ID] [EX] [MEM] [WB]
```

### 流水线冒险

| 类型 | 原因 | 解决方案 |
|------|------|---------|
| 结构冒险 | 硬件资源冲突 | 增加硬件资源 |
| 数据冒险 | 数据依赖 | 前递、停顿 |
| 控制冒险 | 分支指令 | 分支预测 |

### 分支预测

```cpp
class BranchPredictor {
    // 2位饱和计数器
    int state = 0;  // 0-3: strongly not taken → strongly taken
    
public:
    bool predict() {
        return state >= 2;
    }
    
    void update(bool actual) {
        if (actual && state < 3) state++;
        else if (!actual && state > 0) state--;
    }
};
```

---

## RISC与CISC

### 对比

| 特性 | RISC | CISC |
|------|------|------|
| 指令数量 | 少 | 多 |
| 指令长度 | 固定 | 变长 |
| 通用寄存器 | 多(>32) | 少 |
| 功耗 | 低 | 高 |
| 代表架构 | ARM, RISC-V | x86 |

### ARM架构特点

- 加载-存储模式：算术运算只能对寄存器操作
- 指令流水线：3级/5级/8级流水线
- 低功耗设计：适合移动设备

---

## 参考资料

[^1]: 本词条参考[计算机体系结构 - 维基百科](https://zh.wikipedia.org/wiki/计算机体系结构)

[^2]: 《计算机组成与设计：硬件/软件接口》- David A. Patterson, John L. Hennessy

---

## 相关主题

- [[../operating-systems/os-overview|操作系统]] - 硬件资源管理
- [[../algorithms/bit-manipulation|位运算]] - 低层编程技巧
