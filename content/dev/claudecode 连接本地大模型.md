---
title: Claude Code 连接本地大模型
date: 2026-05-18
description: 从安装 Claude Code 开始，配置连接到已部署的本地大模型服务
toc: true
slug: claudecode-connect-local-llm
categories:
    - Dev
    - 技术分享
tags:
    - claude-code
    - 本地大模型
---

## 系统要求

- 操作系统：macOS 10.15+、Windows 10/11、Linux、WSL
- Node.js：18.0 或更高版本

## 前置条件

本地大模型服务已部署完成，提供 OpenAI 兼容的 API 接口。常见部署方式：
- Ollama（默认地址 `http://localhost:11434/v1`）
- LM Studio（默认地址 `http://localhost:1234/v1`）
- vLLM、Ollama Web UI 等自定义部署

## 安装 Claude Code

```shell
npm install -g @anthropic-ai/claude-code
```

国内网络如果 npm 官方源较慢，可以临时使用镜像安装，安装后再切回官方源：

```shell
npm config set registry https://registry.npmmirror.com
npm install -g @anthropic-ai/claude-code
npm config set registry https://registry.npmjs.org
```

验证安装：

```shell
claude --version
```

输出如下:
```shell
2.1.143 (Claude Code)
```

## 配置连接本地大模型

Claude Code 通过 `~/.claude/` 目录下的配置文件进行设置。

> [!IMPORTANT]
> 如下json已经跳过Claude Code登录问题,推荐复制然后改参数


### 1. 创建 config.json

文件路径：`C:\Users\用户\.claude\config.json`（Windows）或 `~/.claude/config.json`（macOS/Linux）

```json
{
  "primaryApiKey": "any"
}
```

### 2. 创建 settings.json

文件路径：`C:\Users\用户\.claude\settings.json`（Windows）或 `~/.claude/settings.json`（macOS/Linux）

```json
{
  "effortLevel": "xhigh",
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "你的key",
    "ANTHROPIC_BASE_URL": "http://localhost:11434",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "Qwen3.6-27B",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL_NAME": "Qwen3.6-27B",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "Qwen3.6-27B",
    "ANTHROPIC_DEFAULT_OPUS_MODEL_NAME": "Qwen3.6-27B",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "Qwen3.6-27B",
    "ANTHROPIC_DEFAULT_SONNET_MODEL_NAME": "Qwen3.6-27B",
    "ANTHROPIC_MODEL": "Qwen3.6-27B",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  },
  "includeCoAuthoredBy": false,
  "model": "haiku",
  "primaryApiKey": "ok"
}
```

### 配置字段说明

| 字段 | 说明 |
| --- | --- |
| `ANTHROPIC_BASE_URL` | 本地大模型服务的 API 地址，注意不要加V1 |
| `ANTHROPIC_AUTH_TOKEN` | 本地服务的认证令牌，根据实际配置修改 |
| `ANTHROPIC_DEFAULT_*_MODEL` | 将 Haiku / Sonnet / Opus 三个档位都指向同一个本地模型 |
| `ANTHROPIC_MODEL` | 默认使用的模型名称 |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | 关闭非必要网络请求，完全本地运行 |
| `model` | 默认使用的模型档位，`haiku` / `sonnet` / `opus` |
| `effortLevel` | 模型推理努力程度，`xhigh` 为最高 |
| `primaryApiKey` | 本地服务通常不校验，设为任意非空字符串即可 |

## 启动 Claude Code

```shell
claude
```

启动后进入交互模式，即可使用本地模型进行对话和代码操作。

## 注意事项

- 修改 `ANTHROPIC_BASE_URL` 为实际本地服务的地址
- `ANTHROPIC_AUTH_TOKEN` 根据本地服务的认证要求配置，若无校验可保留默认值
- 三个模型档位指向同一个模型时，使用 `/model` 命令切换档位不会改变实际模型，仅影响推理策略
- 本地模型能力有限，适合日常辅助开发，复杂任务效果不如云端大模型

## Claude Code for VS Code

除了命令行模式，Claude Code 也提供了 VS Code 插件，可以在编辑器内直接使用。

### 安装

1. 打开 VS Code 扩展市场（`Ctrl+Shift+X`）
2. 搜索 `Claude Code`
3. 点击安装并启用

### 使用

安装完成后，通过 `Ctrl+Shift+P` 打开命令面板，输入 `Claude Code` 即可启动交互面板。

在编辑器中你可以：
- 选中代码后按 `Ctrl+Shift+P` 选择 Claude Code 相关命令，对选中代码进行解释或修改
- 在侧边栏对话框中输入指令，让 Claude Code 阅读项目文件、执行任务
- 使用 `/` 快捷命令，如 `/fast` 切换快速模式

插件复用 `~/.claude/settings.json` 的配置，无需额外设置即可完成本地模型连接。


## （可选）使用 cc switch 快速切换模型

在日常开发中，你可能需要在本地大模型和云端模型（如 DeepSeek-v4）之间切换。手动修改 `settings.json` 比较繁琐，可以使用 `CC switch` 工具来一键完成切换。

`cc switch` 的本质是修改 `~/.claude/settings.json` 文件中的 `env` 字段，替换 `ANTHROPIC_BASE_URL`、`ANTHROPIC_AUTH_TOKEN`、`ANTHROPIC_MODEL` 等配置项。

你可以根据需要自定义配置模板，将多个常用模型写入预设，无需每次手动编辑配置文件。


## 参考资料

- [Claude Code 官方文档](https://docs.anthropic.com/en/docs/claude-code/overview)
- [Ollama 官方文档](https://github.com/ollama/ollama)

- [Claude Code登录问题](https://github.com/farion1231/cc-switch/issues/404)