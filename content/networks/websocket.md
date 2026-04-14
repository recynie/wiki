---
title: WebSocket通信协议
date: 2026-04-13
description: WebSocket协议原理、握手机制与实时通信应用
tags:
  - networks
  - websocket
  - real-time
  - communication
draft: false
permalink:
---

# WebSocket通信协议

WebSocket是一种全双工通信协议，在单个TCP连接上提供实时双向数据传输。

## WebSocket vs HTTP

| 特性 | HTTP | WebSocket |
|------|------|----------|
| 方向 | 半双工 | 全双工 |
| 连接 | 短连接 | 长连接 |
| 通信方式 | 请求-响应 | 主动推送 |
| 头部开销 | 每次请求携带完整头 | 仅建立时握手 |
| 适用场景 | REST API | 实时应用 |

## 协议握手

### 客户端握手请求

```http
GET /ws HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

### 服务端握手响应

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

### 握手验证（Python实现）

```python
import hashlib
import base64
import secrets

def generate_websocket_key():
    """生成Sec-WebSocket-Key"""
    return base64.b64encode(secrets.token_bytes(16)).decode()

def compute_accept_key(key):
    """计算Sec-WebSocket-Accept"""
    GUID = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
    combined = key + GUID
    sha1_hash = hashlib.sha1(combined.encode()).digest()
    return base64.b64encode(sha1_hash).decode()

# 服务端验证示例
def validate_handshake(request_key):
    expected_accept = compute_accept_key(request_key)
    return True  # 验证通过后返回expected_accept
```

## 帧结构

### 数据帧格式

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)            |
|N|V|V|V|       |S|K|             |   (if payload len==126/127) |
| |1|2|3|       | | |             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|                               |         Masking-key            |
|                               |          (if mask=1)           |
+-------------------------------+ - - - - - - - - - - - - - - - +
|                     Payload data                               |
+---------------------------------------------------------------+
```

**opcode**：
- `0x0`：继续帧
- `0x1`：文本帧
- `0x2`：二进制帧
- `0x8`：关闭帧
- `0x9`：Ping帧
- `0xA`：Pong帧

### 掩码处理

客户端发送到服务端必须掩码：

```python
import struct

def mask_payload(mask_key, payload):
    """XOR掩码"""
    mask_bytes = mask_key.to_bytes(4, 'big')
    masked = bytearray()
    for i, byte in enumerate(payload):
        masked.append(byte ^ mask_bytes[i % 4])
    return bytes(masked)

def unmask_payload(mask_key, payload):
    """XOR去掩码"""
    return mask_payload(mask_key, payload)  # XOR两次等于原值
```

## WebSocket服务器实现

### Python实现

```python
import asyncio
import websockets
import json

async def chat_handler(websocket, path):
    # 注册客户端
    client_addr = websocket.remote_address
    print(f"Client connected: {client_addr}")
    
    # 加入聊天室
    clients.add(websocket)
    
    try:
        async for message in websocket:
            # 解析消息
            try:
                data = json.loads(message)
            except json.JSONDecodeError:
                data = {"type": "text", "content": message}
            
            # 广播消息
            broadcast_msg = {
                "type": "message",
                "sender": str(client_addr),
                "content": data.get("content", ""),
                "timestamp": asyncio.get_event_loop().time()
            }
            
            await broadcast(json.dumps(broadcast_msg))
            
    except websockets.exceptions.ConnectionClosed:
        print(f"Client disconnected: {client_addr}")
    finally:
        clients.remove(websocket)

async def broadcast(message):
    """广播到所有连接的客户端"""
    if clients:
        await asyncio.gather(
            *[client.send(message) for client in clients],
            return_exceptions=True
        )

# 客户端集合
clients = set()

# 启动服务器
start_server = websockets.serve(chat_handler, "localhost", 8765)
asyncio.get_event_loop().run_until_complete(start_server)
asyncio.get_event_loop().run_forever()
```

### 客户端JavaScript

```javascript
class WebSocketClient {
    constructor(url) {
        this.ws = new WebSocket(url);
        this.setupEventHandlers();
    }
    
    setupEventHandlers() {
        this.ws.onopen = () => {
            console.log('WebSocket connected');
            this.send({ type: 'join', room: 'main' });
        };
        
        this.ws.onmessage = (event) => {
            const data = JSON.parse(event.data);
            this.handleMessage(data);
        };
        
        this.ws.onerror = (error) => {
            console.error('WebSocket error:', error);
        };
        
        this.ws.onclose = () => {
            console.log('WebSocket disconnected');
            // 自动重连
            setTimeout(() => this.reconnect(), 3000);
        };
    }
    
    handleMessage(data) {
        switch (data.type) {
            case 'message':
                this.displayMessage(data);
                break;
            case 'system':
                this.showSystemMessage(data.content);
                break;
        }
    }
    
    send(data) {
        if (this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(JSON.stringify(data));
        }
    }
    
    reconnect() {
        console.log('Attempting to reconnect...');
        this.ws = new WebSocket(this.ws.url);
        this.setupEventHandlers();
    }
}

// 使用
const client = new WebSocketClient('ws://localhost:8765');
```

## 心跳机制

```python
async def heartbeat(websocket, interval=30):
    """定期发送Ping保持连接"""
    while True:
        try:
            await asyncio.sleep(interval)
            await websocket.ping()
        except Exception:
            break

# 定期检测连接活跃性
async def connection_keeper(websocket):
    last_pong = asyncio.get_event_loop().time()
    
    async def on_pong():
        nonlocal last_pong
        last_pong = asyncio.get_event_loop().time()
    
    websocket.on_pong = on_pong
    
    while True:
        await asyncio.sleep(10)
        current = asyncio.get_event_loop().time()
        if current - last_pong > 60:  # 超过60秒无响应
            await websocket.close()
            break
```

## WebSocket安全问题

### 源验证

```python
ALLOWED_ORIGINS = {'https://example.com', 'https://app.example.com'}

async def validate_origin(request):
    origin = request.headers.get('Origin')
    if origin not in ALLOWED_ORIGINS:
        return False, "Origin not allowed"
    return True, None
```

### 认证与授权

```python
# 在握手中验证Token
async def authenticate(websocket, path):
    # 从URL参数获取token
    parsed = urllib.parse.urlparse(path)
    token = urllib.parse.parse_qs(parsed.query).get('token')
    
    if not token:
        return False, "Missing token"
    
    # 验证JWT
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
        return True, payload
    except jwt.InvalidTokenError:
        return False, "Invalid token"
```

### 速率限制

```python
from collections import defaultdict
import time

class RateLimiter:
    def __init__(self, max_messages=100, window=60):
        self.max_messages = max_messages
        self.window = window
        self.messages = defaultdict(list)
    
    def is_allowed(self, client_id):
        now = time.time()
        # 清理过期记录
        self.messages[client_id] = [
            t for t in self.messages[client_id]
            if now - t < self.window
        ]
        
        if len(self.messages[client_id]) >= self.max_messages:
            return False
        
        self.messages[client_id].append(now)
        return True

rate_limiter = RateLimiter()

async def handle_message(websocket, message):
    client_id = websocket.remote_address
    
    if not rate_limiter.is_allowed(client_id):
        await websocket.send(json.dumps({
            'type': 'error',
            'message': 'Rate limit exceeded'
        }))
        return False
    
    return True
```

## 应用场景

| 场景 | 说明 |
|------|------|
| 聊天应用 | 实时消息推送 |
| 协作编辑 | 多用户同时编辑文档 |
| 游戏 | 低延迟游戏状态同步 |
| 金融行情 | 实时价格推送 |
| IoT设备 | 设备状态监控 |
| 通知系统 | 即时通知推送 |

## 参考

[^1]: RFC 6455 - The WebSocket Protocol
[^2]: WebSocket MDN Documentation

