---
title: Codex CLI
description: 使用官方 @openai/codex npm 包在 OpenSandbox 中调用 OpenAI/Codex 类模型。
---

# Codex/OpenAI CLI 示例

使用官方 `@openai/codex` npm 包在 OpenSandbox 中调用 OpenAI/Codex 类模型。

## 启动 OpenSandbox 服务器 [本地]

预先拉取 code-interpreter 镜像（包含 Node.js）：

```shell
docker pull sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0

# use docker hub
# docker pull opensandbox/code-interpreter:v1.1.0
```

启动本地 OpenSandbox 服务器，日志将在终端中可见：

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
opensandbox-server
```

## 创建并访问 Codex 沙箱

```shell
# Install OpenSandbox package
uv pip install opensandbox

# Run the example (requires SANDBOX_DOMAIN / SANDBOX_API_KEY / OPENAI_API_KEY)
uv run python examples/codex-cli/main.py
```

该脚本在运行时安装 Codex CLI（`npm install -g @openai/codex@latest`）（Node.js 已包含在 code-interpreter 镜像中），然后执行一个简单请求 `codex exec "Compute 1+1 and return JSON with keys result and reasoning." --skip-git-repo-check`。鉴权通过 `OPENAI_API_KEY` 传递；你可以用 `OPENAI_BASE_URL` / `OPENAI_MODEL` 覆盖端点/模型。

## 多轮会话：恢复上一次运行

每次 `codex exec` 运行都是一个会话。对于需要后续回合的流水线（审查 → 修复、规划 → 实现），应恢复会话，而不是用全新的上下文重新开始。

回合 1 — 使用 `--json` 运行并捕获会话 ID。`--json` 将 stdout 变成 JSON Lines（JSONL）事件流；第一个事件 `thread.started` 携带 `thread_id`，即 `codex exec resume` 接受的会话 ID：

```shell
codex exec --json "Remember this for later: my favorite sandbox number is 42." --skip-git-repo-check
```

```json
{"type": "thread.started", "thread_id": "0199a213-81c0-7800-8aa1-bbab2a035a53"}
```

回合 2 — 使用后续提示恢复该会话；模型会回想起前几轮的上下文，因此回复是 `42`：

```shell
codex exec resume "0199a213-81c0-7800-8aa1-bbab2a035a53" "What is my favorite sandbox number? Reply with just the number." --skip-git-repo-check
```

::: tip
`codex exec resume --last "..."` 会继续当前工作目录中最近的一次会话——便于快速的两段式流水线。示例脚本改为捕获显式 ID，在多个会话交错时也能保持正确。
:::

如果你只需要最终消息供下游步骤使用，`-o <path>` / `--output-last-message <path>` 会将其写入文件，而 `--output-schema <file>` 会根据 JSON Schema 校验最终响应。

## 沙箱与审批模式

Codex 对模型生成的 Shell 命令应用自己的沙箱策略；这些标志在 OpenSandbox 容器内对其进行控制：

- `--sandbox workspace-write`（或 `-s workspace-write`）允许模型生成的命令在工作区内写入——这是自主运行的常见选择。
- `--sandbox read-only` 使运行严格保持只读。
- `-a never` / `--ask-for-approval never` 从不暂停等待人工审批，因此非交互式运行不会阻塞在人工环节。
- `--dangerously-bypass-approvals-and-sandbox`（别名 `--yolo`）在无审批、无沙箱的情况下运行每条命令。Codex CLI 文档指出它仅在"外部加固过的环境内"是安全的——用 OpenSandbox 的术语来说，这意味着沙箱运行的是加固的运行时（gVisor、Kata），而不是默认的 runc；参见[安全容器运行时](/guides/secure-container)。

::: warning
`--full-auto` 已弃用——请优先使用 `--sandbox workspace-write`；使用旧标志时 Codex 会打印警告。
:::

## 环境变量

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `SANDBOX_DOMAIN` | `localhost:8080` | 沙箱服务地址 |
| `SANDBOX_API_KEY` | _（本地可选）_ | 服务器需要鉴权时使用的 API key |
| `SANDBOX_IMAGE` | `sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0` | 要使用的沙箱镜像 |
| `OPENAI_API_KEY` | _（必填）_ | 你的 OpenAI API key |
| `OPENAI_BASE_URL` | `https://api.openai.com/v1` | OpenAI API 端点 |
| `OPENAI_MODEL` | `gpt-4o-mini` | 要使用的模型 |

## 参考资料

- [@openai/codex](https://www.npmjs.com/package/@openai/codex) - 官方 OpenAI Codex CLI
- [Codex 非交互模式](https://developers.openai.com/codex/noninteractive) - `codex exec`、JSONL 输出和会话恢复
- [Codex CLI 参考](https://developers.openai.com/codex/cli/reference) - 沙箱与审批标志
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/codex-cli)
