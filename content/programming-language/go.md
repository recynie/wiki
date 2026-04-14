---
title: Go Programming Language
date: 2026-04-14
description: Go 语言基础与核心特性
tags:
  - go
  - programming-language
draft: true
permalink:
---

# Go Programming Language

## 概述

Go（又称 Golang）是 Google 于 2009 年发布的开源编程语言，以简洁、高效、并发支持著称。广泛应用于云原生、微服务、DevOps 工具开发。

## 核心特性

### 设计哲学

- **简洁**：关键字少，语法清晰，降低学习曲线
- **高效**：编译为机器码，性能接近 C
- **并发**：原生支持 goroutine 和 channel
- **跨平台**：支持所有主流操作系统和架构

### 语言结构

```go
package main

import (
    "fmt"
    "context"
)

func main() {
    fmt.Println("Hello, Go!")
}
```

## 基础语法

### 变量与数据类型

```go
// 声明与初始化
var name string = "Go"           // 显式类型声明
age := 14                         // 类型推断
var (
    version string = "1.21"
    built bool = true
)

// 基本类型
var i int = 42           // 整数
var f float64 = 3.14     // 浮点数
var b bool = true        // 布尔值
var s string = "hello"   // 字符串
var bytes []byte         // 切片（动态数组）
var m map[string]int    // 映射（哈希表）
var p *int               // 指针
var c chan int           // 通道
var r interface{}        // 空接口
```

### 控制流

```go
// 条件判断
if x > 0 {
    fmt.Println("positive")
} else if x < 0 {
    fmt.Println("negative")
} else {
    fmt.Println("zero")
}

// 循环（Go 只有 for 循环）
for i := 0; i < 10; i++ {
    fmt.Println(i)
}

// range 遍历
nums := []int{1, 2, 3}
for i, v := range nums {
    fmt.Printf("index: %d, value: %d\n", i, v)
}

// switch
switch day := "Monday"; day {
case "Monday":
    fmt.Println("Start of work week")
case "Friday":
    fmt.Println("End of work week")
default:
    fmt.Println("Regular day")
}
```

### 函数

```go
// 多返回值
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil
}

// 命名返回值
func split(sum int) (x, y int) {
    x = sum * 4 / 9
    y = sum - x
    return // 裸 return
}

// 变长参数
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

// 函数作为值
add := func(a, b int) int {
    return a + b
}
```

## 复合类型

### 结构体

```go
type Person struct {
    Name    string
    Age     int
    Address Address  // 嵌套结构体
}

type Address struct {
    City    string
    Country string
}

// 方法
func (p Person) Greet() string {
    return fmt.Sprintf("Hello, I'm %s", p.Name)
}

// 值接收者 vs 指针接收者
func (p *Person) SetAge(age int) {  // 指针接收者可以修改原值
    p.Age = age
}
```

### 接口

```go
// 接口定义
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// 组合接口
type ReadWriter interface {
    Reader
    Writer
}

// 空接口（类似泛型）
func PrintAny(v interface{}) {
    fmt.Println(v)
}

// 类型断言
var i interface{} = "hello"
s, ok := i.(string)  // 安全断言
```

### 切片与映射

```go
// 切片
nums := []int{1, 2, 3, 4, 5}
slice := nums[1:3]        // [2, 3]，左闭右开
slice = append(slice, 6) // 自动扩容

// 映射
ages := map[string]int{
    "Alice": 30,
    "Bob":   25,
}
delete(ages, "Bob")
val, exists := ages["Alice"] // 检查键存在
```

## 并发编程

### Goroutine

```go
// 启动 goroutine
go func() {
    fmt.Println("Running in goroutine")
}()

// 带名字的函数
func worker(id int) {
    fmt.Printf("Worker %d started\n", id)
}
go worker(1)
```

### Channel

```go
// 创建通道
ch := make(chan int)
buffered := make(chan int, 10) // 带缓冲通道

// 发送与接收
ch <- 42        // 发送
value := <-ch  // 接收

// 关闭通道
close(ch)

// Select（多路复用）
select {
case msg := <-ch1:
    fmt.Println("Received from ch1:", msg)
case ch2 <- data:
    fmt.Println("Sent to ch2")
case <-time.After(time.Second):
    fmt.Println("Timeout")
}
```

### 并发模式

```go
// Worker Pool
func workerPool(numWorkers int, jobs <-chan Job, results chan<- Result) {
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }()
    }
    wg.Wait()
    close(results)
}

// Context 取消
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

select {
case <-ctx.Done():
    fmt.Println("Context cancelled:", ctx.Err())
default:
    // 正常工作
}
```

## 错误处理

```go
// Go 没有异常机制，使用错误值
result, err := doSomething()
if err != nil {
    return fmt.Errorf("doSomething failed: %w", err)
}

// 错误检查
if err != nil {
    if errors.Is(err, syscall.ENOENT) {
        // 文件不存在
    }
}

// 自定义错误
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}
```

## 依赖管理

```bash
# 初始化模块
go mod init github.com/user/project

# 添加依赖
go get github.com/pkg/errors

# 整理依赖
go mod tidy

# 下载依赖
go mod download
```

## 标准库常用包

| 包 | 用途 |
|----|------|
| fmt | 格式化 I/O |
| os, io, bufio | 操作系统交互 |
| flag | 命令行参数解析 |
| context | 上下文传递与取消 |
| sync | 同步原语（Mutex, WaitGroup） |
| time | 时间与定时器 |
| encoding/json | JSON 序列化 |
| net/http | HTTP 客户端/服务端 |
| log | 日志记录 |

## 惯用法

```go
// 短变量声明（函数内）
msg := "hello"

// defer 延迟执行
file, _ := os.Open("data.txt")
defer file.Close()

// 错误优先
if err != nil {
    return err
}

// 组合优于继承（通过接口）
type Animal struct{}
func (a Animal) Speak() { fmt.Println("...") }

type Dog struct {
    Animal  // 嵌入
}

// 避免全局变量，使用依赖注入
```

## 扩展阅读

- [Go 官方文档](https://go.dev/doc/)
- [Go 语言之旅](https://go.dev/tour/)
- [Go 语言设计与实现](https://github.com/changkun/go-modem-body)
- [Uber Go 风格指南](https://github.com/uber-go/guide)

