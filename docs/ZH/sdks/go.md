---
title: Go SDK
description: 覆盖生命周期、execd 和 egress 操作的 OpenSandbox API Go 客户端库。
---

# OpenSandbox Go SDK

[OpenSandbox](https://github.com/opensandbox-group/OpenSandbox/) API 的 Go 客户端库。

为以下 API 领域提供高层 sandbox 辅助方法和低层客户端：
- **Lifecycle** —— 创建、管理和销毁 sandbox 实例
- **Execd** —— 在 sandbox 内执行命令、管理文件、监控指标
- **Egress** —— 在运行时检查和修改 sandbox 网络策略

## 安装

```bash
# go 1.20+
go get github.com/alibaba/OpenSandbox/sdks/sandbox/go
```

## 快速开始

### 创建并管理 sandbox

使用高层辅助方法创建 sandbox、解析其端点，并在运行命令之前等待就绪：

```go
package main

import (
    "context"
    "fmt"
    "log"

    opensandbox "github.com/alibaba/OpenSandbox/sdks/sandbox/go"
)

func run() error {
    ctx := context.Background()
    ttl := 600
    config := opensandbox.ConnectionConfig{
        Domain: "localhost:8080",
        Protocol: "http",
        UseServerProxy: true,
    }
    sandbox, err := opensandbox.CreateSandbox(ctx, config, opensandbox.SandboxCreateOptions{
        Image: "python:3.12",
        TimeoutSeconds: &ttl,
    })
    if err != nil {
        return err
    }
    defer sandbox.Close()
    defer sandbox.Kill(context.Background())

    result, err := sandbox.RunCommand(ctx, "echo sandbox-ready", nil)
    if err != nil {
        return err
    }
    fmt.Println(result.Text())
    return nil
}

func main() {
    if err := run(); err != nil {
        log.Fatal(err)
    }
}
```

`ConnectSandbox` 和 `ResumeSandbox` 仅在你传入可选的 `ReadyOptions` 参数时才检查健康（空的 `ReadyOptions{}` 使用默认检查）。省略它则只解析端点而不做健康检查。暂停是异步的：恢复之前请等待 sandbox info 报告 `Paused`。

以下代码片段应放在一个返回 `error` 的函数内部，并使用上面的存活 `sandbox`、`config` 和 `ctx`。请为每个片段添加其用到的标准库导入。各创建示例互为替代；用完的 sandbox 以 `Kill` 终止。`Close` 目前不会终止 sandbox。

### 以流式输出运行命令

对常规命令输出使用结构化回调：

```go
result, err := sandbox.RunCommand(ctx, "echo 'Hello from sandbox!'", &opensandbox.ExecutionHandlers{
    OnStdout: func(message opensandbox.OutputMessage) error {
        fmt.Println(message.Text)
        return nil
    },
})
if err != nil {
    return err
}
if result.ExitCode == nil || *result.ExitCode != 0 {
    return fmt.Errorf("command failed: %v", result.Error)
}
```

当你需要原始 SSE 事件或命令状态/日志 API 时，使用低层客户端。`event.Data` 包含 JSON；本例需要导入 `encoding/json` 和 `os`：

```go
endpoint, err := sandbox.GetEndpoint(ctx, 44772)
if err != nil {
    return err
}
exec := opensandbox.NewExecdClient(
    config.Protocol + "://" + endpoint.Endpoint, "",
    opensandbox.WithHeaders(endpoint.Headers),
)

err = exec.RunCommand(ctx, opensandbox.RunCommandRequest{
    Command: "echo 'Hello from sandbox!'",
    Timeout: 30000,
}, func(event opensandbox.StreamEvent) error {
    var message struct{ Type, Text string }
    if err := json.Unmarshal([]byte(event.Data), &message); err != nil {
        return err
    }
    switch message.Type {
    case "stdout":
        fmt.Println(message.Text)
    case "stderr":
        fmt.Fprintln(os.Stderr, message.Text)
    case "execution_complete":
        fmt.Println("[done]")
    }
    return nil
})
if err != nil {
    return err
}
```

`RunCommandRequest.Timeout` 的单位是**毫秒**。`InterruptCommand`、`GetCommandStatus` 和 `GetCommandLogs` 未作为高层 `Sandbox` 方法暴露，请使用这个低层客户端。

要执行原生程序，请将上面的请求替换为以下内容。在 Linux 上，它打印字面的 `$HOME` 并将 `hello world` 作为一个参数保留：

```go
_, err = sandbox.RunCommandWithOpts(ctx, opensandbox.RunCommandRequest{
    Argv:    []string{"printf", "%s\n", "$HOME", "hello world"},
    Timeout: 30000,
}, nil)
if err != nil {
    return err
}
```

原生 argv 执行需要更新版本的 execd。可执行文件查找与平台行为参见[命令执行模式](/architecture/data-plane/execd#command-execution)。

### 后台命令

复用流式示例中的 `exec` 客户端来轮询状态和增量日志。命令超时单位为毫秒，与 sandbox TTL 相互独立。

```go
execution, err := sandbox.RunCommandWithOpts(ctx, opensandbox.RunCommandRequest{
    Command: "for i in 1 2 3; do echo step-$i; sleep 1; done",
    Background: true,
    Timeout: 30_000,
}, nil)
if err != nil {
    return err
}
if execution.ID == "" {
    return fmt.Errorf("no command ID returned")
}
cursor := int64(0)
deadline := time.Now().Add(45 * time.Second)
for {
    if time.Now().After(deadline) {
        if err := exec.InterruptCommand(ctx, execution.ID); err != nil {
            return err
        }
        return fmt.Errorf("command did not finish")
    }
    status, err := exec.GetCommandStatus(ctx, execution.ID)
    if err != nil {
        return err
    }
    logs, err := exec.GetCommandLogs(ctx, execution.ID, &cursor)
    if err != nil {
        return err
    }
    fmt.Print(logs.Output)
    cursor = logs.Cursor
    if !status.Running {
        if status.ExitCode == nil || *status.ExitCode != 0 {
            return fmt.Errorf("command failed: %s", status.Error)
        }
        break
    }
    select {
    case <-ctx.Done():
        return ctx.Err()
    case <-time.After(500 * time.Millisecond):
    }
}
```

### 持久 shell 会话

Bash 会话可跨命令保留 shell 变量和工作目录。

```go
session, err := sandbox.CreateSession(ctx)
if err != nil {
    return err
}
defer sandbox.DeleteSession(context.Background(), session.ID)
_, err = sandbox.RunInSession(ctx, session.ID, opensandbox.RunInSessionRequest{
    Command: "cd /tmp; export DEMO=hello",
    Timeout: 30_000,
}, nil)
if err != nil {
    return err
}
result, err := sandbox.RunInSession(ctx, session.ID, opensandbox.RunInSessionRequest{
    Command: "echo \"$DEMO\"; pwd",
    Timeout: 30_000,
}, nil)
if err != nil {
    return err
}
fmt.Println(result.Text())
```

### 持久环境变量

设置由运行时注入到每个后续命令和会话中的环境变量 —— 无需手写 shell 转义去操作 sandbox env 文件。

```go
err := sandbox.SetEnv(ctx, "MY_TOKEN", "it's a safe value")
```

键必须匹配 `[A-Za-z_][A-Za-z0-9_]*`。不含单引号的值按原样存储；含单引号的值使用 env 文件的双引号形式，其中 shell 风格的 `$NAME` 序列可能在运行时加载该文件时被展开。env 文件是追加式的：同一键的最后一次写入生效。如果 sandbox 未能持久化该变量，将返回错误。

关于 sandbox 内的文件系统/进程隔离，参见[隔离会话](/guides/isolation-sessions)。它们与 Bash 会话是相互独立的概念。

### 文件操作

从 `io.Reader` 上传，并在使用后关闭下载流。此示例使用标准库的 `strings`、`io` 和 `os`：

```go
err := sandbox.UploadFile(ctx, strings.NewReader("Hello Sandbox!"), opensandbox.UploadFileOptions{
    Metadata: opensandbox.FileMetadata{Path: "/tmp/demo.txt", Mode: 644},
})
if err != nil {
    return err
}
body, err := sandbox.DownloadFile(ctx, "/tmp/demo.txt", "")
if err != nil {
    return err
}
_, copyErr := io.Copy(os.Stdout, body)
closeErr := body.Close()
if copyErr != nil {
    return copyErr
}
if closeErr != nil {
    return closeErr
}
entries, err := sandbox.ListDirectory(ctx, "/tmp")
if err != nil {
    return err
}
for _, entry := range entries {
    fmt.Println(entry.Path)
}
if err := sandbox.DeleteFiles(ctx, []string{"/tmp/demo.txt"}); err != nil {
    return err
}
```

相同的上传/下载方法也支持二进制文件。向 `DownloadFile` 传入诸如 `"bytes=0-1023"` 的 Range 头可实现部分下载。

#### 带显式身份的文件操作

将文件系统客户端绑定到 Linux UID 和 GID：

```go
files, err := sandbox.FilesWithIdentity(1001, 2000)
if err != nil {
    return err
}
```

绑定的客户端暴露常规文件操作，并共享 sandbox 连接、认证和端点路由头。既有文件客户端保持其默认身份。命令执行时请单独选择 UID/GID。

这需要 Linux 以及支持文件系统执行身份的 Execd 版本。不支持的服务器会返回错误；请求绝不会回退到默认文件系统路由。请将 sandbox 凭证和身份选择保留在可信后端。Linux 文件系统权限依然生效；该特性并不创建租户隔离边界。权限与 workspace 供给详情参见 [Execd](/architecture/data-plane/execd)。

### 暂停与重连

暂停是异步的且取决于运行时。恢复之前请使用截止时间：

```go
pauseCtx, cancel := context.WithTimeout(ctx, 2*time.Minute)
defer cancel()
if err := sandbox.Pause(pauseCtx); err != nil {
    return err
}
for {
    info, err := sandbox.GetInfo(pauseCtx)
    if err != nil {
        return err
    }
    if info.Status.State == "Paused" {
        break
    }
    if info.Status.State == "Failed" {
        return fmt.Errorf("pause failed: %s", info.Status.Message)
    }
    select {
    case <-pauseCtx.Done():
        return pauseCtx.Err()
    case <-time.After(time.Second):
    }
}
resumed, err := opensandbox.ResumeSandbox(ctx, config, sandbox.ID(), opensandbox.ReadyOptions{})
if err != nil {
    return err
}
defer resumed.Close()
if _, err := resumed.Renew(ctx, 30*time.Minute); err != nil {
    return err
}
```

使用 `ConnectSandbox(ctx, config, id, opensandbox.ReadyOptions{})` 可以带就绪检查地附加到已在运行的 sandbox。参见[暂停与恢复](/guides/pause-resume)。

### 资源指标

使用 `metrics, err := sandbox.GetMetrics(ctx)` 读取当前 sandbox 的资源使用。这与 [SDK 创建遥测](/sdks/observability#creation-metrics)相互独立。

### 检查 egress 策略

运行时的 egress 读取和修补直接发送到 sandbox 的 egress sidecar。SDK 首先解析 sandbox 端口 `18080` 上的端点，然后调用 sidecar 的 `/policy` API。

```go
policy, err := sandbox.GetEgressPolicy(ctx)
if err != nil {
    return err
}
fmt.Printf("Mode: %s, Default: %s\n", policy.Mode, policy.Policy.DefaultAction)

_, err = sandbox.PatchEgressRules(ctx, []opensandbox.NetworkRule{
    {Action: "allow", Target: "api.example.com"},
})
if err != nil {
    return err
}
```

基于模板的 sandbox 没有 sandbox 侧的 egress sidecar：SDK 通过 server 的 `OPEN-SANDBOX-ORIGIN` 响应头检测它们（参见 [Fsb 模板管理](#fsb-template-management)），并改为通过生命周期控制面（`/sandboxes/{sandboxId}/networkpolicy`）路由相同的 `GetEgressPolicy` / `PatchEgressRules` / `DeleteEgressRules` 调用。

修补使用合并语义：
- 传入规则优先于具有相同 `target` 的既有规则。
- 其他目标的既有规则保持不变。

### 使用 Credential Vault

Credential Vault 从 egress sidecar 注入出站凭证，同时让真实密钥远离 sandbox 的环境变量、命令、文件和日志。以启用 `CredentialProxy` 的方式创建 sandbox，然后通过 sandbox 辅助方法或 `EgressClient` 写入凭证和绑定。

```go
config := opensandbox.ConnectionConfig{
    Domain:   "localhost:8080",
    Protocol: "http",
    APIKey:   "your-api-key",
}

sandbox, err := opensandbox.CreateSandbox(ctx, config, opensandbox.SandboxCreateOptions{
    Image: "python:3.11",
    NetworkPolicy: &opensandbox.NetworkPolicy{
        DefaultAction: "deny",
        Egress: []opensandbox.NetworkRule{
            {Action: "allow", Target: "api.example.com"},
        },
    },
    CredentialProxy: &opensandbox.CredentialProxyConfig{Enabled: true},
})
if err != nil {
    return err
}

_, err = sandbox.CreateCredentialVault(ctx, opensandbox.CredentialVaultCreateRequest{
    Credentials: []opensandbox.Credential{
        {
            Name: "api-token",
            Source: opensandbox.InlineCredentialSource{
                Type:  opensandbox.CredentialSourceInline,
                Value: "<token>",
            },
        },
    },
    Bindings: []opensandbox.CredentialBinding{
        {
            Name: "api-token",
            Match: opensandbox.CredentialMatch{
                Schemes: []opensandbox.CredentialScheme{opensandbox.CredentialSchemeHTTPS},
                Ports:   []int{443},
                Hosts:   []string{"api.example.com"},
                Paths:   []string{"/v1/*"},
            },
            Auth: opensandbox.CredentialAuth{
                Type:       opensandbox.CredentialAuthAPIKey,
                Name:       "x-api-key",
                Credential: "api-token",
            },
        },
    },
})
```

认证类型、绑定指南以及 Git/curl 示例参见 [Credential Vault](/guides/credential-vault)。

::: warning
Credential Vault 对基于模板的 sandbox 不可用：它们没有 sandbox 侧的 egress sidecar。对它们调用 `sandbox.CredentialVault(ctx)` 会返回错误。
:::

### Client Pool

将 `NewSandboxPoolBuilder` 与内存存储或 `github.com/alibaba/OpenSandbox/sdks/sandbox/go/poolredis` 中的 Redis 适配器搭配使用。Go 支持全部四种获取策略、命名空间退役，以及按获取的 TTL/健康检查覆盖。

Go 使用 `ReconcileInterval` 和 `WarmupConcurrency`；它不暴露 Python/JVM/JavaScript 的 create-QPS、初始延迟或 post-prepare-check 设置。默认值、示例和清理参见 [Client Pool](/guides/client-pool)。
Go 目前不输出内置的 pool warmup 追踪，也不暴露稳定的远程诊断；请使用 [CLI 或 HTTP 诊断](/api/#diagnostics)。

## 生命周期钩子

在 `SandboxCreateOptions` 中设置 `Lifecycle`。`PreStart` 在入口点启动之前完成，而 `Periodic` 钩子在启动后按各自的计划运行。

```go
hookTimeout := 120
sandbox, err := opensandbox.CreateSandbox(ctx, config, opensandbox.SandboxCreateOptions{
    Image: "ubuntu:24.04",
    Lifecycle: &opensandbox.SandboxLifecycle{
        PreStart: &opensandbox.LifecycleHook{
            Command:        []string{"sh", "-c", "echo ready > /tmp/prestart.done"},
            TimeoutSeconds: &hookTimeout,
        },
        Periodic: []opensandbox.PeriodicLifecycleHook{
            {
                Name:           "checkpoint",
                Schedule:       "@every 5m",
                Command:        []string{"sh", "-c", "date -u >> /tmp/checkpoints.log"},
                TimeoutSeconds: &hookTimeout,
            },
        },
    },
})
if err != nil {
    return err
}
defer sandbox.Close()
defer sandbox.Kill(context.Background())
fmt.Println(sandbox.ID())
```

Server 会校验 `TimeoutSeconds`；`PreStart` 接受 1–10800 秒，而 `Periodic` 接受 1–300 秒。两者在省略时均默认为 60 秒。计时、失败行为和 provider 限制参见[生命周期钩子](/guides/lifecycle-hooks)。

### 空闲策略（自动暂停与访问唤醒）

`SandboxLifecycle.IdlePolicy`（OSEP-0024，仅 fast-sandbox）告诉控制面在 ingress 观测到连续空闲流量达到 `IdleTimeoutSeconds`（30–86400）后自动暂停 sandbox，并在下一个请求到来时唤醒它。它属于控制面状态：永远不会到达 guest，在模板模式（`SandboxFromTemplateOptions.IdlePolicy`）下被接受，并被非 fast-sandbox 后端拒绝。省略它则完全保持非自动化行为。

```go
idleTimeout := 300
wakeOnAccess := true
sandbox, err := opensandbox.CreateSandbox(ctx, config, opensandbox.SandboxCreateOptions{
    Image: "ubuntu:24.04",
    Lifecycle: &opensandbox.SandboxLifecycle{
        IdlePolicy: &opensandbox.IdlePolicy{
            IdleTimeoutSeconds: &idleTimeout,
            OnIdle:             opensandbox.OnIdlePause,
            WakeOnAccess:       &wakeOnAccess,
        },
    },
})
```

`ActiveProbe` 让应用在真实工作运行但不产生 ingress 流量时否决暂停：声明一个以 `{"active": bool, "message"?: string}` 应答的端点，当 sandbox 除此之外处于空闲状态时 server 会轮询它；忙碌的应答会重置空闲时钟。无效策略会通过 `IdlePolicy.Validate()` 在客户端快速失败。

### 唤醒重试

对已暂停 sandbox 的请求可能在其恢复仍在进行时到达。网关以 `OpenSandbox-Wake: expired` 标记应答 `503`；客户端识别该标记并在 `Retry-After` 延迟之后重发原始请求（对任何方法都安全 —— 该标记证明请求从未被转发）。唤醒预算默认在所有唤醒周期内为 5 次尝试 / 60 秒；预算耗尽时客户端返回 `*SandboxWakeTimeoutError`（携带 `Attempts` 和 `Waited`），而 server 侧的恢复在后台继续。裸 `503` 响应遵循既有重试规则、不受影响。只要设置了 `WithRetry` 配置，唤醒重试即处于激活状态（即使对手工构造的配置，`WakeRetryEnabled` 也默认启用；传入 `WakeRetryDisabled()` 可退出）；流式连接绕过唤醒重试 —— 处于唤醒中的 sandbox 会以带标记的 503 快速失败而不是挂起连接 —— 与其他官方 SDK 保持一致，且已建立的流绝不会被重新执行。

## Fsb 模板管理

fsb（fast-sandbox microVM）黄金镜像模板通过 `SandboxManager` 管理。模板构建是异步的：`CreateTemplate` 返回时 `Status.Phase` 为 `Pending`；轮询 `GetTemplate` 直到阶段达到 `Succeeded` 或 `Failed`。只有 `Succeeded` 的模板才能创建 sandbox。模板管理需要在 Kubernetes 上配置 Fsb provider。请将发布 URI 替换为你的 server 所配置的位置。

```go
manager := opensandbox.NewSandboxManager(config)

buildCtx, cancel := context.WithTimeout(ctx, 15*time.Minute)
defer cancel()

// Start the async build (starts at TemplatePhasePending)
template, err := manager.CreateTemplate(buildCtx, opensandbox.CreateTemplateRequest{
    Image:          "alpine:3.19",
    Publish:        "s3://bucket/publish",
    ResourceLimits: opensandbox.ResourceLimits{"cpu": "1", "memory": "512Mi", "disk": "2Gi"},
    Readiness:      &opensandbox.TemplateReadiness{Probe: "tcp://127.0.0.1:44772"},
    Metadata:       map[string]string{"team": "backend"},
})
if err != nil {
    return err
}

// Poll until the build finishes
for template.Status.Phase != opensandbox.TemplatePhaseSucceeded &&
    template.Status.Phase != opensandbox.TemplatePhaseFailed {
    select {
    case <-buildCtx.Done():
        return buildCtx.Err()
    case <-time.After(2 * time.Second):
    }
    template, err = manager.GetTemplate(buildCtx, template.TemplateID)
    if err != nil {
        return err
    }
}
if template.Status.Phase == opensandbox.TemplatePhaseFailed {
    return fmt.Errorf("template build failed: %s", template.TemplateID)
}

// List with metadata filters (1-indexed paging)
listed, err := manager.ListTemplates(ctx, opensandbox.ListTemplatesOptions{
    Metadata: map[string]string{"team": "backend"},
    Page:     1,
    PageSize: 20,
})
if err != nil {
    return err
}
fmt.Println(listed)

// When no longer needed: manager.DeleteTemplate(ctx, template.TemplateID)
```

### 从模板创建 Sandbox

使用 `CreateSandboxFromTemplate` 从 `Succeeded` 的模板创建 sandbox。模板模式在 server 侧固定了工作负载形态：模板 ID 之外只允许 `Metadata`、`NetworkPolicy` 和 `Extensions`，且 `TimeoutSeconds` 为必填。

```go
sandbox, err := opensandbox.CreateSandboxFromTemplate(ctx, config, "tpl-abc",
    opensandbox.SandboxFromTemplateOptions{
        TimeoutSeconds: 600,
        NetworkPolicy: &opensandbox.NetworkPolicy{
            DefaultAction: "deny",
            Egress: []opensandbox.NetworkRule{
                {Action: "allow", Target: "api.example.com"},
            },
        },
    })
if err != nil {
    return err
}

fmt.Println(sandbox.Origin()) // "template"
```

## 快照与元数据

`Sandbox.CreateSnapshot` 和 `SandboxManager.CreateSnapshot` 启动快照创建。使用 manager 的 `GetSnapshot`、`ListSnapshots` 和 `DeleteSnapshot` 管理快照，然后向 `SandboxCreateOptions` 传入 `SnapshotID` 进行恢复。恢复之前请轮询快照状态。使用 `Sandbox.PatchMetadata` 或 `SandboxManager.PatchSandboxMetadata` 新增/替换键；nil 值移除键。

快照支持取决于运行时与 server 配置。如果源 sandbox 的剩余 TTL 可能在快照创建期间到期，请先续期。恢复之前请等待 `Ready`。此示例保留快照以便复用：

```go
manager := opensandbox.NewSandboxManager(config)
snapshot, err := sandbox.CreateSnapshot(ctx, opensandbox.CreateSnapshotRequest{Name: "demo"})
if err != nil {
    return err
}
fmt.Println("Snapshot:", snapshot.ID)
snapshotCtx, cancel := context.WithTimeout(ctx, 15*time.Minute)
defer cancel()
for {
    snapshot, err = manager.GetSnapshot(snapshotCtx, snapshot.ID)
    if err != nil {
        return err
    }
    if snapshot.Status.State == opensandbox.SnapshotStateReady {
        break
    }
    if snapshot.Status.State == opensandbox.SnapshotStateFailed {
        return fmt.Errorf("snapshot failed: %s", snapshot.Status.Message)
    }
    select {
    case <-snapshotCtx.Done():
        return snapshotCtx.Err()
    case <-time.After(2 * time.Second):
    }
}
restored, err := opensandbox.CreateSandbox(ctx, config, opensandbox.SandboxCreateOptions{
    SnapshotID: snapshot.ID,
})
if err != nil {
    return err
}
defer restored.Close()
defer restored.Kill(context.Background())
fmt.Println(restored.ID())
// When no longer needed: manager.DeleteSnapshot(ctx, snapshot.ID)
```

新增/替换一个元数据键并移除另一个：

```go
project := "demo"
if _, err := sandbox.PatchMetadata(ctx, opensandbox.MetadataPatch{
    "project": &project, "obsolete-key": nil,
}); err != nil {
    return err
}
```

## API 参考

### LifecycleClient

通过 `NewLifecycleClient(baseURL, apiKey string, opts ...Option)` 创建。

| 方法 | 说明 |
|--------|-------------|
| `CreateSandbox(ctx, req)` | 从容器镜像创建新 sandbox |
| `GetSandbox(ctx, id)` | 按 ID 获取 sandbox 详情 |
| `ListSandboxes(ctx, opts)` | 以过滤和分页列出 sandbox |
| `DeleteSandbox(ctx, id)` | 删除 sandbox |
| `PauseSandbox(ctx, id)` | 暂停运行中的 sandbox |
| `ResumeSandbox(ctx, id)` | 恢复已暂停的 sandbox |
| `RenewExpiration(ctx, id, expiresAt)` | 将 sandbox 过期时间设为未来的时间戳（可缩短或延长） |
| `GetEndpoint(ctx, sandboxID, port, useServerProxy)` | 获取 sandbox 端口的公共端点 |
| `GetSignedEndpoint(ctx, sandboxID, port, expires)` | 获取带签名路由令牌的签名端点 URL |
| `CreateTemplate(ctx, req)` | 声明 fsb 模板（异步黄金镜像构建） |
| `GetTemplate(ctx, templateID)` | 获取模板及其最新构建状态 |
| `ListTemplates(ctx, opts)` | 以元数据过滤和分页列出模板 |
| `DeleteTemplate(ctx, templateID)` | 删除模板 |
| `GetNetworkPolicy(ctx, sandboxID)` | 从控制面获取 sandbox 的 egress 策略 |
| `PatchNetworkPolicy(ctx, sandboxID, rules)` | 将 egress 规则合并到 sandbox 的策略中 |
| `DeleteNetworkPolicyRules(ctx, sandboxID, targets)` | 按 target 移除 sandbox 的 egress 规则 |

### ExecdClient

通过 `NewExecdClient(baseURL, accessToken string, opts ...Option)` 创建。

**健康：**

| 方法 | 说明 |
|--------|-------------|
| `Ping(ctx)` | 检查服务器健康 |

**代码执行：**

| 方法 | 说明 |
|--------|-------------|
| `ListContexts(ctx, language)` | 列出活跃的代码执行上下文 |
| `CreateContext(ctx, req)` | 创建代码执行上下文 |
| `GetContext(ctx, contextID)` | 获取上下文详情 |
| `DeleteContext(ctx, contextID)` | 删除上下文 |
| `DeleteContextsByLanguage(ctx, language)` | 删除某语言的全部上下文 |
| `ExecuteCode(ctx, req, handler)` | 以 SSE 流执行代码 |
| `InterruptCode(ctx, sessionID)` | 中断运行中的代码 |

**命令执行：**

| 方法 | 说明 |
|--------|-------------|
| `CreateSession(ctx)` | 创建 bash 会话 |
| `RunInSession(ctx, sessionID, req, handler)` | 在会话中以 SSE 运行命令 |
| `DeleteSession(ctx, sessionID)` | 删除 bash 会话 |
| `RunCommand(ctx, req, handler)` | 以 SSE 流运行命令 |
| `InterruptCommand(ctx, sessionID)` | 中断运行中的命令 |
| `GetCommandStatus(ctx, commandID)` | 获取命令执行状态 |
| `GetCommandLogs(ctx, commandID, cursor)` | 获取命令 stdout/stderr |

**文件操作：**

| 方法 | 说明 |
|--------|-------------|
| `GetFileInfo(ctx, path)` | 获取文件元数据 |
| `DeleteFiles(ctx, paths)` | 删除文件 |
| `SetPermissions(ctx, req)` | 修改文件权限 |
| `MoveFiles(ctx, req)` | 移动/重命名文件 |
| `SearchFiles(ctx, dir, pattern)` | 按 glob 模式搜索文件 |
| `ListDirectory(ctx, path)` | 列出直接目录内容（server 侧默认深度） |
| `ListDirectoryWithDepth(ctx, path, depth)` | 列出目录内容至指定深度（`0` 返回空） |
| `ReplaceInFiles(ctx, req)` | 文件内的文本替换 |
| `UploadFile(ctx, file, opts)` | 向 sandbox 上传文件 |
| `UploadFiles(ctx, entries)` | 向 sandbox 上传多个文件 |
| `DownloadFile(ctx, remotePath, rangeHeader)` | 从 sandbox 下载文件 |

**目录操作：**

| 方法 | 说明 |
|--------|-------------|
| `CreateDirectory(ctx, path, mode)` | 创建目录（mkdir -p） |
| `DeleteDirectory(ctx, path)` | 递归删除目录 |

**指标：**

| 方法 | 说明 |
|--------|-------------|
| `GetMetrics(ctx)` | 获取系统资源指标 |
| `WatchMetrics(ctx, handler)` | 通过 SSE 流式获取指标 |

### EgressClient

通过 `NewEgressClient(baseURL, authToken string, opts ...Option)` 创建。

| 方法 | 说明 |
|--------|-------------|
| `GetPolicy(ctx)` | 获取当前 egress 策略 |
| `PatchPolicy(ctx, rules)` | 将规则合并到当前策略 |
| `CreateCredentialVault(ctx, req)` | 创建 sandbox 本地的 Credential Vault 状态 |
| `GetCredentialVault(ctx)` | 获取脱敏后的 Credential Vault 状态 |
| `PatchCredentialVault(ctx, req)` | 原子化地修改凭证和绑定 |
| `DeleteCredentialVault(ctx)` | 删除 sandbox 本地的 Credential Vault 状态 |
| `ListCredentialVaultCredentials(ctx)` | 列出脱敏后的凭证元数据 |
| `GetCredentialVaultCredential(ctx, name)` | 获取单个凭证的脱敏元数据 |
| `ListCredentialVaultBindings(ctx)` | 列出脱敏后的绑定元数据 |
| `GetCredentialVaultBinding(ctx, name)` | 获取单个绑定的脱敏元数据 |

## SSE 流

流式输出的方法（`RunCommand`、`ExecuteCode`、`RunInSession`、`WatchMetrics`）接受 `EventHandler` 回调：

```go
type EventHandler func(event opensandbox.StreamEvent) error
```

每个 `StreamEvent` 包含：
- `Event` —— 事件类型（例如 `"stdout"`、`"stderr"`、`"result"`、`"execution_complete"`）。对 NDJSON 流，会自动从 JSON 的 `type` 字段提取。
- `Data` —— 原始事件载荷（NDJSON 流为 JSON 字符串）。
- `ID` —— 可选的事件标识符

从处理器返回非 nil 的错误可提前停止流处理。

## 客户端选项

所有客户端构造函数都接受可选的 `Option` 函数。自定义 HTTP 客户端和健康检查必须遵循 context 取消，超时才能生效：

```go
// Reuse the resolved endpoint from the streaming example.
execClient := opensandbox.NewExecdClient(
    config.Protocol + "://" + endpoint.Endpoint, "",
    opensandbox.WithHeaders(endpoint.Headers),
    opensandbox.WithHTTPClient(&http.Client{Timeout: 60 * time.Second}),
)
if err := execClient.Ping(ctx); err != nil {
    return err
}
```

::: info TLS 证书强度
SDK 创建的 HTTP 客户端默认强制 NIST 2030 最小 TLS 证书强度（RSA >= 2048、EC >= 224、DSA P >= 2048/Q >= 224、哈希 >= 224）。如果必须与遗留端点互通，请在 `TransportConfig` 中设置 `AllowWeakServerCertKeyLengths: true`。
:::

::: tip SDK 遥测
`CreateSandbox` 默认将创建延迟上报到 `POST /v1/metrics/events`。设置 `ConnectionConfig.DisableMetrics` 或 `OPENSANDBOX_DISABLE_METRICS=1` 可选择退出。参见 [SDK 遥测](/sdks/observability#creation-metrics)。
:::

### 跟随重定向

Go SDK 保留 Go 的标准重定向行为，包括文件上传和下载。在重定向到不同源（scheme、主机名或有效端口）时，它会移除客户端的默认及已配置的认证头（包括 `X-EXECD-ACCESS-TOKEN`、`X-API-Key` 或自定义的 `WithAuthHeader` 头名），以及所有名称以 `OPEN-SANDBOX-` 或 `OPENSANDBOX-` 开头（不区分大小写）的头。每个重定向目标都会与初始请求的源比较，因此后续跳转到其他源同样受保护。同源重定向保留这些头。无关的自定义头继续遵循 Go 的标准转发规则。

此保护适用于生命周期（包括创建指标）、execd、egress、文件传输和 SSE 请求。通过 `WithHTTPClient` 或 `ConnectionConfig.HTTPClient` 提供的自定义 `http.Client.CheckRedirect` 仍然控制是否跟随重定向；SDK 在该回调运行之后移除受保护的头，即使回调又把它们加了回来。SDK 会复制所提供的 HTTP 客户端而不是修改它。HTTP 与 HTTPS 的默认端口分别被视为与显式端口 80 和 443 等价。上传方法的转换与请求体重放遵循 Go 的既有规则；此保护不会启用流式 multipart 请求体的重放。

## 错误处理

非 2xx 响应以 `*opensandbox.APIError` 返回：

```go
_, err := sandbox.GetInfo(ctx)
if err != nil {
    var apiErr *opensandbox.APIError
    if errors.As(err, &apiErr) {
        fmt.Printf("HTTP %d: %s — %s\n", apiErr.StatusCode, apiErr.Response.Code, apiErr.Response.Message)
    }
    return err
}
```

## 许可证

Apache 2.0
