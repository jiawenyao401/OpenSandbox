---
title: 遥测
description: OpenSandbox 各组件的 OpenTelemetry 指标参考，包含每个指标的含义、属性和导出配置。
---

# 遥测

OpenSandbox 组件导出 OpenTelemetry（OTLP/HTTP）指标。本页是统一参考：各组件的指标及其含义与属性，以及控制导出的配置。SDK 侧的创建指标和 Pool 预热追踪单独在 [SDK 可观测性](/sdks/observability)中说明。

| 组件 | Meter / service 名称 | 指标族 |
| --- | --- | --- |
| [Lifecycle server](#lifecycle-server) | `opensandbox-server` | HTTP 请求延迟、SDK 上报的创建延迟 |
| [Kubernetes 控制器](#kubernetes-控制器) | `opensandbox/controller` | Allocator 直方图、容量 gauge（+ controller-runtime 端点） |
| [Execd](#execd) | `opensandbox/execd` | HTTP、执行、文件系统、系统资源、OOM、隔离 |
| [Egress sidecar](#egress-sidecar) | `opensandbox/egress` | DNS、策略、nftables、节点/进程资源 |
| [Ingress 网关](#ingress-网关) | `opensandbox-ingress-<version>` | HTTP/路由、上游拨号、网络就绪影子评估、按访问唤醒、主机 |
| [Node agent](#node-agent) | `opensandbox.nodeagent.*` | 日志管道的记录、字节、丢弃、重试、队列深度、sink 延迟 |

## Go 组件的共享 OTLP 配置

Execd、egress、ingress 和 node-agent 共用 `components/internal/telemetry` 中的 OTLP 指标导出器。请在**组件进程环境**中设置这些变量。在 lifecycle server 上或沙箱创建请求中设置它们，不会自动配置另一个容器（例如 egress sidecar）。Server 托管的 egress 端点请参见 [egress 可观测性](/architecture/network/egress#observability-opentelemetry)。

### 导出端点

- `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` 优先于 `OTEL_EXPORTER_OTLP_ENDPOINT`。
- 导出通过 HTTP/protobuf 面向 collector 的 HTTP receiver（通常是 4318 端口）。未实现 gRPC 导出，也未实现通过 `OTEL_EXPORTER_OTLP_PROTOCOL` 切换传输方式。
- 节点 IP 回退：当两个端点都未设置时，execd、egress 和 ingress 会回退到 `HOST_IP:4318`，或从 `/etc/hostinfo` 读取的节点 IP，使用纯 HTTP。如果解析不到节点 IP，则不导出指标。在共享遥测配置中设置了 `DisableEndpointFallback` 的组件（node-agent）会跳过该回退。因此，仅清除端点变量并不总是能禁用指标。

### 禁用指标

设置以下任一标准开关即可阻止指标导出，包括显式端点和节点 IP 回退：

```sh
OTEL_SDK_DISABLED=true
# Or disable metrics only:
OTEL_METRICS_EXPORTER=none
```

取值不区分大小写。其他取值会保持现有导出配置不变；`OTEL_METRICS_EXPORTER` 在这里不会选择替代的导出器。这些开关不会禁用原生应用日志或 execd 本地的 `/metrics` 和 `/metrics/watch` API。它们还会抑制 egress sidecar 的自动 collector 放行规则，包括生效策略重建时。运维提供的放行规则仍然生效。

### 聚合时间性（temporality）

设置 `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` 来选择导出器的标准聚合行为：

| 取值 | Counter 与 Histogram | ObservableCounter | UpDownCounter 与 ObservableUpDownCounter |
|---|---|---|---|
| `cumulative` | 累计 | 累计 | 累计 |
| `delta` | 增量 | 增量 | 累计 |
| `lowmemory` | 增量 | 累计 | 累计 |
| 未设置、空或仅空白 | 增量 | 累计 | 累计 |

取值不区分大小写。无效的非空取值会使用上游导出器的累计默认值，并产生其配置警告。Gauge 始终是瞬时测量。

Prometheus 风格的后端期望累计的 counter 和 histogram。要么在组件环境中设置 `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=cumulative`，要么在采集增量指标时为 Collector 配置 `deltatocumulative` processor。请在组件启动前确定设置；这些环境变量不支持热加载。

## Lifecycle server

可选的 OTLP 导出，面向 server 的 HTTP 请求和 SDK 上报的沙箱创建延迟（`POST /v1/metrics/events`）。默认关闭；HTTP 中间件和接收端点保持活跃，但记录为 no-op。

`[otel]` 下的配置键（权威清单见 [server 配置参考](https://github.com/opensandbox-group/OpenSandbox/blob/main/server/configuration.md)）：

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `enabled` | boolean | `false` | 启用 OTLP 指标导出。 |
| `endpoint` | string \| 省略 | `null` | OTLP HTTP 指标端点。省略时使用 `OTEL_EXPORTER_OTLP_ENDPOINT` / `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT`。 |
| `service_name` | string | `"opensandbox-server"` | `service.name` 资源属性。 |
| `export_interval_millis` | integer | `60000` | 周期导出间隔（≥ 1000）。 |

导出的指标：

| 指标 | 类型 | 单位 | 属性 | 说明 |
|-----|------|------|------------|-------------|
| `server.http.request.duration` | Histogram | `ms` | `http_method`, `http_route`, `http_status_code` | Server HTTP 请求延迟。Histogram 的 count 提供请求量。 |
| `opensandbox.sandbox.create.duration` | Histogram | `ms` | `sdk.language`, `sdk.version`, `success` | SDK 上报的创建延迟，从创建开始到就绪或失败。 |

HTTP 指标使用匹配到的路由模板而不是原始请求路径。未到达匹配路由的请求（包括早期认证失败和未匹配的 URL）使用 `http_route=unknown`。标准 HTTP 方法以大写记录，扩展方法使用 `http_method=OTHER`，以保持属性基数有界。该指标绝不包含沙箱 ID、租户 ID、API key、请求或响应体、查询字符串或其他无界的请求数据。

创建指标的客户端侧控制（`OPENSANDBOX_DISABLE_METRICS=1`）和 Pool 预热追踪，请参见 [SDK 可观测性](/sdks/observability#creation-metrics)。

## Kubernetes 控制器

沙箱控制器为 Pool 分配路径和控制器容量发出 OTLP 指标。设计原则：只保留核心信号（不做投机性埋点）、复用 controller-runtime 已暴露的内容、使用标准 OTel SDK 仪器，使测量能在控制器副本之间聚合。

所有自定义控制器指标通过一个进程级的全局 OTel meter provider 导出，该 provider 通过标准 OpenTelemetry 环境变量配置（参见[为控制器启用 OTLP 导出](#enabling-otlp-export-for-the-controller)）。没有控制器专用的开关。

### Allocator 路径指标

所有仪器都是 meter 作用域 `opensandbox/controller` 上的 `Float64Histogram`，以 `s`（秒）为单位记录，显式的桶边界为 `[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]`（秒）。成功/失败的区分通过 `success` 布尔属性表达；每个属性集上的 histogram `count` 提供成功/失败计数。

| 名称 | 属性 | 覆盖范围 | 发出位置 |
|---|---|---|---|
| `controller.allocator.schedule.duration` | `namespace`, `pool_name`, `success` | 调度决策本身（`Allocator.Schedule`，包括恢复、请求构建和装箱算法） | `pool_controller.go` `scheduleSandbox` |
| `controller.allocator.persist_alloc_state.duration` | `namespace`, `pool_name`, `success` | 持久化分配状态：针对 API server 的 `alloc-status` 注解 Patch | `allocator.go` `annoAllocationSyncer.SetAllocation` |
| `controller.allocator.sync_alloc_result.duration` | `namespace`, `pool_name`, `success` | 将批量分配结果同步到 Pool 的所有沙箱（完整的一轮并发同步） | `pool_controller.go` `doAllocate` |
| `controller.allocator.sync_single_alloc_result.duration` | `namespace`, `pool_name`, `success` | 按沙箱的分配结果同步（内存存储更新 + 注解 Patch）；`allocated` 和 `released` 同步都会记录 | `pool_controller.go` `syncSandboxConcurrently` |

属性语义：

- `namespace`：Pool 的 Kubernetes 命名空间（按沙箱仪器则为 BatchSandbox 的命名空间）。
- `pool_name`：Pool 名称（`pool.name` 语义；按沙箱仪器为 `BatchSandbox.spec.poolRef`）。
- `success`：操作无错误完成时为 `true`，否则为 `false`。

::: info 基数
属性集合有意限定为 (namespace, pool_name, success)；刻意没有 `sandbox_name` 标签。按沙箱的身份信息仍可通过 Kubernetes 对象和事件获得；追踪（traces）才是按沙箱延迟归因的预期载体。
:::

### 容量指标

周期性从 informer 缓存收集的 gauge 型信号，注册在同一个全局 OTel meter provider 上，由一个经 leader 选举的 manager Runnable 驱动，并在关闭时注销：

| 名称 | 类型 | 单位 | 属性 |
|---|---|---|---|
| `controller.pool.pods` | ObservableGauge | `{pod}` | `namespace`, `pool_name`, `state` (`total`/`allocated`/`available`/`updated`) |
| `controller.pool.cpu.requested` | ObservableGauge | `{cpu}` | `namespace`, `pool_name`, `state` |
| `controller.pool.memory.requested` | ObservableGauge | `By` | `namespace`, `pool_name`, `state` |
| `controller.batchsandbox.count` | ObservableGauge | `{batchsandbox}` | `namespace`, `phase`, `allocation_mode` (`pool`/`direct`) |
| `controller.batchsandbox.pods` | ObservableGauge | `{pod}` | `namespace`, `state` (`desired`/`current`/`allocated`/`ready`), `allocation_mode` |
| `controller.capacity.collect.duration` | ObservableGauge | `s` | — |

### Controller-runtime 指标（复用，不重复实现）

controller-runtime 指标端点（默认关闭；用 `--metrics-bind-address` 启用）已经暴露了 workqueue 深度/延迟、各驱动器的 reconcile 计数和时长（`controller_runtime_reconcile_*`）以及 REST client 指标。这些是 reconcile 健康度和 workqueue 信号的来源；控制器不会发出与之竞争的自定义版本。Pod 供给失败恢复还会在该端点上发布 `opensandbox_pod_recovery_*` 计数器 —— 见 [Pod 供给失败恢复](/guides/pod-recovery)。

### 为控制器启用 OTLP 导出

| 环境变量 | 默认值 | 说明 |
|---|---|---|
| `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` | （未设置） | 按信号的 OTLP/HTTP 端点，按原样使用（不追加路径）。 |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | （未设置） | 通用 OTLP/HTTP 端点，作为基础 URL 使用；当没有路径时会追加 `/v1/metrics`。在指标专用变量未设置时生效。 |
| `OTEL_EXPORTER_OTLP_METRICS_HEADERS` / `OTEL_EXPORTER_OTLP_HEADERS` | （未设置） | 附加到导出请求的 W3C Baggage 风格 `key1=value1,key2=value2` Header。 |
| `OTEL_METRIC_EXPORT_INTERVAL` | `60000`（毫秒） | OTLP 指标导出间隔。 |
| `OTEL_SERVICE_NAME` | `opensandbox-controller` | `service.name` 资源属性。 |
| `OTEL_RESOURCE_ATTRIBUTES` | （未设置） | 额外的资源属性（`key1=value1,key2=value2`）。 |
| `OTEL_SDK_DISABLED` / `OTEL_METRICS_EXPORTER` | （未设置） | 设置 `OTEL_SDK_DISABLED=true`，或将 `OTEL_METRICS_EXPORTER` 设为不含 `otlp` 的逗号分隔列表，即可禁用导出。 |

示例（Kubernetes 容器环境变量）：

```yaml
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: http://otel-collector.observability:4318
  - name: OTEL_METRIC_EXPORT_INTERVAL
    value: "30000"
```

行为：

- 未配置端点时，meter provider 保持 no-op，不导出任何内容。
- 容量 gauge 只在当选的 leader 上注册（经 leader 选举的 Runnable）；allocator 直方图在每个副本上都会记录并导出。
- 导出失败由 OTLP 导出器重试；设置/配置错误绝不阻塞控制器启动（遥测降级并记录错误日志）。
- 关闭时，provider 在 5 秒预算内刷出待处理数据。

### Prometheus 渲染

通过 OTLP → Prometheus 转换器导出时（例如 OpenTelemetry Collector 的 `prometheusremotewrite` 导出器或 Prometheus OTLP receiver），点号变为下划线并追加单位，例如：

```
controller_allocator_schedule_duration_seconds{namespace="default",pool_name="pool-a",success="true"}
```

## Execd

Meter：`opensandbox/execd` · Service 名称：`opensandbox-execd-<version>`（由 execd 固定；不读取 `OTEL_SERVICE_NAME`）。

| 指标 | 类型 | 单位 | 属性 | 含义 |
|---|---|---|---|---|
| `execd.http.request.duration` | Histogram | `ms` | `http_method`, `http_route`, `http_status_code` | HTTP 请求延迟。 |
| `execd.execution.duration` | Histogram | `ms` | `operation`, `result` | 代码/命令执行时长。 |
| `execd.filesystem.operations.duration` | Histogram | `ms` | `operation`, `result` | 文件系统操作时长。 |
| `execd.system.process.count` | Observable Gauge | - | - | 当前进程数。 |
| `execd.system.cpu.usage` | Observable Gauge | `%` | - | 沙箱 CPU 使用率百分比：配置了资源限制时取沙箱 cgroup，否则取主机 `/proc`（例如 microVM guest）。 |
| `execd.system.memory.usage_bytes` | Observable Gauge | `By` | - | 沙箱已用内存字节：配置了资源限制时取沙箱 cgroup，否则取主机 `/proc`（例如 microVM guest）。 |
| `execd.system.network.io.bytes` | Observable Counter | `By` | `direction` (`in`/`out`) | 累计网络 I/O 字节。 |
| `execd.system.network.connections.active` | Observable Gauge | - | `protocol` (`tcp`/`udp`) | 当前活跃网络连接数。 |
| `execd.system.oom.kills` | Counter | `{kill}` | `level` (`local`/`descendant`，仅 cgroup v2) | 在沙箱 cgroup 中检测到的 OOM kill（见下文）。 |
| `execd.isolation.run.duration` | Histogram | `ms` | `result` | 隔离会话运行时长。 |
| `execd.isolation.session.count` | Observable Gauge | - | - | 当前活跃隔离会话数。 |
| `execd.isolation.upper.usage_bytes` | Observable Gauge | `By` | - | 隔离会话 upper 目录使用的总字节数。 |

### OOM kill 观测

Execd 每 5 秒轮询自己的 cgroup，并对内核 OOM kill 计数器做差分：cgroup v2 上是 `memory.events`（`oom_kill`），cgroup v1 上是 `memory.oom_control`。覆盖范围是整个沙箱 cgroup 子树 —— 用户和业务进程以及嵌套 cgroup（例如隔离会话），而不仅仅是沙箱主进程。cgroup v2 主机被完整覆盖；cgroup v1 报告单次增量且没有 `level`。有意不使用内核日志（dmesg）：沙箱容器没有 `CAP_SYSLOG`，且内核环形缓冲区不是沙箱作用域的。

每次检测产生两个信号：

- 指标 `execd.system.oom.kills`，带 `level` 属性：`local` 表示沙箱 cgroup 自身进程被 kill，`descendant` 表示嵌套 cgroup 中的 kill。
- 容器 stdout（或 `EXECD_LOG_FILE` 配置的文件）上的一行结构化 warn 日志，附带故障排查上下文：
  - `memory=<used>/<limit> peak=<peak>` —— 检测触发时内存相对于 cgroup 限制的位置
  - `events=[oom=N oom_kill=N oom_group_kill=N]` —— `memory.events` 子计数器的增量（仅 cgroup v2）
  - `kill_cgroups=a,b` —— 子树发生 kill 的一级子 cgroup
  - `exited=[pid=N comm=x ppid=N rss=N age=Ns, ...]` —— 在事件前后两次轮询之间从 `/proc` 消失的进程（受害者候选；数量以 kill 数为界，最多 5 条）

对于经 `POST /internal/init` 重新绑定的预热沙箱，跨越绑定交接的计数器区间会被丢弃，而不是归属到新的沙箱身份。

### 共享属性

- 来自 `OPENSANDBOX_ID` 的 `sandbox_id`（设置时）
- 来自 `OPENSANDBOX_EXECD_METRICS_EXTRA_ATTRS` 的额外键值属性（设置时）

Execd 还通过其 `/metrics` 和 `/metrics/watch` API 提供本地 CPU/内存快照和实时流，独立于 OTLP 导出 —— 见 [execd 架构](/architecture/data-plane/execd#observability)。

### 端点配置

只有设置了至少一个 OTLP 端点（`OTEL_EXPORTER_OTLP_METRICS_ENDPOINT`，回退 `OTEL_EXPORTER_OTLP_ENDPOINT`）才启用导出；否则 execd 将指标保留在本地。两者都未设置时，适用[上文](#export-endpoints)描述的节点 IP 回退。

```bash
export OTEL_EXPORTER_OTLP_METRICS_ENDPOINT="http://otel-collector:4318"
```

## Egress sidecar

Meter：`opensandbox/egress` · Service 名称：`opensandbox-egress-<version>`。

| 指标 | 类型 | 单位 | 含义 |
|---|---|---|---|
| `egress.dns.query.duration` | Histogram | `s` | 上游 DNS 转发延迟（为放行的查询记录）。 |
| `egress.dns.query.failed_total` | Counter | - | 代理无法解析的查询，按 `reason`。 |
| `egress.dns.reply.failed_total` | Counter | - | 决策之后写回复失败的次数，按 `stage`。非零计数意味着查询已被处理但其应答从未到达客户端。 |
| `egress.policy.denied_total` | Counter | - | 被策略拒绝的 DNS 查询数。 |
| `egress.nftables.rules.count` | Observable Gauge | `{element}` | 最近一次成功静态应用后的策略规模近似值（fast-sandbox profile：对每个已安装 subject 的策略求和，deny-first 阶段为 0）。 |
| `egress.nftables.updates.count` | Counter | - | 成功的 nftables 更新次数（静态应用 + 动态 IP 添加）。 |
| `egress.nftables.updates.failed_total` | Counter | - | 失败的 nftables 更新，按 `operation`。 |
| `egress.system.memory.usage_bytes` | Observable Gauge | `By` | **节点**已用内存字节（Linux：gopsutil；非 Linux 构建：`0`）。 |
| `egress.system.cpu.utilization` | Observable Gauge | `1` | **节点** CPU 繁忙比例，范围 `[0,1]`（Linux：gopsutil；非 Linux 构建：`0`）。 |
| `egress.process.memory.usage_bytes` | Observable Gauge | `By` | 归属到 sidecar 自身 cgroup 的内存。仅在 cgroupfs 可读时存在。 |
| `egress.process.cpu.time` | Observable Counter | `s` | sidecar 自身 cgroup 消耗的 CPU 秒数。仅在 cgroupfs 可读时存在。 |

### `system` 与 `process` 的区别

它们测量的是不同的东西，这个区别很重要，因为该 sidecar 是**每个沙箱一个**的：

- `egress.system.*` 来自 gopsutil，即 `/proc/meminfo` 和 `/proc/stat`，在容器内它们描述的是**节点**。因此节点上每个沙箱都会在自己 `sandbox_id` 下发布同一个数值。不要"按沙箱"绘制这些图：序列看起来像按沙箱，实际上是一个节点数值的 N 份拷贝。节点级数据请优先使用 kubelet/cAdvisor 或 node exporter。
- `egress.process.*` 读取自 sidecar 自身的 cgroup（v2 的 `memory.current` 和 `cpu.stat`，回退到 v1 的 `memory.usage_in_bytes` 和 `cpuacct.usage`），因此确实是按沙箱的。

`egress.process.cpu.time` 是**消耗秒数的累计计数器**，不是采样比例：请对其使用 `rate()`。两个 `process` 仪器只有在其 cgroup 文件可读时才注册 —— 不暴露 cgroupfs 的运行时（例如 `secure_runtime` 下的沙箱 Pod）完全不会产生序列，而不是产生一个看起来像 sidecar 空闲的平坦零值。

### DNS 延迟直方图边界

`egress.dns.query.duration` 显式声明了桶边界：

```
0.001  0.0025  0.005  0.01  0.025  0.05  0.1  0.25  0.5  1  2.5  5  10  15  30  60  120  300  600
```

该仪器记录的是**秒**，而 SDK 默认边界是规范中的毫秒阶梯（`0, 5, 10, … 10000`），因此所有现实的延迟都会落入唯一的 `le=5` 桶中，分位数将毫无意义。存在粗粒度的尾部是因为记录的时长覆盖**整个解析器链**：转发会串行遍历各上游，每个上游使用完整的超时（`OPENSANDBOX_EGRESS_DNS_UPSTREAM_TIMEOUT`，默认 2s），因此一个查询合理地可能花费 `timeout x len(upstreams)`。迟到的**成功**同样落入尾部，不仅仅是耗尽后的失败。该链没有有限的最坏情况（`OPENSANDBOX_EGRESS_DNS_UPSTREAM` 接受无界解析器列表），因此超过最后一个边界后，按构造就失去了分位数解析能力，只剩下 `_count`。成功和失败的查询都会进入该直方图，因此其尾部混合了慢解析和耗尽的重试链。

### 失败信号

`egress.dns.query.failed_total` 和 `egress.policy.denied_total` 回答的是不同的问题，混淆它们会使诊断颠倒：

- **denied** —— 策略正常履行了职责。工作负载请求了它不被允许访问的内容。这是正常系统中的预期流量。
- **failed** —— sidecar 未能履行职责。工作负载请求了被允许的内容却得到了 `SERVFAIL`。这永远不应发生。

`reason` 来自一个封闭集合，因此计数器基数固定，且从不附加被查询的名称或错误文本：

| `reason` | 含义 |
|---|---|
| `no_upstreams` | 未配置或未发现任何解析器。 |
| `upstream_error` | 所有解析器都未能应答（网络错误、超时）。 |
| `empty_response` | 某个解析器返回了 nil 消息。 |
| `rcode` | 最后一个解析器应答了值得故障转移的 rcode，例如 `SERVFAIL`。 |

`egress.nftables.updates.failed_total` 覆盖另一类静默失败。其 `operation` 属性是 `static_apply`、`dynamic_add`、`remove`，或者 —— 在 fast-sandbox profile（OSEP-0022）中 —— `deny_first`、`reset`。`dynamic_add` 是需要告警的那一个：添加失败意味着内核从未得知策略允许的 IP，因此链会丢弃本应通过的流量 —— 这在沙箱内部看起来与策略拒绝完全一样，而 `egress.policy.denied_total` 却保持平坦。

`egress.dns.reply.failed_total` 覆盖最后一类静默失败：一个**已被处理**的查询（已决策、可能已转发并从上游得到应答），但随后写回复失败。其 `stage` 属性是 `malformed`、`unknown_source`、`deny`、`upstream_error`、`answer` 之一，且每次失败还会发出一条 `[dns] reply write failed (stage=… remote=… question=…)` 警告，包含远端地址和查询名称。对任何非零值告警：与 `dynamic_add` 一样，`answer` 阶段的失败意味着策略允许的流量没有到达客户端。

`static_apply` 失败发生在启动期间，此时 sidecar 记录日志并退出。指标通过周期性 reader 离开，而 `os.Exit` 会跳过延迟关闭，因此该路径会在终止前显式刷出遥测 —— 否则解释 sidecar 为何死掉的那一条样本永远不会被导出。

### 自动 egress 放行规则

当配置了 OTLP 目标 —— 端点环境变量，或两者都未设置时的导出器回退节点 IP（`HOST_IP` / `/etc/hostinfo`）—— egress 会自动为该主机（域名或 IP，任意端口）注入一条始终放行的 egress 规则，使遥测导出在默认 deny-all 策略下也能工作，无需手工维护放行列表规则。这也覆盖 sidecar 自身的指标导出 —— 它与沙箱共享网络命名空间，否则会被自己的 egress 链阻断。

- 优先级：`OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` 优先于 `OTEL_EXPORTER_OTLP_ENDPOINT`；回退节点 IP 仅在两者都未设置时适用。已设置但非法的端点从不回退（导出器也是如此），因此这种情况下不会注入规则。
- 端点必须是 URL（`https://host:4318/v1/metrics`）—— 即 `otlpmetrichttp` 的环境变量形式。不接受裸 `host:port` 或 `host` 值；FQDN 末尾的根点会被修剪，以与 DNS 策略归一化保持一致。
- 该规则位于 always-allow 层：它能经受住用户对策略的 `POST`/`PATCH`/`DELETE` 更新和 always 规则文件的重新加载。运维仍然可以用 `deny.always` 阻断该目标，后者优先。
- 规则按主机作用域（任意端口），与 egress 规则模型一致；端口不按规则强制。

::: warning
端点请使用完全限定的服务名称或 IP。单标签名称（例如 `otel-collector`）会受到解析器搜索域扩展的影响，而 deny-all DNS 代理会用 NXDOMAIN 应答扩展后的名称（例如 `otel-collector.<ns>.svc.cluster.local`），且不会回退到裸名称，因此自动生成的按主机精确放行规则将无法命中。
:::

### 共享属性

- 来自 `OPENSANDBOX_EGRESS_SANDBOX_ID` 的 `sandbox_id`（设置时）。没有它，不同沙箱的 sidecar 会导出相同的属性集合，其序列在后端发生碰撞 —— 这对按沙箱的 `egress.process.*` gauge 影响最大。
- 来自 `OPENSANDBOX_EGRESS_METRICS_EXTRA_ATTRS` 的额外键值属性（设置时）

### 端点配置

只有设置了至少一个 OTLP 端点才启用导出；否则 egress 将指标保留在本地（不进行 OTLP 导出），并受[上文](#export-endpoints)描述的节点 IP 回退约束。

```bash
export OTEL_EXPORTER_OTLP_METRICS_ENDPOINT="http://otel-collector.sandbox.svc.cluster.local:4318"
# An IP endpoint works as well:
export OTEL_EXPORTER_OTLP_METRICS_ENDPOINT="http://10.0.0.5:4318"
```

server 可以通过 `[egress].otlp_endpoint` 向每个 egress sidecar 注入端点；见 [egress 可观测性](/architecture/network/egress#observability-opentelemetry)。

### 结构化日志（JSON）

Egress 结构化日志由 zap 发出（通常到 stdout）。树内未实现 OTLP 日志导出。

- 公共字段：`sandbox_id`（设置 `OPENSANDBOX_EGRESS_SANDBOX_ID` 时）、从 `OPENSANDBOX_EGRESS_METRICS_EXTRA_ATTRS` 合并进根 logger 的键值对，以及标识事件族的 `opensandbox.event`。
- `opensandbox.event=egress.outbound` —— 在放行路径的 DNS 处理时发出（成功或转发错误），带 `target.host`、`target.ips`、`peer` 和 `error`。
- 策略生命周期：`opensandbox.event=egress.loaded`（初始生效策略已加载）、`egress.updated`（策略更新已应用）、`egress.update_failed`（策略更新失败），带 `egress.default`（`allow`/`deny`）、`rules` 和 `error` 字段。

## Ingress 网关

Meter：`opensandbox/ingress` · Service 名称：`opensandbox-ingress-<version>`。

### HTTP 请求与路由

| 指标 | 类型 | 单位 | 属性 | 说明 |
|---|---|---|---|---|
| `ingress.http.request.count` | Counter | — | `http_method`, `http_status_code`, `proxy_type` | 请求计数（可推导 QPS） |
| `ingress.http.request.duration` | Histogram | `ms` | `http_method`, `http_status_code`, `proxy_type` | 完整请求时长 |
| `ingress.routing.resolutions.count` | Counter | — | `routing_result` | 路由解析计数 |
| `ingress.routing.resolution.duration` | Histogram | `ms` | `routing_result` | 路由解析延迟 |

### 上游连通性

| 指标 | 类型 | 单位 | 属性 | 说明 |
|---|---|---|---|---|
| `ingress.upstream.connect.count` | Counter | — | `connect_result`, `proxy_type` | 按结果统计的上游 TCP 拨号尝试 |
| `ingress.upstream.connect.duration` | Histogram | `ms` | `connect_result`, `proxy_type` | 上游 TCP 拨号时长 |

### 网络就绪（影子评估）

报告最近一个完整窗口的数据；由 `--network-readiness-shadow-*` 参数调节。影子 gauge 从不拦截流量。

| 指标 | 类型 | 单位 | 说明 |
|---|---|---|---|
| `ingress.network.shadow.attempts` | Observable Gauge | — | 窗口内的 TCP 连接尝试次数 |
| `ingress.network.shadow.signal_failures` | Observable Gauge | — | 窗口内的超时或不可达结果 |
| `ingress.network.shadow.distinct_targets` | Observable Gauge | — | 窗口内有界的去重上游目标数 |
| `ingress.network.shadow.distinct_signal_targets` | Observable Gauge | — | 出现超时/不可达结果的去重目标数 |
| `ingress.network.shadow.qualified` | Observable Gauge | — | 窗口样本充足时为 1 |
| `ingress.network.shadow.degraded` | Observable Gauge | — | 合格窗口处于降级状态时为 1 |

### 按访问唤醒（OSEP-0024）

在启用按访问唤醒时发出。恢复饱和、停车场丢弃（parking-lot shedding）、策略门失败和 Redis 写入丢弃是需要关注的信号。

| 指标 | 类型 | 单位 | 属性 | 说明 |
|---|---|---|---|---|
| `ingress.wake.flights` | Counter | — | `outcome` (`triggered`/`joined`/`none`) | 按结果统计的唤醒 flight 数 |
| `ingress.park.active` | UpDownCounter | — | - | 当前停靠中的请求 |
| `ingress.park.wait.duration` | Histogram | `s` | `outcome` (`served`/`budget_exhausted`/`canceled`/`error`) | 每个停靠请求的等待时长；桶大小围绕默认 5s 停靠预算设计（同节点 ~200ms 恢复对比接近预算耗尽的等待） |
| `ingress.park.shed` | Counter | — | - | 因停车场已满而被丢弃的请求数 |
| `ingress.wake.restore.active` | UpDownCounter | — | - | 当前持有恢复许可的唤醒 flight 数（并发恢复 gauge） |
| `ingress.idlep.policy.reads` | Counter | — | `outcome` (`allowed`/`denied`/`error`) | 空闲策略门的结果；持续的 `error` 意味着 RBAC 或 apiserver 问题 |
| `ingress.activity.writes_dropped` | Counter | — | - | 在到达 Redis 之前被丢弃的活动观测 |

### 主机指标（仅 Linux；其他平台为 0）

| 指标 | 类型 | 单位 | 说明 |
|---|---|---|---|
| `ingress.system.cpu.usage` | Observable Gauge | `1` | CPU 繁忙比例 `[0,1]` |
| `ingress.system.memory.usage_bytes` | Observable Gauge | `By` | 已用内存字节 |
| `ingress.connections.active` | Observable Gauge | — | TCP ESTABLISHED 连接数（来自 `/proc/net/tcp`） |

**属性取值：**

- `proxy_type`: `http` | `websocket`
- `routing_result`: `success` | `not_found` | `not_ready` | `error`
- `connect_result`: `success` | `timeout` | `unreachable` | `refused` | `dns_error` | `canceled` | `other`

### 端点配置

```bash
export OTEL_EXPORTER_OTLP_METRICS_ENDPOINT="http://otel-collector:4318"
```

回退：`OTEL_EXPORTER_OTLP_ENDPOINT`。两者都未设置时，导出回退到节点 IP（先 `HOST_IP` 环境变量，再 `/etc/hostinfo`），端口 4318，纯 HTTP；解析不到节点 IP 则不导出指标。用 `OTEL_SDK_DISABLED=true` 或 `OTEL_METRICS_EXPORTER=none` 禁用导出。默认时间性：同步 Counter/Histogram 用 delta，observable gauge 用 cumulative —— 可用 `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` 覆盖。

## Node agent

node agent 的日志管道导出 OTLP 指标，不以任何沙箱或 Pod 标识符作为标签：

| 指标 | 类型 | 单位 | 说明 |
|---|---|---|---|
| `opensandbox.nodeagent.records` | Counter | - | 从源消费并交给 sink 的记录数。 |
| `opensandbox.nodeagent.bytes` | Counter | `By` | 消费的记录负载字节数。 |
| `opensandbox.nodeagent.drops` | Counter | - | 被丢弃的记录（仅在 `drop` 背压下可能发生）。 |
| `opensandbox.nodeagent.retries` | Counter | - | sink 投递重试次数。 |
| `opensandbox.nodeagent.queue.bytes` | UpDownCounter | `By` | 当前排队等待投递的字节数（队列深度代理）。 |
| `opensandbox.nodeagent.sink.consume.duration` | Histogram | `ms` | sink 消费（写入）延迟。 |

它要求显式端点 —— 其他组件使用的节点 IP 回退已被禁用（`DisableEndpointFallback`）：

```yaml
opensandbox-node-agent:
  extraEnv:
    - name: OTEL_EXPORTER_OTLP_METRICS_ENDPOINT
      value: http://otel-collector.observability:4318/v1/metrics
```

完整的 agent 配置请参见 [node agent 配置](/guides/node-agent)。

## SDK 指标

SDK 向 lifecycle server 上报沙箱创建延迟（`POST /v1/metrics/events`），server 在启用 `[otel]` 时将其导出为 `opensandbox.sandbox.create.duration`；SDK 还可以通过应用自身的 OpenTelemetry provider 发出 Pool 预热追踪。客户端侧的启用与禁用见 [SDK 可观测性](/sdks/observability)。
