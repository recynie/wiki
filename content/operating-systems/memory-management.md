---
title: 内存管理
date: 2026-04-13
description: 操作系统内存管理：虚拟内存、分页、分段、换页算法
tags:
  - operating-systems
  - memory-management
  - virtual-memory
  - paging
draft: false
permalink:
---

# 内存管理

内存管理是操作系统的核心功能之一，负责高效、安全地为进程分配和回收内存资源。

## 内存层次

```
┌────────────────────────────────────┐
│           CPU寄存器                 │ ◄─ 访问速度最快，容量最小
├────────────────────────────────────┤
│          L1/L2/L3 缓存             │ ◄─ 几十KB~几MB
├────────────────────────────────────�
│            主存 (RAM)              │ ◄─ 几GB~几百GB
├────────────────────────────────────┤
│        磁盘存储 (SSD/HDD)          │ ◄─ TB级别，访问最慢
└────────────────────────────────────┘
```

## 连续内存分配

### 固定分区分配

将内存划分为固定大小的分区：

```c
// 固定分区示例
typedef struct {
    int id;           // 分区ID
    int size;         // 分区大小
    int start_addr;   // 起始地址
    int process_id;   // 占用进程，-1表示空闲
} Partition;

Partition partitions[] = {
    {0, 64, 0, -1},      // 64KB分区
    {1, 64, 64, 3},      // 已被进程3占用
    {2, 128, 192, -1},    // 128KB分区
    // ...
};
```

**问题**：内部碎片（分配 > 需要）、分区数量固定

### 动态分区分配

根据进程实际需求分配连续空间。

#### 首次适配（First Fit）

```c
#include <stdbool.h>

typedef struct hole {
    int start;
    int size;
    struct hole *next;
} Hole;

Hole *memory = NULL;  // 内存孔链表

void *first_fit(int process_size) {
    Hole *prev = NULL;
    Hole *curr = memory;
    
    while (curr != NULL) {
        if (curr->size >= process_size) {
            // 找到足够大的孔
            if (curr->size == process_size) {
                // 完全匹配，直接移除孔
                if (prev)
                    prev->next = curr->next;
                else
                    memory = curr->next;
            } else {
                // 分割孔
                curr->start += process_size;
                curr->size -= process_size;
            }
            return (void *)curr->start;
        }
        prev = curr;
        curr = curr->next;
    }
    return NULL;  // 没有足够大的孔
}
```

#### 最佳适配（Best Fit）

```c
void *best_fit(int process_size) {
    Hole *best = NULL;
    Hole *prev = NULL;
    Hole *curr = memory;
    int min_remaining = __INT_MAX__;
    
    while (curr != NULL) {
        if (curr->size >= process_size) {
            int remaining = curr->size - process_size;
            if (remaining < min_remaining) {
                min_remaining = remaining;
                best = curr;
            }
        }
        prev = curr;
        curr = curr->next;
    }
    
    if (best) {
        void *addr = (void *)best->start;
        best->start += process_size;
        best->size -= process_size;
        return addr;
    }
    return NULL;
}
```

## 虚拟内存

### 分页（Paging）

将进程的虚拟地址空间划分为固定大小的页，物理内存划分为同样大小的帧：

```
虚拟地址: [页号 | 页内偏移]
          ↓
        分页机制
          ↓
物理地址: [帧号 | 页内偏移]
```

```c
// 虚拟地址到物理地址转换
typedef struct {
    unsigned int page_number;
    unsigned int offset;
} virtual_addr;

typedef struct {
    unsigned int frame_number;
    unsigned int valid;     // 该页是否在内存中
    unsigned int dirty;     // 是否被修改过
    unsigned int reference;  // 访问位
} page_table_entry;

page_table_entry page_table[MAX_PAGES];

// 地址转换
unsigned long translate(unsigned long virtual_addr) {
    virtual_addr va;
    va.value = virtual_addr;
    
    page_table_entry *entry = &page_table[va.page_number];
    
    if (!entry->valid) {
        // 触发缺页中断
        handle_page_fault(va.page_number);
        entry = &page_table[va.page_number];
    }
    
    entry->reference = 1;  // 设置引用位
    
    return (entry->frame_number << PAGE_SHIFT) | va.offset;
}
```

### 多级页表

```c
// 两级页表结构
typedef struct {
    unsigned int p1_index;  // 一级页表索引
    unsigned int p2_index;  // 二级页表索引
    unsigned int offset;     // 页内偏移
} two_level_addr;

// 地址转换
unsigned long two_level_translate(two_level_addr va, unsigned long pgd_base) {
    // 获取一级页表项
    unsigned long *pgd = (unsigned long *)pgd_base;
    unsigned long pde = pgd[va.p1_index];
    
    if (!(pde & 1)) {
        // 一级页表项无效，需要分配二级页表
        allocate_page_table(va.p1_index);
        pde = pgd[va.p1_index];
    }
    
    // 获取二级页表项
    unsigned long *pte = (unsigned long *)(pde & ~0xFFF);
    unsigned long pte_entry = pte[va.p2_index];
    
    if (!(pte_entry & 1)) {
        // 二级页表项无效，缺页
        handle_page_fault(va.p1_index, va.p2_index);
        pte_entry = pte[va.p2_index];
    }
    
    return ((pte_entry & ~0xFFF) << PAGE_SHIFT) | va.offset;
}
```

## 页面置换算法

### LRU（最近最少使用）

```c
#include <stdlib.h>
#include <stdbool.h>

typedef struct lru_node {
    int page_number;
    struct lru_node *prev;
    struct lru_node *next;
} lru_node;

lru_node *lru_list_head = NULL;  // 链表头（最近使用）
lru_node *lru_list_tail = NULL;  // 链表尾（最久未使用）
int lru_frames[NUM_FRAMES];
int frame_count = 0;

// 访问页面
void access_page(int page_num) {
    lru_node *node = find_in_list(page_num);
    
    if (node) {
        // 已在内存中，移动到链表头
        move_to_head(node);
    } else {
        // 缺页
        if (frame_count >= NUM_FRAMES) {
            // 需要置换
            int victim = lru_list_tail->page_number;
            remove_tail();
            evict_page(victim);
        }
        add_to_head(create_node(page_num));
        load_page(page_num);
    }
}
```

### 时钟算法（Clock Algorithm）

```c
// 时钟页面置换算法
typedef struct {
    int page_number;
    unsigned int use_bit;    // 使用位（访问位）
    unsigned int dirty_bit;  // 脏位
} frame_entry;

frame_entry frames[NUM_FRAMES];
int hand = 0;  // 指针位置

int clock_algorithm() {
    while (true) {
        frame_entry *frame = &frames[hand];
        
        if (frame->use_bit == 0) {
            // 选择该页置换
            int victim = hand;
            hand = (hand + 1) % NUM_FRAMES;
            return victim;
        }
        
        // 清除使用位，继续查找
        frame->use_bit = 0;
        hand = (hand + 1) % NUM_FRAMES;
    }
}
```

### 段页式存储

结合分段和分页的优点：

```
逻辑地址: [段号 | 段内偏移]
              ↓
段表查找 → 得到段基址和限长
              ↓
虚拟地址: [页号 | 页内偏移]
              ↓
页表查找 → 得到物理帧号
              ↓
物理地址: [物理帧号 | 页内偏移]
```

## 内存分配算法

### 伙伴系统（Buddy System）

```c
typedef struct block {
    int order;              // 阶（2^order字节）
    int size;
    struct block *next;
} block;

block *free_lists[MAX_ORDER + 1];

block *buddy_alloc(int size) {
    int order = ceil_log2(size);
    
    for (int o = order; o <= MAX_ORDER; o++) {
        if (free_lists[o] != NULL) {
            block *block = free_lists[o];
            free_lists[o] = block->next;
            
            // 分割成更小的块
            while (o > order) {
                o--;
                block *buddy = block + (1 << o);
                buddy->order = o;
                buddy->size = 1 << o;
                buddy->next = free_lists[o];
                free_lists[o] = buddy;
            }
            
            block->order = order;
            return block;
        }
    }
    return NULL;  // 分配失败
}

void buddy_free(block *block) {
    int order = block->order;
    
    while (order < MAX_ORDER) {
        // 计算伙伴地址
        unsigned long addr = (unsigned long)block;
        unsigned long buddy_addr = addr ^ (1 << order);
        block *buddy = (block *)buddy_addr;
        
        // 检查伙伴是否空闲且大小相同
        if (is_free(buddy) && buddy->order == order) {
            remove_from_free_list(buddy);
            // 合并
            block = (addr < buddy_addr) ? block : buddy;
            order++;
        } else {
            break;
        }
    }
    
    // 加入空闲链表
    block->order = order;
    add_to_free_list(block);
}
```

## 参考

[^1]: Silberschatz et al., Operating System Concepts, Chapter 9
[^2]: Tanenbaum, Modern Operating Systems, Chapter 4

