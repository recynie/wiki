---
title: API安全
date: 2026-04-11
description: API安全最佳实践，涵盖JWT认证、OAuth 2.0、API密钥管理、限流策略等核心主题
tags:
  - api-security
  - jwt
  - oauth
  - authentication
  - authorization
draft: false
permalink:
---

## 概述

API安全是现代分布式系统的基础。本指南聚焦JWT认证、OAuth 2.0 Flows、API密钥管理、限流策略等实战主题。[^1]

## JWT（JSON Web Token）

### 结构解析

```
xxxxx.yyyyy.zzzzz
│     │     │
│     │     └── Signature（签名）
│     └── Payload（声明）
└── Header（头部）
```

### Header

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "key-123"  // 密钥ID，用于密钥轮换
}
```

### Payload（声明）

```json
{
  "iss": "https://auth.example.com",    // 发行者
  "sub": "user-12345",                   // 主题（用户ID）
  "aud": "https://api.example.com",     // 受众
  "exp": 1735689600,                     // 过期时间
  "iat": 1735686000,                     // 签发时间
  "jti": "unique-token-id",              // JWT ID（用于撤销）
  "roles": ["admin", "user"]
}
```

### 签名算法选择

| 算法 | 类型 | 适用场景 | 安全性 |
|------|------|---------|--------|
| HS256 | HMAC | 单服务、共享密钥 | 中 |
| RS256 | RSA | 分布式系统 | 高 |
| ES256 | ECDSA | 移动端、高性能 | 高 |
| PS256 | RSA-PSS | 金融场景 | 最高 |
| none | 无 | ❌ **禁止使用** | 不安全 |

**分布式系统推荐RS256/ES256**：

```
服务器A（发行JWT）
    │
    │ 私钥签名
    ▼
JWT Token
    │
    │ 公钥验证（各服务器持有）
    ▼
服务器B、C、D（验证JWT）
```

## JWT安全最佳实践

### 1. 完整验证清单

```typescript
// 必须验证的所有声明
interface JWTValidation {
  // 1. 签名验证（防止篡改）
  signature: verify(token, publicKey),
  
  // 2. 算法验证（防止alg:none攻击）
  algorithm: verifyAlgorithm(token, ['RS256', 'ES256']),
  
  // 3. 发行者验证
  issuer: token.iss === expectedIssuer,
  
  // 4. 受众验证
  audience: token.aud.includes(expectedAudience),
  
  // 5. 过期时间验证
  expiration: token.exp > currentTime,
  
  // 6. 生效时间验证（可选，防重放）
  notBefore: token.nbf <= currentTime,
}
```

### 2. 常见漏洞与防御

#### 敏感数据泄露

```json
// ❌ 错误：JWT payload是Base64编码，非加密
{
  "user_id": 123,
  "password": "super_secret",  // 永远不要放！
  "ssn": "123-45-6789"         // 永远不要放！
}
```

```json
// ✅ 正确：只放必要声明
{
  "sub": "user-123",
  "iat": 1735686000,
  "exp": 1735689600,
  "roles": ["user"]
}
```

#### algorithm:none攻击

```typescript
// ❌ 易受攻击的实现
const decoded = jwt.verify(token, secret, {
  algorithms: ['HS256', 'RS256']  // 允许任意算法
});

// ✅ 安全实现
const decoded = jwt.verify(token, secret, {
  algorithms: ['RS256']  // 明确指定算法
});
```

### 3. 存储与传输

| 存储位置 | HttpOnly | Secure | SameSite | 安全性 |
|---------|----------|--------|----------|-------|
| HttpOnly Cookie | ✅ | ✅ | Lax | **推荐** |
| localStorage | ❌ | N/A | N/A | ⚠️ XSS风险 |
| sessionStorage | ❌ | N/A | N/A | ⚠️ XSS风险 |
| 内存 | ❌ | N/A | N/A | ✅ 需防XSS |

```typescript
// 安全cookie设置
res.cookie('access_token', token, {
  httpOnly: true,    // 防止XSS访问
  secure: true,      // 仅HTTPS传输
  sameSite: 'Lax',   // CSRF防护
  maxAge: 15 * 60 * 1000  // 15分钟
});
```

### 4. Token刷新策略

```typescript
// Refresh Token Rotation：每次刷新都颁发新的refresh token
async function refreshTokens(refreshToken: string) {
  // 1. 验证旧refresh token
  const oldPayload = verifyRefreshToken(refreshToken);
  
  // 2. 检查token是否在黑名单
  const isRevoked = await redis.sismember('revoked_tokens', refreshToken);
  if (isRevoked) {
    throw new Error('Token reuse detected');
  }
  
  // 3. 作废旧refresh token
  await redis.sadd('revoked_tokens', refreshToken);
  
  // 4. 颁发新token对
  const newAccessToken = signAccessToken(oldPayload.userId);
  const newRefreshToken = signRefreshToken(oldPayload.userId);
  
  return { newAccessToken, newRefreshToken };
}
```

### 5. Token撤销方案

```typescript
// 方案1：Redis黑名单
await redis.sadd(`revoked:${token.jti}`, '1');
await redis.expire(`revoked:${token.jti}`, token.exp - now);

// 方案2：Token版本号
// 用户表：token_version = 5
// JWT payload包含version字段
// 验证时对比版本号
```

## OAuth 2.0 Flows

### Flow对比

| Flow | 客户端类型 | 安全性 | 适用场景 |
|------|-----------|--------|---------|
| Authorization Code | Web服务器 | ⭐⭐⭐ | 后端服务 |
| Authorization Code + PKCE | SPA、移动App | ⭐⭐⭐⭐ | 前端应用 |
| Client Credentials | 服务间 | ⭐⭐⭐ | M2M通信 |
| Device Code | CLI工具 | ⭐⭐⭐ | 设备授权 |
| Implicit | - | ⭐⭐ | ❌ 已废弃 |

### Authorization Code Flow

```
┌──────────┐                    ┌──────────┐                    ┌──────────┐
│   User   │                    │   Auth   │                    │   API    │
│  Browser │                    │  Server  │                    │  Server  │
└────┬─────┘                    └────┬─────┘                    └────┬─────┘
     │                               │                               │
     │  1. 访问应用                   │                               │
     │───────────────────────────────>│                               │
     │                               │                               │
     │  2. 重定向到登录页             │                               │
     │<───────────────────────────────│                               │
     │                               │                               │
     │  3. 输入凭证                   │                               │
     │───────────────────────────────>│                               │
     │                               │                               │
     │  4. 授权确认页                 │                               │
     │<───────────────────────────────│                               │
     │                               │                               │
     │  5. 用户同意                    │                               │
     │───────────────────────────────>│                               │
     │                               │                               │
     │  6. 授权码(code)              │                               │
     │<───────────────────────────────│                               │
     │                               │                               │
     │  7. 用code换token（后端直连）  │                               │
     │───────────────────────────────────────────────────────────────>│
     │                               │                               │
     │  8. 返回access_token, id_token│<───────────────────────────────│
     │<───────────────────────────────────────────────────────────────│
```

### PKCE（Proof Key for Code Exchange）

防止授权码拦截攻击：

```typescript
// 1. 生成code_verifier和code_challenge
function generatePKCE() {
  const codeVerifier = crypto.randomBytes(32).toString('base64url');
  const codeChallenge = crypto
    .createHash('sha256')
    .update(codeVerifier)
    .digest('base64url');
  
  return { codeVerifier, codeChallenge };
}

// 2. 授权请求
const { codeVerifier, codeChallenge } = generatePKCE();
session.codeVerifier = codeVerifier;  // 存储用于后续验证

const authUrl = new URL('https://auth.example.com/authorize');
authUrl.searchParams.set('code_challenge', codeChallenge);
authUrl.searchParams.set('code_challenge_method', 'S256');

// 3. 验证code_verifier
async function exchangeCode(code: string, codeVerifier: string) {
  const response = await fetch(tokenEndpoint, {
    method: 'POST',
    body: new URLSearchParams({
      grant_type: 'authorization_code',
      code,
      redirect_uri,
      client_id,
      code_verifier  // 证明持有密钥
    })
  });
}
```

## API限流策略

### 算法对比

| 算法 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| 固定窗口 | 固定时间段内计数 | 简单 | 边界突变 |
| 滑动窗口 | 移动时间窗口 | 平滑 | 实现复杂 |
| 令牌桶 | 固定速率添加令牌 | 允许突发 | 需存储状态 |
| 漏桶 | 固定速率处理 | 平滑输出 | 突发受限 |

### 实现示例

```typescript
// 令牌桶算法
class TokenBucket {
  constructor(
    private capacity: number,
    private refillRate: number,  // 每秒补充令牌数
    private tokens: number,
    private lastRefill: number
  ) {}

  async consume(tokens: number): Promise<boolean> {
    this.refill();
    
    if (this.tokens >= tokens) {
      this.tokens -= tokens;
      return true;  // 允许
    }
    return false;  // 拒绝
  }

  private refill() {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    const newTokens = elapsed * this.refillRate;
    this.tokens = Math.min(this.capacity, this.tokens + newTokens);
    this.lastRefill = now;
  }
}

// 使用Redis分布式限流
async function rateLimit(
  userId: string,
  limit: number,
  windowSeconds: number
): Promise<{ allowed: boolean; remaining: number; reset: number }> {
  const key = `rate:${userId}`;
  const now = Date.now();
  const windowMs = windowSeconds * 1000;
  
  const multi = redis.multi();
  multi.zremrangebyscore(key, 0, now - windowMs);  // 清理过期
  multi.zadd(key, now, `${now}-${Math.random()}`);  // 添加请求
  multi.zcard(key);                                 // 计数
  multi.pexpire(key, windowMs);                     // 设置过期
  
  const results = await multi.exec();
  const count = results[2] as number;
  
  return {
    allowed: count <= limit,
    remaining: Math.max(0, limit - count),
    reset: now + windowMs
  };
}
```

### HTTP响应头

```typescript
// 标准限流响应头
res.set({
  'X-RateLimit-Limit': '100',           // 窗口内最大请求数
  'X-RateLimit-Remaining': '95',        // 剩余请求数
  'X-RateLimit-Reset': '1735689600',    // 窗口重置时间戳
  'Retry-After': '30'                    // 被限流后重试等待秒数
});
```

## API Key管理

### 用途分类

| 类型 | 生成方式 | 安全性 | 适用场景 |
|------|---------|--------|---------|
| 对称密钥 | 预共享 | 中 | 服务间通信 |
| 非对称密钥 | 公私钥对 | 高 | 跨平台认证 |
| HMAC密钥 | 签名密钥 | 高 | API请求签名 |

### 安全实践

```yaml
# ❌ 错误：API Key硬编码
Authorization: ApiKey sk_live_xxxxxxxxxxxxxxxxxxxx

# ✅ 正确：使用请求签名
Authorization: Signature keyId="key-123",algorithm="hmac-sha256",
               headers="(request-target) date digest",
               signature="base64_signature"

# 或使用Bearer Token
Authorization: Bearer eyJhbGc...
```

## 最佳实践清单

### 部署前检查

- [ ] 使用RS256/ES256签名算法
- [ ] access_token TTL ≤ 15分钟
- [ ] refresh_token开启rotation
- [ ] JWT payload不含敏感数据
- [ ] 启用token撤销机制（Redis黑名单）
- [ ] HttpOnly + Secure + SameSite Cookie
- [ ] 实现请求签名或mTLS
- [ ] 配置限流策略
- [ ] 记录审计日志

## 参考资料

[^1]: [JWT Best Current Practices - RFC 8725](https://rfc-editor.org/rfc/rfc8725.html)
[^2]: [OAuth 2.0 Security BCP - RFC 9700](https://www.ietf.org/rfc/rfc9700.html)
[^3]: [JWT Security Best Practices - Curity](https://curity.io/resources/learn/jwt-best-practices/)
