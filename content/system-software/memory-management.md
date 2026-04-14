---
title: Memory Management
date: 2026-04-12
description: 内存管理基础与自定义分配器策略，包括Arena、Stack、Pool、Buddy等分配器
tags:
  - memory-management
  - system-software
  - allocator
draft: false
permalink:
---

# Memory Management

内存管理是系统软件的核心课题，涉及计算机内存的分配、使用和回收。高效的内存管理对于程序性能至关重要，尤其在游戏引擎、数据库、编译器等需要大量内存分配的场景中。[^1]

[^1]: 本段参考 [Doug Lea's Malloc Documentation](http://gee.cs.oswego.edu/dl/html/malloc.html) 及 CMU 15-445 Database Systems materials。

## 内存管理基础

### 内存层级

现代计算机的内存呈现层级结构：

```
┌─────────────────────────────────────┐
│            寄存器                    │  < 1KB, ~0.5ns
├─────────────────────────────────────┤
│         L1/L2/L3 缓存                │  ~32MB, ~1-10ns
├─────────────────────────────────────┤
│              内存                    │  GB级, ~100ns
├─────────────────────────────────────┤
│           SSD/磁盘                   │  GB级, ~100μs
└─────────────────────────────────────┘
```

越快的存储容量越小，成本越高。内存管理的设计目标之一就是**充分利用缓存局部性**，让数据尽可能停留在快速存储层。

### 堆与栈

程序内存主要分为两部分：

| 区域 | 栈 (Stack) | 堆 (Heap) |
|------|------------|-----------|
| 分配方式 | 编译器自动管理 | 程序员手动请求 |
| 方向 | 高地址向低地址增长 | 低地址向高地址增长 |
| 分配速度 | 极快（移动栈指针） | 较慢（需要复杂算法） |
| 释放方式 | 自动（函数返回时） | 手动（free/delete） |
| 碎片化 | 通常不会 | 可能严重 |

### 内存分配器的基本功能

一个内存分配器需要提供以下功能：

1. **分配（Allocate）**：请求一块指定大小的内存
2. **释放（Deallocate）**：归还不再使用的内存
3. **重新分配（Reallocate）**：调整已分配内存的大小
4. **对齐（Alignment）**：确保内存地址满足特定对齐要求（通常是 8 或 16 字节）

## 默认分配器

### malloc 和 free

`malloc`（memory allocation）是 C 标准库中的内存分配函数：

```cpp
#include <stdlib.h>

// 分配指定字节的内存，返回指向起始地址的指针
void* malloc(size_t size);

// 释放之前分配的内存
void free(void* ptr);

// 重新分配内存，可改变大小
void* realloc(void* ptr, size_t new_size);

// 分配并初始化为零
void* calloc(size_t nmemb, size_t size);
```

#### malloc 的实现原理

现代 `malloc` 实现（如 glibc 的 ptmalloc）通常采用**分离空闲链表（Segregated Free Lists）**策略：

```
内存布局：
┌──────────────────────────────────────────────────────┐
│                    堆区域                              │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐       │
│  │ 已分配  │  │  空闲  │  │ 已分配  │  │  空闲  │       │
│  └────────┘  └────────┘  └────────┘  └────────┘       │
└──────────────────────────────────────────────────────┘

空闲链表结构：
┌────┐    ┌────┐    ┌────┐
│块A │───►│块B │───►│块C │───► NULL
└────┘    └────┘    └────┘
```

**分配过程**：
1. 遍历空闲链表，寻找合适的块
2. 若找到的块过大，可能**分割（Split）**成两部分
3. 若找不到合适的块，向 OS 申请更多内存（`brk`/`mmap`）

**释放过程**：
1. 将块加入相应的空闲链表
2. 检查是否可以与相邻空闲块**合并（Coalesce）**

### new 和 delete

C++ 中的 `new/delete` 是面向对象的内存管理机制：

```cpp
// 分配单个对象
int* p = new int(42);

// 分配数组
int* arr = new int[10];

// 释放
delete p;
delete[] arr;
```

#### new/delete 与 malloc 的区别

| 特性 | new/delete | malloc/free |
|------|------------|-------------|
| 返回类型 | 类型安全，自动转型 | `void*`，需手动转型 |
| 构造函数 | 自动调用 | 不调用 |
| 析构函数 | 自动调用 | 不调用 |
| 内存来源 | 堆（相同） | 堆（相同） |
| 失败处理 | 抛异常（`std::bad_alloc`） | 返回 `NULL`（旧标准） |

#### operator new 的底层实现

`new` 实际上调用底层的 `operator new`：

```cpp
// 全局 operator new 的典型实现
void* operator new(size_t size) {
    if (size == 0) size = 1;  // 保证最小分配
    
    // 循环重试直到成功或抛出异常
    while (true) {
        void* p = malloc(size);
        if (p) return p;
        
        // 分配失败，调用 new_handler
        std::new_handler h = std::get_new_handler();
        if (h) h();
        else throw std::bad_alloc();
    }
}
```

### 内存分配的系统调用

当堆空间不足时，分配器会向操作系统申请更多内存：

| 系统调用 | 说明 | 平台 |
|---------|------|------|
| `brk()` | 设置 program break 指针 | Unix |
| `sbrk()` | 增加 program break | Unix |
| `mmap()` | 映射匿名内存或文件 | Unix/Windows |
| `VirtualAlloc()` | Windows 内存分配 | Windows |

```cpp
// 使用 mmap 分配大块内存的示例
#include <sys/mman.h>

void* large_alloc(size_t size) {
    void* p = mmap(nullptr, size, 
                   PROT_READ | PROT_WRITE,
                   MAP_PRIVATE | MAP_ANONYMOUS, 
                   -1, 0);
    return (p == MAP_FAILED) ? nullptr : p;
}
```

## 自定义分配器策略

自定义分配器通过预先管理一块大内存，根据应用场景选择特定策略来优化分配性能。

### Arena（线性）分配器

Arena 分配器（又称 Linear/Bump 分配器）是最简单的分配策略：**只进不退**。

#### 工作原理

```
初始状态：
┌────────────────────────────────────────────┐
│                  Arena                      │
│  [         可用区域          ][ 已使用 ]     │
│  ↑ cursor                                     │
└────────────────────────────────────────────┘

分配时（bump pointer）：
┌────────────────────────────────────────────┐
│                  Arena                      │
│  [    可用    ][ 已分配块 ][ 已使用 ]        │
│           ↑ cursor                          │
└────────────────────────────────────────────┘

重置后（Reset）：
┌────────────────────────────────────────────┐
│                  Arena                      │
│  [              可用区域                    ][ 已使用 ]│
│  ↑ cursor (恢复到起始位置)                   │
└────────────────────────────────────────────┘
```

#### 核心特性

- **分配复杂度**：O(1)，只需移动指针
- **释放复杂度**：O(1)（无操作）或 O(n)（重置所有分配）
- **内存效率**：无外部碎片，但可能有内部碎片
- **线程安全**：需要额外同步（arena 锁或 TLS）

#### 适用场景

1. **游戏引擎的帧分配**：每帧结束时统一重置
2. **编译器的解析阶段**：词法分析→语法分析→语义分析
3. **短期大量临时对象的分配**

#### C++ 实现

```cpp
#include <cstddef>
#include <cstdlib>
#include <mutex>
#include <cstring>

class ArenaAllocator {
public:
    explicit ArenaAllocator(size_t capacity) 
        : capacity_(capacity), used_(0) {
        // 使用 mmap 分配大块内存，对齐到页面边界
        memory_ = static_cast<char*>(
            aligned_alloc(alignof(std::max_align_t), capacity));
        if (!memory_) {
            throw std::bad_alloc();
        }
    }
    
    ~ArenaAllocator() {
        std::free(memory_);
    }
    
    // 禁止拷贝
    ArenaAllocator(const ArenaAllocator&) = delete;
    ArenaAllocator& operator=(const ArenaAllocator&) = delete;
    
    // 分配内存（bump pointer）
    void* allocate(size_t size, size_t alignment = alignof(std::max_align_t)) {
        // 计算对齐后的地址
        std::lock_guard<std::mutex> lock(mutex_);
        
        char* current = memory_ + used_;
        size_t padding = calculatePadding(current, alignment);
        
        if (used_ + padding + size > capacity_) {
            return nullptr;  // 内存不足
        }
        
        char* aligned = current + padding;
        used_ += padding + size;
        
        return aligned;
    }
    
    // 重置 arena（所有分配一起释放）
    void reset() {
        std::lock_guard<std::mutex> lock(mutex_);
        used_ = 0;
    }
    
    size_t used() const { return used_; }
    size_t capacity() const { return capacity_; }
    
private:
    static size_t calculatePadding(const char* ptr, size_t alignment) {
        size_t addr = reinterpret_cast<size_t>(ptr);
        size_t remainder = addr % alignment;
        return (remainder == 0) ? 0 : (alignment - remainder);
    }
    
    char* memory_;
    size_t capacity_;
    size_t used_;
    std::mutex mutex_;  // 简单起见使用全局锁
};

// C++17 aligned_alloc 支持
// GCC/Clang: 使用 posix_memalign 或 aligned_alloc
// MSVC: 使用 _aligned_malloc
```

#### 使用示例

```cpp
#include <iostream>
#include <string>

struct TempObject {
    int id;
    double value;
    char data[64];
};

int main() {
    ArenaAllocator arena(64 * 1024);  // 64KB arena
    
    // 模拟游戏帧
    for (int frame = 0; frame < 3; ++frame) {
        std::cout << "Frame " << frame << ", used: " << arena.used() << " bytes\n";
        
        // 帧内分配大量临时对象
        for (int i = 0; i < 100; ++i) {
            auto* obj = new (arena.allocate(sizeof(TempObject))) TempObject();
            obj->id = i;
            obj->value = i * 1.5;
        }
        
        // 帧结束，重置 arena
        arena.reset();
    }
    
    return 0;
}
```

### Stack（栈式）分配器

栈式分配器在 Arena 基础上增加了 **LIFO（后进先出）** 约束，允许有序释放。

#### 工作原理

```
初始状态：
┌────────────────────────────────────┐
│ bottom                    top      │
│ [          可用区域           ][标记]│
└────────────────────────────────────┘

分配 100 字节后：
┌────────────────────────────────────┤
│ bottom                    top      │
│ [ 已分配 100B ][  可用  ][ 标记 ]   │
└────────────────────────────────────┘

再分配 50 字节后：
┌────────────────────────────────────┤
│ bottom                    top      │
│ [ 100B ][ 50B ][   可用   ][ 标记 ]│
└────────────────────────────────────┘

释放最近的 50B（撤销标记）：
┌────────────────────────────────────┤
│ bottom                    top      │
│ [      100B      ][   可用   ][标记]│
└────────────────────────────────────┘
```

#### 核心特性

- **分配复杂度**：O(1)
- **释放**：必须逆序释放（与分配顺序相反）
- **支持部分释放**：通过标记（Marker）实现任意点重置

#### C++ 实现

```cpp
#include <cstddef>
#include <cstring>
#include <mutex>

class StackAllocator {
public:
    struct Marker {
        size_t offset;
    };
    
    explicit StackAllocator(size_t capacity)
        : capacity_(capacity), offset_(0) {
        memory_ = static_cast<char*>(std::malloc(capacity));
        if (!memory_) {
            throw std::bad_alloc();
        }
    }
    
    ~StackAllocator() {
        std::free(memory_);
    }
    
    // 获取当前标记
    Marker getMarker() const {
        return Marker{offset_};
    }
    
    // 分配内存
    void* allocate(size_t size, size_t alignment = alignof(std::max_align_t)) {
        std::lock_guard<std::mutex> lock(mutex_);
        
        char* current = memory_ + offset_;
        size_t padding = calculatePadding(current, alignment);
        
        if (offset_ + padding + size > capacity_) {
            return nullptr;
        }
        
        char* aligned = current + padding;
        offset_ += padding + size;
        
        return aligned;
    }
    
    // 释放到指定标记（只撤销在该标记之后分配的所有内存）
    void freeToMarker(Marker marker) {
        std::lock_guard<std::mutex> lock(mutex_);
        offset_ = marker.offset;
    }
    
    // 释放最近一次分配（需要记录每次分配的大小）
    void freeLast(size_t last_size) {
        std::lock_guard<std::mutex> lock(mutex_);
        if (last_size <= offset_) {
            offset_ -= last_size;
        }
    }
    
    void reset() {
        std::lock_guard<std::mutex> lock(mutex_);
        offset_ = 0;
    }
    
private:
    static size_t calculatePadding(const char* ptr, size_t alignment) {
        size_t addr = reinterpret_cast<size_t>(ptr);
        size_t remainder = addr % alignment;
        return (remainder == 0) ? 0 : (alignment - remainder);
    }
    
    char* memory_;
    size_t capacity_;
    size_t offset_;
    std::mutex mutex_;
};
```

### Pool（内存池）分配器

内存池分配器管理**固定大小**的内存块，适合大量同类对象的分配。

#### 工作原理

```
空闲链表结构：
┌─────────────────────────┐
│      Pool Allocator      │
│  ┌────┐ ┌────┐ ┌────┐   │
│  │块1 │ │块2 │ │块3 │   │
│  └────┘ └────┘ └────┘   │
│    │      │      │      │
│    ▼      ▼      ▼      │
│  ┌────────────────────┐  │
│  │   Free Block List  │  │
│  │  [next指针][next指针]│  │
│  └────────────────────┘  │
└─────────────────────────┘
```

每个空闲块包含一个 `next` 指针指向下一个空闲块。分配时从链表头部取块，释放时放回链表头部。

#### 核心特性

- **分配复杂度**：O(1)
- **释放复杂度**：O(1)
- **内存效率**：无内部碎片，外部碎片仅限小块
- **固定大小**：只能分配单一尺寸

#### 适用场景

1. **游戏引擎粒子系统**：大量同尺寸粒子
2. **网络包缓冲**：固定大小的数据包
3. **对象池模式**：数据库连接池、线程池

#### C++ 实现

```cpp
#include <cstddef>
#include <cstdlib>
#include <mutex>

class PoolAllocator {
public:
    PoolAllocator(size_t block_size, size_t block_count)
        : block_size_(block_size), block_count_(block_count) {
        // 计算实际块大小（包含链表指针）
        size_t total_size = block_size_ * block_count;
        
        memory_ = std::malloc(total_size);
        if (!memory_) {
            throw std::bad_alloc();
        }
        
        // 初始化空闲链表
        free_list_ = nullptr;
        char* current = static_cast<char*>(memory_);
        
        for (size_t i = 0; i < block_count_; ++i) {
            char* block = current + i * block_size_;
            char** link = reinterpret_cast<char**>(block);
            *link = free_list_;
            free_list_ = block;
        }
    }
    
    ~PoolAllocator() {
        std::free(memory_);
    }
    
    // 分配一个块
    void* allocate() {
        std::lock_guard<std::mutex> lock(mutex_);
        
        if (!free_list_) {
            return nullptr;  // 池已空
        }
        
        char* block = free_list_;
        free_list_ = *reinterpret_cast<char**>(block);
        
        return block;
    }
    
    // 释放一个块
    void deallocate(void* ptr) {
        std::lock_guard<std::mutex> lock(mutex_);
        
        char* block = static_cast<char*>(ptr);
        char** link = reinterpret_cast<char**>(block);
        *link = free_list_;
        free_list_ = block;
    }
    
    size_t blockSize() const { return block_size_; }
    size_t totalBlocks() const { return block_count_; }
    
    // 估算可用块数（不精确，需要额外统计）
    size_t available() const {
        std::lock_guard<std::mutex> lock(mutex_);
        size_t count = 0;
        char* current = free_list_;
        while (current) {
            ++count;
            current = *reinterpret_cast<char**>(current);
        }
        return count;
    }
    
private:
    size_t block_size_;
    size_t block_count_;
    char* memory_;
    char* free_list_;  // 空闲块链表头
    std::mutex mutex_;
};

// 使用模板简化类型安全的分配
template<typename T>
class TypedPoolAllocator {
public:
    explicit TypedPoolAllocator(size_t initial_count = 100) {
        pool_ = new PoolAllocator(sizeof(T), initial_count);
    }
    
    ~TypedPoolAllocator() {
        delete pool_;
    }
    
    template<typename... Args>
    T* create(Args&&... args) {
        void* mem = pool_->allocate();
        if (!mem) return nullptr;
        return new (mem) T(std::forward<Args>(args)...);
    }
    
    void destroy(T* obj) {
        obj->~T();
        pool_->deallocate(obj);
    }
    
private:
    PoolAllocator* pool_;
};
```

### Buddy Allocator

Buddy Allocator 是一种**二进制指数**分配器，将内存划分为 2^k 大小的块。

#### 工作原理

```
初始状态（4KB = 2^12）：
┌────────────────────────────────────┐
│            4KB 块                  │
│              [0]                    │
└────────────────────────────────────┘

分裂（Split）过程：
[0] → 分割成两个 2KB
┌──────────────┬──────────────┐
│    2KB [1]   │    2KB [1]   │
└──────────────┴──────────────┘

再次分裂第一个 2KB：
┌──────┬──────┬──────┬──────┐
│ 1KB  │ 1KB  │ 2KB  │ 2KB  │
│ [2]  │ [2]  │ [1]  │ [1]  │
└──────┴──────┴──────┴──────┘

分配请求（需要 1KB）：
返回第一个 1KB 块 [2]

释放时检查 buddy（相邻块）是否可以合并：
```

#### Buddy 系统的关键特性

- **块大小**：始终为 2^k（k ∈ [min_order, max_order]）
- **分裂**：大块分成两个相等的 "buddy"
- **合并**：只有两个 buddy 都是空闲时才能合并
- **Buddy 地址计算**：`buddy_addr = addr ^ block_size`

#### 核心特性

- **分配复杂度**：O(log N)
- **合并检查**：O(1)
- **无外部碎片**：但有内部碎片（请求大小向上取整到 2^k）
- **内存利用率**：通常 50% 左右

#### C++ 实现

```cpp
#include <cstddef>
#include <cstdlib>
#include <cassert>
#include <vector>
#include <mutex>

class BuddyAllocator {
public:
    // min_order: 最小块 2^min_order 字节
    // max_order: 最大块 2^max_order 字节
    BuddyAllocator(size_t min_order, size_t max_order)
        : min_order_(min_order), max_order_(max_order) {
        
        size_t max_size = 1UL << max_order;
        memory_ = std::malloc(max_size);
        if (!memory_) {
            throw std::bad_alloc();
        }
        
        // 初始化空闲链表数组
        free_lists_.resize(max_order - min_order + 1, nullptr);
        
        // 初始时整个内存是一个大的空闲块
        free_lists_[max_order - min_order] = memory_;
    }
    
    ~BuddyAllocator() {
        std::free(memory_);
    }
    
    void* allocate(size_t size) {
        std::lock_guard<std::mutex> lock(mutex_);
        
        // 找到最小的能满足大小的阶
        size_t order = min_order_;
        size_t block_size = 1UL << order;
        
        while (block_size < size && order < max_order_) {
            ++order;
            block_size <<= 1;
        }
        
        if (order > max_order_) {
            return nullptr;  // 请求过大
        }
        
        // 找到第一个有空块的阶
        size_t target_order = order;
        while (target_order <= max_order_ && 
               free_lists_[target_order - min_order_] == nullptr) {
            ++target_order;
        }
        
        if (target_order > max_order_) {
            return nullptr;  // 没有足够的内存
        }
        
        // 从该阶开始分裂，直到达到请求的阶
        void* block = free_lists_[target_order - min_order_];
        free_lists_[target_order - min_order_] = 
            *static_cast<void**>(block);
        
        while (target_order > order) {
            --target_order;
            size_t buddy_size = 1UL << target_order;
            void* buddy = static_cast<char*>(block) + buddy_size;
            *static_cast<void**>(buddy) = free_lists_[target_order - min_order_];
            free_lists_[target_order - min_order_] = buddy;
        }
        
        return block;
    }
    
    void deallocate(void* ptr, size_t size) {
        std::lock_guard<std::mutex> lock(mutex_);
        
        // 计算阶
        size_t order = min_order_;
        size_t block_size = 1UL << order;
        
        while (block_size < size && order < max_order_) {
            ++order;
            block_size <<= 1;
        }
        
        // Buddy 地址计算
        size_t buddy_mask = block_size;
        size_t addr = reinterpret_cast<size_t>(ptr);
        size_t base = reinterpret_cast<size_t>(memory_);
        
        // 合并直到不能合并为止
        while (order < max_order_) {
            size_t buddy_addr = addr ^ block_size;
            
            // 检查 buddy 是否在内存范围内且相同阶
            if (buddy_addr < base || 
                buddy_addr >= base + (1UL << max_order_)) {
                break;
            }
            
            // 检查 buddy 是否真的空闲
            void** curr = &free_lists_[order - min_order_];
            void** prev = nullptr;
            
            while (*curr != nullptr) {
                if (reinterpret_cast<size_t>(*curr) == buddy_addr) {
                    // 找到 buddy，移除并合并
                    if (prev) {
                        *prev = *curr;
                    } else {
                        free_lists_[order - min_order_] = *curr;
                    }
                    
                    // 合并到低阶
                    --order;
                    block_size >>= 1;
                    
                    // 合并后的块地址是较小的那个
                    if (buddy_addr < addr) {
                        addr = buddy_addr;
                    }
                    ptr = reinterpret_cast<void*>(addr);
                    break;
                }
                prev = curr;
                curr = static_cast<void**>(*curr);
            }
            
            if (*curr == nullptr && prev == nullptr && 
                free_lists_[order - min_order_] == nullptr) {
                // 没有找到 buddy，停止合并
                break;
            }
        }
        
        // 放入对应阶的空闲链表
        *static_cast<void**>(ptr) = free_lists_[order - min_order_];
        free_lists_[order - min_order_] = ptr;
    }
    
private:
    size_t min_order_;
    size_t max_order_;
    void* memory_;
    std::vector<void*> free_lists_;  // free_lists_[i] 对应 2^(min_order_+i) 大小的块
    std::mutex mutex_;
};
```

### Slab Allocator

Slab 分配器最初由 Sun Microsystems 为 Solaris 内核设计，用于解决频繁的小对象分配问题。

#### 工作原理

```
Slab Allocator 结构：

┌─────────────────────────────────────────────────┐
│              Cache (e.g., object size = 64B)     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐          │
│  │ Slab 1  │  │ Slab 2  │  │ Slab 3  │   ...    │
│  │ ○○○○○○○ │  │ ○○○○○○○ │  │ ○○○○○○○ │          │
│  │ ○○○○○○○ │  │ ○○○○○○○ │  │         │          │
│  │ ○○○○○○○ │  │         │  │         │          │
│  └─────────┘  └─────────┘  └─────────┘          │
│    部分空闲     部分空闲      满                   │
└─────────────────────────────────────────────────┘

每个 Slab 包含多个对象：
┌─────────────────────────────────┐
│  Slab (e.g., 4KB)               │
│  ┌────┬────┬────┬────┬────────┐ │
│  │obj1│obj2│obj3│obj4│  ...   │ │
│  └────┴────┴────┴────┴────────┘ │
│           ▲                     │
│         对象大小 64B             │
└─────────────────────────────────┘
```

#### 核心特性

- **分配复杂度**：O(1)
- **缓存友好**：同一 Slab 的对象紧密排列
- **着色（Coloring）**：通过偏移调整避免伪共享
- **预热（Warm-up）**：Slab 创建时预填充常用对象

#### 适用场景

1. **内核内存管理**：Linux/Solaris 内核
2. **数据库缓冲池**：固定大小页的缓存
3. **网络协议栈**：TCP/UDP 报文头

#### C++ 实现

```cpp
#include <cstddef>
#include <cstdlib>
#include <vector>
#include <mutex>

class SlabAllocator {
public:
    struct Slab {
        char* memory;           // Slab 内存起始
        size_t object_size;     // 对象大小
        size_t object_count;    // 对象数量
        size_t free_count;      // 空闲对象数
        
        // 空闲链表（嵌入对象中）
        char* free_list;        // 指向第一个空闲对象
        
        Slab(size_t os, size_t oc) 
            : object_size(os), object_count(oc), free_count(oc), free_list(nullptr) {
            memory = static_cast<char*>(std::malloc(object_size * object_count));
            
            // 初始化空闲链表
            free_list = memory;
            for (size_t i = 0; i < object_count; ++i) {
                char* obj = memory + i * object_size;
                char** next = reinterpret_cast<char**>(obj);
                if (i + 1 < object_count) {
                    *next = memory + (i + 1) * object_size;
                } else {
                    *next = nullptr;
                }
            }
        }
        
        ~Slab() {
            std::free(memory);
        }
        
        bool contains(void* ptr) const {
            char* p = static_cast<char*>(ptr);
            return p >= memory && p < memory + object_size * object_count;
        }
    };
    
    SlabAllocator(size_t object_size, size_t objects_per_slab = 1024)
        : object_size_(object_size), objects_per_slab_(objects_per_slab) {
        // 4KB 对齐
        if (object_size_ < sizeof(void*)) {
            object_size_ = sizeof(void*);
        }
    }
    
    ~SlabAllocator() {
        for (auto* slab : slabs_) {
            delete slab;
        }
    }
    
    void* allocate() {
        std::lock_guard<std::mutex> lock(mutex_);
        
        // 找有空闲对象的 slab
        for (auto* slab : slabs_) {
            if (slab->free_count > 0) {
                char* obj = slab->free_list;
                slab->free_list = *reinterpret_cast<char**>(obj);
                --slab->free_count;
                return obj;
            }
        }
        
        // 没有空闲 slab，创建新的
        auto* new_slab = new Slab(object_size_, objects_per_slab_);
        slabs_.push_back(new_slab);
        
        // 分配第一个对象
        char* obj = new_slab->free_list;
        new_slab->free_list = *reinterpret_cast<char**>(obj);
        --new_slab->free_count;
        return obj;
    }
    
    void deallocate(void* ptr) {
        std::lock_guard<std::mutex> lock(mutex_);
        
        // 找到 ptr 所在的 slab
        for (auto* slab : slabs_) {
            if (slab->contains(ptr)) {
                char* obj = static_cast<char*>(ptr);
                *reinterpret_cast<char**>(obj) = slab->free_list;
                slab->free_list = obj;
                ++slab->free_count;
                return;
            }
        }
    }
    
    size_t objectSize() const { return object_size_; }
    size_t totalObjects() const { return slabs_.size() * objects_per_slab_; }
    size_t freeObjects() const {
        size_t total = 0;
        for (auto* slab : slabs_) {
            total += slab->free_count;
        }
        return total;
    }
    
private:
    size_t object_size_;
    size_t objects_per_slab_;
    std::vector<Slab*> slabs_;
    std::mutex mutex_;
};
```

## 线程安全与并发

### 多线程分配的问题

在多线程环境下，传统的空闲链表分配器成为性能瓶颈：

```
Thread A ──┐
           ├──► [Global Lock] ──► malloc/free ──► 串行化！
Thread B ──┘
```

### Per-CPU/Arena 策略

现代高性能分配器（如 TCMalloc、jemalloc）采用 **per-thread arena** 策略：

```
┌─────────────────────────────────────────────────────┐
│                    全局堆                            │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐             │
│  │ Arena 0 │  │ Arena 1 │  │ Arena 2 │   ...       │
│  └────┬────┘  └────┬────┘  └────┬────┘             │
│       │            │            │                   │
│   CPU 0         CPU 1         CPU 2                  │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐             │
│  │Thread 1 │  │Thread 2 │  │Thread 3 │             │
│  │Thread 4 │  │Thread 5 │  │Thread 6 │             │
│  └─────────┘  └─────────┘  └─────────┘             │
└─────────────────────────────────────────────────────┘
```

每个线程有自己的本地缓存（Thread Cache），大部分分配在本地完成，无需加锁。

### Thread-Local Arena 实现

```cpp
#include <cstddef>
#include <memory>
#include <vector>
#include <thread>
#include <mutex>
#include <array>

class ThreadLocalArena {
public:
    struct TLSlot {
        ArenaAllocator* arena = nullptr;
        char padding[64 - sizeof(ArenaAllocator*)];  // 避免 false sharing
    };
    
    ThreadLocalArena(size_t arena_capacity, size_t num_slots)
        : capacity_(arena_capacity), num_slots_(num_slots) {
        // 主 arena 用于 fallback
        main_arena_ = std::make_unique<ArenaAllocator>(capacity_);
        
        // 初始化 TLS 数组
        tls_array_.resize(num_slots);
    }
    
    ~ThreadLocalArena() = default;
    
    void* allocate(size_t size) {
        TLSlot& slot = getCurrentSlot();
        
        // 第一次访问时创建 thread-local arena
        if (!slot.arena) {
            // 主 arena 加锁分配一个新 arena
            std::lock_guard<std::mutex> lock(mutex_);
            slot.arena = new ArenaAllocator(capacity_);
        }
        
        // 尝试 thread-local arena
        void* ptr = slot.arena->allocate(size);
        if (ptr) return ptr;
        
        // thread-local arena 满了，使用主 arena
        return main_arena_->allocate(size);
    }
    
    void reset() {
        // 只重置当前线程的 arena
        TLSlot& slot = getCurrentSlot();
        if (slot.arena) {
            slot.arena->reset();
        }
    }
    
private:
    TLSlot& getCurrentSlot() {
        // 简单的 slot 选择策略（实际可用 ID）
        static thread_local size_t slot_index = 0;
        return tls_array_[slot_index % num_slots_];
    }
    
    size_t capacity_;
    size_t num_slots_;
    std::unique_ptr<ArenaAllocator> main_arena_;
    std::vector<TLSlot> tls_array_;
    std::mutex mutex_;
};
```

## 内存碎片化

### 外部碎片

外部碎片是**已分配块之间**的空闲空间，无法满足新的分配请求：

```
外部碎片示例：
┌────┐    ┌────────┐    ┌────┐
│已分配│   │ 空闲   │    │已分配│
│  A  │   │  (50B) │    │  B  │
└────┘    └────────┘    └────┘

如果请求 60B，即使总空闲 > 60，也无法满足
```

### 内部碎片

内部碎片是**分配块内部**未被使用的空间：

```
内部碎片示例：
┌────────────────────────┐
│      已分配块 (100B)    │
│  ┌──────────────────┐  │
│  │   请求: 60B      │  │
│  │                  │  │
│  │   内部碎片: 40B  │  │
│  └──────────────────┘  │
└────────────────────────┘
```

### 碎片控制策略

| 策略 | 说明 | 优点 | 缺点 |
|------|------|------|------|
| 合并（Coalescing） | 释放时合并相邻空闲块 | 减少外部碎片 | 增加释放时间 |
| 分割（Splitting） | 分配时分割大块 | 提高利用率 | 增加复杂度 |
| 保险分配 | 多分配额外内存 | 减少碎片 | 浪费内存 |
| 定期压缩 | 移动已分配块 | 消除碎片 | 高成本 |

## 缓存局部性

### 空间局部性与时间局部性

- **空间局部性（Spatial Locality）**：访问某个地址后，很可能会访问附近的地址
- **时间局部性（Temporal Locality）**：访问某个地址后，很可能会再次访问

### 缓存行与内存访问

现代 CPU 以**缓存行（Cache Line）**为单位管理缓存，通常为 64 字节：

```
缓存行示例（64B）：
┌─────────────────────────────────────────────────┐
│ addr: 0x1000                                     │
│  ┌────┬────┬────┬────┬────┬────┬────┬────┐     │
│  │  0 │  1 │  2 │  3 │  4 │  5 │  6 │  7 │ ...  │
│  └────┴────┴────┴────┴────┴────┴────┴────┘     │
│   8B   8B   8B   8B   8B   8B   8B   8B          │
└─────────────────────────────────────────────────┘
```

### 分配器对缓存的影响

```cpp
// 缓存不友好的分配
class BadExample {
    std::vector<Object*> objects_;  // 指针散布在堆中
    
    void process() {
        for (auto* obj : objects_) {
            obj->compute();  // 对象在内存中不连续
        }
    }
};

// 缓存友好的分配（使用连续内存）
class GoodExample {
    std::vector<Object> objects_;  // 对象连续存储
    
    void process() {
        for (auto& obj : objects_) {
            obj.compute();  // 顺序访问，缓存命中率高
        }
    }
};
```

### Slab 分配器的缓存优势

Slab 分配器的对象在同一 Slab 内连续排列：

```
Slab 内对象布局：
┌────────────────────────────────────────────────┐
│ [obj0][obj1][obj2][obj3][obj4][obj5][obj6]...  │
│  ↑     ↑     ↑                                  │
│  同一缓存行可能包含多个对象                      │
└────────────────────────────────────────────────┘

预取（Prefetch）可以大幅提升遍历性能：
for (size_t i = 0; i < n; ++i) {
    __builtin_prefetch(&objects_[i + 4]);  // 预取后续对象
    objects_[i].compute();
}
```

## 总结

内存分配器的选择应根据具体场景决定：

| 分配器 | 分配速度 | 碎片处理 | 适用场景 |
|--------|---------|---------|---------|
| Arena | ★★★★★ | 无外部碎片 | 帧分配、解析器 |
| Stack | ★★★★☆ | 无外部碎片 | LIFO 内存模式 |
| Pool | ★★★★★ | 无内部碎片 | 同尺寸对象池 |
| Buddy | ★★★☆☆ | 适中 | 内核、实时系统 |
| Slab | ★★★★★ | 极低 | 对象缓存、数据库 |

现代高性能分配器（TCMalloc、jemalloc）实际上是**多种策略的组合**：

1. **Thread-Local Cache**：减少锁竞争
2. **分离空闲链表**：减少搜索范围
3. **Segregated Size Class**：平衡碎片与效率
4. **Arena 模式**：批量预分配
5. **智能合并**：控制碎片

理解这些基础概念，有助于在特定场景下设计高效的自定义分配器，或更好地使用现有分配器。

## 参考

- [Doug Lea's Malloc Documentation](http://gee.cs.oswego.edu/dl/html/malloc.html)
- [CMU 15-445: Database Systems](https://15445.courses.cs.cmu.edu/)
- [TCMalloc: Thread-Caching Malloc](https://google.github.io/tcmalloc/)
- [jemalloc: General Purpose Scalable Memory Allocator](https://jemalloc.net/)
- [USENIX ATC: Mimalloc](https://www.microsoft.com/en-us/research/uploads/prod/2019/11/mimalloc-tr25-2019.pdf)
- [Linux Kernel Memory Management: Slab Allocator](https://www.kernel.org/doc/html/latest/mm/slab.html)
