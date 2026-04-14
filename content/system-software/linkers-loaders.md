---
title: 链接器与加载器 (Linkers & Loaders)
date: 2026-04-09
description: 链接器与加载器详解 — 从目标文件到可执行程序
tags:
  - linker
  - loader
  - system-software
draft: false
permalink:
---

# 链接器与加载器 (Linkers & Loaders)

链接器（Linker）和加载器（Loader）是系统软件的核心组件，负责将编译后的目标文件组合成可执行程序，并在程序运行时将其加载到内存中。[^1]

[^1]: 本段参考[How Linkers and Loaders Really Work (Level Up Coding)](https://levelup.gitconnected.com/the-silent-machinery-behind-every-program-63e4d834a9b4)

## 编译流程回顾

完整的程序构建流程：

```
源文件.c/.cpp
    │
    ▼
[编译器] ──► 汇编文件.s
    │
    ▼
[汇编器] ──► 目标文件.o (Object File)
    │
    ▼
[链接器] ──► 可执行文件 (Executable)
    │
    ▼
[加载器] ──► 内存中的程序
```

## 目标文件 (Object File)

### 目标文件的类型

| 类型 | 格式 | 特点 |
|------|------|------|
| 可重定位目标文件 | .o (ELF) | 需要进一步链接 |
| 共享目标文件 | .so (ELF) | 可在运行时加载 |
| 可执行目标文件 | 无扩展名 (ELF) | 可直接运行 |
| 核心转储文件 | core (ELF) | 程序崩溃时的内存快照 |

### ELF 文件结构

ELF（Executable and Linkable Format）是 Linux 使用的目标文件格式：

```
┌─────────────────┐
│ ELF Header      │  ← 魔数、入口点、段表偏移
├─────────────────┤
│ Program Headers │  ← 加载时使用（可执行文件）
│   (Loadable)    │
├─────────────────┤
│ .text           │  ← 代码段（机器指令）
├─────────────────┤
│ .rodata         │  ← 只读数据（字符串常量）
├─────────────────┤
│ .data           │  ← 已初始化的全局/静态变量
├─────────────────┤
│ .bss            │  ← 未初始化的全局/静态变量
├─────────────────┤
│ .symtab         │  ← 符号表
├─────────────────┤
│ .rel/.rela      │  ← 重定位信息
├─────────────────┤
│ .debug          │  ← 调试信息
├─────────────────┤
│ Section Headers │  ← 链接时使用
└─────────────────┘
```

### 查看目标文件

使用 `readelf` 和 `objdump` 查看 ELF 文件：

```bash
# 查看 ELF 头
readelf -h main.o

# 查看段信息
readelf -S main.o

# 查看符号表
readelf -s main.o

# 查看反汇编
objdump -d main.o

# 查看字符串常量
readelf -p .rodata main.o
```

## 符号 (Symbols)

### 符号的类型

| 符号类型 | 说明 | 示例 |
|---------|------|------|
| 全局符号 | 定义并可被其他文件引用 | `int global_var;` |
| 外部符号 | 其他文件定义，当前文件引用 | `extern int other_var;` |
| 局部符号 | 仅当前文件使用 | `static int file_scope;` |
| 函数符号 | 函数入口地址 | `void foo() {}` |
| 段符号 | 指向特定段 | `.text` 段起始 |

### 符号表结构

```cpp
// 符号表条目
struct Elf64_Sym {
    uint32_t st_name;   // 符号名（字符串表索引）
    uint8_t  st_info;  // 符号类型和绑定信息
    uint8_t  st_other; // 符号可见性
    uint16_t st_shndx; // 所在段索引
    uint64_t st_value; // 符号值（地址或偏移）
    uint64_t st_size;  // 符号大小
};
```

### nm 命令查看符号

```bash
nm main.o
# 输出：
# 0000000000000000 T main        # T = 全局符号（在 .text）
# 0000000000000000 D data_var    # D = 已初始化数据
# 0000000000000000 b bss_var     # b = 未初始化数据（局部）
#          U printf              # U = 未定义（外部引用）
```

## 链接过程

### 静态链接 (Static Linking)

静态链接在**编译时**完成，由链接器将多个目标文件合并：

```
a.o + b.o + lib.a
       │
       ▼
   [链接器]
       │
       ▼
   可执行文件
```

### 链接的两阶段

**1. 符号解析（Symbol Resolution）**

将符号引用与符号定义匹配：

```cpp
// a.cpp
int shared = 42;           // 定义全局符号
void func() { shared++; } // 引用外部符号

// b.cpp
extern int shared;        // 声明外部符号
void use() { shared--; }
```

链接器在所有目标文件中查找每个未定义符号的定义。

**2. 重定位（Relocation）**

将符号地址调整为最终的内存位置：

```
原始指令:  CALL 0x0        # 调用地址为 0（占位）
重定位后:  CALL 0x401000   # 调用地址为实际地址
```

### 重定位条目

```cpp
// ELF 重定位条目
struct Elf64_Rela {
    uint64_t r_offset;  // 需要重定位的位置
    uint64_t r_info;    // 符号索引和重定位类型
    int64_t  r_addend;  // 加法常数
};
```

常见的重定位类型：
- `R_X86_64_PC32`：相对 PC 的 32 位地址
- `R_X86_64_32`：绝对 32 位地址
- `R_X86_64_JUMP_SLOT`：PLT 条目重定位

## 静态库 (Static Libraries)

静态库（.a 文件）是多个目标文件的集合，用于按需链接：

```bash
# 创建静态库
ar rcs libfoo.a foo1.o foo2.o

# 查看库内容
ar -t libfoo.a
# foo1.o
# foo2.o

# 链接时使用
gcc main.o -L. -lfoo -o main
```

### 链接器如何处理静态库

链接器按**命令行顺序**处理目标文件和库：

1. 维护一组**未解析符号**
2. 扫描目标文件，添加定义并解析引用
3. 扫描库，只提取包含**未解析符号**定义的目标文件
4. 重复直到没有新符号被解析

**注意**：库的组织顺序很重要！

## 动态链接 (Dynamic Linking)

### 共享库 (Shared Libraries)

共享库在运行时被多个进程共享：

```bash
# 编译共享库
gcc -fPIC -shared -o libfoo.so foo.c

# 编译时链接
gcc main.c -L. -lfoo -Wl,-rpath=. -o main

# 运行时依赖
ldd main
# libfoo.so => ./libfoo.so (0x...)
# libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
```

### 位置无关代码 (PIC)

共享库必须生成**位置无关代码（Position Independent Code）**：

```cpp
// 普通代码（不适合共享库）
int global_var;
void* get_global_addr() { return &global_var; }

// PIC 代码
void* get_global_addr() {
    // 通过 GOT（全局偏移表）访问
    extern void* _GLOBAL_OFFSET_TABLE_;
    return &_GLOBAL_OFFSET_TABLE_[global_var_index];
}
```

### 延迟绑定 (Lazy Binding)

函数调用使用 PLT（过程链接表）延迟绑定：

```
调用者 → PLT[printf] → 首次: 链接器解析 → 实际函数
                          后续: 直接跳转
```

### 运行时加载

程序可以使用 `dlopen` 动态加载共享库：

```cpp
#include <dlfcn.h>

int main() {
    // 加载共享库
    void* handle = dlopen("./libfoo.so", RTLD_LAZY);
    if (!handle) {
        fprintf(stderr, "dlopen error: %s\n", dlerror());
        return 1;
    }
    
    // 获取函数指针
    typedef int (*add_func)(int, int);
    add_func add = (add_func)dlsym(handle, "add");
    
    // 调用函数
    printf("2 + 3 = %d\n", add(2, 3));
    
    // 关闭库
    dlclose(handle);
    return 0;
}
```

## 加载器 (Loader)

加载器负责将可执行文件加载到内存并启动执行：

### 程序加载过程

```
execve("a.out")
    │
    ▼
[内核] 创建新进程
    │
    ▼
读取 ELF 头，确定加载方式
    │
    ▼
根据 Program Headers 加载各段到内存
    │
    ▼
设置栈（参数、环境变量）
    │
    ▼
将控制权交给入口点（_start）
    │
    ▼
C 运行时启动 (crt1.o)
    │
    ▼
main()
```

### Program Headers

```bash
readelf -l a.out

# 输出：
# Elf file type is DYN (Shared object file)
# Entry point 0x1000
# 
# Program Header:
#   PHDR off    0x40 vaddr 0x1000 ftype
#   LOAD off    0x0 vaddr 0x0 prot=RWE type=LOAD
#   DYNAMIC     ...
#   NOTE        ...
```

### 内存布局

进程的标准内存布局：

```
高位地址
┌──────────────────┐
│     内核空间      │  (kernel space)
├──────────────────┤ ← 0xFFFFFFFF
│      栈空间       │  ↓ 增长方向
│      ↓           │
│                  │
│      ↑           │
│      ↑           │
│     堆空间        │  ↑ 增长方向
├──────────────────┤
│   .bss (未初始化) │
├──────────────────┤
│   .data (已初始化) │
├──────────────────┤
│   .rodata (只读)  │
├──────────────────┤
│   .text (代码)    │
├──────────────────┤ ← 0x0400000
│   保留区          │
└──────────────────┘
低位地址
```

## 重定位 (Relocation) 深入

### 符号解析算法

```cpp
// 简化的链接过程
class Linker {
    map<string, Symbol*> globalSymbols;
    vector<ObjectFile*> objectFiles;
    
    void link() {
        // 第一遍：收集所有符号定义
        for (auto* obj : objectFiles) {
            for (auto* sym : obj->symbols) {
                if (sym->isDefined()) {
                    globalSymbols[sym->name] = sym;
                }
            }
        }
        
        // 第二遍：重定位符号引用
        for (auto* obj : objectFiles) {
            for (auto* reloc : obj->relocations) {
                Symbol* target = globalSymbols[reloc->symbolName];
                applyRelocation(reloc, target);
            }
        }
    }
};
```

### 常见重定位类型

| 类型 | 说明 | x86-64 |
|------|------|--------|
| R_X86_64_32 | 32位绝对地址 | `movl $sym, %eax` |
| R_X86_64_PC32 | 32位 PC 相对地址 | `call sym` |
| R_X86_64_PLT32 | PLT 入口地址 | `call printf@PLT` |
| R_X86_64_GOTPCREL | GOT 条目 PC 相对 | `movq sym@got(%rip), %rax` |

## 链接器脚本 (Linker Script)

链接器脚本控制内存布局：

```ld
/* linker.ld */
ENTRY(_start)

SECTIONS
{
    . = 0x10000;          /* 代码加载地址 */
    
    .text : {
        *(.text)
        *(.text.*)
    }
    
    . = ALIGN(0x1000);    /* 4KB 对齐 */
    
    .data : {
        *(.data)
    }
    
    .bss : {
        *(.bss)
        *(COMMON)
    }
}
```

### 使用链接器脚本

```bash
gcc -Wl,-Tlinker.ld -o program main.o
```

## 常见链接错误

### 未定义符号 (Undefined Symbol)

```bash
main.o: undefined reference to `foo'
collect2: error: ld returned 1 exit status
```

解决：确保符号定义所在的目标文件或库被链接。

### 重复定义 (Duplicate Symbol)

```bash
a.o:(.data+0x0): multiple definition of `global_var'
b.o:(.data+0x0): first defined here
```

解决：检查是否有多个目标文件定义了同一全局符号。

### 错误顺序

```bash
undefined reference to `foo'
```

解决：库应该放在被依赖目标之后，或使用 `--start-group` `--end-group`。

## 工具链

### GCC 驱动的工作

```
gcc main.c -o main
    │
    ▼
cpp (预处理器) → main.i
    │
    ▼
cc1 (编译器) → main.s
    │
    ▼
as (汇编器) → main.o
    │
    ▼
ld (链接器) → main
```

### 相关工具

| 工具 | 用途 |
|------|------|
| `ar` | 创建/查看静态库 |
| `ranlib` | 生成库索引 |
| `nm` | 查看符号表 |
| `objdump` | 反汇编 |
| `readelf` | 查看 ELF 结构 |
| `ldd` | 查看动态依赖 |
| `strace` | 跟踪系统调用 |
| `ltrace` | 跟踪库调用 |

## 总结

链接器和加载器是程序从源码到运行的关键桥梁：

1. **链接器**：将目标文件合并，解析符号，生成可执行文件
2. **加载器**：将可执行文件加载到内存，设置执行环境
3. **静态链接**：编译时完成，生成独立可执行文件
4. **动态链接**：运行时加载，节省内存，支持插件
5. **PIC**：位置无关代码是共享库的基础

理解链接和加载对于：
- 调试链接错误
- 理解程序内存布局
- 优化二进制大小
- 开发插件系统

都非常重要。

## 参考

- [CSAPP Chapter 7: Linking](https://csapp.cs.cmu.edu/3e/home.html)
- [How Linkers and Loaders Really Work (Level Up Coding)](https://levelup.gitconnected.com/the-silent-machinery-behind-every-program-63e4d834a9b4)
- [Lecture 7: Object Codes, Loaders and Linkers (Iowa)](https://homepage.divms.uiowa.edu/~jones/opsys/notes/07.shtml)
