---
title: Kubernetes 存储与持久化
date: 2026-04-14
description: Kubernetes PersistentVolume、PersistentVolumeClaim、StorageClass 与 StatefulSet 详解
tags:
  - kubernetes
  - storage
  - persistent-volume
  - statefulset
draft: false
permalink:
---

## 1. 概述

Kubernetes 默认容器存储是**临时性的**——Pod 删除时数据丢失。对于有状态工作负载（数据库、消息队列、文件存储），需要持久化存储方案。[^1]

### 1.1 核心概念

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes 存储架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────┐         ┌─────────────┐         ┌───────────┐  │
│   │ Persistent │ ◄─────► │ Persistent  │ ◄─────► │   Pod     │  │
│   │   Volume    │         │   Volume    │         │           │  │
│   │    (PV)     │         │   Claim     │         │           │  │
│   │             │         │   (PVC)     │         │           │  │
│   └─────────────┘         └─────────────┘         └───────────┘  │
│         ▲                       ▲                                    │
│         │                       │                                    │
│         │                       │                                    │
│   ┌─────┴─────┐           ┌─────┴─────┐                             │
│   │  Storage  │           │   CSI     │                             │
│   │  Class    │           │  Driver   │                             │
│   └───────────┘           └───────────┘                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

| 组件 | 说明 |
|------|------|
| PV | 集群级存储资源，生命周期独立于 Pod |
| PVC | Pod 对存储的请求，消耗 PV 资源 |
| StorageClass | 存储类型定义，支持动态供应 |
| CSI | Container Storage Interface，存储插件标准 |

---

## 2. PersistentVolume（PV）

PV 是集群中的一块存储，由管理员 Provisioning 或通过 StorageClass 动态创建。[^2]

### 2.1 PV 类型

```yaml
# NFS PV 示例
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany        # 支持多节点读写
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: nfs.example.com
    path: /shared/data
```

### 2.2 访问模式

| 模式 | 缩写 | 说明 |
|------|------|------|
| ReadWriteOnce | RWO | 单节点读写（大多数云盘） |
| ReadOnlyMany | ROX | 多节点只读（NFS） |
| ReadWriteMany | RWX | 多节点读写（NFS） |
| ReadWriteOncePod | RWOP | 单 Pod 独占（Kubernetes 1.22+） |

### 2.3 回收策略

| 策略 | 行为 |
|------|------|
| Retain | 删除 PVC 后 PV 保留，需要手动处理 |
| Delete | 删除 PVC 时自动删除 PV 及底层存储 |
| Recycle | 删除 PVC 后执行 `rm -rf /volume/*`，PV 可复用（已废弃） |

---

## 3. PersistentVolumeClaim（PVC）

PVC 是用户对存储的请求，类似于 Pod 请求 CPU 和内存。[^3]

### 3.1 基础 PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-claim
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: fast-ssd
```

### 3.2 Pod 使用 PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: data
          mountPath: /var/lib/app
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: app-data-claim
```

### 3.3 PVC 生命周期

```
      ┌─────────────┐
      │    Pending   │
      └──────┬──────┘
             │ 等待匹配的 PV 或动态供应
             ▼
      ┌─────────────┐
      │    Bound     │◄────────────────┐
      └──────┬──────┘                 │
             │ 正常使用                 │ 重新绑定
             ▼                          │
      ┌─────────────┐                   │
      │  Released    │──────────────────┘
      └──────┬──────┘
             │ 依赖回收策略
             ▼
      ┌─────────────┐
      │   Available  │ (Retain 策略)
      │   Deleted    │ (Delete 策略)
      └─────────────┘
```

### 3.4 PVC 扩展

```yaml
# 编辑 PVC 申请更大存储
kubectl patch pvc app-data-claim -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'

# 查看 StorageClass 是否允许扩展
kubectl get storageclass fast-ssd -o jsonpath='{.allowVolumeExpansion}'
```

---

## 4. StorageClass

StorageClass 定义存储类型和供应策略，实现动态 PV 供应。[^4]

### 4.1 基础 StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/gce-pd    # 云厂商或 CSI 供应器
parameters:
  type: pd-ssd                       # 供应器特定参数
  replication-type: regional-pd
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer   # 延迟绑定直到 Pod 调度
allowVolumeExpansion: true
```

### 4.2 volumeBindingMode

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| Immediate | 创建 PV 时立即绑定 | 预知 Pod 调度位置 |
| WaitForFirstConsumer | Pod 调度后才绑定 PV | 避免跨区域访问延迟 |

### 4.3 常见供应器

| 供应器 | 存储类型 |
|--------|----------|
| kubernetes.io/gce-pd | Google Cloud PD |
| kubernetes.io/aws-ebs | AWS EBS |
| kubernetes.io/azure-disk | Azure Disk |
| kubernetes.io/nfs | NFS |
| pd.csi.storage.googleapis.com | GKE CSI |
| ebs.csi.aws.com | AWS EBS CSI |

---

## 5. StatefulSet 与 volumeClaimTemplates

StatefulSet 是运行有状态工作负载的 Kubernetes 对象，通过 volumeClaimTemplates 为每个 Pod 提供独立的持久存储。[^5]

### 5.1 StatefulSet 特性

| 特性 | 说明 |
|------|------|
| 稳定网络标识 | Pod 名称固定（如 mysql-0, mysql-1） |
| 有序部署/伸缩 | Pod 按序创建/删除（0 → N-1） |
| 有序更新 | 按顺序更新 Pod |
| 稳定存储 | 每个 Pod 拥有独立 PVC |

### 5.2 volumeClaimTemplates

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
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
          image: postgres:16
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 50Gi
```

上述配置创建：
- `data-postgres-0` → PV0
- `data-postgres-1` → PV1
- `data-postgres-2` → PV2

### 5.3 Pod 调度与存储关系

```
Pod postgres-0 被调度到 Node-A
         │
         ▼
┌─────────────────────────────────┐
│  volumeClaimTemplate 创建        │
│  PVC: data-postgres-0           │
└─────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│  PV 绑定（通常与 Node 同区域）    │
│  假设是 Node-A 所在区域的存储     │
└─────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│  Pod 0 重调度时，               │
│  重新挂载原有 PV（数据不变）      │
└─────────────────────────────────┘
```

### 5.4 PVC Retention Policy（K8s 1.32+）

控制 StatefulSet 删除或缩容时 PVC 的行为：[^6]

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain    # StatefulSet 删除时保留 PVC
    whenScaled: Delete    # 缩容时删除多余 PVC
  # ...
```

| 策略 | 删除 StatefulSet | 缩容 |
|------|-----------------|------|
| Retain | 保留 PVC | 保留 PVC |
| Delete | 删除 PVC | 删除 PVC |

---

## 6. 动态供应 vs 静态供应

### 6.1 静态供应

管理员预先创建 PV，用户创建 PVC 绑定到指定 PV：

```yaml
# 管理员创建 PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: static-pv
spec:
  capacity:
    storage: 100Gi
  storageClassName: manual
  hostPath:
    path: /mnt/data
---
# 用户创建 PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  storageClassName: manual
  resources:
    requests:
      storage: 10Gi
```

### 6.2 动态供应

用户只需声明需求，StorageClass 自动创建 PV：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-claim
spec:
  storageClassName: fast-ssd   # 无需预先创建 PV
  resources:
    requests:
      storage: 20Gi
```

---

## 7. 常见问题排查

### 7.1 PVC 一直处于 Pending

```bash
# 检查 PVC 详情
kubectl describe pvc <pvc-name>

# 常见原因：
# 1. 没有匹配的 PV（增加 PV 或确认 StorageClass 配置）
# 2. StorageClass 不存在
# 3. 延迟绑定模式下，Pod 未调度
```

### 7.2 Pod 无法挂载卷

```bash
# 检查 Pod 事件
kubectl describe pod <pod-name>

# 检查 PV 状态
kubectl describe pv <pv-name>

# 验证存储类
kubectl get storageclass
```

### 7.3 存储容量不足

```bash
# 监控 PV 使用率
kubectl get pv -o=custom-columns=NAME:.metadata.name,CAPACITY:.spec.capacity.storage,CLAIM:.spec.claimRef.name,STATUS:.status.phase

# 扩展 PVC（如果 allowVolumeExpansion: true）
kubectl patch pvc <pvc-name> -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'
```

---

## 8. 最佳实践

### 8.1 存储选型

| 应用类型 | 推荐存储 | 访问模式 |
|----------|----------|----------|
| PostgreSQL/MySQL | 云盘（AWS gp3, GKE pd-ssd） | RWO |
| NFS 文件存储 | NFS Server | RWX |
| Kafka/Elasticsearch | 本地 SSD | RWO |
| S3 兼容存储 | 对象存储 CSI | N/A |

### 8.2 安全建议

- 使用加密存储类（storageClassName 包含 "encrypted"）
- 配置 fsGroup 确保正确的权限
- 敏感数据使用 Secret 或外部密钥管理

### 8.3 监控建议

- 监控 PVC 使用率，设置容量告警
- 监控 PV/PVC 事件，及时发现供应失败
- 定期审计 Orphaned PV（无 PVC 引用）

---

## 参考资料

[^1]: Kubernetes 存储官方文档: https://kubernetes.io/docs/concepts/storage/persistent-volumes/
[^2]: Kubernetes Storage Explained: PVs, PVCs, StatefulSets: https://www.blockchain-council.org/Kubernetes/kubernetes-storage-explained-persistent-volumes-storage-classes-statefulsets/
[^3]: Understanding Kubernetes Persistent Volumes and Claims: https://oneuptime.com/blog/post/2026-02-20-kubernetes-persistent-volumes-claims/view
[^4]: GKE 有状态应用部署: https://cloud.google.com/kubernetes-engine/docs/how-to/stateful-apps
[^5]: StatefulSets | Kubernetes: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
[^6]: Configure a Pod to Use a PersistentVolume for Storage: https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/
