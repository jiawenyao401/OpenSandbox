---
title: Fast Sandbox 存储
description: 一个按内容寻址的制品存储，服务黄金镜像和 checkpoint；节点私有缓存并通过点对点交换块。
---

# 存储

Fast Sandbox 将存储简化为一种形态：**沙箱运行或恢复所需的每一样东西都是一个按内容寻址的制品**，存放于单一的 S3 兼容存储中。黄金镜像、暂停 checkpoint 和快照都是制品，通过清单摘要引用 — 从不变异，始终可验证。

![Artifact storage and delivery](../../public/images/fast-sandbox-artifacts.svg)

## 制品从哪里来

- **模板构建**将 OCI 镜像（加上 guest 接线，对 microVM 运行时还有内核）转换为黄金镜像制品并发布；SandboxTemplate CR 记录清单引用和摘要（参见[模板](/architecture/fast-sandbox/templates)）。
- **暂停 checkpoint 和快照**由同一机制写入：运行时状态被捕获、推送，并以摘要优先的方式记录在 SandboxSnapshot CR 中（参见[暂停、恢复与快照](/architecture/fast-sandbox/checkpoints)）。恢复从不信任可变标签。

## 制品到哪里去

节点保留一个**私有状态根** — 每个节点本地的 XFS reflink 支撑的缓存，绝不在节点之间共享。这是刻意为之：fastlet 的即时启动来自节点本地的制品，而跨节点的状态共享会让每个节点成为其他每个节点故障域的一部分。

私有缓存的代价是重复下载，这正是 **[DART](https://github.com/data-accelerator/dart)** 所消除的：每节点一个节点本地的分发守护进程，通过 headless Service 名单实现点对点感知。块级获取（4 MiB）遵循 `cache → peer → origin` — 每个块在整个集群范围内大约只从源站获取一次，然后以点对点方式服务。每节点的块来源计数器使分发路径可观测。

## 刻意*不*存储的东西

- **出口凭据**仅存于出口进程的内存中 — 按主体通过代理路由推送，绝不写入 Secret 卷或磁盘。
- **沙箱运行时写入**（沙箱创建的文件）存在于沙箱自己的分片中；只有在显式的 checkpoint 或快照要求时才会被捕获。沙箱的存储是生命周期的一次行为，而不是后台同步。
- **路由和策略状态**存在于 CRD 中，而不是制品存储中 — 存储承载字节，集群承载语义。

## 运维

制品存储就是普通的 S3 兼容对象存储；容量规划简化为制品大小和保留策略。删除 SandboxSnapshot 记录不会删除已推送的制品 — 存储的保留和清理是运维策略，刻意保持在沙箱生命周期之外。

### 准备节点状态盘

`fast-sandbox` chart 从主机挂载 `runtime.stateRoot`（
默认 `/var/lib/fast-sandbox/firecracker`）。`DirectoryOrCreate` 会在现有文件系统上创建一个
目录；它不会置备单独的磁盘。
使用 `opensandbox` umbrella chart 时，设置 `fast-sandbox.runtime.stateRoot`。
将 chart 值设置为与 helper 挂载点相同的目录。
在每个节点上**部署运行时 DaemonSet 之前**准备好此挂载。
否则，在挂载之后需要排空节点的沙箱并重启运行时 Pod：
已在运行的沙箱可能继续看到旧的文件系统。

优先使用启用 reflink 的专用 XFS 数据盘。对于**全新的、空的设备**，
在验证其身份且确认其中不含所需数据之后：

```bash
sudo mkfs.xfs -m reflink=1 /dev/vdb
sudo mkdir -p /var/lib/fast-sandbox/firecracker
sudo mount -o noatime /dev/vdb /var/lib/fast-sandbox/firecracker
sudo blkid /dev/vdb
```

使用设备的 UUID 将挂载持久化到 `/etc/fstab`。格式化会销毁
设备内容；绝不要在既有状态盘上运行这些命令。
挂载前保持挂载点为空，以免已有状态被遮蔽。

当没有额外设备可用时，仓库提供了
[`scripts/fast-sandbox-state-root-setup.sh`](https://github.com/opensandbox-group/OpenSandbox/blob/main/scripts/fast-sandbox-state-root-setup.sh)：

```bash
bash scripts/fast-sandbox-state-root-setup.sh --dry-run 50G
sudo bash scripts/fast-sandbox-state-root-setup.sh 50G
# Optional: SIZE MOUNTPOINT BACKING_FILE
sudo bash scripts/fast-sandbox-state-root-setup.sh 100G /data/firecracker /data/firecracker.xfs
```

该 helper 分配一个新的 backing file，以 XFS `reflink=1` 格式化，
以 `loop,noatime` 挂载，并验证一次真实的 reflink 复制。它要求 Linux、
`xfsprogs`、GNU coreutils 和 util-linux。它会拒绝已存在的 backing file、
符号链接路径、已挂载的目标和非空挂载点。它会打印一条 fstab
条目供审核；它不会编辑 fstab 或重启工作负载。失败时，
它会保留任何已创建的 backing file 供检查，而不是冒删除
已挂载数据的风险。在清理或重试之前，请检查 `findmnt` 和 `losetup -j BACKING_FILE`。

与稀疏文件不同，该 helper 使用 `fallocate` 预留请求的空间
并检查 backing 文件系统上还剩至少 10 GiB。
这是一次时间点检查，不是主机级预留：其他工作负载仍可能
填满该文件系统，瘦置备存储需要自己的容量
监控。专用磁盘可避免将状态缓存与根盘使用量耦合。

### 容量与就绪

为每个不同缓存模板的 rootfs **加上 guest 内存快照**、
活跃沙箱的可写数据以及缓存开销做预算。以至少 50% 的工作负载
余量起步，并在状态文件系统上保持 **10 GiB 以上可用空间**（默认的
主机就绪底线）。helper 的最小卷大小为 11 GiB；这不是
生产环境配置建议。Reflink 省去了初始复制，但后续
写入仍会分配块。

旧模板制品不会仅仅因为模板版本变化就自动回收。跟踪磁盘使用量，下线不用的模板，排空使用它们的
沙箱，并在手动清理其本地缓存之前停止节点运行时。
不要删除存活的 `images`、`cache` 或 `jails` 目录。保留恢复
所需的任何状态，并预期清理后制品会被再次下载。

置备完成后，验证文件系统和运行时的就绪报告：

```bash
# Use your configured runtime.stateRoot (for example /data/firecracker):
STATE_ROOT=/var/lib/fast-sandbox/firecracker
findmnt -M "$STATE_ROOT"
df -h "$STATE_ROOT"
sudo xfs_info "$STATE_ROOT"   # expect reflink=1
kubectl -n opensandbox-system get daemonsets
# Use the deployed Firecracker runtime DaemonSet name:
kubectl -n opensandbox-system rollout restart daemonset/<runtime-daemonset>
kubectl -n opensandbox-system logs daemonset/<runtime-daemonset> --all-containers --tail=100
```

`stateroot-filesystem` 检查应报告 `type xfs` 和
`reflink CoW supported`。可用空间不足会使就绪失败，并可能移除
Firecracker 调度标签；缺少 reflink 支持会警告并回退
到完整 rootfs 复制。仅有健康的状态盘并不保证就绪：
KVM、内存和运行时资产也必须通过各自的检查。
