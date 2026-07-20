---
title: "如何使用 Codex：OpenAI 的自主编码代理"
slug: "ru-he-shi-yong-codex"
date: 2026-05-29
draft: false
description: "全面介绍 OpenAI Codex CLI 的使用方法，从基础设置到高级技巧，帮你提高开发效率"
tags: ["codex", "ai", "coding", "automation", "openai"]
categories: ["Dev", "技术分享", "开发工具", "AI 编程"]
---

## 什么是 Codex？

[Codex](https://github.com/openai/codex) 是 OpenAI 推出的自主编码代理 CLI 工具，它能够理解自然语言指令并自动完成编程任务。无论是构建新功能、重构代码、审查 PR，还是批量修复问题，Codex 都能成为你的智能编程助手。

## 前置要求

### 1. 安装 Codex

```bash
npm install -g @openai/codex
```

### 2. 认证配置

有两种认证方式：

- **API Key 方式**：设置 `OPENAI_API_KEY` 环境变量
- **OAuth 方式**：运行 `codex login` 进行 OAuth 认证（推荐）

### 3. 必须在 Git 仓库中运行

Codex 要求运行在一个 Git 仓库内，这是强制性的。如果你想在临时目录中运行：

```bash
cd $(mktemp -d) && git init && codex exec '你的任务'
```

## 基本使用

### 一次性任务

最简单的方式是使用 `exec` 命令：

```bash
codex exec '为设置页面添加深色模式切换功能'
```

Codex 会理解你的需求，自动编写代码、修改文件，并在完成后退出。

### 使用 --full-auto 标志

`--full-auto` 标志允许 Codex 在沙箱中自动批准对工作区的文件修改，这对于构建任务非常有用：

```bash
codex exec --full-auto '重构认证模块'
```

## 后台模式（长时间任务）

对于可能需要较长时间完成的任务，可以使用后台模式：

```bash
# 在后台启动任务
codex exec --full-auto '重构认证模块' \
  --workdir /path/to/project \
  --background

# 监控进度
process(action="poll", session_id="<id>")
process(action="log", session_id="<id>")

# 如果 Codex 提问，发送输入
process(action="submit", session_id="<id>", data="yes")

# 如果需要可以终止
process(action="kill", session_id="<id>")
```

## 高级技巧

### 1. 使用 worktree 批量修复问题

如果你有多个问题需要修复，可以使用 Git worktree：

```bash
# 创建 worktree
git worktree add -b fix/issue-78 /tmp/issue-78 main
git worktree add -b fix/issue-99 /tmp/issue-99 main

# 在各自的 worktree 中启动 Codex
codex --yolo exec '修复问题 #78' \
  --workdir /tmp/issue-78 \
  --background

codex --yolo exec '修复问题 #99' \
  --workdir /tmp/issue-99 \
  --background

# 完成后推送
cd /tmp/issue-78
git push -u origin fix/issue-78

# 创建 PR
gh pr create --repo user/repo \
  --head fix/issue-78 \
  --title 'fix: 修复 #78' \
  --body '...'

# 清理 worktree
git worktree remove /tmp/issue-78
```

### 2. PR 审查

你可以让 Codex 帮助审查 Pull Request：

```bash
# 克隆到临时目录进行安全审查
REVIEW=$(mktemp -d)
git clone https://github.com/user/repo.git $REVIEW
cd $REVIEW
gh pr checkout 42
codex review --base origin/main
```

### 3. 批量 PR 审查

```bash
# 获取所有 PR 的 refs
git fetch origin '+refs/pull/*/head:refs/remotes/origin/pr/*'

# 并行审查多个 PR
codex exec '审查 PR #86' \
  --workdir /path/to/project \
  --background

codex exec '审查 PR #87' \
  --workdir /path/to/project \
  --background

# 发布评论
gh pr comment 86 --body '<审查结果>'
```

## 关键标志说明

| 标志 | 效果 |
|------|------|
| `exec "提示"` | 一次性执行，完成后退出 |
| `--full-auto` | 沙箱中自动批准工作区内的文件修改 |
| `--yolo` | 无沙箱，无需批准（最快，但最危险） |
| `--background` | 后台运行 |
| `--workdir` | 指定工作目录 |

## 最佳实践

### 1. 始终使用 PTY

Codex 是一个交互式终端应用，必须使用 PTY（伪终端）：

```bash
# 正确的方式
terminal(command="codex exec '你的任务'", pty=true)

# 错误的方式（会挂起）
terminal(command="codex exec '你的任务'", pty=false)
```

### 2. 确保在 Git 仓库中

Codex 不在 Git 仓库中会拒绝运行。为安全起见，始终确保：

```bash
git status  # 确认当前在 Git 仓库中
```

### 3. 使用 `exec` 进行一次性任务

```bash
# 推荐
codex exec "提示"

# 避免
codex interactive  # 会保持交互状态
```

### 4. 耐心等待长时间任务

对于复杂的任务，Codex 可能需要较长时间。使用后台模式并耐心监控日志：

```bash
process(action="log", session_id="<id>")
```

### 5. 批量任务使用并行

多个 Codex 进程可以同时运行，特别适合批量工作：

```bash
# 同时运行多个任务
codex exec '任务 A' --background &
codex exec '任务 B' --background &
codex exec '任务 C' --background &
```

## 常见问题

### Q: Codex 说"必须在 Git 仓库中运行"怎么办？

A: 确保你在一个 Git 仓库目录中：

```bash
git init  # 如果目录没有初始化 Git
```

### Q: 如何监控后台任务的进度？

A: 使用 `process` 工具：

```bash
process(action="poll", session_id="<id>")  # 检查状态
process(action="log", session_id="<id>")   # 查看日志
```

### Q: `--yolo` 标志安全吗？

A: `--yolo` 禁用了沙箱和保护措施，Codex 可以直接修改你的文件系统。只在你可信的环境中使用时才应该使用此标志。

### Q: 如何撤销 Codex 的修改？

A: 使用 Git 可以方便地回滚：

```bash
git diff      # 查看修改
git checkout . # 撤销所有修改
```

## 总结

Codex 是一个强大的自主编码代理，可以显著提高工作效率。通过自然语言指令，它可以自动完成编程任务，从简单的代码修改到复杂的架构重构。

记住关键点：
- ✅ 使用 `pty=true` 运行
- ✅ 确保在 Git 仓库中
- ✅ 使用 `exec` 进行一次性任务
- ✅ 使用 `--full-auto` 进行构建
- ✅ 长时间任务使用后台模式
- ✅ 批量任务可以使用并行

现在就开始使用 Codex 提升你的开发效率吧！🚀

## 参考资料

- [Codex GitHub 仓库](https://github.com/openai/codex)
- [Hermes Agent 文档](https://hermes-agent.nousresearch.com/docs)
- [Codex CLI 文档](https://github.com/openai/codex/tree/main/packages/codex)
