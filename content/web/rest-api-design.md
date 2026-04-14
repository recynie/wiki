---
title: REST API设计
date: 2026-04-09
description: RESTful API设计原则、规范与最佳实践
tags:
  - rest-api
  - web
  - api-design
draft: false
permalink:
---

# REST API设计

REST（Representational State Transfer）是现代Web服务的主流架构风格。本节介绍REST API的设计原则、规范和最佳实践。

## 1. REST核心原则

### 1.1 六大原则

| 原则 | 说明 |
|------|------|
| **客户端-服务器** | 分离关注点，客户端不关心数据存储 |
| **无状态** | 每个请求包含所有必要信息 |
| **可缓存** | 响应可标记为可缓存或不可缓存 |
| **分层系统** | 允许中间层（负载均衡、缓存） |
| **统一接口** | 资源通过URL定位 |
| **按需代码** | 服务器可返回可执行代码 |

### 1.2 资源命名

**资源 vs 操作**：

| 风格 | 示例 | 说明 |
|------|------|------|
| 资源导向 | `DELETE /users/123` | 删除用户 |
| 操作导向 | `/users/delete?id=123` | ❌ 不推荐 |

**URL结构规范**：
```
https://api.example.com/v1/users/123/orders

- /v1：版本号
- /users：资源集合（复数名词）
- /123：特定资源
- /orders：子资源
```

---

## 2. HTTP方法与语义

### 2.1 方法对照表

| 方法 | 语义 | 幂等性 | 安全性 | 用途 |
|------|------|--------|--------|------|
| GET | 读取资源 | ✓ | ✓ | 查询 |
| POST | 创建资源 | ✗ | ✗ | 创建 |
| PUT | 完整更新 | ✓ | ✗ | 全量更新 |
| PATCH | 部分更新 | ✗ | ✗ | 增量更新 |
| DELETE | 删除资源 | ✓ | ✗ | 删除 |

### 2.2 设计示例

```bash
# 创建用户
POST /v1/users
Body: {"name": "张三", "email": "zhang@example.com"}
Response: 201 Created
         {"id": 1, "name": "张三", "email": "zhang@example.com"}

# 获取用户
GET /v1/users/1
Response: 200 OK
         {"id": 1, "name": "张三", "email": "zhang@example.com"}

# 更新用户（完整）
PUT /v1/users/1
Body: {"name": "张三", "email": "new@example.com"}
Response: 200 OK

# 更新用户（部分）
PATCH /v1/users/1
Body: {"email": "new@example.com"}
Response: 200 OK

# 删除用户
DELETE /v1/users/1
Response: 204 No Content
```

### 2.3 状态码规范

| 分类 | 状态码 | 含义 |
|------|--------|------|
| **1xx** | 100-199 | 信息性状态码 |
| **2xx** | 200-299 | 成功 |
| | 200 | OK（GET/PUT/PATCH成功） |
| | 201 | Created（POST创建成功） |
| | 204 | No Content（DELETE成功，无返回体） |
| **3xx** | 300-399 | 重定向 |
| | 304 | Not Modified（缓存命中） |
| **4xx** | 400-499 | 客户端错误 |
| | 400 | Bad Request（参数错误） |
| | 401 | Unauthorized（未认证） |
| | 403 | Forbidden（无权限） |
| | 404 | Not Found（资源不存在） |
| | 409 | Conflict（资源冲突） |
| | 422 | Unprocessable Entity（验证失败） |
| | 429 | Too Many Requests（限流） |
| **5xx** | 500-599 | 服务端错误 |
| | 500 | Internal Server Error |
| | 503 | Service Unavailable |

---

## 3. 请求与响应设计

### 3.1 请求体设计

```json
// 创建订单请求
{
    "customer_id": 123,
    "items": [
        {"product_id": 1, "quantity": 2},
        {"product_id": 3, "quantity": 1}
    ],
    "shipping_address": {
        "city": "北京",
        "district": "朝阳区",
        "detail": "某街道123号"
    }
}
```

### 3.2 响应体设计

**成功响应**：
```json
{
    "data": {
        "id": 456,
        "status": "pending",
        "total": 299.00,
        "created_at": "2026-04-09T10:30:00Z"
    },
    "meta": {
        "request_id": "req_abc123"
    }
}
```

**分页响应**：
```json
{
    "data": [...],
    "pagination": {
        "page": 1,
        "page_size": 20,
        "total": 100,
        "total_pages": 5
    },
    "links": {
        "first": "/v1/orders?page=1",
        "prev": null,
        "next": "/v1/orders?page=2",
        "last": "/v1/orders?page=5"
    }
}
```

**错误响应**：
```json
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "请求参数验证失败",
        "details": [
            {
                "field": "email",
                "message": "邮箱格式不正确"
            },
            {
                "field": "quantity",
                "message": "数量必须大于0"
            }
        ]
    },
    "meta": {
        "request_id": "req_abc123"
    }
}
```

### 3.3 分页实现

```bash
# 页码分页
GET /v1/orders?page=2&page_size=20

# 偏移分页
GET /v1/orders?offset=40&limit=20

# 光标分页（适合大数据量）
GET /v1/orders?cursor=eyJpZCI6MTIzfQ&limit=20
Response: {
    "data": [...],
    "next_cursor": "eyJpZCI6MTQzfQ"
}
```

---

## 4. 版本管理

### 4.1 版本策略

| 策略 | 示例 | 优缺点 |
|------|------|--------|
| URL路径 | `/v1/users` | 明确直观，但改动大 |
| Query参数 | `/users?version=1` | 灵活，但不够明显 |
| Header | `API-Version: 1` | 干净，但需要客户端设置 |
| Content Negotiation | `Accept: application/vnd.api+v1` | RESTful，但复杂 |

**推荐**：URL路径（最常用、最直观）

### 4.2 版本演进

```bash
# v1: 初始版本
GET /v1/users/1
Response: {"id": 1, "name": "张三", "age": 25}

# v2: 新增字段，废弃字段
GET /v2/users/1
Response: {"id": 1, "name": "张三", "age": 25, "email": "zhang@example.com"}

# v1用户仍可访问，但返回deprecation警告
# Header: Deprecation: true
# Header: Sunset: Sat, 01 Jan 2027 00:00:00 GMT
```

---

## 5. 认证与授权

### 5.1 认证方案

| 方案 | 适用场景 | 说明 |
|------|----------|------|
| **API Key** | 内部服务间调用 | 简单密钥 |
| **Basic Auth** | 简单场景 | 用户名密码Base64编码 |
| **Bearer Token** | 标准方案 | OAuth 2.0 / JWT |
| **HMAC** | 高安全要求 | 请求签名防篡改 |

### 5.2 JWT认证

```bash
# 请求
POST /v1/auth/login
Body: {"username": "user", "password": "pass"}

# 响应
Response: {
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2..."
}

# 后续请求
GET /v1/orders
Header: Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

### 5.3 权限控制

```bash
# 基于角色的权限
GET /v1/admin/users  # 需要 ADMIN 角色

# 资源级权限
GET /v1/users/123/orders  # 只能查看自己的订单
# 服务端需验证: current_user_id == 123
```

---

## 6. 缓存设计

### 6.1 HTTP缓存头

| 头部 | 作用 | 示例 |
|------|------|------|
| Cache-Control | 缓存指令 | `max-age=3600, private` |
| ETag | 资源版本标识 | `"v1.0.0"` |
| Last-Modified | 最后修改时间 | `Wed, 09 Apr 2026 10:00:00 GMT` |
| Expires | 过期时间 | `Thu, 09 Apr 2026 11:00:00 GMT` |
| Vary | 缓存区分键 | `Vary: Accept-Encoding` |

### 6.2 缓存策略示例

```bash
# 允许CDN缓存1小时
GET /v1/products/1
Cache-Control: public, max-age=3600
ETag: "v2.1.0"

# 验证缓存
GET /v1/products/1
If-None-Match: "v2.1.0"
# 如果ETag匹配，返回304 Not Modified
```

### 6.3 REST缓存设计模式

```
# 先更新数据库，再让缓存失效
PUT /v1/users/1
Body: {"name": "新名字"}

# 策略1：更新时删除缓存
DELETE /v1/users/1/cache  # 让下游缓存失效

# 策略2：异步消息通知
# 发布 cache.invalidate 事件
# 订阅服务删除对应缓存
```

---

## 7. 限流设计

### 7.1 限流策略

| 策略 | 说明 | 实现 |
|------|------|------|
| 固定窗口 | 单位时间内限制请求数 | Redis INCR + TTL |
| 滑动窗口 | 连续时间窗口 | ZADD + 删除旧记录 |
| 令牌桶 | 突发流量控制 | 令牌补充速率 |
| 漏桶 | 恒定速率处理 | 队列 + 固定速率消费 |

### 7.2 限流响应头

```bash
# 限流阈值说明
X-RateLimit-Limit: 1000        # 请求总数限制
X-RateLimit-Remaining: 999    # 剩余请求数
X-RateLimit-Reset: 1712656800  # 限流重置时间（Unix时间戳）

# 触发限流
HTTP/1.1 429 Too Many Requests
Retry-After: 3600              # 多少秒后重试
```

### 7.3 限流实现示例

```python
# 令牌桶限流
import time
import redis

r = redis.Redis()

def check_rate_limit(user_id, limit=100, window=60):
    key = f"rate_limit:{user_id}"
    current = r.get(key)
    
    if current is None:
        r.setex(key, window, 1)
        return True
    
    if int(current) >= limit:
        return False
    
    r.incr(key)
    return True

def rate_limit_middleware(request):
    user_id = get_user_id(request)
    if not check_rate_limit(user_id):
        return 429, {"error": "Rate limit exceeded"}
    return 200, {"data": process(request)}
```

---

## 8. API文档与规范

### 8.1 OpenAPI规范

```yaml
openapi: 3.0.0
info:
  title: 用户服务 API
  version: 1.0.0
paths:
  /users/{id}:
    get:
      summary: 获取用户信息
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          description: 用户不存在
components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
        name:
          type: string
        email:
          type: string
```

### 8.2 常用文档工具

| 工具 | 特点 |
|------|------|
| Swagger/OpenAPI | 事实标准 |
| Postman | 客户端+文档 |
| Redoc | 漂亮的文档渲染 |
| Slate | Markdown风格文档 |

---

## 9. 最佳实践清单

### 9.1 设计检查

- [ ] 使用复数名词作为资源名（`/users`而非`/user`）
- [ ] 使用HTTP方法表达操作语义
- [ ] 返回合适的状态码
- [ ] 统一的错误响应格式
- [ ] 支持分页和过滤
- [ ] API版本管理
- [ ] 认证与授权
- [ ] 请求校验与验证
- [ ] 限流保护
- [ ] 缓存Header

### 9.2 安全检查

- [ ] 使用HTTPS
- [ ] 输入验证与SQL注入防护
- [ ] 限流防止DDoS
- [ ] 敏感数据脱敏
- [ ] 不在URL中传递敏感信息
- [ ] CORS配置
- [ ] API Key不提交到Git

---

## 10. 参考资料

[^1]: [REST API Design Best Practices](https://restfulapi.net/)
[^2]: [HTTP Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
[^3]: [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
[^4]: [RFC 6749 - OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749)

---

## 相关主题

- [[../web/http|HTTP协议详解]]
- [[../security/cryptography|密码学基础]]
- [[../distributed-systems/message-queue|消息队列系统]]
