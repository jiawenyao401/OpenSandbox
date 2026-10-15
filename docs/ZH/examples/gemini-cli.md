---
title: Gemini CLI
description: 在 OpenSandbox 中通过 @google/gemini-cli npm 包调用 Google Gemini。
---

# Gemini CLI 示例

在 OpenSandbox 中通过 `@google/gemini-cli` npm 包调用 Google Gemini。

## 启动 OpenSandbox 服务器 [本地]

预先拉取 code-interpreter 镜像（包含 Node.js）：

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

## 创建并访问 Gemini 沙箱

```shell
# 安装 OpenSandbox 包
uv pip install opensandbox

# 运行示例（需要 SANDBOX_DOMAIN / SANDBOX_API_KEY / GEMINI_API_KEY）
uv run python examples/gemini-cli/main.py
```

该脚本会在运行时安装 Gemini CLI（`npm install -g @google/gemini-cli@latest`，code-interpreter 镜像中已包含 Node.js），然后发送一个简单的请求 `gemini "Compute 1+1=?."`。认证通过 `GEMINI_API_KEY` 传入；你可以通过 `GEMINI_BASE_URL` / `GEMINI_MODEL` 覆盖端点/模型。

## 环境变量

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `SANDBOX_DOMAIN` | `localhost:8080` | 沙箱服务地址 |
| `SANDBOX_API_KEY` | _（本地可选）_ | 若服务器要求认证，则提供 API key |
| `SANDBOX_IMAGE` | `sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0` | 要使用的沙箱镜像 |
| `GEMINI_API_KEY` | _（必需）_ | 你的 Google Gemini API key |
| `GEMINI_BASE_URL` | _（可选）_ | Gemini API 端点（例如代理） |
| `GEMINI_MODEL` | `gemini-2.5-flash` | 要使用的模型 |

## 参考

- [@google/gemini-cli](https://www.npmjs.com/package/@google/gemini-cli) - Gemini CLI
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/gemini-cli)
