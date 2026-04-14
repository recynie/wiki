---
title: 量子计算基础
date: 2026-04-13
description: 量子计算（Quantum Computing）基础概念详解
tags:
  - quantum-computing
  - emerging-technology
draft: false
permalink:
---

## 概述

量子计算是一种利用量子力学原理进行信息处理的计算模式。与经典计算机不同，量子计算机使用**量子比特**（Qubit）作为基本信息单元，能够处于多个状态的叠加，从而在某些问题上获得指数级加速。

## 量子比特（Qubit）

量子比特是量子计算机的基本信息单元，与经典比特不同，量子比特可以处于**量子叠加态**：

$$
|\psi\rangle = \alpha|0\rangle + \beta|1\rangle
$$

其中：
- $|0\rangle$ 和 $|1\rangle$ 是标准基态
- $\alpha$ 和 $\beta$ 是复数概率幅，满足 $|\alpha|^2 + |\beta|^2 = 1$
- 测量时，以 $|\alpha|^2$ 的概率坍缩到 $|0\rangle$，以 $|\beta|^2$ 的概率坍缩到 $|1\rangle$

当 $\alpha$ 或 $\beta$ 为零时，量子比特退化为经典比特。

### 状态空间

每增加一个量子比特，状态空间维度翻倍——$n$ 个量子比特的状态空间是 $2^n$ 维的。这意味着：
- 3个量子比特 = 8维状态空间
- 50个量子比特 = 约 $10^{15}$ 维状态空间
- 300个量子比特 = 状态空间比可观测宇宙中的原子数还多

## 量子门（Quantum Gates）

量子门通过矩阵乘法作用于量子态向量，实现状态变换。

### 单量子比特门

| 门 | 符号 | 作用 |
|----|------|------|
| Hadamard | H | 产生叠加态：$|0\rangle \rightarrow |+\rangle$ |
| Pauli-X | X | 比特翻转（NOT门） |
| Pauli-Y | Y | 泡利Y门 |
| Pauli-Z | Z | 泡利Z门（相位翻转） |

**Hadamard门（H）**是产生叠加态的核心门：
$$
H = \frac{1}{\sqrt{2}}\begin{pmatrix} 1 & 1 \\ 1 & -1 \end{pmatrix}
$$

### 多量子比特门

**CNOT（受控非门）**是实现量子纠缠的关键门：
- 当控制量子比特为 $|1\rangle$ 时，对目标量子比特施加NOT门
- 当控制量子比特为 $|0\rangle$ 时，目标量子比特不变

**通用性**：任意量子计算都可以分解为单量子比特门 + CNOT门的序列。

### 量子电路模型

量子计算通常用量子电路表示，横向为时间轴，竖向为量子比特：

```
q0 ──[H]──[X]──●───▶ 测量
q1 ───────[H]──╯
```

## 量子纠缠（Quantum Entanglement）

纠缠是量子力学中的核心现象，指两个或多个量子比特之间存在非经典的关联。

### 贝尔态（Bell States）

最常见的纠缠态是贝尔态：
$$
|\Phi^+\rangle = \frac{|00\rangle + |11\rangle}{\sqrt{2}}
$$

纠缠态的特点：测量一个量子比特会立即影响另一个量子比特的状态，无论它们相距多远。

### 量子并行性

通过叠加态，量子计算机可同时对多个输入值求值函数：
$$
\sum_{x}|x\rangle|f(x)\rangle
$$

这使得量子算法可以利用量子并行性进行加速搜索。但仅有并行性不足以提速，因为测量只能得到一个值，需要配合**干涉**（wave interference）等技术才能放大期望结果。

## 重要量子算法

### Shor算法（1994）

整数分解算法，具有**超多项式加速**。能够破解RSA等公钥加密体系：

- 经典算法：$O(e^{(\log N)^{1/3})}$ 
- Shor算法：$O((\log N)^3)$

### Grover算法（1996）

无结构搜索算法，提供**平方级加速**：

- 经典搜索：$O(N)$
- Grover算法：$O(\sqrt{N})$

可用于加速任何需要遍历搜索的问题，如SAT求解、密码分析等。

### HHL算法（2009）

求解线性方程组的量子算法，在特定条件下有指数级加速。

## 量子计算的应用领域

### 密码学

- **威胁**：Shor算法威胁RSA、Diffie-Hellman等公钥加密体系
- **防御**：后量子密码学（Post-Quantum Cryptography）正在发展抗量子攻击的新算法，如基于格的密码学（Lattice-based Cryptography）

### 量子模拟

- 模拟量子化学和凝聚态物理系统
- 氮固定（Haber-Bosch过程）的模拟
- 新药物研发

### 量子通信

- 量子密钥分发（QKD）利用纠缠实现可证明安全的密钥交换
- 量子隐形传态（Quantum Teleportation）

### 量子机器学习

- 探索量子算法加速机器学习任务
- 目前大多尚处研究阶段

## 当前挑战

### 量子退相干（Quantum Decoherence）

量子比特与环境相互作用导致信息丢失，是构建实用量子计算机的最大障碍。退相干时间通常在微秒到毫秒级。

### 量子纠错（Quantum Error Correction）

- 阈值定理表明可通过增加冗余量子比特来抑制错误
- 代价高昂：分解一个2048位整数可能需要约300万个物理量子比特

### 量子优越性里程碑

- 2019年Google Sycamore处理器宣称实现量子优越性（54量子比特）
- 2023年中国"九章"光量子计算机
- 这些均为科学里程碑，尚未转化为实际应用价值

## 量子计算 vs 经典计算

| 方面 | 经典计算 | 量子计算 |
|------|---------|---------|
| 基本单元 | 比特（0或1） | 量子比特（叠加态） |
| 状态空间 | $N$ | $2^N$ |
| 优势问题 | 通用计算 | 分解、搜索、量子模拟 |
| 当前状态 | 成熟 | NISQ（含噪中等规模量子）时代 |
| 实际应用 | 丰富 | 有限，主要在研究阶段 |

## 参考资料

[^1]: 本段参考[Wikipedia: Quantum Computing](https://en.wikipedia.org/wiki/Quantum_computing)

[^2]: Nielsen & Chuang, *Quantum Computation and Quantum Information*

[^3]: Shor, P.W. (1994), *Algorithms for quantum computation: discrete logarithms and factoring*
