---
title: Kubernetes 基础
date: 2026-04-08
description: Kubernetes（K8s）容器编排平台：Pod、Deployment、Service、存储与网络
tags:
  - kubernetes
  - k8s
  - pod
  - service
  - deployment
  - devops
draft: false
permalink:
---

## Kubernetes 简介

Kubernetes（K8s）是Google开源的容器编排平台，用于自动化容器化应用的部署、扩缩容和管理。[^1]

### 核心功能

- **容器编排**：自动调度容器到合适的节点
- **服务发现与负载均衡**：容器间通信与流量分发
- **自动扩缩容**：根据负载调整容器数量
- **自我修复**：故障自动重启与恢复
- **配置管理与密钥管理**：安全存储配置信息

### 架构组件

```
┌─────────────────────────────────────────────────┐
│                   Control Plane                  │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐          │
│  │ kube-   │  │ kube-   │  │ etcd   │          │
│  │ apiserver│ │scheduler│  │         │          │
│  └─────────┘  └─────────┘  └─────────┘          │
└─────────────────────────────────────────────────┘
                        │
┌─────────────────────────────────────────────────┐
│                    Worker Nodes                  │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐    │
│  │  Node 1   │  │  Node 2   │  │  Node 3   │    │
│  │ ┌───────┐ │  │ ┌───────┐ │  │ ┌───────┐ │    │
│  │ │ kubelet│ │  │ │ kubelet│ │  │ │ kubelet│ │    │
│  │ └───────┘ │  │ └───────┘ │  │ └───────┘ │    │
│  │ ┌───────┐ │  │ ┌───────┐ │  │ ┌───────┐ │    │
│  │ │kube-  │ │  │ │kube-  │ │  │ │kube-  │ │    │
│  │ │proxy  │ │  │ │proxy  │ │  │ │proxy  │ │    │
│  │ └───────┘ │  │ └───────┘ │  │ └───────┘ │    │
│  └───────────┘  └───────────┘  └───────────┘    │
└─────────────────────────────────────────────────┘
```

---

## 核心对象

### Pod

Pod是Kubernetes的最小调度单元，代表一个或多个共享网络的容器。

#### Pod YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: production
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
      env:
        - name: NGINX_HOST
          value: "localhost"
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 3
        periodSeconds: 10
```

#### 创建和操作

```bash
# 创建Pod
kubectl apply -f nginx-pod.yaml

# 查看Pod
kubectl get pods
kubectl get pods -o wide  # 详细信息
kubectl describe pod nginx-pod

# 查看Pod日志
kubectl logs nginx-pod
kubectl logs -f nginx-pod  # 实时跟踪

# 进入Pod（调试用）
kubectl exec -it nginx-pod -- /bin/bash

# 端口转发（本地访问）
kubectl port-forward nginx-pod 8080:80

# 删除Pod
kubectl delete pod nginx-pod
```

### Deployment

Deployment管理Pod的部署和扩缩容。

#### Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
```

#### Deployment 操作

```bash
# 创建Deployment
kubectl apply -f nginx-deployment.yaml

# 查看Deployment
kubectl get deployments
kubectl get deploy

# 查看ReplicaSet
kubectl get rs

# 扩缩容
kubectl scale deployment nginx-deployment --replicas=5
kubectl scale deployment nginx-deployment --replicas=2

# 更新镜像
kubectl set image deployment/nginx-deployment nginx=nginx:1.26

# 查看滚动更新状态
kubectl rollout status deployment/nginx-deployment

# 回滚
kubectl rollout undo deployment/nginx-deployment
kubectl rollout history deployment/nginx-deployment

# 重启
kubectl rollout restart deployment/nginx-deployment
```

### Service

Service为一组Pod提供稳定的访问入口。

#### Service 类型

| 类型 | 说明 | 适用场景 |
|------|------|---------|
| **ClusterIP** | 集群内部IP | 内部服务通信 |
| **NodePort** | 节点端口（30000-32767） | 开发测试 |
| **LoadBalancer** | 云厂商负载均衡 | 生产环境 |
| **ExternalName** | DNS别名 | 外部服务访问 |

#### Service YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80        # Service端口
      targetPort: 80  # Pod端口
```

#### NodePort Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080  # 可选，指定节点端口
```

#### LoadBalancer Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

#### Service 操作

```bash
# 创建Service
kubectl apply -f nginx-service.yaml

# 查看Service
kubectl get services
kubectl get svc

# 查看Endpoints
kubectl get endpoints nginx-service

# 测试Service（集群内部）
kubectl run curl --image=curlimages/curl -it -- sh
# 在Pod内执行: curl http://nginx-service
```

### StatefulSet

用于有状态应用，如数据库。

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

### ConfigMap 和 Secret

#### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "localhost"
  LOG_LEVEL: "info"
  config.json: |
    {
      "feature_flags": {
        "new_ui": true
      }
    }
```

```bash
# 创建
kubectl apply -f configmap.yaml

# 使用
kubectl create configmap app-config --from-literal=KEY=value
kubectl create configmap app-config --from-file=config.json
```

#### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  # 值必须是base64编码
  # echo -n "password" | base64
  password: cGFzc3dvcmQ=
stringData:
  # 直接提供明文，Kubernetes会自动编码
  api-key: "your-api-key"
```

```bash
# 创建
kubectl create secret generic app-secret --from-literal=password=secret
kubectl create secret tls tls-cert --cert=tls.crt --key=tls.key
```

#### 在Pod中使用

```yaml
spec:
  containers:
    - name: app
      env:
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: password
      envFrom:
        - configMapRef:
            name: app-config
```

---

## kubectl 常用命令

```bash
# 资源操作
kubectl apply -f manifest.yaml      # 创建/更新资源
kubectl get pods                     # 查看资源
kubectl describe pod nginx            # 资源详情
kubectl delete -f manifest.yaml      # 删除资源
kubectl edit deployment nginx         # 编辑资源
kubectl explain pod                  # 查看资源定义

# 日志与调试
kubectl logs nginx-pod
kubectl logs --previous nginx-pod    # 上一个容器的日志
kubectl exec -it nginx-pod -- /bin/sh
kubectl port-forward nginx-pod 8080:80
kubectl top pod                      # 资源使用

# 集群信息
kubectl cluster-info                 # 集群信息
kubectl get nodes                    # 查看节点
kubectl get namespaces               # 查看命名空间
kubectl config current-context       # 当前上下文

# 上下文切换
kubectl config use-context context-name
kubectl config get-contexts

# 格式化输出
kubectl get pods -o wide            # 宽格式
kubectl get pods -o json            # JSON格式
kubectl get pods -o yaml            # YAML格式
kubectl get pods -l app=nginx       # 标签筛选
kubectl get pods --field-selector=status.phase=Running
```

---

## 网络模型

### ClusterIP（默认）

集群内部访问，仅集群内IP可访问。

### NodePort

通过节点端口访问：

```
外部 → NodeIP:NodePort → Service:Port → Pod:TargetPort
```

### LoadBalancer

云厂商提供负载均衡器。

### Ingress

外部HTTP/HTTPS入口，URL路由到Service。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /web
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

---

## 存储

### PersistentVolume (PV) 和 PersistentVolumeClaim (PVC)

#### PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce    # 单节点读写
    - ReadOnlyMany     # 多节点只读
    - ReadWriteMany    # 多节点读写
  persistentVolumeReclaimPolicy: Retain  # Retain/Delete/Recycle
  hostPath:
    path: /data/pv
```

#### PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  selector:
    matchLabels:
      type: fast
```

#### 在Pod中使用

```yaml
spec:
  containers:
    - name: app
      volumeMounts:
        - name: data
          mountPath: /app/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: pvc-example
```

### EmptyDir

临时存储，Pod删除后数据丢失。

```yaml
volumes:
  - name: cache
    emptyDir:
      medium: Memory      # 内存文件系统
      sizeLimit: 100Mi
```

### HostPath

挂载主机路径。

```yaml
volumes:
  - name: host-data
    hostPath:
      path: /host/path
      type: Directory
```

---

## 部署实战

### 部署Web应用

```yaml
# web-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web
          image: myapp:1.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 8080
```

```bash
kubectl apply -f web-app.yaml
kubectl get svc -w  # 等待External-IP
```

### 部署数据库

```yaml
# postgres.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: myapp
            - name: POSTGRES_USER
              value: user
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi
```

---

## Helm 包管理器

### Helm 概念

- **Chart**：Kubernetes应用的打包格式
- **Repository**：存储Chart的仓库
- **Release**：Chart的运行实例

### 常用命令

```bash
# 添加仓库
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 搜索Chart
helm search repo nginx

# 安装Chart
helm install my-nginx bitnami/nginx
helm install my-nginx bitnami/nginx --set service.type=loadbalancer

# 查看Release
helm list
helm status my-nginx

# 升级
helm upgrade my-nginx bitnami/nginx --set image.tag=1.25

# 回滚
helm rollback my-nginx 1

# 卸载
helm uninstall my-nginx

# 创建Chart
helm create mychart
```

---

## 参考资料

[^1]: Kubernetes Documentation. https://kubernetes.io/docs/

[^2]: Kubernetes in Action - Marko Lukša. https://kubernetes.io/docs/tutorials/

[^3]: kubectl Cheat Sheet. https://kubernetes.io/docs/reference/kubectl/quick-reference/
