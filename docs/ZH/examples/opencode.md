---
title: OpenCode
description: 在 OpenSandbox 容器中运行 OpenCode 编程智能体。
---

# OpenCode 示例

在 OpenSandbox 容器中运行 [OpenCode](https://opencode.ai/)。该示例会安装官方 CLI，并使用 `opencode run` 以非交互方式执行提示词。

## 启动 OpenSandbox 服务器 [本地]

预先拉取 code-interpreter 镜像（包含 Node.js）：

```shell
docker pull sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0

# 使用 Docker Hub
# docker pull opensandbox/code-interpreter:v1.1.0
```

然后启动本地 OpenSandbox 服务器。日志将显示在终端中：

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
opensandbox-server
```

## 创建并访问 OpenCode 沙箱

```shell
# 安装 OpenSandbox 包
uv pip install opensandbox

# 运行示例。默认的免费模型不需要 API key。
uv run python examples/opencode/main.py
```

该脚本会在运行时安装 OpenCode（`npm install -g opencode-ai@latest`），创建一个隔离的工作目录，并以一个简单的提示词运行 `opencode run`。默认模型目前无需认证即可使用。如需改用 OpenCode provider 的需认证模型，请设置 `OPENCODE_API_KEY` 和 `OPENCODE_MODEL`。

## 环境变量

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `SANDBOX_DOMAIN` | `localhost:8080` | 沙箱服务地址 |
| `SANDBOX_API_KEY` | _（本地可选）_ | 若服务器要求认证，则提供 API key |
| `SANDBOX_IMAGE` | `sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0` | 要使用的沙箱镜像 |
| `OPENCODE_MODEL` | `opencode/deepseek-v4-flash-free` | `provider/model` 格式的模型 ID |
| `OPENCODE_API_KEY` | _（可选）_ | OpenCode provider 需认证模型的 API key |

::: info
默认的免费模型由 OpenCode 限时提供。请勿向免费模型发送私有代码、凭据或其他机密数据；使用前请查阅该 provider 的数据使用条款。需要其他模型时请覆盖 `OPENCODE_MODEL`。其他 provider 可能需要额外的环境变量或配置。
:::

## 参考

- [OpenCode CLI](https://opencode.ai/docs/cli/) - 安装与非交互式 `run` 命令
- [OpenCode Providers](https://opencode.ai/docs/providers) - provider 认证与配置
- [OpenCode Zen](https://opencode.ai/docs/zen/) - 模型可用性与数据使用说明
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/opencode)
