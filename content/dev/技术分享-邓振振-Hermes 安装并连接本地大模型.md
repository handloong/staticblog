---
title: Hermes 安装并连接本地大模型
date: 2026-05-19
description: 从安装 Hermes Agent 开始，配置连接到已部署的本地大模型服务
toc: true
slug: hermes-install-local-llm
categories:
    - Dev
    - 技术分享
tags:
    - hermes-agent
    - 本地大模型
---

## 前置条件

本地大模型服务已部署完成，提供 OpenAI 兼容的 API 接口。常见部署方式：

- Ollama（默认地址 `http://localhost:11434/v1`）
- LM Studio（默认地址 `http://localhost:1234/v1`）
- vLLM、Ollama Web UI 等自定义部署

## 安装 Hermes Agent

### 1. 下载安装脚本

> [!WARNING]
> 警告提示：
> 1. 如果使用官方一键命令可能因为编码问题报错，参考连接：https://github.com/NousResearch/hermes-agent/pull/27419
> 2. 因为要 git clone 所以全程开梯子.

在 PowerShell 中运行以下命令：

```powershell
$u = 'https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.ps1'
$p = Join-Path $env:TEMP 'hermes-install.ps1'
Invoke-WebRequest -Uri $u -OutFile $p
powershell.exe -ExecutionPolicy Bypass -File $p
```

## 配置 Hermes

### 方法一：使用快速设置

安装完成后，可以使用快速设置功能配置 Hermes。

### 方法二：手动配置

打开 **CCSwitch** 配置 Hermes 相关的 API 信息。

- 配置 API 地址
- 设置认证令牌
- 指定默认模型

## 使用 Hermes Gateway

### 安装 Gateway 通道

```shell
hermes gateway setup
```

### 启动 Gateway

配置完成后，通过以下命令启动 Gateway：

```shell
hermes gateway start
```

### 查看状态

可以通过以下命令查看相关状态和配置：

```shell
hermes curator status
```

## 使用建议

- 首次使用建议先通过快速设置完成基础配置
- 后续可以根据需要调整模型参数
- Gateway 保持运行状态以确保 Hermes 正常工作

## 参考资料

- [Hermes Agent 官方文档](https://github.com/NousResearch/hermes-agent)
- [CCSwitch 项目](https://github.com/farion1231/cc-switch)
