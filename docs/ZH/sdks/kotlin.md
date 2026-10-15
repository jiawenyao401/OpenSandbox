---
title: Kotlin/Java SDK
description: 用于创建、管理安全 OpenSandbox 环境并与之交互的 Kotlin SDK。
---

# OpenSandbox Kotlin/Java SDK

使用 Java 或 Kotlin 创建 sandbox、运行命令和管理文件。两种语言使用同一个 JVM SDK 和同步 API。

## 安装

### Gradle（Kotlin DSL）

```kotlin
dependencies {
    implementation("com.alibaba.opensandbox:sandbox:{latest_version}")
}
```

### Maven

```xml
<dependency>
    <groupId>com.alibaba.opensandbox</groupId>
    <artifactId>sandbox</artifactId>
    <version>{latest_version}</version>
</dependency>
```

## 快速开始

Java 示例使用 Java 11+。快速开始会创建一个 sandbox、运行一条 shell 命令，并释放远程与本地资源。

::: tip
运行此示例前，请确保 OpenSandbox 服务已在运行。启动说明参见[快速上手](/getting-started/)指南。
:::

```java
import com.alibaba.opensandbox.sandbox.Sandbox;
import com.alibaba.opensandbox.sandbox.config.ConnectionConfig;
import com.alibaba.opensandbox.sandbox.domain.exceptions.SandboxException;
import com.alibaba.opensandbox.sandbox.domain.models.execd.executions.Execution;

public class QuickStart {
    public static void main(String[] args) {
        // 1. Configure connection
        ConnectionConfig config = ConnectionConfig.builder()
            .domain("api.opensandbox.io")
            .apiKey("your-api-key")
            .build();

        // 2. Create a Sandbox using try-with-resources
        try (Sandbox sandbox = Sandbox.builder()
                .connectionConfig(config)
                .image("ubuntu")
                .build()) {

            try {
                Execution execution = sandbox.commands().run("echo 'Hello Sandbox!'");
                System.out.println(execution.getLogs().getStdout().get(0).getText());
            } finally {
                sandbox.kill();
            } // try-with-resources closes the client even if kill() fails.

        } catch (SandboxException e) {
            // Handle Sandbox specific exceptions
            System.err.println("Sandbox Error: [" + e.getError().getCode() + "] " + e.getError().getMessage());
            System.err.println("Request ID: " + e.getRequestId());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### Kotlin

```kotlin
import com.alibaba.opensandbox.sandbox.Sandbox
import com.alibaba.opensandbox.sandbox.config.ConnectionConfig

fun main() {
    val config = ConnectionConfig.builder()
        .domain("api.opensandbox.io")
        .apiKey("your-api-key")
        .build()
    Sandbox.builder().image("ubuntu").connectionConfig(config).build().use { sandbox ->
        try {
            val result = sandbox.commands().run("echo 'Hello Sandbox!'")
            result.logs.stdout.forEach { print(it.text) }
        } finally {
            sandbox.kill()
        }
    }
}
```

`use` / try-with-resources 释放本地客户端。`kill()` 终止远程 sandbox。其余示例使用 Java；Kotlin 调用相同的方法。

## 生命周期钩子

在 `Sandbox.Builder` 上配置生命周期钩子。`preStart` 在入口点启动之前完成，而 `periodic` 钩子在启动后按各自的计划运行。

```java
import com.alibaba.opensandbox.sandbox.domain.models.sandboxes.LifecycleHook;
import com.alibaba.opensandbox.sandbox.domain.models.sandboxes.PeriodicLifecycleHook;
import com.alibaba.opensandbox.sandbox.domain.models.sandboxes.SandboxLifecycle;

SandboxLifecycle lifecycle = SandboxLifecycle.builder()
    .preStart(LifecycleHook.builder()
        .command("sh", "-c", "echo ready > /tmp/prestart.done")
        .timeoutSeconds(120)
        .build())
    .periodic(PeriodicLifecycleHook.builder()
        .name("checkpoint")
        .schedule("@every 5m")
        .command("sh", "-c", "date -u >> /tmp/checkpoints.log")
        .timeoutSeconds(120)
        .build())
    .build();

Sandbox sandbox = Sandbox.builder()
    .connectionConfig(config)
    .image("ubuntu:24.04")
    .lifecycle(lifecycle)
    .build();
```

Server 会校验 `timeoutSeconds`；`preStart` 接受 1–10800 秒，而 `periodic` 接受 1–300 秒。两者在省略时均默认为 60 秒。计时、失败行为和 provider 限制参见[生命周期钩子](/guides/lifecycle-hooks)。

### 空闲策略（自动暂停与访问唤醒）

`SandboxLifecycle.idlePolicy`（OSEP-0024，仅 fast-sandbox）告诉控制面在 ingress 观测到连续空闲流量达到 `idleTimeoutSeconds`（30–86400）后自动暂停 sandbox，并在下一个请求到来时唤醒它。它属于控制面状态：永远不会到达 guest，在模板模式（`TemplateLauncher.idlePolicy(...)`）下被接受，并被非 fast-sandbox 后端拒绝。省略它则完全保持非自动化行为。

```kotlin
val sandbox = Sandbox.builder()
    .connectionConfig(config)
    .image("ubuntu:24.04")
    .lifecycle(
        SandboxLifecycle.builder()
            .idlePolicy {
                idleTimeoutSeconds(300)
                onIdle(OnIdleAction.PAUSE)
                wakeOnAccess(true)
            }
            .build()
    )
    .build()
```

`activeProbe` 让应用在真实工作运行但不产生 ingress 流量时否决暂停：声明一个以 `{"active": bool, "message"?: string}` 应答的端点，当 sandbox 除此之外处于空闲状态时 server 会轮询它；忙碌的应答会重置空闲时钟。无效策略会在 `IdlePolicy.Builder` 中快速失败。

### 唤醒重试

对已暂停 sandbox 的请求可能在其恢复仍在进行时到达。网关以 `OpenSandbox-Wake: expired` 标记应答 `503`；重试拦截器识别该标记并在 `Retry-After` 延迟之后重发原始请求（对任何方法都安全 —— 该标记证明请求从未被转发）。唤醒预算默认在所有唤醒周期内为 5 次尝试 / 60 秒；预算耗尽时 SDK 抛出 `SandboxWakeTimeoutException`（携带 `attempts` 和 `waited`），而 server 侧的恢复在后台继续。裸 `503` 响应遵循既有重试规则、不受影响。可在 `RetryPolicy` 上调整：`wakeRetryEnabled`（默认 true）、`wakeMaxAttempts`（5）、`wakeOverallDeadline`（60 秒，`null` 表示无上限）。

## 使用示例

使用快速开始中的存活 `sandbox` 和 `config`，并在清理之前运行。Java 代码片段应放在一个处理或声明受检异常的方法内部；import 放在文件顶部。常用 import 为：

```java
import java.time.Duration;
import java.util.*;
import com.alibaba.opensandbox.sandbox.SandboxManager;
import com.alibaba.opensandbox.sandbox.domain.models.sandboxes.*;
import com.alibaba.opensandbox.sandbox.domain.models.execd.executions.*;
import com.alibaba.opensandbox.sandbox.domain.models.execd.filesystem.*;
```

各创建示例互为替代。请关闭每个客户端，并终止不再需要的 sandbox。

### 1. 生命周期管理

管理 sandbox 生命周期，包括续期、暂停和恢复。

```java
// Renew the sandbox
// This resets the expiration time to (current time + duration)
sandbox.renew(Duration.ofMinutes(30));

// Request pause (runtime-dependent)
sandbox.pause();

long deadline = System.nanoTime() + Duration.ofMinutes(2).toNanos();
while (true) {
    if (System.nanoTime() >= deadline) throw new IllegalStateException("Pause timed out");
    SandboxInfo info = sandbox.getInfo();
    if (SandboxState.PAUSED.equals(info.getStatus().getState())) break;
    if (SandboxState.FAILED.equals(info.getStatus().getState())) {
        throw new IllegalStateException(info.getStatus().getMessage());
    }
    Thread.sleep(1000);
}

// Resume returns a new local handle.
try (Sandbox resumed = Sandbox.resumer()
    .sandboxId(sandbox.getId())
    .connectionConfig(config)
    .resume()) {
    System.out.println("State: " + resumed.getInfo().getStatus().getState());
}
```

暂停是异步的且取决于运行时。参见[暂停与恢复](/guides/pause-resume)。要附加到运行中的 sandbox，请使用 `Sandbox.connector().sandboxId(id).connectionConfig(config).connect()`。

通过传入 `timeout(null)` 创建不过期的 sandbox：

```java
Sandbox manual = Sandbox.builder()
    .connectionConfig(config)
    .image("ubuntu")
    .timeout(null)
    .build();
```

### 2. 自定义健康检查

解析端点只能确认路由存在；它不能确认该端口上的应用是健康的。对于服务就绪，应对应用的健康端点发起有边界的请求，并携带返回的端点头。

自定义判断 sandbox 是否健康的逻辑。这会覆盖默认的 ping 检查。请在自定义检查内部设置超时；SDK 无法中断它们。

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

// This HTTP probe uses Java 11+.
HttpClient http = HttpClient.newBuilder().connectTimeout(Duration.ofSeconds(2)).build();
Sandbox sandbox = Sandbox.builder()
    .connectionConfig(config)
    .image("nginx:latest")
    .entrypoint(List.of("nginx", "-g", "daemon off;"))
    .healthCheck(sbx -> {
        try {
            SandboxEndpoint endpoint = sbx.getEndpoint(80);
            HttpRequest.Builder request = HttpRequest.newBuilder()
                .uri(URI.create(config.getProtocol() + "://" + endpoint.getEndpoint() + "/"))
                .timeout(Duration.ofSeconds(2)).GET();
            endpoint.getHeaders().forEach(request::header);
            return http.send(request.build(), HttpResponse.BodyHandlers.discarding()).statusCode() == 200;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("Health check interrupted", e);
        } catch (Exception e) {
            return false;
        }
    })
    .build();
```

### 3. 命令执行与流式输出

实时执行命令并处理输出流。

```java
// Create handlers for streaming output
ExecutionHandlers handlers = ExecutionHandlers.builder()
    .onStdout(msg -> System.out.println("STDOUT: " + msg.getText()))
    .onStderr(msg -> System.err.println("STDERR: " + msg.getText()))
    .onExecutionComplete(complete ->
        System.out.println("Command finished in " + complete.getExecutionTimeInMillis() + "ms")
    )
    .build();

// Execute command with handlers
RunCommandRequest request = RunCommandRequest.builder()
    .command("for i in {1..5}; do echo \"Count $i\"; sleep 0.5; done")
    .handlers(handlers)
    .build();

sandbox.commands().run(request);
```

要执行不带 shell 解析的原生程序，请传入参数列表。在 Linux 上，此示例打印字面的 `$HOME` 并将 `hello world` 作为一个参数保留：

```java
sandbox.commands().run(RunCommandRequest.builder()
    .argv(List.of("printf", "%s\n", "$HOME", "hello world"))
    .build());
```

原生 argv 执行需要更新版本的 execd。可执行文件查找与平台行为参见[命令执行模式](/architecture/data-plane/execd#command-execution)。

#### 后台命令

轮询增量日志和状态。命令超时与 sandbox TTL 相互独立。

```java
Execution execution = sandbox.commands().run(RunCommandRequest.builder()
    .command("for i in 1 2 3; do echo step-$i; sleep 1; done")
    .background(true).timeout(Duration.ofSeconds(30)).build());
String commandId = Objects.requireNonNull(execution.getId(), "No command ID returned");
Long cursor = 0L;
long deadline = System.nanoTime() + Duration.ofSeconds(45).toNanos();
while (true) {
    if (System.nanoTime() >= deadline) {
        sandbox.commands().interrupt(commandId);
        throw new IllegalStateException("Command did not finish");
    }
    CommandStatus status = sandbox.commands().getCommandStatus(commandId);
    CommandLogs logs = sandbox.commands().getBackgroundCommandLogs(commandId, cursor);
    System.out.print(logs.getContent());
    if (logs.getCursor() != null) cursor = logs.getCursor();
    if (Boolean.FALSE.equals(status.getRunning())) {
        if (!Integer.valueOf(0).equals(status.getExitCode())) {
            throw new IllegalStateException("Command failed: " + status.getExitCode() + ", " + status.getError());
        }
        break;
    }
    Thread.sleep(500);
}
```

#### 持久 shell 会话

Bash 会话可跨命令保留 shell 变量和工作目录。

```java
String sessionId = sandbox.commands().createSession("/tmp");
try {
    sandbox.commands().runInSession(sessionId, RunInSessionRequest.builder()
        .command("export DEMO=hello").build());
    Execution result = sandbox.commands().runInSession(sessionId, RunInSessionRequest.builder()
        .command("echo \"$DEMO\"; pwd").build());
    result.getLogs().getStdout().forEach(message -> System.out.print(message.getText()));
} finally {
    sandbox.commands().deleteSession(sessionId);
}
```

#### 持久环境变量

设置由运行时注入到每个后续命令和会话中的环境变量 —— 无需手写 shell 转义去操作 sandbox env 文件。

```java
sandbox.commands().setEnv("MY_TOKEN", "it's a safe value");
```

键必须匹配 `[A-Za-z_][A-Za-z0-9_]*`。不含单引号的值按原样存储；含单引号的值使用 env 文件的双引号形式，其中 shell 风格的 `$NAME` 序列可能在运行时加载该文件时被展开。env 文件是追加式的：同一键的最后一次写入生效。如果 sandbox 未能持久化该变量，将抛出 `SandboxException`。

关于 sandbox 内的文件系统/进程隔离，参见[隔离会话](/guides/isolation-sessions)。它们与 Bash 会话是相互独立的概念。

### 4. 文件操作

管理文件和目录，包括读取、写入、列出、删除和搜索。

```java
// 1. Write file
sandbox.files().write(List.of(
    WriteEntry.builder()
        .path("/tmp/hello.txt")
        .data("Hello World")
        .mode(644)
        .build()
));

// 2. Read file
String content = sandbox.files().readFile("/tmp/hello.txt", "UTF-8", null);
System.out.println("Content: " + content);

// List immediate children; search filters by a filename pattern.
sandbox.files().listDirectory("/tmp").forEach(entry -> System.out.println(entry.getPath()));

// 3. Search files
List<EntryInfo> files = sandbox.files().search(
    SearchEntry.builder()
        .path("/tmp")
        .pattern("*.txt")
        .build()
);
files.forEach(f -> System.out.println("Found: " + f.getPath()));

// 4. Delete file
sandbox.files().deleteFiles(List.of("/tmp/hello.txt"));
```

#### 带显式身份的文件操作

将文件系统客户端绑定到 Linux UID 和 GID：

```kotlin
val files = sandbox.filesWithIdentity(1001, 2000)
```

绑定的客户端暴露常规文件操作，并共享 sandbox 连接、认证和端点路由头。既有文件客户端保持其默认身份。命令执行时请单独选择 UID/GID。

这需要 Linux 以及支持文件系统执行身份的 Execd 版本。不支持的服务器会返回错误；请求绝不会回退到默认文件系统路由。请将 sandbox 凭证和身份选择保留在可信后端。Linux 文件系统权限依然生效；该特性并不创建租户隔离边界。权限与 workspace 供给详情参见 [Execd](/architecture/data-plane/execd)。

### 5. Sandbox 管理（管理面）

使用 `SandboxManager` 执行管理任务和查找既有 sandbox。

```java
try (SandboxManager manager = SandboxManager.builder()
    .connectionConfig(config)
    .build()) {
    // First page only; increase page for subsequent pages.
    PagedSandboxInfos sandboxes = manager.listSandboxInfos(
        SandboxFilter.builder().states(SandboxState.RUNNING).pageSize(10).page(1).build()
    );
    sandboxes.getSandboxInfos().forEach(info -> System.out.println(info.getId()));
}
```

### 资源指标

使用 `sandbox.getMetrics()` 读取当前 sandbox 的资源使用。这与 [SDK 创建遥测](/sdks/observability#creation-metrics)相互独立。

### 6. Client Pool 与可观测性

`SandboxPool` 提供四种获取策略和分阶段 warmup 控制。单进程使用 `InMemoryPoolStateStore`，或在分布式部署中使用可选的 `com.alibaba.opensandbox:sandbox-pool-redis` 模块配合调用方自管的 Jedis 客户端。配置、示例、清理和命名空间退役参见 [Client Pool](/guides/client-pool)。

设置 `ConnectionConfig.builder().enableTracing(true)` 以输出 [pool warmup 追踪](/sdks/observability#pool-warmup-tracing)。JVM SDK 还会将 trace ID 加入 SLF4J MDC。远程日志/事件请使用 `SandboxManager.getDiagnosticLogs` / `getDiagnosticEvents`；参见[诊断](/api/#diagnostics)。创建延迟上报由 [SDK 遥测](/sdks/observability#creation-metrics)单独控制。

## 快照与元数据

使用 `sandbox.createSnapshot` 或 `manager.createSnapshot` 创建快照。`SandboxManager` 暴露 `getSnapshot`、`listSnapshots`、`deleteSnapshot` 以及用于等待异步快照完成的 `waitForSnapshotReady`。使用 `Sandbox.builder().snapshotId(snapshotId).connectionConfig(config).build()` 恢复。

使用 `sandbox.patchMetadata` 或 `manager.patchSandboxMetadata` 新增/替换元数据键；`null` 值移除键。运行时支持情况由[生命周期契约](/api/#1-sandbox-lifecycle-yml)描述。

快照支持取决于运行时与 server 配置。如果源 sandbox 的剩余 TTL 可能在快照创建期间到期，请先续期。JVM 辅助方法会等待 `Ready`，并在失败或超时时抛出异常：

```java
try (SandboxManager manager = SandboxManager.builder().connectionConfig(config).build()) {
    SnapshotInfo snapshot = sandbox.createSnapshot("demo");
    System.out.println("Snapshot: " + snapshot.getId());
    manager.waitForSnapshotReady(snapshot.getId(), Duration.ofMinutes(15));
    try (Sandbox restored = Sandbox.builder()
            .snapshotId(snapshot.getId()).connectionConfig(config).build()) {
        try {
            System.out.println(restored.getId());
        } finally {
            restored.kill();
        }
    }
    // The snapshot is retained. When no longer needed: manager.deleteSnapshot(snapshot.getId());
}
```

元数据更新可以同时包含替换值和移除。请使用接受 null 值的 map（`Map.of` 不行）：

```java
Map<String, String> patch = new HashMap<>();
patch.put("project", "demo");
patch.put("obsolete-key", null);
sandbox.patchMetadata(patch);
```

## 配置

### 1. 连接配置

`ConnectionConfig` 类管理 API server 的连接设置。

::: info 传输超时
`requestTimeout` 限制每个传输阶段 —— 建立连接、等待响应头、以及等待更多响应体 —— 但它永远不会限制持续有进展的健康文件上传或下载。停滞时间超过该超时的传输、或请求体已发出但在超时内未收到响应头的上传，会失败而不是一直挂起（即 Go SDK 的 `ResponseHeaderTimeout` 契约）。显式的 `RetryPolicy.overallDeadline` 仍然会限制整个传输。
:::

| 参数 | 说明 | 默认值 | 环境变量 |
| ---------------- | ------------------------------------------ | ---------------------------- | ---------------------- |
| `apiKey` | 用于认证的 API Key | 可选；启用 server 认证时需要 | `OPEN_SANDBOX_API_KEY` |
| `domain` | sandbox 服务的端点域 | `localhost:8080` | `OPEN_SANDBOX_DOMAIN`  |
| `protocol` | HTTP 协议（http/https） | `http`                       | -                      |
| `requestTimeout` | API 请求超时 | 30 秒 | -                      |
| `debug` | 启用 HTTP 请求的调试日志 | `false`                      | -                      |
| `headers` | 自定义 HTTP 头 | 空 | -                      |
| `connectionPool` | 共享的 OKHttp ConnectionPool | 每个实例由 SDK 创建 | -                      |
| `retryPolicy` | 非流式请求的自动重试策略（参见[自动重试](#_2-automatic-retries)） | 启用（`RetryPolicy()`） | -                 |
| `useServerProxy` | 将 sandbox server 用作 execd/端点请求的代理（例如客户端无法直接访问 sandbox 时） | `false` | -                      |
| `disableMetrics` | 禁用 SDK 创建延迟遥测（参见 [SDK 遥测](/sdks/observability#creation-metrics)） | `false` | `OPENSANDBOX_DISABLE_METRICS` |
| `enableTracing` | 为 pool warmup 启用 OpenTelemetry 追踪（参见 [SDK 追踪](/sdks/observability#pool-warmup-tracing)） | `false` | - |

```java
import okhttp3.ConnectionPool;
import java.util.concurrent.TimeUnit;

// 1. Basic configuration
ConnectionConfig config = ConnectionConfig.builder()
    .apiKey("your-key")
    .domain("api.opensandbox.io")
    .requestTimeout(Duration.ofSeconds(60))
    .build();

// 2. Advanced: Shared Connection Pool
// If you create many Sandbox instances, sharing a connection pool is recommended to save resources.
// SDK default keep-alive is 30 seconds for its own pools.
ConnectionPool sharedPool = new ConnectionPool(50, 30, TimeUnit.SECONDS);

ConnectionConfig sharedConfig = ConnectionConfig.builder()
    .apiKey("your-key")
    .domain("api.opensandbox.io")
    .headers(Map.of(
        "X-Custom-Header", "value",
        "X-Request-ID", "trace-123"
    ))
    .connectionPool(sharedPool) // Inject shared pool
    .build();
```

::: tip SDK 遥测
`Sandbox.builder()...build()` 默认将创建延迟上报到 `POST /v1/metrics/events`。调用 `ConnectionConfig.builder().disableMetrics(true)` 或导出 `OPENSANDBOX_DISABLE_METRICS=1` 可选择退出。参见 [SDK 遥测](/sdks/observability#creation-metrics)。
:::

### 2. 自动重试

SDK 会自动重试瞬时故障。`ConnectionConfig` 会在 SDK 的非流式 HTTP 客户端上安装 `RetryInterceptor`（`com.alibaba.opensandbox.sandbox.transport.RetryPolicy`）。

默认行为：

- **默认启用。** 幂等方法（`GET/HEAD/PUT/DELETE/OPTIONS`）在 `429`、`502`、`503` 以及发送前的传输失败（DNS、TCP 连接、TLS 握手）时会被重试。
- **`POST`/`PATCH` 默认绝不因状态码被重试**，因为请求可能已在 server 侧生效。这些方法在发送前的传输失败（尚未写出任何字节）时仍会被重试。
- 最多 `3` 次重试，采用去相关抖动的指数退避，并遵循 server 的 `Retry-After` 头（上限 60 秒）。
- **SSE / 流式请求绕过一切自动重试**，因为其请求体无法安全重放。SSE 客户端还禁用了 OkHttp 内置的连接恢复，以防止流式命令 POST 被重放。

::: warning 行为变更
SDK 策略重试默认开启。与较早的 SDK 版本相比，这可能增加 HTTP 尝试次数和尾延迟。要禁用新的 SDK 策略重试，请使用 `RetryPolicy.disabled()`；非流式请求将回退到 OkHttp 既有的内置连接恢复。
:::

```java
import com.alibaba.opensandbox.sandbox.transport.RetryPolicy;
import com.alibaba.opensandbox.sandbox.transport.StatusCode;
import java.time.Duration;
import java.util.Set;

// Disable SDK-policy retries and retain OkHttp's built-in connection recovery.
ConnectionConfig config = ConnectionConfig.builder()
    .apiKey("your-key")
    .domain("api.opensandbox.io")
    .retryPolicy(RetryPolicy.disabled())
    .build();

// Custom policy: more retries, an overall wall-clock deadline, and an opt-in to
// retry POST/PATCH on 503 (only safe if your endpoints are idempotent).
ConnectionConfig tuned = ConnectionConfig.builder()
    .apiKey("your-key")
    .domain("api.opensandbox.io")
    .retryPolicy(new RetryPolicy(
        /* maxRetries */ 5,
        /* initialBackoff */ Duration.ofMillis(500),
        /* maxBackoff */ Duration.ofSeconds(30),
        /* backoffMultiplier */ 2.0,
        /* jitter */ com.alibaba.opensandbox.sandbox.transport.JitterMode.DECORRELATED,
        /* retryableStatusCodesIdempotent */ RetryPolicy.DEFAULT_IDEMPOTENT_STATUS,
        /* retryableStatusCodesNonIdempotent */ Set.of(StatusCode.SERVICE_UNAVAILABLE),
        /* perAttemptTimeout */ null,
        /* overallDeadline */ Duration.ofSeconds(20),
        /* onRetry */ null))
    .build();
```

### 3. Sandbox 创建配置

`Sandbox.builder()` 允许配置 sandbox 环境。

| 参数 | 说明 | 默认值 |
| -------------- | ---------------------------------------- | ------------------------------- |
| `image` | 要使用的 Docker 镜像 | image 或快照 ID 二选一 |
| `timeout` | 自动终止超时 | 10 分钟 |
| `entrypoint` | 容器入口命令 | `["tail", "-f", "/dev/null"]`   |
| `resource` | CPU 与内存限制 | `{"cpu": "1", "memory": "2Gi"}` |
| `env` | 环境变量 | 空 |
| `metadata` | 自定义元数据标签 | 空 |
| `extensions` | 不透明的 server 侧扩展参数 | 空 |
| `networkPolicy` | 可选的出站网络策略（egress） | -                             |
| `credentialProxy` | 可选的 Credential Vault 代理启动设置 | -                     |
| `readyTimeout` | 等待 sandbox 就绪的最长时间 | 30 秒 |
| `snapshotId` | 恢复快照而不是使用镜像 | - |
| `resourceRequests` | Kubernetes 资源请求；不得超过限制 | - |
| `lifecycle` | 启动前与周期性钩子 | - |
| `platform` | OS/架构约束 | - |
| `volumes` | Host、PVC 或 OSSFS 挂载 | - |
| `secureAccess` | 要求端点访问凭证 | `false` |

::: warning
`opensandbox.io/` 下的元数据键保留给系统管理的标签，将被 server 拒绝。
:::

```java
import com.alibaba.opensandbox.sandbox.domain.models.sandboxes.NetworkPolicy;
import com.alibaba.opensandbox.sandbox.domain.models.sandboxes.NetworkRule;

Sandbox sandbox = Sandbox.builder()
    .connectionConfig(config)
    .image("python:3.11")
    .timeout(Duration.ofMinutes(30))
    .resource(Map.of("cpu", "2", "memory", "4Gi"))
    .env("PYTHONPATH", "/app")
    .metadata("project", "demo")
    .extension("storage.id", "dataset-001")
    .networkPolicy(
        NetworkPolicy.builder()
            .defaultAction(NetworkPolicy.DefaultAction.DENY)
            .addEgress(
                NetworkRule.builder()
                    .action(NetworkRule.Action.ALLOW)
                    .target("pypi.org")
                    .build()
            )
            .build()
    )
    .build();
```

### 4. 运行时 Egress 策略更新

运行时的 egress 读取和修补直接发送到 sandbox 的 egress sidecar。SDK 首先解析 sandbox 端口 `18080` 上的端点，然后调用 sidecar 的 `/policy` API。

基于模板的 sandbox 没有 sandbox 侧的 egress sidecar：SDK 通过 server 的 `OPEN-SANDBOX-ORIGIN` 响应头检测它们（参见 [Fsb 模板管理](#fsb-template-management)），并改为通过生命周期控制面（`/sandboxes/{sandboxId}/networkpolicy`）路由相同的 `getEgressPolicy` / `patchEgressRules` / `deleteEgressRules` 调用。

修补使用合并语义：
- 传入规则优先于具有相同 `target` 的既有规则。
- 其他目标的既有规则保持不变。
- 在单个修补载荷内，某个 `target` 的第一条规则生效。
- 当前的 `defaultAction` 被保留。

```java
NetworkPolicy policy = sandbox.getEgressPolicy();

sandbox.patchEgressRules(
    List.of(
        NetworkRule.builder().action(NetworkRule.Action.ALLOW).target("www.github.com").build(),
        NetworkRule.builder().action(NetworkRule.Action.DENY).target("pypi.org").build()
    )
);
```

### 5. Credential Vault

Credential Vault 从 egress sidecar 注入出站凭证，同时让真实密钥远离 sandbox 的环境变量、命令、文件和日志。以 `credentialProxyEnabled(true)` 创建 sandbox，然后通过 `sandbox.credentialVault()` 写入凭证和绑定。

```java
import com.alibaba.opensandbox.sandbox.domain.models.sandboxes.Credential;
import com.alibaba.opensandbox.sandbox.domain.models.sandboxes.CredentialAuth;
import com.alibaba.opensandbox.sandbox.domain.models.sandboxes.CredentialBinding;
import com.alibaba.opensandbox.sandbox.domain.models.sandboxes.CredentialMatch;
import com.alibaba.opensandbox.sandbox.domain.models.sandboxes.CredentialVaultCreateRequest;
import com.alibaba.opensandbox.sandbox.domain.models.sandboxes.NetworkPolicy;
import com.alibaba.opensandbox.sandbox.domain.models.sandboxes.NetworkRule;
import java.util.List;

Sandbox sandbox = Sandbox.builder()
    .connectionConfig(config)
    .image("python:3.11")
    .networkPolicy(
        NetworkPolicy.builder()
            .defaultAction(NetworkPolicy.DefaultAction.DENY)
            .addEgress(
                NetworkRule.builder()
                    .action(NetworkRule.Action.ALLOW)
                    .target("api.example.com")
                    .build()
            )
            .build()
    )
    .credentialProxyEnabled(true)
    .build();

sandbox.credentialVault().create(
    CredentialVaultCreateRequest.builder()
        .credentials(
            List.of(
                Credential.builder()
                    .name("api-token")
                    .inlineSource("<token>")
                    .build()
            )
        )
        .bindings(
            List.of(
                CredentialBinding.builder()
                    .name("api-token")
                    .match(
                        CredentialMatch.builder()
                            .schemes(CredentialMatch.Scheme.HTTPS)
                            .hosts("api.example.com")
                            .paths("/v1/*")
                            .build()
                    )
                    .auth(CredentialAuth.apiKey("x-api-key", "api-token"))
                    .build()
            )
        )
        .build()
);
```

认证类型、绑定指南以及 Git/curl 示例参见 [Credential Vault](/guides/credential-vault)。

::: warning
Credential Vault 对基于模板的 sandbox 不可用：它们没有 sandbox 侧的 egress sidecar。对它们调用 `sandbox.credentialVault()` 会抛出异常。
:::

## Fsb 模板管理

fsb（fast-sandbox microVM）黄金镜像模板通过 `SandboxManager` 管理。模板构建是异步的：`createTemplate` 返回时 `status.phase` 为 `Pending`；轮询 `getTemplate` 直到阶段达到 `Succeeded` 或 `Failed`。只有 `Succeeded` 的模板才能创建 sandbox。这些操作需要已配置的 Fsb 运行时。请将示例发布 URI 替换为你的 server 所配置的位置，并使用通过 `SandboxManager.builder().connectionConfig(config).build()` 创建且处于打开状态的 `SandboxManager`。用完后请关闭 manager。

```java
import com.alibaba.opensandbox.sandbox.domain.models.sandboxes.CreateTemplateRequest;
import com.alibaba.opensandbox.sandbox.domain.models.sandboxes.TemplateFilter;
import com.alibaba.opensandbox.sandbox.domain.models.sandboxes.TemplateInfo;
import com.alibaba.opensandbox.sandbox.domain.models.sandboxes.TemplatePhase;
import com.alibaba.opensandbox.sandbox.domain.models.sandboxes.TemplateReadiness;

CreateTemplateRequest request = CreateTemplateRequest.builder()
    .image("alpine:3.19")
    .publish("s3://bucket/publish")
    .resourceLimits(Map.of("cpu", "1", "memory", "512Mi", "disk", "2Gi"))
    .readiness(TemplateReadiness.builder().probe("tcp://127.0.0.1:44772").build())
    .metadata(Map.of("team", "backend"))
    .build();

// Start the async build (starts at TemplatePhase.PENDING)
TemplateInfo template = manager.createTemplate(request);

// Poll for up to 15 minutes; adjust for the image and runtime.
long deadline = System.nanoTime() + Duration.ofMinutes(15).toNanos();
while (!template.getStatus().getPhase().equals(TemplatePhase.SUCCEEDED)
    && !template.getStatus().getPhase().equals(TemplatePhase.FAILED)) {
    if (System.nanoTime() >= deadline) throw new IllegalStateException("Template build timed out");
    Thread.sleep(2000);
    template = manager.getTemplate(template.getTemplateId());
}
if (TemplatePhase.FAILED.equals(template.getStatus().getPhase())) {
    throw new IllegalStateException("Template build failed: " + template.getTemplateId());
}

// List with metadata filters (1-indexed paging)
manager.listTemplates(
    TemplateFilter.builder()
        .metadata(Map.of("team", "backend"))
        .pageSize(20)
        .page(1)
        .build()
);

// When no longer needed: manager.deleteTemplate(template.getTemplateId());
```

### 从模板创建 Sandbox

使用 `Sandbox.fromTemplate()` 从 `Succeeded` 的模板创建 sandbox。模板模式在 server 侧固定了工作负载形态：模板 ID 之外只允许 `metadata`、`networkPolicy` 和 `extensions`，且 `timeout` 为必填。

```java
import java.time.Duration;

Sandbox sandbox = Sandbox.fromTemplate()
    .connectionConfig(config)
    .templateId("tpl_123")
    .timeout(Duration.ofMinutes(30))
    .metadata("project", "demo")
    .networkPolicy(
        NetworkPolicy.builder()
            .defaultAction(NetworkPolicy.DefaultAction.DENY)
            .addEgress(
                NetworkRule.builder()
                    .action(NetworkRule.Action.ALLOW)
                    .target("pypi.org")
                    .build()
            )
            .build()
    )
    .create();
```

创建出的 sandbox 通过 `sandbox.getOrigin()` 报告 `SandboxOrigin.TEMPLATE`。基于模板的 sandbox 没有 egress sidecar，因此 SDK 会自动通过生命周期控制面路由其 egress 策略 —— 包括 `Sandbox.connector()` 和 `Sandbox.resumer()` 的重新附加流程，它们从 server 的 `OPEN-SANDBOX-ORIGIN` 响应头检测来源。
