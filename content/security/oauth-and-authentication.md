---
title: OAuth 2.0与身份认证
date: 2026-04-13
description: OAuth 2.0授权流程、PKCE、OIDC协议与安全最佳实践
tags:
  - security
  - oauth
  - authentication
  - authorization
draft: false
permalink:
---

# OAuth 2.0与身份认证

OAuth 2.0是现代授权框架，让用户无需向第三方应用透露密码即可授权访问其数据。

## OAuth 2.0核心概念

### 角色

| 角色 | 说明 |
|------|------|
| **Resource Owner** | 资源所有者（用户） |
| **Client** | 请求访问的第三方应用 |
| **Authorization Server** | 授权服务器 |
| **Resource Server** | 托管受保护资源的服务器 |

### 授权许可类型

| 类型 | 适用场景 |
|------|----------|
| Authorization Code | 有后端服务器的Web应用 |
| PKCE | 移动应用、单页应用 |
| Client Credentials | 服务间通信 |
| Device Code | 设备限制输入场景 |
| Refresh Token | 访问令牌续期 |

## Authorization Code 授权流程

```
     ┌──────────┐                           ┌──────────┐
     │   User   │                           │    App   │
     └────┬─────┘                           └────┬─────┘
          │                                      │
          │  1. 点击"使用Google登录"               │
          │─────────────────────────────────────►│
          │                                      │
          │  2. 重定向到授权服务器                 │
          │◄─────────────────────────────────────│
          │                                      │
          │  3. 输入凭证并同意授权                │
          │─────────────────────────────────────►│
          │       (授权服务器)                   │
          │                                      │
          │  4. 重定向回App with code            │
          │◄─────────────────────────────────────│
          │                                      │
          │  5. 用code交换access_token           │
          │─────────────────────────────────────►│
          │       (授权服务器)                   │
          │                                      │
          │  6. 返回access_token                 │
          │◄─────────────────────────────────────│
          │                                      │
          │  7. 用token访问资源                  │
          │─────────────────────────────────────►│
          │       (Resource Server)             │
          │                                      │
```

### 授权请求

```http
GET /authorize?
    response_type=code&
    client_id=YOUR_CLIENT_ID&
    redirect_uri=https%3A%2F%2Fyour-app.com%2Fcallback&
    scope=openid%20profile%20email&
    state=random_state_string&
    code_challenge=S256_code_challenge&
    code_challenge_method=S256
Host: authorization-server.com
```

### 令牌交换

```python
import requests
import base64
import hashlib

def exchange_code_for_token(auth_code, code_verifier):
    """用授权码交换访问令牌"""
    
    client_id = "your-client-id"
    client_secret = "your-client-secret"  # 后端保密
    redirect_uri = "https://your-app.com/callback"
    
    # code_verifier是PKCE的一部分
    code_challenge = base64.urlsafe_b64encode(
        hashlib.sha256(code_verifier.encode()).digest()
    ).decode().rstrip('=')
    
    response = requests.post("https://auth-server.com/token", data={
        "grant_type": "authorization_code",
        "code": auth_code,
        "redirect_uri": redirect_uri,
        "client_id": client_id,
        "code_verifier": code_verifier,  # PKCE验证
    })
    
    return response.json()
```

## PKCE（Proof Key for Code Exchange）

PKCE防止授权码拦截攻击，移动端和SPA必须使用。

### 流程

```javascript
// 1. 生成code_verifier（随机字符串）
function generateCodeVerifier() {
    const array = new Uint8Array(32);
    crypto.getRandomValues(array);
    return base64urlEncode(array);
}

// 2. 生成code_challenge
async function generateCodeChallenge(verifier) {
    const encoder = new TextEncoder();
    const data = encoder.encode(verifier);
    const digest = await crypto.subtle.digest('SHA-256', data);
    return base64urlEncode(new Uint8Array(digest));
}

// 3. 登录流程
async function login() {
    const codeVerifier = generateCodeVerifier();
    const codeChallenge = await generateCodeChallenge(codeVerifier);
    
    // 保存到sessionStorage
    sessionStorage.setItem('codeVerifier', codeVerifier);
    
    // 构建授权URL
    const authUrl = new URL('https://auth-server.com/authorize');
    authUrl.searchParams.set('client_id', 'your-client-id');
    authUrl.searchParams.set('response_type', 'code');
    authUrl.searchParams.set('redirect_uri', 'https://your-app.com/callback');
    authUrl.searchParams.set('scope', 'openid profile');
    authUrl.searchParams.set('code_challenge', codeChallenge);
    authUrl.searchParams.set('code_challenge_method', 'S256');
    authUrl.searchParams.set('state', generateRandomState());
    
    window.location.href = authUrl.toString();
}

// 4. 回调处理
async function handleCallback() {
    const urlParams = new URLSearchParams(window.location.search);
    const code = urlParams.get('code');
    const state = urlParams.get('state');
    const codeVerifier = sessionStorage.getItem('codeVerifier');
    
    // 用code和code_verifier交换token
    const tokenResponse = await fetch('https://auth-server.com/token', {
        method: 'POST',
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
        body: new URLSearchParams({
            grant_type: 'authorization_code',
            code: code,
            redirect_uri: 'https://your-app.com/callback',
            client_id: 'your-client-id',
            code_verifier: codeVerifier
        })
    });
    
    const tokens = await tokenResponse.json();
    // tokens.access_token, tokens.id_token, tokens.refresh_token
}
```

## OIDC（OpenID Connect）

OIDC在OAuth 2.0基础上提供身份认证，返回ID Token。

### ID Token结构

ID Token是JWT格式：

```json
{
  "iss": "https://auth-server.com",
  "sub": "user123",           // 用户唯一标识
  "aud": "your-client-id",    // 接收方
  "exp": 1699999999,          // 过期时间
  "iat": 1699996399,          // 签发时间
  "nonce": "random-nonce",
  "name": "John Doe",
  "email": "john@example.com",
  "picture": "https://..."
}
```

### 验证ID Token

```python
import jwt
from jwt import PyJWKClient

def verify_id_token(id_token, client_id):
    # 获取公钥
    jwks_url = "https://auth-server.com/.well-known/jwks.json"
    jwks_client = PyJWKClient(jwks_url)
    signing_key = jwks_client.get_signing_key_from_jwt(id_token)
    
    # 验证并解码
    claims = jwt.decode(
        id_token,
        signing_key.key,
        algorithms=["RS256"],
        audience=client_id,
        options={"verify_exp": True}
    )
    
    return claims
```

## Client Credentials（客户端凭证）

适用于服务间通信，无需用户交互：

```python
def get_service_token():
    """获取服务到服务访问令牌"""
    
    response = requests.post("https://auth-server.com/token", data={
        "grant_type": "client_credentials",
        "client_id": "service-client-id",
        "client_secret": "service-client-secret",
        "scope": "api://backend-api/.default"
    })
    
    return response.json()["access_token"]

# 使用令牌调用API
def call_backend_api(token):
    headers = {"Authorization": f"Bearer {token}"}
    response = requests.get("https://api.backend.com/data", headers=headers)
    return response.json()
```

## Refresh Token刷新

```python
def refresh_access_token(refresh_token):
    """使用刷新令牌获取新的访问令牌"""
    
    response = requests.post("https://auth-server.com/token", data={
        "grant_type": "refresh_token",
        "refresh_token": refresh_token,
        "client_id": "your-client-id",
        # client_secret如果客户端类型需要
    })
    
    return response.json()
    # {
    #     "access_token": "new-access-token",
    #     "refresh_token": "new-refresh-token",  # 可能轮转
    #     "expires_in": 3600
    # }
```

## Token安全最佳实践

| 实践 | 说明 |
|------|------|
| HTTPS传输 | 所有Token传输必须加密 |
| 短期Access Token | Access Token有效期15分钟-1小时 |
| Refresh Token轮转 | 每次刷新生成新Refresh Token |
| Token存储 | Access Token存内存，Refresh Token存安全存储 |
| PKCE必用 | 移动端和SPA必须启用PKCE |
| 令牌轮换 | 检测到旧Token使用立即失效 |

## JWT安全

```python
import jwt
import datetime

# 签发Token
def create_tokens(user_id):
    now = datetime.datetime.utcnow()
    
    access_token = jwt.encode({
        "sub": user_id,
        "iat": now,
        "exp": now + datetime.timedelta(hours=1),
        "type": "access"
    }, SECRET_KEY, algorithm="HS256")
    
    refresh_token = jwt.encode({
        "sub": user_id,
        "iat": now,
        "exp": now + datetime.timedelta(days=7),
        "type": "refresh",
        "jti": generate_jti()  # 唯一标识，用于撤销
    }, SECRET_KEY, algorithm="HS256")
    
    return access_token, refresh_token

# 验证Token
def verify_token(token, expected_type="access"):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        
        if payload["type"] != expected_type:
            raise ValueError("Invalid token type")
        
        # 检查是否在撤销列表
        if is_token_revoked(payload["jti"]):
            raise ValueError("Token has been revoked")
        
        return payload
    except jwt.ExpiredSignatureError:
        raise ValueError("Token expired")
    except jwt.InvalidTokenError:
        raise ValueError("Invalid token")
```

## 参考

[^1]: RFC 6749 - OAuth 2.0 Authorization Framework
[^2]: RFC 7636 - PKCE
[^3]: OpenID Connect Core 1.0

