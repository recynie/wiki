---
title: Terraform 基础设施即代码
date: 2026-04-13
description: Terraform 基础知识：HCL语言、Provider、Resource、状态管理、模块化设计
tags:
  - terraform
  - iac
  - devops
  - hashicorp
  - infrastructure
draft: false
permalink:
---

## Terraform 概述

Terraform 是由 HashiCorp 开发的开源**基础设施即代码**（Infrastructure as Code，IaC）工具，允许用户通过声明式配置文件来定义和管理云资源。[^1]

### 什么是 IaC

传统方式下，开发者通过图形界面或命令行手动管理服务器、网络、存储等基础设施。IaC 核心理念是：**用代码管理基础设施，而非图形界面**。

```bash
# 传统方式：手动点击图形界面或一条条执行命令
# 点击创建EC2实例 → 选择配置 → 点击确认

# IaC方式：用代码声明
terraform apply  # 一条命令完成所有配置
```

### Terraform 的优势

| 特性 | 说明 |
|------|------|
| **声明式配置** | 描述"要什么"而非"如何做"，Terraform 自动规划执行路径 |
| **状态管理** | 跟踪真实基础设施状态，支持回滚和审计 |
| **Provider 生态** | 支持 AWS、Azure、GCP、阿里云等众多云平台 |
| **执行计划** | `plan` 命令预览更改，防止误操作 |
| **幂等性** | 多次执行结果一致，不会重复创建资源 |
| **模块化** | 可复用的配置模块，提升开发效率 |

---

## 核心概念

### HCL (HashiCorp Configuration Language)

HCL 是 Terraform 使用的声明式配置语言，设计目标是**人类可读且机器可解析**。

```hcl
# HCL 示例：定义一个 AWS EC2 实例
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name        = "web-server"
    Environment = "production"
  }
}
```

HCL 特点：
- 块结构：`resource "type" "name" { ... }`
- 键值对：`key = "value"`
- 列表：`["item1", "item2"]`
- 映射：`{ key1 = "value1", key2 = "value2" }`

### Provider

Provider 是 Terraform 与云平台或服务 API 交互的插件。每个 Provider 提供一组 `resource` 和 `data_source`。

```hcl
# 指定 Provider 版本要求
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# 配置 Provider
provider "aws" {
  region = "ap-northeast-1"
  
  # 可选：使用环境变量或其他认证方式
  # access_key = var.access_key
  # secret_key = var.secret_key
}
```

常用 Provider：

| Provider | 说明 |
|----------|------|
| `hashicorp/aws` | AWS 云平台 |
| `hashicorp/azurerm` | Azure 云平台 |
| `hashicorp/google` | Google Cloud Platform |
| `alicloud/alicloud` | 阿里云 |
| `hashicorp/kubernetes` | Kubernetes 集群 |
| `hashicorp/docker` | Docker 容器 |

### Resource

Resource 是基础设施的单个组件，每个 Resource 属于某个 Provider。

```hcl
# AWS S3 存储桶
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket-2026"

  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# AWS VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "main-vpc"
  }
}

# AWS 安全组
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

Resource 依赖：`Terraform 自动分析资源间的依赖关系，按正确顺序创建资源。`

### Data Source

Data Source 用于查询现有基础设施，获取只读信息。

```hcl
# 查询现有 VPC
data "aws_vpc" "existing" {
  id = "vpc-0123456789abcdef0"
}

# 查询可用区
data "aws_availability_zones" "available" {
  state = "available"
}

# 使用查询结果
resource "aws_subnet" "example" {
  vpc_id     = data.aws_vpc.existing.id
  cidr_block = "10.0.1.0/24"
  
  # 引用可用区
  availability_zone = data.aws_availability_zones.available.names[0]
}
```

### Variable

Variable 用于参数化配置，提高配置的复用性和灵活性。

```hcl
# variables.tf
variable "environment" {
  description = "部署环境"
  type        = string
  default     = "dev"
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment 必须是 dev、staging 或 prod。"
  }
}

variable "instance_type" {
  description = "EC2 实例类型"
  type        = string
  default     = "t3.micro"
}

variable "tags" {
  description = "资源标签"
  type        = map(string)
  default     = {}
}

# 使用变量
resource "aws_instance" "server" {
  instance_type = var.instance_type
  tags         = merge(var.tags, { Environment = var.environment })
}
```

变量类型：

| 类型 | 说明 | 示例 |
|------|------|------|
| `string` | 字符串 | `"hello"` |
| `number` | 数字 | `42` |
| `bool` | 布尔值 | `true` / `false` |
| `list()` | 列表 | `["a", "b", "c"]` |
| `map()` | 映射 | `{ key = "value" }` |
| `object()` | 对象 | `{ name = "test", id = 1 }` |
| `set()` | 集合 | 无序唯一值 |

### Output

Output 用于输出创建的资源信息，便于其他配置引用或查看。

```hcl
# outputs.tf
output "instance_id" {
  description = "EC2 实例 ID"
  value       = aws_instance.server.id
}

output "instance_ip" {
  description = "EC2 实例公网 IP"
  value       = aws_instance.server.public_ip
}

output "vpc_info" {
  description = "VPC 信息"
  value = {
    id       = aws_vpc.main.id
    cidr     = aws_vpc.main.cidr_block
    subnets  = aws_subnet.public[*].id
  }
}
```

### Module

Module 是可复用的配置包，封装一组相关资源。

```
modules/
└── networking/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

```hcl
# modules/networking/main.tf
variable "vpc_cidr" {
  description = "VPC CIDR 块"
  type        = string
}

variable "environment" {
  description = "环境名称"
  type        = string
}

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  
  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

resource "aws_subnet" "public" {
  count                   = 3
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${var.environment}-public-subnet-${count.index + 1}"
  }
}

# modules/networking/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "subnet_ids" {
  value = aws_subnet.public[*].id
}
```

使用模块：

```hcl
# main.tf
module "networking" {
  source = "./modules/networking"
  
  vpc_cidr     = "10.0.0.0/16"
  environment  = "production"
}

# 引用模块输出
resource "aws_instance" "server" {
  subnet_id = module.networking.subnet_ids[0]
  # ...
}
```

---

## Terraform 工作流

Terraform 标准工作流包含以下步骤：

### terraform init

初始化工作目录，下载 Provider 插件，分析模块。

```bash
terraform init

# 输出示例
# Initializing the backend...
# Initializing provider plugins...
# - Finding hashicorp/aws versions matching "~> 5.0"...
# - Installing hashicorp/aws v5.31.0...
# Terraform has been successfully initialized!
```

### terraform validate

验证配置文件语法和内部一致性。

```bash
terraform validate

# 成功输出
# Success! The configuration is valid.

# 失败输出
# Error: Missing required argument
#   on main.tf line 10, in resource "aws_instance" "web":
#   10:   ami = var.ami_id
# An argument named "ami_id" is not expected here.
```

### terraform plan

预览将要做的更改，不实际执行。

```bash
terraform plan

# 输出示例
# Plan: 3 to add, 0 to change, 0 to destroy.
#
#   + aws_vpc.main
#       id:                               <computed>
#       cidr_block:                      "10.0.0.0/16"
#
#   + aws_subnet.public[0]
#       id:                               <computed>
#       vpc_id:                          "${aws_vpc.main.id}"
```

`-out` 选项将计划保存到文件：

```bash
terraform plan -out=plan.tfplan
terraform apply plan.tfplan
```

### terraform apply

执行更改，创建、更新或销毁资源。

```bash
terraform apply

# 交互式确认
# Do you want to perform these actions?
#   Terraform will perform the actions described above.
#   Enter 'yes' to continue.

# 自动确认
terraform apply -auto-approve
```

### terraform destroy

销毁创建的资源。

```bash
# 销毁前预览
terraform plan -destroy

# 确认销毁
terraform destroy

# 自动确认
terraform destroy -auto-approve

# 销毁指定资源
terraform destroy -target=aws_instance.server
```

### 其他常用命令

```bash
# 格式化配置文件
terraform fmt

# 查看当前状态
terraform show

# 列出所有资源
terraform state list

# 手动查看状态文件
terraform state pull > state.json

# 移动资源（重构时使用）
terraform state mv aws_instance.old aws_instance.new

# 删除状态中的资源（不再管理）
terraform state rm aws_instance.unmanaged
```

---

## 状态管理 (State)

### State 的作用

Terraform 使用 **state 文件**跟踪真实基础设施状态。

```
# .terraform/terraform.tfstate
{
  "version": 4,
  "terraform_version": "1.8.0",
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "instances": [...]
    }
  ]
}
```

State 的核心作用：

| 作用 | 说明 |
|------|------|
| **映射** | 将配置文件中的 Resource 映射到真实基础设施 |
| **跟踪** | 记录资源当前状态，检测变更 |
| **依赖** | 分析 Resource 间依赖关系 |
| **性能** | 大规模基础设施下避免 API 调用 |

### Local State vs Remote State

**Local State**：状态保存在本地文件。

```hcl
terraform {
  backend "local" {
    path = "terraform.tfstate"
  }
}
```

**Remote State**：状态保存在远程存储，支持团队协作。

```hcl
# S3 + DynamoDB 后端
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "ap-northeast-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

常用 Remote Backend：

| Backend | 说明 | 特性 |
|---------|------|------|
| S3 + DynamoDB | AWS | 支持版本控制、加密、锁 |
| GCS | Google Cloud | 支持版本控制、加密 |
| Azure Blob | Azure | 支持加密 |
| Terraform Cloud | HashiCorp | SaaS，支持远程执行 |
| Consul | HashiCorp | 分布式一致性 |

### State Locking

状态锁防止并发执行导致状态损坏。

```
# 并发 apply 场景
terraform apply  # 终端 A
terraform apply  # 终端 B 同时执行

# 无锁：状态文件损坏，资源冲突
# 有锁：终端 B 等待或报错
```

DynamoDB 表配置：

```json
{
  "TableName": "terraform-locks",
  "KeySchema": [{"AttributeName": "LockID", "KeyType": "HASH"}],
  "AttributeDefinitions": [{"AttributeName": "LockID", "AttributeType": "S"}],
  "BillingMode": "PAY_PER_REQUEST"
}
```

### State 注意事项

> **警告**：永远不要手动修改 state 文件。手动修改会导致状态与真实基础设施不一致，引发难以排查的问题。

```bash
# 错误做法
vim terraform.tfstate  # 不要这样做！

# 正确做法
terraform state mv aws_instance.old aws_instance.new
terraform state rm aws_instance.unmanaged
terraform import aws_instance.existing i-1234567890abcdef0
```

---

## 模块设计

### 模块结构

标准模块结构：

```
modules/
└── <module_name>/
    ├── main.tf          # 资源定义
    ├── variables.tf     # 输入变量
    ├── outputs.tf       # 输出值
    ├── versions.tf      # 版本约束（可选）
    └── README.md        # 文档（可选）
```

### 常用模块示例

**VPC 模块**：

```hcl
# modules/vpc/main.tf
variable "environment" {}
variable "vpc_cidr" {}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

# modules/vpc/variables.tf
variable "environment" {
  type = string
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

# modules/vpc/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "vpc_cidr" {
  value = aws_vpc.main.cidr_block
}
```

**S3 存储桶模块**：

```hcl
# modules/s3_bucket/main.tf
variable "bucket_name" {}
variable "versioning_enabled" {
  default = true
}

resource "aws_s3_bucket" "main" {
  bucket = var.bucket_name
}

resource "aws_s3_bucket_versioning" "main" {
  bucket = aws_s3_bucket.main.id
  
  versioning_configuration {
    status = var.versioning_enabled ? "Enabled" : "Disabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "main" {
  bucket = aws_s3_bucket.main.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# modules/s3_bucket/outputs.tf
output "bucket_id" {
  value = aws_s3_bucket.main.id
}

output "bucket_arn" {
  value = aws_s3_bucket.main.arn
}
```

### 模块设计原则

1. **单一职责**：每个模块专注完成一件事
2. **最小暴露**：只暴露必要的输入变量和输出
3. **合理默认值**：提供合理的默认配置
4. **版本约束**：指定兼容的 Terraform 和 Provider 版本
5. **文档完善**：README 说明模块用途和使用方法

---

## 项目结构最佳实践

### 常见项目结构

```
infra/
├── terraform.tfvars        # 变量值文件
├── main.tf                 # 主配置
├── variables.tf            # 变量定义
├── outputs.tf              # 输出定义
├── versions.tf             # 版本约束
├── modules/                # 本地模块
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── ecs/
│   │   └── ...
│   └── rds/
│       └── ...
├── env/
│   ├── dev/
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   └── prod/
│       ├── terraform.tfvars
│       └── backend.tf
└── README.md
```

### 环境分离策略

**方式一：目录分离**

```
environments/
├── dev/
│   ├── main.tf
│   ├── variables.tf
│   └── terraform.tfvars
├── staging/
│   └── ...
└── prod/
    └── ...
```

**方式二：工作区（Workspace）**

```bash
# 创建工作区
terraform workspace new prod
terraform workspace new staging

# 切换工作区
terraform workspace select prod

# 查看当前工作区
terraform workspace show
```

### terraform.tfvars 文件

```hcl
# terraform.tfvars
environment     = "production"
instance_type   = "t3.medium"
desired_capacity = 2
max_size         = 4
min_size         = 1

tags = {
  Project     = "myapp"
  ManagedBy   = "terraform"
  Environment = "production"
}
```

---

## 基础配置示例

### 创建 EC2 实例

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-1"
}

variable "instance_type" {
  default = "t3.micro"
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2
  instance_type = var.instance_type

  tags = {
    Name = "web-server"
  }
}

output "instance_id" {
  value = aws_instance.web.id
}

output "public_ip" {
  value = aws_instance.web.public_ip
}
```

### 配置 S3 存储桶

```hcl
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket-${var.environment}"

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_s3_bucket_public_access_block" "data" {
  bucket = aws_s3_bucket.data.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_versioning" "data" {
  bucket = aws_s3_bucket.data.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

### 使用模块组合

```hcl
# 完整示例：创建 VPC + EC2
module "vpc" {
  source = "./modules/vpc"

  environment = var.environment
  vpc_cidr    = "10.0.0.0/16"
}

module "ec2" {
  source = "./modules/ec2"

  environment  = var.environment
  subnet_id    = module.vpc.public_subnet_ids[0]
  instance_type = "t3.micro"
}

# 依赖关系自动处理
output "ec2_ip" {
  value = module.ec2.public_ip
}
```

---

## 远程后端配置

### S3 + DynamoDB 配置

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-prod"
    key            = "terraform.tfstate"
    region         = "ap-northeast-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
    
    # 启用状态文件版本管理
    # 需要在 S3 桶上启用版本控制
  }
}
```

创建 DynamoDB 锁表：

```bash
aws dynamodb create-table \
  --table-name terraform-state-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

S3 桶配置：

```bash
# 创建启用版本控制的 S3 桶
aws s3 mb s3://my-terraform-state-prod
aws s3api put-bucket-versioning \
  --bucket my-terraform-state-prod \
  --versioning-configuration Status=Enabled

# 启用服务器端加密
aws s3api put-bucket-encryption \
  --bucket my-terraform-state-prod \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'
```

### Terraform Cloud 配置

```hcl
# backend.tf
terraform {
  backend "remote" {
    organization = "my-org"

    workspaces {
      name = "my-project-prod"
    }
  }
}
```

或使用 `terraform login` 进行认证：

```bash
terraform login
terraform init
```

Terraform Cloud 特性：

| 特性 | 说明 |
|------|------|
| 远程执行 | 在云端运行 Terraform，无需本地配置 |
| 状态管理 | 自动管理状态，支持团队协作 |
| 变量集 | 共享变量，多工作区复用 |
| 运行历史 | 完整的执行记录和审计日志 |
| 策略即代码 | OPA 集成，策略检查 |
| 私有注册表 | 托管私有模块 |

---

## 参考资料

[^1]: Terraform Documentation. https://www.terraform.io/docs

[^2]: Terraform Up & Running - Yevgeniy Brikman. https://www.terraformupandrunning.com/

[^3]: HashiCorp Configuration Language. https://developer.hashicorp.com/terraform/language

[^4]: Terraform Best Practices. https://www.terraform-best-practices.com/

[^5]: AWS Provider Documentation. https://registry.terraform.io/providers/hashicorp/aws/latest/docs
