---
title: Google ADK
description: 将 Google Agent Development Kit (ADK) 与 OpenSandbox 集成，构建工具调用智能体。
---

# Google ADK + OpenSandbox 示例

将 Google Agent Development Kit (ADK) 与 OpenSandbox 集成。ADK 智能体驱动在沙箱内执行的工具调用。

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
# 安装 OpenSandbox + Google ADK 依赖
uv pip install opensandbox google-adk

# 运行示例（需要 SANDBOX_DOMAIN / SANDBOX_API_KEY / GOOGLE_API_KEY）
uv run python examples/google-adk/main.py
```

该脚本使用 ADK 创建一个带有 OpenSandbox 工具（`write_file`、`read_file`、`run_in_sandbox`）的智能体。它会运行若干提示词，打印工具事件，并清理沙箱。

## 环境变量

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `SANDBOX_DOMAIN` | `localhost:8080` | 沙箱服务地址 |
| `SANDBOX_API_KEY` | _（本地可选）_ | 若服务器要求认证，则提供 API key |
| `SANDBOX_IMAGE` | `sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0` | 要使用的沙箱镜像 |
| `GOOGLE_API_KEY` | _（必需）_ | Gemini API key |
| `GOOGLE_ADK_MODEL` | `gemini-2.5-flash` | Gemini 模型名称 |

## 参考

- [Google ADK](https://google.github.io/adk-docs/) - Agent Development Kit
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/google-adk)
