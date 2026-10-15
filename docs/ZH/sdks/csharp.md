---
title: C# SDK
description: 用于创建、管理安全 OpenSandbox 环境并与之交互的 C# SDK。
---

# OpenSandbox C\# SDK

一个用于与 OpenSandbox 进行低层交互的 C# SDK。它提供了创建、管理安全 sandbox 环境并与之交互的能力，包括执行 shell 命令、管理文件和读取资源指标。

## 安装

### NuGet

```bash
dotnet add package Alibaba.OpenSandbox
```

### 程序包管理器

```powershell
Install-Package Alibaba.OpenSandbox
```

## 快速开始

在启用了隐式 using 的 .NET 8+ 控制台项目中运行此示例。它会创建一个 sandbox、运行一条 shell 命令，并释放远程与本地资源。

::: tip
运行此示例前，请确保 OpenSandbox 服务已在运行。启动说明参见[快速上手](/getting-started/)指南。
:::

```csharp
using OpenSandbox;
using OpenSandbox.Config;
using OpenSandbox.Core;

var config = new ConnectionConfig(new ConnectionConfigOptions
{
    Domain = "api.opensandbox.io",
    ApiKey = "your-api-key",
    // Protocol = ConnectionProtocol.Https,
    // RequestTimeoutSeconds = 60,
});

try
{
    await using var sandbox = await Sandbox.CreateAsync(new SandboxCreateOptions
    {
        ConnectionConfig = config,
        Image = "ubuntu",
        TimeoutSeconds = 10 * 60,
    });

    try
    {
        var execution = await sandbox.Commands.RunAsync("echo 'Hello Sandbox!'");
        Console.WriteLine(execution.Logs.Stdout.FirstOrDefault()?.Text);
    }
    finally
    {
        await sandbox.KillAsync();
    } // await using releases the client even if KillAsync fails.
}
catch (SandboxException ex)
{
    Console.Error.WriteLine($"Sandbox Error: [{ex.Error.Code}] {ex.Error.Message}");
    Console.Error.WriteLine($"Request ID: {ex.RequestId}");
}
```

## 生命周期钩子

在 `SandboxCreateOptions` 中设置 `Lifecycle`。`PreStart` 在入口点启动之前完成，而 `Periodic` 钩子在启动后按各自的计划运行。

```csharp
using OpenSandbox.Models;

await using var sandbox = await Sandbox.CreateAsync(new SandboxCreateOptions
{
    ConnectionConfig = config,
    Image = "ubuntu:24.04",
    Lifecycle = new SandboxLifecycle
    {
        PreStart = new LifecycleHook
        {
            Command = new[] { "sh", "-c", "echo ready > /tmp/prestart.done" },
            TimeoutSeconds = 120,
        },
        Periodic = new[]
        {
            new PeriodicLifecycleHook
            {
                Name = "checkpoint",
                Schedule = "@every 5m",
                Command = new[] { "sh", "-c", "date -u >> /tmp/checkpoints.log" },
                TimeoutSeconds = 120,
            },
        },
    },
});
```

Server 会校验 `TimeoutSeconds`；`PreStart` 接受 1–10800 秒，而 `Periodic` 接受 1–300 秒。两者在省略时均默认为 60 秒。计时、失败行为和 provider 限制参见[生命周期钩子](/guides/lifecycle-hooks)。

### 空闲策略（自动暂停与访问唤醒）

`SandboxLifecycle.IdlePolicy`（OSEP-0024，仅 fast-sandbox）告诉控制面在 ingress 观测到连续空闲流量达到 `IdleTimeoutSeconds`（30–86400）后自动暂停 sandbox，并在下一个请求到来时唤醒它。它属于控制面状态：永远不会到达 guest，在模板模式（`SandboxCreateFromTemplateOptions.IdlePolicy`）下被接受，并被非 fast-sandbox 后端拒绝。省略它则完全保持非自动化行为。

```csharp
var sandbox = await Sandbox.CreateAsync(new SandboxCreateOptions
{
    ConnectionConfig = config,
    Image = "ubuntu:24.04",
    Lifecycle = new SandboxLifecycle
    {
        IdlePolicy = new IdlePolicy
        {
            IdleTimeoutSeconds = 300,
            OnIdle = OnIdleAction.Pause,
            WakeOnAccess = true
        }
    }
});
```

`ActiveProbe` 让应用在真实工作运行但不产生 ingress 流量时否决暂停：声明一个以 `{"active": bool, "message"?: string}` 应答的端点，当 sandbox 除此之外处于空闲状态时 server 会轮询它；忙碌的应答会重置空闲时钟。无效策略会通过 `IdlePolicy.Validate()` 在客户端快速失败。

### 唤醒重试

对已暂停 sandbox 的请求可能在其恢复仍在进行时到达。网关以 `OpenSandbox-Wake: expired` 头应答 `503`；HTTP 传输识别该标记并在 `Retry-After` 延迟之后重发原始请求（对任何方法都安全 —— 该标记证明请求从未被转发）。唤醒预算默认在所有唤醒周期内为 5 次尝试 / 60 秒；预算耗尽时 SDK 抛出 `SandboxWakeTimeoutException`（携带 `Attempts` 和 `Waited`），而 server 侧的恢复在后台继续。裸 `503` 响应遵循既有规则、不受影响。可在连接配置上调整：`WakeRetryEnabled`（默认 true）、`WakeMaxAttempts`（5）、`WakeOverallDeadlineSeconds`（60，非正值表示无上限）。SSE 和文件流传输绕过唤醒重试。

## 使用示例

以下代码片段使用快速开始中的 `config` 和存活的 `sandbox`。请在快速开始的清理块之前运行它们，并在文件顶部加上 `using OpenSandbox.Models;`。各创建示例互为替代。`await using` 释放本地客户端；对不再需要的 sandbox 调用 `KillAsync()` 终止。

### 1. 生命周期管理

管理 sandbox 生命周期，包括续期、暂停和恢复。

```csharp
var info = await sandbox.GetInfoAsync();
Console.WriteLine($"State: {info.Status.State}");
Console.WriteLine($"Created: {info.CreatedAt}");
Console.WriteLine($"Expires: {info.ExpiresAt}"); // null when manual cleanup mode is used

await sandbox.PauseAsync();

using var pauseBudget = new CancellationTokenSource(TimeSpan.FromMinutes(2));
while (true)
{
    var current = await sandbox.GetInfoAsync(pauseBudget.Token);
    if (current.Status.State == SandboxStates.Paused) break;
    if (current.Status.State == "Failed")
        throw new InvalidOperationException(current.Status.Message);
    await Task.Delay(1000, pauseBudget.Token);
}

// Resume returns a fresh local handle.
await using var resumed = await sandbox.ResumeAsync();

// Renew: expiresAt = now + timeoutSeconds
await resumed.RenewAsync(30 * 60);
```

暂停是异步的且取决于运行时。参见[暂停与恢复](/guides/pause-resume)。

通过设置 `ManualCleanup = true` 创建不过期的 sandbox：

```csharp
var manual = await Sandbox.CreateAsync(new SandboxCreateOptions
{
    ConnectionConfig = config,
    Image = "ubuntu",
    ManualCleanup = true,
});
```

::: info
`TimeoutSeconds = null` 使用默认 TTL。设置 `ManualCleanup = true` 可禁用自动过期。
:::

### 连接到既有 Sandbox

当你已持有 sandbox ID 并需要一个新的绑定到它的 SDK 实例时，使用 `ConnectAsync`。

```csharp
await using var connected = await Sandbox.ConnectAsync(new SandboxConnectOptions
{
    SandboxId = "existing-sandbox-id",
    ConnectionConfig = config
});
```

### 2. 自定义健康检查

解析端点只能确认路由存在；它不能确认该端口上的应用是健康的。对于服务就绪，应对应用的健康端点发起有边界的请求，并携带返回的端点头。

自定义判断 sandbox 是否就绪/健康的逻辑。

```csharp
using var http = new HttpClient { Timeout = TimeSpan.FromSeconds(2) };
await using var sandbox = await Sandbox.CreateAsync(new SandboxCreateOptions
{
    ConnectionConfig = config,
    Image = "nginx:latest",
    Entrypoint = new[] { "nginx", "-g", "daemon off;" },
    HealthCheck = async (sbx) =>
    {
        var ep = await sbx.GetEndpointAsync(80);
        using var request = new HttpRequestMessage(HttpMethod.Get, await sbx.GetEndpointUrlAsync(80));
        foreach (var header in ep.Headers)
            request.Headers.TryAddWithoutValidation(header.Key, header.Value);
        try
        {
            using var response = await http.SendAsync(request);
            return response.StatusCode == System.Net.HttpStatusCode.OK;
        }
        catch (HttpRequestException) { return false; }
        catch (TaskCanceledException) { return false; }
    },
});
```

### 3. 命令执行与流式输出

实时执行命令并处理输出流。

```csharp
using OpenSandbox.Models;

var handlers = new ExecutionHandlers
{
    OnStdout = msg => { Console.WriteLine($"STDOUT: {msg.Text}"); return Task.CompletedTask; },
    OnStderr = msg => { Console.Error.WriteLine($"STDERR: {msg.Text}"); return Task.CompletedTask; },
    OnExecutionComplete = c => { Console.WriteLine($"Finished in {c.ExecutionTimeMs}ms"); return Task.CompletedTask; },
};

await sandbox.Commands.RunAsync(
    "for i in 1 2 3; do echo \"Count $i\"; sleep 0.2; done",
    handlers: handlers
);
```

要执行不带 shell 解析的原生程序，请传入参数列表。在 Linux 上，此示例打印字面的 `$HOME` 并将 `hello world` 作为一个参数保留：

```csharp
await sandbox.Commands.RunAsync(new[] { "printf", "%s\n", "$HOME", "hello world" });
```

原生 argv 执行需要更新版本的 execd。可执行文件查找与平台行为参见[命令执行模式](/architecture/data-plane/execd#command-execution)。

#### 后台命令

轮询状态和增量日志。命令超时与 sandbox TTL 相互独立。

```csharp
var execution = await sandbox.Commands.RunAsync(
    "for i in 1 2 3; do echo step-$i; sleep 1; done",
    options: new RunCommandOptions
    {
        Background = true,
        TimeoutSeconds = 30,
    });

var commandId = execution.Id ?? throw new InvalidOperationException("No command ID returned");
long cursor = 0;
using var budget = new CancellationTokenSource(TimeSpan.FromSeconds(45));
try
{
    while (true)
    {
        var status = await sandbox.Commands.GetCommandStatusAsync(commandId, budget.Token);
        var logs = await sandbox.Commands.GetBackgroundCommandLogsAsync(commandId, cursor, budget.Token);
        Console.Write(logs.Content);
        cursor = logs.Cursor ?? cursor;
        if (status.Running == false)
        {
            if (status.ExitCode != 0)
                throw new InvalidOperationException($"Command failed: {status.ExitCode}, {status.Error}");
            break;
        }
        await Task.Delay(500, budget.Token);
    }
}
catch (OperationCanceledException) when (budget.IsCancellationRequested)
{
    await sandbox.Commands.InterruptAsync(commandId);
    throw;
}
```

#### 持久 shell 会话

Bash 会话可跨命令保留 shell 变量和工作目录。

```csharp
var sessionId = await sandbox.Commands.CreateSessionAsync(
    new CreateSessionOptions { WorkingDirectory = "/tmp" });
try
{
    await sandbox.Commands.RunInSessionAsync(sessionId, "export DEMO=hello");
    var result = await sandbox.Commands.RunInSessionAsync(sessionId, "echo \"$DEMO\"; pwd");
    foreach (var message in result.Logs.Stdout) Console.Write(message.Text);
}
finally
{
    await sandbox.Commands.DeleteSessionAsync(sessionId);
}
```

#### 持久环境变量

设置由运行时注入到每个后续命令和会话中的环境变量 —— 无需手写 shell 转义去操作 sandbox env 文件。

```csharp
await sandbox.Commands.SetEnvAsync("MY_TOKEN", "it's a safe value");
```

键必须匹配 `[A-Za-z_][A-Za-z0-9_]*`。不含单引号的值按原样存储；含单引号的值使用 env 文件的双引号形式，其中 shell 风格的 `$NAME` 序列可能在运行时加载该文件时被展开。env 文件是追加式的：同一键的最后一次写入生效。如果 sandbox 未能持久化该变量，将抛出 `SandboxException`。

关于 sandbox 内的文件系统/进程隔离，参见[隔离会话](/guides/isolation-sessions)。它们与 Bash 会话是相互独立的概念。

### 4. 文件操作

管理文件和目录，包括读取、写入、列出/搜索和删除。

```csharp
await sandbox.Files.CreateDirectoriesAsync(new[]
{
    new CreateDirectoryEntry { Path = "/tmp/demo", Mode = 755 }
});

await sandbox.Files.WriteFilesAsync(new[]
{
    new WriteEntry { Path = "/tmp/demo/hello.txt", Data = "Hello World", Mode = 644 }
});

var content = await sandbox.Files.ReadFileAsync("/tmp/demo/hello.txt");
Console.WriteLine($"Content: {content}");

var entries = await sandbox.Files.ListDirectoryAsync("/tmp/demo", depth: 1);
foreach (var entry in entries) Console.WriteLine(entry.Path);

var files = await sandbox.Files.SearchAsync(new SearchEntry { Path = "/tmp/demo", Pattern = "*.txt" });
foreach (var file in files)
{
    Console.WriteLine(file.Path);
}

await sandbox.Files.DeleteFilesAsync(new[] { "/tmp/demo/hello.txt" });
await sandbox.Files.DeleteDirectoriesAsync(new[] { "/tmp/demo" });
```

对于二进制数据，请在 `WriteEntry.Data` 中使用 `byte[]` 或 `Stream`，并用 `ReadBytesAsync()` 或 `ReadBytesStreamAsync()` 下载。读取选项支持部分下载。

#### 带显式身份的文件操作

将文件系统客户端绑定到 Linux UID 和 GID：

```csharp
var files = sandbox.FilesWithIdentity(1001, 2000);
```

绑定的客户端暴露常规文件操作，并共享 sandbox 连接、认证和端点路由头。既有文件客户端保持其默认身份。命令执行时请单独选择 UID/GID。

这需要 Linux 以及支持文件系统执行身份的 Execd 版本。不支持的服务器会返回错误；请求绝不会回退到默认文件系统路由。请将 sandbox 凭证和身份选择保留在可信后端。Linux 文件系统权限依然生效；该特性并不创建租户隔离边界。权限与 workspace 供给详情参见 [Execd](/architecture/data-plane/execd)。

### 5. 端点

`GetEndpointAsync()` 返回**不带协议**的端点（例如 `"localhost:44772"`）。如需可直接使用的绝对 URL，请使用 `GetEndpointUrlAsync()`。

```csharp
var endpoint = await sandbox.GetEndpointAsync(44772);
Console.WriteLine(endpoint.EndpointAddress);

var url = await sandbox.GetEndpointUrlAsync(44772);
Console.WriteLine(url); // e.g., "http://localhost:44772"
```

向 sandbox 服务发请求时请转发 `endpoint.Headers`，包括任何 secure-access 凭证。上面的健康检查示例展示了对目标端口上监听的应用发起 HTTP 请求的方式。

### 6. Sandbox 管理（管理面）

使用 `SandboxManager` 执行管理任务和查找既有 sandbox。

```csharp
await using var manager = SandboxManager.Create(new SandboxManagerOptions
{
    ConnectionConfig = config
});

var list = await manager.ListSandboxInfosAsync(new SandboxFilter
{
    States = new[] { SandboxStates.Running },
    Page = 1, // First page only; increase for subsequent pages.
    PageSize = 10
});

foreach (var s in list.Items)
{
    Console.WriteLine(s.Id);
}
```

### 资源指标

使用 `await sandbox.Metrics.GetMetricsAsync()` 读取当前 sandbox 的资源使用。这与 [SDK 创建遥测](/sdks/observability#creation-metrics)相互独立。

## 快照、模板与元数据

| 操作 | 公共 API |
| --- | --- |
| 为 sandbox 创建快照 | `sandbox.CreateSnapshotAsync` 或 `manager.CreateSnapshotAsync` |
| 查看/列出/删除快照 | `manager.GetSnapshotAsync`、`ListSnapshotsAsync`、`DeleteSnapshotAsync` |
| 恢复快照 | `Sandbox.CreateAsync` 携带 `SnapshotId` 且不传 `Image`/`Entrypoint` |
| 管理 Fsb 模板 | `manager.CreateTemplateAsync`、`GetTemplateAsync`、`ListTemplatesAsync`、`DeleteTemplateAsync` |
| 从已发布模板创建 | `Sandbox.CreateFromTemplateAsync` 携带 `SandboxCreateFromTemplateOptions` |
| 修补元数据 | `sandbox.PatchMetadataAsync` 或 `manager.PatchSandboxMetadataAsync` |

恢复之前请轮询快照状态。模板构建是异步的；使用前请等待 `Succeeded`。基于模板的创建需要 `TimeoutSeconds`，并从已发布的模板继承工作负载配置。元数据修补值会新增/替换键；`null` 删除键。后端约束参见[生命周期契约](/api/#1-sandbox-lifecycle-yml)。

快照支持取决于运行时与 server 配置。如果源 sandbox 的剩余 TTL 可能在快照创建期间到期，请先续期。此示例最多等待 15 分钟，并保留快照以便复用：

```csharp
await using var manager = SandboxManager.Create(new SandboxManagerOptions { ConnectionConfig = config });
var snapshot = await sandbox.CreateSnapshotAsync("demo");
Console.WriteLine($"Snapshot: {snapshot.Id}");
using var budget = new CancellationTokenSource(TimeSpan.FromMinutes(15));
while (true)
{
    snapshot = await manager.GetSnapshotAsync(snapshot.Id, budget.Token);
    if (snapshot.Status.State == "Ready") break;
    if (snapshot.Status.State == "Failed")
        throw new InvalidOperationException(snapshot.Status.Message);
    await Task.Delay(2000, budget.Token);
}
await using var restored = await Sandbox.CreateAsync(new SandboxCreateOptions
{
    SnapshotId = snapshot.Id,
    ConnectionConfig = config,
});
try { Console.WriteLine(restored.Id); }
finally { await restored.KillAsync(); }
// When no longer needed: await manager.DeleteSnapshotAsync(snapshot.Id);
```

在模板构建达到 `Succeeded` 后使用既有模板：

```csharp
await using var templated = await Sandbox.CreateFromTemplateAsync(new SandboxCreateFromTemplateOptions
{
    TemplateId = "your-published-template-id",
    TimeoutSeconds = 600,
    ConnectionConfig = config,
});
```

新增/替换一个元数据键并移除另一个：

```csharp
await sandbox.PatchMetadataAsync(new Dictionary<string, string?>
{
    ["project"] = "demo",
    ["obsolete-key"] = null,
});
```

## 配置

### 1. 连接配置

`ConnectionConfig` 类管理 API server 的连接设置。

| 参数 | 说明 | 默认值 | 环境变量 |
| --- | --- | --- | --- |
| `ApiKey` | 用于认证的 API 密钥 | 可选 | `OPEN_SANDBOX_API_KEY` |
| `Domain` | sandbox 服务域（`host[:port]`） | `localhost:8080` | `OPEN_SANDBOX_DOMAIN` |
| `Protocol` | HTTP 协议（`Http`/`Https`） | `Http` | - |
| `RequestTimeoutSeconds` | 应用于 SDK HTTP 调用的请求超时 | `30` | - |
| `UseServerProxy` | 请求 server 代理的 sandbox 端点 URL | `false` | - |
| `Headers` | 应用于每个请求的额外头 | `{}` | - |
| `DisableMetrics` | 禁用 SDK 创建延迟遥测（参见 [SDK 遥测](/sdks/observability#creation-metrics)） | `false` | `OPENSANDBOX_DISABLE_METRICS` |

```csharp
using OpenSandbox.Config;

// 1. Basic configuration
var config = new ConnectionConfig(new ConnectionConfigOptions
{
    Domain = "api.opensandbox.io",
    ApiKey = "your-key",
    RequestTimeoutSeconds = 60,
    // UseServerProxy = true, // Useful when the client cannot access sandbox endpoint directly
});

// 2. Advanced: custom headers
var config2 = new ConnectionConfig(new ConnectionConfigOptions
{
    Domain = "api.opensandbox.io",
    ApiKey = "your-key",
    Headers = new Dictionary<string, string>
    {
        ["X-Custom-Header"] = "value"
    },
});
```

::: tip SDK 遥测
`Sandbox.CreateAsync` 默认将创建延迟上报到 `POST /v1/metrics/events`。设置 `ConnectionConfigOptions.DisableMetrics = true` 或导出 `OPENSANDBOX_DISABLE_METRICS=1` 可选择退出。参见 [SDK 遥测](/sdks/observability#creation-metrics)。
:::

### 2. SDK 日志

SDK 使用 `Microsoft.Extensions.Logging` 抽象。`SdkDiagnosticsOptions` 配置本地日志；它不获取远程 sandbox 的诊断日志或事件。这些请使用 [CLI 或 HTTP API](/api/#diagnostics)。Client Pool 和内置的 pool warmup 追踪目前在 C# 中不可用。
安装 `Microsoft.Extensions.Logging.Console` 以使用下方的 `AddConsole()`。

```csharp
using Microsoft.Extensions.Logging;
using OpenSandbox.Config;

using var loggerFactory = LoggerFactory.Create(builder =>
{
    builder.SetMinimumLevel(LogLevel.Debug);
    builder.AddConsole();
});

var sandbox = await Sandbox.CreateAsync(new SandboxCreateOptions
{
    Image = "python:3.11",
    ConnectionConfig = new ConnectionConfig(),
    Diagnostics = new SdkDiagnosticsOptions
    {
        LoggerFactory = loggerFactory
    }
});
```

### 3. Sandbox 创建配置

`Sandbox.CreateAsync()` 允许配置 sandbox 环境。

| 参数 | 说明 | 默认值 |
| --- | --- | --- |
| `Image` | 要使用的 Docker 镜像 | image 或快照 ID 二选一 |
| `TimeoutSeconds` | 自动终止超时（server 侧 TTL） | 10 分钟 |
| `Entrypoint` | 容器入口命令 | `["tail","-f","/dev/null"]` |
| `Resource` | CPU 与内存限制（字符串映射） | `{"cpu":"1","memory":"2Gi"}` |
| `Env` | 环境变量 | `{}` |
| `Metadata` | 自定义元数据标签 | `{}` |
| `NetworkPolicy` | 可选的出站网络策略（egress） | - |
| `CredentialProxy` | 可选的 Credential Vault 代理启动设置 | - |
| `Volumes` | 可选的存储挂载（`Host` / `PVC` / `OSSFS`，支持 `ReadOnly` 和 `SubPath`） | - |
| `Extensions` | 额外的 server 定义字段 | `{}` |
| `SkipHealthCheck` | 跳过就绪检查（`Running` + 健康检查） | `false` |
| `HealthCheck` | 自定义就绪检查 | - |
| `ReadyTimeoutSeconds` | 等待就绪的最长时间 | 30 秒 |
| `HealthCheckPollingInterval` | 等待期间的轮询间隔（毫秒） | 200 ms |
| `SnapshotId` | 恢复快照而不是传 `Image`；省略 `Entrypoint` | - |
| `ResourceRequests` | Kubernetes 资源请求；不得超过限制 | - |
| `Lifecycle` | 启动前与周期性钩子 | - |
| `Platform` | OS/架构约束 | - |
| `SecureAccess` | 要求端点访问凭证 | `false` |
| `ManualCleanup` | 对镜像/快照创建禁用 TTL 过期 | `false` |

::: warning
`opensandbox.io/` 下的元数据键保留给系统管理的标签，将被 server 拒绝。
:::

```csharp
var sandbox = await Sandbox.CreateAsync(new SandboxCreateOptions
{
    ConnectionConfig = config,
    Image = "python:3.11",
    NetworkPolicy = new NetworkPolicy
    {
        DefaultAction = NetworkRuleAction.Deny,
        Egress = new List<NetworkRule>
        {
            new() { Action = NetworkRuleAction.Allow, Target = "pypi.org" }
        }
    },
    Volumes = new[]
    {
        new Volume
        {
            Name = "workspace",
            Host = new Host { Path = "/tmp/opensandbox-e2e/host-volume-test" },
            MountPath = "/workspace",
            ReadOnly = false
        }
    }
});
```

### 4. 运行时 Egress 策略更新

运行时 egress 策略路由取决于 sandbox 的来源。对基于镜像的 sandbox，SDK 解析端口 `18080` 并调用 sidecar 的 `/policy` API。对基于模板的 sandbox（包括恢复自模板的快照），SDK 检测 `OPEN-SANDBOX-ORIGIN: template` 并通过生命周期 `/sandboxes/{sandboxId}/networkpolicy` API 路由策略操作。

修补使用合并语义：
- 传入规则优先于具有相同 `Target` 的既有规则。
- 其他目标的既有规则保持不变。
- 在单个修补载荷内，某个 `Target` 的第一条规则生效。
- 当前的 `DefaultAction` 被保留。

```csharp
var policy = await sandbox.GetEgressPolicyAsync();

await sandbox.PatchEgressRulesAsync(new[]
{
    new NetworkRule { Action = NetworkRuleAction.Allow, Target = "www.github.com" },
    new NetworkRule { Action = NetworkRuleAction.Deny, Target = "pypi.org" }
});
```

### 5. Credential Vault

Credential Vault 需要 sandbox 侧的 egress 服务，对基于模板的 sandbox 不可用。它从 egress sidecar 注入出站凭证，同时让真实密钥远离 sandbox 的环境变量、命令、文件和日志。以启用 `CredentialProxy` 的方式创建 sandbox，然后通过 `sandbox.CredentialVault` 或 sandbox 辅助方法写入凭证和绑定。

```csharp
var sandbox = await Sandbox.CreateAsync(new SandboxCreateOptions
{
    ConnectionConfig = config,
    Image = "python:3.11",
    NetworkPolicy = new NetworkPolicy
    {
        DefaultAction = NetworkRuleAction.Deny,
        Egress = new List<NetworkRule>
        {
            new() { Action = NetworkRuleAction.Allow, Target = "api.example.com" }
        }
    },
    CredentialProxy = new CredentialProxyConfig { Enabled = true }
});

await sandbox.CreateCredentialVaultAsync(
    new[]
    {
        new Credential
        {
            Name = "api-token",
            Source = new InlineCredentialSource { Value = "<token>" }
        }
    },
    new[]
    {
        new CredentialBinding
        {
            Name = "api-token",
            Match = new CredentialMatch
            {
                Schemes = new[] { "https" },
                Hosts = new[] { "api.example.com" },
                Paths = new[] { "/v1/*" }
            },
            Auth = new CredentialAuth
            {
                Type = "apiKey",
                Name = "x-api-key",
                Credential = "api-token"
            }
        }
    });
```

认证类型、绑定指南以及 Git/curl 示例参见 [Credential Vault](/guides/credential-vault)。

### 6. 超时与重试行为

- `ConnectionConfig.RequestTimeoutSeconds` 控制 SDK HTTP 调用的超时。
- `RunCommandOptions.TimeoutSeconds` 控制命令运行时的执行超时。
- `RunInSessionOptions.TimeoutSeconds` 控制会话运行时的命令执行超时。
- `SandboxCreateOptions.TimeoutSeconds` 控制 sandbox 的 server 侧 TTL。
- `ReadyTimeoutSeconds` 控制就绪等待。对 `ConnectAsync` / `ResumeAsync`，端点发现和健康检查共享该超时。阻塞的自定义代码可能延迟超时的上报。
- SDK 不会自动重试失败的 API 请求；请在调用方代码中视情况实现重试。

### 7. 资源清理

`Sandbox` 和 `SandboxManager` 都实现了 `IAsyncDisposable`。使用完毕后请用 `await using` 或调用 `DisposeAsync()`。

```csharp
await using var sandbox = await Sandbox.CreateAsync(options);
// ... use sandbox ...
// Automatically disposed when leaving scope
```

## 错误处理

当操作失败时，SDK 会抛出 `SandboxException`（以及派生异常，如 `SandboxApiException`、`SandboxReadyTimeoutException` 和 `InvalidArgumentException`）。

```csharp
try
{
    var execution = await sandbox.Commands.RunAsync("echo 'Hello Sandbox!'");
    Console.WriteLine(execution.Logs.Stdout.FirstOrDefault()?.Text);
}
catch (SandboxReadyTimeoutException)
{
    Console.Error.WriteLine("Sandbox did not become ready before the configured timeout.");
}
catch (SandboxApiException ex)
{
    Console.Error.WriteLine($"API Error: status={ex.StatusCode}, requestId={ex.RequestId}, message={ex.Message}");
}
catch (SandboxException ex)
{
    Console.Error.WriteLine($"Sandbox Error: [{ex.Error.Code}] {ex.Error.Message}");
}
```

## 支持的框架

- .NET Standard 2.0（与 .NET Framework 4.6.1+、.NET Core 2.0+、Mono、Xamarin 等实现最大兼容）
- .NET Standard 2.1
- .NET 6.0
- .NET 7.0
- .NET 8.0
- .NET 9.0
- .NET 10.0

## 许可证

Apache License 2.0
