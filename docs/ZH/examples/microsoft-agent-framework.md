---
title: Microsoft Agent Framework
description: 使用 OpenSandbox 运行通用用途的 Microsoft Agent Framework 智能体。
---

# Microsoft Agent Framework 示例

将一个通用用途的 Microsoft Agent Framework 智能体与 OpenSandbox 工作流结合运行。该示例使用 Agent Framework 的函数式工作流 API 创建沙箱、准备并执行 Python 作业、必要时用回退命令重试、向智能体提问一个英文问题，然后销毁沙箱。

Python 客户端使用 OpenSandbox SDK。本地 OpenSandbox Server 通过其 Docker 运行时创建和管理容器。

沙箱使用 Docker Hub 镜像 `opensandbox/code-interpreter:v1.1.0`；无需阿里云容器镜像仓库。

## 启动 OpenSandbox 服务器 [本地]

预先拉取 code-interpreter 镜像（包含 Python 3.12+）：

```shell
docker pull opensandbox/code-interpreter:v1.1.0
```

创建 Python 3.10+ 环境并安装示例项目声明的包：

```shell
uv venv
source .venv/bin/activate

uv pip install \
  --default-index https://packagefeedproxy.microsoft.io/pypi/simple \
  "agent-framework-core==1.19.0" \
  "agent-framework-openai==1.14.4" \
  "opensandbox==1.1.0" \
  "opensandbox-server==0.2.3" \
  "python-dotenv>=1.0,<2"
```

这些按需选择的 Agent Framework 包提供了本示例所需的工作流、智能体以及 OpenAI/Azure OpenAI 客户端 API，而不会安装无关的集成。Microsoft 包代理与示例项目 `pyproject.toml` 中配置的包源一致。

初始化并启动本地 Docker 运行时：

```shell
uv run opensandbox-server init-config ~/.sandbox.toml --example docker
OPENSANDBOX_INSECURE_SERVER=YES uv run opensandbox-server
```

`OPENSANDBOX_INSECURE_SERVER=YES` 明确确认此本地服务器没有 API key。请将其保持绑定在 localhost 上。对于共享部署，请改为配置 `server.api_key`。

该项目固定使用 `opensandbox-server==0.2.3`。上游 `1.1.0` 的 server wheel 缺少生成的 FastPath gRPC 模块，会在启动时失败。

## 配置智能体

复制环境变量模板：

```shell
cp examples/microsoft-agent-framework/.env.example .env
```

配置 OpenAI：

```shell
OPENAI_API_KEY=replace_me
OPENAI_MODEL=gpt-4o-mini
```

或配置 Azure OpenAI：

```shell
AZURE_OPENAI_API_KEY=replace_me
AZURE_OPENAI_ENDPOINT=https://<your-resource>.openai.azure.com/
AZURE_OPENAI_MODEL=<your-deployment-name>
```

当两种配置同时存在时，`OPENAI_API_KEY` 优先。删除它即可回退使用 Azure OpenAI 环境变量。

## 运行示例

从仓库根目录运行：

```shell
uv run python examples/microsoft-agent-framework/main.py
```

默认的英文提示词是：

```text
What model are you using?
```

你可以在 `.env` 中设置其他问题，或仅在单次调用时覆盖：

```shell
AGENT_QUESTION="Explain what OpenSandbox does." \
  uv run python examples/microsoft-agent-framework/main.py
```

该工作流会在沙箱中写入并执行一个小型 Python 作业，然后将问题、配置的模型名和沙箱结果传给通用用途的 Agent Framework 智能体。即使执行或智能体调用失败，沙箱也会被停止并关闭。

## 环境变量

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `SANDBOX_DOMAIN` | `localhost:8080` | 本地沙箱服务地址 |
| `SANDBOX_API_KEY` | _（本地可选）_ | 若已配置，则提供 API key |
| `SANDBOX_IMAGE` | `opensandbox/code-interpreter:v1.1.0` | Docker Hub 沙箱镜像 |
| `SANDBOX_COMMAND` | `python3 /tmp/math.py` | 初始沙箱命令 |
| `SANDBOX_FALLBACK_COMMAND` | `python /tmp/math.py` | 重试命令 |
| `AGENT_QUESTION` | `What model are you using?` | 发送给智能体的英文问题 |
| `OPENAI_API_KEY` | _（使用 OpenAI 时必需）_ | OpenAI API key |
| `OPENAI_MODEL` | `.env.example` 中的 `gpt-4o-mini` | OpenAI 模型 |
| `AZURE_OPENAI_API_KEY` | _（Azure API key 认证时必需）_ | Azure OpenAI API key |
| `AZURE_OPENAI_ENDPOINT` | _（使用 Azure OpenAI 时必需）_ | Azure OpenAI 端点 |
| `AZURE_OPENAI_MODEL` | _（使用 Azure OpenAI 时必需）_ | Azure 部署名称 |

## 参考

- [Microsoft Agent Framework Python 示例](https://github.com/microsoft/agent-framework/tree/main/python/samples)
- [带智能体的函数式工作流](https://github.com/microsoft/agent-framework/blob/main/python/samples/01-get-started/05_functional_workflow_with_agents.py)
- [OpenSandbox](https://github.com/opensandbox-group/OpenSandbox)
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/microsoft-agent-framework)
