---
title: CUDA基础
date: 2026-04-11
description: CUDA并行计算入门——GPU架构、线程层次、内存模型与Kernel编程
tags:
  - cuda
  - parallel-computing
  - gpu
draft: false
permalink:
---

## 1. 概述

**CUDA**（Compute Unified Device Architecture）是NVIDIA开发的并行计算平台和编程模型，允许开发者利用NVIDIA GPU进行通用计算。与传统CPU串行编程不同，CUDA专为大规模并行计算设计，能够同时启动数万个线程执行任务。

GPU的核心优势在于**大规模并行吞吐量**——虽然单个线程的执行效率不如CPU，但通过同时运行大量线程，GPU在图像处理、深度学习、科学计算等领域展现出远超CPU的性能。

## 2. GPU与CPU架构对比

| 特性 | CPU | GPU |
|------|-----|-----|
| **核心数** | 几个~几十个 | 数百~数千个 |
| **设计目标** | 低延迟、高性能 | 高吞吐量、大规模并行 |
| **缓存** | 大容量、多级缓存 | 小容量、高带宽 |
| **控制逻辑** | 复杂、分支预测 | 简单、规则执行 |
| **适用场景** | 串行逻辑、复杂分支 | 数据并行、规则计算 |

现代GPU如NVIDIA H100拥有上万个CUDA核心，专门设计用于处理大规模并行工作负载。

## 3. CUDA编程模型

### 3.1 主机与设备

CUDA采用**异构计算**模型：

- **主机（Host）**：CPU端代码，负责内存管理、Kernel启动、数据传输
- **设备（Device）**：GPU端代码，执行并行计算任务

```cpp
#include <cuda_runtime.h>
#include <stdio.h>

// Kernel定义——在GPU上执行
__global__ void helloFromGPU() {
    printf("Hello from GPU block %d, thread %d\n", 
           blockIdx.x, threadIdx.x);
}

int main() {
    printf("Hello from CPU\n");
    
    // 启动GPU Kernel
    // <<<blocks_per_grid, threads_per_block>>>
    helloFromGPU<<<2, 4>>>();
    
    // 等待GPU完成
    cudaDeviceSynchronize();
    
    return 0;
}
```

### 3.2 Kernel函数

**Kernel**是在GPU上并行执行的函数，使用`__global__`限定符声明：

```cpp
__global__ void vectorAdd(float* A, float* B, float* C, int N) {
    // 计算全局线程索引
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    
    if (i < N) {
        C[i] = A[i] + B[i];
    }
}
```

Kernel调用的三双括号`<<<...>>>`语法用于指定执行配置：
- 第一个参数：Grid中Block的数量
- 第二个参数：每个Block中Thread的数量

## 4. 线程层次结构

CUDA的线程组织为三层层次结构：

```
Grid（网格）
  └── Block（线程块）× N
        └── Thread（线程）× M
```

### 4.1 索引计算

每个线程通过内置变量确定自己的身份：

| 内置变量 | 类型 | 含义 |
|----------|------|------|
| `threadIdx` | `uint3` | 线程在Block内的索引 |
| `blockIdx` | `uint3` | Block在Grid内的索引 |
| `blockDim` | `dim3` | 每个Block的维度 |
| `gridDim` | `dim3` | Grid的维度 |

**一维Grid计算全局索引**：

```cpp
__global__ void process(float* data, int N) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    
    if (idx < N) {
        data[idx] *= 2.0f;
    }
}

// 调用
int numBlocks = (N + threadsPerBlock - 1) / threadsPerBlock;
process<<<numBlocks, threadsPerBlock>>>(data, N);
```

**二维索引示例（矩阵运算）**：

```cpp
// 二维Block，二维Grid
__global__ void matrixAdd(float A[N][N], float B[N][N], float C[N][N]) {
    int row = blockIdx.y * blockDim.y + threadIdx.y;
    int col = blockIdx.x * blockDim.x + threadIdx.x;
    
    if (row < N && col < N) {
        C[row][col] = A[row][col] + B[row][col];
    }
}

// 调用
dim3 threadsPerBlock(16, 16);
dim3 numBlocks((N + 15) / 16, (N + 15) / 16);
matrixAdd<<<numBlocks, threadsPerBlock>>>(A, B, C);
```

### 4.2 线程Block限制

- 每个Block最多**1024个线程**
- Block的所有线程必须在同一个SM（流式多处理器）上执行
- 线程可以共享Block内的共享内存

### 4.3 Warp执行

**Warp**是GPU执行的基本单位，包含**32个线程**。同一个Warp内的线程同时执行相同的指令：

```
Warp 0: Thread 0-31 → 同时执行同一条指令
Warp 1: Thread 32-63 → 同时执行同一条指令
...
```

当Warp内线程出现分支（如`if-else`）时，会产生**分支分化**，降低效率。

## 5. 内存层次结构

CUDA提供多种内存空间，各有不同作用域和带宽：

| 内存类型 | 作用域 | 带宽 | 延迟 |
|----------|--------|------|------|
| 寄存器 | 单线程 | 最高 | 最低 |
| 共享内存 | Block内线程 | 高 | 低 |
| 全局内存 | 所有线程 | 低 | 高 |
| 常量内存 | 所有线程 | 中 | 中 |
| 纹理内存 | 所有线程 | 中 | 中 |

### 5.1 全局内存

全局内存是GPU上容量最大、延迟最高的内存，用于存储主要数据：

```cpp
// 分配设备内存
float* d_data;
cudaMalloc(&d_data, size * sizeof(float));

// 复制数据到设备
cudaMemcpy(d_data, h_data, size * sizeof(float), cudaMemcpyHostToDevice);

// 使用GPU计算...

// 复制结果回主机
cudaMemcpy(h_result, d_data, size * sizeof(float), cudaMemcpyDeviceToHost);

// 释放内存
cudaFree(d_data);
```

### 5.2 共享内存

共享内存是Block内线程共享的快速内存，用于线程间协作：

```cpp
__global__ void sharedMemoryKernel(float* data, int N) {
    // 声明共享内存
    __shared__ float sharedData[256];
    
    int tid = threadIdx.x;
    int gid = blockIdx.x * blockDim.x + threadIdx.x;
    
    // 加载数据到共享内存
    sharedData[tid] = (gid < N) ? data[gid] : 0;
    
    // Block内线程同步
    __syncthreads();
    
    // 在共享内存中进行处理（如归约）
    for (int s = blockDim.x / 2; s > 0; s >>= 1) {
        if (tid < s && gid + s < N) {
            sharedData[tid] += sharedData[tid + s];
        }
        __syncthreads();
    }
    
    // 将结果写回全局内存
    if (tid == 0) {
        data[blockIdx.x] = sharedData[0];
    }
}
```

## 6. 错误处理

CUDA API调用和Kernel执行都可能失败，应进行错误检查：

```cpp
#define CUDA_CHECK(call) \
    do { \
        cudaError_t err = call; \
        if (err != cudaSuccess) { \
            fprintf(stderr, "CUDA error at %s:%d: %s\n", \
                    __FILE__, __LINE__, \
                    cudaGetErrorString(err)); \
            exit(EXIT_FAILURE); \
        } \
    } while(0)

// 使用示例
CUDA_CHECK(cudaMalloc(&d_data, size * sizeof(float)));
CUDA_CHECK(cudaMemcpy(d_data, h_data, size * sizeof(float), cudaMemcpyHostToDevice));
```

## 7. 常用库函数

### 7.1 内存管理

```cpp
cudaMalloc(void** devPtr, size_t size)           // 分配设备内存
cudaFree(void* devPtr)                            // 释放设备内存
cudaMemcpy(void* dst, void* src, size_t, kind)   // 内存复制
cudaMemset(void* devPtr, int value, size_t count) // 内存设置
```

### 7.2 同步操作

```cpp
cudaDeviceSynchronize()    // 设备同步
__syncthreads()             // Block内线程同步
__syncwarp()                // Warp内线程同步
```

### 7.3 设备信息查询

```cpp
int deviceCount;
cudaGetDeviceCount(&deviceCount);

cudaDeviceProp prop;
cudaGetDeviceProperties(&prop, 0);

printf("Device name: %s\n", prop.name);
printf("Total global memory: %zu bytes\n", prop.totalGlobalMem);
printf("Max threads per block: %d\n", prop.maxThreadsPerBlock);
```

## 8. 性能优化原则

### 8.1 内存合并访问

连续线程访问连续内存地址可以触发合并访问，提高带宽利用率：

```cpp
// 合并访问 ✓
__global__ void coalescedAccess(float* data, int N) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < N) {
        float x = data[i];      // 线程i访问data[i]
        data[i] = x * 2.0f;
    }
}

// 非合并访问 ✗（应避免）
__global__ void stridedAccess(float* data, int N) {
    int i = threadIdx.x;
    if (i < N) {
        float x = data[i * 4];  // 线程0访问data[0]，线程1访问data[4]...
    }
}
```

### 8.2 最大化Occupancy

**Occupancy**指每个SM上活跃线程的比例。提高Occupancy的方法：

- 选择合适的Block大小（通常256或512）
- 减少每个线程的寄存器使用
- 合理使用共享内存

### 8.3 避免分支分化

同一Warp内尽量避免条件分支：

```cpp
// 低效：Warp内部分线程走if，部分走else
if (threadIdx.x % 2 == 0) {
    // Half warp执行
} else {
    // 另Half warp执行
}

// 更好：使用谓词执行
bool condition = (threadIdx.x % 2 == 0);
float result = condition ? value1 : value2;
```

## 9. 完整示例：向量加法

```cpp
#include <cuda_runtime.h>
#include <stdio.h>

#define N 1000000
#define BLOCK_SIZE 256

// 错误检查宏
#define CUDA_CHECK(call) \
    do { \
        cudaError_t err = call; \
        if (err != cudaSuccess) { \
            fprintf(stderr, "CUDA error: %s\n", cudaGetErrorString(err)); \
            exit(EXIT_FAILURE); \
        } \
    } while(0)

// 主机端向量加法
void hostVectorAdd(float* h_A, float* h_B, float* h_C) {
    for (int i = 0; i < N; i++) {
        h_C[i] = h_A[i] + h_B[i];
    }
}

// GPU端向量加法Kernel
__global__ void vectorAdd(float* A, float* B, float* C, int N) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < N) {
        C[i] = A[i] + B[i];
    }
}

int main() {
    float *h_A, *h_B, *h_C;      // 主机端内存
    float *d_A, *d_B, *d_C;      // 设备端内存
    size_t size = N * sizeof(float);
    
    // 分配主机内存
    h_A = (float*)malloc(size);
    h_B = (float*)malloc(size);
    h_C = (float*)malloc(size);
    
    // 初始化数据
    for (int i = 0; i < N; i++) {
        h_A[i] = i * 1.0f;
        h_B[i] = i * 2.0f;
    }
    
    // 分配设备内存
    CUDA_CHECK(cudaMalloc(&d_A, size));
    CUDA_CHECK(cudaMalloc(&d_B, size));
    CUDA_CHECK(cudaMalloc(&d_C, size));
    
    // 复制数据到设备
    CUDA_CHECK(cudaMemcpy(d_A, h_A, size, cudaMemcpyHostToDevice));
    CUDA_CHECK(cudaMemcpy(d_B, h_B, size, cudaMemcpyHostToDevice));
    
    // 启动Kernel
    int numBlocks = (N + BLOCK_SIZE - 1) / BLOCK_SIZE;
    vectorAdd<<<numBlocks, BLOCK_SIZE>>>(d_A, d_B, d_C, N);
    
    // 复制结果回主机
    CUDA_CHECK(cudaMemcpy(h_C, d_C, size, cudaMemcpyDeviceToHost));
    
    // 验证结果
    for (int i = 0; i < 10; i++) {
        printf("C[%d] = %f + %f = %f\n", i, h_A[i], h_B[i], h_C[i]);
    }
    
    // 释放内存
    free(h_A); free(h_B); free(h_C);
    cudaFree(d_A); cudaFree(d_B); cudaFree(d_C);
    
    return 0;
}
```

编译运行：

```bash
nvcc -o vector_add vector_add.cu
./vector_add
```

## 10. CUDA与深度学习

CUDA是深度学习框架（TensorFlow、PyTorch）的基础：

```python
# PyTorch使用CUDA示例
import torch

# 检查CUDA是否可用
print(torch.cuda.is_available())

# 创建GPU张量
device = torch.device('cuda')
x = torch.randn(1000, 1000).to(device)
y = torch.randn(1000, 1000).to(device)

# GPU矩阵乘法
z = torch.matmul(x, y)
```

## 11. 参考资料

[^1]: [NVIDIA CUDA Documentation](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)

[^2]: [CUDA Refresher: The CUDA Programming Model](https://developer.nvidia.com/blog/cuda-refresher-cuda-programming-model/)

[^3]: [Programming Massively Parallel Processors](https://www.elsevier.com/books/programming-massively-parallel-processors/chen/978-0-323-91230-5)

---

## 相关主题

- [[../parallel-programming/concurrent-programming-basics|并发编程基础]] — CPU多线程编程
- [[../distributed-systems/distributed-systems-overview|分布式系统概述]] — 多节点协调计算
