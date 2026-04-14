---
title: Docker 容器化基础
date: 2026-04-08
description: Docker 容器化技术：镜像、容器、Dockerfile、Docker Compose
tags:
  - docker
  - container
  - devops
  - dockerfile
draft: false
permalink:
---

## Docker 简介

Docker是一个开源的容器化平台，让开发者能够打包应用及其依赖到可移植的容器中。[^1]

### 容器 vs 虚拟机

| 特性 | 容器 | 虚拟机 |
|------|------|--------|
| 启动时间 | 秒级 | 分钟级 |
| 资源占用 | 轻量（MB级） | 重（GB级） |
| 隔离性 | 进程级 | 硬件级 |
| 性能 | 近原生 | 有损耗 |

### 核心概念

- **镜像（Image）**：只读模板，包含应用和依赖
- **容器（Container）**：镜像的运行实例
- **仓库（Repository）**：存储和分发镜像的场所

---

## Docker 安装与配置

### Linux

```bash
# Ubuntu
sudo apt update
sudo apt install docker.io

# 启动Docker
sudo systemctl start docker
sudo systemctl enable docker

# 添加当前用户到docker组（免sudo）
sudo usermod -aG docker $USER
newgrp docker
```

### macOS / Windows

下载 [Docker Desktop](https://www.docker.com/products/docker-desktop) 安装包进行安装。

### 验证安装

```bash
docker --version
docker info
docker run hello-world
```

---

## 核心命令

### 镜像操作

```bash
# 拉取镜像
docker pull ubuntu:22.04
docker pull python:3.11-slim
docker pull nginx:latest

# 查看本地镜像
docker images
docker image ls

# 删除镜像
docker rmi ubuntu:22.04
docker rmi $(docker images -q)  # 删除所有镜像

# 构建镜像
docker build -t myapp:1.0 .

# 镜像标签
docker tag myapp:1.0 registry.com/myapp:1.0

# 推送镜像
docker push registry.com/myapp:1.0
```

### 容器操作

```bash
# 创建并启动容器
docker run -it ubuntu:22.04 /bin/bash
# -i: 交互模式
# -t: 分配终端

# 后台运行容器
docker run -d --name mynginx nginx
# -d: 后台运行
# --name: 指定容器名

# 查看运行中的容器
docker ps

# 查看所有容器（包括已停止）
docker ps -a

# 停止容器
docker stop mynginx
docker stop $(docker ps -q)  # 停止所有运行中的容器

# 启动已停止的容器
docker start mynginx

# 重启容器
docker restart mynginx

# 删除容器
docker rm mynginx
docker rm $(docker ps -aq)  # 删除所有容器

# 进入运行中的容器
docker exec -it mynginx /bin/bash
docker exec -it mynginx ls /app

# 查看容器日志
docker logs mynginx
docker logs -f mynginx  # 实时跟踪
docker logs --tail 100 mynginx  # 最后100行

# 查看容器信息
docker inspect mynginx
docker inspect --format '{{.NetworkSettings.IPAddress}}' mynginx
```

### 常用选项

```bash
# 端口映射
docker run -d -p 8080:80 nginx
# 主机端口:容器端口

# 卷挂载
docker run -d -v /host/data:/container/data nginx
docker run -d -v $(pwd)/app:/app:ro nginx  # 只读挂载

# 环境变量
docker run -d -e MYSQL_ROOT_PASSWORD=secret mysql

# 容器命名
docker run -d --name mypostgres postgres

# 资源限制
docker run -d --memory=512m --cpus=0.5 nginx

# 网络模式
docker run --network=host nginx  # 主机网络
```

---

## Dockerfile

Dockerfile是构建镜像的脚本文件。

### 常用指令

| 指令 | 说明 |
|------|------|
| `FROM` | 指定基础镜像 |
| `RUN` | 执行命令 |
| `COPY` | 复制文件 |
| `ADD` | 复制文件（支持URL和tar） |
| `WORKDIR` | 设置工作目录 |
| `ENV` | 设置环境变量 |
| `EXPOSE` | 声明端口 |
| `CMD` | 容器启动命令 |
| `ENTRYPOINT` | 入口点 |

### Python 应用示例

```dockerfile
# 基础镜像
FROM python:3.11-slim

# 设置工作目录
WORKDIR /app

# 复制依赖文件
COPY requirements.txt .

# 安装依赖（利用构建缓存）
RUN pip install --no-cache-dir -r requirements.txt

# 复制应用代码
COPY . .

# 环境变量
ENV FLASK_APP=app.py
ENV PYTHONUNBUFFERED=1

# 暴露端口
EXPOSE 5000

# 启动命令
CMD ["python", "app.py"]
```

### Node.js 应用示例

```dockerfile
# 多阶段构建
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .

EXPOSE 3000
CMD ["node", "server.js"]
```

### 构建缓存优化

```dockerfile
# 顺序很重要：频繁变化的文件放最后
COPY package*.json ./
RUN npm ci
COPY . .  # 应用代码放最后
```

### .dockerignore

```dockerignore
node_modules
.git
.env
*.log
dist
coverage
```

---

## Docker Compose

Docker Compose用于定义和运行多容器应用。

### docker-compose.yml

```yaml
version: '3.8'

services:
  # Web应用
  web:
    build: .
    ports:
      - "8080:5000"
    environment:
      - REDIS_HOST=redis
      - DB_HOST=db
    depends_on:
      - redis
      - db
    volumes:
      - ./app:/app
    networks:
      - app-network

  # Redis缓存
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - app-network

  # 数据库
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-network

volumes:
  redis-data:
  postgres-data:

networks:
  app-network:
    driver: bridge
```

### 常用命令

```bash
# 启动所有服务
docker-compose up

# 后台运行
docker-compose up -d

# 重新构建镜像
docker-compose up --build

# 停止并删除
docker-compose down

# 查看日志
docker-compose logs -f web

# 扩展服务
docker-compose up -d --scale web=3

# 执行命令
docker-compose exec web python manage.py migrate
```

---

## 网络与存储

### 网络模式

| 模式 | 说明 |
|------|------|
| bridge | 默认模式，容器间通信 |
| host | 容器使用主机网络 |
| overlay | Docker Swarm跨主机网络 |
| none | 无网络 |

```bash
# 创建网络
docker network create mynetwork

# 容器连接到网络
docker run -d --network=mynetwork --name=app1 nginx

# 容器间通信（通过DNS）
docker run -d --network=mynetwork --name=app2 nginx
# app1 可以通过 app2 这个名字访问 app2
```

### 存储卷

```bash
# 创建卷
docker volume create myvolume

# 查看卷
docker volume ls
docker volume inspect myvolume

# 容器使用卷
docker run -d -v myvolume:/data nginx

# 主机路径挂载
docker run -d -v /host/path:/container/path nginx
```

---

## 实战：完整Web应用

### 项目结构

```
project/
├── app/
│   ├── app.py
│   └── templates/
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
└── .dockerignore
```

### Flask 应用

```python
# app/app.py
from flask import Flask, jsonify
import redis
import os

app = Flask(__name__)

redis_client = redis.Redis(
    host=os.environ.get('REDIS_HOST', 'localhost'),
    port=6379
)

@app.route('/')
def index():
    return jsonify({'message': 'Hello from Docker!'})

@app.route('/count')
def count():
    count = redis_client.incr('hits')
    return jsonify({'hits': count})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### requirements.txt

```
flask==3.0.0
redis==5.0.1
gunicorn==21.2.0
```

### Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ ./app/

ENV FLASK_APP=app/app.py

EXPOSE 5000

CMD ["gunicorn", "-b", "0.0.0.0:5000", "app:app"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "8080:5000"
    environment:
      REDIS_HOST: redis
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

### 启动

```bash
docker-compose up --build
curl http://localhost:8080/count
```

---

## 最佳实践

### 镜像优化

1. **使用轻量级基础镜像**
   ```dockerfile
   # 推荐
   FROM python:3.11-slim
   FROM alpine:latest
   
   # 避免
   FROM python:3.11  # 太大
   FROM ubuntu:latest
   ```

2. **多阶段构建**
   ```dockerfile
   # 构建阶段
   FROM golang:1.21 AS builder
   WORKDIR /app
   COPY . .
   RUN go build -o myapp

   # 运行阶段
   FROM alpine:latest
   COPY --from=builder /app/myapp /usr/local/bin/
   CMD ["myapp"]
   ```

3. **减少层数**
   ```dockerfile
   # 合并RUN指令
   RUN apt-get update && \
       apt-get install -y package1 package2 && \
       apt-get clean && rm -rf /var/lib/apt/lists/*
   ```

4. **利用构建缓存**
   ```dockerfile
   # 变化少的放前面
   COPY requirements.txt .
   RUN pip install -r requirements.txt
   COPY . .
   ```

### 安全建议

1. **不要用root运行容器**
   ```dockerfile
   RUN addgroup -S appgroup && adduser -S appuser -G appgroup
   USER appuser
   ```

2. **扫描镜像漏洞**
   ```bash
   docker scan myimage:tag
   ```

3. **使用可信基础镜像**
   ```dockerfile
   FROM python:3.11.0@sha256:xxx  # 指定摘要
   ```

---

## 参考资料

[^1]: Docker Official Documentation. https://docs.docker.com/

[^2]: Docker Deep Dive - Nigel Poulton. https://www.docker.com/resources/

[^3]: Best practices for writing Dockerfiles. https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
