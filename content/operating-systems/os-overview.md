---
title: 操作系统概述
date: 2026-04-06
description: 操作系统（OS）是管理计算机硬件与软件资源的系统软件
tags:
  - operating-system
  - fundamentals
draft: false
permalink:
---

# 操作系统概述

操作系统（Operating System, OS）是管理计算机硬件与软件资源的**系统软件**，是用户与计算机硬件之间的接口。

## 操作系统的功能

1. **进程管理**：创建、调度、终止进程
2. **内存管理**：分配和回收内存空间
3. **文件系统管理**：组织和管理文件
4. **设备管理**：管理输入输出设备
5. **网络管理**：处理网络通信

## 进程与线程

### 进程

进程（Process）是程序的一次执行，是操作系统**资源分配**的基本单位。

```cpp
// Linux下创建进程
#include <unistd.h>
#include <sys/types.h>

pid_t pid = fork();  // 创建子进程
if (pid == 0) {
    // 子进程
    execlp("/bin/ls", "ls", NULL);
} else if (pid > 0) {
    // 父进程
    wait(NULL);
}
```

### 线程

线程（Thread）是CPU**调度**的基本单位，同一进程内的线程共享进程资源。

```cpp
#include <pthread.h>

void* thread_func(void* arg) {
    int* n = (int*)arg;
    // 线程工作
    return NULL;
}

pthread_t tid;
pthread_create(&tid, NULL, thread_func, &arg);
pthread_join(tid, NULL);
```

### 进程 vs 线程

| 特征 | 进程 | 线程 |
|------|------|------|
| 资源分配 | 独立地址空间 | 共享进程资源 |
| 创建/销毁 | 慢 | 快 |
| 通信 | IPC复杂 | 直接读写共享数据 |
| 切换 | 慢 | 快 |

## 进程调度算法

### 常见调度算法

| 算法 | 特点 | 适用场景 |
|------|------|---------|
| FCFS | 简单公平 | 教学理解 |
| SJF/SPF | 最短作业优先 | 优化周转时间 |
| RR | 时间片轮转 | 分时系统 |
| 优先级调度 | 优先级高者先执行 | 实时系统 |
| 多级反馈队列 | 综合多种优点 | 通用系统 |

### C++实现简单的调度模拟

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Process {
    int pid;        // 进程ID
    int burst;      // 执行时间
    int priority;   // 优先级
    int arrival;    // 到达时间
    int waiting;    // 等待时间
    int turnaround; // 周转时间
};

void fcfsSchedule(vector<Process>& processes) {
    sort(processes.begin(), processes.end(), 
         [](const Process& a, const Process& b) {
             return a.arrival < b.arrival;
         });
    
    int time = 0;
    for (auto& p : processes) {
        if (time < p.arrival) time = p.arrival;
        p.waiting = time - p.arrival;
        time += p.burst;
        p.turnaround = p.waiting + p.burst;
    }
}
```

## 内存管理

### 内存分配方式

1. **连续分配**：固定分区、可变分区
2. **离散分配**：分页、分段、段页式

### 分页系统

将内存划分为固定大小的**页框**，进程的逻辑地址空间划分为**页面**。

```cpp
// 简化的分页地址转换
struct PageTableEntry {
    int frame;      // 页框号
    bool valid;    // 是否有效
};

int translateAddress(int logical_addr, int page_size, 
                     const vector<PageTableEntry>& page_table) {
    int page_num = logical_addr / page_size;
    int offset = logical_addr % page_size;
    
    if (!page_table[page_num].valid) {
        throw runtime_error("Page fault");
    }
    
    return page_table[page_num].frame * page_size + offset;
}
```

### 虚拟内存

虚拟内存让程序访问比物理内存更大的地址空间，主要技术包括：

- **请求分页**：按需调入页面
- **页面置换算法**：LRU、FIFO、Clock算法

## 死锁

### 死锁的四个必要条件

1. **互斥**：资源一次只能被一个进程使用
2. **占有并等待**：进程保持资源的同时请求其他资源
3. **非抢占**：资源不能被强制夺取
4. **循环等待**：形成进程资源的循环链

### 银行家算法

银行家算法用于避免死锁：

```cpp
#include <bits/stdc++.h>
using namespace std;

struct BankerAlgorithm {
    int n, m;  // n进程数, m资源种类数
    vector<vector<int>> max_need;      // 最大需求
    vector<vector<int>> allocation;     // 已分配
    vector<vector<int>> need;          // 仍需
    vector<int> available;              // 可用资源
    
    bool isSafe() {
        vector<int> work = available;
        vector<bool> finish(n, false);
        
        for (int iter = 0; iter < n; ++iter) {
            bool found = false;
            for (int i = 0; i < n; ++i) {
                if (!finish[i]) {
                    bool canProceed = true;
                    for (int j = 0; j < m; ++j) {
                        if (need[i][j] > work[j]) {
                            canProceed = false;
                            break;
                        }
                    }
                    if (canProceed) {
                        for (int j = 0; j < m; ++j) {
                            work[j] += allocation[i][j];
                        }
                        finish[i] = true;
                        found = true;
                    }
                }
            }
            if (!found) return false;
        }
        return true;
    }
};
```

## 文件系统

### 文件系统结构

```
boot block → super block → inode bitmap → data bitmap → inode table → data blocks
```

### inode结构

每个文件对应一个inode，存储元数据：

```cpp
struct inode {
    int inode_no;           // inode编号
    mode_t mode;            // 文件类型和权限
    uid_t uid;              // 文件所有者ID
    off_t size;             // 文件大小
    time_t atime;           // 访问时间
    time_t mtime;           // 修改时间
    time_t ctime;           // 状态改变时间
    int blocks;             // 占用块数
    int block_ptr[15];      // 直接/间接块指针
};
```

## 参考

- [操作系统 - 维基百科](https://zh.wikipedia.org/wiki/操作系统)
- [Operating System Concepts - Silberschatz]
