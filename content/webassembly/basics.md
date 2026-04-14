---
title: WebAssembly入门
date: 2026-04-11
description: WebAssembly（WASM）技术详解，涵盖边缘计算、插件系统、浏览器应用等实战场景
tags:
  - webassembly
  - wasm
  - wasi
  - edge-computing
  - sandbox
draft: false
permalink:
---

## 概述

WebAssembly（Wasm）是一种紧凑的二进制指令格式，设计为可在浏览器中以接近原生的性能执行。它也是一种通用的运行时目标，不局限于JavaScript生态系统。[^1]

### 核心特性

- **高效执行**：二进制格式，解析速度比JavaScript快
- **沙盒安全**：在独立内存空间中运行，天然隔离
- **跨平台**：同一二进制可在不同架构和操作系统运行
- **语言无关**：支持C/C++、Rust、Go、AssemblyScript等

### 演进历程

```
2015  Wasm首次演示
2017  MVP版本发布
2019  Wasm Core Spec成为W3C标准
2022  WASI Preview 1（Node.js支持）
2024  WASI Preview 2（Component Model）
2026  WASM生产级应用广泛落地
```

## 核心概念

### 模块结构

```
┌─────────────────────────┐
│  WebAssembly Module     │
├─────────────────────────┤
│  Type Section          │  类型定义
│  Import Section        │  导入函数
│  Function Section      │  函数定义
│  Table Section         │  间接调用表
│  Memory Section        │  内存布局
│  Export Section        │  导出接口
│  Code Section          │  实际字节码
└─────────────────────────┘
```

### 内存模型

```c
// C/C++编译到Wasm
#include <stdio.h>

int main() {
    // Wasm线性内存：初始4KB，可扩展
    int arr[10] = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    int sum = 0;
    for (int i = 0; i < 10; i++) {
        sum += arr[i];
    }
    printf("Sum: %d\n", sum);
    return 0;
}
```

## 应用场景

### 1. 边缘计算（主流场景）[^2]

WASM的杀手级场景不是浏览器，而是边缘计算：

| 运行时 | 语言支持 | 冷启动时间 | 适用场景 |
|-------|---------|-----------|---------|
| Cloudflare Workers | Rust/Go/C++/JS | <1ms | 全球CDN边缘 |
| Fastly Compute | Rust/C/Go | <1ms | 边缘计算 |
| Fermyon Spin | Rust/Go/JS | <1ms | Serverless |
| wasmCloud | 任意语言 | <1ms | 分布式应用 |

**实际案例**：

```rust
// Cloudflare Workers WASM示例
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn processRequest(req: &str) -> String {
    // 在边缘节点执行
    format!("Processed: {}", req)
}
```

**性能数据**：
- Cloudflare Workers每日处理约40亿次WASM调用
- 冷启动时间<1ms（vs Lambda@Edge的100ms+）

### 2. 插件系统（安全沙盒）

取代危险的`eval()`或进程隔离：

| 方案 | 隔离性 | 性能开销 | 复杂度 |
|-----|-------|---------|-------|
| Subprocess | 高 | 高（IPC延迟） | 中 |
| Native Plugin | 无 | 无 | 低 |
| **WASM Plugin** | **高** | **低** | **中** |

**Grafana案例**：2025年迁移到WASM插件系统[^2]

```rust
// Grafana WASM数据源插件示例
use wasm_plugin_sdk::*;

struct MyDataSource {
    base_url: String,
}

impl DataSource for MyDataSource {
    fn query(&self, request: QueryRequest) -> Result<Response, Error> {
        // 安全执行用户提供的查询逻辑
        // 即使插件有漏洞也不会影响主机
    }
}
```

**Shopify Oxygen**：商户自定义结算逻辑

```rust
// 安全的商家代码执行
pub fn execute_merchant_logic(
    logic: &[u8],  // WASM字节码
    context: &CheckoutContext
) -> Result<RenderedTemplate, Error> {
    let instance = WasmInstance::new(logic)?;
    instance.invoke("checkout", context)
}
```

### 3. 浏览器端计算密集型任务[^3]

| 应用 | 案例 | 性能提升 |
|-----|------|---------|
| Figma | 专业级渲染引擎 | 接近原生性能 |
| Google Earth | 3D地形渲染 | 首次加载<3s |
| AutoCAD Web | 浏览器CAD | 无需插件 |
| DaVinci Resolve | 视频预览 | 流畅播放 |

**Notion案例**：WASM SQLite提升页面导航速度20%

```javascript
// Notion的WASM SQLite架构
const wasmModule = await WebAssembly.compileStreaming(
  fetch('/sqlite.wasm')
);

// 使用OPFS持久化
const accessHandle = await opfsDir.createSyncAccessHandle();
const db = new SQLiteDatabase(wasmModule, accessHandle);
```

## WASI（WebAssembly System Interface）

WASI是Wasm的模块化系统接口标准，使Wasm可在浏览器外运行：

### 核心模块

```rust
// WASI标准库引入
wasi::stdio::println!("Hello from WASI!");
wasi::fs::read_file("config.json")?;
wasi::random::getrandom_bytes(32)?;
```

### Component Model（组件模型）

WASI Preview 2引入的组件模型：

```
┌────────────────────────────────────────┐
│           Component Model               │
├────────────────────────────────────────┤
│  WIT Interface (接口定义语言)            │
│  ┌─────────────────────────────────┐   │
│  │ interface database {             │   │
│  │   query(sql: string) -> ResultSet│   │
│  │ }                               │   │
│  └─────────────────────────────────┘   │
│                                         │
│  组件可组合：组件A + 组件B = 新组件       │
└────────────────────────────────────────┘
```

## 实战开发

### Rust编译到Wasm

```rust
// src/lib.rs
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn fibonacci(n: u32) -> u32 {
    match n {
        0 => 0,
        1 => 1,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}
```

```toml
# Cargo.toml
[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
```

```bash
# 编译
cargo build --target wasm32-wasip1 --release
wasm-pack build --target web
```

### AssemblyScript（TypeScript到Wasm）

```typescript
// index.ts
export function add(a: i32, b: i32): i32 {
  return a + b;
}

export function sumArray(arr: i32[]): i32 {
  let result = 0;
  for (let i = 0; i < arr.length; i++) {
    result += arr[i];
  }
  return result;
}
```

```bash
# 编译
npm install -g assemblyscript
asc index.ts --target release
```

### JavaScript加载Wasm

```javascript
// Web方式
const response = await fetch('/module.wasm');
const bytes = await response.arrayBuffer();
const { instance } = await WebAssembly.instantiate(bytes, {
  env: {
    // 导入函数
    log: (ptr, len) => console.log(readString(ptr, len))
  }
});

instance.exports.fibonacci(10);
```

## 适用性判断

### 适合WASM的场景

✅ 边缘函数（Cloudflare Workers等）  
✅ 插件系统（用户代码执行）  
✅ 计算密集型（图像处理、加解密）  
✅ 跨平台部署（一次编译，到处运行）  
✅ 沙盒隔离需求

### 不适合的场景

❌ 通用服务器端运行时（Docker/Kubernetes更适合）  
❌ 复杂依赖管理（与npm/PyPI生态竞争）  
❌ 需要大量syscall的应用  
❌ 实时性能要求极高的场景（直接原生代码）

## 参考资料

[^1]: [WebAssembly官网](https://webassembly.org/)
[^2]: [WebAssembly in Production 2026 - iBuidl](https://ibuidl.org/blog/webassembly-production-use-cases-2026-20260310)
[^3]: [WebAssembly Use Cases - official docs](https://webassembly.org/docs/use-cases/)
