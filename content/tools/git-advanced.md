---
title: Git 进阶技巧
date: 2026-04-13
description: Git 高级操作：Rebase、Submodule、Hooks、Reflog、等
tags:
  - tools
  - git
  - version-control
draft: false
permalink:
---

## 概述

本文介绍 Git 的高级操作技巧，适合已经熟悉基础命令的开发者。

## Rebase（变基）

### 基本概念

Rebase 将提交"重写"到另一个分支的基础上，产生线性历史。

```bash
# 当前在 feature 分支，想要基于 main 分支的最新提交
git rebase main

# 等价于：
git checkout main
git pull
git checkout feature
git rebase main
```

### Rebase vs Merge

```
Merge:
  main:    A---B---C
                \
  feature:       D---E

结果:
  main:    A---B---C---M
                \     /
                 D---E

Rebase:
  main:    A---B---C
                \
  feature:       D'--E'

结果（变基后）:
  main:    A---B---C
                  \
  feature:         D'--E'
```

### 交互式 Rebase

```bash
# 修改最近 3 个提交
git rebase -i HEAD~3

# 交互式界面:
pick abc1234 feat: 添加用户注册功能
pick def5678 fix: 修复登录bug
pick ghi9012 feat: 添加邮箱验证

# 变基操作:
# p, pick = 使用提交
# r, reword = 修改提交信息
# e, edit = 暂停进行修改
# s, squash = 与前一个合并
# f, fixup = 与前一个合并（丢弃提交信息）
# d, drop = 删除提交
```

### Rebase 黄金法则

> **不要对已经推送到远程的提交进行 Rebase！**

如果其他人基于你的旧提交工作，Rebase 会导致历史分叉和冲突。

## Submodule（子模块）

### 添加子模块

```bash
# 添加一个仓库作为子模块
git submodule add https://github.com/example/lib.git libs/lib

# 指定分支
git submodule add -b main https://github.com/example/lib.git libs/lib
```

### 克隆包含子模块的仓库

```bash
# 方法1：递归克隆
git clone --recursive https://github.com/example/project.git

# 方法2：先克隆再初始化
git clone https://github.com/example/project.git
git submodule init
git submodule update
```

### 更新子模块

```bash
# 进入子模块目录手动更新
cd libs/lib
git checkout main
git pull

# 回到主仓库提交子模块的变更
cd ../..
git add libs/lib
git commit -m "update lib submodule"
```

### 子模块的高级操作

```bash
# 从主仓库拉取所有子模块更新
git submodule update --remote --merge

# 在父仓库中查看所有子模块状态
git submodule status

# 在父仓库中执行子模块命令
git submodule foreach 'git status'
```

## Git Hooks

### 客户端钩子

```bash
# 安装钩子脚本
mv my-hook.sh .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit
```

### 常用钩子

#### pre-commit

```bash
#!/bin/bash
# .git/hooks/pre-commit

# 检查代码格式（以 ESLint 为例）
if git diff --cached --name-only | grep -E '\.js$' > /dev/null; then
    npx eslint --quiet $(git diff --cached --name-only) 
    if [ $? -ne 0 ]; then
        echo "ESLint check failed"
        exit 1
    fi
fi
```

#### commit-msg

```bash
#!/bin/bash
# .git/hooks/commit-msg

# 检查提交信息格式
commit_msg=$(cat "$1")
pattern="^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .{1,50}"

if ! [[ $commit_msg =~ $pattern ]]; then
    echo "Invalid commit message format"
    echo "Format: type(scope): description"
    echo "Types: feat, fix, docs, style, refactor, test, chore"
    exit 1
fi
```

#### post-commit

```bash
#!/bin/bash
# .git/hooks/post-commit

# 自动推送子模块
git submodule foreach 'git push'
```

### 服务器端钩子（.git/hooks/server/*）

```bash
# pre-receive - 在接收推送前执行
# post-receive - 在推送完成后执行
```

## Reflog（引用日志）

Reflog 记录所有 HEAD 和分支的移动历史，可用于恢复"丢失"的提交。

```bash
# 查看所有操作历史
git reflog

# 输出示例:
# abc1234 HEAD@{0}: commit: feat: 添加新功能
# def5678 HEAD@{1}: rebase: onto main
# ghi9012 HEAD@{2}: checkout: moving to feature
```

### 恢复误删的分支

```bash
# 找到分支最后所在的提交
git reflog

# 假设找到 HEAD@{2} 是分支最后的提交
git checkout -b recovered-branch HEAD@{2}
```

### 恢复 Rebase 前的状态

```bash
# Rebase 后发现问题
git reflog
# abc1234 HEAD@{0}: rebase -i (finish): returning to refs/heads/feature
# def5678 HEAD@{1}: rebase -i (finish): ...

# 恢复到 Rebase 前
git reset --hard HEAD@{1}
```

## Stash（暂存）

### 基础用法

```bash
# 暂存当前修改
git stash

# 命名 stash
git stash save "WIP: 正在进行的功能"

# 查看 stash 列表
git stash list
# stash@{0}: On feature: WIP: 正在进行的功能
# stash@{1}: On main: WIP: 旧的工作
```

### 高级用法

```bash
# 包含未跟踪文件
git stash -u

# 包含忽略的文件
git stash -a

# 从 stash 创建分支
git stash branch new-branch-name

# 查看 stash 内容
git stash show
git stash show -p  # 完整 diff
```

### 交互式 Stash

```bash
# 选择性 stash
git stash -p

# 选择操作:
# y - stash this hunk
# n - skip this hunk
# s - split into smaller hunks
# ? - help
```

## Cherry-Pick

选择性地应用某个提交。

```bash
# 应用指定提交
git cherry-pick abc1234

# 应用多个提交
git cherry-pick abc1234 def5678

# 继续 cherry-pick（解决冲突后）
git cherry-pick --continue

# 取消 cherry-pick
git cherry-pick --abort

# 摘取但不提交
git cherry-pick -n abc1234
```

## Bisect（二分查找）

自动定位引入 bug 的提交。

```bash
# 开始二分查找
git bisect start

# 标记当前版本为坏
git bisect bad

# 标记已知好的版本
git bisect good v1.0.0

# Git 会自动 checkout 中间版本测试
# 完成后标记结果
git bisect good  # 或 git bisect bad

# 完成后返回
git bisect reset
```

### 自动二分查找

```bash
# 提供测试脚本
git bisect start
git bisect bad HEAD
git bisect good v1.0.0
git bisect run npm test
```

## Worktree（工作树）

同时在多个分支上工作。

```bash
# 创建新的工作树
git worktree add ../feature-fix feature

# 查看所有工作树
git worktree list

# 移除工作树
git worktree remove ../feature-fix
```

## 配置技巧

### Aliases

```bash
# 简化常用命令
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --decorate"

# 高级别名
git config --global alias.unstage "reset HEAD --"
git config --global alias.last "log -1 HEAD"
```

### 智能大小写敏感

```bash
# macOS/Linux 文件系统大小写
git config --global core.ignorecase false
```

### 推送配置

```bash
# 推送时设置上游分支
git config --global push.default current

# 推送匹配所有分支
git config --global push.follow-tags true
```

## 参考

- [Git Pro Book](https://git-scm.com/book/en/v2)
- [Git Internals](https://git-scm.com/book/en/v2/Git-Internals)
