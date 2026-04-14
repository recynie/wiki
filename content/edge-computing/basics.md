---
title: 边缘计算架构与实现
date: 2026-04-14
description: 边缘计算（Edge Computing）是一种将计算、存储和网络服务推向网络边缘的分布式计算范式，旨在减少延迟、节省带宽并提升应用响应速度。
tags:
  - edge-computing
  - distributed-systems
  - cloudflare-workers
  - serverless
draft: false
permalink:
---

## 一、定义：什么是边缘计算

边缘计算（Edge Computing）是一种分布式计算范式，其核心思想是将计算、存储和网络服务从传统的数据中心（centralized cloud）推向更接近用户设备和数据源的「边缘」位置。[^1]

### 传统云计算 vs 边缘计算

| 特性 | 传统云计算 | 边缘计算 |
|------|-----------|----------|
| 延迟 | 100-200ms（往返） | <10ms（本地处理） |
| 带宽 | 需上传完整数据 | 边缘过滤、压缩 |
| 可用性 | 依赖网络连接 | 支持离线/弱网 |
| 数据主权 | 数据集中在云端 | 数据在本地保留 |
| 适用场景 | 复杂分析、存储 | 实时响应、缓存 |

传统的云端架构要求所有客户端请求必须先到达远程数据中心，经过处理后再返回响应。这种模式在面对[[../distributed-systems/distributed-systems|分布式系统]]中的低延迟需求时显得力不从心。边缘计算通过在地理上分散的边缘节点处理请求，打破了这一瓶颈。

### 边缘计算的核心价值

1. **低延迟**：物理距离的缩短使得响应时间从数百毫秒降低到个位数毫秒
2. **带宽节省**：仅传输必要数据，减少核心网络负载
3. **隐私合规**：敏感数据可在本地处理，无需上传云端
4. **高可用性**：即使与云端的连接中断，边缘节点仍能提供核心服务

---

## 二、架构模式

### 2.1 边缘原生（Edge-Native）vs 边缘部署（Edge-Deployed）

**边缘原生**应用从一开始就是为边缘环境设计的，充分利用边缘的特性：

```yaml
# 边缘原生架构示例
architecture:
  design_principle: "Edge-First"
  data_strategy: "Edge-First, Cloud-Backup"
  compute_model: "Event-driven, Stateless"
  examples:
    - Cloudflare Workers
    - Vercel Edge Functions
    - Deno Deploy
```

**边缘部署**则是将传统云端应用的部分功能迁移到边缘：

```yaml
# 边缘部署架构示例
architecture:
  design_principle: "Cloud-Core, Edge-Extension"
  data_strategy: "Centralized, CDN-Cached"
  compute_model: "Hybrid (Cloud + Edge)"
  patterns:
    - CDN caching
    - Stale-While-Revalidate
    - Edge Side Includes (ESI)
```

#### CDN 缓存模式

CDN（Content Delivery Network）是最常见的边缘部署模式。通过在全球部署的边缘节点缓存静态资源和部分动态内容，实现就近访问：

```yaml
# CDN 缓存配置示例
cache_policy:
  static_assets:
    ttl: 31536000  # 1 year for immutable assets
    patterns:
      - "/*.css"
      - "/*.js"
      - "/*.ico"
      - "/assets/*"
  dynamic_content:
    ttl: 3600      # 1 hour with revalidation
    stale_while_revalidate: 86400  # 24 hours grace period
```

### 2.2 两层脑模型（Two-Tier Brain Model）

边缘计算架构常采用「反射-推理」的两层脑模型来分配计算任务：

```
┌─────────────────────────────────────────────────────────────┐
│                        云端（推理脑）                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  复杂分析     │  │  机器学习    │  │  全局协调与策略      │  │
│  │  模型训练    │  │  推理（延迟容忍）│  │  配置下发            │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                        网络通信（可能有延迟）
                              │
┌─────────────────────────────────────────────────────────────┐
│                       边缘（反射脑）                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  实时响应    │  │  数据过滤    │  │  身份验证（简单）     │  │
│  │  <10ms      │  │  压缩加密    │  │  路由重写            │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

- **边缘（反射脑）**：处理需要即时响应（<10ms）的任务，如请求路由、认证、缓存命中、A/B 测试分流等
- **云端（推理脑）**：处理复杂但延迟容忍的分析、模型训练、全局配置计算等

这种模型与[[../distributed-systems/raft-consensus|Raft 共识算法]]中的领导者与跟随者角色有相似之处——边缘像跟随者处理本地快速操作，而云端像领导者进行全局协调。

### 2.3 常见架构模式

#### 模式一：边缘中间件 + 中心源（Edge Middleware + Centralized Origin）

```
用户请求 → 边缘节点 → [缓存命中?] → 是 → 直接返回
                          ↓ 否
                    中心源获取 → 边缘缓存 → 返回给用户
```

#### 模式二：过期重验证（Stale-While-Revalidate）

允许边缘在缓存过期后立即返回过期数据（stale），同时在后台异步从源站获取最新数据并更新缓存：

```yaml
# Cloudflare Cache-API 配置示例
cache_rules:
  - name: "Stale-While-Revalidate for API"
    path: "/api/*"
    cache_ttl: 60
    stale_while_revalidate: 300
    serve_stale_on_error: true
```

#### 模式三：边缘渲染（Edge Rendering）

在边缘节点执行服务端渲染（SSR），结合[[../frontend/react|React]]或 Vue 的[[../frontend/nextjs|Next.js]]框架实现：

```javascript
// Vercel Edge Function 示例
export const config = {
  runtime: 'edge',
};

export default function handler(req) {
  const url = new URL(req.url);
  const locale = req.headers.get('x-locale') || 'en';
  
  // 在边缘生成个性化内容
  const html = renderPage({
    locale,
    timestamp: Date.now(),
    edgeLocation: getEdgeLocation(req),
  });
  
  return new Response(html, {
    headers: { 'content-type': 'text/html' },
  });
}
```

---

## 三、平台现状（2026年）

### 3.1 主要平台对比

| 平台 | 技术栈 | 全球 PoPs | 冷启动 | 内存限制 | 最大执行时间 | 特色 |
|------|--------|-----------|--------|----------|--------------|------|
| **Cloudflare Workers** | V8 Isolates | 200+ | <5ms | 128MB | 50ms（CPU时间） | KV 存储、 Durable Objects |
| **Vercel Edge Functions** | V8 Isolates | 126 | <5ms | 128MB | 无限制（响应时间） | Next.js 原生集成 |
| **AWS Lambda@Edge** | Node.js | 225+ | >100ms | 128MB（Origin） | 30秒 | CloudFront 深度集成 |
| **CloudFront Functions** | JavaScript | 225+ | <1ms | 2MB | 1ms（CPU时间） | 超低延迟，查看器请求阶段 |
| **Deno Deploy** | V8 Isolates | 35+ | <5ms | 512MB | 50ms（CPU时间） | 原生 TypeScript |
| **Fastly Compute** | Wasm/JS | 50+ | <1ms | 256MB | 30秒 | Wasm 支持，低级控制 |

### 3.2 Cloudflare Workers 架构详解

Cloudflare Workers 是目前最流行的边缘计算平台之一，其核心技术基于 V8 JavaScript 引擎的轻量级隔离机制：

```javascript
// Cloudflare Worker 完整示例：请求路由 + KV 读取 + 缓存
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    
    // 1. 边缘中间件：请求路径重写
    if (url.pathname.startsWith('/api/users/')) {
      const userId = url.pathname.split('/')[3];
      return handleUserRequest(userId, request, env);
    }
    
    // 2. 缓存优先策略
    const cacheKey = new Request(url.toString(), request);
    const cache = caches.default;
    const cachedResponse = await cache.match(cacheKey);
    
    if (cachedResponse) {
      // 缓存命中，添加 Age 头用于调试
      const age = parseInt(cachedResponse.headers.get('Age') || '0');
      const response = new Response(cachedResponse.body, cachedResponse);
      response.headers.set('X-Cache', `HIT (age: ${age}s)`);
      return response;
    }
    
    // 3. 缓存未命中，从源站获取
    const originResponse = await fetch(request);
    
    // 仅缓存 GET 请求，且响应成功
    if (request.method === 'GET' && originResponse.ok) {
      const response = new Response(originResponse.body, originResponse);
      response.headers.set('Cache-Control', 'public, max-age=3600');
      await cache.put(cacheKey, response.clone());
      return response;
    }
    
    return originResponse;
  },
};

// Worker KV 操作示例
async function handleUserRequest(userId, request, env) {
  const cacheKey = `user:${userId}`;
  
  // 尝试从 KV 读取
  let userData = await env.USER_KV.get(cacheKey, 'json');
  
  if (!userData) {
    // KV 未命中，从 API 获取并写入
    const apiResponse = await fetch(`https://api.example.com/users/${userId}`);
    userData = await apiResponse.json();
    await env.USER_KV.put(cacheKey, JSON.stringify(userData), { expirationTtl: 3600 });
  }
  
  return new Response(JSON.stringify(userData), {
    headers: { 'Content-Type': 'application/json' },
  });
}
```

### 3.3 Vercel Edge Functions

Vercel Edge Functions 与[[../frontend/nextjs|Next.js]]深度集成，支持边缘服务端渲染：

```typescript
// Vercel Edge Middleware：全球认证与 A/B 测试
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export const config = {
  matcher: '/((?!_next/static|_next/image|favicon.ico).*)',
};

export function middleware(request: NextRequest) {
  const url = request.nextUrl.clone();
  
  // 1. 地理位置路由
  const country = request.geo?.country || 'US';
  if (country === 'CN' && !url.pathname.startsWith('/cn/')) {
    url.pathname = `/cn${url.pathname}`;
    return NextResponse.redirect(url);
  }
  
  // 2. A/B 测试分流
  const bucket = getBucket(request);
  url.searchParams.set('variant', bucket);
  
  const response = NextResponse.rewrite(url);
  response.headers.set('X-AB-Variant', bucket);
  
  return response;
}

function getBucket(request: NextRequest): string {
  // 基于 Cookie 或随机分配
  const cookie = request.cookies.get('ab_bucket');
  if (cookie) return cookie.value;
  
  const bucket = Math.random() < 0.5 ? 'control' : 'experiment';
  return bucket;
}
```

---

## 四、实战实现

### 4.1 CloudFront + Lambda@Edge 完整配置

使用 Terraform（HCL）创建完整的 CloudFront 分配，配合 Lambda@Edge 实现边缘逻辑：

```hcl
# Terraform 配置文件：CloudFront + Lambda@Edge
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Lambda@Edge 函数定义
resource "aws_lambda_function" "edge_auth" {
  filename         = "lambda_edge_auth.zip"
  function_name    = "edge-auth-function"
  role             = aws_iam_role.lambda_edge_role.arn
  handler          = "index.handler"
  runtime          = "nodejs20.x"
  source_code_hash = filebase64sha256("lambda_edge_auth.zip")
  
  # Lambda@Edge 必须部署在 us-east-1
  provider = aws.us_east_1
  
  memory_size = 128
  timeout     = 5
  
  environment {
    variables = {
      ALLOWED_ORIGINS = "example.com,www.example.com"
      AUTH_SECRET     = var.auth_secret
    }
  }
}

# IAM 角色：Lambda@Edge 执行角色
resource "aws_iam_role" "lambda_edge_role" {
  name = "lambda-edge-execution-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      },
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "edgelambda.amazonaws.com"
        }
      }
    ]
  })
}

# CloudFront 分配
resource "aws_cloudfront_distribution" "main" {
  origin {
    domain_name = aws_lb.main.dns_name
    origin_id   = "main-origin"
    
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }
  
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "Main distribution with Lambda@Edge"
  default_root_object = "index.html"
  
  # 缓存行为配置
  default_cache_behavior {
    target_origin_id       = "main-origin"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD"]
    compress               = true
    
    # Lambda@Edge 关联：查看器请求阶段
    lambda_function_association {
      lambda_arn   = aws_lambda_function.edge_auth.arn
      event_type   = "viewer-request"
      include_body = false
    }
    
    # Lambda@Edge 关联：源请求阶段（缓存键修改）
    lambda_function_association {
      lambda_arn   = aws_lambda_function.edge_auth.arn
      event_type   = "origin-request"
      include_body = false
    }
    
    forwarded_values {
      query_string = true
      cookies {
        forward = "all"
      }
      headers = ["Accept", "Accept-Language", "Authorization"]
    }
    
    min_ttl     = 0
    default_ttl = 3600
    max_ttl     = 86400
  }
  
  # 价格等级：全球分发
  price_class = "PriceClass_All"
  
  # SSL 证书
  viewer_certificate {
    cloudfront_default_certificate = true
  }
  
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
}

# 自定义错误响应：边缘处理 404
resource "aws_cloudfront_distribution" "main" {
  # ... previous config ...
  
  custom_error_response {
    error_code         = 404
    response_code      = 200
    response_page_path = "/404.html"
    error_caching_min_ttl = 300
  }
}
```

### 4.2 Cloudflare Workers KV 全局数据存储

Workers KV 是 Cloudflare 提供的全局键值存储，读取速度极快（<10ms）：

```javascript
// KV 数据存储完整示例
class UserSessionStore {
  constructor(kvNamespace) {
    this.kv = kvNamespace;
    this.SESSION_TTL = 86400; // 24 hours
  }
  
  // 创建会话
  async createSession(userId, metadata = {}) {
    const sessionId = crypto.randomUUID();
    const session = {
      id: sessionId,
      userId,
      createdAt: Date.now(),
      lastAccessed: Date.now(),
      metadata,
    };
    
    await this.kv.put(`session:${sessionId}`, JSON.stringify(session), {
      expirationTtl: this.SESSION_TTL,
    });
    
    // 索引：按用户查找会话
    await this.kv.put(`user:${userId}:sessions`, sessionId, {
      expirationTtl: this.SESSION_TTL,
    });
    
    return sessionId;
  }
  
  // 验证会话
  async validateSession(sessionId) {
    const session = await this.kv.get(`session:${sessionId}`, 'json');
    
    if (!session) {
      return { valid: false, reason: 'SESSION_NOT_FOUND' };
    }
    
    const now = Date.now();
    const maxIdleTime = 30 * 60 * 1000; // 30 分钟空闲超时
    
    if (now - session.lastAccessed > maxIdleTime) {
      await this.invalidateSession(sessionId);
      return { valid: false, reason: 'SESSION_EXPIRED' };
    }
    
    // 更新最后访问时间
    session.lastAccessed = now;
    await this.kv.put(`session:${sessionId}`, JSON.stringify(session), {
      expirationTtl: this.SESSION_TTL,
    });
    
    return { valid: true, session };
  }
  
  // 作废会话
  async invalidateSession(sessionId) {
    const session = await this.kv.get(`session:${sessionId}`, 'json');
    if (session) {
      await this.kv.delete(`session:${sessionId}`);
      await this.kv.delete(`user:${session.userId}:sessions`);
    }
  }
}

// 使用示例
export default {
  async fetch(request, env) {
    const store = new UserSessionStore(env.SESSIONS);
    const url = new URL(request.url);
    
    if (url.pathname === '/login' && request.method === 'POST') {
      // 登录逻辑
      const { userId } = await request.json();
      const sessionId = await store.createSession(userId, {
        ip: request.headers.get('CF-Connecting-IP'),
        country: request.headers.get('CF-IPCountry'),
      });
      
      const response = new Response(JSON.stringify({ success: true }), {
        status: 200,
      });
      response.headers.set('Set-Cookie', `session=${sessionId}; HttpOnly; Secure; SameSite=Strict`);
      return response;
    }
    
    if (url.pathname === '/api/protected') {
      const sessionCookie = request.headers.get('Cookie')?.match(/session=([^;]+)/)?.[1];
      if (!sessionCookie) {
        return new Response('Unauthorized', { status: 401 });
      }
      
      const validation = await store.validateSession(sessionCookie);
      if (!validation.valid) {
        return new Response('Invalid session', { status: 401 });
      }
      
      return new Response(JSON.stringify({ user: validation.session.userId }), {
        headers: { 'Content-Type': 'application/json' },
      });
    }
    
    return new Response('Not Found', { status: 404 });
  },
};
```

### 4.3 DynamoDB 全局表：边缘友好的数据访问

AWS DynamoDB Global Tables 提供多区域复制，使数据在边缘位置也能快速访问：

```typescript
// DynamoDB 全局表客户端配置
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, GetCommand, PutCommand } from '@aws-sdk/lib-dynamodb';

class EdgeDataStore {
  constructor() {
    // 使用边缘优化的区域路由
    this.client = new DynamoDBClient({
      region: 'auto', // 自动路由到最近区域
      endpoint: process.env.DYNAMODB_ENDPOINT, // 局部端点用于测试
    });
    
    this.docClient = DynamoDBDocumentClient.from(this.client, {
      marshallOptions: {
        removeUndefinedValues: true,
        convertEmptyValues: false,
      },
    });
    
    this.tableName = 'edge-user-data';
  }
  
  // 读取用户配置（边缘优先）
  async getUserPreferences(userId) {
    const result = await this.docClient.send(
      new GetCommand({
        TableName: this.tableName,
        Key: {
          pk: `USER#${userId}`,
          sk: 'PREFERENCES',
        },
        // 边缘节点直接读取本地副本
        ConsistentRead: false, // 最终一致性，更快
      })
    );
    
    return result.Item || null;
  }
  
  // 写入用户操作日志（异步复制到其他区域）
  async logUserAction(userId, action, metadata) {
    const item = {
      pk: `USER#${userId}`,
      sk: `ACTION#${Date.now()}`,
      action,
      metadata,
      ttl: Math.floor(Date.now() / 1000) + 30 * 24 * 60 * 60, // 30天后过期
      // 用于 GSI 查询
      gsipk: `USER#${userId}`,
      gsisk: action,
    };
    
    await this.docClient.send(
      new PutCommand({
        TableName: this.tableName,
        Item: item,
      })
    );
    
    return { success: true };
  }
}

export { EdgeDataStore };
```

```yaml
# DynamoDB 全局表 Terraform 配置
resource "aws_dynamodb_table" "edge_user_data" {
  name           = "edge-user-data"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "pk"
  range_key      = "sk"
  
  attribute {
    name = "pk"
    type = "S"
  }
  
  attribute {
    name = "sk"
    type = "S"
  }
  
  # 全局二级索引：按用户+操作类型查询
  global_secondary_index {
    name            = "gsi-user-actions"
    hash_key        = "gsipk"
    range_key       = "gsisk"
    projection_type = "ALL"
  }
  
  point_in_time_recovery {
    enabled = true
  }
  
  ttl {
    attribute_name = "ttl"
    enabled        = true
  }
}

# 启用 DynamoDB Streams 用于触发 Lambda@Edge 关联
resource "aws_dynamodb_table" "edge_user_data" {
  # ... previous config ...
  
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"
}
```

---

## 五、边缘原生 API 网关

边缘原生 API 网关将传统的中心化 API Gateway 功能下沉到边缘，实现全球一致的认证、限流和路由策略，同时保持亚 50ms 的端到端延迟。

### 5.1 核心架构

```
全球用户
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│                   边缘节点（200+ PoPs）                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ 认证鉴权  │  │  限流    │  │  路由    │  │  缓存    │   │
│  │ JWT验证  │  │ Token桶  │  │ 路径匹配  │  │ KV存储   │   │
│  │ 签权验证  │  │ 地域限制  │  │ 版本管理  │  │ 响应缓存  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└────────────────────────────────────────────────────────────┘
    │                                               │
    │ 鉴权通过 + 限流通过                            │ 限流触发/异常
    ▼                                               ▼
┌───────────────┐                          ┌────────────────┐
│   后端服务     │                          │   边缘直接响应   │
│ (Lambda/ECS) │                          │   (429/401)    │
└───────────────┘                          └────────────────┘
```

### 5.2 边缘限流实现

```javascript
// 边缘限流中间件：令牌桶算法
class EdgeRateLimiter {
  constructor(kvNamespace, options = {}) {
    this.kv = kvNamespace;
    this.tokensPerSecond = options.tokensPerSecond || 10;
    this.maxBurst = options.maxBurst || 20;
    this.windowSeconds = options.windowSeconds || 60;
  }
  
  async isAllowed(clientId) {
    const key = `ratelimit:${clientId}`;
    const now = Date.now();
    const windowMs = this.windowSeconds * 1000;
    
    // 读取当前限流状态
    let state = await this.kv.get(key, 'json');
    
    if (!state) {
      // 初始化状态
      state = {
        tokens: this.maxBurst,
        lastRefill: now,
      };
    }
    
    // 补充令牌
    const elapsed = now - state.lastRefill;
    const tokensToAdd = (elapsed / 1000) * this.tokensPerSecond;
    state.tokens = Math.min(this.maxBurst, state.tokens + tokensToAdd);
    state.lastRefill = now;
    
    // 检查是否允许请求
    if (state.tokens < 1) {
      const retryAfter = Math.ceil((1 - state.tokens) / this.tokensPerSecond);
      return {
        allowed: false,
        remaining: 0,
        retryAfter,
        limit: this.maxBurst,
      };
    }
    
    // 消耗一个令牌
    state.tokens -= 1;
    await this.kv.put(key, JSON.stringify(state), {
      expirationTtl: this.windowSeconds,
    });
    
    return {
      allowed: true,
      remaining: Math.floor(state.tokens),
      retryAfter: 0,
      limit: this.maxBurst,
    };
  }
}

// 边缘认证中间件
class EdgeAuthenticator {
  constructor(kvNamespace, secretKey) {
    this.kv = kvNamespace;
    this.secretKey = secretKey;
  }
  
  async authenticate(request) {
    const authHeader = request.headers.get('Authorization');
    if (!authHeader?.startsWith('Bearer ')) {
      return { valid: false, error: 'MISSING_TOKEN', status: 401 };
    }
    
    const token = authHeader.slice(7);
    
    // 验证 JWT
    const payload = await this.verifyJWT(token);
    if (!payload) {
      return { valid: false, error: 'INVALID_TOKEN', status: 401 };
    }
    
    // 检查 token 是否在撤销列表中
    const isRevoked = await this.kv.get(`revoked:${payload.jti}`, 'json');
    if (isRevoked) {
      return { valid: false, error: 'TOKEN_REVOKED', status: 401 };
    }
    
    return { valid: true, user: payload.sub, scopes: payload.scope };
  }
  
  async verifyJWT(token) {
    try {
      const [header, payload, signature] = token.split('.');
      const expectedSig = await this.computeSignature(header, payload);
      
      if (signature !== expectedSig) {
        return null;
      }
      
      const claims = JSON.parse(atob(payload));
      
      // 检查过期时间
      if (claims.exp && claims.exp < Date.now() / 1000) {
        return null;
      }
      
      return claims;
    } catch {
      return null;
    }
  }
  
  async computeSignature(header, payload) {
    const data = `${header}.${payload}`;
    const key = await crypto.subtle.importKey(
      'raw',
      new TextEncoder().encode(this.secretKey),
      { name: 'HMAC', hash: 'SHA-256' },
      false,
      ['sign']
    );
    const signature = await crypto.subtle.sign('HMAC', key, new TextEncoder().encode(data));
    return btoa(String.fromCharCode(...new Uint8Array(signature)));
  }
}

// 完整的边缘网关中间件
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const limiter = new EdgeRateLimiter(env.RATE_LIMIT_KV, {
      tokensPerSecond: 100,
      maxBurst: 200,
    });
    const auth = new EdgeAuthenticator(env.AUTH_KV, env.JWT_SECRET);
    
    // 提取客户端标识（优先使用 API Key，其次 IP）
    const clientId = request.headers.get('X-API-Key') 
      || request.headers.get('CF-Connecting-IP')
      || 'anonymous';
    
    // 1. 限流检查
    const rateLimitResult = await limiter.isAllowed(clientId);
    if (!rateLimitResult.allowed) {
      return new Response(JSON.stringify({
        error: 'RATE_LIMIT_EXCEEDED',
        retryAfter: rateLimitResult.retryAfter,
      }), {
        status: 429,
        headers: {
          'Retry-After': rateLimitResult.retryAfter.toString(),
          'X-RateLimit-Limit': rateLimitResult.limit.toString(),
          'X-RateLimit-Remaining': '0',
        },
      });
    }
    
    // 2. 认证检查（仅对 /api/* 路径）
    if (url.pathname.startsWith('/api/')) {
      const authResult = await auth.authenticate(request);
      if (!authResult.valid) {
        return new Response(JSON.stringify({ error: authResult.error }), {
          status: authResult.status,
        });
      }
      
      // 将用户信息注入到请求头中，传递给后端
      request = new Request(request, {
        headers: {
          ...Object.fromEntries(request.headers),
          'X-User-Id': authResult.user,
          'X-User-Scopes': authResult.scopes,
        },
      });
    }
    
    // 3. 路由到后端服务
    const backendResponse = await fetch(request);
    
    // 4. 添加限流头到响应
    const response = new Response(backendResponse.body, backendResponse);
    response.headers.set('X-RateLimit-Limit', rateLimitResult.limit.toString());
    response.headers.set('X-RateLimit-Remaining', rateLimitResult.remaining.toString());
    
    return response;
  },
};
```

---

## 六、边缘-云端协同架构

### 6.1 六大核心组件

一个完整的边缘-云端协同系统通常包含以下六个核心组件：

```
┌─────────────────────────────────────────────────────────────┐
│                    1. 全球流量管理                           │
│         (DNS 智能解析 + Anycast 路由 + 健康检查)             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    2. 边缘计算层                              │
│      (CDN + Edge Functions + Edge KV/Wasm Storage)          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ 内容缓存  │  │ 请求路由  │  │ 认证鉴权  │  │ A/B测试  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              │  数据同步策略                    │
              │  (CRDT / 事件溯源 / 异步复制)    │
              ▼                               ▼
┌─────────────────────────┐    ┌─────────────────────────────────┐
│     3. 边缘存储层          │    │      4. 云端数据湖               │
│  (KV/Redis/对象存储)     │    │  (S3/GCS + Data Lake)          │
│  - 热点数据缓存           │    │  - 完整历史数据                  │
│  - 用户会话               │    │  - 分析与训练                     │
└─────────────────────────┘    └─────────────────────────────────┘
              │                               │
              │         ┌─────────────────────┘
              ▼         ▼
┌─────────────────────────────────────────────────────────────┐
│                    5. 边缘-云端网络                           │
│       (专用骨干网 / 异步通道 / 数据压缩传输)                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    6. 统一管控平面                           │
│        (配置下发 / 监控告警 / 策略管理 / 密钥分发)            │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 典型设计模式

#### 模式一：CQRS + 边缘缓存

命令（写入）在边缘处理并异步同步到云端，查询（读取）直接在边缘满足：

```javascript
// CQRS 边缘实现
class EdgeCQRS {
  constructor(commandKV, queryKV) {
    this.commandStore = commandKV; // 写入
    this.queryStore = queryKV;     // 读取
  }
  
  // 命令：边缘写入
  async handleCommand(command) {
    const { type, payload, aggregateId } = command;
    
    // 生成事件
    const event = {
      id: crypto.randomUUID(),
      type,
      payload,
      aggregateId,
      timestamp: Date.now(),
      version: await this.getNextVersion(aggregateId),
    };
    
    // 写入命令存储
    await this.commandStore.put(
      `events:${aggregateId}:${event.id}`,
      JSON.stringify(event),
      { expirationTtl: 30 * 24 * 60 * 60 } // 30天保留
    );
    
    // 立即更新查询视图（读写分离）
    await this.updateQueryView(aggregateId);
    
    // 异步同步到云端（通过 Queue）
    await this.syncToCloud(event);
    
    return { success: true, eventId: event.id };
  }
  
  // 查询：边缘直接读取
  async handleQuery(aggregateId) {
    return await this.queryStore.get(`view:${aggregateId}`, 'json');
  }
  
  async updateQueryView(aggregateId) {
    // 重建聚合视图
    const events = await this.loadEvents(aggregateId);
    const view = this.rebuildAggregate(events);
    await this.queryStore.put(`view:${aggregateId}`, JSON.stringify(view));
  }
}
```

#### 模式二：边缘优先的乐观锁定

```javascript
// 乐观锁定实现
class OptimisticLocking {
  constructor(kvStore) {
    this.store = kvStore;
  }
  
  async updateWithLock(key, updateFn, maxRetries = 3) {
    for (let attempt = 0; attempt < maxRetries; attempt++) {
      const item = await this.store.get(key, 'json');
      const originalVersion = item?.version || 0;
      
      const updated = updateFn(item);
      updated.version = originalVersion + 1;
      
      // 使用 CAS（Compare-And-Swap）
      const success = await this.store.put(
        `${key}:lock`,
        JSON.stringify({ version: updated.version, data: updated }),
        { onlyIf: { version: originalVersion } }
      );
      
      if (success) {
        return updated;
      }
      
      // 版本冲突，等待后重试
      await new Promise(r => setTimeout(r, 50 * (attempt + 1)));
    }
    
    throw new Error('MAX_RETRIES_EXCEEDED');
  }
}
```

---

## 七、典型应用场景

### 7.1 IoT（物联网）

边缘计算在 IoT 场景中的核心价值在于处理海量传感器数据的实时响应：

```yaml
# IoT 边缘计算架构示例
iot_edge_architecture:
  data_flow:
    sensors:
      - temperature_sensors
      - vibration_sensors
      - cameras
    edge_node:
      - data_filtering: "remove noise and outliers"
      - aggregation: "1-minute windows"
      - anomaly_detection: "threshold-based alerts"
      - local_control: "emergency shutdown"
    cloud:
      - model_training: "ML on aggregated data"
      - historical_analysis: "trend analysis"
      - config_update: "push new rules to edge"
  
  latency_requirements:
    emergency_shutdown: "<5ms"
    alarm_trigger: "<100ms"
    data_upload_interval: "1-60 seconds"
```

### 7.2 媒体分发

```yaml
# 媒体分发边缘优化
media_delivery:
  video_streaming:
    cdn_strategy: "multi-bitrate adaptive streaming"
    edge_processing:
      - thumbnail_generation
      - format_conversion
      - drm_license_delivery
    cache_policy:
      popular_content: "TTL=24h, preload on predictive basis"
      long_tail: "on-demand fetch from origin"
  
  live_streaming:
    edge_ingest: "RTMP/HLS ingest at nearest PoP"
    transcoding: "distributed transcode clusters"
    distribution: "HLS/DASH via global CDN"
```

### 7.3 AI 推理

边缘 AI 推理是近年发展最快的场景之一，典型应用包括：

```yaml
# 边缘 AI 推理架构
ai_edge_inference:
  model_deployment:
    model_formats:
      - TensorFlow Lite
      - ONNX Runtime
      - WebNN
    optimization:
      - quantization: "INT8"
      - pruning: "sparse weights"
      - distillation: "smaller student models"
  
  inference_pipeline:
    input_preprocessing: "resize, normalize (edge)"
    model_inference: "TFLite runtime (edge)"
    output_postprocessing: "NMS, filtering (edge)"
    cloud_fallback: "complex cases delegated to cloud"
  
  use_cases:
    - image_classification: "quality inspection"
    - speech_recognition: "voice assistant"
    - object_detection: "autonomous vehicles"
    - nlp: "real-time translation"
```

### 7.4 全球 API 分发

```yaml
# 全球 API 分发抖动架构
global_api:
  edge_layer:
    authentication: "JWT verification, API key validation"
    rate_limiting: "per-client, per-endpoint"
    request_routing: "geo-based, latency-based"
    response_caching: "GET responses, 304 handling"
  
  backend_layer:
    primary_region: "us-east-1 (or nearest to origin)"
    read_replicas: "global cross-region replication"
    write_forwarding: "writes routed to primary"
  
  consistency:
    strategy: "eventual consistency for reads"
    conflict_resolution: "last-write-wins or custom"
    sync_protocol: "async, batched"
```

---

## 八、权衡与注意事项

### 8.1 边缘计算适合的场景

✅ **强烈适合**：

- 需要亚秒级响应的用户交互（如认证、路由、A/B 测试）
- 高访问量的静态内容分发
- 全球用户的会话管理
- 地理位置敏感的路由和个性化
- 降低核心网络带宽成本

### 8.2 边缘计算的局限性

❌ **不太适合**：

- **需要强一致性的事务**：边缘节点之间的数据同步存在延迟
- **计算密集型任务**：边缘节点资源受限（内存 128MB，CPU 时间限制）
- **大型数据处理**：不适合进行大规模数据分析或机器学习训练
- **频繁写入的数据**：每次写入都需要考虑与云端同步的问题
- **需要共享状态的复杂逻辑**：边缘函数的无状态特性使共享状态复杂化

### 8.3 数据局部性考量

```yaml
# 数据局部性决策框架
data_locality_decision:
  questions:
    - question: "数据敏感度如何？"
      factors:
        - PII/隐私数据: "优先边缘处理，不上传云端"
        - 业务数据: "可接受边缘缓存"
        - 公开数据: "完全可边缘缓存"
    
    - question: "一致性要求？"
      factors:
        - 强一致性: "写操作必须到云端主库"
        - 最终一致性: "边缘异步同步可接受"
        - 无一致性: "边缘直接处理"
    
    - question: "数据量大小？"
      factors:
        - 小（<1MB）: "可完全存放在边缘 KV"
        - 中（1-100MB）: "边缘缓存热点数据"
        - 大（>100MB）: "边缘只存索引，源在云端"
    
    - question: "访问模式？"
      factors:
        - 读多写少: "边缘缓存非常适合"
        - 写多读少: "考虑云端优先"
        - 读写均衡: "需要仔细设计同步策略"
```

---

## 九、参考资料

[^1]: 本段参考 [Edge Computing - Wikipedia](https://en.wikipedia.org/wiki/Edge_computing)

### 官方文档

| 平台 | 文档链接 |
|------|---------|
| Cloudflare Workers | https://developers.cloudflare.com/workers/ |
| Cloudflare Workers KV | https://developers.cloudflare.com/kv/ |
| Vercel Edge Functions | https://vercel.com/docs/edge-functions |
| AWS Lambda@Edge | https://docs.aws.amazon.com/lambda/latest/dg/lambda-edge.html |
| CloudFront Functions | https://docs.aws.amazon.com/cloudfront/latest/DeveloperGuide/cloudfront-functions.html |
| AWS DynamoDB Global Tables | https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html |
| Deno Deploy | https://docs.deno.com/deploy/ |
| Fastly Compute | https://developer.fastly.com/ |

### 延伸阅读

- [[../distributed-systems/distributed-systems|分布式系统核心概念]]
- [[../distributed-systems/raft-consensus|Raft 共识算法]]
- [[../devops/kubernetes|Kubernetes 容器编排]]
- [[../databases/redis|Redis 缓存实战]]

