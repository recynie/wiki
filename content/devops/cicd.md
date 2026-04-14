---
title: CI/CD 基础 (CI/CD Basics)
date: 2026-04-09
description: 持续集成与持续部署基础概念与实践
tags:
  - cicd
  - devops
  - automation
draft: false
permalink:
---

# CI/CD 基础 (CI/CD Basics)

CI/CD（持续集成/持续部署）是现代软件开发的核心理念，通过自动化构建、测试和部署流程，提高软件交付效率和质量。[^1]

[^1]: 本段参考[Jenkins 文档](https://www.jenkins.io/doc/) 和 [GitHub Actions 文档](https://docs.github.com/en/actions)

## CI/CD 概念

### 持续集成 (Continuous Integration)

开发者频繁地将代码集成到主干（通常每天多次）。每次集成通过**自动化构建**和**测试**进行验证。

**核心实践**：
- 提交代码后自动触发构建
- 所有测试必须通过才能合并
- 保持构建快速完成
- 立即修复构建失败

### 持续交付 (Continuous Delivery)

代码变更自动准备好发布到测试/预生产环境。在手动控制下进行生产部署。

### 持续部署 (Continuous Deployment)

代码变更自动部署到生产环境，无需人工干预。

```
CI/CD Pipeline 流程：

代码提交 → 自动化测试 → 自动化构建 → 部署到测试环境
                                              │
                                    ◄─────────┘
                                    （持续交付需要手动批准）
                                              │
                                              ▼
                                    部署到生产环境
                                              │
                                    （持续部署自动执行）
```

## CI/CD 流水线 (Pipeline)

### 典型流水线阶段

| 阶段 | 说明 | 工具 |
|------|------|------|
| 源码检出 | 拉取代码 | Git |
| 依赖安装 | 安装项目依赖 | npm, pip, Maven |
| 代码检查 | Lint, 静态分析 | ESLint, Pylint |
| 单元测试 | 运行快速测试 | Jest, pytest |
| 集成测试 | 运行集成测试 | Selenium, Cypress |
| 构建 | 编译/打包 | gcc, webpack |
| 镜像构建 | Docker 镜像 | Docker |
| 部署 | 部署到环境 | kubectl, terraform |
| 验证 | 健康检查 | curl, monitoring |

### 流水线示例

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # 代码检查阶段
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install flake8 black
      
      - name: Run linters
        run: |
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
          black --check .
      
      - name: Check types
        run: |
          pip install mypy
          mypy .

  # 测试阶段
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Run tests with pytest
        run: |
          pytest tests/ --cov=src --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml

  # 构建阶段
  build:
    needs: [lint, test]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build Docker image
        run: |
          docker build -t myapp:${{ github.sha }} .
          docker tag myapp:${{ github.sha }} myregistry/myapp:latest
      
      - name: Push to registry
        run: |
          echo ${{ secrets.DOCKER_TOKEN }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
          docker push myregistry/myapp:latest

  # 部署阶段
  deploy:
    needs: [build]
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Kubernetes
        run: |
          kubectl config set-cluster k8s-cluster --server=${{ secrets.K8S_SERVER }}
          kubectl config set-credentials deployer --token=${{ secrets.K8S_TOKEN }}
          kubectl config use-context k8s-cluster
          kubectl rollout restart deployment/myapp
```

## GitHub Actions

### 基本概念

- **Workflow**：整个自动化流程
- **Job**：一组步骤（Step）
- **Step**：具体的操作
- **Action**：可复用的步骤单元

### 工作流文件结构

```yaml
name: Workflow Name

# 触发条件
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # 每天凌晨2点
  workflow_dispatch:  # 允许手动触发

# 环境变量
env:
  NODE_VERSION: '18'

# 任务定义
jobs:
  job-name:
    runs-on: ubuntu-latest
    
    # 条件执行
    if: github.event_name == 'push'
    
    # 容器配置
    container:
      image: node:18
      options: --user root
    
    # 步骤
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup
        run: npm ci
      
      - name: Build
        run: npm run build
      
      - name: Test
        run: npm test
```

### 常用 Action

| Action | 用途 |
|--------|------|
| `actions/checkout` | 检出代码 |
| `actions/setup-node` | 设置 Node.js |
| `actions/setup-python` | 设置 Python |
| `docker/build-push-action` | 构建并推送 Docker 镜像 |
| `aws-actions/configure-aws-credentials` | 配置 AWS 凭证 |
| `hashicorp/setup-terraform` | 设置 Terraform |

### 矩阵构建

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node-version: [16, 18, 20]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm test
```

## Jenkins

### Jenkinsfile 示例

```groovy
// Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'registry.example.com'
        IMAGE_NAME = 'myapp'
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 1, unit: 'HOURS')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                script {
                    def image = docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}:${env.BUILD_NUMBER}")
                }
            }
        }
        
        stage('Test') {
            steps {
                parallel(
                    "Unit Tests": {
                        sh 'npm run test:unit'
                    },
                    "Integration Tests": {
                        sh 'npm run test:integration'
                    }
                )
            }
        }
        
        stage('Security Scan') {
            steps {
                sh 'trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKER_REGISTRY}/${IMAGE_NAME}:${env.BUILD_NUMBER}'
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                sh '''
                    kubectl config use-context staging
                    kubectl set image deployment/myapp app=${DOCKER_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}
                    kubectl rollout status deployment/myapp
                '''
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                sh '''
                    kubectl config use-context production
                    kubectl set image deployment/myapp app=${DOCKER_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}
                    kubectl rollout status deployment/myapp
                '''
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
```

## Docker 与镜像构建

### Dockerfile 优化

```dockerfile
# 多阶段构建
# 阶段1：构建
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# 阶段2：运行
FROM node:18-alpine AS runner
WORKDIR /app

# 从构建阶段复制产物
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

# 使用非 root 用户
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### 镜像安全扫描

```bash
# 使用 Trivy 扫描
trivy image --severity HIGH,CRITICAL myapp:latest

# 使用 Grype
grype myapp:latest

# 使用 Clair
clair-scanner myapp:latest
```

## 自动化测试

### 测试金字塔

```
           ┌───────────┐
           │   E2E     │  ← 少量，耗时
           ├───────────┤
           │ Integration│  ← 中等量
           ├───────────┤
           │   Unit     │  ← 大量，快速
           └───────────┘
```

### 测试配置示例

```yaml
# pytest.yml (GitHub Action)
- name: Run pytest
  run: |
    pytest tests/ \
      --cov=src \
      --cov-report=xml \
      --cov-report=html \
      --junitxml=test-results.xml \
      --tb=short
- name: Upload test results
  uses: actions/upload-artifact@v3
  if: always()
  with:
    name: test-results
    path: test-results.xml
```

## 部署策略

### 蓝绿部署 (Blue-Green Deployment)

```
旧版本 (Blue) ◄──── 流量切换 ────► 新版本 (Green)
  :3000                              :3001
```

```yaml
# Kubernetes 蓝绿部署
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: green  # 切换这个标签
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myapp:green
        ports:
        - containerPort: 3000
```

### 金丝雀发布 (Canary Release)

逐步将流量从旧版本切换到新版本：

```yaml
# 10% 流量到新版本
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  port: 80
  targetPort: 3000
---
apiVersion: v1
kind: Pod
metadata:
  name: myapp-canary
  labels:
    app: myapp
    version: canary
spec:
  containers:
  - name: myapp
    image: myapp:canary
---
# 90% 流量通过标签选择器
```

### 滚动更新 (Rolling Update)

Kubernetes 默认策略，逐步替换旧版本 Pod：

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

## 环境管理

### GitOps 工作流

```
开发者 ──► Git ──► CI Pipeline ──► 自动部署到测试
                            │
                            ▼
                      GitOps Operator
                            │
                            ▼
                   自动同步到生产集群
```

### ArgoCD 示例

```yaml
# app.yaml (ArgoCD Application)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp.git
    targetRevision: HEAD
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## 监控与反馈

### 构建通知

```yaml
- name: Notify on failure
  if: failure()
  run: |
    curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
      -H 'Content-Type: application/json' \
      -d '{"text": "Build failed! :x: <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View logs>"}'
```

### 构建徽章

```markdown
[![CI](https://github.com/myorg/myapp/actions/workflows/ci.yml/badge.svg)](https://github.com/myorg/myapp/actions/workflows/ci.yml)
```

## 工具对比

| 工具 | 类型 | 特点 |
|------|------|------|
| GitHub Actions | CI/CD | 与 GitHub 深度集成，免费额度 |
| Jenkins | CI/CD | 插件丰富，自托管 |
| GitLab CI | CI/CD | GitLab 内置，YAML 配置 |
| CircleCI | CI/CD | 云原生，快速 |
| ArgoCD | CD (GitOps) | Kubernetes 原生 |
| Tekton | CD (K8s) | Kubernetes 原生，标准格式 |

## 最佳实践

1. **保持流水线快速**：测试分层，避免不必要的步骤
2. **使用缓存**：缓存依赖加速构建
3. **失败早期检测**：将快速测试放在前面
4. **不可变镜像**：镜像构建后不修改
5. **版本化配置**：所有配置在 Git 中管理
6. **原子部署**：部署要么完全成功，要么回滚
7. **监控部署**：部署后立即验证

## 参考

- [GitHub Actions 文档](https://docs.github.com/en/actions)
- [Jenkins 文档](https://www.jenkins.io/doc/)
- [ArgoCD 文档](https://argo-cd.readthedocs.io/)
- [GitOps](https://www.gitops.tech/)
