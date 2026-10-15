---
title: Kimi CLI
description: 在 OpenSandbox 容器中运行 Kimi Code CLI（Moonshot AI）。
---

# Kimi CLI 示例

在 OpenSandbox 容器中运行 [Kimi Code CLI](https://github.com/MoonshotAI/kimi-cli)（Moonshot AI）。

## 启动 OpenSandbox 服务器 [本地]

预先拉取 code-interpreter 镜像（包含 Python 3.12+）：

```shell
docker pull sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0

# 使用 docker hub
# docker pull opensandbox/code-interpreter:v1.1.0
```

然后启动本地 OpenSandbox 服务器，stdout 日志将显示在终端中：

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
opensandbox-server
```

## 创建并访问 Kimi 沙箱

```shell
# 安装 OpenSandbox 包
uv pip install opensandbox

# 运行示例（需要 SANDBOX_DOMAIN / SANDBOX_API_KEY / KIMI_API_KEY）
uv run python examples/kimi-cli/main.py
```

该脚本会在运行时安装 Kimi Code CLI（`pip install kimi-cli`，code-interpreter 镜像中已包含 Python 3.12+），然后发送一个简单的请求 `kimi -p "Compute 1+1=?."`。认证通过 `KIMI_API_KEY` 传入，你可以通过 `KIMI_BASE_URL` / `KIMI_MODEL_NAME` 覆盖端点/模型。

## 环境变量

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `SANDBOX_DOMAIN` | `localhost:8080` | 沙箱服务地址 |
| `SANDBOX_API_KEY` | _（本地可选）_ | 若服务器要求认证，则提供 API key |
| `SANDBOX_IMAGE` | `sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0` | 要使用的沙箱镜像 |
| `KIMI_API_KEY` | _（必需）_ | 你的 Moonshot AI / Kimi API key |
| `KIMI_BASE_URL` | _（可选）_ | Kimi API 端点（默认为 Kimi 官方端点） |
| `KIMI_MODEL_NAME` | `kimi-k2.5` | 要使用的模型 |

## 参考

- [Kimi Code CLI](https://github.com/MoonshotAI/kimi-cli) - Kimi Code CLI 官方仓库
- [Moonshot AI 平台](https://platform.moonshot.ai/) - API key 管理与文档
- [Kimi CLI 文档](https://moonshotai.github.io/kimi-cli/en/) - 完整 CLI 文档
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/kimi-cli)
