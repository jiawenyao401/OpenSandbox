---
title: 示例
description: OpenSandbox 的真实使用示例，涵盖编码智能体、浏览器自动化、桌面环境和训练工作负载。
---

# 示例

OpenSandbox 提供了开箱即用的示例，涵盖 SDK 用法、智能体集成、浏览器自动化以及训练工作负载。

::: tip
所有示例源代码均可在 GitHub 上的 [`examples/`](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples) 目录中获取。
:::

## 编码智能体

在隔离的沙箱中运行编码 CLI 和 AI 智能体框架。

| 示例 | 说明 |
|---------|-------------|
| [Claude Code](/examples/claude-code) | 在沙箱中运行 Claude Code CLI |
| [Gemini CLI](/examples/gemini-cli) | 在沙箱中运行 Gemini CLI |
| [Codex CLI](/examples/codex-cli) | 在沙箱中运行 OpenAI Codex CLI |
| [应用模型补丁](/examples/apply-patch) | 使用自定义沙箱镜像应用 Codex 风格的文件补丁 |
| [OpenCode](/examples/opencode) | 在沙箱中运行 OpenCode 编码智能体 |
| [Qwen Code](/examples/qwen-code) | 在沙箱中运行 Qwen Code CLI |
| [Kimi CLI](/examples/kimi-cli) | 在沙箱中运行 Kimi CLI（Moonshot AI） |
| [LangGraph](/examples/langgraph) | 结合沙箱的 LangGraph 状态机工作流 |
| [Deep Agents](/examples/deep-agents) | 在沙箱中运行 Deep Agents 的文件/Shell 工具 |
| [Google ADK](/examples/google-adk) | 使用 OpenSandbox 工具的 Google ADK 智能体 |
| [OpenClaw](/examples/openclaw) | 在沙箱内运行 OpenClaw Gateway |
| [DeerFlow](/examples/deer-flow) | DeerFlow 智能体回合及其工具在沙箱中运行 |

## 浏览器与桌面

在沙箱中执行浏览器工作负载并托管桌面环境。

| 示例 | 说明 |
|---------|-------------|
| [Chrome](/examples/chrome) | 带 VNC 和 DevTools 的 Chromium 沙箱 |
| [Playwright](/examples/playwright) | Playwright + Chromium 无头测试 |
| [Desktop](/examples/desktop) | 带 VNC 的完整桌面环境 |
| [VS Code](/examples/vscode) | 沙箱中的 VS Code Web（code-server） |

## 核心用法

基础的沙箱操作与 SDK 工作流。

| 示例 | 说明 |
|---------|-------------|
| [Code Interpreter](/examples/code-interpreter) | 端到端的 Code Interpreter SDK 工作流 |
| [定时客户端池](/examples/client-pool-schedule) | 使用 Go 的池 Resize API 按业务时段调整空闲容量 |
| [AIO Sandbox](/examples/aio-sandbox) | All-in-One 沙箱搭建 |
| [Agent Sandbox](/examples/agent-sandbox) | Kubernetes agent-sandbox 集成 |
| [AKS + Kata](/examples/aks-kata) | AKS 部署，具备 Kata VM 隔离、入口、出口和 Credential Vault |
| [Windows](/examples/windows) | 通过 KVM/QEMU 运行 Windows 沙箱 |
| [Harbor 评估](/examples/harbor-evaluation) | 运行 Harbor 智能体评估，每次试验一个沙箱 |

## 存储

沙箱的持久化与共享存储方案。

| 示例 | 说明 |
|---------|-------------|
| [宿主机卷挂载](/examples/host-volume-mount) | 将宿主机目录挂载到沙箱中 |
| [Docker PVC 卷](/examples/docker-pvc-volume-mount) | Docker 命名卷挂载 |
| [Docker OSSFS 卷](/examples/docker-ossfs-volume-mount) | Docker OSSFS（OSS FUSE）挂载 |
| [rclone 卷挂载](/examples/rclone-volume-mount) | 通过外部 Docker 插件或 Kubernetes CSI 卷使用远程存储 |
| [Kubernetes PVC](/examples/kubernetes-pvc-volume-mount) | Kubernetes PersistentVolumeClaim 挂载 |

## 如何运行

1. 启动 OpenSandbox 服务器（参见[快速开始](/getting-started/)）
2. 设置环境变量：`export SANDBOX_DOMAIN=...`、`export SANDBOX_API_KEY=...`
3. 按需添加特定服务商的变量（例如 `ANTHROPIC_AUTH_TOKEN`、`OPENAI_API_KEY`）
4. 进入示例目录并运行：`python main.py`

::: tip
大多数示例都包含一个 `main.py` 入口文件。其他语言请遵循各示例的运行说明。部分示例还包含用于容器化执行的 `Dockerfile`。
:::
