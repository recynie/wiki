---
title: WebGPU Rendering Basics
date: 2026-04-16
description: WebGPU渲染基础，涵盖初始化流程、渲染管线构建、WGSL着色器和实战三角形渲染
tags:
  - webgpu
  - rendering
  - graphics
  - wgsl
  - pipeline
draft: false
permalink:
---

## 1. 初始化流程

WebGPU初始化需要获取GPU适配器并创建逻辑设备：

```javascript
async function initWebGPU() {
  // 1. 检查支持
  if (!navigator.gpu) {
    throw Error("WebGPU not supported.");
  }

  // 2. 请求适配器（物理GPU）
  const adapter = await navigator.gpu.requestAdapter();
  if (!adapter) {
    throw Error("Couldn't request WebGPU adapter.");
  }

  // 3. 请求设备（逻辑GPU）
  const device = await adapter.requestDevice();

  // 4. 获取Canvas上下文
  const canvas = document.getElementById('canvas');
  const context = canvas.getContext('webgpu');
  
  // 5. 配置渲染格式
  const format = navigator.gpu.getPreferredCanvasFormat();
  context.configure({
    device: device,
    format: format,
    alphaMode: 'premultiplied',
  });

  return { device, context, format };
}
```

### 适配器选项

```javascript
// 请求特定类型的GPU
const adapter = await navigator.gpu.requestAdapter({
  powerPreference: 'high-performance',  // 独显
  // powerPreference: 'low-power'       // 集显
});
```

## 2. Canvas与渲染上下文

### 配置参数

```javascript
const context = canvas.getContext('webgpu');

context.configure({
  device: device,
  format: navigator.gpu.getPreferredCanvasFormat(),
  alphaMode: 'premultiplied',  // 或 'opaque'
  compositingAlphaMode: 'opaque',
  usage: GPUTextureUsage.RENDER_ATTACHMENT,
});
```

### 常用纹理用途

| 用途 | 说明 |
|------|------|
| `RENDER_ATTACHMENT` | 颜色附件（屏幕输出） |
| `STORAGE_BINDING` | 存储绑定（计算着色器） |
| `COPY_SRC` | 复制源 |
| `COPY_DST` | 复制目标 |
| `TEXTURE_BINDING` | 纹理绑定 |

## 3. 渲染管线构建

### 管线结构

```
┌─────────────────────────────────────────────────────────┐
│                   Render Pipeline                        │
├─────────────────────────────────────────────────────────┤
│  Shader Module (WGSL代码)                                │
│                                                         │
│  ┌─────────────┐    ┌─────────────┐                   │
│  │ Vertex Stage│ →  │Fragment Stage│                   │
│  │             │    │              │                   │
│  │ @vertex     │    │ @fragment    │                   │
│  └─────────────┘    └─────────────┘                   │
│                                                         │
│  Pipeline Descriptor                                      │
│  ├── vertex: { module, entry }                          │
│  ├── fragment: { module, entry, targets }               │
│  ├── primitive: { topology, stripIndexFormat }          │
│  └── layout: 'auto' | pipelineLayout                   │
└─────────────────────────────────────────────────────────┘
```

### 创建渲染管线

```javascript
const pipeline = device.createRenderPipeline({
  layout: 'auto',
  vertex: {
    module: device.createShaderModule({
      code: `
        @vertex
        fn vertexMain(
          @builtin(vertex_index) vertexIndex: u32
        ) -> @builtin(position) vec4f {
          let positions = array<vec2f, 3>(
            vec2f(0.0, 0.5),
            vec2f(-0.5, -0.5),
            vec2f(0.5, -0.5)
          );
          return vec4f(positions[vertexIndex], 0.0, 1.0);
        }
      `,
    }),
    entryPoint: 'vertexMain',
  },
  fragment: {
    module: device.createShaderModule({
      code: `
        @fragment
        fn fragmentMain() -> @location(0) vec4f {
          return vec4f(1.0, 0.0, 0.0, 1.0);
        }
      `,
    }),
    entryPoint: 'fragmentMain',
    targets: [{ format: navigator.gpu.getPreferredCanvasFormat() }],
  },
  primitive: {
    topology: 'triangle-list',
  },
});
```

## 4. 顶点着色器

### WGSL顶点着色器基础

```wgsl
@vertex
fn vertexMain(
  @builtin(vertex_index) vertexIndex: u32
) -> @builtin(position) vec4f {
  // 顶点位置数组
  let positions = array<vec2f, 3>(
    vec2f(0.0, 0.5),    // 顶点0
    vec2f(-0.5, -0.5),  // 顶点1
    vec2f(0.5, -0.5)    // 顶点2
  );
  
  // 返回裁剪空间坐标 (x, y, z, w)
  return vec4f(positions[vertexIndex], 0.0, 1.0);
}
```

### 使用顶点缓冲

```javascript
// 创建顶点缓冲
const vertexBuffer = device.createBuffer({
  size: vertices.byteLength,
  usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST,
});

// 复制数据到GPU
device.queue.writeBuffer(vertexBuffer, 0, vertices);

// WGSL中接收顶点数据
// shader:
// @vertex
// fn vertexMain(
//   @location(0) position: vec3f,
//   @location(1) color: vec3f,
// ) -> VertexOutput { ... }
```

### 带属性的顶点着色器

```javascript
// 顶点数据：位置(3 floats) + 颜色(3 floats)
const vertexBufferLayout = {
  arrayStride: 6 * 4,  // 24 bytes per vertex
  attributes: [
    { format: 'float32x3', offset: 0, shaderLocation: 0 },    // position
    { format: 'float32x3', offset: 12, shaderLocation: 1 }, // color
  ],
};
```

```wgsl
struct VertexInput {
  @location(0) position: vec3f,
  @location(1) color: vec3f,
}

struct VertexOutput {
  @builtin(position) position: vec4f,
  @location(0) color: vec3f,
}

@vertex
fn vertexMain(input: VertexInput) -> VertexOutput {
  var output: VertexOutput;
  output.position = vec4f(input.position, 1.0);
  output.color = input.color;
  return output;
}
```

## 5. 片元着色器

### WGSL片元着色器基础

```wgsl
@fragment
fn fragmentMain() -> @location(0) vec4f {
  // 返回红色 (R, G, B, A)
  return vec4f(1.0, 0.0, 0.0, 1.0);
}
```

### 带坐标的片元着色器

```wgsl
@fragment
fn fragmentMain(
  @builtin(position) position: vec4f
) -> @location(0) vec4f {
  // 基于像素位置计算颜色
  let normalized = position.xy / vec2f(800.0, 600.0);
  return vec4f(normalized.x, normalized.y, 0.5, 1.0);
}
```

### 多输出目标

```javascript
fragment: {
  targets: [
    { format: 'bgra8unorm', writeMask: GPUColorWrite.ALL },
    { format: 'rgba16float', writeMask: GPUColorWrite.ALL }, // MRT
  ],
}
```

```wgsl
struct FragmentOutput {
  @location(0) color0: vec4f,
  @location(1) color1: vec4f,
}

@fragment
fn fragmentMain() -> FragmentOutput {
  var output: FragmentOutput;
  output.color0 = vec4f(1.0, 0.0, 0.0, 1.0);
  output.color1 = vec4f(0.0, 1.0, 0.0, 1.0);
  return output;
}
```

## 6. 命令编码与提交

### 完整的渲染循环

```javascript
async function render() {
  const { device, context, format } = await initWebGPU();
  
  // 创建着色器模块和管线（通常只创建一次）
  const pipeline = createRenderPipeline(device, format);
  
  // 渲染循环
  function frame() {
    // 1. 获取当前纹理（即将显示的）
    const textureView = context.getCurrentTexture().createView();
    
    // 2. 创建命令编码器
    const commandEncoder = device.createCommandEncoder();
    
    // 3. 开始渲染通道
    const renderPass = commandEncoder.beginRenderPass({
      colorAttachments: [{
        view: textureView,
        clearValue: { r: 0.0, g: 0.0, b: 0.0, a: 1.0 },
        loadOp: 'clear',
        storeOp: 'store',
      }],
    });
    
    // 4. 设置管线
    renderPass.setPipeline(pipeline);
    
    // 5. 绑定顶点缓冲（如有）
    // renderPass.setVertexBuffer(0, vertexBuffer);
    
    // 6. 绘制
    renderPass.draw(3);  // 3个顶点 = 1个三角形
    
    // 7. 结束渲染通道
    renderPass.end();
    
    // 8. 提交命令
    device.queue.submit([commandEncoder.finish()]);
    
    // 9. 请求下一帧
    requestAnimationFrame(frame);
  }
  
  requestAnimationFrame(frame);
}
```

### 渲染通道参数

| 参数 | 说明 |
|------|------|
| `colorAttachments` | 颜色附件配置数组 |
| `view` | 渲染目标纹理视图 |
| `clearValue` | 清除颜色 |
| `loadOp` | `'load'` 或 `'clear'` |
| `storeOp` | `'store'` 或 `'discard'` |
| `depthStencilAttachment` | 深度模板附件（可选） |

## 7. 实战：渲染彩色三角形

### 完整示例

```javascript
const shaderCode = `
  struct VertexInput {
    @location(0) position: vec3f,
    @location(1) color: vec3f,
  }

  struct VertexOutput {
    @builtin(position) position: vec4f,
    @location(0) color: vec3f,
  }

  @vertex
  fn vertexMain(input: VertexInput) -> VertexOutput {
    var output: VertexOutput;
    output.position = vec4f(input.position, 1.0);
    output.color = input.color;
    return output;
  }

  @fragment
  fn fragmentMain(input: VertexOutput) -> @location(0) vec4f {
    return vec4f(input.color, 1.0);
  }
`;

// 顶点数据：位置(x,y,z) + 颜色(r,g,b)
const vertices = new Float32Array([
  // 顶点0: 顶部，红色
  0.0, 0.5, 0.0,   1.0, 0.0, 0.0,
  // 顶点1: 左下，绿色
  -0.5, -0.5, 0.0,  0.0, 1.0, 0.0,
  // 顶点2: 右下，蓝色
  0.5, -0.5, 0.0,   0.0, 0.0, 1.0,
]);

// 创建顶点缓冲
const vertexBuffer = device.createBuffer({
  size: vertices.byteLength,
  usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST,
});
device.queue.writeBuffer(vertexBuffer, 0, vertices);

// 创建渲染管线
const pipeline = device.createRenderPipeline({
  layout: 'auto',
  vertex: {
    module: device.createShaderModule({ code: shaderCode }),
    entryPoint: 'vertexMain',
    buffers: [{
      arrayStride: 24,  // 6 floats * 4 bytes
      attributes: [
        { shaderLocation: 0, offset: 0, format: 'float32x3' },
        { shaderLocation: 1, offset: 12, format: 'float32x3' },
      ],
    }],
  },
  fragment: {
    module: device.createShaderModule({ code: shaderCode }),
    entryPoint: 'fragmentMain',
    targets: [{ format: format }],
  },
  primitive: { topology: 'triangle-list' },
});

// 在渲染循环中绘制
renderPass.setVertexBuffer(0, vertexBuffer);
renderPass.draw(3);
```

## 8. 常见问题

### 为什么三角形不显示？

1. 检查顶点坐标是否在NDC范围内（-1到1）
2. 确认 `setVertexBuffer` 正确调用
3. 验证 `fragment.targets` 格式匹配

### 纹理格式不匹配

```javascript
// 获取浏览器偏好的格式
const format = navigator.gpu.getPreferredCanvasFormat();

// 确保fragment target使用相同格式
fragment: {
  targets: [{ format: format }],
}
```

### 性能优化

- **避免每帧创建Pipeline**：Pipeline创建昂贵，应缓存
- **批量提交命令**：减少 `queue.submit` 调用次数
- **使用Uniform缓冲**：存储频繁变化的着色器参数

## 参考资料

- [MDN WebGPU API - Rendering](https://developer.mozilla.org/en-US/docs/Web/API/WebGPU_API)
- [WebGPU Fundamentals](https://webgpufundamentals.org/webgpu/lessons/webgpu-fundamentals.html)
- [Chrome WebGPU Samples](https://chromeos.dev/en/gpu/webgpu)
