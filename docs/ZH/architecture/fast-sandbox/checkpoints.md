---
title: 暂停、恢复与快照
description: Fast Sandbox 如何将 microVM 运行时状态 checkpoint 到制品存储、释放 Fastlet 容量、在任意 Fastlet 上恢复，并将快照发布为黄金镜像制品。
---

# 暂停、恢复与快照

Fast Sandbox 将运行时状态 checkpoint 到与黄金镜像相同的按内容寻址制品存储中。暂停沙箱会写入完整的 checkpoint 并释放其 Fastlet，因此暂停的沙箱不占用任何计算资源；恢复时会在准入选择的任意 Fastlet 上还原 checkpoint — 不一定是原主机。快照（snapshot）是同一机制对外的形态：它将运行中沙箱的状态发布为可复用的制品集，新沙箱可以从中创建。

公共生命周期 API 在各后端之间保持不变：`POST /sandboxes/{id}/pause`、`/resume` 和 `/sandboxes/{id}/snapshots` 返回 **202 Accepted**，工作异步完成。服务器通过 gRPC 向 FastPath 提交意图，并在持久的 `sandbox.fast.io` Sandbox 记录上观察进度 — 与其他所有变更相同的"提交并收敛"模式（参见[高可用](/architecture/fast-sandbox/ha)）。

## checkpoint 包含什么

暂停 checkpoint 将沙箱的运行时状态捕获为存储中的一个制品集：

| 组成部分 | 内容 |
|---|---|
| `rootfs` | 沙箱的可写根文件系统 |
| `vmstate` | Firecracker 支持的模板的 MicroVM 设备和 vCPU 状态 |
| `memory` | guest 内存，捕获期间通过节点的 `/dev/shm` 暂存 |
| `SHA256SUMS` | 该制品集的摘要清单 |

Sandbox 记录的 `status.runtime.checkpoint` 携带结果：由沙箱 UID 派生的 `checkpointID`、请求暂停的 spec 代数，以及一个 `pauseAttempt` 重试纪元；已发布制品集的 `manifestRef`（`s3://` URI）；其摘要和大小。checkpoint 以摘要优先的方式写入 — **只有当制品集完整进入存储后，暂停才会被报告为完成** — 而没有记录 checkpoint 的 Paused 沙箱无法恢复，只能重置。

## 暂停与恢复

**暂停**（`POST /sandboxes/{id}/pause`）：

1. 服务器读取 Sandbox 记录，用其 `uid` 对调用加栅栏，并将携带请求 ID 的 `PauseSandbox` 发送给 FastPath。
2. FastPath 记录意图（`spec.state: Paused`）；运行时捕获 rootfs、VM 状态和 guest 内存，发布制品集，在 Sandbox 状态中记录 checkpoint，然后**释放 Fastlet 容量**。
3. 状态通过观察收敛：Sandbox 记录上的 `Pausing → Paused`。一旦暂停，经过入口的路由即被拒绝 — 暂停的沙箱是不可达的，而不是降级的。

**恢复**（`POST /sandboxes/{id}/resume`）：

1. 服务器用 UID 加栅栏并提交 `ResumeSandbox`。
2. FastPath 将记录翻转回 `Running`；控制器还原记录的 checkpoint，**可能在不同的 Fastlet 上** — 让恢复具备可移植性的是制品存储，而非节点亲和。
3. 恢复会推进路由代数，因此客户端必须重新解析端点；现有路由不会跟随沙箱迁移到新主机。

只有 Ready 状态的运行时才能暂停；UID 不匹配和缺少 checkpoint 会以 `409 Conflict` 的形式呈现。

## 快照（Snapshot）

快照将运行中沙箱的状态转化为持久的、可复用的制品集 — 以与模板黄金镜像相同的按内容寻址布局发布，因此恢复的快照和原生模板在创建时的行为完全一致。

1. `POST /sandboxes/{id}/snapshots` 仅接受 Running 状态的源沙箱，以确定性名称和 `sandboxRef.uid` 栅栏创建 `SandboxSnapshot` 记录，然后持久化 `CREATING` 行 — 运行时意图始终存在于一个可读的行之后，在每个副本上都是如此。
2. 平台推动该记录经历 `Pending → Creating → Publishing → Succeeded/Failed`。创建过程可能在状态捕获期间短暂暂停沙箱；发布过程写入制品集以及一条以快照模板名称为键的索引条目。
3. 服务器通过监听记录来收敛该行，在读取时复查，并以 CAS 写入终态；恢复会重新提交相同的幂等意图，因此崩溃的服务器留下的行是可恢复的，而不是卡死的。
4. `POST /sandboxes {"snapshotId": ...}` 从 `READY` 快照恢复：索引条目像模板一样解析，创建请求路由到 Fast Sandbox。沙箱 ID 是新的；快照是一个起点，而非延续。

快照名称是具有最后写入者胜出语义的集群级索引键，因此 FastPath 在接受新持有者之前会拒绝同名非终态记录。删除快照记录不会删除已推送的制品 — 存储的保留策略由运维方决定（参见[存储](/architecture/fast-sandbox/storage)）。

## 为什么写时复制在这里是安全的

- **checkpoint 是不可变制品。** 恢复和还原从不信任可变标签；每个制品集都通过清单摘要引用，并可对照 `SHA256SUMS` 验证。
- **节点本地的 rootfs 副本是 CoW reflink。** 每个节点维护一个私有的 XFS 状态根，其中每沙箱的 rootfs 实例是缓存制品的 reflink 副本 — 即时置备而无需复制字节，也不会跨故障域共享状态（参见 [Firecracker](/architecture/fast-sandbox/firecracker)）。
- **分发是摊销的，而非共享的。** [DART](https://github.com/data-accelerator/dart) 使每个块在整个集群范围内大约只从源站获取一次，并通过节点本地缓存向对等节点提供服务，因此暂停后的恢复只需付出接近本地克隆的代价。

## 与 BatchSandbox 暂停/恢复的关系

Fast Sandbox 和 Kubernetes operator 运行着**两套相互独立的 checkpoint 机制**。`sandbox.fast.io` Sandbox 记录为基于模板的沙箱保存 microVM 运行时状态（rootfs + VM 状态 + 内存）；operator 的 `SandboxSnapshot`（`sandbox.opensandbox.io`）将容器 rootfs — 以及 QEMU 契约下的 VM 状态 — 提交到 OCI registry（参见[暂停与恢复](/guides/pause-resume)）。它们共享生命周期 API 和 202 语义，但不共享制品格式或控制平面。
