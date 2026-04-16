---
title: WebGPU 3D Rendering Advanced
date: 2026-04-16
description: WebGPU 3D渲染进阶，涵盖3D数学、矩阵变换、光照模型、场景管理和实战3D立方体渲染
tags:
  - webgpu
  - 3d
  - rendering
  - matrix
  - lighting
  - shader
draft: false
permalink:
---

## 1. 3D数学基础

### 向量

```wgsl
// 3D向量
var direction: vec3f = vec3f(1.0, 0.0, 0.0);

// 归一化
let normalized = normalize(direction);

// 点积（投影）
let dotProduct = dot(direction, vec3f(0.0, 1.0, 0.0));  // = 0

// 叉积（法向量）
let normal = cross(vec3f(1.0, 0.0, 0.0), vec3f(0.0, 1.0, 0.0));  // = (0, 0, 1)
```

### 矩阵运算

```wgsl
// 创建4x4矩阵
var matrix: mat4x4f = mat4x4f(
  1.0, 0.0, 0.0, 0.0,  // 第一列
  0.0, 1.0, 0.0, 0.0,  // 第二列
  0.0, 0.0, 1.0, 0.0,  // 第三列
  0.0, 0.0, 0.0, 1.0   // 第四列
);

// 矩阵乘法
let result = matrixA * matrixB;

// 逆矩阵
let inverseMatrix = inverse(matrix);

// 转置矩阵
let transposedMatrix = transpose(matrix);
```

## 2. 矩阵变换

### 变换矩阵类型

| 变换 | 矩阵形式 |
|------|----------|
| 平移 | `[1,0,0,tx; 0,1,0,ty; 0,0,1,tz; 0,0,0,1]` |
| 缩放 | `[sx,0,0,0; 0,sy,0,0; 0,0,sz,0; 0,0,0,1]` |
| 旋转X | `[1,0,0,0; 0,cos,-sin,0; 0,sin,cos,0; 0,0,0,1]` |
| 旋转Y | `[cos,0,sin,0; 0,1,0,0; -sin,0,cos,0; 0,0,0,1]` |
| 旋转Z | `[cos,-sin,0,0; sin,cos,0,0; 0,0,1,0; 0,0,0,1]` |

### WGSL变换函数

```wgsl
// 平移矩阵
fn translate(d: vec3f) -> mat4x4f {
  return mat4x4f(
    1.0, 0.0, 0.0, 0.0,
    0.0, 1.0, 0.0, 0.0,
    0.0, 0.0, 1.0, 0.0,
    d.x, d.y, d.z, 1.0
  );
}

// 缩放矩阵
fn scale(s: vec3f) -> mat4x4f {
  return mat4x4f(
    s.x, 0.0, 0.0, 0.0,
    0.0, s.y, 0.0, 0.0,
    0.0, 0.0, s.z, 0.0,
    0.0, 0.0, 0.0, 1.0
  );
}

// 绕Y轴旋转
fn rotateY(angle: f32) -> mat4x4f {
  let c = cos(angle);
  let s = sin(angle);
  return mat4x4f(
    c, 0.0, s, 0.0,
    0.0, 1.0, 0.0, 0.0,
    -s, 0.0, c, 0.0,
    0.0, 0.0, 0.0, 1.0
  );
}
```

## 3. 投影变换

### 正交投影

```wgsl
fn ortho(left: f32, right: f32, bottom: f32, top: f32, near: f32, far: f32) -> mat4x4f {
  let lr = 1.0 / (right - left);
  let bt = 1.0 / (top - bottom);
  let nf = 1.0 / (near - far);
  return mat4x4f(
    2.0 * lr, 0.0, 0.0, 0.0,
    0.0, 2.0 * bt, 0.0, 0.0,
    0.0, 0.0, 2.0 * nf, 0.0,
    -(right + left) * lr, -(top + bottom) * bt, (far + near) * nf, 1.0
  );
}
```

### 透视投影

```wgsl
fn perspective(fovY: f32, aspect: f32, near: f32, far: f32) -> mat4x4f {
  let f = 1.0 / tan(fovY / 2.0);
  let nf = 1.0 / (near - far);
  return mat4x4f(
    f / aspect, 0.0, 0.0, 0.0,
    0.0, f, 0.0, 0.0,
    0.0, 0.0, (far + near) * nf, -1.0,
    0.0, 0.0, 2.0 * far * near * nf, 0.0
  );
}
```

### 视图矩阵（LookAt）

```wgsl
fn lookAt(eye: vec3f, center: vec3f, up: vec3f) -> mat4x4f {
  let z = normalize(eye - center);      // Z轴（视线方向）
  let x = normalize(cross(up, z));     // X轴
  let y = cross(z, x);                 // Y轴
  return mat4x4f(
    x.x, y.x, z.x, 0.0,
    x.y, y.y, z.y, 0.0,
    x.z, y.z, z.z, 0.0,
    -dot(x, eye), -dot(y, eye), -dot(z, eye), 1.0
  );
}
```

## 4. Uniform缓冲

### 创建Uniform缓冲

```javascript
const uniformBufferSize = 256;  // 对齐到256字节边界
const uniformBuffer = device.createBuffer({
  size: uniformBufferSize,
  usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
});

// Uniform数据布局
const uniformData = new Float32Array([
  // 视图投影矩阵 (16 floats)
  ...viewProjectionMatrix,
  // 模型矩阵 (16 floats)
  ...modelMatrix,
  // 其他uniforms...
]);
```

### 着色器中使用Uniform

```wgsl
struct Uniforms {
  viewProjection: mat4x4f,
  model: mat4x4f,
  normalMatrix: mat3x3f,
}

@group(0) @binding(0) var<uniform> uniforms: Uniforms;

@vertex
fn vertexMain(input: VertexInput) -> VertexOutput {
  var output: VertexOutput;
  output.position = uniforms.viewProjection * uniforms.model * vec4f(input.position, 1.0);
  output.normal = uniforms.normalMatrix * input.normal;
  return output;
}
```

## 5. 光照模型

### 方向光

```wgsl
struct DirectionalLight {
  direction: vec3f,
  color: vec3f,
}

fn calcDirectionalLight(
  light: DirectionalLight,
  normal: vec3f,
  viewDir: vec3f,
  baseColor: vec3f
) -> vec3f {
  // 漫反射
  let NdotL = max(dot(normal, light.direction), 0.0);
  let diffuse = baseColor * light.color * NdotL;
  
  // 高光反射
  let reflectDir = reflect(-light.direction, normal);
  let VdotR = pow(max(dot(viewDir, reflectDir), 0.0), 32.0);
  let specular = light.color * VdotR * 0.5;
  
  return diffuse + specular;
}
```

### 点光源

```wgsl
struct PointLight {
  position: vec3f,
  color: vec3f,
  constant: f32,
  linear: f32,
  quadratic: f32,
}

fn calcPointLight(
  light: PointLight,
  fragPos: vec3f,
  normal: vec3f,
  viewDir: vec3f,
  baseColor: vec3f
) -> vec3f {
  let lightDir = normalize(light.position - fragPos);
  
  // 漫反射
  let diff = max(dot(normal, lightDir), 0.0);
  
  // 衰减
  let distance = length(light.position - fragPos);
  let attenuation = 1.0 / (light.constant + light.linear * distance + 
                            light.quadratic * distance * distance);
  
  return baseColor * light.color * diff * attenuation;
}
```

### 聚光灯

```wgsl
struct SpotLight {
  position: vec3f,
  direction: vec3f,
  color: vec3f,
  cutoff: f32,      // 余弦值
  outerCutoff: f32,
}

fn calcSpotLight(
  light: SpotLight,
  fragPos: vec3f,
  normal: vec3f,
  viewDir: vec3f,
  baseColor: vec3f
) -> vec3f {
  let lightDir = normalize(light.position - fragPos);
  
  // 漫反射
  let diff = max(dot(normal, lightDir), 0.0);
  
  // 聚光角度
  let theta = dot(lightDir, -light.direction);
  let epsilon = light.cutoff - light.outerCutoff;
  let intensity = clamp((theta - light.outerCutoff) / epsilon, 0.0, 1.0);
  
  return baseColor * light.color * diff * intensity;
}
```

## 6. 深度测试与混合

### 配置深度测试

```javascript
const pipeline = device.createRenderPipeline({
  // ...
  primitive: {
    topology: 'triangle-list',
    cullMode: 'back',  // 'none' | 'front' | 'back'
    frontFace: 'ccw', // 'ccw' | 'cw'
  },
  depthStencil: {
    format: 'depth24plus',
    depthWriteEnabled: true,
    depthCompare: 'less',  // 'never' | 'less' | 'equal' | 'less-equal' | 'greater' | etc.
  },
});
```

### 混合模式

```javascript
const pipeline = device.createRenderPipeline({
  // ...
  fragment: {
    // ...
    targets: [{
      format: format,
      blend: {
        color: {
          srcFactor: 'src-alpha',
          dstFactor: 'one-minus-src-alpha',
          operation: 'add',
        },
        alpha: {
          srcFactor: 'one',
          dstFactor: 'one-minus-src-alpha',
          operation: 'add',
        },
      },
    }],
  },
});
```

## 7. 场景管理与对象

### 简单场景图

```javascript
class SceneObject {
  constructor(mesh, material) {
    this.mesh = mesh;
    this.material = material;
    this.transform = mat4.create();
    this.children = [];
  }
  
  addChild(child) {
    this.children.push(child);
  }
  
  getWorldMatrix() {
    return this.transform;
  }
}

class Camera {
  constructor() {
    this.position = vec3.create(0, 0, 5);
    this.target = vec3.create(0, 0, 0);
    this.up = vec3.create(0, 1, 0);
    this.viewMatrix = mat4.create();
    this.projectionMatrix = mat4.create();
  }
  
  updateViewMatrix() {
    this.viewMatrix = mat4.lookAt(this.position, this.target, this.up);
  }
}
```

## 8. 实战：3D立方体渲染

### 顶点数据

```javascript
const vertices = new Float32Array([
  // 位置 (x,y,z)          // 法向量 (nx,ny,nz)
  // 前面
  -0.5, -0.5,  0.5,       0.0, 0.0, 1.0,
   0.5, -0.5,  0.5,       0.0, 0.0, 1.0,
   0.5,  0.5,  0.5,       0.0, 0.0, 1.0,
  -0.5,  0.5,  0.5,       0.0, 0.0, 1.0,
  // 后面
   0.5, -0.5, -0.5,       0.0, 0.0, -1.0,
  -0.5, -0.5, -0.5,       0.0, 0.0, -1.0,
  -0.5,  0.5, -0.5,       0.0, 0.0, -1.0,
   0.5,  0.5, -0.5,       0.0, 0.0, -1.0,
  // 其他面...
]);

const indices = new Uint16Array([
  // 前面
  0, 1, 2,  2, 3, 0,
  // 后面
  4, 5, 6,  6, 7, 4,
  // 顶面
  3, 2, 7,  7, 6, 3,
  // 底面
  5, 4, 0,  0, 4, 5,
  // 右面
  1, 4, 7,  7, 2, 1,
  // 左面
  5, 0, 3,  3, 6, 5,
]);
```

### 完整着色器

```wgsl
struct Uniforms {
  viewProjection: mat4x4f,
  model: mat4x4f,
  normalMatrix: mat3x3f,
  lightDir: vec3f,
  lightColor: vec3f,
}

struct VertexInput {
  @location(0) position: vec3f,
  @location(1) normal: vec3f,
}

struct VertexOutput {
  @builtin(position) position: vec4f,
  @location(0) worldPos: vec3f,
  @location(1) normal: vec3f,
}

@group(0) @binding(0) var<uniform> uniforms: Uniforms;

@vertex
fn vertexMain(input: VertexInput) -> VertexOutput {
  var output: VertexOutput;
  let worldPos = uniforms.model * vec4f(input.position, 1.0);
  output.position = uniforms.viewProjection * worldPos;
  output.worldPos = worldPos.xyz;
  output.normal = uniforms.normalMatrix * input.normal;
  return output;
}

@fragment
fn fragmentMain(input: VertexOutput) -> @location(0) vec4f {
  // 基础颜色
  let baseColor = vec3f(0.8, 0.2, 0.2);
  
  // 归一化法向量
  let N = normalize(input.normal);
  let L = normalize(uniforms.lightDir);
  let V = normalize(vec3f(0.0, 0.0, 5.0) - input.worldPos);
  
  // 漫反射
  let diff = max(dot(N, L), 0.0);
  let diffuse = baseColor * uniforms.lightColor * diff;
  
  // 环境光
  let ambient = baseColor * 0.1;
  
  return vec4f(diffuse + ambient, 1.0);
}
```

### 渲染循环

```javascript
function render() {
  const commandEncoder = device.createCommandEncoder();
  
  const renderPass = commandEncoder.beginRenderPass({
    colorAttachments: [{
      view: context.getCurrentTexture().createView(),
      loadOp: 'clear',
      storeOp: 'store',
    }],
    depthStencilAttachment: {
      view: depthTexture.createView(),
      depthLoadOp: 'clear',
      depthClearValue: 1.0,
      depthStoreOp: 'store',
    },
  });
  
  // 更新Uniforms
  const t = performance.now() / 1000;
  const modelMatrix = rotateY(t);
  const normalMatrix = mat3.fromMat4(modelMatrix);
  device.queue.writeBuffer(uniformBuffer, 0, new Float32Array([
    ...projectionMatrix,
    ...viewMatrix,
    ...modelMatrix,
    ...normalMatrix,
    Math.cos(t), Math.sin(t), 1.0,  // lightDir
    1.0, 1.0, 1.0,                   // lightColor
  ]));
  
  renderPass.setPipeline(pipeline);
  renderPass.setVertexBuffer(0, vertexBuffer);
  renderPass.setIndexBuffer(indexBuffer, 'uint16');
  renderPass.setBindGroup(0, bindGroup);
  renderPass.drawIndexed(36);
  
  renderPass.end();
  device.queue.submit([commandEncoder.finish()]);
  
  requestAnimationFrame(render);
}
```

## 9. 性能优化

### 减少Draw Call

- **实例化渲染**：使用 `drawIndexedInstanced`
- **批量处理**：合并相同材质的对象
- **LOD系统**：远处对象使用简化模型

### 纹理优化

- **MIP Map**：减少远处纹理闪烁
- **纹理图集**：合并小纹理减少采样
- **压缩纹理**：使用BC/DXT/ASTC格式

### 着色器优化

- **避免分支**：`select()` 替代 `if`
- **减少纹理采样**：缓存计算结果
- **使用uster存储**：高频访问数据放入uster

## 参考资料

- [WebGPU Fundamentals - 3D Math](https://webgpufundamentals.org/webgpu/lessons/webgpu-3d-perspective-projection.html)
- [WebGPU Fundamentals - Lighting](https://webgpufundamentals.org/webgpu/lessons/webgpu-lighting-and-shading.html)
- [LearnOpenGL - Transformations](https://learnopengl.com/Getting-started/Transformations)
