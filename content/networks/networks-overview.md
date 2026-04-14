---
title: 计算机网络概述
date: 2026-04-06
description: 计算机网络是由地理上分散的计算机通过通信设备和线路连接起来的系统
tags:
  - networks
  - fundamentals
draft: false
permalink:
---

# 计算机网络概述

计算机网络是现代信息社会的基础设施，实现了硬件、软件和数据的共享。

## 网络分层模型

### OSI七层模型

| 层级 | 名称 | 功能 | 典型协议 |
|------|------|------|---------|
| 7 | 应用层 | 提供网络服务 | HTTP, FTP, DNS |
| 6 | 表示层 | 数据格式转换 | TLS, SSL |
| 5 | 会话层 | 会话管理 | NetBIOS |
| 4 | 传输层 | 端到端传输 | TCP, UDP |
| 3 | 网络层 | 路由与转发 | IP, ICMP |
| 2 | 数据链路层 | 相邻节点传输 | Ethernet |
| 1 | 物理层 | 比特传输 | 光纤, 电缆 |

### TCP/IP四层模型

```
应用层（HTTP, FTP, DNS）
传输层（TCP, UDP）
网络层（IP）
网络接口层（Ethernet）
```

## IP协议

### IPv4地址

IPv4地址是32位二进制数，通常用点分十进制表示：

```
11000000.10101000.00000001.00000001
↓ 点分十进制
192.168.1.1
```

### IP地址分类

| 类别 | 范围 | 用途 |
|------|------|------|
| A类 | 0.0.0.0 - 127.255.255.255 | 大型网络 |
| B类 | 128.0.0.0 - 191.255.255.255 | 中型网络 |
| C类 | 192.0.0.0 - 223.255.255.255 | 小型网络 |
| D类 | 224.0.0.0 - 239.255.255.255 | 多播 |
| E类 | 240.0.0.0 - 255.255.255.255 | 保留 |

### 子网掩码

子网掩码用于划分网络号和主机号：

```
IP:      192.168.1.100
Mask:    255.255.255.0
Network: 192.168.1.0
Host:    100
```

### C++实现IP地址处理

```cpp
#include <bits/stdc++.h>
using namespace std;

struct IPv4 {
    uint8_t a, b, c, d;
    
    uint32_t toInt() const {
        return (a << 24) | (b << 16) | (c << 8) | d;
    }
    
    static IPv4 fromInt(uint32_t x) {
        return {(uint8_t)(x >> 24), (uint8_t)(x >> 16),
                (uint8_t)(x >> 8), (uint8_t)x};
    }
    
    bool inSameSubnet(const IPv4& other, const IPv4& mask) const {
        return (toInt() & mask.toInt()) == (other.toInt() & mask.toInt());
    }
};
```

## TCP协议

### TCP三次握手

```
客户端                  服务器
  |                       |
  |-------- SYN ----------->|
  |<------ SYN-ACK -------|
  |-------- ACK ----------->|
  |                       |
  |    建立连接完成       |
```

### TCP四次挥手

```
客户端                  服务器
  |                       |
  |-------- FIN ----------->|
  |<------ ACK -----------|
  |<------ FIN -----------|
  |-------- ACK ----------->|
  |                       |
  |    断开连接完成       |
```

### TCP状态转换

```cpp
enum class TCPState {
    CLOSED,
    LISTEN,
    SYN_SENT,
    SYN_RECEIVED,
    ESTABLISHED,
    FIN_WAIT_1,
    FIN_WAIT_2,
    CLOSE_WAIT,
    CLOSING,
    LAST_ACK,
    TIME_WAIT
};
```

### TCP滑动窗口

滑动窗口用于流量控制和拥塞控制：

```cpp
struct SlidingWindow {
    int base;           // 最早未确认字节序号
    int next_seq;       // 下一个要发送的字节序号
    int window_size;    // 窗口大小
    unordered_map<int, Packet> buffer;  // 已发送未确认的包
    
    void send(int seq, const string& data) {
        Packet p{seq, data};
        buffer[seq] = p;
        // 实际发送逻辑
        next_seq += data.size();
    }
    
    bool receiveAck(int ack) {
        if (ack >= base) {
            // 删除已确认的包
            for (auto it = buffer.begin(); it != buffer.end();) {
                if (it->first < ack) it = buffer.erase(it);
                else ++it;
            }
            base = ack;
            return true;
        }
        return false;
    }
};
```

## UDP协议

UDP是无连接的传输协议，开销小，速度快：

```cpp
// UDP服务端
int server_fd = socket(AF_INET, SOCK_DGRAM, 0);
struct sockaddr_in addr = {AF_INET, htons(8080), INADDR_ANY};

bind(server_fd, (struct sockaddr*)&addr, sizeof(addr));

char buf[1024];
struct sockaddr_in client;
socklen_t len = sizeof(client);
recvfrom(server_fd, buf, sizeof(buf), 0, 
         (struct sockaddr*)&client, &len);

sendto(server_fd, buf, strlen(buf), 0,
       (struct sockaddr*)&client, len);
```

## HTTP协议

### 请求方法

| 方法 | 说明 | 幂等性 |
|------|------|--------|
| GET | 获取资源 | 幂等 |
| POST | 提交数据 | 非幂等 |
| PUT | 更新资源 | 幂等 |
| DELETE | 删除资源 | 幂等 |

### HTTP请求/响应结构

```
请求:
GET /index.html HTTP/1.1\r\n
Host: example.com\r\n
\r\n

响应:
HTTP/1.1 200 OK\r\n
Content-Type: text/html\r\n
Content-Length: 123\r\n
\r\n
<html>...
```

### C++实现简单的HTTP客户端

```cpp
#include <bits/stdc++.h>
#include <sys/socket.h>
#include <netinet/in.h>
using namespace std;

string httpGet(const string& host, const string& path) {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    
    struct hostent* server = gethostbyname(host.c_str());
    struct sockaddr_in addr = {
        AF_INET,
        htons(80),
        *(struct in_addr*)server->h_addr
    };
    
    connect(sock, (struct sockaddr*)&addr, sizeof(addr));
    
    string request = "GET " + path + " HTTP/1.1\r\n"
                     "Host: " + host + "\r\n"
                     "Connection: close\r\n"
                     "\r\n";
    
    send(sock, request.c_str(), request.size(), 0);
    
    string response;
    char buf[4096];
    int n;
    while ((n = recv(sock, buf, sizeof(buf), 0)) > 0) {
        response.append(buf, n);
    }
    
    close(sock);
    return response;
}
```

## 路由算法

### Dijkstra最短路径

路由协议使用Dijkstra算法计算最短路径：

```cpp
// 简化的路由表计算
struct Router {
    int id;
    unordered_map<int, int> dist;  // 到各节点的距离
    unordered_map<int, int> next_hop;  // 下一跳
    
    void runDijkstra(const vector<vector<pair<int,int>>>& graph) {
        const int INF = 1e9;
        int n = graph.size();
        vector<int> d(n, INF);
        vector<bool> visited(n, false);
        d[id] = 0;
        
        for (int i = 0; i < n; ++i) {
            int u = -1;
            for (int j = 0; j < n; ++j) {
                if (!visited[j] && (u == -1 || d[j] < d[u])) {
                    u = j;
                }
            }
            if (d[u] == INF) break;
            visited[u] = true;
            
            for (auto [v, w] : graph[u]) {
                if (d[u] + w < d[v]) {
                    d[v] = d[u] + w;
                    next_hop[v] = (v == u ? v : next_hop[u]);
                }
            }
        }
        dist = unordered_map<int, int>(d.begin(), d.end());
    }
};
```

## 参考

- [计算机网络 - 维基百科](https://zh.wikipedia.org/wiki/计算机网络)
- [TCP/IP协议栈](https://tools.ietf.org/rfc/)
