---
title: TCP/IP协议栈深入
date: 2026-04-08
description: TCP/IP协议栈深入理解：TCP三次握手四次挥手、滑动窗口、拥塞控制、DNS、CDN
tags:
  - tcp-ip
  - network
  - tcp
  - dns
  - cdn
draft: false
permalink:
---

## 概述

TCP/IP协议栈是互联网的基础协议族，涵盖了从物理链路到应用层的完整网络通信体系。[^1]

---

## TCP协议深入

### TCP头部结构

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Sequence Number                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   Acknowledgment Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Offset|  Resvd  | UAPRSF|               Window               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Checksum           |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options and Padding                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 三次握手与四次挥手

#### 三次握手（建立连接）

```
客户端                                服务器
  │                                    │
  │────────── SYN (seq=x) ────────────▶│  1. 客户端发送SYN
  │◀──────── SYN-ACK (seq=y, ack=x+1) ─│  2. 服务器发送SYN+ACK
  │────────── ACK (ack=y+1) ───────────▶│  3. 客户端发送ACK
  │                                    │
  │        TCP连接建立完成              │
```

**状态变化**：
- CLOSED → SYN_SENT → ESTABLISHED（客户端）
- LISTEN → SYN_RCVD → ESTABLISHED（服务器）

#### 四次挥手（断开连接）

```
客户端                                服务器
  │                                    │
  │────────── FIN (seq=u) ────────────▶│  1. 主动方发送FIN
  │◀───────── ACK (ack=u+1) ───────────│  2. 被动方发送ACK
  │                                    │  3. 被动方准备关闭
  │◀───────── FIN (seq=w) ────────────│  4. 被动方发送FIN
  │────────── ACK (ack=w+1) ───────────▶│  5. 主动方发送ACK
  │                                    │
  │    TIME_WAIT (2MSL后) → CLOSED     │
```

**为什么需要TIME_WAIT**：
1. 确保被动方收到最后的ACK
2. 等待所有迷途分组消失

### 滑动窗口机制

滑动窗口用于**流量控制**，避免发送方过快导致接收方缓存溢出。

```cpp
class SlidingWindow {
    int base;              // 最早未确认字节序号
    int next_seq;           // 下一个要发送的字节序号
    int window_size;        // 接收方通告的窗口大小
    
    vector<packet> buffer; // 已发送但未确认的包
    
    void send_data(int seq, const string& data) {
        packet p;
        p.seq = seq;
        p.data = data;
        p.transmit_time = now();
        buffer.push_back(p);
        next_seq += data.size();
        
        // 实际发送（受窗口大小限制）
        while (next_seq - base >= window_size) {
            // 窗口满，等待确认
            wait();
        }
    }
    
    void handle_ack(int ack_num) {
        if (ack_num > base) {
            // 删除已确认的包
            remove_acked_packets(ack_num);
            base = ack_num;
            // 可以发送更多数据
        }
    }
};
```

### 拥塞控制

拥塞控制用于避免网络过载，TCP通过**拥塞窗口（Congestion Window, cwnd）**实现。

#### 四个算法

| 阶段 | 算法 | 行为 |
|------|------|------|
| 慢启动 | Slow Start | cwnd指数增长 |
| 拥塞避免 | Congestion Avoidance | cwnd线性增长 |
| 快重传 | Fast Retransmit | 3个重复ACK触发 |
| 快恢复 | Fast Recovery | 调整cwnd |

#### 慢启动与拥塞避免

```cpp
class CongestionControl {
    int cwnd = 1;                    // 拥塞窗口（SMSS）
    int ssthresh = 65535;           // 慢启动阈值
    bool in_slow_start = true;
    
    void on_ack() {
        if (in_slow_start) {
            cwnd *= 2;               // 指数增长
            if (cwnd >= ssthresh) {
                in_slow_start = false;
            }
        } else {
            cwnd += 1;               // 线性增长
        }
    }
    
    void on_timeout() {
        ssthresh = cwnd / 2;
        cwnd = 1;
        in_slow_start = true;        // 回到慢启动
    }
    
    void on_fast_retransmit() {
        ssthresh = cwnd / 2;
        cwnd = ssthresh + 3;         // 快恢复
        in_slow_start = false;
    }
};
```

#### 拥塞窗口变化图

```
cwnd
  ^
  │           __________
  │          /          \_______
  │    ____/                      \___
  │___/                                \___
  │_________________________________________→ time
     SS    CA      FRT     FR      SS
     
SS = Slow Start, CA = Congestion Avoidance
FRT = Fast Retransmit, FR = Fast Recovery
```

### TCP状态机

```
    主动打开                                        被动打开
        │                                             │
        ▼                                             ▼
   CLOSED ────────▶LISTEN                           LISTEN
        │                │                                ▲
        │ 发送SYN        │                                │
        ▼                ▼                                │
    SYN_SENT ◀──────────▶SYN_RCVD ◀───────────────────────│
        │                │                                │
        │ 收到ACK       │收到ACK                          │
        ▼                ▼                                │
    ESTABLISHED ◀───────────────────────────────▶ESTABLISHED
        │                │                                │
        │ 发送FIN       │收到FIN                          │
        ▼                ▼                                │
    FIN_WAIT_1 ────────▶CLOSING ────────▶TIME_WAIT ◀───────┤
        │                │                                │
        │ 收到ACK       │收到ACK                          │
        ▼                ▼                                │
    FIN_WAIT_2 ◀────────┴───────────◀─────CLOSE_WAIT ◀─────┘
        │                               │ 发送FIN
        │            ┌──────────────────┘
        ▼            ▼
    TIME_WAIT ◀─────┘
```

---

## DNS（域名系统）

### DNS层次结构

```
根域名服务器 (.)
    │
    ├── .com (.com服务器)
    │     └── example.com (权威服务器)
    │
    ├── .cn (.cn服务器)
    │     └── baidu.com
    │
    └── .org (.org服务器)
```

### DNS查询过程

```
用户 ──▶ 递归服务器 ──▶ 根服务器 ──▶ .com服务器 ──▶ example.com
         ◀─────────── ◀────────── ◀──────────── ◀────────────
```

### DNS记录类型

| 类型 | 含义 | 示例 |
|------|------|------|
| A | IPv4地址 | example.com → 93.184.216.34 |
| AAAA | IPv6地址 | example.com → 2606:2800:220:1 |
| CNAME | 别名 | www.example.com → example.com |
| MX | 邮件交换 | mail.example.com → 10 mail.example.com |
| NS | 名称服务器 | example.com → ns1.example.com |

### DNS缓存与TTL

```cpp
class DNSCache {
    struct CacheEntry {
        string domain;
        string ip;
        time_t expire_time;
    };
    
    vector<CacheEntry> cache;
    
    string lookup(const string& domain) {
        for (auto& e : cache) {
            if (e.domain == domain && e.expire_time > now()) {
                return e.ip;              // 缓存命中
            }
        }
        // 缓存未命中，查询DNS服务器
        string ip = query_dns_server(domain);
        cache.push_back({domain, ip, now() + TTL});
        return ip;
    }
};
```

---

## CDN（内容分发网络）

### CDN工作原理

```
用户 ──▶ 最近的CDN边缘节点 ──▶ 源站
              │
              ├──── 缓存命中 ──▶ 直接返回
              │
              └──── 缓存未命中 ──▶ 回源获取 ──▶ 缓存 ──▶ 返回
```

### CDN关键技术

| 技术 | 说明 |
|------|------|
| 就近访问 | DNS负载均衡返回最近节点 |
| 缓存策略 | 边缘节点缓存静态内容 |
| 动态加速 | 专线/优化路由加速动态内容 |
| HTTPS | SSL/TLS终端卸载 |

### 负载均衡算法

```cpp
// 几种负载均衡策略
class LoadBalancer {
    vector<Server> servers;
    
    // 1. 轮询（Round Robin）
    Server round_robin() {
        static int idx = 0;
        return servers[idx++ % servers.size()];
    }
    
    // 2. 最少连接（Least Connections）
    Server least_connections() {
        return *min_element(servers.begin(), servers.end(),
            [](auto& a, auto& b) { return a.connections < b.connections; });
    }
    
    // 3. IP哈希
    Server ip_hash(const string& client_ip) {
        int hash = std::hash<string>{}(client_ip);
        return servers[hash % servers.size()];
    }
};
```

---

## 网络安全基础

### TLS/SSL握手

```
客户端                                服务器
  │                                    │
  │──────── ClientHello ──────────────▶│  1. 支持的加密套件
  │◀────── ServerHello + 证书 ─────────│  2. 选择加密算法 + 证书
  │◀────── ServerHelloDone ────────────│
  │                                    │
  │──────── 预主密钥（用公钥加密）──────▶│  3. 建立会话密钥
  │                                    │
  │  双方计算会话密钥                   │
  │                                    │
  │──────── Finished ─────────────────▶│  4. 切换到加密通信
  │◀────── Finished ───────────────────│
  │                                    │
  │        加密通信开始                 │
```

### 对称加密 vs 非对称加密

| 类型 | 优点 | 缺点 | 典型应用 |
|------|------|------|---------|
| 对称加密 | 速度快 | 密钥分发难 | 数据加密（AES） |
| 非对称加密 | 密钥分发方便 | 速度慢 | 密钥交换、数字签名 |

**TLS工作方式**：先用非对称加密交换会话密钥，再用对称加密传输数据。

---

## 相关协议对比

### TCP vs UDP

| 特性 | TCP | UDP |
|------|-----|-----|
| 连接性 | 面向连接 | 无连接 |
| 可靠性 | 可靠 | 不可靠 |
| 有序性 | 有序 | 无序 |
| 速度 | 较慢 | 快 |
| 首部开销 | 20-60字节 | 8字节 |
| 适用场景 | 文件传输、Web | 视频流、DNS |

### HTTP/1.1 vs HTTP/2 vs HTTP/3

| 版本 | 关键特性 | 传输层 |
|------|---------|-------|
| HTTP/1.1 | 持久连接、管道化 | TCP |
| HTTP/2 | 多路复用、头部压缩、Server Push | TCP |
| HTTP/3 | QUIC（UDP）、0-RTT | UDP |

---

## 参考资料

[^1]: 本词条参考《TCP/IP详解 卷1：协议》- W. Richard Stevens

---

## 相关主题

- [[networks-overview|计算机网络概述]] - 网络分层模型、IP协议基础
- [[http|HTTP协议详解]] - 应用层协议
