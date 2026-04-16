---
title: Kubernetes Operator 开发
date: 2026-04-14
description: Kubernetes Operator 模式详解：CRD 设计、Reconciliation 循环、最佳实践
tags:
  - kubernetes
  - operator
  - crd
  - controller
draft: false
permalink:
---

## 1. Operator 模式概述

Operator 是 Kubernetes 中的一种自定义控制器模式，它将运维知识编码为软件，通过自定义资源（Custom Resource）来管理复杂的、有状态的应用生命周期。

### 1.1 为什么需要 Operator

传统的 Kubernetes 资源（如 Deployment、StatefulSet）只能描述**做什么**（what），无法表达**如何做**（how）。例如：

- PostgreSQL 集群需要自动故障切换
- etcd 需要周期性备份
- 自定义应用需要特定的初始化流程

这些问题无法通过声明式配置解决，需要**命令式运维知识**。

### 1.2 Operator 的核心思想

```
运维知识 → 代码 → 控制器 → 自定义资源
```

Operator 将运维手册（runbook）转化为代码，使运维操作自动化、可重复、可靠。[^1]

---

## 2. 核心概念

### 2.1 CRD（Custom Resource Definition）

CRD 是扩展 Kubernetes API 的机制，允许用户定义新的资源类型：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  names:
    kind: Database
    plural: databases
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
```

### 2.2 自定义资源（Custom Resource）

基于 CRD 创建的具体资源实例：

```yaml
apiVersion: example.com/v1
kind: Database
metadata:
  name: orders-db
spec:
  engine: postgresql
  version: "16"
  replicas: 3
  storage:
    size: 100Gi
    storageClass: fast-ssd
```

### 2.3 控制器（Controller）

控制器是一个持续运行的循环，比较**期望状态**与**实际状态**，并采取行动消除差异：

```
┌─────────────────────────────────────────────────────────┐
│                    Reconciliation Loop                   │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Watch CR ──▶ Fetch CR ──▶ Compare ──▶ Reconcile         │
│     ↑                                      │             │
│     └──────────────────────────────────────┘             │
│                     (repeat)                              │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Reconciliation 循环

Reconciliation 循环是 Operator 的核心，是理解 Operator 设计的关键。[^2]

### 3.1 关键原则

**幂等性（Idempotent）**：运行 reconcile 100 次与运行 1 次应该产生相同结果。使用 `CreateOrUpdate` 语义而非单纯的 `Create`。

**Level-triggered（电平触发）**：不关注"发生了什么变化"，而关注"当前状态是什么"。这使得控制器对事件丢失和重复具有鲁棒性。

**返回 RequeueAfter**：如果集群处于过渡状态（如节点正在启动），返回 `RequeueAfter` 而非立即重试。

### 3.2 Reconciliation 函数结构

```go
func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. 获取自定义资源
    database := &examplev1.Database{}
    if err := r.Get(ctx, req.NamespacedName, database); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. 处理删除
    if !database.DeletionTimestamp.IsZero() {
        // 执行清理逻辑
        return r.handleDeletion(ctx, database)
    }

    // 3. 添加 Finalizer
    if !controllerutil.ContainsFinalizer(database, finalizerName) {
        controllerutil.AddFinalizer(database, finalizerName)
        return ctrl.Result{}, r.Update(ctx, database)
    }

    // 4. 验证资源有效性
    if err := r.validateSpec(database); err != nil {
        return ctrl.Result{}, err
    }

    // 5. 执行 Reconcile 逻辑
    return r.reconcileDatabase(ctx, database)
}
```

### 3.3 CreateOrUpdate 模式

```go
func (r *DatabaseReconciler) reconcileStatefulSet(ctx context.Context, db *examplev1.Database) error {
    sts := &appsv1.StatefulSet{}
    err := r.Get(ctx, types.NamespacedName{Name: db.Name, Namespace: db.Namespace}, sts)
    
    if errors.IsNotFound(err) {
        // 不存在则创建
        sts = r.createStatefulSet(db)
        return r.Create(ctx, sts)
    }
    
    if err != nil {
        return err
    }
    
    // 已存在则更新（如果需要）
    if !r.stsMatchesSpec(sts, db) {
        sts = r.updateStatefulSet(sts, db)
        return r.Update(ctx, sts)
    }
    
    return nil
}
```

---

## 4. CRD 设计原则

### 4.1 Spec 与 Status 分离

| 字段 | 作用 | 所有者 |
|------|------|--------|
| `spec` | 用户期望状态（不可被控制器修改） | 用户 |
| `status` | 实际观察到的状态（仅由控制器写入） | 控制器 |

```yaml
apiVersion: example.com/v1
kind: Database
metadata:
  name: orders-db
spec:
  replicas: 3          # 用户设置
  storage: 100Gi       # 用户设置
status:
  phase: Running       # 控制器更新
  replicas: 3         # 控制器更新
  conditions:
    - type: Ready
      status: "True"
```

### 4.2 版本管理

从 `v1alpha1` 开始，仅在语义稳定后升级到 `v1`。使用转换 Webhook 处理版本间差异：

```yaml
versions:
  - name: v1alpha1
    served: true
    storage: false
  - name: v1beta1
    served: true
    storage: false
  - name: v1
    served: true
    storage: true
```

### 4.3 CEL 验证

使用 Common Expression Language 在 CRD 级别进行验证：

```yaml
spec:
  validation:
    openAPIV3Schema:
      properties:
        spec:
          required: ["replicas", "storage"]
          properties:
            replicas:
              type: integer
              minimum: 1
              maximum: 100
            storage:
              type: object
              required: ["size"]
              properties:
                size:
                  type: string
                  pattern: '^[0-9]+Gi$'
```

---

## 5. Finalizer

Finalizer 是一种机制，用于在删除自定义资源前执行清理逻辑（如清理外部资源）。[^3]

### 5.1 为什么需要 Finalizer

当删除一个 CR 时，Kubernetes 只标记删除时间戳，不立即删除。控制器检测到 `DeletionTimestamp` 非零后执行清理，完成后移除 finalizer，资源才被真正删除。

### 5.2 实现示例

```go
const finalizerName = "databases.example.com/finalizer"

func (r *DatabaseReconciler) handleDeletion(ctx context.Context, db *examplev1.Database) (ctrl.Result, error) {
    // 检查是否需要清理外部资源
    if controllerutil.ContainsFinalizer(db, finalizerName) {
        // 执行清理逻辑（如删除云盘快照、清理 DNS 记录）
        if err := r.cleanupExternalResources(ctx, db); err != nil {
            return ctrl.Result{}, err
        }
        
        // 移除 finalizer
        controllerutil.RemoveFinalizer(db, finalizerName)
        return ctrl.Result{}, r.Update(ctx, db)
    }
    
    return ctrl.Result{}, nil
}
```

### 5.3 Owner Reference vs Finalizer

| 机制 | 用途 | 适用范围 |
|------|------|----------|
| Owner Reference | Kubernetes 资源级联删除 | 同集群资源 |
| Finalizer | 外部资源清理 | 集群外资源（云存储、DNS） |

---

## 6. 测试策略

### 6.1 单元测试（envtest）

使用 controller-runtime 提供的 envtest 进行单元测试：

```go
import (
    "sigs.k8s.io/controller-runtime/pkg/controller"
    "sigs.k8s.io/controller-runtime/pkg/envtest"
    "sigs.k8s.io/controller-runtime/pkg/reconcile"
)

var cfg *rest.Config
var k8sClient client.Client
var testEnv *envtest.Environment

func TestDatabaseReconciler(t *testing.T) {
    // 创建测试 CR
    db := &examplev1.Database{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "test-db",
            Namespace: "default",
        },
        Spec: examplev1.DatabaseSpec{
            Replicas: 3,
        },
    }
    
    // 创建请求
    req := reconcile.Request{
        NamespacedName: types.NamespacedName{
            Name:      db.Name,
            Namespace: db.Namespace,
        },
    }
    
    // 执行 Reconcile
    _, err := r.Reconcile(ctx, req)
    if err != nil {
        t.Fatalf("reconcile error: %v", err)
    }
    
    // 验证结果
    // ...
}
```

### 6.2 E2E 测试（kind）

使用 kind（Kubernetes in Docker）进行完整的生命周期测试：

```bash
# 创建 kind 集群
kind create cluster --name operator-test

# 加载 operator 镜像
kind load docker-image my-operator:latest --name operator-test

# 部署 CRD 和 Operator
kubectl apply -f config/crd/bases/example.com_databases.yaml
kubectl apply -f deploy/operator.yaml

# 运行 E2E 测试
go test -v ./e2e/...
```

---

## 7. 何时使用 / 不使用 Operator

### 7.1 适合使用 Operator 的场景

- 运行有状态应用（数据库、消息队列）且规模较大
- 运维手册需要反复执行且容易出错
- 需要应用级别的故障恢复（如数据库主从切换）
- 需要自动化升级、备份操作

### 7.2 不适合使用 Operator 的场景

- 无状态应用：Deployment + Helm 已经足够
- 一次性操作：手动执行即可
- 团队工程能力有限：Operator 需要 3-6 个月工程投入

### 7.3 Operator 成熟度模型

| 级别 | 描述 |
|------|------|
| Level 1 | 基础安装 |
| Level 2 | 无缝升级 |
| Level 3 | 完整生命周期管理（备份、恢复、故障切换） |
| Level 4 | 深度洞察（指标、日志） |
| Level 5 | 自动驾驶（基于负载自动优化） |

---

## 8. 主流 Operator 框架

| 框架 | 特点 | 适用场景 |
|------|------|----------|
| Kubebuilder | 官方维护，与 Kubernetes API 集成好 | 新项目首选 |
| Operator SDK | 功能丰富，支持 Helm/Ansible | 现有项目集成 |
| MetaController | 轻量级 | 简单控制器 |

---

## 参考资料

[^1]: Kubernetes Operator Pattern: Building Custom Controllers for Stateful Applications: https://mdsanwarhossain.me/blog-kubernetes-operator-pattern.html
[^2]: Building Your First Kubernetes Operator with Kubebuilder: https://timderzhavets.com/blog/building-your-first-kubernetes-operator-with/
[^3]: Kubernetes Operators Best Practices: https://redhat.com/en/blog/kubernetes-operators-best-practices
[^4]: Operator Sample Go Documentation (IBM): https://ibm.github.io/operator-sample-go-documentation/
[^5]: Kubernetes Operators in 2025: Best Practices, Patterns, and Real-World Insights: https://outerbyte.com/kubernetes-operators-2025-guide/
