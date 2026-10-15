---
title: DeerFlow
description: 通过 DeerFlow 2.1.0 中内置的 OpenSandboxProvider 在 OpenSandbox 上运行 DeerFlow 智能体回合。
---

# DeerFlow + OpenSandbox 示例

[DeerFlow](https://github.com/bytedance/deer-flow) 是一个开源的深度研究智能体
框架。自 2.1.0 起，它内置了 OpenSandbox 沙箱 provider，因此智能体调用的 `bash`、文件和
搜索工具在 OpenSandbox 服务器上的沙箱内运行，而不是在
网关宿主机上。

该 provider 位于 DeerFlow 中（`deerflow.community.opensandbox:OpenSandboxProvider`），驱动
OpenSandbox Python SDK。OpenSandbox 侧不需要任何额外配置：本示例
启动一个服务器，将 DeerFlow 的 `sandbox` 配置段指向它，并通过 DeerFlow 自己的
provider 运行沙箱工作。

## 启动 OpenSandbox 服务器 [本地]

启动本地 OpenSandbox 服务器，日志将在终端中可见：

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
opensandbox-server
```

::: info Docker 运行时要求
服务器默认使用 `runtime.type = "docker"`，因此它**必须**能够访问正在运行的
Docker 守护进程。Docker Desktop 用户可通过 `docker version` 确认；在 Colima（macOS）上，
启动服务器前请先导出 `DOCKER_HOST="unix://${HOME}/.colima/default/docker.sock"`。
:::

## 使用 OpenSandbox provider 搭建 DeerFlow

DeerFlow 是一个需要克隆的应用而非 PyPI 库——其 harness 及配套的
`deerflow-extension-api` 都是 uv workspace 成员——因此 provider extra 需要
从检出版本安装。DeerFlow 要求 Python 3.12 或更高版本。

```shell
git clone https://github.com/bytedance/deer-flow.git
cd deer-flow

# optional: the full local workspace setup (backend, frontend, and pre-commit hooks)
make install

# backend dependencies plus the OpenSandbox provider
cd backend && uv sync --all-packages --extra opensandbox && cd ..
```

只有在 `sandbox.use` 选中该 provider 时才会导入它。

::: warning 普通 sync 后需重新添加 extra
`opensandbox` 是一个可选 extra，普通的 `uv sync --locked`（即 `make install`
为后端执行的操作）会将其从环境中剔除。如果后续的 `make install` 或 `uv sync`
移除了它，请再次运行 `cd backend && uv sync --all-packages --extra opensandbox`。
docker-compose 部署不受影响：它在镜像构建时根据 `.env` 中的
`UV_EXTRAS=opensandbox` 安装该 extra。
:::

## 运行示例

该示例直接驱动 DeerFlow 的 provider，因此不需要模型配置，也不需要
网关进程。它写入一个仅包含 `sandbox` 配置段的最小
`config.yaml`，并将 `DEER_FLOW_CONFIG_PATH` 指向它。

使用 DeerFlow 后端环境、从你的 OpenSandbox 检出目录中运行：

```shell
uv run --project ../deer-flow/backend python examples/deer-flow/main.py
```

任何能够 `import deerflow` 的解释器都可以——例如已激活 DeerFlow
检出的 `backend/.venv` 的 shell。

该脚本通过 `provider.acquire()` 创建沙箱，运行一条命令，用 DeerFlow 的文件 API 写入一个 Python
脚本，执行它，然后依次演练 `read_file`、`list_dir`、`glob`、
`grep`，以及仅限 `/mnt/user-data` 的 `download_file`。沙箱被释放回
热池，并在退出时由 `provider.shutdown()` 销毁。

你应该会看到类似以下的输出：

```text
[config] DeerFlow config: /tmp/opensandbox-deerflow-xxxx/config.yaml
[create] DeerFlow sandbox c37d0a7b8a926499 bound to remote b5419ac9-f80b-4263-b403-9bd74ee54c43
[command] Linux-6.6.87.2-microsoft-standard-WSL2-x86_64-with-glibc2.39
[file] Wrote /mnt/user-data/workspace/fib.py
[command] [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
[file] Read back: def fibonacci(count):
[list_dir] ['/mnt/user-data/workspace', '/mnt/user-data/workspace/fib.py']
[glob] ['/mnt/user-data/workspace/fib.py']
[grep] /mnt/user-data/workspace/fib.py:1: def fibonacci(count):
[download] 170 bytes fetched from /mnt/user-data/workspace/fib.py
[cleanup] Sandbox destroyed
```

沙箱 ID、远程 ID 和平台字符串在每次运行中都会不同。

::: tip
DeerFlow 在加载这个生成的配置时会记录 `No models are configured ...`。在仅使用
provider 的路径上，该提示是符合预期的——脚本从不调用模型。
:::

## 将 DeerFlow 智能体指向 OpenSandbox

对于智能体运行，请将 sandbox 配置段放入 DeerFlow 检出的 `config.yaml` 中（`make config`
会基于 `config.example.yaml` 创建一个）：

```yaml
sandbox:
  use: deerflow.community.opensandbox:OpenSandboxProvider
  image: python:3.11
  domain: localhost:8080
  protocol: http
  ready_timeout: 120        # create/readiness deadline; covers a first-run image pull
  sandbox_timeout: 14400    # remote lifetime in seconds; 0 means explicit cleanup only
  bash_command_timeout: 600 # default deadline for commands run in the sandbox
  replicas: 3               # active + warm sandboxes per gateway process
  idle_timeout: 600         # destroy a warm sandbox after this many idle seconds; 0 disables
```

| 选项 | 默认值 | 说明 |
|--------|---------|-------------|
| `image` | `python:3.11` | 沙箱镜像。任何带有 POSIX shell 的镜像都可以。 |
| `domain` | `OPEN_SANDBOX_DOMAIN`，否则 `localhost:8080` | OpenSandbox 服务器地址 |
| `api_key` | `OPEN_SANDBOX_API_KEY` | API key，如果服务器需要鉴权 |
| `protocol` | `http` | 任何非环回域名请使用 `https` |
| `request_timeout` | `30` | 管理 API 请求超时（秒） |
| `ready_timeout` | `30` | 创建与就绪截止时间（秒） |
| `use_server_proxy` | `false` | 通过服务器路由 execd 和文件流量 |
| `sandbox_timeout` | `14400` | 服务器端生命周期（秒）；`0` 禁用续期 |
| `bash_command_timeout` | `600` | 默认命令截止时间（秒） |
| `replicas` | `3` | 每个网关进程的活跃加预热沙箱上限 |
| `idle_timeout` | `600` | 预热沙箱在空闲多少秒后被销毁；`0` 禁用 |
| `environment` | `{}` | 注入沙箱命令的环境变量。以 `$` 开头的值会从网关进程环境中解析。 |

当已导出 `OPEN_SANDBOX_API_KEY` 和 `OPEN_SANDBOX_DOMAIN` 时，`api_key` 和 `domain` 可以
省略，本示例就是通过这种方式传递它们的。当网关
能够访问 OpenSandbox 管理服务但无法直接访问沙箱 `execd` 端点时，
请设置 `use_server_proxy: true`。

然后运行一次无头智能体回合。该提示会让智能体调用其 `bash` 工具，该工具现在
在 OpenSandbox 内执行：

```shell
deerflow --print "Run python3 -c 'print(2 ** 16)' and report the output"
```

`deerflow` 在交互式运行时打开终端工作台，而 `--print` 和 `--json` 运行
一次无头回合。网关和 Web UI 通过 `make dev` 启动，docker-compose
部署通过 `.env` 中的 `UV_EXTRAS=opensandbox` 设置来访问 provider。

## 行为说明

- provider 会为每个有效的 `(user_id, thread_id)` 范围创建一个沙箱，并在每个回合后将其放入
  进程内的热池。只有相同的作用范围可以取回它，且取回前会先运行
  健康检查。
- 创建只有在 SDK 就绪检查和 DeerFlow 的
  `/mnt/user-data/{workspace,uploads,outputs}` 引导都成功后才返回。引导失败会销毁
  新创建的远程沙箱。
- 每次操作都会续期沙箱的服务器端生命周期；未携带显式
  超时的命令使用 `bash_command_timeout`。对同一沙箱的操作是串行化的，因此
  一次短暂的续期不会缩短进行中的长命令的时限。
- `execute_command` 会转发每次调用的环境变量和超时，并在返回的文本中保留 stdout、
  stderr 和非零退出状态。文件读写使用 OpenSandbox 的
  原生文件系统 API。
- `list_dir`、`glob` 和 `grep` 在沙箱内运行可移植的 `find`/`grep` 命令。所有
  路径必须是绝对路径且不允许路径遍历；产物下载被限制在
  `/mnt/user-data` 内。
- 命令路径上的 HTTP 404 或 410、不健康的会话或传输中断都会驱逐失效的
  客户端，以便下次获取时冷启动一个替代沙箱。
- `replicas` 是每个网关进程的软上限：DeerFlow 尚未在进程之间协调沙箱所有权，
  因此每个基于 OpenSandbox 的部署使用一个网关进程才能让该上限
  有意义。

## 环境变量

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `SANDBOX_DOMAIN` | `localhost:8080` | 服务器地址（主机及可选端口） |
| `SANDBOX_PROTOCOL` | `http` | 访问服务器使用的协议（`http` 或 `https`） |
| `SANDBOX_API_KEY` | _（本地可选）_ | 服务器需要鉴权时使用的 API key |
| `SANDBOX_IMAGE` | `python:3.11` | 本示例使用的沙箱镜像 |
| `OPEN_SANDBOX_DOMAIN` | _（未设置）_ | DeerFlow 配置省略 `domain` 时的服务器地址 |
| `OPEN_SANDBOX_API_KEY` | _（未设置）_ | DeerFlow 配置省略 `api_key` 时的 API key |

## 参考资料

- [DeerFlow](https://github.com/bytedance/deer-flow) - 开源深度研究智能体框架
- [DeerFlow OpenSandbox provider](https://github.com/bytedance/deer-flow/blob/main/backend/packages/harness/deerflow/community/opensandbox/README.md) - Provider 文档
- [deer-flow PR #4877](https://github.com/bytedance/deer-flow/pull/4877) - OpenSandbox provider 贡献
- [OpenSandbox Python SDK](https://pypi.org/project/opensandbox/)
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/deer-flow)
