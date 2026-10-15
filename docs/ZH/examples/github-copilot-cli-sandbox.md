---
title: GitHub Copilot CLI
description: 在本地 OpenSandbox 容器中运行 GitHub Copilot CLI。
---

# GitHub Copilot CLI 示例

在 OpenSandbox 容器内运行官方 GitHub Copilot CLI。该示例会下载官方 CLI 发行版，并使用 `copilot -sp` 以非交互方式执行提示词。

该示例遵循与 OpenCode 示例相同的本地工作流。Python 客户端只使用 OpenSandbox SDK；本地 OpenSandbox Server 通过其 Docker 运行时创建和管理容器。

沙箱使用 Docker Hub 镜像 `opensandbox/code-interpreter:v1.1.0`；不使用阿里云容器镜像仓库。

## 启动 OpenSandbox 服务器 [本地]

预先拉取 code-interpreter 镜像：

```shell
docker pull opensandbox/code-interpreter:v1.1.0
```

安装依赖并初始化本地 Docker 运行时：

```shell
uv sync
uv run opensandbox-server init-config ~/.sandbox.toml --example docker
OPENSANDBOX_INSECURE_SERVER=YES uv run opensandbox-server
```

`OPENSANDBOX_INSECURE_SERVER=YES` 明确确认此本地服务器没有 API key。请将其保持绑定在 localhost 上。对于共享部署，请改为配置 `server.api_key`。

该项目当前固定使用 `opensandbox-server==0.2.3`。已知上游 `1.1.0` 的 server wheel 缺少生成的 FastPath gRPC 模块，会在启动时失败。

## 创建并访问 GitHub Copilot 沙箱

创建一个具有 **Copilot Requests** 权限的 GitHub 细粒度个人访问令牌（fine-grained PAT）。将其保存在被 gitignore 的本地 `.env` 文件中：

```shell
GH_TOKEN=github_pat_replace_me
```

在仓库根目录运行示例：

```shell
uv run python examples/github-copilot-cli-sandbox/main.py
```

该脚本会创建一个基于 Docker 的沙箱，使用官方安装脚本安装 GitHub Copilot CLI，在一个隔离的工作目录中运行 `copilot -sp`，随后销毁容器。

Microsoft npm 代理目前不提供 `@github/copilot`，因此本示例使用官方安装器，它会下载并校验 GitHub Release 制品。

## 环境变量

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `GH_TOKEN` | _（必需）_ | 具有 **Copilot Requests** 权限的细粒度 PAT |
| `GITHUB_TOKEN` | _（可选）_ | 在未设置 `GH_TOKEN` 时使用 |
| `SANDBOX_DOMAIN` | `localhost:8080` | 本地沙箱服务地址 |
| `SANDBOX_API_KEY` | _（本地可选）_ | 若已配置，则提供 API key |
| `SANDBOX_IMAGE` | `opensandbox/code-interpreter:v1.1.0` | Docker Hub 沙箱镜像 |
| `COPILOT_PROMPT` | 算术示例 | 传给 `copilot -sp` 的提示词 |
| `COPILOT_VERSION` | 最新稳定版本 | 可选的要安装的官方 CLI 发行版 |

::: warning
切勿提交 GitHub 令牌。如果令牌出现在终端输出、文档、源代码或聊天记录中，请立即撤销并创建新令牌。
:::

## 参考

- [GitHub Copilot CLI](https://github.com/github/copilot-cli)
- [GitHub Copilot CLI 快速入门](https://docs.github.com/en/copilot/get-started/cli-quickstart)
- [源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/github-copilot)
