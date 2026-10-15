---
title: Fast Sandbox
description: OpenSandbox 与 fast-sandbox 平台的集成 — 通过 Fastlet 池和 FastPath 控制平面交付基于模板的沙箱。
---

# Fast Sandbox 集成

Fast Sandbox 是一个运行时后端，将 OpenSandbox 连接到外部的 [fast-sandbox](https://github.com/opensandbox-group/fast-sandbox) 平台。它在预热的 **Fastlet** pod 中置备基于模板的 **Firecracker microVM** 沙箱，将 Kubernetes 调度器、watch 传播和 kubelet 路径从创建热路径中移除。

适配器和端点集成位于本仓库中。FastPath、Fastlet 运行时以及 `sandbox.fast.io` CRD 由 fast-sandbox 平台所有，按原样使用。

## 它如何融入整体

- **后端选择**：Fast Sandbox 不是一个独立的 `runtime.type`。在 `runtime.type = "kubernetes"` 下，`CompositeSandboxService` 将携带 `templateId` 的创建请求路由到 `FastSandboxService`，并将对现有 `fsb-` 前缀沙箱 ID 的操作分派给它。基于镜像和 snapshot 的请求继续使用 Kubernetes 工作负载提供程序。
- **控制协议**：服务器和入口通过 gRPC 与 fast-sandbox 控制平面（FastPath v2）通信。gRPC 在 FastPath 处终止：FastPath/Reconciler 到 Fastlet 的控制协议是 HTTP 上的 JSON（`/api/v2/fastlet/*`）。FastPath 负责变更和实时运行时操作；持久化状态的读取使用 Kubernetes 对 `sandbox.fast.io` 自定义资源的 LIST/WATCH。
- **租户**：租户提供程序将每个经过身份验证的租户映射到一个 fast-sandbox 命名空间；请求范围内的操作从上下文中解析它，后台工作则回退到按沙箱 ID 查找。

![Fast Sandbox architecture](../../public/images/fast-sandbox-architecture.svg)

👉 <a href="/architecture/fast-sandbox/architecture.html" target="_blank" rel="noopener">交互式架构图</a> — 在全视图可探索的地图中了解沙箱创建、模板制品流转、checkpoint 和部署拓扑。

## 核心组件

| 组件 | 职责 | 设计目标 |
|---|---|---|
| FastPath（gRPC 控制平面） | 处理每一次变更和实时操作：创建、更新、到期、端点解析、snapshot | 通过内存中放置实现常数时间创建；可重启，因为持久状态存放在 CRD 中，而不是这里 |
| Reconciler（控制器） | 将 CR 推动为现实：构建模板、运行池、放置沙箱、完成 snapshot | 通过观察收敛 — 无需 leader 选举，任何重启后都从 CR 重放 |
| Sandbox CRD（`sandbox.fast.io`） | 每个沙箱的意图、放置、策略绑定和观察结果的持久记录 | 意图在任何组件故障下都能幸存；CR 是事实来源 |
| Fastlet（预热 pod） | 在一个网络域中承载多个沙箱；提供运行时容量和共享服务 | 密度与隔离兼顾：预置的网络槽位，共享出口和 DNS 在多个沙箱间摊销 |
| 运行时代理（每节点） | 通过 unix socket 管理 Firecracker VM 生命周期；节点就绪检查和自我标注；安装资产 | 节点自治：控制平面委托，节点在接纳工作负载前自行验证 |
| [DART](https://github.com/data-accelerator/dart)（每节点） | 节点状态根之间的点对点制品分发 | 摊销源站开销：每个块在整个集群范围内大约只从存储获取一次 |
| 出口进程（每 Fastlet） | 多沙箱出站策略：主体、Sandbox Actions 处理器、DNS 和网络强制执行 | 每个沙箱 fail-closed 策略，无需每沙箱 sidecar — 策略可以迟到，但绝不能先开 |
| 模板构建器 | 将 OCI 镜像转换为黄金镜像制品（内置 guest 接线、init、execd） | 在任何请求之前固定工作负载：创建只表达意图，而非配置 |
| Janitor | 清扫失败操作产生的孤立资源 | 协调会完成被中断的工作；janitor 移除不应再存在的东西 |

## 设计原则

- **不新增公共 API**：基于模板的创建和 `fsb-` ID 分派沿用现有的生命周期契约。SDK 在不同后端之间无需修改即可工作，也不存在沙箱到后端的注册存储 — ID 前缀就是分派器。
- **CRD 优先的控制**：每个沙箱的持久意图存放在 `sandbox.fast.io` 资源中；FastPath gRPC 承载变更热路径。Kubernetes 调度器、watch 传播和 kubelet 离开创建路径 — 持久写入不会。
- **fail-closed 的多租户出口**：每个主体在注册时即拒绝流量，直到其数据平面就绪 — 策略交付可以迟到，但绝不能提前开放。
- **经过身份验证的路由**：沙箱路由是在入口处验证的已签名 `f1.` 作用域；端点签发与就绪状态解耦。
- **基于模板的 Firecracker**：黄金镜像为 Firecracker microVM 构建 — 运行时形态在构建时固定，而非按请求确定。

## 文档

| 页面 | 内容 |
|---|---|
| <a href="/architecture/fast-sandbox/architecture.html" target="_blank" rel="noopener">交互式架构</a> | 可探索的组件地图：沙箱创建、模板制品流转、checkpoint 和部署拓扑 |
| [模板](/architecture/fast-sandbox/templates) | 模板目录、黄金镜像构建、制品发布 |
| [调度](/architecture/fast-sandbox/scheduling) | 池与 Fastlet、创建热路径、生命周期以及暂停/恢复 |
| [网络](/architecture/fast-sandbox/networking) | 网络槽位、入站路由作用域与解析、出站出口强制执行 |
| [暂停、恢复与快照](/architecture/fast-sandbox/checkpoints) | Checkpoint 制品、跨 Fastlet 的暂停/恢复、snapshot 发布与恢复 |
| [高可用](/architecture/fast-sandbox/ha) | 哪种故障能被哪些机制幸存：控制平面、节点、沙箱 |
| [存储](/architecture/fast-sandbox/storage) | 制品存储、点对点块分发、节点状态根 |
| [Firecracker](/architecture/fast-sandbox/firecracker) | 主机要求、每 VM 网络模型、代理委托 |
| [性能](/architecture/fast-sandbox/performance) | ACK 创建、请求、snapshot 以及同节点暂停/恢复的测量数据 |
