---
title: Fast Sandbox Firecracker
description: fast-sandbox 集成中的 Firecracker microVM 支持 — 每 VM 网络、隔离保证以及快照还原特性。
---

# Firecracker 支持

Fast Sandbox 模板为 Firecracker microVM 构建黄金镜像，每个沙箱都作为真正的 KVM 虚拟机运行在 Fastlet pod 的网络命名空间中，与共享出口进程并存。每 VM 网络、隔离和 checkpoint 机制已在真实硬件上完成端到端验证。

## 拓扑

```text
host
└── Fastlet pod netns (docker bridge)
    └── fastlet container (privileged, /dev/kvm)
        ├── egress container (--network container:fastlet)   shared control plane
        ├── firecracker vm1 (10.30.0.5, gw 10.30.0.1)        policy + vault per VM
        └── firecracker vm2 (10.30.0.6, gw 10.30.0.2)        policy + vault per VM
              tap ── br0 ── veth ──► pod netns (per-subject policy enforcement)
```

这些 microVM 是[出口配置](/architecture/fast-sandbox/networking#outbound-one-egress-control-plane-n-subjects)的一等主体：每个 VM 通过 `SET_BINDING` 及其附属块（IP、网关、主机 veth、私有 CIDR）注册，在 `sandbox.data-plane-ready` 之前保持默认拒绝，并拥有策略、凭据和内核规则的独立分片。模板构建和按需制品分发在[模板](/architecture/fast-sandbox/templates)和[存储](/architecture/fast-sandbox/storage)中描述。

## 节点架构：代理委托与 P2P 分发

fast-sandbox 控制平面不直接驱动 Firecracker。每个节点运行一个 **runtime-agent**（DaemonSet），在本地管理 VM 生命周期；控制平面通过主机 unix socket（`/run/fast-sandbox/firecracker/runtime.sock`）将 VM 操作委托给它。就绪状态通过同一 API 上报 — 代理在响应 Firecracker 健康检查的同时回答 `dartUp`。

黄金镜像制品由 **DART** 分发，这是一个具备点对点回退能力的节点本地守护进程。每个节点保留一个私有的状态根子目录，因此一个节点的缓存永远不会满足另一个节点的拉取；块级获取（4 MiB）遵循 `cache → peer → origin` 路径，S3 兼容的源站对每个块在整个集群范围内大约只被命中一次。

![Firecracker node architecture](../../public/images/fast-sandbox-firecracker-node.svg)

对等发现使用 headless 的 `dart` Service：每个守护进程在开始拉取前必须看到完整的成员名单，因此第二个节点的首次创建由第一个节点的块提供服务，而不是源站。每节点的块来源计数器（`cache` / `peer` / `origin`）使分发路径可观测。

## 主机与节点要求

以下是集成在节点承载 Firecracker 沙箱之前实际执行的检查（chart + 环境工具）：

**主机**（裸金属，或暴露了 KVM 的嵌套虚拟化 VM）：

- **主机内核 ≥ 5.10。**
- `/dev/kvm` 存在并透传进节点容器，连同 `/sys` 下的 KVM 设备节点。只要 KVM 被暴露，嵌套虚拟化即可工作；没有它的主机永远不会被打标签，也永远不会收到 microVM 沙箱。
- `/dev/net/tun` — TAP 设备承载每个沙箱网络命名空间的数据平面。Firecracker 驱动只探测 `/dev/kvm` 和 `/dev/net/tun`；`vhost-vsock` 被有意不挂载。
- `/dev/shm` 大小超出容器默认值 — snapshot 内存暂存发生在这里。
- **cgroup v2**（统一层级）；cgroup v1 主机直接被拒绝。
- `fs.inotify.max_user_instances` 在集群存续期间于主机上调高。
- 支撑节点状态根的 **XFS 文件系统且支持 reflink**：每沙箱的 rootfs 副本是写时复制的 reflink，安装过程会显式探测 reflink — 如果探测失败，安装会拒绝而不是降级。禁用 XFS 会回退到完整复制。Reflink 只在单一文件系统内有效，这正是每个节点绑定单一共享 XFS 卷中自己的子树的原因。
- 余量：节点就绪门默认要求 **10 GiB 可用磁盘**和 **2 GiB 可用内存**。

**节点就绪是自助式的。** firecracker-runtime DaemonSet 在每个候选节点上运行，执行主机检查，安装 Firecracker 资产（默认版本 `v1.16.1`；支持可选的自定义内核 URL — 此部分在每次就绪检查时热重载），然后才为节点打标签（`sandbox.fast.io/kvm=true`、`fast-sandbox.io/firecracker-node=true`）并设置 `FirecrackerReady` 条件。池只将被调度的 Fastlet 放置在已打标签的节点上；运维人员从不手工打标签。

**本仓库不断言的事项：** NVMe 要求 — 上述检查是 chart 和环境工具中表达的那些；更深入的检查（特定存储设备）将位于 fast-sandbox 运行时源代码中，它不在本仓库范围内。

## 网络模型

- **每 VM 网关地址。** 每个 VM 在 pod netns 中获得自己的 veth 对和网关 IP。在 VM 之间共享一个网关会在 `arp_ignore=1` 下失效：第二个 VM 的 ARP 得不到应答，其流量永远无法离开 guest。每 VM 网关与真实的每 VM veth 相匹配，出口按附属关系自动分派。
- **网络命名空间可见性。** `/var/run/netns` 在两个容器中都以 `:rslave` 传播方式挂载，使容器启动后创建的 VM netns 对管理工具可见；默认的 `rprivate` 传播会隐藏它们。
- **guest 流量绕过容器 netfilter。** 从 pod 视角看，跨越 VM netns 的流量是 L2 桥接流量。因此权威的强制执行层是 pod-netns 转发链 — pod-netns 层，而不是任何容器 OUTPUT 路径。

生命周期顺序是固定的：创建 VM netns → `SET_BINDING`（默认拒绝）→ `data-plane-ready` → 沙箱流量在其自身策略下流动。

## 已测量的特性

ACK 测量涵盖模板创建、请求延迟、实时快照以及同节点暂停/恢复；结果和环境细节见[性能](/architecture/fast-sandbox/performance)。冷内核启动和 DART P2P 性能在该次运行中未经验证。

## 隔离保证

- **每 VM 策略强制执行**：每个 VM 的出站流量在 DNS 和网络层由其自身的主体策略过滤。
- **跨 VM 隔离**：一个 VM 访问另一个 VM 的主机时，会在 DNS 解析阶段被其自身策略拒绝。
- **防篡改**：guest 在 pod 网络命名空间中没有 `NET_ADMIN`；强制执行规则从 guest 内部不可见也不可更改。
