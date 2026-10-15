---
title: Server
description: 生命周期控制平面——它的分层架构，以及单一 API 覆盖可插拔运行时、异步生命周期、限定范围的持久化和快速失败安全等设计决策背后的考量。
---

# 生命周期服务端

服务端是 OpenSandbox 部署的控制平面：所有 SDK、CLI 和 MCP 客户端都通过同一个经过认证的 HTTP API 访问它，而无需关心沙箱实际由什么运行。它负责生命周期编排——校验请求、执行访问控制、持久化服务端管理的记录——并将实际的沙箱工作委托给所配置的运行时后端（Docker、Kubernetes 或 Fast Sandbox 集成）。

![沙箱生命周期状态](../../public/images/server-lifecycle.svg)

## 架构

![服务端内部架构](../../public/images/server-architecture.svg)

服务端是一个分层组织的 FastAPI 应用，每层都可以在接口之后被替换：

- **API 层**（`api/`）— 轻薄的 HTTP 路由（生命周期、代理、池、模板、网络策略、诊断、指标）及其请求/响应模式。所有校验都发生在这里。
- **中间件**（`middleware/`） — 横切关注点：认证、请求 ID、日期头部、HTTP 指标、CORS。
- **服务层**（`services/`） — 生命周期状态机和运行时集成，下文详述。
- **仓储**（`repositories/`） — 仅针对服务端管理记录的持久化。
- **租户**（`tenants/`） — 可插拔的身份提供者，根据请求的 API key 解析其租户。
- **集成**（`integrations/`） — 可选的附加组件，例如 renew-on-access 消费者和 OpenTelemetry 指标。

两条规则定义了分层，二者都为了让控制平面保持诚实且可替换：

1. **API 永远不直接与运行时通信。** 一切都经由服务接口，因此后端细节不会越过边界泄漏——这正是 API 层得以保持轻薄、运行时可以仅凭配置替换的原因。
2. **沙箱不是服务端数据库里的行。** Docker 容器、Kubernetes 资源或 fast-sandbox CR 才是事实来源。服务端只持久化它拥有的东西（快照元数据和模板目录），这正是存储后端可以替换、故障转移可以处理的原因。

## 设计决策

### 一套契约覆盖可插拔运行时

所有运行时都实现同一个 `SandboxService` 接口，通过配置选择（`runtime.type`）：本地和单机部署用 Docker，通过控制器或 agent-sandbox 运行容器工作负载用 Kubernetes，基于模板的 microVM 用 Fast Sandbox。在 Kubernetes 下，`CompositeSandboxService` 将容器工作负载提供者与 `FastSandboxService` 组合——基于 `templateId` 的创建或以 `fsb-` 为前缀的 ID 路由到 Fast Sandbox，其余全部路由到容器提供者。API、响应模型和 SDK 在三者之间完全一致；Fast Sandbox 组合在 [Fast Sandbox](/architecture/fast-sandbox/) 中描述。

### 带绝对过期时间的异步生命周期

创建在沙箱运行之前就返回（`Pending`），客户端轮询状态或使用 SDK 就绪辅助工具。这使 API 在慢运行时下保持非阻塞——供给一个 Pod 或 microVM 绝不能让 HTTP 请求一直挂起。过期是绝对 TTL 而非空闲超时：服务端或运行时重启不会重置它，因此沙箱的生命周期仅凭创建时间即可审计。

沙箱被观测为 `Pending` 或 `Running`，经过 `Pausing` / `Paused` / `Resuming` / `Stopping` 过渡，最终进入 `Terminated`（退出码 0）或 `Failed`（非零）。`resume()` 被有意设计为暂停态操作，绝不是一个通用的重启：已经退出的工作负载处于 `Terminated` 或 `Failed`，对此的诚实回答是创建一个替代沙箱，而不是悄悄复活。在 Kubernetes 上暂停会保留沙箱 ID 并还原根文件系统——在可选择的 QEMU 模式下，甚至包括 guest 内存。参见[暂停与恢复](/guides/pause-resume)。

对于 Kubernetes BatchSandbox 工作负载，删除可能在任务清理和池释放完成之前使对象仍然可见。在此窗口内，无论之前的运行时 phase 如何，列表和单个沙箱查询都会报告 `Stopping`，原因为 `DELETING`，`lastTransitionAt` 为删除时间戳。一旦 Kubernetes 移除了该对象，它会从列表中消失，单个查询返回 `404`。

### 端点解析是契约的一部分

客户端永远不需要猜测如何访问沙箱服务。`GET /sandboxes/{id}/endpoints/{port}` 返回以下三种形式之一（由运行时和部署拓扑决定）的可达地址：直接地址（Docker 宿主机映射或 Pod IP）、ingress 网关路由，或者在直接路径和网关路径都不存在时使用的服务端代理 URL。代理是兼容性兜底，不是快速路径。

### 快速失败的安全姿态

服务端拒绝在不安全的配置下启动，而不是降级运行：在非交互式环境中没有 API key 是致命错误，除非明确确认接受不安全；所配置的运行时会在启动时针对不安全暴露进行校验；启用多租户时，每个可枚举的租户命名空间都必须存在且可访问，然后才接受流量。宁可启动时失败，也不在请求时失败，因为配置错误的隔离在每个请求层面是不可见的。

### 以命名空间隔离实现的多租户

多租户是可选的，且仅限 Kubernetes。API key 来自租户提供者（本地文件或远程 HTTP 查询），而不是单一的服务端密钥，并且每个经过认证的请求都限定在该租户的命名空间内：创建落入其中，读取和修改只解析到该租户的沙箱。身份存放在提供者中——而非服务端配置里——因此租户管理仍由拥有身份的平台负责。池路由目前仍是共享资源。配置方法：[多租户](/guides/multi-tenancy)。

### 限定范围的持久化，限定范围的高可用

服务端管理的记录——快照元数据和模板目录——默认持久化到 SQLite，或在显式配置时使用 PostgreSQL。默认部署形态是**单活**：单副本、升级期间短暂中断 API、SQLite。

PostgreSQL 并不会让服务端变成水平扩展的 API。它只启用一个非常窄的多副本能力：两个副本协调 Kubernetes 运行时上的公共快照创建。通用的负载均衡生命周期 HA 目前不支持——也不假装支持。

双副本设计不依赖租约和 leader 选举，而是借助两条既有不变式：`SandboxSnapshot` CR 的名称是确定性的，且 Kubernetes API 本身就是一个已串行化、可 watch 的存储。

1. **先观测再创建。** 副本在创建确定命名的 CR 之前先检查它是否存在；对等副本更早的创建（即使已完成）会被收敛；真正的创建竞争会被 `409 AlreadyExists` 吸收并视为成功。
2. **对终态写入做 CAS。** 终态的数据库结果以期望的前置状态为条件写入，因此当两个副本竞争完成同一快照时，一个胜出，另一个的写入是空操作。
3. **恢复扫描。** 崩溃的副本会留下非终态的行；存活的副本按间隔重新扫描未完成的行，并通过同样的 CR 观测将其推进到完成。

这里不声称跨数据库、Kubernetes 和镜像仓库的 exactly-once 保证——恢复间隔决定的是对等接管延迟，而不是原子性。双副本拓扑的 Helm 接线：[Kubernetes 部署](/deployment/#use-postgresql-for-server-persistence)。

## 后续阅读

- [Kubernetes 控制器](/architecture/control-plane/operator) — 服务端在 Kubernetes 上编程的 operator
- [Fast Sandbox](/architecture/fast-sandbox/) — 基于模板的 microVM 组合
- [多租户](/guides/multi-tenancy)、[暂停与恢复](/guides/pause-resume) — 运维指南
- [服务端配置参考](https://github.com/opensandbox-group/OpenSandbox/blob/main/server/configuration.md) — 所有配置项及默认值
