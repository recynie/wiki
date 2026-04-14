---
title: TLS/SSL Deep Dive
date: 2026-04-13
description: TLS协议深度解析：握手、证书、密码套件与前向保密
tags:
  - tls
  - ssl
  - network
  - security
draft: true
permalink:
---

## 1. TLS 握手演进

### 1.1 TLS 1.2 握手详解

TLS 1.2 是 2008 年发布的协议，其握手过程采用 2-RTT（Two Round Trip Time）模式。[^1]

#### 完整的 TLS 1.2 握手流程（ECDHE 为例）

```
Client                                  Server
   |                                        |
   |-------- ClientHello ------------------>|
   |        (Client Random,                |
   |         Cipher Suites,                |
   |         Supported Curves,             |
   |         Extension: supported_groups)   |
   |                                        |
   |<------- ServerHello -------------------|
   |        (Server Random,                |
   |         Selected Cipher Suite,        |
   |         Selected Curve)               |
   |                                        |
   |<------- Certificate -------------------|
   |        (Server Certificate Chain)     |
   |                                        |
   |<------- ServerKeyExchange ------------|
   |        (ECDH params: p, a, b,         |
   |         generator point G,            |
   |         server public key Q,          |
   |         signature of handshake        |
   |         messages using long-term key) |
   |                                        |
   |<------- CertificateRequest ------------|
   |        (可选：请求客户端证书)           |
   |                                        |
   |<------- ServerHelloDone --------------|
   |                                        |
   |                                        |
   |-------- ClientKeyExchange ------------>|
   |        (ECDH client public key)       |
   |                                        |
   |-------- Certificate ------------------>|
   |        (客户端证书链，可选)              |
   |                                        |
   |-------- CertificateVerify ------------>|
   |        (用客户端私钥签名                |
   |         all previous handshake msgs)  |
   |                                        |
   |-------- ChangeCipherSpec ------------->|
   |-------- Finished -------------------->|
   |        (handshake messages hash,      |
   |         encrypted with session key)   |
   |                                        |
   |<-------- ChangeCipherSpec -------------|
   |<-------- Finished --------------------|
   |                                        |
   |======= 加密数据传输 ========|
```

#### TLS 1.2 支持的密钥交换方式

| 密钥交换方式 | 特点 | 安全性 |
|------------|------|--------|
| RSA | 客户端生成 pre-master secret，用服务器公钥加密传输 | ❌ 不支持前向保密（PFS） |
| DH（有限域） | 离散对数问题，参数较大 | ⚠️ 逐渐淘汰 |
| ECDH（椭圆曲线） | 比 DH 更短的密钥 | ⚠️ 不支持 PFS（静态ECDH） |
| DHE（有限域 ephemeral） | 支持 PFS，但计算量大 | ✅ 安全但慢 |
| ECDHE（椭圆曲线 ephemeral） | 支持 PFS，性能好 | ✅ 推荐 |

#### RSA 密钥交换的致命缺陷

TLS 1.2 中的 RSA 密钥交换存在严重安全隐患：

```
攻击场景：攻击者记录所有历史流量
                    ┌─────────────────────────────┐
                    │     如果服务器私钥泄露        │
                    └─────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  攻击者可以用私钥解密 pre-master secret                          │
│  → 推导出会话密钥 Session Key                                   │
│  → 解密所有历史加密流量                                         │
└─────────────────────────────────────────────────────────────────┘
```

**结论**：RSA 密钥交换 **不支持前向保密（Perfect Forward Secrecy）**，这也是 TLS 1.3 将其移除的重要原因。

### 1.2 TLS 1.3 握手详解

TLS 1.3 于 2018 年发布，进行了根本性的安全改革。[^2]

#### TLS 1.3 核心改进

| 特性 | TLS 1.2 | TLS 1.3 |
|------|---------|---------|
| RTT 数 | 2-RTT（或 1-RTT + 0-RTT） | **1-RTT**（或 0-RTT with PSK） |
| 支持的密钥交换 | RSA、DH、ECDH、ECDHE、DHE | **仅 ECDHE**（PFS 强制） |
| RSA 密钥交换 | ✅ 支持 | ❌ **移除** |
| SHA-1 签名 | ✅ 支持 | ❌ **移除** |
| 3DES、RC4 | ✅ 支持 | ❌ **移除** |
| 握手加密 | 完成后才加密 | **握手时即加密**（0-RTT 除外） |
| 密钥派生函数 | MD5/SHA-1/SHA-256/SHA-384 | **HKDF**（仅 SHA-256/SHA-384） |
| 向前兼容模式 | 复杂 | ✅ 简化 |

#### TLS 1.3 完整握手流程（1-RTT）

```
Client                                  Server
   |                                        |
   |-------- ClientHello ------------------>|
   |        (supported_versions=TLS 1.3,   |
   |         supported groups,             |
   |         key_share* (client DH),       |
   |         supported signature algs,     |
   |         server_name,                  |
   |         pre_shared-key*,              |
   |         early_data* (0-RTT))          |
   |                                        |
   |                                        |
   |        *** 关键：密钥已可计算 ***      |
   |        (通过 (EC)DHE shared secret    |
   |         + HKDF 派生早期密钥)          |
   |                                        |
   |<------- ServerHello ------------------|
   |        (selected group,               |
   |         key_share (server DH),        |
   |         selected cipher suite,        |
   |         signature algorithms,         |
   |         pre_shared-key*)              |
   |                                        |
   |<------- {EncryptedExtensions} --------|
   |        (扩展加密，只含服务器配置)       |
   |        (不包含 certificate_request)   |
   |                                        |
   |<------- {CertificateRequest}* ---------|
   |        (可选，如果需要客户端认证)       |
   |                                        |
   |<------- {Certificate}* ---------------|
   |        (加密的服务器证书链)           |
   |                                        |
   |<------- {CertificateVerify} ----------|
   |        (用服务器私钥签名              |
   |         Transcript-Hash)              |
   |                                        |
   |<------- {Finished} -------------------|
   |        (握手消息Transcript的MAC)      |
   |                                        |
   |                                        |
   |        *** 双向认证完成 ***           |
   |        *** 所有握手消息已加密 ***     |
   |                                        |
   |-------- {Certificate}* -------------->|
   |-------- {CertificateVerify} -------->|
   |-------- {Finished} ------------------>|
   |                                        |
   |======= 加密数据传输 ========|
```

#### 0-RTT 握手（Session Resumption with PSK）

TLS 1.3 支持 0-RTT 恢复，通过预共享密钥（PSK）实现：

```
Client                                  Server
   |                                        |
   |-------- ClientHello ------------------>|
   |        (pre_shared_key=PSK,           |
   |         early_data=1,                 |
   |         key_share=(client DH))         |
   |                                        |
   |-------- Early Data ------------------->|
   |        (应用数据，用 early_secret     |
   |         + HKDF 派生的密钥加密)         |
   |                                        |
   |<------- ServerHello ------------------|
   |        (pre_shared_key selection)     |
   |                                        |
   |<------- {Finished} -------------------|
   |                                        |
   |======== 应用数据（1-RTT）========|
```

⚠️ **0-RTT 安全风险**：Early Data 可能被重放攻击（Replay Attack），因为它不依赖服务器的确认。

### 1.3 TLS 1.2 vs TLS 1.3 对比

| 维度 | TLS 1.2 | TLS 1.3 |
|------|---------|---------|
| **标准化年份** | 2008 | 2018 |
| **握手 RTT** | 2-RTT（完整握手的 RSA/ECDHE） | 1-RTT（ECDHE） |
| **0-RTT** | 1-RTT + 0-RTT（仅限 session resumption） | ✅ 原生支持 0-RTT |
| **密钥交换算法** | RSA、DH、ECDH、ECDHE、DHE | **仅 ECDHE** |
| **对称加密算法** | 3DES、RC4、AES-CBC、AES-GCM | **仅 AES-GCM、ChaCha20-Poly1305** |
| **MAC 算法** | MD5、SHA-1、SHA-256、SHA-384 | **AEAD**（集成在 cipher 中） |
| **密钥派生** | 多种自定义 PRF | **HKDF**（标准化） |
| **证书类型** | RSA、DSS、ECDSA | **仅 RSA、ECDSA** |
| **前向保密** | 可选（需用 DHE/ECDHE） | **强制** |
| **握手加密** | ServerHello 之后才加密 | **从 EncryptedExtensions 开始** |
| **压缩** | ✅ 支持 | ❌ **移除** |
| **自定义 DH 组** | ✅ 支持 | ❌ 移除，仅限标准化曲线 |
| **时间戳在 ServerHello** | ✅ 有 | ❌ 无（改用 key_share） |

## 2. 证书验证

### 2.1 证书链结构

数字证书形成一条信任链，从根证书到终端实体证书：[^3]

```
┌─────────────────────────────────────────────────────────────┐
│                        信任锚点                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Root CA（根证书颁发机构）                            │    │
│  │  - 自签名（Issuer = Subject）                        │    │
│  │  - 存储在操作系统/浏览器的根证书库                    │    │
│  │  - 有效期通常 15-20 年                               │    │
│  │  - 离线存储，高安全保护                               │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                 │
│                           │ 签发                            │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Intermediate CA（中间证书颁发机构）                   │    │
│  │  - 由 Root CA 签发                                   │    │
│  │  - 可有多级 Intermediate（中间证书链）                │    │
│  │  - 用于日常签发终端实体证书                            │    │
│  │  - 有效期通常 5-10 年                                 │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                 │
│                           │ 签发                            │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  End-entity Certificate（终端实体证书）               │    │
│  │  - 由 Intermediate CA 签发                          │    │
│  │  - Subject = 域名（如 *.example.com）                │    │
│  │  - 包含公钥、域名信息、SAN 等                         │    │
│  │  - 有效期通常 1 年（Let's Encrypt）或 2-3 年          │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 证书验证步骤

完整的证书验证流程如下：

#### 步骤 1：构建证书链

```
验证时，服务器会提供完整证书链：
[End-entity Cert] → [Intermediate CA 1] → [Intermediate CA 2] → ... → [Root CA]

浏览器/客户端需要：
1. 下载所有中间证书（如果服务器未提供）
2. 从叶子证书（叶子到根）构建链
3. 每个证书验证其签发者
```

#### 步骤 2：签名验证

```
对证书链中的每个证书进行签名验证：

┌─────────────────────────────────────────────────────────────┐
│  验证 Certificate[i] 的签名                                  │
│                                                             │
│  1. 获取 Certificate[i] 的签发者 Issuer                     │
│  2. 在信任库中找到对应的 Issuer 证书                         │
│  3. 使用 Issuer 公钥验证 Certificate[i] 的签名               │
│  4. 递归验证直到根证书（根证书自签名验证）                    │
└─────────────────────────────────────────────────────────────┘
```

#### 步骤 3：有效期检查

```
每个证书都有有效期的起止日期：

  Not Before                      Not After
      │                              │
      ▼                              ▼
─────┼────────────────────────────────┼─────▶ 时间
     │                              │
   证书生效                         证书过期

验证规则：
- 当前时间必须在 Not Before 和 Not After 之间
- 浏览器会警告过期证书
```

#### 步骤 4：吊销检查（Revocation Check）

当证书需要被作废时（如私钥泄露），通过以下方式检查：

| 机制 | 全称 | 工作原理 | 优点 | 缺点 |
|------|------|---------|------|------|
| **CRL** | Certificate Revocation List | 发布一个已吊销证书序列号列表 | 简单 | 列表可能很大，实时性差 |
| **OCSP** | Online Certificate Status Protocol | 实时查询单个证书状态 | 实时、带宽小 | 隐私问题（向第三方暴露访问的网站） |
| **OCSP Stapling** | - | 服务器自己获取 OCSP 响应并随证书一起提供 | 实时 + 隐私 | 需要服务器端配置 |
| **CRLSets** | - | Chrome 内置的已知的吊销列表 | 快速 | 可能不完整 |

#### 步骤 5：域名验证（SAN 检查）

```
证书的 Subject Alternative Name (SAN) 扩展用于指定证书有效的域名：

Certificate Subject: C=CN, ST=Beijing, L=Beijing, O=Example Inc, CN=*.example.com

Subject Alternative Names:
  - DNS:*.example.com      （通配符域名）
  - DNS:example.com        （精确域名）
  - DNS:www.example.com    （具体子域名）
  - IP:192.168.1.1         （IP 地址，可选）

验证逻辑：
1. 客户端获取连接目标的域名（如 www.example.com）
2. 检查目标域名是否在 SAN 列表中
3. 支持通配符匹配（*.example.com 匹配 www.example.com）
```

### 2.3 Subject Alternative Name（SAN）

SAN 是 X.509 证书中最重要的扩展之一，用于指定证书有效的所有标识符：

```bash
# 查看证书 SAN 的 OpenSSL 命令
openssl x509 -in certificate.pem -text -noout | grep -A 1 "Subject Alternative Name"

# 输出示例：
# Subject Alternative Name:
#     DNS:*.example.com, DNS:example.com, DNS:api.example.com
```

#### SAN vs Common Name（CN）

| 特性 | Common Name (CN) | Subject Alternative Name (SAN) |
|------|------------------|-------------------------------|
| 位置 | Subject 基本信息 | 扩展字段（extension） |
| 多值支持 | ❌ 仅一个值 | ✅ 多个 DNS、IP、email |
| 浏览器兼容性 | ⚠️ 逐渐废弃 | ✅ 现代浏览器要求 |
| 通配符 | ✅ 支持 | ✅ 支持 |

⚠️ **重要**：自 2019 年起，所有公开信任的 CA 签发的证书必须包含 SAN，CN 字段仅用于显示目的。

### 2.4 自签名证书

自签名证书是由自签名 CA 签发的证书，不依赖公共信任链：

```
┌─────────────────────────────────────────────────────────────┐
│                    自签名证书结构                            │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  自签名 CA 证书                                       │  │
│   │  - Issuer = Subject（自签名）                        │  │
│   │  - 签名用自身私钥                                     │  │
│   │  - 需要手动导入到信任库                               │  │
│   └─────────────────────────────────────────────────────┘  │
│                           │                                 │
│                           │ 签发                            │
│                           ▼                                 │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  自签名终端实体证书                                   │  │
│   │  - 由自签名 CA 签发                                  │  │
│   │  - 用于内网、开发环境                                 │  │
│   └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

#### 自签名证书的使用场景

| 场景 | 是否信任 | 典型用途 |
|------|---------|---------|
| 内网 HTTPS | ✅ 手动导入 CA | 企业内部系统 |
| 开发/测试环境 | ✅ 手动信任或忽略 | localhost、dev 环境 |
| 公共服务 | ❌ 浏览器阻止 | 不应使用 |

```bash
# 生成自签名证书的示例命令
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem \
    -days 365 -nodes \
    -subj "/CN=localhost" \
    -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"
```

## 3. 密码套件

### 3.1 密码套件格式

TLS 密码套件遵循统一命名规范：

```
TLS_<版本>_<密钥交换>_<加密算法>_<MAC算法>_<PRF或HKDF>
```

#### TLS 1.2 密码套件格式

```
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
   │      │    │    │      │      │
   │      │    │    │      │      └─ PRF 算法
   │      │    │    │      └─ MAC 算法（用于 TLS 1.2 握手）
   │      │    │    └─ 加密算法 + 模式
   │      │    └─ 终端实体证书类型
   │      └─ 密钥交换算法
   └─ TLS 协议版本
```

#### TLS 1.3 密码套件格式

TLS 1.3 简化了密码套件格式，因为密钥交换固定为 ECDHE：

```
TLS_AES_128_GCM_SHA256
   │    │   │      │
   │    │   │      └─ HKDF 算法
   │    │   └─ AEAD 认证加密算法
   │    └─ 密钥长度
   └─ TLS 1.3（无版本后缀）
```

### 3.2 对称加密算法对比

TLS 1.3 只保留了 AEAD（Authenticated Encryption with Associated Data）算法：

#### AES-GCM

| 特性 | 说明 |
|------|------|
| 类型 | AEAD（认证加密） |
| 密钥长度 | 128、256 位 |
| 模式 | Galois/Counter Mode |
| 特点 | 高性能，硬件加速广泛 |
| 安全性 | ✅ 安全（已被证明） |
| nonce | 每次加密必须不同（通常用序列号） |

#### ChaCha20-Poly1305

| 特性 | 说明 |
|------|------|
| 类型 | AEAD（认证加密） |
| 密钥长度 | 256 位 |
| 祖源 | Daniel J. Bernstein 设计 |
| 特点 | 软件实现高效，无硬件加速时性能好 |
| 安全性 | ✅ 安全（已被证明） |
| nonce | 96 位（重用会灾难性失败） |

#### 对比场景

| 场景 | 推荐算法 | 原因 |
|------|---------|------|
| 有 AES-NI 硬件加速 | AES-GCM | 硬件加速，速度更快 |
| 移动设备、低功耗 | ChaCha20-Poly1305 | 软件实现高效，省电 |
| TLS 1.3 | 二选一或两者皆可 | TLS 1.3 强制 AEAD |

### 3.3 ECDHE 密钥交换详解

ECDHE（Elliptic Curve Diffie-Hellman Ephemeral）是 TLS 1.3 唯一支持的密钥交换方式：

#### ECDHE 工作原理

```
┌─────────────────────────────────────────────────────────────┐
│                   ECDHE 密钥交换流程                          │
│                                                             │
│  Client                                       Server         │
│    │                                             │           │
│    │  选择椭圆曲线（secp256r1/secp384r1 等）        │           │
│    │                                             │           │
│    │  生成私钥 d_C（随机数）                        │           │
│    │  计算公钥 Q_C = d_C × G                       │           │
│    │                                             │           │
│    │  ──────── ClientKeyExchange ──────────>     │           │
│    │  发送：Q_C（客户端椭圆曲线公钥）               │           │
│    │                                             │           │
│    │                                             │  生成私钥 d_S（随机数）│
│    │                                             │  计算公钥 Q_S = d_S × G│
│    │                                             │           │
│    │  <────── ServerKeyExchange ──────────       │           │
│    │  发送：Q_S（服务器椭圆曲线公钥）               │           │
│    │                                             │           │
│    │  计算共享密钥：                               │           │
│    │  P = d_C × Q_S                               │           │
│    │    = d_C × (d_S × G)                         │           │
│    │    = (d_C × d_S) × G                         │           │
│    │                                             │           │
│    │                                             │  计算共享密钥：│
│    │                                             │  P = d_S × Q_C│
│    │                                             │    = d_S × (d_C × G)│
│    │                                             │    = (d_S × d_C) × G│
│    │                                             │           │
│    │  === 共享密钥 P 计算完成（双方相同） ===        │           │
│    │                                             │           │
└─────────────────────────────────────────────────────────────┘
```

#### 为什么 ECDHE 提供前向保密？

```
关键：每次会话使用不同的临时密钥对

┌─────────────────────────────────────────────────────────────┐
│  ECDHE 的前向保密机制                                         │
│                                                             │
│  攻击场景：长期私钥泄露                                       │
│                                                             │
│  假设攻击者获取了服务器的 RSA/EC private key：                │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  对于 RSA 密钥交换（TLS 1.2）：                       │   │
│  │  - 攻击者可直接解密 pre-master secret                 │   │
│  │  - 解密所有历史会话                                   │   │
│  │  ❌ 无前向保密                                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  对于 ECDHE 密钥交换：                                │   │
│  │  - 长期私钥只用于签名                                 │   │
│  │  - 每次会话的 session ephemeral key 已删除            │   │
│  │  - 即使长期私钥泄露，攻击者没有 ephemeral keys        │   │
│  │  ✅ 有前向保密                                        │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 3.4 TLS 1.3 密码套件优先级

现代 TLS 库按优先级排序密码套件：

```bash
# Nginx 推荐密码套件配置（TLS 1.3）
ssl_protocols TLSv1.3;
ssl_ciphers 'TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256';
ssl_prefer_server_ciphers off;  # TLS 1.3 要求关闭

# OpenSSL 密码套件列表
TLS_AES_256_GCM_SHA384         # AES-256-GCM + SHA-384
TLS_CHACHA20_POLY1305_SHA256   # ChaCha20-Poly1305 + SHA-256
TLS_AES_128_GCM_SHA256         # AES-128-GCM + SHA-256
```

#### 密码套件选择原则

| 原则 | 说明 |
|------|------|
| **AEAD 优先** | 选择 AEAD 算法（AES-GCM、ChaCha20-Poly1305） |
| **PFS 强制** | 必须支持 ECDHE（前向保密） |
| **密钥长度** | 对称加密 ≥ 128 位，ECDH ≥ 256 位 |
| **SHA 族** | TLS 1.3 仅用 SHA-256/SHA-384 |
| **禁用** | RC4、3DES、MD5、SHA-1、EXPORT 套件 |

## 4. 前向保密（Perfect Forward Secrecy）

### 4.1 概念定义

**前向保密（Perfect Forward Secrecy，PFS）** 是一种密码学属性，确保即使长期密钥（如服务器私钥）泄露，历史会话的通信内容仍然保密。[^4]

```
┌─────────────────────────────────────────────────────────────┐
│                    PFS vs 非 PFS 对比                        │
│                                                             │
│  非 PFS（RSA 密钥交换）：                                     │
│  ┌──────────┐         ┌──────────┐         ┌──────────┐     │
│  │  长期私钥 │ ──────▶ │ 会话密钥 │ ──────▶ │ 历史流量 │     │
│  │  泄露    │         │  可推导  │         │  可解密  │     │
│  └──────────┘         └──────────┘         └──────────┘     │
│       │                                               │     │
│       └───────────────────────────────────────────────┘     │
│                     所有历史流量沦陷                          │
│                                                             │
│  PFS（ECDHE 密钥交换）：                                      │
│  ┌──────────┐         ┌──────────┐         ┌──────────┐     │
│  │  长期私钥 │ ──────▶ │ 签名验证 │         │ 历史流量 │     │
│  │  泄露    │         │ (仅此用) │         │  已加密  │     │
│  └──────────┘         └──────────┘         └──────────┘     │
│                           │                                  │
│                           ▼                                  │
│                      ┌──────────┐                            │
│                      │ Ephemeral │                           │
│                      │ Key 已销毁 │                           │
│                      └──────────┘                            │
│                                                             │
│       攻击者即使拿到长期私钥，也无法解密历史会话              │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 PFS 的实现方式

TLS 中实现 PFS 有两种主要方式：

#### ECDHE（前向保密推荐实现）

```
ECDHE = Ephemeral ECDH

特性：
- 每次会话生成新的临时 ECDH 密钥对
- 私钥在会话结束后立即销毁
- 服务器用长期私钥对 ECDHE 参数签名
- 签名仅用于认证，不用于加密会话密钥
```

#### DHE（前向保密但已淘汰）

```
DHE = Diffie-Hellman Ephemeral

特性：
- 使用有限域离散对数问题
- 密钥长度需 >= 2048 位（推荐 4096 位）才能保证安全
- 计算量比 ECDHE 大 3-6 倍
- TLS 1.3 已移除 DHE 支持
```

#### ECDHE vs DHE

| 特性 | ECDHE | DHE |
|------|-------|-----|
| 数学基础 | 椭圆曲线离散对数 | 有限域离散对数 |
| 密钥长度（同等安全） | 256 位（secp256r1） | 2048 位 |
| 计算性能 | 快（~3-6x DHE） | 慢 |
| TLS 1.3 支持 | ✅ 是 | ❌ 否 |
| 标准化曲线 | ✅（P-256/P-384/X25519） | ⚠️ 自定义参数风险 |

### 4.3 TLS 1.3 为何强制 PFS

TLS 1.3 强制前向保密是出于深刻的安全考量：

#### TLS 1.3 的设计决策

```
TLS 1.3 移除的密码套件：
┌─────────────────────────────────────────────────────────────┐
│  ❌ TLS_RSA_*              （RSA 密钥交换，无 PFS）          │
│  ❌ TLS_DH_*               （静态 DH，无 PFS）               │
│  ❌ TLS_ECDH_*             （静态 ECDH，无 PFS）             │
│  ❌ TLS_ECDHE_*_SHA1       （SHA-1 不安全）                  │
│  ❌ TLS_DHE_*_SHA          （DHE 性能问题）                  │
│  ❌ TLS_RSA_WITH_*_CBC_*   （CBC 模式有攻击面）             │
│  ❌ TLS_3DES_*             （64 位块大小，容易碰撞）          │
│  ❌ TLS_RC4_*              （RC4 有已知偏差攻击）             │
└─────────────────────────────────────────────────────────────┘

TLS 1.3 保留的密码套件：
┌─────────────────────────────────────────────────────────────┐
│  ✅ TLS_AES_128_GCM_SHA256                                  │
│  ✅ TLS_AES_256_GCM_SHA384                                  │
│  ✅ TLS_CHACHA20_POLY1305_SHA256                            │
└─────────────────────────────────────────────────────────────┘
```

#### 前向保密的实际意义

| 威胁场景 | 无 PFS | 有 PFS |
|---------|--------|--------|
| 服务器私钥泄露 | 所有历史流量可解密 | ✅ 历史流量安全 |
| 被动收集流量后私钥泄露 | 历史流量可解密 | ✅ 历史流量安全 |
| 量子计算威胁（未来） | ❌ 灾难性 | ⚠️ 需后量子密码学 |
| 证书误签发 | 攻击者可用证书解密 | ✅ 历史流量安全 |

#### 部署建议

```nginx
# Nginx TLS 配置（强制 PFS + TLS 1.3）
server {
    listen 443 ssl;
    http2 on;

    # TLS 版本
    ssl_protocols TLSv1.3 TLSv1.2;

    # 密码套件（按优先级排序）
    ssl_ciphers 'TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256';
    ssl_prefer_server_ciphers off;

    # 椭圆曲线（TLS 1.3 强制 ECDHE）
    ssl_ecdh_curve X25519:secp384r1:secp256r1;

    # Session tickets（禁用或配置正确）
    ssl_session_tickets off;  # 或使用 ticket key rotation
}
```

## 附录：常见 TLS 配置问题排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| `ssl_error_rx_record_too_long` | 协议误用（如 HTTP 流量到 HTTPS 端口） | 检查端口配置是否正确 |
| `ERR_CERT_COMMON_NAME_INVALID` | CN 已废弃，域名不在 SAN 中 | 使用 SAN 而非 CN |
| `ERR_CERT_REVOKED` | 证书被吊销 | 检查证书状态，更新证书 |
| `ERR_SSL_VERSION_OR_CIPHER_MISMATCH` | 密码套件不匹配 | 启用现代密码套件 |
| 握手超时 | 网络问题或防火墙 | 检查 443 端口可达性 |
| 证书链不完整 | 中间证书缺失 | 配置完整的证书链 |

## 参考资料

[^1]: RFC 5246 - The Transport Layer Security (TLS) Protocol Version 1.2. IETF, 2008.

[^2]: RFC 8446 - The Transport Layer Security (TLS) Protocol Version 1.3. IETF, 2018.

[^3]: CA/Browser Forum Baseline Requirements for the Issuance and Management of Publicly-Trusted Certificates.

[^4]: What is Perfect Forward Secrecy? - Cloudflare Learning Center.
