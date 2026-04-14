---
title: OpenMP基础
date: 2026-04-11
description: OpenMP并行编程入门——共享内存多线程编程模型与指令用法
tags:
  - openmp
  - parallel-computing
  - multi-threading
draft: false
permalink:
---

## 1. 概述

**OpenMP**（Open Multi-Processing）是一个跨平台的共享内存多线程编程API，支持C、C++和Fortran。它基于**fork-join并行模型**：主线程（Master Thread）创建多个从线程并行执行任务，然后汇合（join）回主线程。

OpenMP的核心优势在于**简洁性**——只需在串行代码中添加编译指令（Pragmas），即可实现并行化。

## 2. Fork-Join模型

```
        主线程
           │
    ┌──────┴──────┐
    │  Fork       │
    ▼             ▼
  线程1        线程2        线程N
    │             │             │
    └──────┬──────┘
           │
        Join
           │
           ▼
        主线程
```

OpenMP程序开始时只有一个主线程执行。当遇到并行区域时，主线程Fork出多个线程并行执行任务。任务完成后，所有线程汇合回主线程，继续串行执行。

## 3. 编译与运行

### 3.1 编译选项

```bash
# GCC
g++ -fopenmp -o program program.cpp

# Clang
clang++ -fopenmp -o program program.cpp

# Intel
icpc -qopenmp -o program program.cpp
```

### 3.2 运行设置

```bash
# 设置线程数
export OMP_NUM_THREADS=8

# 运行程序
./program
```

## 4. 基本指令

### 4.1 parallel指令

`parallel`指令创建一个并行区域：

```cpp
#include <omp.h>
#include <stdio.h>

int main() {
    #pragma omp parallel
    {
        int thread_id = omp_get_thread_num();
        int num_threads = omp_get_num_threads();
        
        printf("Hello from thread %d of %d\n", 
               thread_id, num_threads);
    }
    return 0;
}
```

输出示例（4线程）：

```
Hello from thread 0 of 4
Hello from thread 2 of 4
Hello from thread 1 of 4
Hello from thread 3 of 4
```

### 4.2 条件编译

确保代码在不支持OpenMP的系统上也能编译：

```cpp
#ifdef _OPENMP
    #include <omp.h>
#else
    #define omp_get_thread_num() 0
    #define omp_get_num_threads() 1
#endif
```

## 5. Work-Sharing Constructs

Work-Sharing将任务分配给线程组，是OpenMP中最常用的并行化手段。

### 5.1 for指令

`for`指令将循环迭代分配给线程：

```cpp
#include <omp.h>
#include <stdio.h>

int main() {
    const int N = 100;
    int a[N], b[N], c[N];
    
    // 初始化
    for (int i = 0; i < N; i++) {
        a[i] = i;
        b[i] = i * 2;
    }
    
    // 并行向量加法
    #pragma omp parallel for
    for (int i = 0; i < N; i++) {
        c[i] = a[i] + b[i];
    }
    
    printf("c[50] = %d\n", c[50]);  // 输出: 100
    return 0;
}
```

### 5.2 sections与section指令

`sections`将不同的代码块分配给不同线程：

```cpp
#pragma omp parallel
{
    #pragma omp sections
    {
        #pragma omp section
        {
            printf("Section 1 executed by thread %d\n", 
                   omp_get_thread_num());
        }
        
        #pragma omp section
        {
            printf("Section 2 executed by thread %d\n", 
                   omp_get_thread_num());
        }
        
        #pragma omp section
        {
            printf("Section 3 executed by thread %d\n", 
                   omp_get_thread_num());
        }
    }
}
```

### 5.3 single指令

`single`指定某代码块只由一个线程执行：

```cpp
#pragma omp parallel
{
    printf("All threads execute this.\n");
    
    #pragma omp single
    {
        printf("Only one thread reads input.\n");
    }
    
    printf("All threads execute this too.\n");
}
```

## 6. 数据作用域属性

OpenMP通过子句（Clause）管理变量的共享属性：

| 子句 | 含义 |
|------|------|
| `shared` | 变量在所有线程间共享 |
| `private` | 每个线程有变量的私有副本 |
| `firstprivate` | 私有副本，初始化为原值 |
| `lastprivate` | 私有副本，最后迭代的值写回原变量 |
| `default` | 指定默认共享属性 |

### 6.1 shared与private

```cpp
int x = 10;  // 共享变量

#pragma omp parallel for private(x)
for (int i = 0; i < N; i++) {
    x = i;  // 每个线程有自己的x副本
    // ...
}
// x的值未定义（私有副本不写回）
```

### 6.2 reduction子句

`reduction`对每个线程的私有变量进行归约：

```cpp
int sum = 0;

#pragma omp parallel for reduction(+:sum)
for (int i = 1; i <= 100; i++) {
    sum += i;  // 每个线程有自己的sum副本，最后累加
}

printf("sum = %d\n", sum);  // 输出: 5050
```

支持的归约操作符：`+`、`-`、`*`、`min`、`max`、`&`、`|`、`^`、`&&`、`||`

### 6.3 firstprivate与lastprivate

```cpp
int result = 100;

#pragma omp parallel for firstprivate(result) lastprivate(result)
for (int i = 0; i < N; i++) {
    result += i;  // 初始化为100，每个线程独立累加
}

printf("result = %d\n", result);  // lastprivate：最后一个迭代的值
```

## 7. 同步构造

### 7.1 critical指令

`critical`创建临界区，同一时刻只允许一个线程执行：

```cpp
int counter = 0;

#pragma omp parallel for
for (int i = 0; i < 1000; i++) {
    #pragma omp critical
    {
        counter++;  // 原子操作，避免竞态条件
    }
}
```

### 7.2 atomic指令

`atomic`比`critical`更高效，针对单一内存操作：

```cpp
#pragma omp parallel for
for (int i = 0; i < 1000; i++) {
    #pragma omp atomic
    counter++;
}
```

### 7.3 barrier指令

`barrier`同步点，等待所有线程到达后继续：

```cpp
#pragma omp parallel
{
    // 第一阶段：并行计算
    compute_phase1();
    
    #pragma omp barrier  // 等待所有线程完成
    
    // 第二阶段：依赖第一阶段结果
    compute_phase2();
}
```

### 7.4 master指令

`master`指定代码只由主线程执行：

```cpp
#pragma omp parallel
{
    // 所有线程并行执行
    work1();
    
    #pragma omp master
    {
        // 只有主线程执行
        print_results();
    }
    
    // 所有线程继续并行执行
    work2();
}
```

## 8. 调度策略

OpenMP提供多种循环调度策略，通过`schedule`子句指定：

### 8.1 static调度

将迭代均匀分配给线程，预先确定：

```cpp
#pragma omp parallel for schedule(static)
// 线程0: 0-24, 线程1: 25-49, ...
```

### 8.2 dynamic调度

运行时动态分配迭代，线程完成一个chunk后请求下一个：

```cpp
#pragma omp parallel for schedule(dynamic, 10)
for (int i = 0; i < 100; i++) {
    process(i);  // 负载不均衡时更高效
}
```

### 8.3 guided调度

初始大chunk，逐渐减小chunk大小：

```cpp
#pragma omp parallel for schedule(guided, 8)
for (int i = 0; i < 1000; i++) {
    process(i);
}
```

### 8.4 runtime调度

由环境变量决定调度策略：

```bash
export OMP_SCHEDULE="dynamic,5"
```

## 9. 嵌套并行

OpenMP支持嵌套并行：

```cpp
omp_set_nested(1);  // 启用嵌套

#pragma omp parallel
{
    #pragma omp parallel for
    for (int i = 0; i < 10; i++) {
        // 内层并行
    }
}
```

## 10. 环境变量

| 环境变量 | 作用 |
|----------|------|
| `OMP_NUM_THREADS` | 设置线程数 |
| `OMP_SCHEDULE` | 设置默认调度策略 |
| `OMP_DYNAMIC` | 启用/禁用动态线程数调整 |
| `OMP_NESTED` | 启用/禁用嵌套并行 |

## 11. 完整示例：矩阵乘法

```cpp
#include <omp.h>
#include <stdio.h>
#define N 512
#define BLOCK_SIZE 64

void matrixMultiply(double A[N][N], double B[N][N], double C[N][N]) {
    #pragma omp parallel for collapse(2) schedule(dynamic)
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < N; j++) {
            double sum = 0.0;
            for (int k = 0; k < N; k++) {
                sum += A[i][k] * B[k][j];
            }
            C[i][j] = sum;
        }
    }
}

int main() {
    double A[N][N], B[N][N], C[N][N];
    
    // 初始化矩阵
    #pragma omp parallel for collapse(2)
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < N; j++) {
            A[i][j] = i + j;
            B[i][j] = i - j;
            C[i][j] = 0.0;
        }
    }
    
    double start = omp_get_wtime();
    matrixMultiply(A, B, C);
    double end = omp_get_wtime();
    
    printf("Matrix multiplication took %f seconds\n", end - start);
    printf("C[100][100] = %f\n", C[100][100]);
    
    return 0;
}
```

编译运行：

```bash
g++ -fopenmp -O2 -o matmul matmul.cpp
./matmul
```

## 12. 常见问题

### 12.1 竞态条件

多个线程同时访问共享变量且至少一个写操作时，需要同步：

```cpp
// 错误：竞态条件
#pragma omp parallel for
for (int i = 0; i < N; i++) {
    result += compute(i);  // 多个线程同时写result
}

// 正确：使用reduction
#pragma omp parallel for reduction(+:result)
for (int i = 0; i < N; i++) {
    result += compute(i);
}
```

### 12.2 数据依赖

循环迭代间存在依赖时不能直接并行化：

```cpp
// 错误：存在数据依赖
for (int i = 1; i < N; i++) {
    a[i] = a[i-1] + b[i];  // 依赖前一个迭代
}

// 正确：去除依赖或使用扫描算法
```

### 12.3 死锁

嵌套锁使用不当可能导致死锁：

```cpp
#pragma omp parallel
{
    #pragma omp critical (lock1)
    {
        #pragma omp critical (lock2)  // 可能死锁
        { /* ... */ }
    }
}
```

## 13. 参考资料

[^1]: [OpenMP Official Website](https://www.openmp.org/)

[^2]: [OpenMP API Specification](https://www.openmp.org/specifications/)

[^3]: [LLNL OpenMP Tutorial](https://hpc-tutorials.llnl.gov/openmp/)

---

## 相关主题

- [[../parallel-programming/concurrent-programming-basics|并发编程基础]] — 线程同步与锁
- [[../parallel-programming/cuda|CUDA基础]] — GPU并行计算
- [[../parallel-programming/actor-model|Actor模型]] — 消息传递并发模式
