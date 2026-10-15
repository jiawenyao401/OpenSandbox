---
title: Claude Code
description: 在 OpenSandbox 容器中通过 claude-cli npm 包访问 Claude。
---

# Claude Code 示例

在 OpenSandbox 中通过 `claude-cli` npm 包访问 Claude。

## 启动 OpenSandbox 服务器 [本地]

预先拉取 code-interpreter 镜像（包含 Node.js）：

```shell
docker pull sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0

# use docker hub
# docker pull opensandbox/code-interpreter:v1.1.0
```

然后启动本地 OpenSandbox 服务器，stdout 日志将在终端中可见：

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
opensandbox-server
```

## 创建并访问 Claude 沙箱

```shell
# Install OpenSandbox package
uv pip install opensandbox

# Run the example (requires SANDBOX_DOMAIN / SANDBOX_API_KEY / ANTHROPIC_AUTH_TOKEN)
uv run python examples/claude-code/main.py
```

该脚本在运行时安装 Claude CLI（`npm i -g @anthropic-ai/claude-code@latest`）（Node.js 已包含在 code-interpreter 镜像中），然后发送一个简单请求 `claude "Compute 1+1=?."`。鉴权通过 `ANTHROPIC_AUTH_TOKEN` 传递，你可以用 `ANTHROPIC_BASE_URL` / `ANTHROPIC_MODEL` 覆盖端点/模型。

![Claude Code screenshot](../public/images/claude-code-screenshot.jpg)

## 无头模式与会话恢复

该示例脚本还演示了编码智能体集成常用的两种模式：带结构化输出的无头运行，以及恢复同一对话的后续回合。

### 回合 1：带结构化输出的无头运行

```shell
claude -p "Remember this for later: my favorite sandbox number is 42." --output-format json
```

使用 `-p`（print 模式）时，CLI 会回答提示后退出。`--output-format json` 会将回复、`session_id`、用量和成本元数据作为一个 JSON 对象打印，因此脚本会提取 `session_id` 供下一回合使用。

### 回合 2：恢复会话

```shell
claude -p "What is my favorite sandbox number? Reply with just the number." \
  --resume <session_id> --output-format json
```

`--resume` 会在上下文完整保留的情况下继续特定对话——回复会回想起回合 1 中的 "42"。`--continue` 恢复的是最近的对话；当同一沙箱中存在多个对话时，显式的 `--resume <session_id>` 能保持确定性。

::: tip 并发后续请求
在 `--resume` / `--continue` 上添加 `--fork-session` 可以创建一个新的会话 ID，而不是复用原始 ID，这样针对同一对话的并行后续请求就不会相互覆盖。
:::

### 无人值守运行中的权限提示

在 `-p` 运行中没有终端可以响应权限提示：本来会触发提示的请求将被拒绝，因此需要工具（文件编辑、Shell 命令）的任务可能失败。针对智能体工作负载有两种常见补救措施：

- `--dangerously-skip-permissions`（等同于 `--permission-mode bypassPermissions`）无需提示即可运行工具——这是临时的 OpenSandbox 容器内的常见选择，沙箱本身就是隔离边界。
- `--permission-mode dontAsk` 保持失败即关闭（fail-closed）：只有权限规则预先批准的工具才会运行——这是锁定 CI 的标准模式。（`--permission-prompts none`，Claude Code v2.1.259+，是 print 模式下的等价选项。）

::: warning 交互式 UI 需要 TTY
交互式 REPL（`claude "..."`）在终端上渲染其 UI，包括权限提示。通过普通命令管道无法响应提示，因此运行可能会无限期等待。脚本化运行请使用 `-p`，或通过 [PTY 会话](/architecture/data-plane/execd)驱动交互式 UI。
:::

要在输出生成时即时接收（令牌级事件）而不是单个最终 JSON 对象，请使用 `--output-format stream-json --verbose --include-partial-messages`。

## 环境变量

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `SANDBOX_DOMAIN` | `localhost:8080` | 沙箱服务地址 |
| `SANDBOX_API_KEY` | _（本地可选）_ | 服务器需要鉴权时使用的 API key |
| `SANDBOX_IMAGE` | `sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0` | 要使用的沙箱镜像 |
| `ANTHROPIC_AUTH_TOKEN` | _（必填）_ | 你的 Anthropic 鉴权令牌 |
| `ANTHROPIC_BASE_URL` | _（可选）_ | Anthropic API 端点（例如自托管代理） |
| `ANTHROPIC_MODEL` | `claude_sonnet4` | 模型名称 |

## 参考资料

- [claude-code](https://www.npmjs.com/package/claude-code) - Claude Code CLI 的 NPM 包
- [Claude Code 无头模式](https://code.claude.com/docs/en/headless) - Print 模式、输出格式和会话恢复
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/claude-code)
