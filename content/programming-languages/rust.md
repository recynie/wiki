---
title: Rust基础
date: 2026-04-11
description: Rust编程语言入门：所有权系统、借用检查器、生命周期
tags:
  - rust
  - systems-programming
  - memory-safety
draft: false
permalink:
---

## 概述

Rust是一种追求**内存安全**的系统编程语言，无需垃圾回收器或运行时开销。它通过**所有权系统（Ownership）**、**借用检查器（Borrow Checker）**和**生命周期（Lifetime）**在编译时消除数据竞争、空指针解引用等常见错误。[^1]

Rust的设计目标：
- **内存安全**：编译时消除悬空指针、缓冲区溢出
- **并发安全**：无数据竞争的线程通信
- **零成本抽象**：高性能，可直接替代C++
- **实用主义**：成熟的工具链（cargo）和生态系统

## 所有权系统

所有权是Rust最核心的概念，每个值有**且只有一个所有者**，当所有者离开作用域时，值被自动释放。

### 三条核心规则

1. Rust中每个值都有一个**所有者**
2. 同一时刻只能有**一个所有者**
3. 当所有者离开作用域时，值被**Dropped（释放）**

### 移动语义

```rust
fn main() {
    // String在堆上分配，所有权从s1转移到s2
    let s1 = String::from("hello");
    let s2 = s1;  // s1的所有权移动到s2
    
    // println!("{}", s1);  // 编译错误：s1已无效
    println!("{}", s2);  // 正确：s2拥有所有权
}
```

对比C++的移动语义：

```cpp
// C++移动语义（类似但不完全相同）
std::string s1 = "hello";
std::string s2 = std::move(s1);  // s1变为空壳
// std::cout << s1 << std::endl; // 未定义行为！
std::cout << s2 << std::endl;    // "hello"
```

### Copy trait

对于栈上分配的简单类型（如i32），默认实现`Copy`trait，赋值时直接复制：

```rust
fn main() {
    let x = 42;
    let y = x;  // i32实现了Copy，x和y都有效
    
    println!("x = {}, y = {}", x, y);  // 正确
}
```

**实现了Copy的类型**：
- 所有整数类型（i32、u64等）
- 浮点数类型（f32、f64）
- 布尔类型（bool）
- 字符类型（char）
- 元组（仅当所有元素都实现Copy）

### 克隆（Clone）

对于需要深拷贝的数据，手动调用`clone()`：

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();  // 显式深拷贝
    
    println!("s1 = {}, s2 = {}", s1, s2);  // 两者都有效
}
```

## 借用与引用

借用允许**临时使用值而不获取所有权**，类似C++的引用，但有更严格的规则。

### 不可变借用（Immutable Borrow）

```rust
fn calculate_length(s: &String) -> usize {
    s.len()
}  // s离开作用域，但不释放String，因为只是借用

fn main() {
    let s1 = String::from("hello");
    let len = calculate_length(&s1);  // &表示借用
    
    println!("'{}'的长度是{}", s1, len);  // s1仍然有效
}
```

### 可变借用（Mutable Borrow）

当需要修改数据时，使用可变引用：

```rust
fn append_world(s: &mut String) {
    s.push_str(", world");
}

fn main() {
    let mut s = String::from("hello");
    append_world(&mut s);
    println!("{}", s);  // "hello, world"
}
```

**可变借用规则**：
- **同一作用域**内，只能有**一个可变引用**，或**多个不可变引用**，不能同时存在
- 避免数据竞争（data race）

```rust
fn main() {
    let mut s = String::from("hello");
    
    // ❌ 编译错误：同时存在可变和不可变引用
    let r1 = &s;
    let r2 = &mut s;  // 错误！
    println!("{} and {}", r1, r2);
    
    // ✅ 正确：按顺序使用
    let r1 = &s;
    let r2 = &s;
    println!("{} and {}", r1, r2);  // r1、r2使用完毕
    
    let r3 = &mut s;  // 现在可以了
    r3.push('!');
}
```

### 与C++引用对比

| C++ | Rust | 区别 |
|-----|------|------|
| `T&` | `&T` | 不可变借用 |
| `T&`（const） | `&mut T` | Rust可精确区分 |
| `T*` | 不推荐使用 | Rust的借用更安全 |

## 生命周期

生命周期是Rust防止**悬空引用**的机制，编译器确保所有引用始终有效。

### 生命周期标注

```rust
// 编译器无法推断返回引用的生命周期，需要标注
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

`'a`是一个生命周期参数，表示：
- 返回值引用的生命周期与输入引用中**较短的那个**相同
- 防止返回指向已释放内存的引用

### 生命周期省略规则

以下情况Rust自动推断生命周期，无需手动标注：

1. 每个**输入引用**获得自己的生命周期
2. 如果只有一个输入引用，且有输出引用，其生命周期赋给输出引用
3. 如果有`&self`或`&mut self`方法，所有生命周期赋给输出引用

### 结构体中的生命周期

包含引用的结构体必须标注生命周期：

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,  // 必须标注
}

fn main() {
    let novel = String::from("Call me Ishmael...");
    let first_sentence = novel.split('.').next().unwrap();
    
    let excerpt = ImportantExcerpt {
        part: first_sentence,
    };
}
```

### 生命周期推导示例

```rust
// C++ 版本（危险！）
struct Person {
    std::string* address;  // 指针可能指向已释放的内存
};

Person* findYounger(Person* p1, Person* p2) {
    return p1->age < p2->age ? p1 : p2;  // 可能返回悬空指针
}

// Rust 版本（安全）
struct Person {
    name: String,
    age: u32,
}

fn find_younger<'a>(p1: &'a Person, p2: &'a Person) -> &'a Person {
    if p1.age < p2.age { p1 } else { p2 }
}
```

## 所有权与C++ RAII对比

Rust的所有权系统可以理解为**无处不在的RAII**。[^2]

| C++ | Rust | 行为 |
|-----|------|------|
| `new T` / `delete T` | `Box::new(T)` | 手动管理（不推荐） |
| `std::unique_ptr<T>` | `Box<T>` | 独占所有权 |
| `std::shared_ptr<T>` | `Rc<T>` / `Arc<T>` | 引用计数（单线程/多线程） |
| `std::weak_ptr<T>` | `Weak<T>` | 弱引用避免循环 |
| 裸指针`T*` | `&T` / `&mut T` | 安全借用 |

```rust
use std::sync::Arc;
use std::thread;

// 多线程共享数据
fn main() {
    let data = Arc::new(vec![1, 2, 3]);
    
    let data_clone = Arc::clone(&data);
    let handle = thread::spawn(move || {
        println!("线程中: {:?}", data_clone);
    });
    
    println!("主线程: {:?}", data);
    handle.join().unwrap();
}
```

## 常见所有权模式

### 1. 转移所有权

```rust
fn consume(s: String) {
    println!("消费: {}", s);
}

fn main() {
    let s = String::from("hello");
    consume(s);  // 所有权转移
    // println!("{}", s);  // 编译错误
}
```

### 2. 借用使用

```rust
fn print_length(s: &String) {
    println!("长度: {}", s.len());
}

fn main() {
    let s = String::from("hello");
    print_length(&s);  // 借用，不转移所有权
    println!("{}", s);  // 仍然有效
}
```

### 3. 复合类型的所有权

```rust
struct Config {
    timeout: u32,
    debug: bool,
}

fn update_config(cfg: &mut Config) {
    cfg.timeout = 300;
}

fn main() {
    let mut cfg = Config { timeout: 100, debug: false };
    update_config(&mut cfg);
    println!("timeout: {}", cfg.timeout);
}
```

## 借用检查器实战技巧

### 避免常见错误

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    
    // ❌ 迭代过程中不能修改集合
    for i in &v {
        v.push(*i + 1);  // 编译错误！
    }
    
    // ✅ 方案1：先收集要添加的元素
    let mut to_add = Vec::new();
    for i in &v {
        to_add.push(*i + 1);
    }
    v.extend(to_add);
    
    // ✅ 方案2：使用索引
    let len = v.len();
    for i in 0..len {
        v.push(v[i] + 1);
    }
}
```

### 结构体字段的独立借用

```rust
struct Container {
    a: i32,
    b: i32,
}

fn main() {
    let mut c = Container { a: 1, b: 2 };
    
    // ✅ 可以同时借用不同字段
    let r_a = &mut c.a;
    let r_b = &mut c.b;
    *r_a += 10;
    *r_b += 20;
    println!("a={}, b={}", c.a, c.b);
}
```

## 错误处理与Option

Rust没有null，使用`Option<T>`表示可能不存在的值：

```rust
fn find_user(id: u32) -> Option<&'static str> {
    if id == 1 {
        Some("Alice")
    } else {
        None
    }
}

fn main() {
    match find_user(1) {
        Some(name) => println!("找到用户: {}", name),
        None => println!("用户不存在"),
    }
    
    // 使用if let简化
    if let Some(name) = find_user(2) {
        println!("{}", name);
    } else {
        println!("不存在");
    }
}
```

## 总结

Rust的所有权系统是革命性的设计：
- **编译时安全**：消除数据竞争、悬空指针等
- **零运行时开销**：无需GC，无需引用计数（除非显式使用）
- **学习曲线陡峭**：但一旦掌握，代码质量显著提升

对于C++开发者，关键是理解：
- 所有权 ≈ 严格的RAII
- 借用 ≈ 更安全的引用
- 生命周期 ≈ 编译时保证的引用有效性

[^1]: 本段参考[Rust for C++ Developers: Complete Migration Guide](https://reintech.io/blog/rust-for-cpp-developers-migration-guide)
[^2]: 本段参考[Rust ownership and C++ RAII comparison](https://github.com/microsoft/RustTraining)
