---
title: Kubernetes 控制器
description: 在 Kubernetes 上运行沙箱的 operator——它的三个自定义资源、恒定时间交付背后的 reconciler 设计，以及任务编排。
---

# Kubernetes 控制器

控制器是支撑容器沙箱的 Kubernetes operator。它监视三个自定义资源，并将其转化为运行中的沙箱，具备 Pod 级调度、用于恒定时间交付的预热池、可选的任务编排，以及基于快照的暂停/恢复。生命周期服务端代表 API 客户端创建和管理这些资源，但这些资源就是普通的 Kubernetes 对象——可见、可审计、可直接使用。

![控制器资源模型](../../public/images/operator-model.svg)

## 总体设计

控制器管理器运行三个 reconciler，每个自定义资源对应一个。它们都遵循标准的 Kubernetes 模式：监视期望状态，将其与观测到的 Pod 做对比，并以幂等操作完成收敛。

![控制器 reconciler](../../public/images/operator-reconcilers.svg)

- **BatchSandboxReconciler** — 负责沙箱 Pod：在直接（非池化）模式下扩缩容，解析池分配，调度任务，强制执行 `expireTime`，并驱动暂停/恢复的交接（`internal/controller/batchsandbox_controller.go`）。
- **PoolReconciler** — 负责池 Pod 并监视 `BatchSandbox` 对象：将预热 Pod 调度给等待中的沙箱，扩缩容缓冲区，滚动更新模板，并处理驱逐（`internal/controller/pool_controller.go`）。
- **SandboxSnapshot reconciler** — 将 `SandboxSnapshot` 转化为源节点上的提交 `Job`，并记录推送镜像的结果（`internal/controller/sandboxsnapshot_controller.go`）。

两个契约将各 reconciler 连接起来而不产生耦合：

- **分配注解。** 池 reconciler 在 `BatchSandbox` 上写入 `sandbox.opensandbox.io/alloc-status`（已分配 Pod、`poolRef`、generation）和 `alloc-release`；`endpoints` 注解携带已解析的端点，供服务端消费。Pod 携带 `sandbox.opensandbox.io/pool-name` 和 `pool-revision` 标签。
- **任务分发。** batch reconciler 运行一个进程内 `TaskScheduler`，将任务指派给 Pod，并调用每个沙箱 Pod 内运行的任务执行器 HTTP 服务。

## BatchSandbox

`BatchSandbox` 是主力资源：以声明式方式描述一个或多个相同的沙箱。

| Spec 字段 | 含义 |
|---|---|
| `replicas` | 沙箱数量（默认 1） |
| `template` | 每个副本启动所依据的 Pod 模板 |
| `poolRef` | 从指定名称的 `Pool` 分配，而不是调度新 Pod；与 `template` 互斥 |
| `shardPatches` | 应用在 `template` 之上的按副本补丁——单个副本可以在镜像、环境变量或资源上不同 |
| `taskTemplate` | 可选，每个副本在分配后运行的进程，由 Pod 内的任务执行器执行 |
| `shardTaskPatches` | `taskTemplate` 的按副本变体——同一批次内异构的任务 |
| `taskResourcePolicyWhenCompleted` | 任务达到 Succeeded/Failed 后沙箱资源的处理方式：`Retain`（默认）保留资源直到删除，`Release` 立即释放 |
| `expireTime` | 绝对删除期限，由控制器强制执行 |
| `pause` | 暂停/恢复意图：`true` 暂停，`false` 恢复。控制器从不清除该字段；它通过 `status.pauseObservedGeneration` 确认进度，该字段同时也控制重新进入 |

如果 `poolRef` 为空，控制器仍可使用可配置的 profile 自动选择池：谓词插件（容量、镜像、资源、节点选择器）过滤候选者，然后由打分插件（默认为最少分配）选出一个（`internal/controller/poolassign/`）。

Status 将沙箱健康与任务进度区分开：`replicas` / `allocated` / `ready` 和 `phase` / `conditions` 描述运行时；`taskRunning` / `taskSucceed` / `taskFailed` / `taskPending` / `taskUnknown` 计数器加上 `taskLastErrorMessage` 描述任务。

## Pool

`Pool` 持有预热的 Pod，使分配成为对既有容量的索取，而不是一轮调度往返。

| Spec 字段 | 含义 |
|---|---|
| `template` | 预热 Pod 启动所依据的 Pod 模板 |
| `capacitySpec` | `bufferMin`/`bufferMax` 控制 ready 区间；`poolMin`/`poolMax` 限定整个池 |
| `scaleStrategy` | `maxUnavailable` 限制池的扩缩容速度（绝对值或百分比，默认 25%） |
| `updateStrategy` | 针对既有 Pod 滚动发布模板变更的同一上限 |
| `recycleStrategy` | 已分配 Pod 释放归还后的处理方式：`Delete`（默认）、`Restart` 或 `Noop` |

Status 报告 `total` / `allocated` / `available` / `updated`，以及标识当前模板版本的 `revision`——模板的 SHA-256 哈希（前 8 字节）。

### 池设计

池维护一个**两级容量模型**：

![池容量模型](../../public/images/operator-pool-capacity.svg)

- **Buffer** — 预热、未分配、处于 Running 且 Ready 的 Pod。这是分配的来源，维持在 `bufferMin` 和 `bufferMax` 之间。控制器持续向该区间收敛：一次耗尽 buffer 的分配会触发扩容；释放出的容量超过 `bufferMax` 会触发缩容。
- **总池** — 池拥有的所有 Pod，无论是否已分配，受 `poolMin` 和 `poolMax` 限制。仍在启动中的 Pod 计入总量，但绝不会被打包进可用的 buffer，因此池永远不会承诺它无法交付的容量。

扩缩容是有意限速的：每轮 reconcile 至多创建或删除 `maxUnavailable` 个 Pod（减去已处于 not-ready 的），因此一波分配洪峰只会带来平稳的新 Pod 爬坡，而不是惊群。

**分配与归还。** 分配是对预热 Pod 的恒定时间索取：池 reconciler 将 Pod 调度给等待中的沙箱，然后通过分配注解持久化这一指派。当使用某 Pod 的沙箱消失后，该 Pod 归还池中，`recycleStrategy` 决定其命运：

- `Delete`（默认）— 归还的 Pod 会被一个全新 Pod 替换，因此每次分配都从模板的干净状态开始。
- `Restart` — 容器原地重启：比整体更换 Pod 更便宜，仍能清除进程状态。
- `Noop` — 原样归还：最快，但文件系统和进程状态会泄漏到下一次分配。

**模板更新。** 编辑池模板会改变 revision；控制器删除仍停留在旧 revision 的空闲 Pod（绝不会删除已分配给沙箱的 Pod），并在 `updateStrategy` 上限下按新 revision 补充，`status.updated` 跟踪进度。

**驱逐。** 给空闲池 Pod 打上 `pool.opensandbox.io/evict` 标签即请求将其移除。控制器保护已分配给 `BatchSandbox` 的 Pod，删除空闲 Pod，并让补充机制恢复 buffer。

**耗尽。** 当没有任何符合条件且有空位的池时，控制器会在等待中的 `BatchSandbox` 上设置 `PoolAllocationPending` condition（原因为 `PoolCapacityExhausted`），并每 5 秒复查一次；服务端会等待一段有界的时间窗口，若容量始终不释放则返回可重试的 `429`。在该窗口内释放的空位仍可满足请求。

### 生命周期 API 对池的约束

由于池 Pod 在任何请求之前就已存在，生命周期 API 无法向其注入按请求的状态：

- `entrypoint` / `env` 作为分配后的任务交付，而不是改写 Pod——在 Pod spec 中看到原始模板是预期行为。
- `volumes` 和 `networkPolicy` 不能与池引用组合使用；请在池模板中预先配置。
- 池 Pod 必须满足任务契约（Pod 内任务执行器、bootstrap entrypoint、已安装 execd）才能用于生命周期 API。

## 任务编排

任务是可选的：没有 `taskTemplate` 的 `BatchSandbox` 就是纯粹的分配。有了它，每个副本运行一个定义好的进程——整个批次内一致，或通过 shard patch 按沙箱定制。

进程任务可以在 **Local**（任务执行器容器内部，默认）或 **Remote**（通过 `nsenter` 在主容器内部）运行，通过 `execMode` 按任务选择。进程任务还支持 `preStart` 和 `postStop` 生命周期钩子：失败或超时的 `preStart` 会阻止主进程启动，而 `postStop` 在每种终态下都会运行，包括删除后的清理——适合准备输入和将输出持久化到挂载的卷。

## SandboxSnapshot

`SandboxSnapshot` 是暂停/恢复背后的检查点记录。它的 spec 只有一个字段——目标 `BatchSandbox` 名称（同一命名空间）；控制器负责解析 Pod 和节点。镜像仓库、推送凭据和快照类型来自控制器启动参数，而不是 spec。Status 携带结果：`Pending → Committing → Succeed / Failed` 的 phase，`rootfs-v1`（默认）或 `qemu-v1` 的 `format`，每个容器提交的镜像 URI 及其不可变配置摘要，源 Pod 和节点，以及在 QEMU 契约下——一个独立的 VM 状态镜像，附带负载摘要和恢复兼容性摘要。参见 [QEMU VMState 快照](/guides/qemu-vmstate-snapshots)。

暂停会持久化 rootfs，将其推送到所配置的镜像仓库，并释放 Pod 和池空位；恢复时从快照镜像重建工作负载并保留同一沙箱 ID。暂停/恢复仅限单副本沙箱（`spec.replicas=1`）。暂停期间，控制器管理一个以沙箱命名的内部快照；提交以 Job 的形式在源节点上运行。镜像仓库凭据、保留策略和容量规划是运维者的职责——删除快照记录不会删除已推送的镜像。

![暂停/恢复阶段](../../public/images/operator-pause-phases.svg)

设置 `--snapshot-image-uri-template`（Helm：`controller.snapshot.imageURITemplate`）可在首次推送前自定义快照镜像名称。空模板保留默认命名规则；命名字段和日期/时区格式化辅助函数参见[自定义镜像名称](/guides/pause-resume#custom-image-names)。

## 读取状态

phase 报告沙箱运行时健康；conditions 解释原因。两者都是 Kubernetes 原生的，可用普通 `kubectl` 查看。

| Phase | 含义 |
|---|---|
| `Pending` | 尚未观测到运行且就绪的沙箱 Pod |
| `Succeed` | 至少一个沙箱 Pod 处于运行且就绪——这是稳态，**不代表**任务完成 |
| `Pausing` / `Paused` / `Resuming` | 暂停/恢复的过渡状态 |
| `Failed` | 终态运行时失败——请检查 conditions 和 Pod 事件 |

Conditions（`Ready`、`Progressing`、`Paused`、`PauseFailed`、`ResumeFailed`、`PodFailed`、`PoolAllocationPending`）携带原因和消息；只有当某个 condition 存在且状态为 `True` 时才应计数，并且在信任更新后的 status 之前，应先核对 `status.observedGeneration` 与 `metadata.generation` 是否一致。

## 性能

启用池化后，实测交付 100 个沙箱：

| 场景 | 总耗时 |
|---|---|
| SIG Agent-Sandbox（并发 1 / 10 / 50） | 76.4 s / 23.2 s / 33.9 s |
| BatchSandbox | 0.92 s |

## 安装

通过 base 和 controller Helm chart 安装 CRD 和控制器（开发环境也可用 Kustomize）；[Kubernetes 部署](/deployment/)页面讲解了完整的技术栈，[控制器源码](https://github.com/opensandbox-group/OpenSandbox/tree/main/kubernetes)记录了 chart 取值和本地构建。
