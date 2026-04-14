---
title: Git 基础教程
date: 2026-04-06
description: Git 分布式版本控制系统入门指南
tags:
  - git
  - 版本控制
  - 工具
draft: false
permalink:
---

## Git 简介

### 版本控制的概念

版本控制是一种记录文件变化历史的系统，使得开发者能够追踪文件的修改、协同工作，并在需要时回滚到之前的版本。[^1]

常见的版本控制系统包括：
- **本地版本控制**：如 RCS，在本地记录文件差异
- **集中式版本控制**：如 SVN，所有版本存储在中央服务器
- **分布式版本控制**：如 Git，每个客户端都有完整的版本库副本

### Git 的诞生背景

Git 由 Linux 内核开发者 Linus Torvalds 于 2005 年创建，用于替代 BitKeeper 管理 Linux 内核源代码的开发。[^2]

Git 的设计目标：
- 速度
- 简单的设计
- 对非线性开发模式的强力支持
- 完全分布式
- 高效处理大型项目

### 与 SVN 的区别

| 特性 | Git | SVN |
|------|-----|-----|
| 类型 | 分布式 | 集中式 |
| 克隆 | 完整仓库副本 | 仅最新版本 |
| 分支 | 轻量级，快速 | 较重，操作复杂 |
| 网络依赖 | 可离线工作 | 必须连接服务器 |
| 存储方式 | 快照流 | 差异存储 |

## 基础配置

在使用 Git 之前，需要进行基础配置：

```bash
# 设置用户信息
git config --global user.name "Your Name"
git config --global user.email "your@email.com"

# 设置默认编辑器
git config --global core.editor vim

# 设置默认分支名
git config --global init.defaultBranch main

# 查看所有配置
git config --list

# 查看特定配置
git config user.name
```

常用编辑器选项：
- `vim` - 命令行编辑器
- `code --wait` - VS Code
- `nano` - 简单文本编辑器

## 基础操作

### 初始化仓库

```bash
# 在当前目录初始化新仓库
git init

# 在当前目录初始化新仓库，指定分支名
git init -b main

# 初始化裸仓库（用于服务器）
git init --bare
```

### 克隆仓库

```bash
# 克隆远程仓库
git clone https://github.com/user/repo.git

# 克隆到指定目录
git clone https://github.com/user/repo.git my-folder

# 克隆指定分支
git clone --branch develop https://github.com/user/repo.git

# 浅克隆（只包含最近的历史）
git clone --depth 1 https://github.com/user/repo.git
```

### 暂存文件

```bash
# 暂存特定文件
git add file.txt

# 暂存所有更改
git add .

# 暂存所有更改（简洁写法）
git add -A

# 暂存所有已跟踪文件的更改（不包括新文件）
git add -u

# 交互式暂存
git add -i
```

### 提交

```bash
# 提交暂存区的文件
git commit -m "提交信息"

# 提交所有已跟踪文件的更改
git commit -am "提交信息"

# 修改最后一次提交
git commit --amend

# 提交但不暂存（跳过 git add）
git commit -a -m "提交信息"
```

### 查看状态

```bash
# 查看工作区状态
git status

# 简洁模式
git status -s

# 显示分支信息
git status -b
```

状态符号说明：
- `??` - 未跟踪的新文件
- `A` - 已暂存的新文件
- `M` - 已修改的文件
- `D` - 已删除的文件

### 查看提交历史

```bash
# 查看完整提交历史
git log

# 单行显示
git log --oneline

# 显示最近 N 次提交
git log -n 5

# 显示分支合并图
git log --graph --oneline --all

# 显示作者和时间信息
git log --format="%h %an %ar %s"

# 查看特定文件的提交历史
git log --follow file.txt
```

## 分支管理

### 查看和创建分支

```bash
# 查看本地分支
git branch

# 查看所有分支（包括远程）
git branch -a

# 查看已合并到当前分支的分支
git branch --merged

# 查看未合并到当前分支的分支
git branch --no-merged

# 创建新分支
git branch feature-branch

# 创建新分支并切换
git checkout -b feature-branch

# 创建新分支（现代写法）
git switch -c feature-branch
```

### 切换分支

```bash
# 切换到已有分支
git checkout main

# 切换到最近分支
git checkout -

# 创建并切换（现代写法）
git switch -c new-branch

# 切换到已有分支（现代写法）
git switch main
```

### 合并分支

```bash
# 将指定分支合并到当前分支
git merge feature-branch

# 合并并创建一个合并提交
git merge --no-ff feature-branch

# 取消合并
git merge --abort
```

### 变基

变基是将一系列提交「复制」到另一个位置的过程：

```bash
# 变基当前分支到目标分支
git rebase main

# 交互式变基（修改提交历史）
git rebase -i HEAD~3

# 继续变基（解决冲突后）
git rebase --continue

# 取消变基
git rebase --abort
```

**变基黄金法则**：不要对已推送到远程的提交进行变基。

### 冲突解决

合并或变基时可能产生冲突：

```bash
# 1. 查看冲突文件
git status

# 2. 手动编辑冲突文件，解决冲突
# 冲突标记：
# <<<<<<< HEAD
# 当前分支的内容
# =======
# 被合并分支的内容
# >>>>>>> branch-name

# 3. 暂存解决后的文件
git add file.txt

# 4. 完成合并/变基
git commit    # 用于合并
git rebase --continue    # 用于变基
```

使用工具解决冲突：

```bash
# 使用合并工具
git mergetool

# 使用 VS Code
code --wait file.txt
```

## 远程协作

### 远程仓库

```bash
# 查看远程仓库
git remote

# 查看远程仓库详细信息
git remote -v

# 添加远程仓库
git remote add origin https://github.com/user/repo.git

# 重命名远程仓库
git remote rename origin upstream

# 删除远程仓库
git remote remove origin

# 修改远程仓库 URL
git remote set-url origin https://github.com/user/repo.git
```

### 推送

```bash
# 推送到远程仓库
git push

# 推送到指定分支
git push origin main

# 推送并设置上游分支
git push -u origin feature-branch

# 强制推送（谨慎使用）
git push --force

# 推送所有分支
git push --all
```

### 拉取并合并

```bash
# 拉取并合并远程分支
git pull

# 拉取并变基（如果分支是基于远程分支）
git pull --rebase

# 拉取指定分支
git pull origin main
```

### 仅拉取

```bash
# 仅拉取远程分支，不自动合并
git fetch

# 拉取特定远程分支
git fetch origin main

# 拉取所有远程分支
git fetch --all
```

## 高级操作

### 暂存工作现场

当需要切换分支但不想提交当前工作时，可以使用 stash：

```bash
# 暂存当前工作现场
git stash

# 暂存并添加说明
git stash save "工作说明"

# 查看暂存列表
git stash list

# 应用最近的暂存
git stash apply

# 应用最近的暂存并删除
git stash pop

# 应用特定暂存
git stash apply stash@{2}

# 删除暂存
git stash drop stash@{0}

# 清空所有暂存
git stash clear
```

### 重置 HEAD

```bash
# 查看提交历史，找到需要重置到的 commit
git log --oneline

# 软重置（保留更改在暂存区）
git reset --soft HEAD~1

# 混合重置（保留更改在工作区）
git reset --mixed HEAD~1

# 硬重置（丢弃所有更改，危险！）
git reset --hard HEAD~1

# 重置到特定提交
git reset --hard abc1234
```

### 撤销提交

`git revert` 创建一个新的提交来撤销之前的提交：

```bash
# 撤销最后一次提交
git revert HEAD

# 撤销特定提交
git revert abc1234

# 不自动提交
git revert --no-commit abc1234
```

**区别**：
- `reset` - 回退历史记录，丢弃提交
- `revert` - 创建新提交，保留完整历史

### 选择性应用提交

```bash
# 应用特定提交到当前分支
git cherry-pick abc1234

# 应用多个提交
git cherry-pick abc1234 def5678

# 继续 cherry-pick（解决冲突后）
git cherry-pick --continue

# 取消 cherry-pick
git cherry-pick --abort

# 不要自动提交
git cherry-pick --no-commit abc1234
```

## Git 工作流

### Git Flow

Git Flow 是一种成熟的分支模型，适合有固定发布周期的项目。[^3]

分支类型：
- `main` - 生产环境代码
- `develop` - 开发主分支
- `feature/*` - 功能分支
- `release/*` - 发布分支
- `hotfix/*` - 热修复分支

```
main:     ●────────────────●────────────────●
             \                \              \
develop:    ●─●─●─●─●─●─●─●─●─●─●─●─●─●─●─●─●─●
               \              / \            /
feature:       ●─●─●─●─●─●   /   \──●─●─●─●─●
                              /
release:                    ●─●─●─●─●─●─●─●─●
```

常用命令：

```bash
# 开始新功能
git checkout develop
git checkout -b feature/new-feature

# 完成功能
git checkout develop
git merge --no-ff feature/new-feature

# 开始发布
git checkout develop
git checkout -b release/1.0.0
# ... 修复发布前问题
git checkout main
git merge release/1.0.0
git tag -a v1.0.0 -m "Version 1.0.0"
git checkout develop
git merge release/1.0.0

# 开始热修复
git checkout main
git checkout -b hotfix/fix-bug
# ... 修复问题
git checkout main
git merge hotfix/fix-bug
git tag -a v1.0.1 -m "Hotfix 1.0.1"
git checkout develop
git merge hotfix/fix-bug
```

### GitHub Flow

GitHub Flow 是一种更简单的流程，适合持续部署的项目。

原则：
1. `main` 分支始终可部署
2. 所有工作在新分支进行
3. 定期 push 到远程
4. 打开 Pull Request
5. 合并后立即部署

流程：

```bash
# 1. 从 main 创建新分支
git checkout main
git checkout -b feature/new-feature

# 2. 开发并提交
git add .
git commit -m "Add new feature"

# 3. Push 到远程
git push -u origin feature/new-feature

# 4. 在 GitHub 打开 Pull Request
# 5. 代码审查
# 6. 合并到 main
# 7. 删除分支
git branch -d feature/new-feature
```

### Trunk-Based Development

Trunk-Based Development（TBD）是一种所有开发者直接在主干分支工作的模型。

特点：
- 所有人都在同一个分支工作
- 频繁集成到主干
- 使用特性开关（Feature Flags）控制新功能
- 短生命周期特性分支或无分支

```bash
# 直接在 main 分支开发
git checkout main
git pull origin main

# 短期特性分支
git checkout -b feature/new-feature
# ... 小步提交
git checkout main
git merge --no-ff feature/new-feature
```

## 参考资料

[^1]: 版本控制系统 - 维基百科：https://zh.wikipedia.org/wiki/版本控制
[^2]: Git 官方文档：https://git-scm.com/book/zh/v2
[^3]: A successful Git branching model：https://nvie.com/posts/a-successful-git-branching-model
