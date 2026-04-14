---
title: Kubernetes进阶概念
date: 2026-04-11
description: Kubernetes进阶教程：高级调度、存储进阶、安全策略、运维实践与监控体系
tags:
  - kubernetes
  - devops
  - cloud-native
draft: false
permalink:
---

## Kubernetes进阶概念

本文档涵盖Kubernetes进阶主题，包括高级调度、存储进阶、安全策略和运维实践。[^1]

---

## Pod生命周期与重启策略

### Pod相位

Pod的`status.phase`表示其生命周期阶段：

| Phase | 说明 |
|-------|------|
| **Pending** | Pod已被Kubernetes系统接受，正在等待调度 |
| **Running** | Pod已绑定到节点，容器正在运行 |
| **Succeeded** | 所有容器正常终止，不会重启 |
| **Failed** | 容器异常终止，restartPolicy为Never或Fail |
| **Unknown** | 无法获取Pod状态 |

### 重启策略

```yaml
spec:
  restartPolicy: Always   # 默认值，容器退出后总是重启
  # restartPolicy: OnFailure   # 容器非正常退出时重启
  # restartPolicy: Never       # 永不重启
```

### Init Container

Init容器在主容器启动前执行，常用于依赖准备：

```yaml
spec:
  initContainers:
    - name: init-db
      image: busybox:1.36
      command: ['sh', '-c', 'until nslookup mysql; do sleep 2; done;']
    - name: migrate
      image: myapp:migrate
      env:
        - name: DATABASE_HOST
          value: "mysql"
  containers:
    - name: app
      image: myapp:1.0
```

### 容器钩子

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      lifecycle:
        postStart:
          exec:
            command: ["/bin/sh", "-c", "echo 'Container started' > /tmp/started"]
        preStop:
          exec:
            command: ["/bin/sh", "-c", "nginx -s quit"]
```

---

## 资源限制

### Resource Requirements

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      resources:
        requests:    # 调度时所需的最小资源
          memory: "128Mi"
          cpu: "100m"
        limits:       # 资源上限
          memory: "256Mi"
          cpu: "500m"
```

### LimitRange

限制命名空间内容器资源使用：

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
    - type: Container
      default:
        memory: "256Mi"
        cpu: "200m"
      defaultRequest:
        memory: "128Mi"
        cpu: "100m"
      max:
        memory: "1Gi"
        cpu: "1000m"
      min:
        memory: "64Mi"
        cpu: "50m"
```

### ResourceQuota

限制命名空间总资源：

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "10"
```

---

## 高级调度

### 亲和性与反亲和性

#### 节点亲和性（Node Affinity）

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          preference:
            matchExpressions:
              - key: workload-type
                operator: In
                values:
                  - production
```

#### Pod亲和性与反亲和性

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - frontend
          topologyKey: kubernetes.io/hostname
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values:
                    - cache
            topologyKey: kubernetes.io/hostname
```

### 污点与容忍

#### 节点污点（Taints）

```bash
# 添加污点
kubectl taint nodes node1 dedicated=postgres:NoSchedule
kubectl taint nodes node1 gpu=true:NoExecute
kubectl taint nodes node1 maintenance=true:PreferNoSchedule

# 移除污点
kubectl taint nodes node1 dedicated- postgres:NoSchedule
```

#### Pod容忍（Tolerations）

```yaml
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "postgres"
      effect: "NoSchedule"
    - key: "gpu"
      operator: "Exists"
      effect: "NoExecute"
      tolerationSeconds: 300
    - key: "maintenance"
      operator: "Exists"
      effect: "PreferNoSchedule"
```

### 优先级与抢占

#### 优先级类

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
description: "高优先级任务"
```

#### 使用优先级

```yaml
spec:
  priorityClassName: high-priority
  containers:
    - name: critical-app
      image: myapp:1.0
```

### 拓扑分布约束

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: web
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app: web
```

---

## 存储进阶

### StorageClass

#### NFS StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: nfs.io/provisioner
parameters:
  archiveOnDelete: "false"
  pathPattern: "/data/pv-${pvc.metadata.name}"
  server: nfs-server.example.com
  mountOptions:
    - vers=4.1
```

#### 云厂商示例

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: regional-pd
  fstype: ext4
```

### 动态存储供应

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 50Gi
```

### 持久卷扩容

#### 启用扩容

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable
provisioner: kubernetes.io/gce-pd
allowVolumeExpansion: true
```

#### 在线扩容

```bash
# 修改PVC大小
kubectl patch pvc dynamic-pvc -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'

# 查看扩容状态
kubectl describe pvc dynamic-pvc
```

### 持久卷的Retain策略

| 策略 | 说明 |
|------|------|
| **Retain** | 删除PVC后PV保留，需手动处理 |
| **Delete** | 删除PVC时自动删除PV和存储 |
| **Recycle** | 删除PVC后自动执行清理并重新供应 |

---

## 安全进阶

### NetworkPolicy

#### 限制入站流量

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
      tier: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - protocol: TCP
          port: 9090
```

#### 限制出站流量

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-egress-policy
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: TCP
          port: 53
```

### Security Context

#### Pod级别

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    supplementalGroups: [1000, 2000]
  containers:
    - name: app
      image: myapp:1.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE
```

#### Container级别

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
  seLinuxOptions:
    level: "s0:c123,c456"
```

### RBAC角色权限控制

#### Role和ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
    resourceNames: ["app-config"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["nodes/stats"]
    verbs: ["get"]
```

#### RoleBinding和ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: default
    namespace: production
  - kind: User
    name: developer
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
  - kind: Group
    name: system:nodes
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

### Pod Security Policy

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: MustRunAsNonRoot
  fsGroup:
    rule: RunAsAny
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
```

---

## 运维进阶

### Helm包管理器

#### Helm模板结构

```
mychart/
├── Chart.yaml
├── values.yaml
├── charts/
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl
│   └── NOTES.txt
└── templates/tests/
    └── test-connection.yaml
```

#### values.yaml示例

```yaml
replicaCount: 3

image:
  repository: myapp
  tag: "1.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    memory: 256Mi
    cpu: 500m
  requests:
    memory: 128Mi
    cpu: 100m

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

#### 模板函数

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          {{- with .Values.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
```

#### 常用Helm命令

```bash
# 仓库管理
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus https://prometheus-community.github.io/helm-charts
helm repo update

# 安装
helm install my-release bitnami/nginx --namespace production --create-namespace
helm install my-release ./mychart --values values.prod.yaml

# 升级与回滚
helm upgrade my-release bitnami/nginx --set image.tag=1.16
helm rollback my-release 1

# 模板渲染
helm template my-release ./mychart
helm diff upgrade my-release ./mychart

# 依赖管理
helm dependency build ./mychart
helm dependency update ./mychart
```

### Operator模式

#### Operator结构

```
myoperator/
├── config/
│   ├── crd/
│   │   └── bases/
│   │       └── myapp.mycompany.io_apps.yaml
│   ├── rbac/
│   │   └── role.yaml
│   └── manager/
│       └── manager.yaml
├── api/
│   └── v1/
│       └── app_types.go
├── controllers/
│   └── app_controller.go
└── main.go
```

#### 自定义资源定义

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: apps.myapp.example.com
spec:
  group: myapp.example.com
  names:
    kind: App
    listKind: AppList
    plural: apps
    singular: app
    shortNames:
      - app
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas:
                  type: integer
                  minimum: 1
                image:
                  type: string
                port:
                  type: integer
```

### 自动扩缩容

#### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 500Mi
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
```

#### Vertical Pod Autoscaler (VPA)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed:
          memory: 128Mi
          cpu: 50m
        maxAllowed:
          memory: 1Gi
          cpu: 1000m
        controlledResources:
          - memory
          - cpu
```

#### Cluster Autoscaler

```bash
# 部署Cluster Autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --set cloudProvider=aws \
  --set autoDiscovery.clusterName=my-cluster \
  --set awsRegion=us-east-1
```

---

## 监控与日志

### Prometheus + Grafana

#### Prometheus Operator

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      team: frontend
  resources:
    requests:
      memory: 400Mi
  retention: 15d
  retentionSize: 10GB
```

#### ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: web-app-monitor
  labels:
    team: frontend
spec:
  selector:
    matchLabels:
      app: web-app
  endpoints:
    - port: metrics
      path: /metrics
      interval: 15s
  namespaceSelector:
    matchNames:
      - production
```

#### Grafana Dashboard

```yaml
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDashboard
metadata:
  name: web-app-dashboard
spec:
  json: |
    {
      "dashboard": {
        "title": "Web App Metrics",
        "panels": [
          {
            "title": "Request Rate",
            "type": "graph",
            "targets": [
              {
                "expr": "rate(http_requests_total[5m])",
                "legendFormat": "{{method}} {{path}}"
              }
            ]
          }
        ]
      }
    }
```

### ELK日志收集

#### Fluent Bit配置

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Log_Level     info
        Daemon        off

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            docker
        Tag               kube.*
        Refresh_Interval  5

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token

    [OUTPUT]
        Name            es
        Match           kube.*
        Host            elasticsearch.logging.svc
        Port            9200
        HTTP_User       elastic
        HTTP_Passwd     changeme
        Logstash_Format On
        Logstash_Prefix kubernetes
        Retry_Limit     False
```

#### Fluentd部署

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccount: fluentd
      containers:
        - name: fluentd
          image: fluent/fluentd:v1.16
          env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: "elasticsearch.logging.svc"
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"
          resources:
            limits:
              memory: 512Mi
            requests:
              cpu: 100m
              memory: 200Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
```

---

## 参考资料

[^1]: Kubernetes Documentation. https://kubernetes.io/docs/

[^2]: Kubernetes in Action - Marko Lukša. https://kubernetes.io/docs/tutorials/

[^3]: Helm Documentation. https://helm.sh/docs/

[^4]: Prometheus Operator. https://prometheus-operator.dev/
