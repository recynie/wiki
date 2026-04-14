---
title: Linux 基础
date: 2026-04-06
description: Linux 操作系统基础入门教程，涵盖文件系统、基础命令、用户权限、进程管理和网络命令等内容。
tags:
  - linux
  - shell
  - command-line
draft: false
permalink:
---

## 1. Linux 简介

### 1.1 Unix 历史

Unix 诞生于 1969 年的贝尔实验室，由 Ken Thompson 和 Dennis Ritchie 开发。最初是用汇编语言写的，后来 Dennis Ritchie 发明了 C 语言并用 C 重写了 Unix，使得 Unix 具备良好的可移植性。

Unix 的设计哲学是「小而精」——每个程序只做一件事，但做好它，并通过管道将它们组合起来完成复杂任务。这种哲学深深影响了后续的操作系统，包括 Linux。

### 1.2 Linux 内核

Linux 由 Linus Torvalds 于 1991 年在芬兰赫尔辛基大学求学期间开始开发。最初只是一个个人项目，旨在创建一个代替 Minix 的操作系统。如今，Linux 已成为世界上最成功的开源项目之一，内核代码由全球数千名开发者共同维护。

Linux 内核的主要功能包括：
- 进程管理
- 内存管理
- 文件系统
- 设备驱动
- 网络管理

### 1.3 发行版

由于 Linux 是开源的，任何人都可以基于内核打包自己的发行版。主要发行版包括：

| 发行版 | 特点 | 适用场景 |
|--------|------|----------|
| Ubuntu | 用户友好，社区活跃 | 桌面、服务器、入门 |
| CentOS/Rocky Linux | 企业级，稳定 | 服务器、生产环境 |
| Arch Linux | 滚动更新，高度定制 | 高级用户、定制系统 |
| Debian | 稳定可靠，软件包丰富 | 服务器、桌面 |
| Fedora | 前沿技术，验证新特性 | 开发者、桌面 |

---

## 2. 文件系统

### 2.1 目录结构

Linux 采用树形目录结构，所有文件从根目录 `/` 开始：

```bash
/               # 根目录
├── bin/        # 存放二进制可执行文件（系统启动时需要的命令）
├── etc/        # 系统配置文件
├── home/       # 普通用户的主目录
├── var/        # 经常变化的文件（日志、缓存等）
├── usr/        # 用户程序目录（应用程序、库文件等）
├── root/       # 超级用户 root 的主目录
├── tmp/        # 临时文件
├── dev/        # 设备文件
└── proc/       # 虚拟文件系统，展示内核信息
```

### 2.2 绝对路径 vs 相对路径

- **绝对路径**：以 `/` 开头的完整路径，例如 `/home/aroma/Documents`
- **相对路径**：相对于当前目录的路径，例如 `./Documents` 或 `../tools`

```bash
# 绝对路径
cd /home/aroma/Documents

# 相对路径（假设当前在 /home/aroma）
cd Documents          # 进入当前目录下的 Documents
cd ..                # 进入上一级目录
cd ../tools          # 进入上一级目录下的 tools
```

### 2.3 文件类型

Linux 中一切皆文件，主要文件类型包括：

| 类型 | 标识 | 说明 | 示例 |
|------|------|------|------|
| 普通文件 | `-` | 存放数据 | `file.txt`, `program` |
| 目录 | `d` | 文件夹 | `/home`, `/bin` |
| 符号链接 | `l` | 快捷方式 | `link -> target` |
| 字符设备 | `c` | 字符设备文件 | `/dev/tty` |
| 块设备 | `b` | 块设备文件 | `/dev/sda` |

---

## 3. 基础命令

### 3.1 文件操作

```bash
ls          # 列出目录内容
ls -la      # 列出详细信息（包括隐藏文件）
cd /path    # 切换目录
pwd         # 显示当前目录
mkdir dir   # 创建目录
rm file     # 删除文件
rm -r dir   # 递归删除目录
cp src dst  # 复制文件
mv src dst  # 移动/重命名文件
ln -s target link   # 创建符号链接
```

### 3.2 查看文件

```bash
cat file        # 显示整个文件内容
less file       # 分页查看（可上下滚动）
more file       # 分页查看（只能向下）
head -n 10 file # 查看文件前 10 行
tail -n 10 file # 查看文件后 10 行
tail -f file    # 实时查看文件更新
grep "pattern" file  # 搜索文件中匹配的行
```

### 3.3 文本处理

```bash
awk '{print $1}' file      # 提取第一列
sed 's/old/new/g' file    # 替换文本
sort file                 # 排序
uniq file                 # 去重（需先排序）
wc -l file                # 统计行数
cut -d':' -f1 file        # 按分隔符提取字段
```

### 3.4 查找

```bash
find /path -name "*.txt"       # 在指定路径下查找文件
find /path -type f -mtime -7   # 查找 7 天内修改过的文件
which python                   # 查找命令所在路径
whereis python                 # 查找命令的二进制、源码和手册位置
```

---

## 4. 用户与权限

### 4.1 文件权限

每个文件有三组权限：所有者（owner）、用户组（group）、其他用户（others）。每组包含读（r）、写（w）、执行（x）三种权限。

```bash
# 权限表示示例
-rwxr-xr-x  # 所有者可读写执行，组可读执行，其他可读执行 (755)
-rw-r--r--  # 所有人可读，所有者可写 (644)
```

### 4.2 chmod - 改变权限

```bash
chmod 755 file           # 数字方式设置权限
chmod u+x file           # 给所有者添加执行权限
chmod g-w file           # 给组移除写权限
chmod a+r file           # 给所有人添加读权限
chmod -R 755 dir         # 递归设置目录权限
```

### 4.3 chown - 改变所有者

```bash
chown user file              # 改变文件所有者
chown user:group file        # 改变所有者和组
chown -R user:group dir      # 递归改变目录
```

### 4.4 用户管理

```bash
useradd newuser      # 创建新用户
passwd username      # 设置或修改用户密码
su - username        # 切换到指定用户
exit                # 退出当前用户
whoami              # 显示当前用户名
sudo command        # 以超级用户权限执行命令
```

---

## 5. 进程管理

### 5.1 查看进程

```bash
ps              # 查看当前终端的进程
ps aux          # 查看所有进程的详细信息
ps -ef          # 查看所有进程（BSD 风格）
top             # 动态查看进程（按 CPU 使用率排序）
htop            # 更友好的进程查看工具（需安装）
```

### 5.2 终止进程

```bash
kill PID             # 正常终止进程
kill -9 PID          # 强制终止进程
kill -l              # 查看所有信号
pkill process_name   # 按名称终止进程
```

### 5.3 后台与前台

```bash
command &           # 直接后台运行
Ctrl+Z              # 挂起当前进程
bg                  # 将挂起的进程放到后台继续运行
fg                  # 将后台进程切换到前台
jobs                # 查看后台任务列表
```

### 5.4 nohup - 脱离终端运行

```bash
nohup command &           # 忽略挂断信号，后台运行
nohup command > output.log 2>&1 &   # 重定向输出到日志文件
```

---

## 6. 网络命令

### 6.1 网络配置

```bash
ping -c 4 host              # 测试网络连通性（发送 4 个包）
ifconfig                    # 查看网络接口信息（旧命令）
ip addr show                # 查看网络接口信息（新命令）
ip link set eth0 up         # 启用网络接口
```

### 6.2 端口与连接

```bash
netstat -tuln              # 查看监听端口
ss -tuln                   # 查看监听端口（更现代）
netstat -an                # 查看所有连接
ss -s                      # 查看连接统计
```

### 6.3 远程操作

```bash
ssh user@host              # 远程连接
ssh -p 2222 user@host       # 指定端口连接
scp file user@host:/path   # 远程复制文件
scp -r dir user@host:/path  # 递归复制目录
```

### 6.4 下载

```bash
wget URL                   # 下载文件
wget -c URL                # 断点续传
curl URL                   # 获取网页内容
curl -O URL                # 下载文件
curl -X POST -d "data" URL # 发送 POST 请求
```

---

## 7. 管道与重定向

### 7.1 管道 `|`

管道将前一个命令的输出作为后一个命令的输入：

```bash
ls -la | grep ".txt"           # 列出文件并筛选 txt 文件
cat file | sort | uniq         # 排序并去重
ps aux | grep python           # 查找 python 进程
df -h | grep sda               # 查看特定磁盘使用情况
```

### 7.2 输出重定向

```bash
command > file          # 重定向输出到文件（覆盖）
command >> file         # 追加重定向
command 2> file         # 重定向错误输出
command &> file         # 重定向所有输出
command > file 2>&1     # 重定向标准输出和错误输出
```

### 7.3 输入重定向

```bash
command < file          # 从文件读取输入
command << EOF
  multi-line input
EOF
```

---

## 8. 常用快捷键

| 快捷键 | 功能 |
|--------|------|
| `Ctrl + C` | 取消当前命令 |
| `Ctrl + Z` | 挂起当前进程 |
| `Ctrl + D` | 退出终端/EOF |
| `Ctrl + L` | 清屏 |
| `Ctrl + A` | 光标移到行首 |
| `Ctrl + E` | 光标移到行尾 |
| `Ctrl + U` | 删除光标前的内容 |
| `Ctrl + K` | 删除光标后的内容 |
| `Tab` | 自动补全 |
| `↑` / `↓` | 命令历史上下翻页 |

---

## 参考资料

[^1]: [Linux 官方文档](https://www.kernel.org/doc/html/latest/)
[^2]: [鸟哥的 Linux 私房菜](http://linux.vbird.org/)
[^3]: [Linux Command Line Basics - Linux Foundation](https://www.linuxfoundation.org/)

[^1]: 本文档为 Linux 基础入门指南，适合初学者快速上手。
