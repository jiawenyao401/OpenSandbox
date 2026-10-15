---
title: SDK 可观测性
description: 配置 SDK 创建指标与 OpenTelemetry pool warmup 追踪。
---

# SDK 可观测性

使用创建指标跟踪 sandbox 启动延迟，使用 pool warmup 追踪定位缓慢阶段。它们拥有各自独立的控制和导出路径：

| 信号 | 用途 | SDK 支持 | 默认 | 导出路径 |
| --- | --- | --- | --- | --- |
| [Pool warmup 追踪](#pool-warmup-tracing) | 检查创建、就绪、准备、续期和空闲发布 | Python、JavaScript/TypeScript、Kotlin/Java | 关闭；在连接配置中启用追踪 | 应用自有的 OpenTelemetry provider 与 exporter |
| [创建指标](#creation-metrics) | 衡量 sandbox 创建延迟与成功 | 全部五个 SDK | 开启；通过配置或 `OPENSANDBOX_DISABLE_METRICS=1` 禁用 | SDK 上报到生命周期 server，server 在启用 `[otel]` 时导出指标 |

OpenTelemetry（OTel）提供追踪与指标基础设施；启用追踪不会启用或禁用创建指标。远程 sandbox 日志与事件参见[诊断 API](/api/#diagnostics)。

## Pool warmup 追踪

Python、JavaScript/TypeScript 和 Kotlin/Java SDK 可以为客户端侧的 `SandboxPool` warmup 路径输出 [OpenTelemetry](https://opentelemetry.io/) 追踪。每个 warmup 任务成为一条追踪，覆盖完整生命周期 —— 从 reconcile 循环提交任务开始，到预热好的 sandbox 被提交到空闲缓冲为止 —— 并带有按阶段的 span，帮助你找到真正的 warmup 瓶颈。

追踪是**可选启用**的（Python 中 `enable_tracing=True`、JavaScript 中 `enableTracing: true`，或 JVM 上 `.enableTracing(true)`），并且是**尽力而为**的：如果应用中没有 OpenTelemetry SDK + exporter，所有 span 调用都是 no-op，不会导出任何内容。追踪绝不影响 pool 行为。

### SDK 支持

| SDK | Pool warmup 追踪 |
| --- | --- |
| Python async/sync | 阶段 span、sandbox 身份、终态分类、就绪计数器 |
| JavaScript/TypeScript | 阶段 span、pool 身份、成功/失败及错误类型 |
| Kotlin/Java | 阶段 span、sandbox 身份、终态分类、就绪计数器、SLF4J MDC |
| Go / C# | 无内置 pool warmup 追踪 |

本指南描述默认分支。启用之前请检查你所安装 SDK 的 `ConnectionConfig` 中是否存在 `enable_tracing` / `enableTracing`。追踪默认**关闭**，独立于默认**开启**的[创建延迟遥测](/sdks/observability#creation-metrics)。

### 启用追踪

#### 1. 为应用添加 OpenTelemetry SDK + exporter

这些 SDK 仅依赖 OpenTelemetry API（默认 no-op）。要真正导出追踪，你需要自带 SDK 和 exporter。Python：

```bash
pip install opentelemetry-sdk opentelemetry-exporter-otlp-proto-http
```

Node.js 请安装与应用兼容的 OpenTelemetry SDK 和 exporter。参见 [OpenTelemetry Node.js 配置](https://opentelemetry.io/docs/languages/js/getting-started/nodejs/)。

Kotlin/Java：

```kotlin
dependencies {
    implementation("io.opentelemetry:opentelemetry-api:1.51.0")
    implementation("io.opentelemetry:opentelemetry-sdk:1.51.0")
    implementation("io.opentelemetry:opentelemetry-exporter-otlp:1.51.0")
}
```

#### 2. 配置全局 OpenTelemetry provider

warmup span 使用各语言的全局 provider。请在应用启动时、创建或启动 pool 之前配置它。在 JavaScript 中，向全局 `@opentelemetry/api` 实例注册 provider、async 上下文管理器和 W3C propagator（Node.js 的 OpenTelemetry SDK 可以一次性完成这些配置）。Python 使用 `opentelemetry.trace.set_tracer_provider(...)`；Kotlin/Java 配置 `GlobalOpenTelemetry`，例如：

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;

SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .addSpanProcessor(BatchSpanProcessor.create(
        OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://otel-collector:4317")
            .build()))
    .build();

OpenTelemetrySdk sdk = OpenTelemetrySdk.builder()
    .setTracerProvider(tracerProvider)
    .build();

GlobalOpenTelemetry.set(sdk);
```

::: tip Propagators
`OpenTelemetrySdk.builder()` 默认使用 **noop propagators**。如果你希望 SDK 向生命周期请求注入 W3C `traceparent` 头（使生命周期 server 在支持追踪后能加入同一条追踪），请显式配置 W3C 传播：

```java
.setPropagators(ContextPropagators.create(W3CTraceContextPropagator.getInstance()))
```
:::

::: tip 采样
为控制追踪量，可在 `SdkTracerProvider` 上使用 `parentbased_traceidratio(0.1)` 之类的采样策略。trace-id 比例采样能让同一次 warmup 的客户端与 server span 保持一致。
:::

#### 3. 为 pool 开启追踪

JavaScript / TypeScript：

```ts
import { ConnectionConfig } from "@alibaba-group/opensandbox";

const config = new ConnectionConfig({ enableTracing: true });
// Pass config as connectionConfig when constructing SandboxPool.
```

Python：

```python
from opensandbox.config import ConnectionConfig

config = ConnectionConfig(enable_tracing=True)
```

Kotlin/Java：

```java
ConnectionConfig config = ConnectionConfig.builder()
    .enableTracing(true)
    .build();

SandboxPool pool = SandboxPool.builder()
    .poolName("demo-pool")
    .maxIdle(3)
    .stateStore(new InMemoryPoolStateStore())
    .connectionConfig(config)
    .creationSpec(PoolCreationSpec.builder().image("ubuntu:22.04").build())
    .build();
```

将该连接配置传给 pool。SDK 开关默认为 `false`；OpenTelemetry exporter 与采样配置仍归应用所有。请在冲刷并关闭应用 tracer provider 之前先关闭 pool。

### 追踪了什么

每个 warmup 任务产生**一条追踪**，包含一个根 span 和六种可能的阶段类型（它们是根下的兄弟节点，因此各阶段时长可以独立比较）。每个就绪阶段由一个 span 汇总其全部延迟尝试；未配置的可选阶段不会出现：

| Span 名称 | 覆盖范围 |
|-----------|--------|
| `pool.warmup`（根） | Warmup 任务，从准入到完成。Python/JVM 会把它回溯到提交时间以包含排队等待 |
| `pool.warmup.create` | Sandbox 创建器的调用。内置生命周期路径只做一次 HTTP 尝试；就绪不再是该 span 的一部分 |
| `pool.warmup.readiness` | 完整的准备前就绪阶段（`warmupHealthCheck` 或 `ping`），包括全部延迟尝试 |
| `pool.warmup.prepare` | `warmupSandboxPreparer` 的单次调用（用户初始化脚本 / 设置工作） |
| `pool.warmup.post_prepare_readiness` | 完整的可选准备后验证阶段，包括全部延迟尝试 |
| `pool.warmup.renew` | 提交 sandbox 前的 TTL 续期 |
| `pool.warmup.commit` | 主锁续期 + 针对状态存储的 `putIdle` |

#### 各语言的属性

三个 SDK 都会输出 `pool.name`、`pool.owner`、`pool.run.generation` 和 `pool.leader.epoch`。JavaScript 目前额外输出 `warmup.result`（`success` 或 `failure`）和 `warmup.error.type`；它不输出 sandbox 身份、终态分类或下文的就绪尝试计数器。仅凭其 success 标志无法证明 sandbox 已被提交到空闲缓冲。

以下更丰富的根属性适用于 **Python 和 Kotlin/Java**：

| 属性 | 值 |
|-----------|-------|
| `pool.name` | Pool 名称 |
| `pool.owner` | Pool 所有者 ID |
| `pool.run.generation` | Pool 运行世代 |
| `pool.leader.epoch` | 本次 warmup 被准入时捕获的 leader 世代 |
| `sandbox.id` | 创建进度足以取得 sandbox ID 时的 sandbox ID |
| `sandbox.image` | 创建镜像 |
| `warmup.stage` | 终态阶段：`admission`、`create`、`readiness`、`prepare`、`post_prepare_readiness`、`renew` 或 `commit` |
| `warmup.result` | `success`、`failure`、`dropped` 或 `cancelled` |
| `warmup.reason` | 结果不成功时的稳定终态原因 |
| `warmup.error.category` | 稳定的错误类别，如 `rate_limit`、`http_4xx`、`http_5xx`、`timeout`、`connection`、`callback` 或 `state_store` |
| `warmup.error.type` | 可用时为异常类名 |

Python/JVM 的就绪汇总 span 还会暴露 `warmup.health.attempt_count`、`warmup.health.false_count`、`warmup.health.exception_count` 和 `warmup.scheduler.delay_ms`。失败会通过 `recordException` 记录在受影响的阶段 span 上。根 span 保留分类后的终态阶段、结果、原因和 OpenTelemetry 错误状态，而不重复阶段异常事件。

::: warning 开发快照属性迁移
较早的开发快照使用不带命名空间的 `result` 和 `drop.reason` 属性。受支持的 schema 在追踪和结构化日志中一致地使用 `warmup.result` 和 `warmup.reason`。旧键不会并行输出；请更新任何基于开发快照创建的仪表盘。
:::

### 将日志关联到追踪

在 Kotlin/Java 中，warmup 追踪进行期间，pool 会将 trace ID 发布到 SLF4J [MDC](https://www.slf4j.org/api/org/slf4j/MDC.html)：

| MDC 键 | 值 |
|---------|-------|
| `trace_id` | 当前 trace ID |
| `span_id` | 当前 span ID |

MDC 需要真实的 SLF4J provider（logback、log4j2 等）。在日志 pattern 中添加这些键一次，pool 的每条日志行就都会携带追踪上下文：

```xml
<pattern>%d %-5level [%thread] %logger{36} trace_id=%X{trace_id} span_id=%X{span_id} - %msg%n</pattern>
```

### 查询追踪

trace ID 是随机的，因此 warmup 追踪无法直接"按 pool 名"查找。可靠的途径有：

1. **JVM 日志关联。** 配置 MDC 后，pool 会在其 warmup 日志行上记录 `pool_name` 和 `sandbox_id`（例如 `Pool warmup sandbox entered idle`）。在日志中搜索某个 `sandbox_id` —— 匹配的日志行携带 `trace_id`，可直接在追踪后端中打开。
2. **在追踪后端中按属性查询。** 按时间窗口和属性过滤 span，例如 TraceQL `{ span.pool.name = "demo-pool" }`（Grafana Tempo），或在 Jaeger 中按 `pool.name=...` 做标签搜索。从 span 派生指标的后端（Tempo metrics、Datadog span analytics）允许先查看每个 `pool.name` 的 `pool.warmup` 时长百分位，再下钻到慢追踪。
3. **Trace-id 比例采样。** 在采样开启时，日志中的 `trace_id` 与后端对同一次 warmup 是一致的。

#### 瓶颈下钻

下文的就绪计数器和回溯排队计时适用于 Python/JVM。对 JavaScript，请比较阶段时长和四个 pool 身份属性。

```
pool.warmup root duration (p50/p95/p99) per pool.name
  └─ phase spans: create / readiness / prepare / post_prepare_readiness / renew / commit
       └─ single trace: root start gap = queue wait, then each phase duration
```

| 症状 | 可能原因 |
|---------|--------------|
| `pool.warmup.create` 之前的长间隙 | 创建任务在等待执行器线程；将 `warmupCreateQps` 与创建延迟比较 |
| 创建与第一个就绪 span 之间的长间隙 | 预期的 `warmupHealthCheckInitialDelay`，或因 `warmupConcurrency` 过低导致延迟阶段容量耗尽 |
| `pool.warmup.create` 缓慢 | 生命周期创建 API 缓慢（例如镜像拉取 / execd 启动） |
| `pool.warmup.readiness` 缓慢且 `warmup.health.attempt_count` 偏高 | sandbox 启动或配置的就绪谓词是瓶颈 |
| `pool.warmup.prepare` 缓慢 | 你的 `warmupSandboxPreparer` 工作是瓶颈 |
| `pool.warmup.post_prepare_readiness` 缓慢且尝试次数偏高 | 已准备的服务尚未健康，或其验证谓词缓慢 |
| `pool.warmup.renew` 缓慢 | 生命周期 API 的 TTL 续期 |
| `pool.warmup.commit` 缓慢 | 状态存储的租约续期或空闲发布（例如 Redis 往返） |

## 创建指标

OpenSandbox SDK 默认将 sandbox 创建延迟上报到所配置的生命周期 server。此上报与可选启用的 [pool warmup 追踪](/sdks/observability#pool-warmup-tracing)相互独立。上报是尽力而为的：失败绝不影响 `Sandbox.create`，且载荷不包含用户内容。

### 要求

`POST /v1/metrics/events` 端点和下述 SDK 上报器要求以下最低版本。较旧的 SDK 只是不发出事件；较旧的 server 会以 `404` 拒绝未知路由，SDK 会静默吞掉（参见下文[版本偏差](#version-skew)）。

| 组件 | 最低版本 |
|-----------|-----------------|
| Server（`opensandbox-server`） | `0.2.2` |
| Python SDK（`opensandbox`） | `0.1.15` |
| JavaScript / TypeScript SDK（`@alibaba-group/opensandbox`） | `0.1.11` |
| Go SDK（`github.com/alibaba/OpenSandbox/sdks/sandbox/go`） | `1.0.5` |
| C# SDK（`Alibaba.OpenSandbox`） | `0.1.5` |
| Kotlin / Java SDK（`com.alibaba.opensandbox:sandbox`） | `1.0.17` |

#### 版本偏差

每个 SDK 的上报都是"发出即忘"（fire-and-forget）的：POST 在后台任务/线程上运行，上报失败被忽略。调试日志因 SDK 而异；例如 JavaScript 不会记录这些失败。这意味着你可以独立升级 SDK 和 server：

- **新 SDK、旧 server（`< 0.2.2`）**：server 对 `/v1/metrics/events` 返回 `404`。SDK 忽略该响应。`Sandbox.create` 行为不变，不会产生用户可见的错误。视 SDK 而定可能输出一条 debug 消息。
- **旧 SDK、新 server**：SDK 不发出事件。server 直方图对该客户端不记录任何内容。
- **网络错误、TLS 故障、超时**：与 `404` 情况相同 —— 被吞掉，`Sandbox.create` 不受影响。

### 发送了什么

创建成功或失败后，SDK 以"发出即忘"的方式 POST 到 `POST /v1/metrics/events`：

```json
{
  "eventType": "sandbox.create",
  "sandboxId": "sbx_...",
  "image": "python:3.12",
  "createDurationMs": 1842,
  "success": true
}
```

- 创建早期失败时可能省略 `sandboxId` / `image`。
- SDK 语言和版本来自 HTTP `User-Agent` 头（例如 `OpenSandbox-Python-SDK/0.1.15`），而非请求体字段。

server 以 `204` 接受该事件，并在启用 `[otel]` 时记录 OTEL 直方图。参见 [server 配置](https://github.com/opensandbox-group/OpenSandbox/blob/main/server/configuration.md#otel)。

### 何时运行

| SDK | 触发时机 |
|-----|---------|
| Python（async + sync） | `Sandbox.create` / 同步创建完成或抛出异常之后 |
| JavaScript / TypeScript | `Sandbox.create` 完成或失败之后 |
| Go | `CreateSandbox` 完成或失败之后 |
| C# | `Sandbox.CreateAsync` 完成或失败之后 |
| Kotlin | 独立的 `Sandbox.builder()...build()` 或 pool 直接创建回退完成或失败之后 |

::: info Kotlin 分阶段 pool warmup
Kotlin 的分阶段 warmup 刻意**不**发出旧版 `sandbox.create` 事件。其创建阶段在就绪轮询、可选准备、准备后验证、续期和空闲提交之前即返回，因此把该部分阶段当作既有的端到端创建直方图上报，会使该指标的含义与独立创建不同。

此排除仅适用于分阶段 warmup。独立创建和 pool 直接创建回退仍正常上报。请使用 pool 的结构化摘要日志和可选的 [warmup 追踪](/sdks/observability#pool-warmup-tracing)来观测完整的分阶段 warmup 生命周期。
:::

### 如何禁用

默认开启。通过以下任一方式退出：

1. 环境变量（所有 SDK）：

```bash
export OPENSANDBOX_DISABLE_METRICS=1
```

2. 连接配置字段：

::: code-group

```python [Python]
from opensandbox import Sandbox
from opensandbox.config import ConnectionConfig

config = ConnectionConfig(disable_metrics=True)
sandbox = await Sandbox.create("python:3.12", connection_config=config)
```

```typescript [JavaScript]
import { ConnectionConfig, Sandbox } from "@alibaba-group/opensandbox";

const connectionConfig = new ConnectionConfig({ disableMetrics: true });
const sandbox = await Sandbox.create({
  image: "python:3.12",
  connectionConfig,
});
```

```go [Go]
cfg := opensandbox.ConnectionConfig{DisableMetrics: true}
sandbox, err := opensandbox.CreateSandbox(ctx, cfg, opensandbox.SandboxCreateOptions{
    Image: "python:3.12",
})
```

```csharp [C#]
using OpenSandbox;
using OpenSandbox.Config;

var connectionConfig = new ConnectionConfig(new ConnectionConfigOptions
{
    DisableMetrics = true,
});
var sandbox = await Sandbox.CreateAsync(new SandboxCreateOptions
{
    Image = "python:3.12",
    ConnectionConfig = connectionConfig,
});
```

```kotlin [Kotlin]
import com.alibaba.opensandbox.sandbox.Sandbox
import com.alibaba.opensandbox.sandbox.config.ConnectionConfig

val connectionConfig = ConnectionConfig.builder()
    .disableMetrics(true)
    .build()
val sandbox = Sandbox.builder()
    .image("python:3.12")
    .connectionConfig(connectionConfig)
    .build()
```

:::

这些请求会发送到你所配置的生命周期 server。如果你不希望记录这些额外流量或创建延迟数据，请使用退出选项。
