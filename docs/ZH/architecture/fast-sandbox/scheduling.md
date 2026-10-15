---
title: Fast Sandbox 调度
description: 池与 Fastlet — 创建热路径、容量与放置，以及从创建到暂停、恢复和到期的沙箱生命周期。
---

# 调度

Fast Sandbox 将沙箱调度进预热的 **Fastlet** pod **池**中。池固定运行时配置和资源形态；一个 Fastlet 承载多个沙箱；一个沙箱占用一个 Fastlet 的一部分。结果是：创建沙箱就是对已存在容量的一次常数时间准入，而不是一轮全新的调度。

![The create hot path](../../public/images/fast-sandbox-create-path.svg)

## 池

`SandboxPool` 是容量和策略的单元：

- **`capacity`** 决定 Fastlet 舰队的规模 — 保留多少个热 pod，以及上限。
- **`runtime`** 选择一个不可变的、平台所有的运行时配置（Firecracker profile）。池中的所有沙箱共享它。
- **`sandboxResources`** 是该池产出的每个沙箱的不可变资源形态。
- **`fastletTemplate`** 是 Fastlet pod 自身的普通 Kubernetes pod 模板 — 节点选择器、亲和性和打散保持 Kubernetes 原生。
- **`infraComponents`** 声明 pod 本地的基础设施（例如出口 action 处理器）。它被编译为一个不可变的修订版本：更新它会滚动出新的 Fastlet，而既有沙箱继续停留在它们被接纳时的修订版本上。
- **`maxSandboxesPerPod`** 约束密度；**`warmImages`** 列出要预拉取的制品，受保护不被普通缓存淘汰，因此就绪永远不会等待拉取。

## 放置与准入

当服务器提交创建请求时，FastPath 会对内存中的候选排序 — 先是镜像缓存亲和性，然后是归一化负载，最后以稳定哈希决胜 — 并在选定的 Fastlet 上原子性地接纳沙箱。两个性质使这不仅是快，而且是安全的：

- **网络槽位先于运行时存在**：netns、veth 和伪装按沙箱预先置备，附属身份加策略绑定随准入一起交付。沙箱在其策略就绪之前永远不可达（参见[网络](/architecture/fast-sandbox/networking)）。
- **Sandbox CR 先被完整写入**：意图、初始策略和池引用在运行时启动之前就已持久化，因此任何失败中断的准入都可以通过协调恢复，而不是泄漏一个创建到一半的沙箱。

## 沙箱生命周期

Sandbox CR 通过三个字段承载生命周期：

- **`state`** — `Running` 保持一个活跃运行时；`Paused` 将其 checkpoint 到制品存储并释放 Fastlet 容量。恢复会在**可能是不同的 Fastlet 上**还原 checkpoint，这就是为什么恢复之后路由必须重新解析。
- **`expireTime`** — 由平台强制执行的绝对删除期限。
- **`failurePolicy` + `recoveryTimeoutSeconds`** — Fastlet 丢失时怎么办（默认：失联 60 秒后手动恢复）。`resetRevision` 触发一次有意的重置和重新调度。

任务执行是准入的一部分：请求的 entrypoint 作为任务交付给沙箱，而不是烘焙进 pod，这正是预热的 Fastlet 能够服务任意模板的原因（参见[模板](/architecture/fast-sandbox/templates)）。

快照和暂停/恢复遵循与其他所有变更相同的提交并收敛模式 — 参见[暂停、恢复与快照](/architecture/fast-sandbox/checkpoints)和[高可用](/architecture/fast-sandbox/ha)。
