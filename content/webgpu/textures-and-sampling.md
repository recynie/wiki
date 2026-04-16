---
title: WebGPU Textures and Sampling
date: 2026-04-16
description: WebGPU纹理与采样详解，涵盖纹理创建、视图、采样器配置和实战3D纹理映射
tags:
  - webgpu
  - texture
  - sampling
  - graphics
  - wgsl
draft: false
permalink:
---

## 1. 纹理概述

纹理（Texture）是GPU用于存储图像数据的资源类型，可用于渲染和计算场景。

### 纹理 vs Buffer

| 特性 | Buffer | Texture |
|------|--------|---------|
| 维度 | 1D | 1D/2D/3D/Cube |
| 访问方式 | 随机访问 | 采样器访问 |
| 过滤 | 无 | 可配置（线性、最近等） |
| MIP Map | 不支持 | 支持 |
| 用途 | 通用数据 | 图像/表面数据 |

## 2. GPUTexture创建

### 基本创建

```javascript
const texture = device.createTexture({
  size: [width, height, depthOrArrayLayers],
  format: 'rgba8unorm',
  usage: GPUTextureUsage.TEXTURE_BINDING |
         GPUTextureUsage.COPY_DST |
         GPUTextureUsage.RENDER_ATTACHMENT,
});
```

### 常用格式

| 格式 | 说明 | 用途 |
|------|------|------|
| `rgba8unorm` | 8位RGBA，归一化 | 颜色纹理 |
| `bgra8unorm` | 8位BGRA，归一化 | Canvas首选 |
| `rgba16float` | 16位浮点 | HDR渲染 |
| `rgba32float` | 32位浮点 | 计算/高精度 |
| `depth24plus` | 24位深度 | 深度测试 |
| `stencil8` | 8位模板 | 模板缓冲 |

### 纹理尺寸

```javascript
// 2D纹理
const tex2d = device.createTexture({
  size: [width, height],
  format: 'rgba8unorm',
  usage: ...,
});

// 2D纹理数组
const tex2dArray = device.createTexture({
  size: [width, height, layerCount],
  format: 'rgba8unorm',
  usage: ...,
});

// 3D纹理
const tex3d = device.createTexture({
  size: [width, height, depth],
  format: 'rgba8unorm',
  usage: ...,
});

// 立方体贴图
const texCube = device.createTexture({
  size: [width, height, 6],
  dimension: 'cube',
  format: 'rgba8unorm',
  usage: ...,
});
```

## 3. GPUTextureView

### 创建视图

```javascript
// 默认视图（整个纹理）
const defaultView = texture.createView();

// 指定mip级别
const mipView = texture.createView({
  baseMipLevel: 0,
  mipLevelCount: 1,
});

// 指定数组层
const layerView = texture.createView({
  baseArrayLayer: 0,
  arrayLayerCount: 1,
});

// 立方体贴图视图
const cubeView = texture.createView({
  dimension: 'cube',
  baseArrayLayer: 0,
  arrayLayerCount: 6,
});
```

### 视图格式重解释

```javascript
// 创建视图时可指定不同格式（兼容性检查后）
const view = texture.createView({
  format: 'rgba8unorm',  // 与纹理原始格式不同的格式
});
```

## 4. GPUSampler

### 创建采样器

```javascript
const sampler = device.createSampler({
  // 放大过滤
  magFilter: 'linear',    // 'linear' | 'nearest'
  
  // 缩小过滤
  minFilter: 'linear',
  
  // MIP过滤
  mipmapFilter: 'linear',
  
  // 寻址模式
  addressModeU: 'clamp-to-edge',
  addressModeV: 'clamp-to-edge',
  addressModeW: 'clamp-to-edge',
  
  // 各向异性过滤
  maxAnisotropy: 16,
  
  // 比较函数（用于PCF等）
  compare: 'less',
});
```

### 过滤模式详解

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| `nearest` | 最近邻过滤 | 像素艺术、UI图标 |
| `linear` | 双线性插值 | 照片、通用纹理 |
| `mipmap` | MIP过滤 | 减少远处闪烁 |

### 寻址模式

| 模式 | 行为 |
|------|------|
| `clamp-to-edge` | 边缘像素重复 |
| `repeat` | 纹理重复 |
| `mirror-repeat` | 镜像重复 |
| ` clamp-to-border` | 使用边框颜色 |

## 5. 纹理与着色器绑定

### Bind Group Layout

```javascript
const bindGroupLayout = device.createBindGroupLayout({
  entries: [
    {
      binding: 0,
      visibility: GPUShaderStage.FRAGMENT,
      texture: {
        sampleType: 'float',      // 'float' | 'unfilterable-float' | 'depth' | 'sint' | 'uint'
        viewDimension: '2d',      // '1d' | '2d' | '2d-array' | 'cube' | 'cube-array' | '3d'
        multisampled: false,
      },
    },
    {
      binding: 1,
      visibility: GPUShaderStage.FRAGMENT,
      sampler: {
        type: 'filtering',         // 'filtering' | 'non-filtering' | 'comparison'
      },
    },
  ],
});
```

### WGSL纹理采样

```wgsl
@group(0) @binding(0) var myTexture: texture_2d<f32>;
@group(0) @binding(1) var mySampler: sampler;

@fragment
fn fragmentMain(
  @location(0) uv: vec2f,
) -> @location(0) vec4f {
  // 采样纹理
  return textureSample(myTexture, mySampler, uv);
}
```

### 不同纹理类型

```wgsl
// 2D纹理
var<uniform> tex2d: texture_2d<f32>;
var<uniform> sampler2d: sampler;

// 2D数组纹理
var<uniform> tex2dArray: texture_2d_array<f32>;
var<uniform> sampler2dArray: sampler;

// 立方体贴图
var<uniform> texCube: texture_cube<f32>;
var<uniform> samplerCube: sampler;

// 3D纹理
var<uniform> tex3d: texture_3d<f32>;
var<uniform> sampler3d: sampler;

// 深度纹理
var<uniform> texDepth: texture_depth_2d;
var<uniform> samplerDepth: sampler_comparison;
```

## 6. 图像加载与处理

### 加载图像为纹理

```javascript
async function loadImageAsTexture(device, url) {
  const img = await loadImage(url);
  
  // 创建纹理
  const texture = device.createTexture({
    size: [img.width, img.height],
    format: 'rgba8unorm',
    usage: GPUTextureUsage.TEXTURE_BINDING |
           GPUTextureUsage.COPY_DST |
           GPUTextureUsage.RENDER_ATTACHMENT,
  });
  
  // 上传图像数据
  device.queue.copyExternalImageToTexture(
    { source: img },
    { destination: texture, origin: [0, 0] },
    [img.width, img.height]
  );
  
  return texture;
}

function loadImage(url) {
  return new Promise((resolve, reject) => {
    const img = new Image();
    img.crossOrigin = 'anonymous';
    img.onload = () => resolve(img);
    img.onerror = reject;
    img.src = url;
  });
}
```

### 从Canvas创建纹理

```javascript
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
ctx.fillStyle = 'red';
ctx.fillRect(0, 0, 100, 100);

const texture = device.createTexture({
  size: [canvas.width, canvas.height],
  format: 'rgba8unorm',
  usage: GPUTextureUsage.TEXTURE_BINDING |
         GPUTextureUsage.COPY_DST,
});

device.queue.copyExternalImageToTexture(
  { source: canvas },
  { destination: texture },
  [canvas.width, canvas.height]
);
```

## 7. 纹理拷贝与转换

### 纹理间拷贝

```javascript
const commandEncoder = device.createCommandEncoder();

// 设置拷贝源和目标
commandEncoder.copyTextureToTexture(
  {
    texture: sourceTexture,
    mipLevel: 0,
    origin: { x: 0, y: 0, z: 0 },
  },
  {
    texture: destTexture,
    mipLevel: 0,
    origin: { x: 0, y: 0, z: 0 },
  },
  [width, height, depth]
);

device.queue.submit([commandEncoder.finish()]);
```

### 纹理到Buffer拷贝

```javascript
const outputBuffer = device.createBuffer({
  size: width * height * 4,
  usage: GPUBufferUsage.COPY_DST | GPUBufferUsage.MAP_READ,
});

const commandEncoder = device.createCommandEncoder();
commandEncoder.copyTextureToBuffer(
  { texture: sourceTexture },
  { buffer: outputBuffer, bufferOffset: 0 },
  [width, height]
);

device.queue.submit([commandEncoder.finish()]);
```

## 8. MIP Map

### 生成MIP链

```javascript
const texture = device.createTexture({
  size: [width, height],
  format: 'rgba8unorm',
  usage: GPUTextureUsage.TEXTURE_BINDING |
         GPUTextureUsage.COPY_DST |
         GPUTextureUsage.RENDER_ATTACHMENT,
  mipLevelCount: Math.ceil(Math.log2(Math.max(width, height))) + 1,
});

// 需要手动生成每层MIP
// 或使用Blender等工具预生成
```

### WGSL中的MIP采样

```wgsl
// 自动选择MIP级别（基于纹理坐标导数）
textureSampleBaseClampToEdge(tex, samp, uv);

// 显式指定MIP级别
textureSampleLevel(tex, samp, uv, level);

// LOD查询
let lod = textureQueryLod(tex, uv);
```

## 9. 立方体贴图

### 创建和使用

```javascript
// 6个面需要分别上传
const cubeTexture = device.createTexture({
  size: [size, size, 6],
  dimension: 'cube',
  format: 'rgba8unorm',
  usage: GPUTextureUsage.TEXTURE_BINDING |
         GPUTextureUsage.COPY_DST,
});

// 上传每个面的数据
for (let i = 0; i < 6; i++) {
  device.queue.copyExternalImageToTexture(
    { source: cubeFaces[i] },
    { destination: cubeTexture, origin: [0, 0, i] },
    [size, size]
  );
}
```

```wgsl
@group(0) @binding(0) var envMap: texture_cube<f32>;
@group(0) @binding(1) var envSampler: sampler;

@fragment
fn fragmentMain(uv: vec2f) -> @location(0) vec4f {
  // 从立方体贴图采样需要3D方向向量
  let direction = normalize(vec3f(uv * 2.0 - 1.0, 1.0));
  return textureSample(envMap, envSampler, direction);
}
```

## 10. 深度纹理

### 创建深度纹理

```javascript
const depthTexture = device.createTexture({
  size: [width, height],
  format: 'depth24plus',
  usage: GPUTextureUsage.RENDER_ATTACHMENT,
});

// 或使用 stencil8
const depthStencilTexture = device.createTexture({
  size: [width, height],
  format: 'depth24plus-stencil8',
  usage: GPUTextureUsage.RENDER_ATTACHMENT,
});
```

### 渲染通道使用

```javascript
const renderPass = commandEncoder.beginRenderPass({
  colorAttachments: [{
    view: colorView,
    loadOp: 'clear',
    storeOp: 'store',
  }],
  depthStencilAttachment: {
    view: depthTexture.createView(),
    depthLoadOp: 'clear',
    depthClearValue: 1.0,
    depthStoreOp: 'store',
    stencilLoadOp: 'clear',
    stencilClearValue: 0,
    stencilStoreOp: 'store',
  },
});
```

## 11. 实战：3D纹理映射

### 顶点着色器

```wgsl
struct VertexInput {
  @location(0) position: vec3f,
  @location(1) normal: vec3f,
  @location(2) uv: vec2f,
}

struct VertexOutput {
  @builtin(position) position: vec4f,
  @location(0) uv: vec2f,
  @location(1) normal: vec3f,
}

@vertex
fn vertexMain(input: VertexInput) -> VertexOutput {
  var output: VertexOutput;
  output.position = vec4f(input.position, 1.0);
  output.uv = input.uv;
  output.normal = input.normal;
  return output;
}
```

### 片元着色器

```wgsl
@group(0) @binding(0) var diffuseTexture: texture_2d<f32>;
@group(0) @binding(1) var diffuseSampler: sampler;

@fragment
fn fragmentMain(input: VertexOutput) -> @location(0) vec4f {
  // 采样漫反射纹理
  let color = textureSample(diffuseTexture, diffuseSampler, input.uv);
  return vec4f(color.rgb * (input.normal.z * 0.5 + 0.5), color.a);
}
```

## 参考资料

- [MDN WebGPU API - Texture](https://developer.mozilla.org/en-US/docs/Web/API/WebGPU_API)
- [WebGPU Fundamentals - Textures](https://webgpufundamentals.org/webgpu/lessons/webgpu-textures.html)
- [WGSL Texture Functions](https://gpuweb.github.io/gpuweb/wgsl/#texture-builtin-functions)
