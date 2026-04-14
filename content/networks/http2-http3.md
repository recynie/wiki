---
title: HTTP/2与HTTP/3协议
date: 2026-04-13
description: HTTP/2多路复用、服务器推送及HTTP/3 QUIC协议
tags:
  - networks
  - http
  - http2
  - http3
  - quic
draft: false
permalink:
---

# HTTP/2与HTTP/3协议

HTTP协议经历了从1.1到2再到3的重大演进，每一次升级都显著提升了Web性能。

## HTTP/1.1的问题

```
请求1: ------>                             <------
请求2:      ------>                       <------
请求3:           ------>                  <------
```

- **队头阻塞**：同一TCP连接上，HTTP请求必须等待前一个响应完成
- **头部冗余**：每次请求都发送相同头部，未压缩
- **无优先级**：所有资源同等优先级

## HTTP/2核心特性

### 多路复用（Multiplexing）

HTTP/2在一个TCP连接上并行传输多个请求和响应：

```
┌─────────────────────────────────────┐
│          TCP Connection             │
│  ┌─────┐  ┌─────┐  ┌─────┐         │
│  │Stream│  │Stream│  │Stream│         │
│  │  1   │  │  3   │  │  5   │         │
│  │(req) │  │(req) │  │(req) │         │
│  │     │  │     │  │     │         │
│  │     │  │     │  │     │         │
│  └─────┘  └─────┘  └─────┘         │
└─────────────────────────────────────┘
```

**帧（Frame）结构**：

```
+---------------+
|Type (8 bits) |   帧类型：HEADERS, DATA, SETTINGS, etc.
|  Length      |   长度（3字节）
|  (24 bits)   |
|---------------|
|  Stream ID   |   流标识符（31 bits）
|  (31 bits)   |
|---------------|
|    Flags     |   标志位
|  (8 bits)    |
|---------------|
|             |
|  Payload    |   载荷
|             |
+---------------+
```

### 头部压缩（HPACK）

使用静态表和动态表压缩头部：

```python
# HPACK伪代码
class HPACK:
    def __init__(self):
        self.static_table = STATIC_TABLE  # 61个预定义条目
        self.dynamic_table = []           # 动态表
    
    def encode(self, name, value):
        # 1. 检查静态表
        idx = self.static_table.find(name, value)
        if idx:
            return encode_integer(idx, 7)  # 高位0b0
        
        # 2. 检查动态表
        idx = self.dynamic_table.find(name, value)
        if idx:
            return encode_integer(62 + idx, 6)  # 高位0b01
        
        # 3. 直接编码
        return encode_literal(name, value)
    
    def decode(self, data):
        # 解码逻辑...
        pass
```

### 服务器推送（Server Push）

服务器主动推送客户端未请求的资源：

```nginx
# Nginx配置示例
server {
    location / {
        http2_push_preload on;
    }
}
```

```html
<!-- HTML中预加载提示 -->
<link rel="preload" href="/style.css" as="style">
<link rel="preload" href="/app.js" as="script">
```

### 流优先级

```http
HEADERS frame (stream 1, weight 256, exclusive)
HEADERS frame (stream 3, weight 128, dependency: 1)
HEADERS frame (stream 5, weight 64, dependency: 1)
```

```
Stream 1 (weight 256)
    ├── Stream 3 (weight 128)
    └── Stream 5 (weight 64)
```

## HTTP/2问题

HTTP/2虽然解决了HTTP/1.1的队头阻塞，但仍受TCP层困扰：

1. **TCP队头阻塞**：丢包导致所有流阻塞
2. **握手延迟**：TCP+TLS需要1-2个RTT
3. **连接迁移**：移动网络切换时连接重建

## HTTP/3与QUIC协议

### QUIC特性

QUIC是基于UDP的传输协议，解决了TCP的问题：

| 特性 | TCP | QUIC |
|------|-----|------|
| 建立连接 | TCP握手 + TLS握手 | 0-RTT / 1-RTT |
| 队头阻塞 | TCP层阻塞 | 仅当前流阻塞 |
| 连接迁移 | IP+Port绑定 | Connection ID |
| 拥塞控制 | 独立实现 | 内置可插拔 |

### QUIC包结构

```
+--------------+-------------+-------------+
|  Header      |   Frame     |  Frame      |
|  (可变长度)   |   1         |  ...        |
+--------------+-------------+-------------+
```

**连接建立（1-RTT握手）**：

```
Client                                          Server
  │                                               │
  │──────── Initial (CID, crypto, SYN) ─────────►│
  │                                               │
  │◄─────── Initial (CID, crypto, SYN-ACK) ──────│
  │                                               │
  │──────── Handshake (crypto, data) ───────────►│
  │                                               │
  │◄────── Handshake (crypto, data) ─────────────│
  │                                               │
  │═══════════ 0-RTT Data =====================═►│
  │                                               │
  │◄══════════ 1-RTT Data (streams) ─────────────│
  │                                               │
```

### HTTP/3帧类型

```python
# HTTP/3帧类型
HTTP3_FRAME_TYPES = {
    0x00: "DATA",          # 应用数据
    0x01: "HEADERS",       # 头部
    0x02: "CANARY_PUSH",   # 推送
    0x03: "SETTINGS",      # 设置
    0x04: "PING",          # 保活
    0x06: "GOAWAY",        # 关闭连接
    0x07: "MAX_DATA",      # 流控
    0x08: "MAX_HEADERS_DATA",  # 流控
}
```

### 丢包处理对比

```
TCP丢包：
Stream 1:  ████████████████░░░░░░░░░░░░░░░░░░
Stream 2:  ████████████████░░░░░░░░░░░░░░░░░░
           [丢包] [全部阻塞等待重传]

QUIC丢包：
Stream 1:  ████████████████░░░░░░░░░░░░░░░░░░
Stream 2:  ████████████████░░░░░░░░░░░░░░░░░░
           [丢包] [Stream 2继续，其他流不受影响]
```

## 性能对比

| 指标 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|------|----------|--------|--------|
| 延迟 | 高 | 中 | 低 |
| 并行 | 需要多连接 | 多路复用 | 多路复用 |
| 丢包影响 | 队头阻塞 | 队头阻塞 | 最小化 |
| 0-RTT | 不支持 | 不支持 | 支持 |
| 握手 | 1 RTT | 2 RTT | 1 RTT |

## 实际应用

### Nginx配置HTTP/2

```nginx
server {
    listen 443 ssl http2;
    
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    # HTTP/2服务器推送
    http2_push_preload on;
    
    location / {
        root /var/www/html;
    }
}
```

### Nginx配置HTTP/3

```nginx
server {
    listen 443 ssl;
    listen 443 http3;
    
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    # QUIC配置
    quic_retry on;
    ssl_early_data on;
}
```

## 参考

[^1]: RFC 9113 - HTTP/2
[^2]: RFC 9114 - HTTP/3
[^3]: IETF QUIC Working Group

