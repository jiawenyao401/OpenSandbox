---
title: Python SDK
description: 用于创建、管理安全 OpenSandbox 环境并与之交互的 Python SDK。
---

# OpenSandbox Python SDK

使用异步或同步 Python API 创建 sandbox、运行命令和管理文件。

## 安装

### pip

```bash
pip install opensandbox
```

### uv

```bash
uv add opensandbox
```

## 快速开始

以下示例展示如何创建 sandbox 并执行 shell 命令。

::: tip
运行此示例前，请确保 OpenSandbox 服务已在运行。启动说明参见[快速上手](/getting-started/)指南。
:::

```python
import asyncio
from opensandbox.sandbox import Sandbox
from opensandbox.config import ConnectionConfig
from opensandbox.exceptions import SandboxException

async def main():
    # 1. Configure connection
    config = ConnectionConfig(
        domain="api.opensandbox.io",
        api_key="your-api-key"
    )

    # 2. Create a Sandbox
    try:
        sandbox = await Sandbox.create(
            "ubuntu",
            connection_config=config
        )
        try:
            # 3. Execute a shell command
            execution = await sandbox.commands.run("echo 'Hello Sandbox!'")

            # 4. Print output
            print(execution.logs.stdout[0].text)
        finally:
            # 5. Terminate the remote sandbox and close local resources
            await sandbox.destroy()

    except SandboxException as e:
        # Handle Sandbox specific exceptions
        print(f"Sandbox Error: [{e.error.code}] {e.error.message}")
        # Server logs can be correlated by this request id (if available)
        print(f"Request ID: {e.request_id}")
    except Exception as e:
        print(f"Error: {e}")

if __name__ == "__main__":
    asyncio.run(main())
```

### 同步快速开始

如果你偏好同步 API，请使用 `SandboxSync` / `SandboxManagerSync` 和 `ConnectionConfigSync`：

```python
from datetime import timedelta

from opensandbox import SandboxSync
from opensandbox.config import ConnectionConfigSync

config = ConnectionConfigSync(
    domain="api.opensandbox.io",
    api_key="your-api-key",
    request_timeout=timedelta(seconds=30),
)

sandbox = SandboxSync.create("ubuntu", connection_config=config)
try:
    execution = sandbox.commands.run("echo 'Hello Sandbox!'")
    print(execution.logs.stdout[0].text)
finally:
    sandbox.destroy()
```

`destroy()` 适用于创建-使用-丢弃的工作流。它会先调用 `kill()` 再调用 `close()`，并且在远程终止失败时仍会关闭本地资源。上下文管理器继续只调用 `close()`，因此远程 sandbox 仍然保持可用，可供后续 `connect()` 使用，除非你显式 kill 或 destroy 它。

## Client Pool 与可观测性

同步应用使用 `SandboxPoolSync`，asyncio 使用 `SandboxPoolAsync`。两者都提供内存与 Redis 存储、四种获取策略以及分阶段 warmup 控制。示例、配置、清理和分布式部署参见 [Client Pool](/guides/client-pool)。

启用 `ConnectionConfig(enable_tracing=True)` 或 `ConnectionConfigSync(enable_tracing=True)` 以获得 [pool warmup 追踪](/sdks/observability#pool-warmup-tracing)。远程日志/事件请使用[诊断](/api/#diagnostics) manager API。创建延迟上报由 [SDK 遥测](/sdks/observability#creation-metrics)单独控制。

## 生命周期钩子

创建 sandbox 时传入 `SandboxLifecycle`。`pre_start` 在入口点启动之前完成，而 `periodic` 钩子在启动后按各自的计划运行。

```python
from opensandbox.models.sandboxes import (
    LifecycleHook,
    PeriodicLifecycleHook,
    SandboxLifecycle,
)

sandbox = await Sandbox.create(
    "ubuntu:24.04",
    connection_config=config,
    lifecycle=SandboxLifecycle(
        pre_start=LifecycleHook(
            command=["sh", "-c", "echo ready > /tmp/prestart.done"],
            timeout_seconds=120,
        ),
        periodic=[
            PeriodicLifecycleHook(
                name="checkpoint",
                schedule="@every 5m",
                command=["sh", "-c", "date -u >> /tmp/checkpoints.log"],
                timeout_seconds=120,
            )
        ],
    ),
)
```

Server 会校验 `timeout_seconds`；`pre_start` 接受 1–10800 秒，而 `periodic` 接受 1–300 秒。两者在省略时均默认为 60 秒。计时、失败行为和 provider 限制参见[生命周期钩子](/guides/lifecycle-hooks)。

### 空闲策略（自动暂停与访问唤醒）

`SandboxLifecycle.idle_policy`（OSEP-0024，仅 fast-sandbox）告诉控制面在 ingress 观测到连续空闲流量达到 `idle_timeout_seconds`（30–86400）后自动暂停 sandbox，并在下一个请求到来时唤醒它。它属于控制面状态：永远不会到达 guest，在模板模式（`Sandbox.create_from_template(idle_policy=...)`）下被允许，并被非 fast-sandbox 后端拒绝。省略它则完全保持非自动化行为。

```python
sandbox = await Sandbox.create(
    "ubuntu:24.04",
    connection_config=config,
    timeout_seconds=86400,
    lifecycle=SandboxLifecycle(
        idle_policy=IdlePolicy(
            idle_timeout_seconds=300,
            on_idle="pause",
            wake_on_access=True,
        )
    ),
)
```

`active_probe` 让应用在真实工作运行但不产生 ingress 流量时否决暂停：声明一个以 `{"active": bool, "message"?: string}` 应答的端点，当 sandbox 除此之外处于空闲状态时 server 会轮询它；忙碌的应答会重置空闲时钟。

```python
IdlePolicy(
    idle_timeout_seconds=300,
    on_idle="pause",
    active_probe=ActivityProbe(port=8080, path="/active", timeout_seconds=2),
)
```

### 唤醒重试

对已暂停 sandbox 的请求可能在其恢复仍在进行时到达。网关以 `OpenSandbox-Wake: expired` 标记应答 `503`；SDK 识别该标记并在 `Retry-After` 延迟之后重发原始请求（对任何方法都安全 —— 该标记证明请求从未被转发）。唤醒预算默认在所有唤醒周期内为 5 次尝试 / 60 秒；预算耗尽时 SDK 抛出 `SandboxWakeTimeoutException`（携带 `attempts` 和 `waited`），而 server 侧的恢复在后台继续。裸 `503` 响应遵循既有重试规则、不受影响。流式（SSE）调用绕过唤醒重试。

## 使用示例

以下代码片段使用快速开始中的 `config` 和存活的 `sandbox`。异步片段请放在 async 函数内部、其清理块之前运行。各创建示例互为替代；每个 sandbox 不再需要时调用 `destroy()`。同步 API 请使用 `SandboxSync` 和 `SandboxManagerSync`，并省略 SDK 调用和上下文管理器上的 `await` / `async`。

### 1. 生命周期管理

管理 sandbox 生命周期，包括续期、暂停和恢复。

```python
import time
from datetime import timedelta

# Renew the sandbox
# This resets the expiration time to (current time + duration)
await sandbox.renew(timedelta(minutes=30))

# Request pause (runtime-dependent)
await sandbox.pause()

deadline = time.monotonic() + 120
while True:
    if time.monotonic() >= deadline:
        raise TimeoutError("Sandbox did not pause within 120 seconds")
    info = await sandbox.get_info()
    if info.status.state == "Paused":
        break
    if info.status.state == "Failed":
        raise RuntimeError(info.status.message)
    await asyncio.sleep(1)

# Resume creates a new local handle.
resumed = await Sandbox.resume(sandbox_id=sandbox.id, connection_config=config)
await sandbox.close()
sandbox = resumed

# Get current status
info = await sandbox.get_info()
print(f"State: {info.status.state}")
print(f"Expires: {info.expires_at}")  # None when no automatic expiration is configured
```

暂停是异步的且取决于运行时。参见[暂停与恢复](/guides/pause-resume)。使用 `await Sandbox.connect(sandbox_id, connection_config=config)` 可以在不恢复的情况下附加到已在运行的 sandbox。

通过显式传入 `timeout=None` 创建不过期的 sandbox。省略 `timeout` 则使用默认的 10 分钟 TTL：

```python
manual = await Sandbox.create(
    "ubuntu",
    connection_config=config,
    timeout=None,
)
```

### 2. 自定义健康检查

解析端点只能确认路由存在；它不能确认该端口上的应用是健康的。对于服务就绪，应对应用的健康端点发起有边界的请求，并携带返回的端点头。

使用内置健康探针时，创建、连接和恢复期间的就绪检查在健康端点返回 HTTP 401 或 403 时会立即失败。SDK 会以原始状态、错误详情和请求 ID 抛出 `SandboxApiException`，而不是等待 `SandboxReadyTimeoutException`。重试之前请检查端点凭证或权限。瞬时健康失败保持既有的轮询行为；内置健康探针失败时 `is_healthy()` 仍返回 `False`。

自定义判断 sandbox 是否健康的逻辑。这会覆盖默认的 ping 检查。同步检查必须自行设置超时，因为 SDK 无法中断它们；异步检查不得阻塞事件循环或吞掉取消。

```python
import httpx

async def custom_health_check(sbx: Sandbox) -> bool:
    endpoint = await sbx.get_endpoint(80)
    url = f"{sbx.connection_config.protocol}://{endpoint.endpoint}/"
    try:
        async with httpx.AsyncClient(timeout=2) as client:
            response = await client.get(url, headers=endpoint.headers)
        return response.status_code == 200
    except httpx.RequestError:
        return False

sandbox = await Sandbox.create(
    "nginx:latest",
    entrypoint=["nginx", "-g", "daemon off;"],
    connection_config=config,
    health_check=custom_health_check,
)
```

### 3. 命令执行与流式输出

实时执行命令并处理输出流。

```python
from opensandbox.models.execd import ExecutionHandlers, RunCommandOpts

# Define async handlers for streaming output
async def handle_stdout(msg):
    print(f"STDOUT: {msg.text}")

async def handle_stderr(msg):
    print(f"STDERR: {msg.text}")

async def handle_complete(complete):
    print(f"Command finished in {complete.execution_time_in_millis}ms")

# Create handlers (all handlers must be async)
handlers = ExecutionHandlers(
    on_stdout=handle_stdout,
    on_stderr=handle_stderr,
    on_execution_complete=handle_complete
)

# Execute command with handlers
result = await sandbox.commands.run(
    "for i in {1..5}; do echo \"Count $i\"; sleep 0.5; done",
    handlers=handlers
)
```

要执行不带 shell 解析的原生程序，请传入参数列表。在 Linux 上，此示例打印字面的 `$HOME` 并将 `hello world` 作为一个参数保留：

```python
result = await sandbox.commands.run(["printf", "%s\n", "$HOME", "hello world"])
```

原生 argv 执行需要更新版本的 execd。可执行文件查找与平台行为参见[命令执行模式](/architecture/data-plane/execd#command-execution)。

#### 后台命令

启动一条有边界的命令、读取增量日志并检查其退出状态。命令超时与 sandbox TTL 是相互独立的设置。

```python
import time
from datetime import timedelta
from opensandbox.models.execd import RunCommandOpts

execution = await sandbox.commands.run(
    'for i in 1 2 3; do echo "step $i"; sleep 1; done',
    opts=RunCommandOpts(background=True, timeout=timedelta(seconds=30)),
)
if not execution.id:
    raise RuntimeError("No command ID returned")
cursor = 0
deadline = time.monotonic() + 45
while True:
    if time.monotonic() >= deadline:
        await sandbox.commands.interrupt(execution.id)
        raise TimeoutError("Command did not finish")
    status = await sandbox.commands.get_command_status(execution.id)
    logs = await sandbox.commands.get_background_command_logs(execution.id, cursor)
    print(logs.content, end="")
    cursor = logs.cursor if logs.cursor is not None else cursor
    if status.running is False:
        if status.exit_code != 0:
            raise RuntimeError(f"Command failed: {status.exit_code}, {status.error}")
        break
    await asyncio.sleep(0.5)
```

#### 持久 shell 会话

使用 Bash 会话可跨命令保留 shell 变量和工作目录。用完后请删除会话。

```python
session_id = await sandbox.commands.create_session(working_directory="/tmp")
try:
    await sandbox.commands.run_in_session(session_id, "export DEMO=hello")
    result = await sandbox.commands.run_in_session(session_id, 'echo "$DEMO"; pwd')
    print("".join(message.text for message in result.logs.stdout))
finally:
    await sandbox.commands.delete_session(session_id)
```

#### 持久环境变量

设置由运行时注入到每个后续命令和会话中的环境变量 —— 无需手写 shell 转义去操作 sandbox env 文件。

```python
await sandbox.commands.set_env("MY_TOKEN", "it's a safe value")
```

键必须匹配 `[A-Za-z_][A-Za-z0-9_]*`。不含单引号的值按原样存储；含单引号的值使用 env 文件的双引号形式，其中 shell 风格的 `$NAME` 序列可能在运行时加载该文件时被展开。env 文件是追加式的：同一键的最后一次写入生效。如果 sandbox 未能持久化该变量，将抛出 `SandboxException`。同步 API 在 `sandbox.commands` 上暴露相同的方法。

关于 sandbox 内需要文件系统/进程隔离的命令，参见[隔离会话](/guides/isolation-sessions)。它们与 Bash 会话是相互独立的概念。

### 4. 文件操作

管理文件和目录，包括读取、写入、列出、删除和搜索。

```python
from opensandbox.models.filesystem import DirectoryListEntry, WriteEntry, SearchEntry

# 1. Write file
await sandbox.files.write_files([
    WriteEntry(
        path="/tmp/hello.txt",
        data="Hello World",
        mode=644
    )
])

# 2. Read file
content = await sandbox.files.read_file("/tmp/hello.txt")
print(f"Content: {content}")

# List immediate children; search filters by a filename pattern.
entries = await sandbox.files.list_directory(DirectoryListEntry(path="/tmp", depth=1))
print([entry.path for entry in entries])

# 3. Search files
files = await sandbox.files.search(
    SearchEntry(
        path="/tmp",
        pattern="*.txt"
    )
)
for f in files:
    print(f"Found: {f.path}")

# 4. Delete file
await sandbox.files.delete_files(["/tmp/hello.txt"])
```

对于二进制数据，请向 `WriteEntry.data` 传入 `bytes` 并使用 `read_bytes()`；大文件下载使用 `read_bytes_stream()`。`read_file()` 和 `read_bytes()` 还接受 `offset` 和 `limit` 实现部分读取。

#### 带显式身份的文件操作

将文件系统客户端绑定到 Linux UID 和 GID：

```python
files = sandbox.files_with_identity(1001, 2000)
```

异步与同步 Sandbox 客户端都暴露该辅助方法。

绑定的客户端暴露常规文件操作，并共享 sandbox 连接、认证和端点路由头。既有文件客户端保持其默认身份。命令执行时请单独选择 UID/GID。

这需要 Linux 以及支持文件系统执行身份的 Execd 版本。不支持的服务器会返回错误；请求绝不会回退到默认文件系统路由。请将 sandbox 凭证和身份选择保留在可信后端。Linux 文件系统权限依然生效；该特性并不创建租户隔离边界。权限与 workspace 供给详情参见 [Execd](/architecture/data-plane/execd)。

### 5. Sandbox 管理（管理面）

使用 `SandboxManager` 执行管理任务和查找既有 sandbox。

```python
from opensandbox.manager import SandboxManager
from opensandbox.models.sandboxes import SandboxFilter

# Create manager using async context manager
async with await SandboxManager.create(connection_config=config) as manager:

    # First page only; increase page to retrieve subsequent pages.
    sandboxes = await manager.list_sandbox_infos(
        SandboxFilter(
            states=["Running"],
            page_size=10
        )
    )

    for info in sandboxes.sandbox_infos:
        print(f"Found sandbox: {info.id}")
```

### 资源指标

使用 `await sandbox.get_metrics()` 读取当前 sandbox 的资源使用。这与 [SDK 创建遥测](/sdks/observability#creation-metrics)相互独立。

## 快照、模板与元数据

| 操作 | 公共 API |
| --- | --- |
| 为 sandbox 创建快照 | `sandbox.create_snapshot(name=...)` 或 `manager.create_snapshot(sandbox_id, name=...)` |
| 查看/列出/删除快照 | `manager.get_snapshot`、`list_snapshots`、`delete_snapshot` |
| 恢复快照 | `Sandbox.create(snapshot_id=..., connection_config=config)` |
| 管理 Fsb 模板 | `manager.create_template`、`get_template`、`list_templates`、`delete_template` |
| 从已发布模板创建 | `Sandbox.create_from_template(template_id, timeout=..., connection_config=config)` |
| 修补元数据 | `sandbox.patch_metadata` 或 `manager.patch_sandbox_metadata` |

这些 API 在同步 SDK 上同样存在。快照创建和模板构建是异步的：恢复或使用模板之前请先检查状态。模板必须达到 `Succeeded`；基于模板的创建需要 TTL，并从模板继承环境、资源、卷和生命周期钩子。元数据修补值会新增/替换键；`None` 删除键。后端约束参见[生命周期契约](/api/#1-sandbox-lifecycle-yml)。

快照支持取决于运行时与 server 配置。如果源 sandbox 的剩余 TTL 可能在快照创建期间到期，请先续期。此示例最多等待 15 分钟、恢复一个新 sandbox，并保留快照以便复用：

```python
import time
from opensandbox.manager import SandboxManager

async with await SandboxManager.create(connection_config=config) as manager:
    snapshot = await sandbox.create_snapshot(name="demo")
    print("Snapshot:", snapshot.id)
    deadline = time.monotonic() + 900
    while True:
        if time.monotonic() >= deadline:
            raise TimeoutError(f"Snapshot {snapshot.id} is not ready")
        snapshot = await manager.get_snapshot(snapshot.id)
        if snapshot.status.state == "Ready":
            break
        if snapshot.status.state == "Failed":
            raise RuntimeError(snapshot.status.message)
        await asyncio.sleep(2)
    restored = await Sandbox.create(snapshot_id=snapshot.id, connection_config=config)
    try:
        print(restored.id)
    finally:
        await restored.destroy()
    # When no longer needed: await manager.delete_snapshot(snapshot.id)
```

在模板构建达到 `Succeeded` 后从既有模板创建：

```python
from datetime import timedelta

templated = await Sandbox.create_from_template(
    "your-published-template-id",
    timeout=timedelta(minutes=10),
    connection_config=config,
)
```

在运行中的 sandbox 上新增、替换或移除元数据：

```python
await sandbox.patch_metadata({"project": "demo", "obsolete-key": None})
```

## 配置

### 1. 连接配置

`ConnectionConfig` 类管理 API server 的连接设置。

| 参数 | 说明 | 默认值 | 环境变量 |
| ----------------- | ------------------------------------------ | ---------------------------- | ---------------------- |
| `api_key` | 用于认证的 API Key | 可选；启用 server 认证时需要 | `OPEN_SANDBOX_API_KEY` |
| `domain` | sandbox 服务的端点域 | `localhost:8080` | `OPEN_SANDBOX_DOMAIN`  |
| `protocol` | HTTP 协议（http/https） | `http`                       | -                      |
| `request_timeout` | API 请求超时 | 30 秒 | -                      |
| `debug` | 启用 HTTP 请求的调试日志 | `False`                      | -                      |
| `headers` | 自定义 HTTP 头 | 空 | -                      |
| `follow_redirects` | 为 SDK 请求跟随 HTTP 重定向 | `False` | -                      |
| `event_hooks` | 适配器客户端的额外 httpx 钩子 | 空 | -                      |
| `transport` | 共享的 httpx transport（pool/代理/重试）；自定义 transport 必须遵循请求超时 | 每个实例由 SDK 创建 | -                      |
| `retry_policy` | 非流式请求的自动重试策略（参见[自动重试](#_2-automatic-retries)） | 启用（`RetryPolicy()`） | -                 |
| `use_server_proxy` | 将 sandbox server 用作 execd/端点请求的代理（例如客户端无法直接访问 sandbox 时） | `False` | -                      |
| `disable_metrics` | 禁用 SDK 创建延迟遥测（参见 [SDK 遥测](/sdks/observability#creation-metrics)） | `False` | `OPENSANDBOX_DISABLE_METRICS` |
| `enable_tracing` | 为 pool warmup 启用 OpenTelemetry 追踪（参见 [SDK 追踪](/sdks/observability#pool-warmup-tracing)） | `False` | - |

Python Code Interpreter 继承 sandbox 的连接配置。设置 `use_server_proxy=True` 时，上下文请求和流式代码执行会以 `OPEN-SANDBOX-API-KEY` 携带所配置的 API 密钥供 server 认证。在直连模式下，SDK 不会自动将该密钥附加到代码执行请求。这同时适用于异步与同步 API。

启用 `follow_redirects` 时，同源重定向会保留请求头。源是 scheme、主机和端口的组合，因此任一变化都是跨源的。在跟随跨源重定向之前，SDK 会移除所有名称以 `OPEN-SANDBOX-` 或 `OPENSANDBOX-` 开头（不区分大小写）的头。其他自定义头不会被自动剥离。

文件上传从不跟随重定向，即使启用了 `follow_redirects`，因为流式或基于文件的多部分请求体不总是能安全重放。这同时适用于直接分块上传和经 server 代理的上传。上传收到的重定向响应会以原始 3xx 状态作为 `SandboxApiException` 呈现。

`ConnectionConfig.event_hooks` 接受异步 httpx 钩子，而 `ConnectionConfigSync.event_hooks` 接受同步钩子。配置的请求钩子在 SDK 安全钩子之前运行，因此它们无法向跨源请求重新添加受保护的 OpenSandbox 头。生命周期遥测遵循 `follow_redirects` 和 SDK 安全钩子，但不会调用配置的用户钩子。

```python
from datetime import timedelta

# 1. Basic configuration
config = ConnectionConfig(
    api_key="your-key",
    domain="api.opensandbox.io",
    request_timeout=timedelta(seconds=60)
)

# 2. Advanced: Custom headers and custom transport
# If you create many Sandbox instances, configuring a shared transport is recommended to optimize resource usage.
# SDK default keep-alive is 30 seconds for its own transports.
import httpx

config = ConnectionConfig(
    api_key="your-key",
    domain="api.opensandbox.io",
    headers={
        "X-Custom-Header": "value",
        "X-Request-ID": "trace-123",
    },
    transport=httpx.AsyncHTTPTransport(
        limits=httpx.Limits(
            max_connections=100,
            max_keepalive_connections=50,
            keepalive_expiry=30.0,
        )
    ),
)

# If you provide a custom transport, you are responsible for closing it:
# await config.transport.aclose()
```

### 2. 自动重试

SDK 会自动重试瞬时故障。`ConnectionConfig` / `ConnectionConfigSync` 会在默认共享 transport 之外安装一个重试包装器，由 `retry_policy`（`opensandbox.transport.RetryPolicy`）控制。

默认行为：

- **默认启用。** 幂等方法（`GET/HEAD/PUT/DELETE/OPTIONS`）在 `429`、`502`、`503` 以及发送前的传输失败（DNS、TCP 连接、TLS 握手、新建连接被重置）时会被重试。
- **`POST`/`PATCH` 默认绝不因状态码被重试**，因为请求可能已在 server 侧生效。这些方法在发送前的传输失败（尚未写出任何字节）时仍会被重试。
- 最多 `3` 次重试，采用去相关抖动的指数退避，并遵循 server 的 `Retry-After` 头（上限 60 秒）。
- **SSE / 流式请求完全绕过重试**（请求体不可重放）。

::: warning 行为变更
重试默认开启。与较早的 SDK 版本相比，这可能增加 HTTP 尝试次数和尾延迟。如果你依赖快速失败语义，请用 `RetryPolicy.disabled()` 显式退出。
:::

```python
from datetime import timedelta

from opensandbox.transport import RetryPolicy

# Fast-fail: never retry (also skips fresh-connection recovery).
config = ConnectionConfig(
    api_key="your-key",
    domain="api.opensandbox.io",
    retry_policy=RetryPolicy.disabled(),
)

# Custom policy: more retries, an overall wall-clock deadline, and an
# opt-in to retry POST/PATCH on 503 (only safe if your endpoints are
# idempotent).
from http import HTTPStatus

config = ConnectionConfig(
    api_key="your-key",
    domain="api.opensandbox.io",
    retry_policy=RetryPolicy(
        max_retries=5,
        overall_deadline=timedelta(seconds=20),
        retryable_status_codes_non_idempotent=frozenset(
            {HTTPStatus.SERVICE_UNAVAILABLE}
        ),
    ),
)
```

::: info
如果你传入自定义 `transport`，SDK **不会**包装它；安装重试行为将是你的责任。
:::

### 3. Sandbox 创建配置

`Sandbox.create()` 允许配置 sandbox 环境。

| 参数 | 说明 | 默认值 |
| --------------- | ---------------------------------------- | ------------------------------- |
| `image` | Docker 镜像规格 | image 或快照 ID 二选一 |
| `timeout` | 自动终止超时 | 10 分钟 |
| `entrypoint` | 容器入口命令 | `["tail", "-f", "/dev/null"]`   |
| `resource` | CPU 与内存限制 | `{"cpu": "1", "memory": "2Gi"}` |
| `env` | 环境变量 | 空 |
| `metadata` | 自定义元数据标签 | 空 |
| `network_policy` | 可选的出站网络策略（egress） | -                             |
| `credential_proxy` | 可选的 Credential Vault 代理启动设置 | -                     |
| `ready_timeout` | 端点发布与健康检查的总预算 | 30 秒 |
| `snapshot_id` | 恢复快照而不是传 `image` | - |
| `resource_requests` | Kubernetes 资源请求；不得超过限制 | - |
| `lifecycle` | 启动前与周期性钩子 | - |
| `platform` | OS/架构约束 | - |
| `volumes` | Host、PVC 或 OSSFS 挂载 | - |
| `secure_access` | 要求端点访问凭证 | `False` |
| `skip_health_check` | 跳过健康检查；仍会等待端点发布 | `False` |

::: warning
`opensandbox.io/` 下的元数据键保留给系统管理的标签，将被 server 拒绝。
:::

```python
from datetime import timedelta

from opensandbox.models.sandboxes import NetworkPolicy, NetworkRule

sandbox = await Sandbox.create(
    "python:3.11",
    connection_config=config,
    timeout=timedelta(minutes=30),
    resource={"cpu": "2", "memory": "4Gi"},
    env={"PYTHONPATH": "/app"},
    metadata={"project": "demo"},
    network_policy=NetworkPolicy(
        defaultAction="deny",
        egress=[NetworkRule(action="allow", target="pypi.org")],
    ),
)
```

### 4. 运行时 Egress 策略更新

运行时 egress 策略路由取决于 sandbox 的来源。对基于镜像的 sandbox，SDK 解析端口 `18080` 并调用 sidecar 的 `/policy` API。对基于模板的 sandbox（包括恢复自模板的快照），SDK 检测 `OPEN-SANDBOX-ORIGIN: template` 并通过生命周期 `/sandboxes/{sandboxId}/networkpolicy` API 路由策略操作。

修补使用合并语义：
- 传入规则优先于具有相同 `target` 的既有规则。
- 其他目标的既有规则保持不变。
- 在单个修补载荷内，某个 `target` 的第一条规则生效。
- 当前的 `defaultAction` 被保留。

```python
from opensandbox.models.sandboxes import NetworkRule

policy = await sandbox.get_egress_policy()

await sandbox.patch_egress_rules(
    [
        NetworkRule(action="allow", target="www.github.com"),
        NetworkRule(action="deny", target="pypi.org"),
    ]
)
```

### 5. Credential Vault

Credential Vault 需要 sandbox 侧的 egress 服务，对基于模板的 sandbox 不可用。它从 egress sidecar 注入出站凭证，同时让真实密钥远离 sandbox 的环境变量、命令、文件和日志。以启用 `credential_proxy` 的方式创建 sandbox，然后通过 `sandbox.credential_vault` 写入凭证和绑定。

```python
from opensandbox.models.sandboxes import (
    Credential,
    CredentialBinding,
    CredentialProxyConfig,
    NetworkPolicy,
    NetworkRule,
)

sandbox = await Sandbox.create(
    "python:3.11",
    connection_config=config,
    network_policy=NetworkPolicy(
        defaultAction="deny",
        egress=[NetworkRule(action="allow", target="api.example.com")],
    ),
    credential_proxy=CredentialProxyConfig(enabled=True),
)

await sandbox.credential_vault.create(
    credentials=[Credential(name="api-token", source={"value": "<token>"})],
    bindings=[
        CredentialBinding(
            name="api-token",
            match={
                "schemes": ["https"],
                "hosts": ["api.example.com"],
                "paths": ["/v1/*"],
            },
            auth={"type": "apiKey", "name": "x-api-key", "credential": "api-token"},
        )
    ],
)
```

认证类型、绑定指南以及 Git/curl 示例参见 [Credential Vault](/guides/credential-vault)。
