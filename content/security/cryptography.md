---
title: 信息安全基础
date: 2026-04-07
description: 涵盖密码学基础、Web安全、安全实践等信息安全核心概念的入门指南
tags:
  - security
  - cryptography
draft: false
permalink:
---

## 密码学基础

### 对称加密 (Symmetric Encryption)

**原理**：加密和解密使用相同的密钥[^1]

**常见算法**：
- DES (Data Encryption Standard) - 56位密钥，已不安全
- AES (Advanced Encryption Standard) - 128/192/256位密钥，目前主流
- RC4、ChaCha20 - 流密码

**特点**：
- 优点：速度快，适合大量数据
- 缺点：密钥分发困难

```cpp
#include <bits/stdc++.h>
using namespace std;

// AES加密示例（简化版）
void aes_example() {
    // AES-256 密钥长度32字节
    unsigned char key[32] = {0};
    unsigned char plaintext[16] = {0};
    unsigned char ciphertext[16] = {0};
    
    // 实际应用中应使用专业的加密库如OpenSSL
    // 此处仅为概念演示
}
```

### 非对称加密 (Asymmetric Encryption)

**原理**：加密用公钥，解密用私钥[^1]

**常见算法**：
- RSA - 基于大数分解难题
- ECC (Elliptic Curve Cryptography) - 椭圆曲线密码体制
- Diffie-Hellman - 密钥交换

**特点**：
- 优点：密钥分发方便，可用于数字签名
- 缺点：速度慢，不适合大量数据

```cpp
#include <bits/stdc++.h>
using namespace std;

// RSA加密示例（简化版）
void rsa_example() {
    // 实际应用中应使用专业的加密库如OpenSSL
    // 此处仅为概念演示
    
    // p和q应为大素数
    long long p = 61, q = 53;
    long long n = p * q;  // 3233
    long long phi = (p - 1) * (q - 1);  // 3120
    
    // 公钥e通常取65537
    long long e = 17;
    // 私钥d满足 e*d ≡ 1 (mod phi)
    long long d = 2753;
}
```

### 混合加密

实际应用中结合两者[^1]：
1. 用非对称加密传递对称密钥
2. 用对称加密加密实际数据

例如 HTTPS 的加密机制。

## 哈希函数 (Hash Function)

**特性**：
1. 确定性：相同输入产生相同输出
2. 单向性：由输出无法推出输入
3. 抗碰撞性：难以找到两个不同输入产生相同输出

**常见算法**：
- MD5 - 128位，已被破解
- SHA-1 - 160位，已有碰撞
- SHA-256/SHA-3 - 目前安全

**应用**：
- 密码存储（加盐哈希）
- 消息完整性校验
- 区块链

```cpp
#include <bits/stdc++.h>
#include <openssl/sha.h>
using namespace std;

// SHA-256哈希示例
string sha256_example(const string& input) {
    unsigned char hash[SHA256_DIGEST_LENGTH];
    SHA256((unsigned char*)input.c_str(), input.length(), hash);
    
    string output;
    for(int i = 0; i < SHA256_DIGEST_LENGTH; i++) {
        char buf[3];
        sprintf(buf, "%02x", hash[i]);
        output += buf;
    }
    return output;
}
```

## 数字签名 (Digital Signature)

**原理**：
1. 对消息计算哈希值
2. 用私钥加密哈希值
3. 接收方用公钥解密，对比哈希值

**作用**：认证、防篡改、防抵赖

**常见算法**：RSA签名、DSA、ECDSA

```cpp
#include <bits/stdc++.h>
using namespace std;

// 数字签名示例（简化版）
void signature_example() {
    // 1. 计算消息哈希
    string message = "Hello, Digital Signature!";
    unsigned char hash[32] = {0};  // SHA-256
    
    // 2. 用私钥加密哈希值（签名）
    // 3. 接收方用公钥解密，对比哈希值
    
    // 实际应用中应使用专业的加密库如OpenSSL
}
```

## 数字证书与PKI

**数字证书**：包含公钥和身份信息，由CA签发

**PKI (Public Key Infrastructure)**：
- CA (Certificate Authority) - 证书颁发机构
- RA (Registration Authority) - 注册机构
- 证书吊销列表 CRL

**证书格式**：X.509

## Web安全基础

### XSS (Cross-Site Scripting)

跨站脚本攻击，注入恶意JavaScript代码[^2]。

**防御**：
- 输入过滤
- 输出编码
- HttpOnly Cookie

### CSRF (Cross-Site Request Forgery)

跨站请求伪造，诱导用户发起恶意请求[^2]。

**防御**：
- CSRF Token
- 验证Referer
- SameSite Cookie

### SQL注入

通过输入注入SQL代码[^2]。

**防御**：
- 参数化查询
- 输入验证
- 最小权限原则

```cpp
#include <bits/stdc++.h>
using namespace std;

// SQL注入防御示例
void sql_injection_defense() {
    // 不安全的写法（易被SQL注入）
    // string query = "SELECT * FROM users WHERE name = '" + username + "'";
    
    // 安全的参数化查询
    // 使用预处理语句
}
```

### 中间人攻击 (MITM)

攻击者插入到通信双方之间[^2]。

**防御**：
- HTTPS
- 证书校验
- 密钥交换协议

## 安全实践

1. **密码安全**：
   - 使用强密码
   - 不同网站使用不同密码
   - 使用密码管理器
   - 启用双因素认证

2. **密钥管理**：
   - 绝不硬编码密钥
   - 使用环境变量或密钥管理服务
   - 定期轮换

3. **传输安全**：
   - 使用HTTPS
   - 证书校验
   - 禁用SSLv3及以下版本

## 相关主题

- [[../tools/git|Git]] - 版本控制中的安全实践
- [[../networks/networks-overview|计算机网络]] - 安全协议基础

## 参考资料

- 本页内容综合自多个安全教程[^1][^2]

[^1]: 信息安全基础 - 百度百科
[^2]: Web安全攻防之道 - 博客园
