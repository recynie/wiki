---
title: 并发编程基础
date: 2026-04-09
description: 并发编程核心概念：线程、进程、同步原语、竞态条件、死锁
tags:
  - concurrency
  - multithreading
  - parallel-programming
draft: false
permalink:
---

# 并发编程基础

并发编程是现代软件开发的核心技能之一。本节介绍并发编程的基础概念，帮助读者理解多线程编程的核心原理。

## 1. 并发与并行的区别

| 概念 | 定义 | 实现方式 | CPU依赖 |
|------|------|----------|---------|
| **并发（Concurrency）** | 交替执行多个任务，通过时间片轮转实现 | 单核或多核CPU均可 | 不依赖多核 |
| **并行（Parallelism）** | 同时执行多个任务 | 必须多核CPU | 依赖多核 |

```cpp
// 并发：单核CPU交替执行
// 线程A: task1 -> task2 -> task3
// 线程B: ---------> task1 -> task2

// 并行：多核CPU同时执行
// 核心1: task1 -> task2
// 核心2: task1 -> task2
```

**核心要点**：
- 并发解决**利用率**问题（IO阻塞时切换任务）
- 并行解决**速度**问题（利用多核同时计算）

---

## 2. 进程与线程

### 2.1 进程（Process）

进程是操作系统分配资源的基本单位，每个进程有独立的地址空间。

```cpp
// 进程的特点：
// - 独立虚拟地址空间
// - 代码段、数据段、堆、栈相互隔离
// - 进程间通信需要IPC机制
```

### 2.2 线程（Thread）

线程是CPU调度的基本单位，同一进程的线程共享地址空间。

```cpp
// 线程的特点：
// - 共享进程的地址空间（代码段、堆、全局变量）
// - 每个线程有独立的栈和寄存器
// - 线程间通信可直接读写共享数据（需同步）
```

### 2.3 进程与线程的选择

| 场景 | 推荐 |
|------|------|
| CPU密集型任务 | 多进程（避免GIL限制） |
| IO密集型任务 | 多线程（共享资源方便） |
| 需要资源隔离 | 多进程 |
| 需要大量共享数据 | 多线程 |

---

## 3. 竞态条件

### 3.1 定义

**竞态条件（Race Condition）**：多个线程对共享数据的访问顺序会影响最终结果。

```cpp
// 经典竞态条件示例
int count = 0;

void increment() {
    // 看似是原子操作，实则由三步组成：
    // 1. 读取count的值
    // 2. 将count加1
    // 3. 写回count
    count++;  
}

// 线程A和线程B可能执行顺序：
// A: 读取count=0
// B: 读取count=0  <-- A还未写回
// A: count=1, 写回
// B: count=1, 写回  <-- 覆盖了A的结果
// 最终count=1，而非预期的2
```

### 3.2 竞态条件的成因

1. **共享资源**：多个线程访问同一份数据
2. **非原子操作**：复合操作不是原子性的
3. **执行顺序不确定**：取决于线程调度

### 3.3 解决竞态条件的方法

| 方法 | 原理 | 适用场景 |
|------|------|----------|
| **互斥锁** | 保证同一时刻只有一个线程访问 | 写入操作 |
| **原子变量** | 硬件支持的原子指令 | 简单计数器 |
| **无锁数据结构** | CAS操作实现线程安全 | 高性能场景 |

---

## 4. 同步原语

### 4.1 互斥锁（Mutex）

```cpp
#include <pthread.h>

pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;

void safe_increment() {
    pthread_mutex_lock(&mtx);
    count++;
    pthread_mutex_unlock(&mtx);
}
```

**互斥锁的两种类型**：
- **忙等待锁（自旋锁）**：循环检测锁是否可用，适合临界区很短的情况
- **阻塞锁**：锁不可用时线程挂起，适合临界区较长的情况

### 4.2 信号量（Semaphore）

```cpp
#include <semaphore.h>

sem_t sem;
sem_init(&sem, 0, 1);  // 初始值1

void wait_operation() {
    sem_wait(&sem);  // P操作，减1
    // 临界区
    sem_post(&sem);  // V操作，加1
}
```

**信号量 vs 互斥锁**：
- 互斥锁：二进制信号量，只能被一个线程持有
- 信号量：计数信号，可以被多个线程同时持有

### 4.3 条件变量（Condition Variable）

```cpp
pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
bool ready = false;

void wait_for_ready() {
    pthread_mutex_lock(&mtx);
    while (!ready) {
        pthread_cond_wait(&cond, &mtx);  // 释放锁并等待
    }
    pthread_mutex_unlock(&mtx);
}

void set_ready() {
    pthread_mutex_lock(&mtx);
    ready = true;
    pthread_cond_signal(&cond);  // 唤醒等待的线程
    pthread_mutex_unlock(&mtx);
}
```

### 4.4 屏障（Barrier）

```cpp
#include <pthread.h>

pthread_barrier_t barrier;
pthread_barrier_init(&barrier, NULL, 3);  // 等待3个线程

void sync_point() {
    // 所有线程到达这里之前都会阻塞
    pthread_barrier_wait(&barrier);
    // 之后所有线程同时继续执行
}
```

---

## 5. 死锁

### 5.1 死锁的必要条件（ Coffman 条件）

1. **互斥条件**：资源只能被一个线程持有
2. **持有并等待**：线程持有资源同时等待其他资源
3. **不可抢占**：资源不能被强制释放
4. **循环等待**：形成线程间的循环等待链

### 5.2 死锁示例

```cpp
// 线程A持有锁1，等待锁2
pthread_mutex_lock(&lock1);
pthread_mutex_lock(&lock2);  // 可能阻塞

// 同时线程B持有锁2，等待锁1
pthread_mutex_lock(&lock2);
pthread_mutex_lock(&lock1);  // 死锁！
```

### 5.3 预防死锁的方法

| 方法 | 原理 |
|------|------|
| **固定顺序加锁** | 所有线程按相同顺序获取锁 |
| **try-lock + 回退** | 获取锁失败时释放已持有的锁 |
| **锁层次结构** | 将锁分成多个层次，只能从高层向低层加锁 |

```cpp
// 预防死锁：固定顺序加锁
void lock_in_order(Resource* r1, Resource* r2) {
    if (r1 < r2) {  // 按地址顺序确定加锁顺序
        pthread_mutex_lock(&r1->mtx);
        pthread_mutex_lock(&r2->mtx);
    } else {
        pthread_mutex_lock(&r2->mtx);
        pthread_mutex_lock(&r1->mtx);
    }
}
```

---

## 6. 线程安全与内存可见性

### 6.1 内存可见性问题

```cpp
// 线程A
while (!flag) {
    sleep(1);
}
print("Done");

// 线程B
flag = true;
```

**问题**：由于编译器优化和CPU缓存，线程A可能永远看不到线程B对flag的修改。

### 6.2 Volatile关键字

```cpp
volatile bool flag = false;
```

**作用**：
- 告诉编译器每次读取都要从内存读取（禁用缓存优化）
- 不保证操作的原子性
- 不保证CPU之间的缓存一致性

### 6.3 内存屏障

内存屏障确保指令重排序不会破坏可见性：

| 屏障类型 | 作用 |
|---------|------|
| Store Barrier | 强制所有之前的store操作刷新到缓存 |
| Load Barrier | 强制所有之后的load操作从主存读取 |
| Full Barrier | 同时包含Store和Load Barrier |

---

## 7. C++11 并发编程

### 7.1 std::thread

```cpp
#include <thread>
#include <atomic>

std::atomic<int> counter(0);

void increment() {
    for (int i = 0; i < 1000; ++i) {
        counter.fetch_add(1);  // 原子加法
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    
    t1.join();
    t2.join();
    
    std::cout << counter << std::endl;  // 输出2000
    return 0;
}
```

### 7.2 std::mutex

```cpp
#include <mutex>

class SafeCounter {
    std::mutex mtx;
    int counter = 0;
public:
    void increment() {
        std::lock_guard<std::mutex> lock(mtx);
        counter++;
    }
    
    int get() {
        std::lock_guard<std::mutex> lock(mtx);
        return counter;
    }
};
```

### 7.3 std::condition_variable

```cpp
#include <mutex>
#include <condition_variable>

std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void consumer() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, []{ return ready; });  // 等待直到ready为true
    std::cout << "Consumer: ready!\n";
}

void producer() {
    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;
    }
    cv.notify_one();  // 通知等待的线程
}
```

---

## 8. 常见并发模式

### 8.1 生产者-消费者模式

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>

std::queue<Task> tasks;
std::mutex mtx;
std::condition_variable cv;

void producer() {
    for (int i = 0; i < 10; ++i) {
        std::lock_guard<std::mutex> lock(mtx);
        tasks.push(Task{i});
        cv.notify_one();  // 通知消费者
    }
}

void consumer() {
    while (true) {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, []{ return !tasks.empty(); });
        Task t = tasks.front();
        tasks.pop();
        lock.unlock();
        // 处理任务
        if (t.id == -1) break;  // 结束信号
    }
}
```

### 8.2 读者-写者模式

```cpp
class ReadWriteLock {
    std::mutex mtx;
    std::condition_variable read_cv;
    int readers = 0;
    bool writer_active = false;
    
public:
    void lock_read() {
        std::unique_lock<std::mutex> lock(mtx);
        read_cv.wait(lock, []{ return !writer_active; });
        readers++;
    }
    
    void unlock_read() {
        std::lock_guard<std::mutex> lock(mtx);
        readers--;
        if (readers == 0) {
            read_cv.notify_all();
        }
    }
    
    void lock_write() {
        std::unique_lock<std::mutex> lock(mtx);
        writer_active = true;
        read_cv.wait(lock, []{ return readers == 0; });
    }
    
    void unlock_write() {
        std::lock_guard<std::mutex> lock(mtx);
        writer_active = false;
        read_cv.notify_all();
    }
};
```

---

## 9. 锁的分类

| 类型 | 特点 | 适用场景 |
|------|------|----------|
| **悲观锁** | 假设并发冲突频繁，先加锁再操作 | 写入操作 |
| **乐观锁** | 假设冲突少，使用CAS不加锁 | 读多写少 |
| **公平锁** | 按请求顺序获取锁 | 需要严格顺序 |
| **非公平锁** | 允许插队，性能更高 | 一般场景 |
| **自旋锁** | 循环等待，不释放CPU | 临界区极短 |
| **阻塞锁** | 线程挂起，让出CPU | 临界区较长 |

### 9.1 乐观锁与CAS

```cpp
// CAS操作示例
class AtomicCounter {
    std::atomic<int> value;
public:
    void increment() {
        int old_value = value.load();
        // 不断重试直到成功
        while (!value.compare_exchange_weak(old_value, old_value + 1)) {
            // compare_exchange_weak失败时old_value会被更新
        }
    }
};
```

---

## 10. 实战注意事项

### 10.1 避免的常见错误

1. **不要在持有多锁时调用未知代码**
2. **避免锁的粒度过大**
3. **优先使用RAII管理锁的生命周期**
4. **减少锁的持有时间**

### 10.2 性能优化建议

1. **减小临界区**：锁内的代码越少越好
2. **减少锁竞争**：使用线程本地存储
3. **读写分离**：使用读写锁
4. **无锁结构**：在高竞争场景考虑无锁队列

---

## 11. 参考资料

[^1]: 《操作系统概念》- Abraham Silberschatz
[^2]: 《Java并发编程实战》- Brian Goetz
[^3]: 《C++ Concurrency in Action》- Anthony Williams
[^4]: [C++ Standard Library Reference](https://en.cppreference.com/w/cpp/thread)

---

## 相关主题

- [[../distributed-systems/distributed-systems-overview|分布式系统基础]]
- [[../operating-systems/os-overview|操作系统概述]]
- [[../data-structure/queue|队列与消息队列]]
