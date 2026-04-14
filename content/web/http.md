---
title: HTTP协议详解
date: 2026-04-08
description: HTTP协议详解 — 超文本传输协议的核心概念、请求响应结构、状态码、缓存机制、HTTPS安全以及HTTP/2、HTTP/3新特性
tags:
  - http
  - network
  - web
  - https
draft: false
permalink:
---

## 概述

HTTP（HyperText Transfer Protocol）是万维网的基础协议，用于在客户端（浏览器）和服务器之间传输超文本资源。HTTP是一种无状态协议，每个请求都是独立的，服务器不保留客户端的历史请求信息。

### 协议版本演进

| 版本 | 年份 | 主要特性 |
|------|------|----------|
| HTTP/0.9 | 1991 | 仅支持GET请求 |
| HTTP/1.0 | 1996 | 引入请求/响应头、状态码 |
| HTTP/1.1 | 1999 | 持久连接、管道化、缓存优化 |
| HTTP/2 | 2015 | 多路复用、头部压缩、服务器推送 |
| HTTP/3 | 2022 | 基于QUIC、0-RTT、改进的拥塞控制 |

## 请求与响应结构

### HTTP请求格式

```
请求行
GET /index.html HTTP/1.1
请求头
Host: example.com
User-Agent: Mozilla/5.0
Accept: text/html
空行 (CRLF)
可选请求体
```

### HTTP响应格式

```
状态行
HTTP/1.1 200 OK
响应头
Content-Type: text/html
Content-Length: 1234
空行 (CRLF)
响应体
<!DOCTYPE html>
<html>...
```

## 请求方法

| 方法 | 语义 | 安全性 | 幂等性 |
|------|------|--------|--------|
| GET | 获取资源 | ✅ | ✅ |
| POST | 提交数据 | ❌ | ❌ |
| PUT | 替换资源 | ❌ | ✅ |
| DELETE | 删除资源 | ❌ | ✅ |
| HEAD | 获取响应头 | ✅ | ✅ |
| OPTIONS | 获取支持的方法 | ✅ | ✅ |
| PATCH | 部分更新 | ❌ | ❌ |

### 请求示例

```bash
# GET请求
GET /api/users?id=1 HTTP/1.1
Host: api.example.com
Authorization: Bearer <token>
Accept: application/json

# POST请求
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Content-Length: 58

{"name":"张三","email":"zhang@example.com"}
```

## 状态码

### 1xx - 信息性状态码

| 状态码 | 含义 |
|--------|------|
| 100 Continue | 服务器收到请求的部分，客户端继续 |
| 101 Switching Protocols | 协议切换（如HTTP→WebSocket） |

### 2xx - 成功状态码

| 状态码 | 含义 |
|--------|------|
| 200 OK | 请求成功 |
| 201 Created | 资源创建成功 |
| 204 No Content | 成功但无返回内容 |
| 206 Partial Content | 部分内容（用于断点续传） |

### 3xx - 重定向状态码

| 状态码 | 含义 |
|--------|------|
| 301 Moved Permanently | 永久重定向 |
| 302 Found | 临时重定向 |
| 304 Not Modified | 资源未修改（缓存） |
| 307 Temporary Redirect | 临时重定向（保持方法） |
| 308 Permanent Redirect | 永久重定向（保持方法） |

### 4xx - 客户端错误状态码

| 状态码 | 含义 |
|--------|------|
| 400 Bad Request | 请求格式错误 |
| 401 Unauthorized | 未认证 |
| 403 Forbidden | 无权限 |
| 404 Not Found | 资源不存在 |
| 405 Method Not Allowed | 方法不支持 |
| 409 Conflict | 资源冲突 |
| 422 Unprocessable Entity | 语义错误 |
| 429 Too Many Requests | 请求过多 |

### 5xx - 服务器错误状态码

| 状态码 | 含义 |
|--------|------|
| 500 Internal Server Error | 服务器内部错误 |
| 502 Bad Gateway | 网关错误 |
| 503 Service Unavailable | 服务不可用 |
| 504 Gateway Timeout | 网关超时 |

## HTTP Headers

### 通用头（General Headers）

```
Cache-Control: max-age=3600
Connection: keep-alive
Date: Tue, 08 Apr 2026 12:00:00 GMT
Via: 1.1 varnish
```

### 请求头（Request Headers）

```
Host: example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Accept: text/html,application/json
Accept-Language: zh-CN,en;q=0.9
Accept-Encoding: gzip, deflate, br
Authorization: Bearer <token>
Cookie: session_id=abc123
Referer: https://google.com
```

### 响应头（Response Headers）

```
Content-Type: text/html; charset=utf-8
Content-Length: 1234
Content-Encoding: gzip
Server: nginx/1.20.1
Set-Cookie: session=xyz; HttpOnly; Secure; SameSite=Strict
```

### 自定义头

以 `X-` 前缀开头（已不推荐），建议使用标准化扩展头。

## 缓存机制

### 缓存控制策略

```http
# 不缓存
Cache-Control: no-store

# 可缓存但需验证
Cache-Control: no-cache

# 私有缓存（如浏览器）
Cache-Control: private

# 公共缓存（如CDN）
Cache-Control: public

# 最大生存时间
Cache-Control: max-age=3600

# 必须重新验证
Cache-Control: must-revalidate
```

### 缓存新鲜度计算

```
Freshness = max-age + (Date - Last-Modified) * 0.1
```

### 条件请求

```http
# 基于修改时间
If-Modified-Since: Tue, 08 Apr 2026 10:00:00 GMT

# 基于ETag（唯一标识）
If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"
```

### 缓存决策流程

```
请求到达
    ↓
检查本地缓存
    ↓
缓存新鲜？
  是 → 直接返回（Cache Hit）
  否 → 发送条件请求
    ↓
服务器返回304？
  是 → 更新缓存元数据，返回
  否 → 返回新资源，更新缓存
```

## HTTPS安全

### TLS握手过程（简化）

```
客户端                          服务器
   |                               |
   |------- ClientHello --------->|
   |      (支持 TLS 版本、加密套件) |
   |                               |
   |<------ ServerHello ------------|
   |      (选定 TLS 版本、加密套件)  |
   |<------ Certificate ------------|
   |      (服务器证书)              |
   |<------ ServerHelloDone --------|
   |                               |
   |------- ClientKeyExchange ---->|
   |      (预主密钥，用公钥加密)     |
   |                               |
   |------- ChangeCipherSpec ----->|
   |------- Finished ------------->|
   |                               |
   |<------ ChangeCipherSpec ------|
   |<------ Finished --------------|
   |                               |
   ====== 加密通信开始 =============
```

### HTTPS特点

- **加密**：传输内容加密，防止窃听
- **认证**：证书验证服务器身份
- **完整性**：MAC校验，防止篡改

## HTTP/2新特性

### 多路复用（Multiplexing）

```http
# HTTP/1.1 管线化问题：
GET /a.js HTTP/1.1
GET /b.js HTTP/1.1  # 需等待前一个响应

# HTTP/2 多路复用：
# 在单个TCP连接上并行传输多个请求和响应
stream 1: GET /a.js
stream 2: GET /b.js
stream 3: GET /c.js
```

### 头部压缩（HPACK）

```http
# 使用静态表和动态表压缩头部
# 典型压缩率：70-90%
```

### 服务器推送（Server Push）

```http
# 服务器主动推送资源
< HTTP/2 200
< content-type: text/html
< link: </style.css>; rel=preload; as=style
< link: </app.js>; rel=preload; as=script
```

### 二进制分帧

```
HTTP/2 将消息分解为帧：
+---+---+---+---+---+---+---+
| HEADERS     |   frame     |
| frame, cont |   continues |
+---+---+---+---+---+---+---+

DATA frames contain the payload
```

## HTTP/3新特性

### QUIC协议

HTTP/3 基于 QUIC（Quick UDP Internet Connections），运行在 UDP 上：

| 特性 | HTTP/2 | HTTP/3 |
|------|--------|--------|
| 传输层 | TCP | UDP (QUIC) |
| 连接建立 | 1-RTT | 0-RTT (恢复) |
| 线头阻塞 | 有 | 无 |
| 拥塞控制 | TCP | QUIC内置 |

### 0-RTT恢复

```
首次连接：
客户端 -> 服务器: ClientHello (0-RTT数据)
服务器 -> 客户端: ServerHello + 加密响应

恢复连接：
客户端 -> 服务器: ClientHello + 0-RTT数据 (复用会话票据)
服务器 -> 客户端: 加密响应
```

## REST API设计原则

### REST约束

1. **客户端-服务器**：分离关注点
2. **无状态**：每个请求包含所有信息
3. **可缓存**：响应标明了缓存性
4. **分层系统**：通过中间层扩展
5. **统一接口**：资源通过URI标识

### URL设计

```http
# 资源命名
GET  /users          # 获取用户列表
GET  /users/123      # 获取特定用户
POST /users          # 创建用户
PUT  /users/123      # 更新用户
DELETE /users/123    # 删除用户

# 嵌套资源
GET  /users/123/posts    # 获取用户的文章
GET  /users/123/orders/5 # 获取用户的特定订单

# 过滤和分页
GET  /users?page=2&per_page=20
GET  /users?status=active&sort=created_at
```

### 状态码使用

```http
# 成功
GET   /users/123     -> 200 OK
POST  /users         -> 201 Created
PUT   /users/123     -> 200 OK 或 204 No Content
DELETE /users/123    -> 204 No Content

# 错误
POST  /users         -> 400 Bad Request (格式错误)
GET   /users/999     -> 404 Not Found
POST  /users         -> 422 Unprocessable Entity (验证失败)
```

## Cookie与Session

### Cookie机制

```http
# 服务器设置Cookie
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=3600

# 客户端后续请求自动发送
Cookie: session_id=abc123
```

### Session管理流程

```
1. 用户登录
2. 服务器创建Session，存储在内存/Redis
3. 服务器生成Session ID，通过Cookie返回客户端
4. 后续请求，Cookie自动携带Session ID
5. 服务器通过Session ID查找用户信息
```

### HttpOnly vs Secure

```
HttpOnly: JavaScript无法通过 document.cookie 访问
Secure: 仅在HTTPS连接发送
SameSite: 防止CSRF攻击
```

## 常用工具

### curl示例

```bash
# 基本GET请求
curl https://api.example.com/users

# POST请求
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name":"张三"}'

# 带认证
curl -H "Authorization: Bearer <token>" \
  https://api.example.com/protected

# 下载文件
curl -O https://example.com/file.zip

# 跟随重定向
curl -L https://example.com

# 显示响应头
curl -I https://example.com
```

## 参考文献

[^1]: RFC 9110 - HTTP Semantics
[^2]: RFC 9111 - HTTP Caching
[^3]: MDN Web Docs - HTTP

---

*本页面内容基于HTTP协议标准RFC文档整理*
