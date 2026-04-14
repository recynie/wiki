---
title: Ansible 自动化运维工具
date: 2026-04-13
description: Ansible自动化运维工具：主机清单、Playbook、Roles、模块与最佳实践
tags:
  - ansible
  - devops
  - automation
  - configuration-management
  - playbook
draft: false
permalink:
---

## Ansible 简介

Ansible是一款开源的自动化运维工具，基于Python开发，用于批量配置主机、部署应用和执行运维任务。[^1]

### 核心特点

- **无代理（Agentless）**：无需在被管节点安装任何软件，通过SSH协议通信
- **幂等性（Idempotency）**：多次执行结果一致，重复执行不会产生副作用
- ** YAML 语法**：使用人类可读的YAML格式描述配置和任务
- **模块化设计**：丰富的内置模块，支持自定义扩展
- **并行执行**：支持多主机并行操作，提高效率

### 架构组件

```
┌─────────────────────────────────────────────────────────────┐
│                      Ansible 控制节点                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   Inventory  │  │   Playbook  │  │   Modules   │        │
│  │   主机清单    │  │   剧本      │  │   模块库    │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
│                          │                                  │
│                    ┌─────┴─────┐                            │
│                    │   Engine   │                            │
│                    └─────┬─────┘                            │
└──────────────────────────┼──────────────────────────────────┘
                           │ SSH
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Host 1     │    │  Host 2     │    │  Host N     │
│  被管节点   │    │  被管节点   │    │  被管节点   │
└─────────────┘    └─────────────┘    └─────────────┘
```

### 与其他工具对比

| 特性 | Ansible | Chef | Puppet | SaltStack |
|------|---------|------|--------|-----------|
| 代理需求 | 无 | 需要 | 需要 | 需要 |
| 学习曲线 | 低 | 中 | 中 | 中 |
| 配置语言 | YAML | Ruby DSL | Ruby DSL | YAML/Python |
| 社区活跃度 | 非常高 | 高 | 高 | 中 |
| 无服务器架构 | 支持 | 不支持 | 不支持 | 不支持 |

---

## 安装与配置

### 安装方式

#### Ubuntu / Debian

```bash
sudo apt update
sudo apt install ansible -y
```

#### RHEL / CentOS / Fedora

```bash
sudo dnf install ansible -y
```

#### macOS

```bash
brew install ansible
```

#### pip 安装（推荐）

```bash
pip install ansible
```

#### 验证安装

```bash
ansible --version
# ansible 2.10.x
#   config file = None
#   configured module search path = ['~/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
#   ansible python module location = /usr/lib/python3.11/site-packages/ansible
#   executable location = /usr/bin/ansible
```

### 环境准备

#### 控制节点要求

- Linux / macOS / Windows（通过WSL）
- Python 3.8 或更高版本
- SSH 客户端

#### 被管节点要求

- SSH 服务运行中
- Python 3.5 或更高版本（大多数模块需要）

### SSH 配置

#### 免密码登录（密钥认证）

```bash
# 生成SSH密钥对
ssh-keygen -t ed25519 -C "ansible@control-node" -f ~/.ssh/ansible_key

# 复制公钥到被管节点
ssh-copy-id -i ~/.ssh/ansible_key.pub user@192.168.1.10
ssh-copy-id -i ~/.ssh/ansible_key.pub user@192.168.1.11

# 测试连接
ssh -i ~/.ssh/ansible_key user@192.168.1.10 "hostname"
```

#### SSH 配置优化

编辑 `~/.ssh/config` 文件：

```
# Ansible 控制节点配置
Host *
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    IdentityFile ~/.ssh/ansible_key
    ConnectTimeout 10
    ServerAliveInterval 60

# 主机别名示例
Host webserver
    HostName 192.168.1.10
    User admin
    Port 22
```

### ansible.cfg 配置文件

#### 配置文件查找顺序

1. `ANSIBLE_CONFIG` 环境变量指定的文件
2. `./ansible.cfg`（当前目录）
3. `~/.ansible.cfg`（用户家目录）
4. `/etc/ansible/ansible.cfg`

#### 常用配置项

```ini
[defaults]
# 主机清单路径
inventory = ./inventory

# 远程用户
remote_user = admin

# 私钥路径
private_key_file = ~/.ssh/ansible_key

# 关闭主机密钥检查
host_key_checking = False

# 并行任务数
forks = 10

# 失败时是否继续
# uncomment to change
# any_unreachable_sts = False

# 输出格式（minimal | yaml | json | tree）
# display_skipped_hosts = True

[privilege_escalation]
# 权限提升
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
# SSH优化
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

---

## 主机清单（Inventory）

主机清单定义了被管节点的信息，是Ansible连接主机的依据。

### 静态清单

静态清单是一个文本文件，默认路径为 `/etc/ansible/hosts`。

#### INI 格式

```ini
# 单个主机
web1.example.com

# 主机带端口
db1.example.com:2222

# 主机组
[webservers]
web1.example.com
web2.example.com
192.168.1.20

[dbservers]
db1.example.com
db2.example.com

[mail]
mail.example.com

# 主机组嵌套
[production:children]
webservers
dbservers
mail

# 主机组变量（仅用于该组）
[webservers:vars]
http_port=80
max_clients=200
ansible_user=admin
ansible_ssh_private_key_file=~/.ssh/ansible_key

[dbservers:vars]
ansible_user=dbadmin
db_port=5432
```

#### YAML 格式

```yaml
all:
  hosts:
    web1.example.com:
    web2.example.com:
  children:
    webservers:
      hosts:
        web1.example.com:
        web2.example.com:
    dbservers:
      hosts:
        db1.example.com:
        db2.example.com:
  vars:
    ansible_user: admin
```

### 主机范围与模式匹配

```ini
# IP范围
[webservers]
192.168.1.[10:20]    # 192.168.1.10 到 192.168.1.20

# 字母范围（web01-web05）
[appserver]
web[01:05].example.com

# 多个组
[multi]
webservers[1:3]
dbservers[1:2]
```

### 动态清单

动态清单脚本从外部来源（如云API、CMDB）获取主机信息。

#### AWS EC2 动态清单

```bash
# 安装EC2插件
pip install boto3 botocore

# 配置AWS凭证
export AWS_ACCESS_KEY_ID='AK...'
export AWS_SECRET_ACCESS_KEY='...'

# 使用动态清单
ansible-inventory -i ec2.py --list
```

#### 自定义动态清单脚本

```python
#!/usr/bin/env python3
# my_inventory.py
import json

def get_inventory():
    inventory = {
        'group': {
            'hosts': ['server1.example.com', 'server2.example.com'],
            'vars': {
                'ansible_user': 'admin'
            }
        },
        '_meta': {
            'hostvars': {
                'server1.example.com': {
                    'host_ip': '192.168.1.10',
                    'environment': 'prod'
                },
                'server2.example.com': {
                    'host_ip': '192.168.1.11',
                    'environment': 'staging'
                }
            }
        }
    }
    return inventory

if __name__ == '__main__':
    print(json.dumps(get_inventory()))
```

使用动态清单：

```bash
chmod +x my_inventory.py
ansible all -i my_inventory.py -m ping
```

### 主机清单命令

```bash
# 列出所有主机
ansible-inventory -i inventory --list

# 图形化显示
ansible-inventory -i inventory --graph

# 列出特定组的主机
ansible-inventory -i inventory --host=web1.example.com

# 测试清单连通性
ansible all -i inventory -m ping
```

---

## 常用模块

Ansible模块是执行具体任务的核心组件，分为内置模块和自定义模块。

### 模块分类

| 类别 | 模块示例 | 用途 |
|------|---------|------|
| 包管理 | apt, yum, dnf, pip | 安装/卸载软件包 |
| 文件操作 | copy, template, file, lineinfile | 文件管理 |
| 服务管理 | service, systemd | 启动/停止/重启服务 |
| 命令执行 | command, shell, script | 执行命令 |
| 系统配置 | user, group, cron, hostname | 系统设置 |
| 网络 | get_url, uri, fetch | 网络相关操作 |
| 数据库 | mysql_db, postgresql_db | 数据库管理 |
| 云服务 | ec2, azure_rm, gce | 云资源管理 |

### 包管理模块

#### apt（Debian/Ubuntu）

```bash
# 安装软件包
ansible all -m apt -a "name=nginx state=present"

# 安装多个包
ansible all -m apt -a "name=nginx,git,vim state=present"

# 更新软件包缓存并升级
ansible all -m apt -a "update_cache=yes state=latest"

# 删除软件包
ansible all -m apt -a "name=nginx state=absent"

# 安装特定版本
ansible all -m apt -a "name=nginx=1.18.0 state=present"

# 安装.deb文件
ansible all -m apt -a "deb=/tmp/package.deb"
```

#### yum/dnf（RHEL/CentOS/Fedora）

```bash
# 安装
ansible all -m yum -a "name=httpd state=present"

# 更新所有包
ansible all -m yum -a "name=* state=latest"

# 删除
ansible all -m yum -a "name=httpd state=absent"

# 使用dnf（Fedora）
ansible all -m dnf -a "name=nginx state=present"
```

#### pip（Python包）

```bash
# 安装Python包
ansible all -m pip -a "name=flask state=present"

# 安装 requirements.txt
ansible all -m pip -a "requirements=/opt/requirements.txt"

# 指定版本
ansible all -m pip -a "name=django==4.2.0 state=present"
```

### 文件操作模块

#### copy 模块

复制文件到远程主机：

```bash
# 复制本地文件到远程
ansible all -m copy -a "src=/local/file dest=/remote/path"

# 带权限和所有者
ansible all -m copy -a "src=/local/file dest=/remote/path mode=0644 owner=www-data group=www-data"

# 备份原文件
ansible all -m copy -a "src=/local/file dest=/remote/path backup=yes"

# 目录内容
ansible all -m copy -a "src=/local/dir/ dest=/remote/path directory_mode=True"
```

#### template 模块

复制模板文件（Jinja2），支持变量替换：

```bash
# 复制模板
ansible all -m template -a "src=config.j2 dest=/etc/app/config.conf"

# 带权限
ansible all -m template -a "src=nginx.conf.j2 dest=/etc/nginx/nginx.conf mode=0644"
```

模板文件示例 `nginx.conf.j2`：

```nginx
server {
    listen {{ http_port }};
    server_name {{ server_name }};
    
    location / {
        proxy_pass http://{{ upstream_backend }};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    access_log /var/log/nginx/{{ app_name }}_access.log;
    error_log /var/log/nginx/{{ app_name }}_error.log;
}
```

#### file 模块

创建/修改文件属性：

```bash
# 创建目录
ansible all -m file -a "path=/opt/app state=directory mode=0755 owner=www-data group=www-data"

# 创建符号链接
ansible all -m file -a "src=/etc/resolv.conf dest=/etc/dns.conf state=link"

# 创建空文件
ansible all -m file -a "path=/tmp/test.txt state=touch mode=0644"

# 删除文件或目录
ansible all -m file -a "path=/opt/app state=absent"
```

#### lineinfile 模块

修改文件中的特定行：

```bash
# 确保一行存在
ansible all -m lineinfile -a "path=/etc/sysctl.conf line='kernel.shmmax = 68719476736'"

# 使用正则表达式修改
ansible all -m lineinfile -a "path=/etc/hosts regexp='^127.0.0.1' line='127.0.0.1 localhost'"

# 在特定行后插入
ansible all -m lineinfile -a "path=/etc/file line='new line' insertafter='^pattern'"

# 删除匹配的行
ansible all -m lineinfile -a "path=/etc/file regexp='^# old config' state=absent"
```

### 服务管理模块

#### service/systemd 模块

```bash
# 启动服务
ansible all -m service -a "name=nginx state=started"

# 停止服务
ansible all -m service -a "name=nginx state=stopped"

# 重启服务
ansible all -m service -a "name=nginx state=restarted"

# 重新加载配置
ansible all -m service -a "name=nginx state=reloaded"

# 设置开机启动
ansible all -m service -a "name=nginx state=started enabled=yes"

# 使用systemd模块（推荐）
ansible all -m systemd -a "name=nginx state=restarted enabled=yes daemon_reload=yes"
```

### 命令执行模块

#### command 模块

直接执行命令（不通过shell）：

```bash
# 执行简单命令
ansible all -m command -a "ls -la /home"

# 使用creates选项（幂等性）
ansible all -m command -a "creates=/var/lock/example /usr/bin/some_script.sh"

# 使用removes选项
ansible all -m command -a "removes=/var/lock/example /usr/bin/cleanup.sh"

# 执行多个命令
ansible all -m command -a "cmd: ls /home && df -h && uptime"
```

#### shell 模块

通过shell执行（支持管道、重定向）：

```bash
# 支持管道
ansible all -m shell -a "ps aux | grep nginx"

# 重定向输出
ansible all -m shell -a "echo 'text' > /tmp/file"

# 复杂命令
ansible all -m shell -a "for i in {1..5}; do echo $i; done"
```

#### script 模块

在远程主机执行本地脚本：

```bash
# 创建本地脚本
cat > /tmp/setup.sh << 'EOF'
#!/bin/bash
yum install -y nginx
systemctl enable nginx
systemctl start nginx
EOF

# 在所有远程主机执行
ansible all -m script -a "/tmp/setup.sh"
```

### 系统模块

#### user 模块

```bash
# 创建用户
ansible all -m user -a "name=appuser comment='Application User' shell=/bin/bash"

# 创建系统用户（无登录shell）
ansible all -m user -a "name=appservice system=yes shell=/sbin/nologin"

# 设置密码
ansible all -m user -a "name=appuser password='{{ \"mypassword\" | password_hash(\"sha512\") }}'"

# 添加SSH密钥
ansible all -m user -a "name=appuser generate_ssh_key=yes ssh_key_bits=4096"

# 删除用户
ansible all -m user -a "name=appuser state=absent remove=yes"
```

#### group 模块

```bash
# 创建组
ansible all -m group -a "name=www-data"

# 创建系统组
ansible all -m group -a "name=appsystem system=yes"

# 删除组
ansible all -m group -a "name=appsystem state=absent"
```

#### cron 模块

```bash
# 添加定时任务
ansible all -m cron -a "name='backup' minute='0' hour='2' job='/usr/local/bin/backup.sh'"

# 每天凌晨3点执行
ansible all -m cron -a "name='daily-task' hour='3' day='*' month='*' weekday='*' job='/opt/daily.sh'"

# 删除定时任务
ansible all -m cron -a "name='backup' state=absent"

# 禁用定时任务
ansible all -m cron -a "name='backup' disabled=yes"
```

### 获取模块帮助

```bash
# 查看模块文档
ansible-doc copy
ansible-doc service
ansible-doc user

# 列出所有模块
ansible-doc -l

# 查看特定模块的示例
ansible-doc -s copy
```

---

## Playbook 基础

Playbook是Ansible的核心组件，使用YAML格式描述配置和任务，实现基础设施即代码（IaC）。

### YAML 语法基础

```yaml
# 注释
# 键值对
name: value
number: 42
enabled: true

# 列表
packages:
  - nginx
  - git
  - vim

# 嵌套对象
server:
  host: localhost
  port: 8080
  config:
    timeout: 30

# 多行字符串
description: |
  This is a multi-line
  string value.

inline: >-
  This is a folded
  single line.
```

### Playbook 基本结构

```yaml
---
# 第一层：文件开头（可选）
- name: Play名称                          # Play描述
  hosts: webservers                        # 目标主机
  remote_user: admin                       # 远程用户
  become: yes                              # 是否提权
  become_user: root                        # 提权用户
  
  vars:                                    # 变量定义
    http_port: 80
    app_name: myapp
  
  vars_files:                              # 变量文件
    - vars/secrets.yml
  
  tasks:                                   # 任务列表
    - name: Task名称
      模块名: 参数
```

### 完整 Playbook 示例

```yaml
---
- name: 配置 Web 服务器
  hosts: webservers
  become: yes
  become_user: root
  
  vars:
    nginx_version: "1.18.0"
    app_directory: /opt/myapp
  
  vars_files:
    - secrets.yml
  
  pre_tasks:                               # 主任务前执行
    - name: 更新 apt 缓存
      apt:
        update_cache: yes
      when: ansible_os_family == "Debian"
  
  tasks:
    # 安装 Nginx
    - name: 安装 Nginx
      apt:
        name: nginx
        state: present
        version: "{{ nginx_version }}"
      notify: 重启 Nginx
    
    # 配置 Nginx
    - name: 复制 Nginx 配置
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: 重启 Nginx
    
    # 部署应用
    - name: 创建应用目录
      file:
        path: "{{ app_directory }}"
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'
    
    - name: 部署应用文件
      copy:
        src: app/
        dest: "{{ app_directory }}/"
        owner: www-data
        group: www-data
    
    # 启动服务
    - name: 启动 Nginx
      service:
        name: nginx
        state: started
        enabled: yes
  
  handlers:                                 # 处理器
    - name: 重启 Nginx
      service:
        name: nginx
        state: restarted
```

### 任务（Tasks）

任务定义了在目标主机上执行的操作。

```yaml
tasks:
  # 基本任务格式
  - name: 安装软件
    apt:
      name: vim
      state: present
  
  # 多个操作（使用列表）
  - name: 配置时间同步
    blockinfile:
      path: /etc/chrony/chrony.conf
      block: |
        server 0.pool.ntp.org iburst
        server 1.pool.ntp.org iburst
      marker: "# {mark} ANSIBLE MANAGED BLOCK"
  
  # 忽略错误继续执行
  - name: 执行可能失败的操作
    command: /opt/risky-script.sh
    ignore_errors: yes
    register: result
  
  # 注册变量并使用
  - name: 获取系统信息
    command: uname -a
    register: system_info
  
  - name: 显示系统信息
    debug:
      var: system_info.stdout
  
  # 等待条件
  - name: 等待端口就绪
    wait_for:
      port: 5432
      host: localhost
      delay: 5
      timeout: 30
      state: started
```

### 处理器（Handlers）

处理器是任务调用的特殊任务，只有被任务通知时才执行。

```yaml
handlers:
  # 基本处理器
  - name: 重启 Nginx
    service:
      name: nginx
      state: restarted
  
  # 带条件
  - name: 重启应用
    service:
      name: myapp
      state: restarted
    listen: "restart services"
  
  # 多个处理器
  - name: 重载配置
    service:
      name: nginx
      state: reloaded
    listen: "restart services"

tasks:
  - name: 修改配置
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: 重启 Nginx          # 通知单个处理器
    # notify: "restart services"  # 通知多个处理器（通过listen）
```

### 执行 Playbook

```bash
# 语法检查
ansible-playbook playbook.yml --syntax-check

# 列出要执行的任务（不实际执行）
ansible-playbook playbook.yml --list-tasks

# 列出主机
ansible-playbook playbook.yml --list-hosts

# 模拟执行（检查）
ansible-playbook playbook.yml --check

# 模拟执行并显示差异
ansible-playbook playbook.yml --check --diff

# 指定主机清单
ansible-playbook -i inventory playbook.yml

# 指定主机（调试用）
ansible-playbook playbook.yml --limit webserver1

# 跳过特定标签
ansible-playbook playbook.yml --skip-tags=nginx

# 执行单个标签
ansible-playbook playbook.yml --tags=nginx

# 详细输出
ansible-playbook playbook.yml -v

# 更详细输出（-vvvv）
ansible-playbook playbook.yml -vvvv
```

---

## Playbook 进阶

### 变量（Variables）

#### 定义变量

```yaml
# Play级别变量
- name: 示例 Play
  hosts: webservers
  vars:
    app_name: myapp
    app_version: "1.0.0"
    app_ports:
      - 80
      - 443
    app_config:
      debug: true
      log_level: info
```

```yaml
# 任务级别变量
tasks:
  - name: 创建应用
    user:
      name: "{{ app_name }}"
      comment: "{{ app_name }} user"
```

#### 变量来源

```yaml
# 1. 在Play中定义
vars:
  http_port: 8080

# 2. 外部变量文件
vars_files:
  - vars/secrets.yml
  - vars/app_config.yml

# 3. 命令行传递
# ansible-playbook playbook.yml -e "http_port=9000"
vars:
  http_port: "{{ http_port }}"

# 4. Inventory变量
# 在 inventory 文件中定义
[webservers:vars]
http_port=80

# 5. 注册变量
tasks:
  - name: 执行命令
    command: whoami
    register: whoami_result
  
  - name: 使用注册变量
    debug:
      var: whoami_result.stdout

# 6.Facts变量（系统信息）
# Ansible自动收集的主机信息
debug:
  msg: "{{ ansible_hostname }} - {{ ansible_os_family }} - {{ ansible_default_ipv4.address }}"

# 7. 魔法变量
# Ansible定义的特殊变量
debug:
  msg: "{{ groups.all }}"        # 所有主机列表
  msg: "{{ hostvars[inventory_hostname] }}"  # 当前主机变量
```

#### 变量优先级（从低到高）

1. Inventory 变量
2. Play `vars`
3. Play `vars_files`
4. 角色 `defaults/main.yml`
5. 命令行 `-e`
6. 角色 `vars/main.yml`
7. 角色 `vars`（非 `main.yml`）

### 条件判断

#### when 语句

```yaml
tasks:
  # 单条件
  - name: 安装 Apache（Debian）
    apt:
      name: apache2
      state: present
    when: ansible_os_family == "Debian"
  
  # 多条件（AND）
  - name: 安装 Memcached
    apt:
      name: memcached
      state: present
    when:
      - ansible_os_family == "Debian"
      - ansible_memory_mb.total >= 512
  
  # OR 条件
  - name: 安装编辑器
    apt:
      name: vim
      state: present
    when: (ansible_os_family == "Debian") or (ansible_os_family == "RedHat")
  
  # NOT 条件
  - name: 非生产环境执行
    debug:
      msg: "这不是生产环境"
    when: environment != "production"
  
  # 变量存在性检查
  - name: 当变量存在时执行
    debug:
      msg: "{{ custom_var }}"
    when: custom_var is defined
  
  # 字符串判断
  - name: 字符串匹配
    debug:
      msg: "Development server"
    when: ansible_hostname is match("dev-*")
```

#### changed_when 和 failed_when

```yaml
tasks:
  - name: 执行自定义脚本
    command: /opt/check-status.sh
    register: result
    changed_when: result.rc == 0
    failed_when: result.rc > 2
  
  - name: 检查结果
    fail:
      msg: "Service is unhealthy"
    when: "'ERROR' in result.stdout"
```

### 循环（Loops）

#### 标准循环

```yaml
tasks:
  # 循环安装包
  - name: 安装多个软件
    apt:
      name: "{{ item }}"
      state: present
    loop:
      - nginx
      - git
      - vim
      - curl
  
  # 循环创建用户
  - name: 创建多个用户
    user:
      name: "{{ item.name }}"
      state: present
      shell: "{{ item.shell | default('/bin/bash') }}"
    loop:
      - { name: 'user1', shell: '/bin/bash' }
      - { name: 'user2', shell: '/sbin/nologin' }
      - { name: 'user3' }
```

#### with_* 循环

```yaml
tasks:
  # with_items（等同于 loop）
  - name: 创建目录
    file:
      path: "{{ item }}"
      state: directory
    with_items:
      - /opt/app1
      - /opt/app2
      - /opt/data
  
  # with_dict（字典循环）
  - name: 配置多个网站
    template:
      src: site.conf.j2
      dest: "/etc/nginx/sites-available/{{ item.key }}.conf"
    with_dict: "{{ sites }}"
  # sites = { site1: { port: 80 }, site2: { port: 8080 } }
  
  # with_fileglob（文件匹配）
  - name: 复制配置文件
    copy:
      src: "{{ item }}"
      dest: "/etc/app/config/"
      mode: '0644'
    with_fileglob:
      - "configs/*.conf"
```

#### 循环控制

```yaml
tasks:
  # 跳过循环中的特定项
  - name: 安装软件（非测试环境）
    apt:
      name: "{{ item }}"
      state: present
    loop:
      - nginx
      - postgresql
      - redis
    when: item != "postgresql" or environment == "production"
  
  # 循环中的错误处理
  - name: 批量执行（继续即使失败）
    apt:
      name: "{{ item }}"
      state: present
    loop:
      - package1
      - package2
      - package3
    register: apt_result
    failed_when: "'No package' not in apt_result.stderr"
```

### 错误处理

#### block 和 rescue

```yaml
tasks:
  - name: 部署应用
    block:
      - name: 备份旧版本
        command: /opt/backup.sh
      
      - name: 部署新版本
        command: /opt/deploy.sh
      
      - name: 验证部署
        uri:
          url: http://localhost:8080/health
          status_code: 200
    rescue:
      - name: 部署失败，回滚
        command: /opt/rollback.sh
      
      - name: 通知负责人
        debug:
          msg: "部署失败，请检查！"
```

#### 错误处理变量

```yaml
tasks:
  - name: 执行部署
    block:
      - name: 运行部署脚本
        command: /opt/deploy.sh
    register: deploy_result
    changed_when: "'SUCCESS' in deploy_result.stdout"
    failed_when: deploy_result.rc > 0
  
  - name: 处理失败
    debug:
      msg: "部署失败，退出码: {{ deploy_result.rc }}"
    when: deploy_result is failed
  
  - name: 处理成功
    debug:
      msg: "部署成功"
    when: deploy_result is succeeded
```

#### changed 和 failed 条件组合

```yaml
tasks:
  - name: 执行健康检查
    command: /opt/healthcheck.sh
    register: health
    failed_when: health.rc not in [0, 1]
    changed_when: health.rc == 0
```

### 标签（Tags）

```yaml
tasks:
  - name: 安装依赖
    apt:
      name: "{{ item }}"
    loop:
      - nginx
      - git
    tags:
      - install
      - nginx
  
  - name: 配置 Nginx
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    tags:
      - nginx
      - config
  
  - name: 部署应用
    copy:
      src: app/
      dest: /opt/app
    tags:
      - deploy

# 使用标签
ansible-playbook playbook.yml --tags=install
ansible-playbook playbook.yml --tags=nginx,config
ansible-playbook playbook.yml --skip-tags=deploy
```

---

## Roles 角色

Roles是组织Playbook的高级方式，通过预定义目录结构实现任务、变量、处理器、模板的模块化。

### 目录结构

```
playbook.yml
roles/
└── common/
    ├── defaults/           # 默认变量（最低优先级）
    │   └── main.yml
    ├── files/              # 静态文件
    │   ├── config.conf
    │   └── script.sh
    ├── handlers/            # 处理器
    │   └── main.yml
    ├── meta/               # 角色元数据
    │   └── main.yml
    ├── tasks/              # 任务
    │   └── main.yml
    ├── templates/          # 模板文件（Jinja2）
    │   ├── nginx.conf.j2
    │   └── app.conf.j2
    ├── tests/              # 测试
    │   ├── inventory
    │   └── test.yml
    └── vars/              # 角色变量（高优先级）
        └── main.yml
```

### 创建 Role

```bash
# 使用 ansible-galaxy 创建
ansible-galaxy init roles/common

# 查看结构
tree roles/common/
```

### Role 示例：Web 服务器

#### roles/webserver/defaults/main.yml

```yaml
---
# 默认变量（可被覆盖）
http_port: 80
server_name: localhost
doc_root: /var/www/html
app_user: www-data
nginx_version: "1.18.0"
```

#### roles/webserver/vars/main.yml

```yaml
---
# 角色内部变量（优先级高）
nginx_config_dir: /etc/nginx/sites-available
nginx_enable_dir: /etc/nginx/sites-enabled
```

#### roles/webserver/tasks/main.yml

```yaml
---
- name: 安装 Nginx
  apt:
    name:
      - nginx
      - python3-pip
    state: present
    update_cache: yes
  notify: 重启 Nginx

- name: 创建文档根目录
  file:
    path: "{{ doc_root }}"
    state: directory
    owner: "{{ app_user }}"
    group: "{{ app_user }}"
    mode: '0755'

- name: 复制 Nginx 配置
  template:
    src: nginx.conf.j2
    dest: "{{ nginx_config_dir }}/{{ server_name }}"
    owner: root
    group: root
    mode: '0644'
  notify: 启用站点配置

- name: 部署静态文件
  synchronize:
    src: files/html/
    dest: "{{ doc_root }}/"
  when: synchronize_available|default(false)

- name: 启动 Nginx
  service:
    name: nginx
    state: started
    enabled: yes
```

#### roles/webserver/handlers/main.yml

```yaml
---
- name: 重启 Nginx
  service:
    name: nginx
    state: restarted

- name: 重新加载 Nginx
  service:
    name: nginx
    state: reloaded

- name: 启用站点配置
  file:
    src: "{{ nginx_config_dir }}/{{ server_name }}"
    dest: "{{ nginx_enable_dir }}/{{ server_name }}"
    state: link
  notify: 重新加载 Nginx
```

#### roles/webserver/templates/nginx.conf.j2

```nginx
server {
    listen {{ http_port }};
    server_name {{ server_name }};
    
    root {{ doc_root }};
    index index.html index.htm;
    
    access_log /var/log/nginx/{{ server_name }}_access.log;
    error_log /var/log/nginx/{{ server_name }}_error.log;
    
    location / {
        try_files $uri $uri/ =404;
    }
    
    {% if nginx_gzip|default(true) %}
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    {% endif %}
}
```

#### roles/webserver/files/script/healthcheck.sh

```bash
#!/bin/bash
curl -f http://localhost:{{ http_port }}/health || exit 1
```

### 使用 Role

#### playbook.yml

```yaml
---
- name: 部署 Web 服务器
  hosts: webservers
  become: yes
  
  roles:
    - role: common
      tags: common
    
    - role: webserver
      vars:
        http_port: 8080
        server_name: myapp.example.com
      tags: web
  
  post_tasks:
    - name: 验证部署
      script: roles/webserver/files/script/healthcheck.sh
      register: health_result
      failed_when: health_result.rc != 0
```

### Role 依赖（Dependencies）

#### roles/common/meta/main.yml

```yaml
---
allow_duplicates: yes
dependencies:
  - role: monitoring
    vars:
      monitoring_enabled: true
```

### Role 搜索路径

```bash
# 默认搜索路径
# 1. playbook同目录下的 roles/
# 2. ~/.ansible/roles/
# 3. /etc/ansible/roles/

# 指定额外路径
ansible-playbook playbook.yml --roles-path /opt/ansible/roles
```

### ansible-galaxy 命令

```bash
# 搜索角色
ansible-galaxy search nginx

# 查看角色信息
ansible-galaxy info username.rolename

# 安装角色
ansible-galaxy install username.rolename

# 从 requirements.yml 安装
ansible-galaxy install -r requirements.yml

# requirements.yml 示例
# roles/
#   - src: geerlingguy.nginx
#     version: "3.7.0"
#     name: nginx
#   - src: geerlingguy.mysql
#     version: "3.3.0"
```

---

## 最佳实践

### 项目目录结构

```
project/
├── ansible.cfg                 # Ansible配置
├── inventory/                  # 主机清单
│   ├── production/            # 生产环境
│   │   ├── hosts
│   │   └── group_vars/
│   │       └── all.yml
│   └── staging/               # 测试环境
│       ├── hosts
│       └── group_vars/
├── playbooks/                 # Playbook存放
│   ├── site.yml               # 主入口
│   ├── webserver.yml
│   ├── database.yml
│   └── monitoring.yml
├── roles/                      # 角色
│   ├── common/
│   ├── webserver/
│   ├── database/
│   └── monitoring/
├── vars/                      # 全局变量
│   └── env.yml
├── templates/                 # 共享模板
├── files/                     # 共享文件
├── scripts/                   # 辅助脚本
├── requirements.yml           # 依赖角色
├── .gitignore
└── README.md
```

### inventory/production/hosts

```ini
[webservers]
web[1:5].example.com

[dbservers]
db1.example.com
db2.example.com ansible_user=dbadmin

[production:children]
webservers
dbservers

[production:vars]
environment=production
ansible_user=admin
```

### ansible.cfg 最佳配置

```ini
[defaults]
inventory = inventory/production
remote_user = admin
host_key_checking = False
retry_files_enabled = False
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 86400

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no
```

### Playbook 编写规范

```yaml
---
# 1. 总是指定 name
- name: 安装 Nginx

# 2. 使用 handler 处理服务重启
- name: 配置 Nginx
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: 重启 Nginx

# 3. 使用 block/rescue 处理错误
- name: 部署应用
  block:
    - name: 执行部署
      command: /opt/deploy.sh
  rescue:
    - name: 回滚
      command: /opt/rollback.sh

# 4. 使用 tags 分类任务
- name: 安装依赖
  apt:
    name: "{{ item }}"
  loop:
    - nginx
    - git
  tags:
    - install

# 5. 避免 bare variables
- name: 正确
  debug:
    msg: "{{ variable_name }}"

# 6. 使用 with_dict 或 loop 替代重复任务
- name: 创建多个目录
  file:
    path: "{{ item }}"
    state: directory
  loop:
    - /opt/app1
    - /opt/app2
    - /opt/data
```

### 安全最佳实践

```yaml
# 1. 使用 vault 加密敏感数据
# ansible-vault create vars/secrets.yml
# ansible-vault edit vars/secrets.yml

- name: 使用加密变量文件
  hosts: webservers
  vars_files:
    - vars/secrets.yml        # 需要输入密码

# 2. 不在命令行或代码中明文存储密码
# 使用 Ansible Vault 或环境变量

# 3. 限制 privilege escalation
- name: 仅必要时提权
  hosts: webservers
  become: yes
  become_user: root           # 指定具体用户，不滥用

# 4. 审计和日志
# 配置日志记录
ansible-playbook playbook.yml | tee execution.log
```

### 模块化设计

#### 通用角色设计

```yaml
# roles/base/tasks/main.yml
---
- name: 更新系统
  apt:
    upgrade: dist
    update_cache: yes
    autoremove: yes
  when: ansible_os_family == "Debian"

- name: 设置主机名
  hostname:
    name: "{{ inventory_hostname }}"

- name: 配置 DNS
  lineinfile:
    path: /etc/resolv.conf
    line: "{{ item }}"
  loop:
    - "nameserver {{ dns_server1 }}"
    - "nameserver {{ dns_server2 }}"

# roles/base/defaults/main.yml
---
dns_server1: 8.8.8.8
dns_server2: 8.8.4.4
```

#### 组合 Playbook

```yaml
# site.yml - 主入口
---
- name: 基础配置（所有主机）
  import_playbook: playbooks/base.yml

- name: Web 服务器配置
  import_playbook: playbooks/webservers.yml

- name: 数据库服务器配置
  import_playbook: playbooks/databases.yml

# playbooks/base.yml
---
- name: 应用基础配置
  hosts: all
  roles:
    - base
```

### 版本控制

```bash
# .gitignore
*.log
*.retry
.vault_pass
.pyc
__pycache__/
//tmp/
/inventory/*/
!inventory/.gitkeep

# 提交规范
# git commit -m "feat: add nginx configuration role"
# git commit -m "fix: correct database user permissions"
# git commit -m "refactor: restructure playbook hierarchy"
```

### 测试和验证

```bash
# 语法检查
ansible-playbook site.yml --syntax-check

# 列出主机和任务
ansible-playbook site.yml --list-hosts --list-tasks

# Dry run
ansible-playbook site.yml --check --diff

# 限制执行主机
ansible-playbook site.yml --limit webserver1

# 测试特定角色
ansible-playbook site.yml --tags nginx --check
```

---

## 实战案例

### 部署 Node.js 应用

```yaml
# deploy-nodejs.yml
---
- name: 部署 Node.js 应用
  hosts: webservers
  become: yes
  vars:
    app_name: mynodeapp
    app_version: "1.0.0"
    app_port: 3000
    node_version: "18"
  
  vars_files:
    - vars/{{ environment }}.yml
  
  tasks:
    - name: 安装 Node.js
      apt:
        name:
          - nodejs
          - npm
          - nginx
        state: present
        update_cache: yes
    
    - name: 创建应用用户
      user:
        name: "{{ app_name }}"
        home: /opt/{{ app_name }}
        shell: /bin/bash
        system: yes
    
    - name: 创建应用目录
      file:
        path: "/opt/{{ app_name }}"
        state: directory
        owner: "{{ app_name }}"
        group: "{{ app_name }}"
        mode: '0755'
    
    - name: 复制应用代码
      synchronize:
        src: app/
        dest: "/opt/{{ app_name }}/"
        delete: yes
        rsync_opts:
          - "--exclude=node_modules"
      become_user: "{{ app_name }}"
    
    - name: 安装依赖
      npm:
        name: "{{ item }}"
        state: present
        global: yes
      loop:
        - pm2
        - forever
    
    - name: 配置 PM2
      copy:
        src: ecosystem.config.js
        dest: "/opt/{{ app_name }}/"
        owner: "{{ app_name }}"
        mode: '0644'
    
    - name: 启动应用
      shell: |
        cd /opt/{{ app_name }}
        pm2 start ecosystem.config.js
        pm2 save
      become_user: "{{ app_name }}"
      environment:
        NODE_ENV: "{{ environment }}"
    
    - name: 配置 Nginx 反向代理
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/sites-available/{{ app_name }}
        mode: '0644'
      notify: 重新加载 Nginx
    
    - name: 启用 Nginx 配置
      file:
        src: /etc/nginx/sites-available/{{ app_name }}
        dest: /etc/nginx/sites-enabled/{{ app_name }}
        state: link
      notify: 重新加载 Nginx

  handlers:
    - name: 重新加载 Nginx
      systemd:
        name: nginx
        state: reloaded
```

### 配置多环境部署

```yaml
# inventory/staging/hosts
[webservers]
staging-web1.example.com
staging-web2.example.com

[dbservers]
staging-db.example.com

[staging:children]
webservers
dbservers

# inventory/production/hosts
[webservers]
prod-web[1:5].example.com

[dbservers]
prod-db1.example.com
prod-db2.example.com

[production:children]
webservers
dbservers

# vars/staging.yml
---
environment: staging
app_port: 3000
debug: true
replicas: 1

# vars/production.yml
---
environment: production
app_port: 8080
debug: false
replicas: 3

# 部署时选择环境
# ansible-playbook deploy.yml -i inventory/staging
# ansible-playbook deploy.yml -i inventory/production
```

---

## 参考资料

[^1]: Ansible Documentation. https://docs.ansible.com/

[^2]: Ansible Up & Running - Lorin Hochstein. https://www.ansible.com/resources/get-started

[^3]: Ansible Best Practices. https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html
