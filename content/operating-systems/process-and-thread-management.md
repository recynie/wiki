---
title: 进程与线程管理
date: 2026-04-13
description: 进程线程概念、生命周期、同步机制与调度基础
tags:
  - operating-systems
  - process
  - thread
  - synchronization
draft: false
permalink:
---

# 进程与线程管理

进程和线程是操作系统资源分配和调度的基本单位，理解其原理是掌握操作系统核心的关键。

## 进程基础

### 进程定义

进程是程序的一次执行实例，是操作系统进行资源分配的基本单位。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();
    
    if (pid < 0) {
        perror("fork failed");
        exit(1);
    } else if (pid == 0) {
        // 子进程
        printf("Child process: PID=%d, PPID=%d\n", getpid(), getppid());
        sleep(1);
    } else {
        // 父进程
        printf("Parent process: PID=%d, Child PID=%d\n", getpid(), pid);
        wait(NULL);  // 等待子进程
    }
    
    return 0;
}
```

### 进程状态

```
                    ┌─────────────┐
                    │    创建态   │
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
            ┌──────│    就绪态   │◄──────┐
            │      └──────┬──────┘       │
            │             │              │
            │             ▼              │
            │      ┌─────────────┐        │
            │      │    运行态   │───────┤
            │      └──────┬──────┘        │
            │             │              │
            ▼             ▼              │
      ┌─────────────┐ ┌─────────────┐    │
      │    阻塞态   │ │    退出态   │    │
      └──────┬──────┘ └─────────────┘    │
             │                           │
             └───────────────────────────┘
```

### 进程控制块（PCB）

操作系统通过PCB保存进程状态信息：

```c
struct PCB {
    int pid;                    // 进程标识符
    enum state state;           // 进程状态
    int priority;               // 进程优先级
    unsigned long pc;            // 程序计数器
    unsigned long sp;            // 栈指针
    struct file *files[NR_OPEN]; // 文件描述符表
    struct mm_struct *mm;        // 内存管理信息
    // ... 其他资源信息
};
```

## 线程基础

### 线程定义

线程是进程内的执行流，是CPU调度的基本单位，同一进程内的线程共享地址空间。

```c
#define _GNU_SOURCE
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

void *thread_func(void *arg) {
    int *num = (int *)arg;
    printf("Thread %d: started\n", *num);
    sleep(1);
    printf("Thread %d: finished\n", *num);
    return NULL;
}

int main() {
    pthread_t threads[3];
    int ids[3] = {1, 2, 3};
    
    // 创建线程
    for (int i = 0; i < 3; i++) {
        pthread_create(&threads[i], NULL, thread_func, &ids[i]);
    }
    
    // 等待线程结束
    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }
    
    printf("All threads finished\n");
    return 0;
}
```

### 进程 vs 线程

| 维度 | 进程 | 线程 |
|------|------|------|
| 资源拥有 | 独立地址空间 | 共享地址空间 |
| 创建/销毁 | 开销大 | 开销小 |
| 通信 | IPC机制 | 直接读写共享内存 |
| 切换 | 上下文切换慢 | 上下文切换快 |
| 独立性 | 相互隔离 | 共享资源，需同步 |

## 线程同步

### 互斥锁（Mutex）

```c
#include <pthread.h>

typedef struct {
    int shared_data;
    pthread_mutex_t lock;
} SharedCounter;

void init_counter(SharedCounter *counter) {
    counter->shared_data = 0;
    pthread_mutex_init(&counter->lock, NULL);
}

void increment_counter(SharedCounter *counter) {
    pthread_mutex_lock(&counter->lock);
    counter->shared_data++;
    pthread_mutex_unlock(&counter->lock);
}

void destroy_counter(SharedCounter *counter) {
    pthread_mutex_destroy(&counter->lock);
}
```

### 信号量（Semaphore）

```c
#include <semaphore.h>

sem_t empty;  // 空槽位数
sem_t full;   // 满槽位数
pthread_mutex_t mutex;

void producer() {
    // 生产数据
    sem_wait(&empty);    // 消耗空槽位
    pthread_mutex_lock(&mutex);
    // 添加到缓冲区
    pthread_mutex_unlock(&mutex);
    sem_post(&full);     // 增加满槽位
}

void consumer() {
    sem_wait(&full);     // 消耗满槽位
    pthread_mutex_lock(&mutex);
    // 从缓冲区取出
    pthread_mutex_unlock(&mutex);
    sem_post(&empty);    // 增加空槽位
}
```

### 条件变量（Condition Variable）

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int ready = 0;

void *producer(void *arg) {
    // 生产数据
    pthread_mutex_lock(&mutex);
    ready = 1;
    pthread_cond_signal(&cond);  // 通知消费者
    pthread_mutex_unlock(&mutex);
}

void *consumer(void *arg) {
    pthread_mutex_lock(&mutex);
    while (!ready) {
        pthread_cond_wait(&cond, &mutex);  // 等待信号
    }
    // 消费数据
    pthread_mutex_unlock(&mutex);
}
```

### 读写锁

```c
pthread_rwlock_t rwlock;

void read_data() {
    pthread_rwlock_rdlock(&rwlock);
    // 读取共享数据
    pthread_rwlock_unlock(&rwlock);
}

void write_data() {
    pthread_rwlock_wrlock(&rwlock);
    // 写入共享数据
    pthread_rwlock_unlock(&rwlock);
}
```

## 进程间通信（IPC）

| 机制 | 特点 | 适用场景 |
|------|------|----------|
| 管道 | 半双工，父子进程 | 简单数据流 |
| 消息队列 | 消息结构化 | 多生产者-消费者 |
| 共享内存 | 高效，直接访问 | 高性能数据共享 |
| 信号量 | 计数器同步 | 进程同步 |
| Socket | 可跨机器通信 | 分布式系统 |

### 共享内存示例

```c
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/stat.h>

int shmid = shmget(IPC_PRIVATE, 4096, IPC_CREAT | 0666);
int *shared_data = (int *)shmat(shmid, NULL, 0);

// 使用共享内存
*shared_data = 42;

// 分离
shmdt(shared_data);

// 删除
shmctl(shmid, IPC_RMID, NULL);
```

## 死锁

### 死锁必要条件

1. **互斥条件**：资源一次只能被一个进程占用
2. **占有并等待**：进程保持已占有资源的同时请求其他资源
3. **不可抢占条件**：已分配资源不能被强制抢占
4. **循环等待**：形成进程-资源循环链

### 死锁处理策略

| 策略 | 方法 | 优缺点 |
|------|------|--------|
| 预防 | 破坏死锁条件 | 资源利用率低 |
| 避免 | 银行家算法 | 需要预知最大需求 |
| 检测 | 资源分配图 | 允许死锁发生，定期检测 |
| 解除 | 剥夺/终止进程 | 可能造成损失 |

### 银行家算法

```c
#include <stdbool.h>

#define NUM_PROCESSES 5
#define NUM_RESOURCES 3

int available[NUM_RESOURCES];           // 可用资源
int maximum[NUM_PROCESSES][NUM_RESOURCES];    // 最大需求
int allocation[NUM_PROCESSES][NUM_RESOURCES];  // 已分配
int need[NUM_PROCESSES][NUM_RESOURCES];        // 需求

bool is_safe_state() {
    int work[NUM_RESOURCES];
    bool finish[NUM_PROCESSES] = {false};
    
    // 复制可用资源
    for (int i = 0; i < NUM_RESOURCES; i++)
        work[i] = available[i];
    
    // 寻找安全序列
    for (int count = 0; count < NUM_PROCESSES; count++) {
        bool found = false;
        for (int p = 0; p < NUM_PROCESSES; p++) {
            if (!finish[p]) {
                bool can_allocate = true;
                for (int r = 0; r < NUM_RESOURCES; r++) {
                    if (need[p][r] > work[r]) {
                        can_allocate = false;
                        break;
                    }
                }
                if (can_allocate) {
                    for (int r = 0; r < NUM_RESOURCES; r++)
                        work[r] += allocation[p][r];
                    finish[p] = true;
                    found = true;
                }
            }
        }
        if (!found) return false;
    }
    return true;
}
```

## 参考

[^1]: Silberschatz et al., Operating System Concepts
[^2]: Tanenbaum, Modern Operating Systems

