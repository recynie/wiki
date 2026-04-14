---
title: CDN（内容分发网络）
date: 2026-04-13
description: 深入理解CDN工作原理、架构设计及最佳实践
tags:
  - cdn
  - networks
  - performance
draft: false
permalink:
---

## 概述

CDN（Content Delivery Network，内容分发网络）是由地理上分布式部署的边缘服务器组成的网络，通过将内容缓存到离用户更近的位置来加速内容交付。[^1]

Google研究发现，搜索结果延迟400毫秒用户流失率显著增加。在当今互联网时代，如果你的内容加载慢，用户会悄然离开。CDN的存在确保这不会发生——它们静默地加速每一个图片、API调用、视频片段和软件更新。

## CDN解决的核心问题

### 物理限制

数据以光速在光纤中传输，但距离造成的往返延迟仍然可观：

```
用户（东京）→ 源服务器（纽约）：往返约200ms
用户（东京）→ 东京边缘服务器：往返约5ms
```

### 三大挑战

1. **距离**：物理距离直接决定延迟
2. **容量**：源服务器难以应对突发流量
3. **可用性**：单点故障导致服务中断

## 工作原理

### 架构组件

```
┌─────────────────────────────────────────────────────────────┐
│                        CDN架构                               │
│                                                             │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐               │
│  │ PoP 1   │     │ PoP 2   │     │ PoP 3   │   ← 边缘节点  │
│  │ Tokyo   │     │ LA      │     │ NY      │               │
│  └────┬────┘     └────┬────┘     └────┬────┘               │
│       │               │               │                     │
│       └───────────────┼───────────────┘                     │
│                       │                                     │
│                 ┌─────┴─────┐                              │
│                 │   Shield   │  ← 源站保护层                │
│                 └─────┬─────┘                              │
│                       │                                     │
│                 ┌─────┴─────┐                              │
│                 │   Origin  │  ← 源服务器                  │
│                 └───────────┘                              │
└─────────────────────────────────────────────────────────────┘
```

### 请求流程

```
用户浏览器
    │
    ▼
┌─────────────┐
│ DNS解析     │ ← CNAME指向CDN域名
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ 路由到最近  │ ← Anycast/BGP路由
│ 边缘节点   │
└──────┬──────┘
       │
       ▼
┌─────────────┐     ┌─────────────┐
│ 缓存命中？  │ ──Yes──▶ 直接返回  │
└──────┬──────┘       └─────────────┘
       │ No
       ▼
┌─────────────┐
│ 缓存未命中  │ → 获取源站内容 → 缓存 → 返回
└─────────────┘
```

### DNS如何引导流量

```bash
# 用户请求 www.example.com
# 权威DNS返回 CNAME: www.example.com → customer.cdnprovider.net
# CDN DNS返回 Anycast IP（根据用户位置选择最近PoP）
```

## 缓存机制

### 缓存键（Cache Key）

CDN使用请求的以下部分作为缓存键：
- HTTP方法（GET、POST等）
- URL路径
- 查询参数（部分）
- Accept-Encoding

### 缓存指令

通过HTTP响应头控制：

```http
# 缓存1天（public）
Cache-Control: public, max-age=86400

# 不缓存
Cache-Control: no-store

# 缓存但需验证
Cache-Control: no-cache, max-age=0

# 重新验证
Cache-Control: must-revalidate, max-age=86400

# 边缘缓存5分钟，浏览器10分钟
Cache-Control: s-maxage=300, max-age=600
```

### 验证机制

```http
# 条件请求
If-None-Match: "etag-123"
If-Modified-Since: Wed, 21 Oct 2015 07:28:00 GMT

# 响应
HTTP/1.1 304 Not Modified  # 缓存有效
HTTP/1.1 200 OK            # 内容变化，返回新内容
```

### Stale-while-revalidate

允许在后台更新缓存，减少用户等待：

```http
Cache-Control: max-age=600, stale-while-revalidate=30
```

## 架构类型

### Pull CDN（被动缓存）

边缘服务器**按需**从源站拉取内容：

- **优点**：无需手动管理内容发布
- **缺点**：首次访问较慢（缓存未命中）
- **适用**：大多数网站和Web应用

### Push CDN（主动推送）

内容**主动推送**到边缘节点：

- **优点**：预热缓存，首次访问即命中
- **缺点**：需要手动管理内容同步
- **适用**：大型媒体文件、软件分发

### 反向代理型

CDN位于源站前方，拦截所有流量：

```
Client → CDN Edge → CDN Shield → Origin Server
```

## Anycast路由

现代CDN使用Anycast进行流量调度：

1. 相同的IP前缀从多个地理位置**同时宣告**
2. 互联网路由协议（BGP）自动选择最近路径
3. 用户请求被路由到最近的PoP

**优势**：
- 自动故障转移
- DDoS攻击流量分散吸收
- 无需DNS变更即可切换路由

## CDN高级功能

### 边缘计算

现代CDN不仅是缓存服务器，还能执行边缘逻辑：

```javascript
// Cloudflare Workers示例
addEventListener('fetch', event => {
    event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
    const url = new URL(request.url)
    
    // 边缘重写
    if (url.pathname === '/api/users') {
        url.pathname = '/api/v1/users'
    }
    
    return fetch(url)
}
```

### 图像优化

```html
<!-- 自动格式转换和压缩 -->
<img src="//cdn.example.com/photo.jpg" 
     width="800" 
     loading="lazy">

<!-- 响应式图像 -->
<img srcset="small.jpg 480w, large.jpg 1200w"
     sizes="(max-width: 600px) 480px, 1200px">
```

### TLS终止

CDN边缘节点完成TLS握手：

```
用户 → CDN Edge: TLS握手（快速，本地）
CDN Edge → 源站: 内部安全连接
```

## 安全功能

### DDoS防护

CDN分布式架构天然抵御DDoS：

- 流量分散到全球节点
- 边缘节点过滤恶意请求
- 自动速率限制

### Web应用防火墙（WAF）

```yaml
# 规则示例
- name: block-sql-injection
  match: request.uri contains "' OR 1=1"
  action: block

- name: rate-limit-api
  match: request.uri startsWith "/api/"
  action: rate-limit 100/minute
```

### 证书管理

```bash
# 自动HTTPS
*.example.com → 自动申请并续期SSL证书
```

## 性能指标

### 关键指标

| 指标 | 说明 | 目标 |
|------|------|------|
| TTFB | Time To First Byte | < 200ms |
| 缓存命中率 | Cache Hit Ratio | > 95% |
| 边缘延迟 | Edge Latency | < 50ms |
| 可用性 | Uptime | > 99.99% |

### 优化策略

1. **提高缓存命中率**：
   - 使用内容哈希（`/app.js?v=abc123`）
   - 合理的TTL设置
   - 避免缓存绕过

2. **减少源站负载**：
   - 启用Brotli压缩
   - 合并小文件
   - 使用骨架屏/预加载

3. **优化移动体验**：
   - 边缘压缩
   - 协议升级（HTTP/2、HTTP/3）
   - 减少DNS查询

## 适用场景

| 场景 | 推荐CDN类型 | 原因 |
|------|------------|------|
| 电商网站 | Pull + 动态加速 | 大量静态内容 + 频繁促销 |
| 视频流媒体 | Push + 直播 | 大文件、预缓存 |
| 游戏/软件分发 | Push | 大文件、版本控制 |
| API加速 | 边缘计算 | 动态内容、边缘鉴权 |
| 全球企业应用 | 多CDN | 高可用、地理覆盖 |

## CDN选择因素

1. **PoP分布**：节点数量和地理位置
2. **缓存效率**：命中率和技术优化
3. **安全功能**：WAF、DDoS防护
4. **边缘计算**：是否能执行自定义逻辑
5. **成本模型**：请求量 vs 带宽
6. **API和监控**：可观测性

## 最佳实践

1. **分离静态与动态内容**
   ```html
   <!-- 静态资源走CDN -->
   <script src="//cdn.example.com/app.js">
   
   <!-- 动态API直连 -->
   <fetch src="/api/users">
   ```

2. **使用长期缓存 + 内容哈希**
   ```bash
   # 文件名包含内容哈希
   app.abc123.js  # 内容变化 → 新哈希
   ```

3. **配置合适的CORS头**
   ```http
   Access-Control-Allow-Origin: https://example.com
   ```

4. **监控缓存命中率**
   ```javascript
   // CDN分析面板
   // 目标: > 95% 缓存命中率
   ```

5. **正确设置源站健康检查**
   ```yaml
   origin:
     health_check:
       path: /health
       interval: 10s
       timeout: 5s
   ```

## 参考资料

[^1]: [Content Delivery Networks (CDN) Explained - NOC.org](https://noc.org/learn/what-is-a-cdn)
