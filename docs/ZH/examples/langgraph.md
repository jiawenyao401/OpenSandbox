---
title: LangGraph
description: 使用图驱动的控制流将 LangGraph 与 OpenSandbox 集成，实现智能体编排。
---

# LangGraph 智能体 + OpenSandbox 示例

使用图驱动的控制流将 LangGraph 与 OpenSandbox 集成。该示例使用显式的状态机节点来创建、准备、运行、检查和清理沙箱，并使用一个决策节点在运行步骤失败时通过回退命令重试。

## 启动 OpenSandbox 服务器 [本地]

预先拉取 code-interpreter 镜像（包含 Python）：

```shell
docker pull sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0

# 使用 docker hub
# docker pull opensandbox/code-interpreter:v1.1.0
```

启动本地 OpenSandbox 服务器，日志将显示在终端中：

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
opensandbox-server
```

## 运行示例

```shell
# 安装 OpenSandbox + LangGraph 依赖
uv pip install opensandbox langgraph langchain-anthropic

# 运行示例（需要 SANDBOX_DOMAIN / SANDBOX_API_KEY / ANTHROPIC_API_KEY）
uv run python examples/langgraph/main.py
```

该工作流会写入文件、执行一个作业、失败时使用回退命令重试（默认
`python` 与 `python3`），然后用 Claude 总结结果并清理沙箱实例。

![LangGraph + OpenSandbox 截图](../public/images/langgraph-screenshot.jpg)

## 环境变量

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `SANDBOX_DOMAIN` | `localhost:8080` | 沙箱服务地址 |
| `SANDBOX_API_KEY` | _（本地可选）_ | 若服务器要求认证，则提供 API key |
| `SANDBOX_IMAGE` | `sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0` | 要使用的沙箱镜像 |
| `ANTHROPIC_API_KEY` | _（未设置 `ANTHROPIC_AUTH_TOKEN` 时必需）_ | 你的 Anthropic API key |
| `ANTHROPIC_AUTH_TOKEN` | _（备选）_ | 备选的 Anthropic 认证令牌（使用 `Authorization` 头） |
| `ANTHROPIC_BASE_URL` | _（可选）_ | Anthropic API 端点覆盖 |
| `ANTHROPIC_MODEL` | `claude-3-5-sonnet-latest` | 要使用的模型 |

::: warning
`ANTHROPIC_API_KEY` 和 `ANTHROPIC_AUTH_TOKEN` 不应同时设置。
:::

## 参考

- [LangGraph](https://langchain-ai.github.io/langgraph/) - 智能体工作流框架
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/langgraph)
