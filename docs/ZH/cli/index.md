---
title: CLI
description: 用于管理沙箱、执行命令、操作文件以及安装智能体技能的 osb 命令行界面。
---

# OpenSandbox CLI

`osb` 是 OpenSandbox 的命令行界面，专为常见的日常使用流程而设计：

- 创建和管理沙箱
- 在沙箱内运行命令
- 读取和修改沙箱文件
- 检查运行时 egress 策略
- 管理沙箱本地的 Credential Vault 状态
- 收集底层诊断信息
- 为编码智能体安装 OpenSandbox 专属技能

它底层使用 OpenSandbox Python SDK，旨在成为从终端到可用沙箱工作流的最短路径。

## 安装

::: code-group

```bash [pip]
pip install opensandbox-cli
```

```bash [uv]
uv tool install opensandbox-cli
```

```bash [pipx]
pipx install opensandbox-cli
```

:::

确认安装成功：

```bash
osb --help
osb --version
```

## 开始之前

确保可以访问一台 OpenSandbox 服务端。如果在本机运行，请先启动服务端，再将 CLI 指向它。

```bash
opensandbox-server
```

## 快速开始

### 1. 初始化配置

```bash
osb config init
osb config set connection.domain localhost:8080
osb config set connection.protocol http
osb config set connection.api_key <your-api-key>
osb config show -o json
```

如果想使用非默认的配置文件，可在根命令级别为整次调用指定：

```bash
osb --config /tmp/dev.toml config init
osb --config /tmp/dev.toml config set connection.domain localhost:8080
osb --config /tmp/dev.toml config show -o json
```

### 2. 创建沙箱

```bash
osb sandbox create --image python:3.12 --timeout 30m -o json
```

如果先设置好默认值，后续的创建命令可以更简短：

```bash
osb config set defaults.image python:3.12
osb config set defaults.timeout 30m
osb sandbox create -o json
```

### 3. 确认沙箱可用

```bash
osb sandbox get <sandbox-id> -o json
osb sandbox health <sandbox-id> -o json
```

### 4. 在沙箱内运行命令

在沙箱命令负载前使用 `--`。

```bash
osb command run <sandbox-id> -o raw --argv -- python -c "print(1 + 1)"
```

### 5. 读取或写入文件

```bash
osb file write <sandbox-id> /tmp/hello.txt -c "hello" -o json
osb file cat <sandbox-id> /tmp/hello.txt -o raw
```

### 6. 清理

```bash
osb sandbox kill <sandbox-id> -o json
```

## 常见任务

### 创建沙箱

基础用法：

```bash
osb sandbox create --image python:3.12
```

私有镜像：

```bash
osb sandbox create \
  --image my-registry.example.com/team/app:latest \
  --image-auth-username alice \
  --image-auth-password <token>
```

手动清理模式：

```bash
osb sandbox create --image python:3.12 --timeout none
```

显式 entrypoint argv：

```bash
osb sandbox create \
  --image python:3.12 \
  --entrypoint python \
  --entrypoint -m \
  --entrypoint http.server
```

带网络策略和卷创建：

```bash
osb sandbox create \
  --image python:3.12 \
  --network-policy-file network-policy.json \
  --volumes-file volumes.json
```

启用 Credential Vault 代理创建：

```bash
osb sandbox create --image python:3.12 --network-policy-file network-policy.json --credential-proxy -o json
```

从模板创建（golden image）：

```bash
osb sandbox create --template <template-id> --timeout 30m -o json
```

模板模式在服务端固定了工作负载形态：`--image`、`--env`、`--resource`、`--entrypoint`、`--volumes-file` 和 `--credential-proxy` 不能与 `--template` 组合使用。仅 `--timeout`（必填）、`--metadata`、`--extension` 和 `--network-policy-file` 可以伴随使用。只有处于 `Succeeded` 阶段的模板才可用。参见[管理模板](#manage-templates)。

从快照创建：

```bash
osb sandbox create --snapshot-id <snapshot-id> --timeout 30m -o json
```

`--image`、`--template` 和 `--snapshot-id` 互斥。

### 管理模板

模板是 golden image 构建；构建是异步运行的，因此需要轮询 `template get` 直到状态阶段变为 `Succeeded`。模板管理需要 Kubernetes 支持的运行时。

```bash
osb template create --image python:3.12 --publish s3://bucket/publish -o json
osb template get <template-id> -o json
osb template list -o json
osb template delete <template-id> -o json
```

`template create` 还支持 `--resource cpu=1 memory=512Mi disk=2Gi`、`--entrypoint`（每个 argv 项重复一次）、`--env KEY=VALUE`、`--metadata KEY=VALUE`、`--format native|overlaybd`、`--readiness-probe` 和 `--warmup-seconds`。

### 管理快照

快照捕获沙箱的状态，可用于创建新的沙箱。

```bash
osb snapshot create <sandbox-id> --name golden -o json
osb snapshot get <snapshot-id> -o json
osb snapshot list --sandbox-id <sandbox-id> -o json
osb snapshot delete <snapshot-id> -o json
```

### 列出和检查沙箱

```bash
osb sandbox list
osb sandbox list -o json
osb sandbox list --state running --state paused
osb sandbox get <sandbox-id> -o json
osb sandbox metrics <sandbox-id>
osb sandbox metrics <sandbox-id> --watch -o raw
```

### 暂停、恢复和续期

```bash
osb sandbox pause <sandbox-id> -o json
osb sandbox get <sandbox-id> -o json
# 带截止时间重复调用 get，直到 status.state 变为 Paused；若为 Failed 则停止。
# 然后再恢复：
osb sandbox resume <sandbox-id> --resume-timeout 60s -o json
osb sandbox renew <sandbox-id> --timeout 30m -o json
```

暂停是异步的。暂停请求返回成功仅表示请求已被接受。各运行时的具体行为请参见 [暂停与恢复](/guides/pause-resume)。

### 暴露服务

```bash
osb sandbox endpoint <sandbox-id> --port 8080 -o json
```

请将返回的 endpoint 和 headers 配合使用。该命令只负责解析路由，并不会启动服务，也不能证明应用是健康的。

### 运行命令

前台流式执行：

```bash
osb command run <sandbox-id> -o raw -- sh -lc 'echo ready'
```

可追踪的后台执行：

```bash
osb command run <sandbox-id> --background -o json -- sh -c "sleep 10; echo done"
osb command status <sandbox-id> <execution-id> -o json
osb command logs <sandbox-id> <execution-id> -o json
```

默认情况下，CLI 会对每个参数做 shell 引号转义，并将它们拼接成一条命令字符串。若要使用管道、重定向或沙箱侧的变量展开，请显式传入 shell 和脚本：

```bash
osb command run <sandbox-id> -o raw -- sh -c 'echo "$HOME"; printf "ready\n" | cat'
```

使用 `--argv` 可不经过 shell 直接原生执行。它要求 execd 支持 `argv` 请求字段：

```bash
osb command run <sandbox-id> -o raw --argv -- python3 -c "import sys; print(sys.argv[1:])" "a b" '$HOME' "x'y" ""
osb command interrupt <sandbox-id> <execution-id> -o json
```

前台的 `command run` 支持 `-o raw`；后台执行支持 `table`、`json` 或 `yaml`。

后台执行的元数据只有在 execd 确认启动后才会返回。如果在确认之前发生执行错误或流提前结束，CLI 会以非零退出码退出，且不打印成功元数据。后台启动成功并不意味着进程已结束；请使用 `command status` 和 `command logs` 查看其最终结果。

持久 shell 会话：

```bash
osb command session create <sandbox-id> --workdir /workspace -o json
osb command session run <sandbox-id> <session-id> -o raw -- pwd
osb command session run <sandbox-id> <session-id> -o raw -- export FOO=bar
osb command session run <sandbox-id> <session-id> -o raw -- sh -c 'echo $FOO'
osb command session delete <sandbox-id> <session-id> -o json
```

### 操作文件

```bash
osb file upload <sandbox-id> ./local.txt /workspace/local.txt -o json
osb file download <sandbox-id> /workspace/result.json ./result.json -o json
osb file search <sandbox-id> /workspace --pattern "*.py" -o json
osb file info <sandbox-id> /workspace/main.py -o json
osb file replace <sandbox-id> /workspace/app.py --old old --new new -o json
osb file chmod <sandbox-id> /workspace/script.sh --mode 755 -o json
```

对于普通文件，`file download` 只有在整个下载成功后才会替换本地目标文件。如果下载失败或被中断，已存在的文件保持不变，临时下载文件会被删除。目标目录必须可写，以便 CLI 在替换文件前先暂存下载内容。

已有的设备文件（如 `/dev/null`）和命名管道会直接接收下载数据。它们不会被替换，且如果下载失败或中断，已写入的字节无法回滚。

指向标准输出的目标（如 `/dev/stdout` 和 `/dev/fd/1`）即使 stdout 被重定向到普通文件，也会直接流式输出。这类下载在所有输出格式下都会省略成功消息，以确保 stdout 中只包含文件字节；错误仍然输出到 stderr。失败或被中断的流可能包含部分数据。

### 管理运行时 egress 策略

查看当前策略：

```bash
osb egress get <sandbox-id> -o json
```

修改特定规则：

```bash
osb egress patch <sandbox-id> --rule allow=pypi.org --rule deny=internal.example.com -o json
```

如果要排查连通性问题，请用真实命令验证实际行为：

```bash
osb command run <sandbox-id> -o raw -- curl -I https://pypi.org
```

### 管理 Credential Vault

Credential Vault 操作通过 Python SDK 调用沙箱的 egress sidecar。在写入 vault 状态之前，需使用 `--credential-proxy` 和显式网络策略创建沙箱。基于模板的沙箱不支持 Credential Vault。负载示例和版本检查请参见 [Credential Vault](/guides/credential-vault)。

```bash
osb credential-vault create <sandbox-id> --file vault.yaml -o json
osb credential-vault get <sandbox-id> -o json
osb credential-vault patch <sandbox-id> --file mutation.yaml -o json
osb credential-vault credential list <sandbox-id> -o json
osb credential-vault binding list <sandbox-id> -o json
osb credential-vault delete <sandbox-id> -o json
```

::: warning
使用 `--file -` 从 stdin 读取 JSON/YAML 负载。不要将明文凭证值作为命令行参数传递；应将其保存在负载流或文件中。
:::

### 收集诊断信息

使用稳定的诊断命令获取基于 API 的日志和事件描述符。

```bash
osb diagnostics events <sandbox-id> --scope runtime -o raw
osb diagnostics events <sandbox-id> --scope all -o raw
osb diagnostics logs <sandbox-id> --scope container -o raw
osb diagnostics logs <sandbox-id> --scope all -o json
osb diagnostics events <sandbox-id> --scope runtime -o json
osb diagnostics logs <sandbox-id> --scope container -o yaml
```

稳定诊断命令必须提供 `--scope`。内置服务端对日志支持 `container` 和 `all`，对事件支持 `runtime` 和 `all`。对于不可用的 scope（包括生命周期事件），Docker/Kubernetes 会返回 `DIAGNOSTICS_SCOPE_UNSUPPORTED`。当后端只能提供部分数据时，尽力而为的 scope 可能包含 `warnings` 字段。raw 输出会内联打印诊断文本，或者在诊断以临时 URL 形式交付时打印内容 URL；raw 输出不会下载该 URL。JSON/YAML 使用 Python 模型字段名，如 `content_url`、`content_type` 和 `expires_at`（HTTP API 使用 camelCase）。各运行时的差异请参见 [诊断](/api/#diagnostics)。较旧的服务端版本对 scoped 诊断可能仍返回 `DIAGNOSTICS_NOT_IMPLEMENTED`。

::: info
旧版 DevOps 诊断仍处于实验阶段。请优先使用 `osb diagnostics logs/events` 获取稳定的、基于 API 的日志与事件收集。
:::

```bash
osb devops inspect <sandbox-id> -o raw
osb devops summary <sandbox-id> -o raw
```

## 输出格式

输出格式的选择是命令级别的，而非全局的。

- `table`：人类可读的表格和面板
- `json`：机器可读的 JSON
- `yaml`：机器可读的 YAML
- `raw`：无格式的文本或流式输出

示例：

```bash
osb sandbox list -o json
osb sandbox list -o yaml
osb file cat <sandbox-id> /workspace/hello.txt -o raw
```

并非每个命令都支持每种格式。不确定时，请在具体命令上使用 `--help`。

## 命令组

主要命令组包括：

- `osb sandbox`：生命周期管理
- `osb template`：golden image 模板构建
- `osb snapshot`：快照管理
- `osb command`：命令执行与持久会话
- `osb file`：文件和目录操作
- `osb egress`：运行时 egress 策略
- `osb credential-vault`：沙箱本地的凭证与绑定
- `osb diagnostics`：稳定的诊断日志与事件
- `osb devops`：实验性的旧版诊断
- `osb config`：本地 CLI 配置
- `osb skills`：面向 AI 工具的内置技能

CLI 不提供 Client Pool 或池追踪命令；这些工作流请使用 SDK/API。Python SDK 中的功能并不自动对应一条 CLI 命令。

可以直接探索各命令组：

```bash
osb sandbox --help
osb template --help
osb snapshot --help
osb command --help
osb file --help
osb skills --help
```

## 智能体技能

CLI 内置了面向编码智能体和智能体工具的 OpenSandbox 技能。

内置技能：

- `sandbox-lifecycle`
- `command-execution`
- `file-operations`
- `network-egress`
- `sandbox-troubleshooting`

支持的目标：

| 目标 | 安装位置 |
| --- | --- |
| `claude` | `./.claude/skills/<name>/SKILL.md` 或 `~/.claude/skills/<name>/SKILL.md` |
| `cursor` | `./.cursor/rules/` 或 `~/.cursor/rules/` |
| `codex` | `./.codex/skills/<name>/SKILL.md` 或 `~/.codex/skills/<name>/SKILL.md` |
| `copilot` | `./.github/copilot-instructions.md` 或 `~/.copilot/copilot-instructions.md` |
| `windsurf` | `./.windsurfrules` 或 `~/.windsurfrules` |
| `cline` | `./.clinerules` 或 `~/.cline/rules/opensandbox-<name>.md` |
| `opencode` | `./.agents/skills/<name>/SKILL.md` 或 `~/.agents/skills/<name>/SKILL.md` |

常用流程：

```bash
osb skills list
osb skills show sandbox-lifecycle
osb skills install sandbox-lifecycle --target codex --scope project
osb skills install --all-builtins --target codex --scope global
osb skills uninstall sandbox-troubleshooting --target claude --scope global
```

在脚本或智能体中使用结构化输出：

```bash
osb skills install sandbox-lifecycle --target codex --scope project -o json
```

## 配置模型

CLI 按以下顺序解析配置：

1. 根命令 CLI 标志，如 `--api-key`、`--domain`、`--protocol`、`--request-timeout`、`--config`
2. 环境变量，如 `OPEN_SANDBOX_API_KEY` 和 `OPEN_SANDBOX_DOMAIN`
3. 配置文件，默认为 `~/.opensandbox/config.toml`
4. SDK 默认值

配置命令：

```bash
osb config init
osb config show
osb config set connection.domain localhost:8080
osb config set connection.protocol http
osb config set defaults.image python:3.12
osb config set defaults.timeout 30m
```

配置文件示例：

```toml
[connection]
api_key = "your-api-key"
domain = "localhost:8080"
protocol = "http"
request_timeout = 30
use_server_proxy = false

[output]
color = true

[defaults]
image = "python:3.12"
timeout = "30m"
```

## 开发

在本 monorepo 中进行本地开发：

```bash
cd cli
uv sync
uv run osb --help
uv run pytest
```

本仓库对 OpenSandbox Python SDK 使用了本地 `uv` source override，因此从 `cli/` 目录运行时会解析到 monorepo 中检出的 SDK。
