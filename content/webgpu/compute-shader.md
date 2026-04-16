---
title: WebGPU Compute Shader
date: 2026-04-16
description: WebGPU计算着色器详解，涵盖WGSL语法、Workgroup模型、Buffer管理和实战示例
tags:
  - webgpu
  - compute-shader
  - gpu
  - parallel
  - wgsl
draft: false
permalink:
---

## 1. 计算着色器概述

计算着色器（Compute Shader）是一种在GPU上执行通用并行计算的着色器类型。与渲染管线不同，计算着色器不绘制图形，而是利用GPU的大规模并行能力处理数据。[^1]

### 计算着色器 vs 渲染管线

| 特性 | 计算着色器 | 渲染管线 |
|------|------------|----------|
| 用途 | 通用计算 | 图形渲染 |
| 输入 | Buffer/Texture | 顶点/纹理 |
| 输出 | Buffer/Texture | 帧缓冲 |
| 线程模型 | workgroup | 顶点/片元 |
| 编程模型 | CSP/数据并行 | 图形管线 |

### 应用场景

- **科学计算**：矩阵运算、线性代数
- **图像处理**：卷积、滤镜、傅里叶变换
- **物理模拟**：粒子系统、流体动力学
- **机器学习**：张量运算、神经网络推理

## 2. WGSL计算着色器基础

### 基本语法

```wgsl
@compute @workgroup_size(64)
fn computeMain(
  @builtin(global_invocation_id) global_id: vec3<u32>
) {
  let index = global_id.x;
  // 计算逻辑
}
```

### 关键装饰器

| 装饰器 | 说明 |
|--------|------|
| `@compute` | 标记为计算着色器入口 |
| `@workgroup_size(N)` | 每个workgroup的线程数 |
| `@builtin(global_invocation_id)` | 全局唯一线程ID |
| `@builtin(workgroup_id)` | workgroup在dispatch中的位置 |
| `@builtin(local_invocation_id)` | workgroup内的线程位置 |

## 3. Workgroup与线程模型

### 概念层次

```
┌─────────────────────────────────────────────────────────┐
│              dispatchWorkgroups(4, 3, 2)                 │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │ Workgroup   │  │ Workgroup   │  │ Workgroup   │       │
│  │ (0,0,0)     │  │ (1,0,0)     │  │ (2,0,0)     │       │
│  │ ┌───┬───┐  │  │ ┌───┬───┐  │  │ ┌───┬───┐  │       │
│  │ │ T │ T │  │  │ │ T │ T │  │  │ │ T │ T │  │       │
│  │ ├───┼───┤  │  │ ├───┼───┤  │  │ ├───┼───┤  │       │
│  │ │ T │ T │  │  │ │ T │ T │  │  │ │ T │ T │  │       │
│  │ └───┴───┘  │  │ └───┴───┘  │  │ └───┴───┘  │       │
│  └─────────────┘  └─────────────┘  └─────────────┘       │
│        ...              ...              ...              │
└─────────────────────────────────────────────────────────┘

T = Thread (线程)
每个线程执行相同的计算逻辑
```

### 内置变量

```wgsl
@compute @workgroup_size(2, 4, 2)
fn main(
  @builtin(global_invocation_id) global_id: vec3<u32>,
  @builtin(workgroup_id) group_id: vec3<u32>,
  @builtin(local_invocation_id) local_id: vec3<u32>,
  @builtin(num_workgroups) num_groups: vec3<u32>
) {
  // global_id: 线程在所有workgroups中的全局索引 (0 ~ 4*3*2*8-1)
  // group_id: 当前workgroup在dispatch中的位置
  // local_id: 当前线程在workgroup内的位置
  // num_groups: dispatch的workgroup数量
}
```

### Workgroup大小限制

| 限制 | 典型值 |
|------|--------|
| `maxComputeInvocationsPerWorkgroup` | 256 |
| `maxComputeWorkgroupSizeX` | 256 |
| `maxComputeWorkgroupSizeY` | 256 |
| `maxComputeWorkgroupSizeZ` | 64 |

**最佳实践**：通常选择64或128作为workgroup大小，除非有特定原因。

## 4. Buffer管理

### Buffer创建

```javascript
// 创建只读输入缓冲
const inputBuffer = device.createBuffer({
  size: array.byteLength,
  usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST,
});

// 创建输出缓冲
const outputBuffer = device.createBuffer({
  size: outputSize,
  usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_SRC,
});

// 创建可映射缓冲（用于CPU读取结果）
const readBuffer = device.createBuffer({
  size: outputSize,
  usage: GPUBufferUsage.MAP_READ | GPUBufferUsage.COPY_DST,
});
```

### Buffer用途标志

| 标志 | 说明 |
|------|------|
| `MAP_READ` | 可映射读取（CPU→GPU） |
| `MAP_WRITE` | 可映射写入（GPU→CPU） |
| `COPY_SRC` | 作为复制源 |
| `COPY_DST` | 作为复制目标 |
| `VERTEX` | 顶点缓冲 |
| `UNIFORM` | Uniform缓冲 |
| `STORAGE` | 存储缓冲（计算着色器） |
| `INDEX` | 索引缓冲 |

### 数据传输

```javascript
// CPU → GPU
device.queue.writeBuffer(buffer, 0, dataArray);

// GPU → CPU (需要映射)
async function readBuffer(device, buffer, size) {
  // 1. 创建临时读取缓冲
  const readBuffer = device.createBuffer({
    size: size,
    usage: GPUBufferUsage.MAP_READ | GPUBufferUsage.COPY_DST,
  });
  
  // 2. 编码复制命令
  const commandEncoder = device.createCommandEncoder();
  commandEncoder.copyBufferToBuffer(
    buffer, 0,      // 源
    readBuffer, 0,  // 目标
    size
  );
  device.queue.submit([commandEncoder.finish()]);
  
  // 3. 映射并读取
  await readBuffer.mapAsync(GPUMapMode.READ);
  const data = new Float32Array(readBuffer.getMappedRange());
  
  // 4. 处理完成后取消映射
  readBuffer.unmap();
  
  return data;
}
```

## 5. Bind Group与资源绑定

### Bind Group Layout

```javascript
const bindGroupLayout = device.createBindGroupLayout({
  entries: [
    {
      binding: 0,
      visibility: GPUShaderStage.COMPUTE,
      buffer: { type: 'read-only-storage' },
    },
    {
      binding: 1,
      visibility: GPUShaderStage.COMPUTE,
      buffer: { type: 'read-only-storage' },
    },
    {
      binding: 2,
      visibility: GPUShaderStage.COMPUTE,
      buffer: { type: 'storage' },
    },
  ],
});
```

### Bind Group

```javascript
const bindGroup = device.createBindGroup({
  layout: bindGroupLayout,
  entries: [
    { binding: 0, resource: { buffer: inputBufferA } },
    { binding: 1, resource: { buffer: inputBufferB } },
    { binding: 2, resource: { buffer: outputBuffer } },
  ],
});
```

### WGSL中的绑定

```wgsl
@group(0) @binding(0) var<storage, read> inputA: array<f32>;
@group(0) @binding(1) var<storage, read> inputB: array<f32>;
@group(0) @binding(2) var<storage, read_write> output: array<f32>;

@compute @workgroup_size(64)
fn main(@builtin(global_invocation_id) id: vec3<u32>) {
  let index = id.x;
  output[index] = inputA[index] + inputB[index];
}
```

## 6. 计算管线

### 创建计算管线

```javascript
const computePipeline = device.createComputePipeline({
  layout: device.createPipelineLayout({
    bindGroupLayouts: [bindGroupLayout],
  }),
  compute: {
    module: device.createShaderModule({
      code: computeShaderCode,
    }),
    entryPoint: 'main',
  },
});
```

### 编码计算命令

```javascript
function dispatchCompute(device, pipeline, bindGroup, dataSize) {
  const commandEncoder = device.createCommandEncoder();
  
  const computePass = commandEncoder.beginComputePass();
  computePass.setPipeline(pipeline);
  computePass.setBindGroup(0, bindGroup);
  computePass.dispatchWorkgroups(
    Math.ceil(dataSize / 64),  // x
    1,  // y
    1   // z
  );
  computePass.end();
  
  device.queue.submit([commandEncoder.finish()]);
}
```

## 7. 实战：向量相加

### 完整示例

```javascript
// WGSL计算着色器
const shaderCode = `
  @group(0) @binding(0) var<storage, read> a: array<f32>;
  @group(0) @binding(1) var<storage, read> b: array<f32>;
  @group(0) @binding(2) var<storage, read_write> output: array<f32>;

  @compute @workgroup_size(64)
  fn main(@builtin(global_invocation_id) id: vec3<u32>) {
    let index = id.x;
    output[index] = a[index] + b[index];
  }
`;

async function vectorAddition(device, a, b) {
  const n = a.length;
  const bufferSize = n * 4;  // f32 = 4 bytes

  // 创建缓冲
  const aBuffer = device.createBuffer({
    size: bufferSize,
    usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST,
  });
  const bBuffer = device.createBuffer({
    size: bufferSize,
    usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST,
  });
  const outputBuffer = device.createBuffer({
    size: bufferSize,
    usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_SRC,
  });

  // 复制数据
  device.queue.writeBuffer(aBuffer, 0, a);
  device.queue.writeBuffer(bBuffer, 0, b);

  // 创建Bind Group
  const bindGroup = device.createBindGroup({
    layout: pipeline.getBindGroupLayout(0),
    entries: [
      { binding: 0, resource: { buffer: aBuffer } },
      { binding: 1, resource: { buffer: bBuffer } },
      { binding: 2, resource: { buffer: outputBuffer } },
    ],
  });

  // 执行计算
  const commandEncoder = device.createCommandEncoder();
  const pass = commandEncoder.beginComputePass();
  pass.setPipeline(pipeline);
  pass.setBindGroup(0, bindGroup);
  pass.dispatchWorkgroups(Math.ceil(n / 64));
  pass.end();
  device.queue.submit([commandEncoder.finish()]);

  // 读取结果
  return await readBuffer(device, outputBuffer, bufferSize);
}
```

## 8. 实战：矩阵乘法

### WGSL矩阵乘法

```wgsl
struct Matrix {
  size: vec2f,
}

@group(0) @binding(0) var<storage, read> a: Matrix;
@group(0) @binding(1) var<storage, read> aData: array<f32>,
@group(0) @binding(2) var<storage, read> b: Matrix,
@group(0) @binding(3) var<storage, read> bData: array<f32>,
@group(0) @binding(4) var<storage, read_write> c: Matrix,
@group(0) @binding(5) var<storage, read_write> cData: array<f32>,

@compute @workgroup_size(8, 8)
fn main(
  @builtin(global_invocation_id) id: vec3<u32>
) {
  let row = id.x;
  let col = id.y;
  
  if (row >= u32(a.size.x) || col >= u32(b.size.y)) {
    return;
  }
  
  var sum = 0.0;
  for (var k = 0u; k < u32(a.size.y); k = k + 1u) {
    let aVal = aData[row * u32(a.size.y) + k];
    let bVal = bData[k * u32(b.size.y) + col];
    sum = sum + aVal * bVal;
  }
  
  cData[row * u32(c.size.y) + col] = sum;
}
```

## 9. 性能优化

### Workgroup大小选择

```wgsl
// 一般建议：64
@workgroup_size(64)

// 适合2D计算
@workgroup_size(8, 8)

// 适合3D计算
@workgroup_size(4, 4, 4)
```

### 内存访问模式

- **合并访问**：相邻线程访问相邻内存
- **避免bank冲突**：shared memory访问模式
- **使用local memory**：高频访问数据放入workgroup本地内存

### race condition处理

```wgsl
// 错误：多个线程同时写入同一位置
output[index] = value;  // race condition!

// 正确：使用原子操作
storageBarrier();  // 确保所有写操作完成
workgroupBarrier();
```

## 10. 常见问题

### dispatch数量计算

```javascript
// 假设有1024个元素，workgroup大小为64
const workgroupSize = 64;
const numElements = 1024;
const numWorkgroups = Math.ceil(numElements / workgroupSize);

// dispatch
pass.dispatchWorkgroups(numWorkgroups);
```

### Buffer映射失败

```javascript
// 原因：Buffer正在被GPU使用时映射
await buffer.mapAsync(GPUMapMode.READ);
// 如果GPU仍在使用，会抛出错误

// 解决：确保GPU操作完成后再映射
device.queue.submit([commandEncoder.finish()]);
await device.queue.onSubmittedWorkDone();
await buffer.mapAsync(GPUMapMode.READ);
```

## 参考资料

[^1]: [Chrome for Developers - GPU Compute](https://developer.chrome.com/docs/capabilities/web-apis/gpu-compute)
[^2]: [WebGPU Fundamentals - Compute Shaders](https://webgpufundamentals.org/webgpu/lessons/webgpu-compute-shaders.html)
[^3]: [WGSL Specification](https://gpuweb.github.io/gpuweb/wgsl/)
