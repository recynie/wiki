---
title: gRPC and RPC Protocols
date: 2026-04-14
description: gRPC 与 RPC 协议详解
tags:
  - grpc
  - rpc
  - distributed-systems
draft: true
permalink:
---

# gRPC and RPC Protocols

## 概述

RPC（Remote Procedure Call，远程过程调用）是一种通信协议，允许程序像调用本地函数一样调用远程计算机上的函数。gRPC 是 Google 主导的开源 RPC 框架，基于 HTTP/2 和 Protocol Buffers。

## RPC 基础概念

### 工作原理

```
┌──────────────┐                              ┌──────────────┐
│   Client     │         Network              │   Server     │
│              │                              │              │
│  Stub/Proxy  │ ──── RPC Request ────▶       │  Stub/Proxy  │
│              │                              │              │
│  Client      │ ◀──── RPC Response ────      │  Server      │
│  Program     │                              │  Program     │
└──────────────┘                              └──────────────┘
```

**调用流程**：

1. **Client Stub**：将参数打包成消息（Marshaling）
2. **Client Runtime**：发送请求到网络
3. **Server Runtime**：接收请求，传递给 Server Stub
4. **Server Stub**：解析消息，调用本地函数（Unmarshaling）
5. **Server Function**：执行业务逻辑
6. **Response**：结果原路返回

### RPC 与 REST 对比

| 特性 | RPC | REST |
|------|-----|------|
| 抽象层次 | 远程过程调用 | 资源状态转移 |
| 通信协议 | TCP/UDP/HTTP | HTTP |
| 数据格式 | Protobuf/Thrift/JSON | JSON/XML |
| 契约 | IDL（接口定义语言）| OpenAPI/Swagger |
| 适用场景 | 内部服务、低延迟 | Web API、跨组织 |

## Protocol Buffers

Protocol Buffers（Protobuf）是 Google 的序列化协议，用于定义服务接口和消息格式。

### 消息定义

```protobuf
syntax = "proto3";

package user;

option go_package = "github.com/example/user;user";

// 枚举类型
enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
}

// 消息类型
message PhoneNumber {
    string number = 1;
    PhoneType type = 2;
}

message Person {
    string name = 1;          // 字段编号（1-15 使用 1 字节编码）
    int32 id = 2;
    string email = 3;
    repeated PhoneNumber phones = 4;  // repeated 表示数组
    map<string, string> attributes = 5;  // 映射类型
}

// 服务定义
service UserService {
    rpc GetUser (GetUserRequest) returns (Person);
    rpc CreateUser (CreateUserRequest) returns (Person);
    rpc ListUsers (ListUsersRequest) returns (ListUsersResponse);
}
```

### 字段编号

- **1-15**：单字节编码，适合频繁使用的字段
- **16-2047**：双字节编码
- 最大编号：2^29 - 1 = 536,870,911
- 保留字（reserved）防止字段编号冲突

### 生成代码

```bash
# 安装 protoc 编译器
protoc --version

# Go 代码生成
protoc --go_out=. --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
       user.proto

# Python 代码生成
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. user.proto
```

## gRPC 核心特性

### 四种通信模式

```protobuf
// 1. 一元调用（Unary RPC）— 标准的请求-响应
rpc GetUser (GetUserRequest) returns (Person);

// 2. 服务器流式RPC — 服务器返回数据流
rpc ListUsers (ListUsersRequest) returns (stream Person);

// 3. 客户端流式RPC — 客户端发送数据流
rpc CreateUsers (stream CreateUserRequest) returns (CreateUsersResponse);

// 4. 双向流式RPC — 双方都可发送数据流
rpc Chat (stream ChatMessage) returns (stream ChatMessage);
```

### 元数据与拦截器

```go
// 元数据（Metadata）
md := metadata.New(map[string]string{"Authorization": "Bearer token"})
ctx := metadata.NewOutgoingContext(ctx, md)

// 读取元数据
md, ok := metadata.FromIncomingContext(ctx)

// 拦截器（Interceptor）
func unaryInterceptor(ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler) (interface{}, error) {
    
    // 前置处理
    log.Printf("Received: %v", req)
    
    // 调用处理函数
    resp, err := handler(ctx, req)
    
    // 后置处理
    log.Printf("Response: %v", resp)
    
    return resp, err
}

// 创建服务器时注册拦截器
server := grpc.NewServer(
    grpc.UnaryInterceptor(unaryInterceptor),
    grpc.StreamInterceptor(streamInterceptor),
)
```

### 错误处理

```go
import "google.golang.org/grpc/codes"
import "google.golang.org/grpc/status"

// 服务器返回错误
return nil, status.Errorf(codes.NotFound, "user %d not found", userID)

// 客户端处理错误
_, err := client.GetUser(ctx, &user.GetUserRequest{Id: 123})
if err != nil {
    st, ok := status.FromError(err)
    if ok {
        switch st.Code() {
        case codes.NotFound:
            fmt.Println("User not found")
        case codes.Unauthenticated:
            fmt.Println("Auth required")
        default:
            fmt.Println("Unknown error:", st.Message())
        }
    }
}
```

### 双向流示例

```go
// 定义流式服务
service ChatService {
    rpc Chat (stream ChatMessage) returns (stream ChatMessage);
}

// 服务器端实现
func (s *chatServer) Chat(stream grpc.BidiStreamingServer[ChatMessage, ChatMessage]) error {
    for {
        msg, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }
        
        // 处理消息
        response := &ChatMessage{
            Content: fmt.Sprintf("Echo: %s", msg.Content),
        }
        
        if err := stream.Send(response); err != nil {
            return err
        }
    }
}

// 客户端调用
stream, err := client.Chat(ctx)
go func() {
    for {
        if err := stream.Send(&ChatMessage{Content: "hello"}); err != nil {
            log.Fatal(err)
        }
    }
}()
```

## gRPC vs REST 性能对比

| 指标 | gRPC | REST (JSON/HTTP) |
|------|------|------------------|
| 序列化速度 | Protobuf 快 3-10x | JSON 相对慢 |
| 消息体积 | 小（压缩编码）| 大（文本格式）|
| 协议 | HTTP/2 多路复用 | HTTP/1.1 短连接 |
| 流支持 | 原生支持双向流 | 需要 WebSocket |
| 代码生成 | IDL 自动生成 | 手动或 OpenAPI |
| 浏览器支持 | 需要 grpc-web | 原生支持 |

## 实际应用场景

### 微服务通信

gRPC 适合内部微服务间的低延迟通信：

- 服务网格（Istio）原生支持 gRPC
- 支持 TLS 双向认证
- 流式场景（实时数据推送）

### 移动端调用后端

- 体积小、省电
- 低延迟
- 但需要考虑 iOS/Android 的 gRPC 库支持

### 跨语言服务调用

- IDL 定义接口，天然跨语言
- C++, Java, Python, Go, Node.js 等全面支持

## 扩展阅读

- [gRPC 官方文档](https://grpc.io/docs/)
- [Protocol Buffers 文档](https://developers.google.com/protocol-buffers)
- [gRPC 官方示例](https://github.com/grpc/grpc-go/tree/master/examples)
- [RPC 设计指南](https://arxiv.org/abs/1809.01702)

