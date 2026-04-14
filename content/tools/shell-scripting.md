---
title: Shell脚本指南
date: 2026-04-13
description: Shell脚本（bash）编程指南：基础语法、函数、错误处理、最佳实践
tags:
  - shell
  - bash
  - scripting
  - devops
draft: false
permalink:
---

## 简介

Shell脚本是Linux/Unix系统管理的核心工具，用于自动化重复性任务、批量处理和系统运维。本指南介绍Bash脚本的编写规范和最佳实践。[^1]

## 脚本开头

每个Bash脚本应以shebang开头：

```bash
#!/usr/bin/env bash
```

使用 `env` 而不是硬编码路径，使脚本更具可移植性。

## 严格模式

启用严格模式避免常见错误：

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

| 选项 | 作用 |
|------|------|
| `set -e` | 命令失败时立即退出 |
| `set -u` | 使用未定义变量时报错 |
| `set -o pipefail` | 管道中任一命令失败则整个管道失败 |

## 变量

### 基本使用

```bash
# 定义变量（等号两边不能有空格）
name="Alice"
age=25

# 使用变量（加引号防止单词分割）
echo "Name: $name"
echo "Age: ${age}"

# 字符串操作
${#var}          # 长度
${var:0:5}       # 子串
${var#pattern}   # 去掉开头匹配
${var##pattern}  # 贪婪匹配
${var%pattern}   # 去掉结尾匹配
${var%%pattern}  # 贪婪匹配
```

### 引号规则

```bash
# 双引号：解析变量和转义字符
echo "Hello, $name"        # Hello, Alice

# 单引号：原样输出
echo 'Hello, $name'        # Hello, $name

# 命令替换
now=$(date +%Y-%m-%d)
files=$(ls *.txt)          # 可能有空格问题
mapfile -t files < <(ls *.txt)  # 安全读取
```

## 条件判断

### 字符串比较

```bash
# 注意：使用 [[ ]] 而非 [ ]
if [[ "$name" == "Alice" ]]; then
    echo "Hi Alice"
elif [[ "$age" -lt 18 ]]; then
    echo "Minor"
else
    echo "Adult"
fi

# 字符串运算符
[[ -z "$str" ]]    # 空字符串
[[ -n "$str" ]]    # 非空
[[ "$a" == "$b" ]] # 相等
[[ "$a" != "$b" ]] # 不等
```

### 数值比较

```bash
[[ "$age" -eq 25 ]]   # 等于
[[ "$age" -ne 18 ]]   # 不等于
[[ "$age" -gt 18 ]]   # 大于
[[ "$age" -ge 18 ]]   # 大于等于
[[ "$age" -lt 30 ]]   # 小于
[[ "$age" -le 30 ]]   # 小于等于
```

### 文件测试

```bash
[[ -e "$file" ]]      # 存在
[[ -f "$file" ]]      # 普通文件
[[ -d "$dir" ]]       # 目录
[[ -r "$file" ]]      # 可读
[[ -w "$file" ]]      # 可写
[[ -x "$file" ]]      # 可执行
[[ "$f1" -nt "$f2" ]] # f1比f2新
```

## 循环

### for循环

```bash
# 列表循环
for item in apple banana cherry; do
    echo "Item: $item"
done

# C风格循环
for ((i = 0; i < 10; i++)); do
    echo "i = $i"
done

# 遍历文件
for f in *.txt; do
    echo "File: $f"
done

# 遍历目录
for f in /path/to/dir/*; do
    if [[ -f "$f" ]]; then
        echo "File: $f"
    fi
done
```

### while循环

```bash
# 条件循环
count=0
while [[ $count -lt 5 ]]; do
    echo "Count: $count"
    ((count++))
done

# 读取行
while IFS= read -r line; do
    echo "Line: $line"
done < file.txt

# 无限循环
while true; do
    echo "Running..."
    sleep 1
done
```

## 函数

### 基本定义

```bash
# 定义函数
greet() {
    local name="$1"  # 局部变量
    echo "Hello, $name!"
    return 0
}

# 调用函数
greet "Alice"

# 返回值（通过echo）
get_sum() {
    local a="$1"
    local b="$2"
    echo $((a + b))
}

result=$(get_sum 3 5)
echo "Sum: $result"
```

### 参数处理

```bash
process_args() {
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -h|--help)
                show_help
                return 0
                ;;
            -v|--verbose)
                verbose=true
                ;;
            -n|--name)
                name="$2"
                shift
                ;;
            *)
                echo "Unknown: $1"
                ;;
        esac
        shift
    done
}
```

## 错误处理

### trap捕获信号

```bash
cleanup() {
    local exit_code=$?
    echo "Cleaning up..."
    # 删除临时文件等
    rm -rf /tmp/my_script.*
    exit $exit_code
}

trap cleanup EXIT ERR INT TERM
```

### 退出码

```bash
# 自定义退出码
EXIT_OK=0
EXIT_USAGE=64
EXIT_DEPENDENCY=65
EXIT_RUNTIME=66

if [[ ! -f "$config_file" ]]; then
    echo "Error: config file not found" >&2
    exit $EXIT_USAGE
fi
```

## 数组和关联数组

### 普通数组

```bash
# 定义
fruits=("apple" "banana" "cherry")

# 访问
echo "${fruits[0]}"    # 第一个元素
echo "${fruits[@]}"   # 所有元素
echo "${#fruits[@]}"  # 数组长度

# 追加
fruits+=("date")

# 切片
echo "${fruits[@]:1:2}"  # 跳过第一个，取两个
```

### 关联数组（字典）

```bash
# 定义（需要声明）
declare -A user_info
user_info["name"]="Alice"
user_info["email"]="alice@example.com"

# 遍历
for key in "${!user_info[@]}"; do
    echo "$key: ${user_info[$key]}"
done
```

## 日志和输出

```bash
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" >&2
}

log "INFO" "Starting process"
log "ERROR" "Failed to connect"
log "WARN" "Retry in 5 seconds"

# 彩色输出
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m'  # No Color

echo -e "${RED}Error${NC}: Something went wrong"
echo -e "${GREEN}Success${NC}: Operation completed"
```

## 常用技巧

### 命令替换

```bash
# 获取命令输出
hostname=$(hostname)
files=$(ls -1)
count=$(wc -l < file.txt)

# 进程替换
diff <(sort a.txt) <(sort b.txt)
while read -r line; do
    echo "$line"
done < <(grep "pattern" file.txt)
```

### xargs并行

```bash
# 串行
cat files.txt | xargs process_file

# 并行（-P 指定并行数）
cat files.txt | xargs -P 4 -I {} process_file {}

# 查找并执行
find . -name "*.txt" -print0 | xargs -0 grep "pattern"
```

### 处理选项

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
    cat <<EOF
Usage: $0 [OPTIONS] <input>

Options:
    -h, --help      Show this help
    -v, --verbose   Enable verbose output
    -n, --name NAME Set name
    -o, --output    Output file

Example:
    $0 -v -n Alice input.txt
EOF
}

verbose=false
name=""
output=""

while [[ $# -gt 0 ]]; do
    case "$1" in
        -h|--help) usage; exit 0 ;;
        -v|--verbose) verbose=true; shift ;;
        -n|--name) name="$2"; shift 2 ;;
        -o|--output) output="$2"; shift 2 ;;
        -*) echo "Unknown option: $1"; exit 1 ;;
        *) break ;;
    esac
done
```

## ShellCheck静态检查

使用ShellCheck检测脚本问题：

```bash
# 安装
sudo apt install shellcheck  # Ubuntu
brew install shellcheck      # macOS

# 检查
shellcheck myscript.sh

# 忽略特定警告
# shellcheck disable=SC2086
```

常见警告：
- SC2086: Double quote to prevent globbing
- SC2010: Don't use ls | grep
- SC2068: Double quote array expansion

## 脚本模板

```bash
#!/usr/bin/env bash
# script.sh - 脚本描述
#
# Usage: script.sh [OPTIONS] <input>
#
# Options:
#   -h, --help      Show this help
#   -v, --verbose   Enable verbose output
#
set -euo pipefail
IFS=$'\n\t'

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" >&2
}

die() {
    log "FATAL: $*"
    exit 1
}

cleanup() {
    local exit_code=$?
    # Add cleanup logic here
    exit "$exit_code"
}
trap cleanup EXIT

check_dependencies() {
    local deps=("$@")
    for dep in "${deps[@]}"; do
        if ! command -v "$dep" &>/dev/null; then
            die "Missing dependency: $dep"
        fi
    done
}

main() {
    check_dependencies curl jq
    
    if [[ $# -lt 1 ]]; then
        echo "Usage: $0 <input>"
        exit 1
    fi
    
    log "Starting $(basename "$0")"
    
    # Your logic here
    
    log "Done"
}

main "$@"
```

## 参考资料

[^1]: Bash脚本最佳实践. https://www.shellcheck.net/

