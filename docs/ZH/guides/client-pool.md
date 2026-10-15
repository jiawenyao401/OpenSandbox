---
title: 客户端池
description: SDK 侧沙箱池的工作原理、配置方式，以及每种受支持 SDK 的最小示例。
---

# 客户端池

OpenSandbox SDK 提供了一个实验性的**客户端侧沙箱池**，它在服务器上保持少量预热就绪的沙箱缓冲，使 `acquire()` 能够快速返回，而不必在热路径上支付完整的沙箱创建延迟。

支持 Python（async 与 sync）、JavaScript/TypeScript、Kotlin/Java 和 Go。
C# 目前不提供客户端池。

::: warning 实验性
客户端池 API 被标记为实验性，可能在小版本之间发生变化。如果你在生产环境中依赖它，请固定 SDK 版本。
:::

## 它实际池化的是什么

池**不会**池化 SDK 的 `Sandbox` 对象。它池化的是**运行在 OpenSandbox 服务器上已预热、就绪的沙箱 ID**。

将池用于反复需要新沙箱且对延迟敏感的 worker。池保持远程沙箱就绪；它与 HTTP 连接池是相互独立的。借出的沙箱仍然消耗服务器资源，必须显式清理。

![客户端池架构](../public/images/client-pool-architecture.svg)

两条流程并发进行：

- **预热（仅 leader）。** 每个节点上都会运行一个后台对账循环。持有主锁的节点计算空闲缺口并补充它。Python、JavaScript 和 Kotlin 采用名义上一秒的节奏，且每个完成的 tick 最多接纳 `warmup_create_qps` 次新创建。Kotlin 以固定速率调度；JavaScript 在上一次对账仍在运行时跳过该 tick；Python 在上一个 tick 完成后等待一秒。Go 使用可配置的 `reconcile_interval`，并用 `warmup_concurrency` 限制每个 tick。成功的预热会独立发布到空闲缓冲中，TTL 为 `idle_timeout`。
- **获取（任意节点）。** `acquire()` 从存储中弹出一个空闲 ID，将 `Sandbox` 客户端连接到它，可选地运行健康检查和 `renew()` 直到调用者提供的超时时间，然后将其交给调用者。非 leader 节点可以自由获取；只有补充和收缩受 leader 锁限制。

空闲成员包含沙箱 ID 及其过期时间，绝不是 HTTP 连接或客户端的 `Sandbox` 对象。状态存储还协调 leader 租约、共享的空闲目标和 TTL，以及命名空间销毁围栏。基于 Redis 的存储使这种协调和空闲成员在进程和 Pod 之间可见。

预热路径——即上面的仅 leader 补充流程——值得深入了解，因为它是池中唯一由分布式锁控制的部分：

![预热对账时序](../public/images/client-pool-warmup-sequence.svg)

### 生命周期模型

在优雅运行期间，每个池实例依次经历 `NOT_STARTED → STARTING → RUNNING → DRAINING → STOPPED`。启动失败或非优雅关闭可以直接转换为 `STOPPED`。
如果在等待 Python 的 `SandboxPoolAsync.start()` 时被取消，它会等待启动所拥有的资源清理完成，即使调用者在清理期间反复取消。清理成功后，池会回到 `STOPPED`，然后传播 `CancelledError`，因此同一实例之后可以再次启动。
如果在进一步取消后清理本身失败，`CancelledError` 具有优先权，并将清理错误保留为其原因。如果没有进一步取消，清理错误会正常传播。无论哪种情况，生命周期状态都是 `STOPPED`，允许再次尝试 `start()`，而不是静默地停留在 `STARTING`。关闭失败时资源清理可能仍然不完整。
健康状况单独跟踪为 `HEALTHY | DEGRADED | DRAINING | STOPPED`；在连续 `degraded_threshold` 次预热或对账失败后，池进入 `DEGRADED`。Go 在降级时应用指数补充退避。Python、JavaScript 和 Kotlin 继续以其名义一秒节奏尝试对账：`warmup_create_qps` 是它们的压力控制手段，`snapshot().backoff_active` / `snapshot().backoffActive` 仅为兼容性保留，且始终为 `false`。
调用者无需直接观察这些状态——`snapshot()` 将其暴露出来用于诊断。

Python、JavaScript 和 Kotlin 内置预热创建只发出一次生命周期请求。Python 和 Kotlin 针对HTTP 429、其他可重试状态和传输恢复禁用连接级重试策略；JavaScript 的生命周期传输不添加自动重试。没有池级的 `Retry-After` 节流。自定义 `PooledSandboxCreator` 通过 `PooledSandboxCreateContext.connection_config` 或 `createConnectionConfig` 接收预热连接配置，并且必须遵守它以及 `skipHealthCheck`，以保持分阶段行为。失败的创建会被记录，后续的 tick 可以接纳替换工作。普通的独立创建和 `AcquirePolicy.DIRECT_CREATE` 保留其通常的传输行为。

![客户端池生命周期状态机](../public/images/client-pool-lifecycle.svg)

### 没有 `release()`

沙箱是临时的。一旦你调用了 `acquire()`，沙箱就归你所有，直到你 `destroy()` / `kill()` 它。`max_idle` 限制的是**温热缓冲**，而不是应用代码借用的沙箱数量，也不是 `DIRECT_CREATE` 回退产生的沙箱数量。

在 Python 的 `SandboxPoolAsync` 中，所有权在 `acquire()` 返回时转移。如果调用在连接到或创建沙箱后失败或被取消，池会尝试杀死该沙箱，并在传播错误之前关闭其本地资源。续订期间或最终池状态检查期间的取消（包括清理期间的重复取消）不会跳过该清理。一旦获取成功，调用者仍然负责处置。

## 空缓冲行为：`AcquirePolicy`

全部四个池 SDK 都暴露这些策略。获取会消耗一个候选；它不会等待未来的预热工作来填满空缓冲。

| 策略 | 尝试的空闲候选 | 候选用尽时 |
| --- | --- | --- |
| `FAIL_FAST` | 最多一个 | 返回池为空/获取错误 |
| `DIRECT_CREATE`（默认） | 最多一个 | 创建一个新沙箱 |
| `RETRY_NEXT_IDLE` | 最多 `max_acquire_retries` | 返回池为空/获取错误 |
| `RETRY_NEXT_IDLE_THEN_CREATE` | 最多 `max_acquire_retries` | 创建一个新沙箱 |

`max_acquire_retries` / `maxAcquireRetries` / `MaxAcquireRetries` 默认为 `3`。它限制的是**总候选尝试次数**，而不是第一次之后的额外三次重试。每个候选可能消耗获取就绪预算；直接创建会增加其自身的启动延迟。

`start()` 开始后台补充，但不等待缓冲填满。启动期间使用 `DIRECT_CREATE`，或在使用 `FAIL_FAST` 之前观察空闲数量。池不是并发限制：直接创建和已借出的沙箱可能超过 `max_idle`。

## 配置

各 SDK 共享池的概念，但它们的调度接口不同。下表是规范参考；确切的 camelCase / snake_case 命名请参考各语言的构建器或构造函数。

| 参数 | Python / JavaScript / Kotlin 默认值 | Go 默认值 | 含义 |
| --- | --- | --- | --- |
| `pool_name` | 必填 | 必填 | 一个分布式池所有节点共享的逻辑命名空间 |
| `owner_id` | 自动（`pool-owner-<uuid>`） | 自动（`pool-owner-<host/pid>`） | 此进程用于主锁所有权的身份；**每个节点必须唯一** |
| `max_idle` | 必填（≥ 0） | 必填（≥ 0） | 空闲缓冲的目标大小和上限 |
| `state_store` | Python/Kotlin 必填；JavaScript 默认为内存 | 构建器默认为内存 | `InMemoryPoolStateStore` 或基于 Redis 的存储 |
| `connection_config` | 必填 | 必填 | 用于生命周期和 execd 调用 |
| `creation_spec` | Python/Kotlin 必填；JavaScript 仅在没有 creator 时必填 | 仅在未设置 `sandbox_creator` 时必填 | 预热沙箱的模板；确切字段因语言而异 |
| `sandbox_creator` | `null` | `null` | 可选回调，覆盖默认创建。Python 和 Kotlin 仍需要 `creation_spec`；JavaScript 和 Go 允许仅有 creator 的池。 |
| `warmup_create_qps` | `10` | 不可用 | 每个完成的对账 tick 允许的最大预热创建数 |
| `warmup_concurrency` | `128` | `max(1, ceil(max_idle * 0.2))` | Python / JavaScript / Kotlin：创建后的并发工作；不控制创建准入。Go：每个 tick 的创建上限和工作线程并发。 |
| `primary_lock_ttl` | `60 秒` | `60 秒` | Leader 租约 TTL |
| `reconcile_interval` | 名义 `1 秒`，未暴露 | `30 秒`，可配置 | 对调节奏；较长的 Python/JavaScript tick 会降低有效速率 |
| `degraded_threshold` | `3` | `3` | 进入 `DEGRADED` 前的连续失败次数；只有 Go 以退避暂停补充 |
| `acquire_ready_timeout` | `30 秒` | `30 秒` | 等待返回的沙箱就绪的最长时间 |
| `acquire_health_check_polling_interval` | `200 毫秒` | `200 毫秒` | 获取期间的就绪轮询间隔 |
| `acquire_health_check` | `null` | `null` | 获取的自定义就绪谓词 |
| `acquire_skip_health_check` | `false`；JavaScript 也有每次获取的覆盖 | 每次获取的选项 | 在获取时跳过就绪检查 |
| `acquire_min_remaining_ttl` | `min(60 s, idle_timeout / 2)` | `min(60 s, idle_timeout / 2)` | 在获取时丢弃比此更接近过期的空闲项 |
| `warmup_ready_timeout` | `30 秒` | `30 秒` | 预热沙箱的最大就绪检查窗口 |
| `warmup_health_check_initial_delay` | `0 秒` | 不可用 | 成功创建与第一次就绪检查之间的延迟 |
| `warmup_health_check_polling_interval` | `500 毫秒` | `200 毫秒` | 预热期间的就绪轮询间隔；Python / JavaScript / Kotlin 也将其用于创建后检查 |
| `warmup_health_check` | `null` | `null` | 自定义预热就绪谓词 |
| `warmup_sandbox_preparer` | `null` | `null` | 在就绪后、发布到空闲缓冲前运行一次 |
| `warmup_post_prepare_health_check` | `null` | 不可用 | preparer 之后的可选验证；重试不会重新运行 preparer |
| `warmup_post_prepare_health_check_timeout` | `30 秒` | 不可用 | 创建后验证的重试窗口 |
| `warmup_skip_health_check` | `false` | `false` | 在预热期间跳过准备前的就绪阶段 |
| `idle_timeout` | `24 小时` | `24 小时` | 池创建沙箱的服务器端 TTL |
| `drain_timeout` | `30 秒` | `30 秒` | 优雅关闭期间等待进行中操作的最长时间 |

### JavaScript 设置与 creator 覆盖范围

JavaScript 使用上面的 Python/JVM 调度默认值：`warmupCreateQps: 10`、`warmupConcurrency: 128`、名义上一秒的对调节奏、500 毫秒预热轮询和 30 秒就绪窗口。其存储默认为 `InMemoryPoolStateStore`；提供 `sandboxCreator` 时可以省略 `creationSpec`。时间设置的名称中包含单位，例如 `idleTimeoutSeconds` 和 `warmupHealthCheckInitialDelayMillis`。

默认池 creator 暴露的选项少于独立沙箱创建：

| 创建字段 | Python | JavaScript | Kotlin/Java | Go |
| --- | --- | --- | --- | --- |
| 快照恢复 | 否 | 是 | 否 | 是 |
| 资源配置 | 是 | 是 | 是 | 是 |
| 生命周期钩子 | 否 | 是 | 否 | 否 |
| 凭证代理（Credential Proxy） | 否 | 是 | 是 | 是 |

全部四个独立沙箱创建 API 都接受这些字段。当默认池 spec 不够用时，使用自定义 `sandbox_creator` / `sandboxCreator`，并遵守提供的连接配置、就绪控制和超时。

JavaScript 和 Go 还在每次获取时接受最小剩余 TTL 和跳过健康检查的覆盖。Python 和 Kotlin 将这些暴露为池级配置。

### 分阶段预热

分阶段预热将创建准入与创建后工作分离：

![分阶段预热流程](../public/images/client-pool-staged-warmup.svg)

1. 在每个对账 tick 上，leader 最多接纳 `min(max_idle - idle - warming, warmup_create_qps)` 次创建。生命周期创建请求只尝试一次，并返回客户端而不运行其正常的内联就绪循环。创建执行限制是内部的，解析为 `ceil(warmup_create_qps * 1.5)`，因此慢创建无法无界增长活跃创建工作。自定义 creator 必须遵守其 `PooledSandboxCreateContext` 提供的预热连接配置和 `skipHealthCheck`。
2. 创建的沙箱进入延迟阶段队列。第一次就绪检查在 `warmup_health_check_initial_delay` 之后运行；失败则每隔 `warmup_health_check_polling_interval` 重试，直到 `warmup_ready_timeout`，包括截止时间时的最后一次检查。
3. `warmup_sandbox_preparer` 运行一次。如果配置了，`warmup_post_prepare_health_check` 然后以相同的轮询间隔重试，直到 `warmup_post_prepare_health_check_timeout`；重试绝不重新运行 preparer。
4. 健康的沙箱被续订并提交到空闲缓冲。`warmup_concurrency` 限制活跃的创建后工作。不成功就绪轮询之间的延迟不会永久占用全部容量。

没有 Python、JavaScript 或 Kotlin 的 `reconcile_interval` 设置，也没有补充退避。对于使用旧设置的 Python/Kotlin 配置，删除 `reconcile_interval=...` / `reconcileInterval(...)`，选择 `warmup_create_qps` / `warmupCreateQps(...)` 来控制创建准入，并将 `warmup_concurrency` / `warmupConcurrency(...)` 仅用于创建后容量。

`warmup_create_qps` 是准入上限，不是完成率保证。创建延迟、就绪、准备、状态存储延迟、事件循环或执行器容量，以及延迟/跳过的对账 tick，都可能降低观察到的预热吞吐量。

### 选择状态存储

- **`InMemoryPoolStateStore`** — 仅限单进程。适用于开发、测试和单实例 worker。对于 gunicorn/uvicorn worker、Celery 或 Kubernetes 副本，它不是进程范围的。
- **基于 Redis 的存储**（`RedisPoolStateStore`、`AsyncRedisPoolStateStore`、JVM 上的 `sandbox-pool-redis`、Go 中的 `poolredis`、JavaScript 中的 `@alibaba-group/opensandbox/pool-redis`）— 多进程或多 Pod 部署必需。一个逻辑池中的所有节点必须共享相同的 `pool_name` 和 Redis `key_prefix`，并且每个进程必须使用**唯一**的 `owner_id`。

![单节点与分布式池拓扑](../public/images/client-pool-topology.svg)

### 适用于所有部署的规则

- `max_idle` 仅限制温热缓冲。它不限制借出的沙箱或 `DIRECT_CREATE` 回退。
- 共享一个池的所有节点必须使用相同的创建和预热定义。如果该定义发生变化，请在**新的** `pool_name`（或 Redis `key_prefix`）下推出并退役旧命名空间（见下文"退役旧池命名空间"）。不要尝试将更改后的模板重新填充到相同的 `pool_name` 中：`release_all_idle()` 不会围栏其他节点，不会降低 `max_idle`，也不会阻止任何当前 leader（可能仍在运行旧代码）在滚动部署期间立即将旧模板沙箱 ID 重新发布到共享缓冲。
- `resize(max_idle)` 和 `release_all_idle()` 可以从任何节点调用。
- 将 `primary_lock_ttl` 配置为高于 `warmup_ready_timeout` 加上预期的 preparer 时间和运维余量。SDK 独立续订领导权并对每次提交进行围栏，但丢失租约会导致进行中的预热被丢弃而不是发布。

## 最小用法

### Python (sync)

```python
from datetime import timedelta

from opensandbox import (
    AcquirePolicy,
    InMemoryPoolStateStore,
    PoolCreationSpec,
    SandboxPoolSync,
)
from opensandbox.config import ConnectionConfigSync

pool = SandboxPoolSync(
    pool_name="demo-pool",
    owner_id="worker-1",
    max_idle=2,
    state_store=InMemoryPoolStateStore(),
    connection_config=ConnectionConfigSync(domain="api.opensandbox.io"),
    creation_spec=PoolCreationSpec(image="ubuntu:22.04"),
    warmup_create_qps=10,
    warmup_concurrency=128,
)

pool.start()
try:
    sandbox = pool.acquire(
        sandbox_timeout=timedelta(minutes=30),
        policy=AcquirePolicy.DIRECT_CREATE,
    )
    try:
        result = sandbox.commands.run("echo pool-ok")
        print(result.logs.stdout[0].text)
    finally:
        sandbox.destroy()
finally:
    pool.shutdown(graceful=True)
```

### Python (asyncio)

`SandboxPoolAsync` 具有相同的接口，外加一个 `async with` 上下文管理器：

```python
from datetime import timedelta

from opensandbox import (
    AcquirePolicy,
    InMemoryAsyncPoolStateStore,
    PoolCreationSpec,
    SandboxPoolAsync,
)
from opensandbox.config import ConnectionConfig

async with SandboxPoolAsync(
    pool_name="demo-pool",
    owner_id="worker-1",
    max_idle=2,
    state_store=InMemoryAsyncPoolStateStore(),
    connection_config=ConnectionConfig(domain="api.opensandbox.io"),
    creation_spec=PoolCreationSpec(image="ubuntu:22.04"),
) as pool:
    sandbox = await pool.acquire(
        sandbox_timeout=timedelta(minutes=30),
        policy=AcquirePolicy.DIRECT_CREATE,
    )
    try:
        result = await sandbox.commands.run("echo pool-ok")
    finally:
        await sandbox.destroy()
```

### Kotlin / Java

```java
SandboxPool pool = SandboxPool.builder()
    .poolName("demo-pool")
    .ownerId("worker-1")
    .maxIdle(3)
    .stateStore(new InMemoryPoolStateStore())
    .connectionConfig(config)
    .creationSpec(PoolCreationSpec.builder()
        .image("ubuntu:22.04")
        .entrypoint(List.of("tail", "-f", "/dev/null"))
        .build())
    .warmupReadyTimeout(Duration.ofSeconds(45))
    .build();

pool.start();
try {
    Sandbox sb = pool.acquire(Duration.ofMinutes(10), AcquirePolicy.DIRECT_CREATE);
    try {
        sb.commands().run("echo pool-ok");
    } finally {
        try {
            sb.kill();
        } finally {
            sb.close();
        }
    }
} finally {
    pool.shutdown(true);
}
```

### Go

在返回 `error` 的函数内，带有 `ctx` 和通常的 `context`、`fmt`、`time` 和 `opensandbox` 导入：

```go
pool, err := opensandbox.NewSandboxPoolBuilder().
    PoolName("demo-pool").
    OwnerID("worker-1").
    MaxIdle(3).
    ConnectionConfig(opensandbox.ConnectionConfig{Domain: "api.opensandbox.io"}).
    CreationSpec(opensandbox.PoolCreationSpec{Image: "ubuntu:22.04"}).
    StateStore(opensandbox.NewInMemoryPoolStateStore()).
    Build()
if err != nil {
    return err
}
if err := pool.Start(ctx); err != nil {
    return err
}
defer pool.Shutdown(context.Background(), true)

policy := opensandbox.AcquirePolicyDirectCreate
sb, err := pool.Acquire(ctx, opensandbox.AcquireOptions{
    SandboxTimeout: 10 * time.Minute,
    Policy:         &policy,
})
if err != nil {
    return err
}
defer sb.Close()
defer sb.Kill(context.Background())

result, err := sb.RunCommand(ctx, "echo pool-ok", nil)
if err != nil {
    return err
}
fmt.Println(result.Logs.Stdout)
return nil
```

### JavaScript / TypeScript

```ts
import { AcquirePolicy, SandboxPool } from "@alibaba-group/opensandbox";

const pool = SandboxPool.create({
  poolName: "demo-pool",
  maxIdle: 2,
  connectionConfig: { domain: "localhost:8080", useServerProxy: true },
  creationSpec: { image: "python:3.12" },
  warmupCreateQps: 10,
  warmupConcurrency: 128,
});

await pool.start();
try {
  const sandbox = await pool.acquire({
    sandboxTimeoutSeconds: 600,
    policy: AcquirePolicy.DIRECT_CREATE,
  });
  try {
    const result = await sandbox.commands.run("echo pool-ok");
    console.log(result.logs.stdout);
  } finally {
    try {
      await sandbox.kill();
    } finally {
      await sandbox.close();
    }
  }
} finally {
  await pool.shutdown(true);
}
```

要跨进程共享状态，安装 `redis`，在应用中连接它，并将此存储作为 `stateStore` 传递：

```ts
import { createClient } from "redis";
import { RedisPoolStateStore } from "@alibaba-group/opensandbox/pool-redis";

const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();
const stateStore = new RedisPoolStateStore({
  client: redis,
  keyPrefix: "opensandbox:pool:prod",
});
// Pass stateStore to SandboxPool.create(...).
// After all pools/managers using it are shut down:
// await redis.quit();
```

调用者拥有 Redis 连接。自定义 JavaScript preparer 和健康检查必须自行限制其工作：取消可以停止等待回调，但不停止回调本身。

## 诊断与运维

每个池 SDK 都暴露这些操作（名称遵循语言约定）：

- `snapshot()` — 池阶段、健康状况、计数器（空闲大小、进行中的预热、连续失败次数、最后错误）。
- `snapshot_idle_entries()` — 当前空闲沙箱 ID 及其过期时间戳。
- `resize(max_idle)` — 在运行时更改目标缓冲大小。有关营业时间规模的调整，请参见[定时客户端池示例](/examples/client-pool-schedule)，它从应用级控制器调用 `Resize`。
- `release_all_idle()` — 清空当前可见的空闲缓冲，并尽力杀死每个条目，而不停止池。在临时上游问题后强制进行全新预热很有用。它**不会**更改 `max_idle`，**不会**围栏其他节点，也**不会**阻止活跃 leader 立即补充——因此它不是在同一 `pool_name` 上交换创建模板的安全方式。对于这种情况，请在新的 `pool_name` 下退役整个命名空间（见下文）。

现有的清理方法保留其原始执行行为。对于可选的有界并行清理，使用 Python 的 `release_all_idle_parallel(max_workers=50)`、Kotlin 的 `releaseAllIdle(concurrency)`、JavaScript 的 `releaseAllIdle(concurrency)`，或 Go 的具体方法 `(*DefaultSandboxPool).ReleaseAllIdleParallel(ctx, maxWorkers)`。这些方法验证正并发值，并等待每个被清空的 ID 都收到一次尽力而为的杀死尝试。Go 方法有意放在 `SandboxPool` 接口之外，以保持与第三方实现者的兼容性。

### 追踪预热

Python、JavaScript 和 Kotlin/Java 会发出可选的 OpenTelemetry 预热 span。在 Python 中设置 `enable_tracing=True`，或在 JavaScript/JVM 连接配置中设置 `enableTracing: true` / `.enableTracing(true)`。应用提供其 OpenTelemetry provider 和 exporter。阶段名称和特定语言的属性请参见 [SDK Tracing](/sdks/observability#pool-warmup-tracing)。

池 `shutdown` 停止该本地池实例；它不会终止已交给调用者的沙箱。要退役共享命名空间，请使用下面的管理器。

### 退役旧池命名空间

每个受支持的池 SDK 都暴露一个 `SandboxPoolManager`，其 `destroy` 操作应用相同的 `DESTROYING → DESTROYED` 协议：

1. 在状态存储中写入 `DESTROYING` 围栏，任何仍在运行的同伴实例都会看到它并停止补充，而不是与退役竞争。
2. 尽力清空并杀死每个空闲沙箱，受 drain 超时限制。
3. 清除持久化的每池状态。
4. 写入带墓碑 TTL（默认 7 天）的 `DESTROYED` 墓碑，这样未来的调用者无法静默重新绑定到相同的 `pool_name`。

Destroy 是幂等的：对已经墓碑化的命名空间调用它会报告 `DESTROYED`，而不清空或杀死任何东西。如果 drain 或清理无法完成，命名空间保持 `DESTROYING`，调用报告 destroy 未完成；重试是安全的，并从上次中断的地方继续。

**Python / Kotlin** — `SandboxPoolManager.destroy(poolName, options)`，通过 `PoolDestroyOptions`（`strategy`、`drain_timeout`、`tombstone_ttl`）配置。

**JavaScript** — `SandboxPoolManager.create({ stateStore, connectionConfig })`，然后 `await manager.destroy(poolName, options)`。

**Go** — `(*SandboxPoolManager).Destroy(ctx, poolName, options)`：

```go
manager, err := opensandbox.NewSandboxPoolManagerBuilder().
    StateStore(store).
    ConnectionConfig(connCfg).
    Build()
if err != nil {
    return err
}

result, err := manager.Destroy(ctx, "orders-v2", opensandbox.PoolDestroyOptions{})
if err != nil {
    return err
}
log.Printf("retired %s: drained=%d killed=%d",
    result.PoolName, result.DrainedIdleCount, result.KilledIdleCount)
```

`PoolDestroyOptions` 与其他 SDK 对应。`Strategy` 选择算法，且只实现了 `PoolDestroyForce`。`DrainTimeout` 和 `TombstoneTTL` 是 `*time.Duration`：将它们留空以使用默认值（30 秒和 7 天），或设置显式零值以在没有截止时间的情况下清空，并写入永不过期的墓碑。

围栏使退役变得安全，而不必先停止每个写入者，它在两个级别上被强制执行。状态存储以 `*PoolDestroyedError` 拒绝 `PutIdle`、`SetMaxIdle` 和 `SetIdleEntryTTL`，并且不发放主锁，这阻止了补充。池本身在启动时、每次获取之前、获取持有活跃沙箱时以及每个对账 tick 上都会检查围栏：存活的同伴在其下一个 tick 上完全停止，进行中的获取会失败，而不是通过直接创建回退在退役的命名空间中铸造新沙箱，而在围栏落地前刚获得的沙箱会被杀死而不是交出。获取后的检查很重要，因为空闲获取被有意留为无围栏，以便 `destroy` 可以清空：一旦 ID 被取走，`destroy` 就无法再触及它，因此获取者必须自行处置它。
针对已墓碑化的 `PoolName` 启动新池会因同样的原因失败，因此重新绑定该名称需要等待墓碑 TTL 过期或轮换到新的 `PoolName`。

一个有意的例外：如果状态存储本身不可达，destroy 状态便不可知，因此那些在存储中断时已经回退到直接创建的策略（`DIRECT_CREATE`、`RETRY_NEXT_IDLE_THEN_CREATE`）会假定 `ACTIVE` 并继续，与现有的 `try_take_idle` 中断行为一致。`FAIL_FAST` 和 `RETRY_NEXT_IDLE` 则呈现中断。这种放宽在已从空闲缓冲取走的沙箱处停止：那里的检查是 fail-closed 的，不可达的存储意味着沙箱会被杀死，因为不再有其他东西在跟踪它。

## 延伸阅读

- Python: [`/sdks/python`](/sdks/python) &mdash; `SandboxPoolSync`、`SandboxPoolAsync`、Redis 存储。
- JavaScript: [SDK 入口点](/sdks/javascript) — `SandboxPool` 和 `/pool-redis` 导出。
- Kotlin: [`/sdks/kotlin`](/sdks/kotlin) &mdash; `SandboxPool` 构建器、`sandbox-pool-redis` 模块。
- Go: [`/sdks/go`](/sdks/go) &mdash; `SandboxPool` 接口、`RedisPoolStateStore`、分布式部署说明。
