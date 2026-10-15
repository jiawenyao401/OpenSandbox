---
title: Deep Agents
description: 运行一个 Deep Agent，其文件和 Shell 工具在 OpenSandbox 沙箱内执行。
---

# Deep Agents + OpenSandbox 示例

运行一个 [Deep Agent](https://github.com/langchain-ai/deepagents)，其文件和 Shell 工具
在 OpenSandbox 沙箱内执行。
[`langchain-sandbox-opensandbox`](https://pypi.org/project/langchain-sandbox-opensandbox/)
后端将 OpenSandbox Python SDK 适配到 Deep Agents 的 `BaseSandbox` 接口，因此智能体执行的每一次
`ls` / `read_file` / `write_file` / `glob` / `grep` / 命令调用都是沙箱化的。

该后端针对运行中的服务器通过了完整的 `langchain-tests`
`SandboxIntegrationTests` 一致性测试套件
（86/86）。

## 启动 OpenSandbox 服务器 [本地]

启动本地 OpenSandbox 服务器，日志将在终端中可见：

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
opensandbox-server
```

## 运行示例

```shell
# Install Deep Agents + the OpenSandbox backend
uv pip install deepagents langchain-sandbox-opensandbox

# Run the example (requires SANDBOX_DOMAIN / ANTHROPIC_API_KEY)
uv run python examples/deep-agents/main.py
```

该智能体将一个 Python 脚本写入沙箱、运行它并报告输出——所有文件
和 Shell 操作都发生在沙箱内而非宿主机上。沙箱在
退出时被销毁（已终止且本地资源已关闭）。

## 环境变量

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `SANDBOX_DOMAIN` | `localhost:8080` | 沙箱服务地址（主机及可选端口） |
| `SANDBOX_PROTOCOL` | `http` | 访问服务器使用的协议（`http` 或 `https`） |
| `SANDBOX_API_KEY` | _（本地可选）_ | 服务器需要鉴权时使用的 API key |
| `ANTHROPIC_API_KEY` | _（必填）_ | 默认 Deep Agents 模型使用的 Anthropic API key |

## 参考资料

- [Deep Agents](https://github.com/langchain-ai/deepagents) - 智能体框架
- [langchain-sandbox-opensandbox](https://pypi.org/project/langchain-sandbox-opensandbox/) - Deep Agents 的 OpenSandbox 后端
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/deep-agents)
