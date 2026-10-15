---
title: Node Agent 配置
description: 部署并配置 OpenSandbox Node Agent —— 启用 DaemonSet、选择文件或 OSS sink、按需开启 syscall 采集，以及安全地运维它。
---

# Node Agent 配置

[Node Agent](/architecture/data-plane/node-agent) 是一个可选的 DaemonSet，在每个 Linux 节点上采集沙箱的 stdout/stderr（以及可选的 syscall），并将其写入持久化存储。本指南介绍如何启用它、选择 sink，以及保护其投递保证的运维规则。完整的 values 参考请查看 [node-agent chart README](https://github.com/opensandbox-group/OpenSandbox/tree/main/manifests/charts/node-agent)。

## 前提条件

- 一个通过 [OpenSandbox 部署](/deployment/)安装的 Kubernetes 集群 —— Agent 与部署顺序无关，你可以在任意时间启用它。
- 每个 Linux 节点自动运行一个 Agent：chart 固定了 `kubernetes.io/os: linux`。
- 每个节点需要本地磁盘存放 checkpoint 状态（`hostPaths.state`，默认 `/var/lib/opensandbox/nodeagent`），它必须能跨 Pod 重启保留 —— 它是一个 `hostPath` 卷。

Agent 通过身份标识（`opensandbox.io/id` 标签、`sandbox` 容器）发现沙箱 Pod，并跳过 Pool 预热的 Pod。无需任何按 Pod 的配置。

## 启用 Agent

**Umbrella chart** —— 设置子 chart 开关：

```bash
helm upgrade opensandbox opensandbox \
  --namespace opensandbox-system \
  --set opensandbox-node-agent.enabled=true
```

**独立 chart** —— 直接安装：

```bash
helm install node-agent manifests/charts/node-agent \
  --namespace opensandbox-system \
  --create-namespace
```

验证 DaemonSet 已完成发布且 Agent 报告就绪。就绪状态是有意义的：只有当配置、恢复状态、sink 目标和主机预留全部有效时它才会变绿。

```bash
kubectl rollout status daemonset -l app.kubernetes.io/component=node-agent -n opensandbox-system

kubectl port-forward -n opensandbox-system daemonset/node-agent 8080:8080
curl http://127.0.0.1:8080/readyz   # {"ready":true,"reasons":[]}
```

::: tip 务必设置真实的集群 ID
`config.clusterID`（默认 `dev-cluster`）在每条记录和每个对象路径中标识你的集群，并且是 Agent 目标身份的一部分。上线前设置一次 —— 之后再修改它属于目标变更，需要执行排水操作（参见[更换 sink 或集群 ID](#changing-sink-or-cluster-id)）。
:::

## 选择 sink

每个 Agent 只能有一个 sink 目标。记录落在
`<keyPrefix>/<cluster>/<namespace>/<sandbox_id>/<pod_uid>/` 下，以编号的代（generation）形式存放，每个流附带一个 `sandbox.finalized.<revision>.json` 完整性标记。

以下示例使用 umbrella-chart 的 key 前缀（`opensandbox-node-agent.`）；如果你安装的是独立 chart，请去掉该前缀。

### 本地文件（默认）

```yaml
# values-node-agent.yaml
opensandbox-node-agent:
  enabled: true
  config:
    clusterID: prod-cluster
  sink:
    type: file
    file:
      path: /var/lib/opensandbox/nodeagent-data
      maxBytes: 1073741824      # rotate a generation at 1 GiB
      maxFiles: 16              # per stream
      maxTotalBytes: 10737418240
      retention: 24h            # cleanup after the marker's repair window
```

请保持 `sink.file.path` 与 `hostPaths.fileData` 同步 —— chart 用它们分别作为挂载点和写入目标。基于保留期的清理在 Agent 内部运行，且只会删除完整的、已终结（finalized）的记录族。

文件 sink 适用于单节点调试和小型集群。任何需要跨节点丢失场景持久化的用途，请使用 OSS。

### 阿里云 OSS

先创建凭据。采集凭据必须能够追加、读取和写入 —— **绝不能删除**。删除通过单独的离线工具（见下文）完成，并使用其自身的凭据。

```bash
kubectl create secret generic nodeagent-oss-credentials \
  --namespace opensandbox-system \
  --from-literal=access-key-id=... \
  --from-literal=access-key-secret=...        # optional: --from-literal=session-token=...
```

```yaml
opensandbox-node-agent:
  enabled: true
  config:
    clusterID: prod-cluster
  sink:
    type: oss
    oss:
      endpoint: https://oss-cn-zhangjiakou.aliyuncs.com   # HTTPS origin required
      bucket: my-sandbox-logs
      keyPrefix: logs
      existingSecret: nodeagent-oss-credentials
```

Bucket 必须满足 Agent 的完整性假设 —— 否则启动将永久失败：

- 禁用版本控制，无 WORM 保留策略
- 没有前缀与 `<keyPrefix>/<clusterID>/` 重叠的生命周期规则（生命周期删除会破坏标记的保证）

对象以可追加写入方式写入，并封存为不可变的代；标记写入后禁止覆盖，因此消费端可以通过大小和校验和进行验证。由于采集凭据不能删除，清理是一项显式的、感知标记的离线操作 —— 在具备删除权限凭据的运维机器上运行 [`nodeagent-oss-cleanup`](https://github.com/opensandbox-group/OpenSandbox/tree/main/components/nodeagent/cmd/oss-cleanup)。它先规划再删除，并要求显式的目标排水确认。

## 按需开启 syscall 采集

默认源集合只有 `container-logs`。加入 `syscalls` 会启用一个内嵌的 eBPF 程序，记录每个沙箱发出的每一次 syscall 事件：

```yaml
opensandbox-node-agent:
  config:
    sources:
      - container-logs
      - syscalls
```

当 `syscalls` 存在时，chart 会自动处理 Kubernetes 侧的管道配置：以只读方式挂载主机 cgroup v2 层级和 tracefs，并添加 `BPF` 和 `PERFMON` 能力。节点本身必须提供：

- Linux 内核 5.11 或更新
- cgroup v2 统一层级
- tracefs 挂载在 `/sys/kernel/tracing`

syscall 记录落在自己的 `<cluster>/_streams/syscall/…` 树中，并携带自己的标记。某些情况下 syscall 标记按设计会报告为 `incomplete` —— 例如，tracer 只有在 Kubernetes 报告容器 ID 之后才会挂载，因此挂载前的区间会被报告为缺口，而不是声称已覆盖。

## 关键 values

| Value | 默认值 | 控制内容 |
|---|---|---|
| `config.sources` | `["container-logs"]` | 启用的源；镜像必须包含每个命名的源 |
| `config.clusterID` | `dev-cluster` | 记录、路径和目标身份中的集群名称 |
| `config.memoryBudgetBytes` | 256 MiB | 缓冲记录的全局队列预算 |
| `config.perSandboxQueueBytes` | 16 MiB | 触发背压前的每沙箱队列预算 |
| `config.perSandboxRateLimit` | `0`（关闭） | 每沙箱记录速率限制 |
| `config.dropPolicy` | `block` | `block` 应用背压；`drop` 丢弃记录并在标记中记账 |
| `config.sinkTimeout` / `config.retryMaxInterval` | 30s / 30s | sink 写入超时和重试退避上限 |
| `config.stateDir` + `hostPaths.state` | `/var/lib/opensandbox/nodeagent` | Checkpoint 状态位置（保持两者同步） |
| `config.maxLineBytes` | 1 MiB | 最长日志行；超长行被丢弃并记账 |
| `config.endedStateRetention` | 24h | 已结束的流在状态被清理前仍可接收迟到数据的时长 |
| `resources` | 50m/128Mi – 1/512Mi | DaemonSet 资源（实验性默认值 —— 生产容量规划前请先做基准测试） |

默认值遵循安全姿态：`block` 背压意味着 sink 变慢会拖慢采集而不是丢失数据。只有当你理解被丢弃的记录不可恢复时才切换到 `drop` —— 它们会在流标记中报告，但无法找回。

## 指标

Agent 导出 OTLP 指标（`opensandbox.nodeagent.*`：records、bytes、drops、retries、queue depth、sink latency），不以任何沙箱或 Pod 标识符作为标签。它要求显式配置端点 —— 其他组件使用的节点 IP 回退已被禁用：

```yaml
opensandbox-node-agent:
  extraEnv:
    - name: OTEL_EXPORTER_OTLP_METRICS_ENDPOINT
      value: http://otel-collector.observability:4318/v1/metrics
```

指标清单请参见 [telemetry](/guides/telemetry#node-agent)，共享导出器行为请参见[共享 OTLP 配置](/guides/telemetry#shared-otlp-configuration-for-go-components)。

## 运维规则

以下规则用于保护投递保证 —— 违反它们会使流变为 `incomplete`，或使 Agent 保持在未就绪状态（这正是预期的、可见的结果）：

- **绝不删除或轮转状态目录。** 它保存的是 checkpoint 和终结意图，不是数据负载。在流进行中删除它将丧失恢复能力。
- **不要在 Agent 之外轮转或删除文件 sink 的输出。** Agent 拥有其布局和清理逻辑。
- **就绪状态是诊断手段。** `kubectl get daemonset` + `curl /readyz` 会给出具体原因（`invalid-config`、`state-unavailable`、`operation-retrying`、`store-stale` 等）。未就绪的 Agent 绝不会静默丢弃数据。
- **重启的结果是诚实，而不是无损。** Agent 重启后，受影响的标记会报告 `incomplete` —— 保证是在可证明的覆盖窗口内至少投递一次，标记不会声称更多。

### 更换 sink 或集群 ID

目标身份是 sink 配置与集群 ID 的摘要。如果你修改了 `sink.*` 或 `config.clusterID`，Agent 会拒绝复用旧状态 —— 这是设计使然，以免把一个集群的记录追加到另一个目标上。安全地更换目标：

1. 停止采集（将 DaemonSet 缩容到零）并让各流完成终结。
2. 排空旧目标（对于 OSS，完成离线清理/排水确认）。
3. 更新 values 并重新部署。
