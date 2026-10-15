---
title: 架构
description: OpenSandbox 架构总览，涵盖客户端 SDK、生命周期控制平面、运行时后端以及数据平面组件。
---

# OpenSandbox 架构

OpenSandbox 是一个面向 AI 应用的通用沙箱平台。它提供客户端 SDK 和工具、协议定义、生命周期控制平面、Docker 与 Kubernetes 运行时后端（同时支持容器工作负载和基于模板的 microVM），以及沙箱内执行组件，覆盖命令执行、文件操作、代码解释器、浏览器自动化、桌面环境和训练工作负载。

本文描述当前仓库的架构，以及公共契约、服务端实现、运行时提供者和沙箱数据平面组件之间的主要边界。

## 架构总览

![OpenSandbox 架构](../public/images/architecture-overview.svg)

该架构区分了三条主要交互路径：

- **生命周期管理**（橙色箭头）：应用、智能体和运维人员向 FastAPI 生命周期服务端发送生命周期请求（创建、列举、续期、暂停、恢复或删除沙箱，以及管理快照和模板）。服务端持久化元数据，并将供给与编排工作委托给所配置的运行时后端。
- **执行与服务访问**（蓝色箭头）：对于命令、文件、PTY 和 Jupyter 代码执行，或访问用户工作负载服务，客户端绕过生命周期编排，直接连接到已解析的端点——通过宿主机/Pod 直接地址、生命周期服务端代理，或可选的 Kubernetes ingress 网关。
- **出站控制**（绿色箭头）：沙箱向外部 API、模型提供方或 Web 服务发出的出站请求，可以通过出站控制进行过滤和安全加固（容器工作负载使用每个沙箱一个的 sidecar，FastSandbox 使用每个 Fastlet Pod 一个的共享出站进程），支持 DNS/nftables 策略，以及可选的 Credential Vault 凭据注入。

运行时与网络能力取决于所选后端；图中展示的是共享架构，并不要求部署每一个组件。

OpenSandbox 围绕六个实用层面组织：

1. **客户端层** - 供应用、智能体和运维人员使用的 SDK、`osb` CLI 和 MCP 服务端。
2. **协议层** - `specs/` 下的 OpenAPI 契约，覆盖生命周期、诊断、沙箱内执行和出站策略。
3. **生命周期控制平面** - `server/` 下的 FastAPI 服务端，负责请求认证、配置校验、持久化服务端管理的记录，并将生命周期工作委托给所配置的运行时服务。
4. **运行时后端** - 用于本地和单机部署的 Docker，以及 Kubernetes 及其容器工作负载提供者（BatchSandbox 或 `kubernetes-sigs/agent-sandbox`），并可选集成 FastSandbox 以支持基于模板的 microVM。
5. **沙箱数据平面** - 容器或 microVM 内的用户工作负载与 `execd` 守护进程，可选支持 Jupyter/代码解释器。卷与出站集成取决于运行时。
6. **网络与安全平面** - 端点解析、服务端代理、Kubernetes ingress 网关路由、安全端点访问、出站策略执行、凭据保管库、资源限制以及安全容器运行时。

这种划分是有意为之：SDK 和工具应当依赖公共契约，服务端应当负责生命周期编排，运行时提供者应当负责平台特定的资源创建，而 `execd`/出站组件应当负责发生在沙箱网络与文件系统命名空间内部的操作。

具体实现按架构层次记录在 `architecture/` 之下：

| 层次 | 职责 | 文档 |
|---|---|---|
| 控制平面 | 生命周期编排、工作负载编排、快照 | [Server](/architecture/control-plane/server)、[Kubernetes 控制器](/architecture/control-plane/operator) |
| 数据平面 | 沙箱内执行、节点侧采集 | [Execd](/architecture/data-plane/execd)、[Node Agent](/architecture/data-plane/node-agent) |
| 网络 | 入站访问、出站策略 | [Ingress](/architecture/network/ingress)、[Egress](/architecture/network/egress)、[单机网络](/architecture/network/single-host-network) |
| 后端集成 | fast-sandbox 平台上基于模板的沙箱 | [Fast Sandbox](/architecture/fast-sandbox/) |

## 1. 客户端层

客户端层是 OpenSandbox 面向开发者的入口。

### 1.1 沙箱 SDK

沙箱 SDK 将生命周期操作和沙箱内操作封装在语言原生的 API 之下：

- Python：`sdks/sandbox/python`
- JavaScript/TypeScript：`sdks/sandbox/javascript`
- Java/Kotlin：`sdks/sandbox/kotlin`
- C#/.NET：`sdks/sandbox/csharp`
- Go：`sdks/sandbox/go`

常见能力包括：

- 创建、列举、查看、暂停、恢复、续期和删除沙箱。
- 为沙箱端口解析服务端点。
- 执行命令，支持流式输出以及后台状态/日志轮询。
- 管理文件和目录。
- 从 `execd` 读取资源指标。
- 在挂载了出站 sidecar 时，查看或修改运行时出站策略。

生成的 OpenAPI 客户端与手写的适配器并存。生成的代码负责普通的请求/响应 API；手写层负责 SDK 的人机工学、流式传输、传输层生命周期、错误映射和高层模型。

### 1.2 代码解释器 SDK

代码解释器 SDK 构建在沙箱 SDK 和 `execd` 代码执行 API 之上。它们管理代码执行上下文，并提供面向语言的代码执行辅助工具。

官方代码解释器镜像维护在 [opensandbox-group/sandbox-images](https://github.com/opensandbox-group/sandbox-images)（此前位于 `sandboxes/code-interpreter/`；参见[迁移参考](/reference/code-interpreter-image-migration)）。它提供 Python、Java、Node.js 和 Go 运行时，以及 Python、Java、TypeScript/JavaScript、Go 和 Bash 的 Jupyter kernel。具体语言版本由镜像控制，并通过 `PYTHON_VERSION`、`JAVA_VERSION`、`NODE_VERSION` 和 `GO_VERSION` 等环境变量选择。

### 1.3 CLI 与 MCP

`cli/` 下的 `osb` CLI 是日常沙箱操作的终端接口：

- `osb sandbox`：生命周期与端点管理
- `osb command`：命令执行、后台日志和 shell 会话
- `osb file`：文件与目录操作
- `osb egress`：运行时出站策略的查看与修改
- `osb devops`：底层诊断
- `osb skills`：OpenSandbox 专属的智能体技能安装

`sdks/mcp/sandbox/python` 下的 MCP 服务端向支持 MCP 的客户端（如 Claude Code 和 Cursor）暴露聚焦的沙箱生命周期、命令和文本文件工具。

## 2. 协议层

OpenSandbox 将 `specs/` 视为公共契约的唯一事实来源。

### 2.1 生命周期 API

`specs/sandbox-lifecycle.yml` 定义了生命周期 API，由服务端以 `/v1` 为基础路径提供。

主要资源组：

- **Sandboxes**：从镜像、快照或模板创建；列举、获取、删除、暂停、恢复、续期过期时间以及解析端口端点。FastSandbox 还支持通过 `GET/PUT/PATCH/DELETE /sandboxes/{sandboxId}/networkpolicy` 查看和管理运行时网络策略，并在端点查询时通过 `OPEN-SANDBOX-ORIGIN` 响应头向客户端报告沙箱来源。
- **Snapshots**：从沙箱创建持久快照，列举快照，获取快照状态，删除快照。
- **Templates**：在 FastSandbox 使用的模板目录（`/templates`）中创建、列举、查看和删除模板定义。

重要的请求特性：

- `image` 或 `snapshotId` 用于容器启动，或 `templateId` 用于 Kubernetes 模式下的 FastSandbox 路径。
- `entrypoint`、环境变量、元数据以及不透明的 `extensions`。
- `resourceLimits` 用于 CPU、内存、GPU 及未来的资源键。
- `platform` 约束。
- `volumes` 用于宿主机路径、平台托管的命名卷/PVC，以及 OSSFS。
- `networkPolicy` 用于出站网络策略配置。
- `secureAccess` 用于需要端点凭据的 Kubernetes ingress 网关部署。

### 2.2 诊断 API

`specs/diagnostic-api.yml` 定义了尽力而为的沙箱日志与事件诊断描述符。服务端还提供实用的 DevOps 诊断路由，为运维人员和 AI 排障工作流返回纯文本。

### 2.3 Execd API

`specs/execd-api.yaml` 定义了由 `components/execd/` 暴露的沙箱内执行 API。

主要能力：

- 健康检查：`GET /ping`
- 代码上下文与执行：`/code/contexts`、`/code/context`、`/code`
- Bash 会话：`/session`
- 命令：`/command`、命令状态、后台命令日志
- 文件与目录：`/files/*`、`/directories`
- 指标：`/metrics`、`/metrics/watch`

命令和代码执行使用 Server-Sent Events 流式输出。当前 `execd` 实现还包括 `/pty` 下的交互式 PTY WebSocket 端点，用于长时间存活的 shell 会话。

### 2.4 出站 API

`specs/egress-api.yaml` 定义了由 `components/egress/` 暴露的运行时策略与凭据代理 API：

- **策略**：`GET /policy` 查看当前策略；`PATCH /policy` 在运行时修改规则。
- **Credential Vault**：对 `/credential-vault` 的 `GET`、`POST`、`PATCH` 和 `DELETE`，用于管理沙箱本地的凭据绑定和 token 注入，而不向工作负载暴露原始密钥。

对于容器后端，调用者解析出站 sidecar 端口对应的沙箱端点并直接连接（当启用 sidecar 认证时转发端点认证头）。对于 FastSandbox，运行时网络策略通过生命周期 API（`/sandboxes/{sandboxId}/networkpolicy`）管理，而共享出站进程（每个 Fastlet Pod 一个）使用内部请求头路由策略和按主体的凭据保管库操作。

## 3. 生命周期控制平面

`server/` 下的生命周期服务端是一个 FastAPI 应用。它负责请求校验、API-key 认证、服务端配置、生命周期编排、端点格式化、诊断以及服务端管理的持久化。

### 3.1 服务端结构

关键包：

- `opensandbox_server/main.py`：应用启动、中间件、路由注册、运行时校验以及续期意图启动逻辑。
- `opensandbox_server/api/`：生命周期路由、代理路由、池路由、诊断路由以及请求/响应模式。
- `opensandbox_server/services/`：生命周期服务接口以及 Docker/Kubernetes 实现。
- `opensandbox_server/services/k8s/`：Kubernetes 工作负载提供者、端点解析、卷/出站辅助、informer 支持以及特定提供者的映射。
- `opensandbox_server/repositories/`：持久化适配器，用于服务端管理的快照记录和 FastSandbox 模板目录。
- `opensandbox_server/integrations/renew_intent/`：可选的访问时自动续期集成。
- `opensandbox_server/middleware/`：API-key 认证和请求 ID 中间件。

### 3.2 运行时服务选择

服务端根据 `[runtime].type` 选择其生命周期实现：

- `docker` -> `DockerSandboxService`
- `kubernetes` -> `CompositeSandboxService`，它将 `KubernetesSandboxService`（由 BatchSandbox 或 agent-sandbox 提供者支撑）与用于基于模板的 microVM 执行的 `FastSandboxService` 组合在一起。

顶层 `runtime.type` 配置项严格限定为 `docker` 或 `kubernetes`。所有实现都满足同一个 `SandboxService` 接口，使 API 路由保持轻薄，将行为委托给服务层。在 Kubernetes 模式下，`CompositeSandboxService` 会检查创建请求：指定 `templateId` 的请求分发给 `FastSandboxService`，而指定 `image` 或 `snapshotId` 的容器请求分发给 `KubernetesSandboxService`。对已有沙箱的操作按其 ID 前缀路由（以 `fsb-` 开头的 ID 路由到 `FastSandboxService`）。运行时特定细节都留在服务边界之内。

### 3.3 服务端持久化

`[store]` 配置选择 SQLite（默认，位于 `~/.opensandbox/opensandbox.db`）或 PostgreSQL。服务端通过独立的仓储适配器（`repositories/snapshots/` 和 `repositories/templates/`）持久化快照元数据和 FastSandbox 模板目录。数据库中存储的模板目录是公共的事实来源；Kubernetes `SandboxTemplate` 资源驱动黄金镜像构建，并提供异步状态更新。

### 3.4 服务端代理与端点解析

生命周期端点 API 返回沙箱内某个服务端口可达的地址。根据运行时和配置，端点可能是：

- Docker 宿主机/网桥映射的端点。
- Kubernetes ingress 网关端点。
- 当 `use_server_proxy=true` 时，`/sandboxes/{sandboxId}/proxy/{port}` 下的服务端代理 URL。

基于路由头的 ingress 端点在其端点元数据中包含 `OpenSandbox-Ingress-To`。
当端点被改写为服务端代理 URL 时，服务端只移除这一个 ingress 路由头，
并保留其他必需的端点头。

服务端代理支持 HTTP 和 WebSocket 流量，并与可选的访问时续期行为集成。对于 HTTP 响应，它会剥离逐跳头部和后端 `Server` 头，同时保留来源的 `Date`；只有当响应中没有 `Date` 时，服务端才会添加当前的 `Date`。以单个 `/` 开头的根相对 `Location` 值会被重定位到同一沙箱代理路由之下，而绝对 URL、网络路径引用（`//host/path`）以及普通的相对路径值则原样转发。

HTTP 代理响应保留沙箱服务的状态码、响应体和 `Content-Type`，包括重定向和后端错误。生成的服务端 OpenAPI 对每种支持的 HTTP 方法都描述了 `200` 和 `default` 响应，媒体类型为 `*/*` 且没有固定的负载模式，因为负载由沙箱服务决定。服务端校验和认证仍在转发之前执行；显式的 `422` 校验响应也保留在文档中。这些响应声明同时覆盖根路径和 `/v1` 别名，包括带后端路径和不带后端路径的情况。

## 4. 运行时后端

### 4.1 Docker 运行时

Docker 运行时是本地和单机后端。它直接与 Docker 守护进程通信，管理容器、定时器、标签、卷、端口、可选 sidecar 和快照。

核心职责：

- 拉取公共或私有镜像，包括按请求的镜像仓库认证。
- 创建带有 CPU、内存、GPU、平台、能力、AppArmor、seccomp、PID 和安全运行时设置的容器。
- 在启动用户 entrypoint 之前，将 `[runtime].execd_image` 中的 `execd` 二进制文件暂存到沙箱中，并安装引导启动器。
- 支持网络模式 `host`、`bridge`、`none`、`pasta`、`slirp4netns` 以及自定义用户定义网络。用户态模式（`pasta`、`slirp4netns`）不暴露可从宿主机路由的容器 IP，只能通过发布的宿主机端口访问。
- 在非 host 网络模式下为 `execd` 和用户服务端点分配宿主机端口。
- 服务端重启后为已有的受管容器恢复过期定时器。
- 支持宿主机 bind mount、通过 `pvc` 卷模型映射的 Docker 命名卷，以及基于 OSSFS 的挂载。
- 当请求了 `networkPolicy` 且 Docker 网络兼容时，挂载出站 sidecar。
- 创建基于 Docker 的持久快照作为本地镜像，并从这些快照镜像恢复沙箱。

Docker 的暂停/恢复使用容器级 pause/resume。对于带有出站 sidecar 的沙箱，
暂停时会先冻结沙箱容器再冻结 sidecar，恢复时则先解冻 sidecar 再解冻沙箱容器。
Docker 快照通过公共快照 API 暴露。

### 4.2 Kubernetes 运行时

Kubernetes 运行时将实际的工作负载创建委托给由 `kubernetes.workload_provider` 选择的工作负载提供者。

支持的提供者：

- `batchsandbox` - 默认提供者，由 OpenSandbox 的 Kubernetes 控制器和 `BatchSandbox` CRD 支撑。
- `agent-sandbox` - 针对 `kubernetes-sigs/agent-sandbox` 的提供者。

Kubernetes 服务端路径负责：

- Kubernetes 客户端初始化和基于 informer 的读取。
- 从镜像请求创建工作负载；`snapshotId` 启动会在快照记录支持恢复时解析为已存储的可恢复镜像。
- 为 BatchSandbox 和 agent-sandbox 清单合并模板。
- 在提供者支持的情况下处理按请求的镜像拉取 Secret。
- 将资源限制和 GPU 翻译为 Kubernetes 扩展资源。
- 平台约束和 RuntimeClass 集成以支持安全运行时。
- 卷、出站 sidecar 以及安全端点访问注解。
- 通过直接的工作负载数据或 ingress 网关配置进行端点解析。
- 将暂停/恢复委托给提供者。
- 来自 Kubernetes 资源的纯文本诊断。

### 4.3 BatchSandbox 控制器

`kubernetes/` 下的 Kubernetes 控制器实现了 OpenSandbox 专属的 CRD，用于高吞吐和池化的沙箱交付：

- `BatchSandbox`：从一个 Pod 模板创建一个或多个沙箱副本。
- `Pool`：维护预热资源以实现快速分配。
- `SandboxSnapshot`：快照记录，供 BatchSandbox 的暂停/恢复以及公共快照工作流使用。

BatchSandbox 同时支持基于模板的创建和通过 `extensions.poolRef` 进行的基于池的创建。它还支持面向批处理和 RL 类工作负载的可选任务编排。

对于支持的单副本 BatchSandbox 工作负载，rootfs 暂停/恢复路径会将沙箱文件系统提交为一个 OCI 镜像并释放运行时资源；恢复时从该镜像重建工作负载并保留沙箱 ID。配置了 QEMU 快照契约的工作负载则使用 VM 状态检查点与恢复路径。

公共快照 API 支持 Docker 和兼容的 `BatchSandbox` 工作负载。`DockerSnapshotRuntime` 将容器提交为本地镜像，而 `KubernetesSnapshotRuntime` 创建一个公共的 `SandboxSnapshot` CR，用于提交镜像并推送到 OCI 镜像仓库。这与内部的暂停/恢复不同——后者在释放计算资源的同时保持沙箱连续性。公共快照并不支持所有 Kubernetes 提供者或 VM 运行时。

### 4.4 FastSandbox 集成

在 Kubernetes 模式下，`FastSandboxService` 将 OpenSandbox 连接到外部的 `fast-sandbox` 平台，以进行基于模板的 microVM 供给：

- **路由**：`CompositeSandboxService` 将携带 `templateId` 的创建请求路由到 `FastSandboxService`，并按 `fsb-` ID 前缀路由对已有沙箱的操作。FastSandbox 是 Kubernetes 组合的一部分，而不是一个独立的 `runtime.type` 值。
- **控制协议**：通过 gRPC FastPath v2（端口 9090）与外部 `fast-sandbox` 控制平面通信。
- **状态**：FastPath 负责变更操作和实时运行时操作；Kubernetes LIST/WATCH 读取持久化的 `sandbox.fast.io/v1alpha2` 资源及其观测结果。
- **暂停/恢复**：委托给 FastPath，由其在恢复时还原检查点。客户端必须在恢复后重新解析端点。
- **网络**：所配置的 Fastlet 池提供一个共享出站进程（每个 Fastlet Pod 一个）；适配器通过 FastPath Actions 发送网络策略。这与挂载每个沙箱一个的容器 sidecar 不同。当前服务端适配器在该路径上会拒绝 `credentialProxy.enabled` 创建标志和卷挂载。

适配器和端点集成位于本仓库；FastPath 和 microVM 运行时由外部平台提供。实现边界参见[服务分发](https://github.com/opensandbox-group/OpenSandbox/blob/main/server/opensandbox_server/services/composite_service.py)与 [FastSandbox 适配器](https://github.com/opensandbox-group/OpenSandbox/blob/main/server/opensandbox_server/services/fast_sandbox/service.py)，完整设计参见 [Fast Sandbox 集成](/architecture/fast-sandbox/)文档。

## 5. 沙箱数据平面

每个沙箱运行用户的镜像和 entrypoint，OpenSandbox 控制进程被注入其周围。

### 5.1 Execd

`components/execd/` 是一个用 Gin 构建的 Go 守护进程。它运行在沙箱内部，暴露执行 API。

职责：

- 支持 SSE 流式输出的 shell 命令执行。
- 后台命令状态与增量日志获取。
- 持久化的 bash 会话。
- 基于 WebSocket 的交互式 PTY 会话。
- 文件与目录操作。
- 用于在隔离环境中执行命令的进程隔离会话。
- 基于 Jupyter 的代码上下文与代码执行。
- 本地 CPU/内存指标和可选的 OpenTelemetry 指标导出。
- 通过 `X-EXECD-ACCESS-TOKEN` 实现的可选共享访问 token 校验。

在 Docker 中，服务端将 `execd` 暂存进容器并安装引导脚本。在 Kubernetes BatchSandbox 模板模式下，一个 init 容器将 `execd` 和 `bootstrap.sh` 从所配置的 `execd_image` 复制到主沙箱容器挂载的 `emptyDir` 卷中。

对于 FastSandbox，模板构建会在 microVM 启动之前将 `execd` 包含在 guest 镜像中。

### 5.2 代码解释器运行时

代码解释器沙箱镜像在沙箱内启动 Jupyter。`execd` 通过 HTTP/WebSocket 与 Jupyter 通信，并将 Jupyter kernel 消息转换为 OpenSandbox 流式事件。

代码解释器 SDK 是可选的高层客户端。更低层的执行 API 仍然可以通过沙箱 SDK 和直接的 `execd` 客户端使用。

### 5.3 卷

生命周期 API 暴露运行时无关的卷模型：

- `host`：绑定一个允许的宿主机路径。
- `pvc`：平台托管的命名存储。Docker 将其映射为 Docker 命名卷；Kubernetes 将其映射为 PersistentVolumeClaim。
- `ossfs`：通过服务端/运行时集成挂载阿里云 OSS。

运行时提供者以不同方式校验和落地这些卷定义，但 API 形态保持共享。卷挂载支持 Docker 和 Kubernetes 容器工作负载；FastSandbox 适配器当前会拒绝卷挂载。

### 5.4 出站 Sidecar

`components/egress/` 负责执行出站网络策略，并为沙箱提供凭据代理。

能力：

- FQDN 与通配符域名的允许/拒绝规则。
- `dns` 模式用于 DNS 过滤。
- `dns+nft` 模式用于 DNS 加上 nftables 对已解析 IP 以及 CIDR/IP 规则的强制执行（在支持的情况下）。
- Credential Vault / Proxy：透明 TLS MITM 代理（在 `dns+nft` 模式下激活），基于宿主机管理的密钥绑定为出站 HTTPS 请求提供限定范围的凭据注入，使原始密钥不进入沙箱工作负载。
- 通过 `/policy` 进行运行时策略查看与修改。
- 可选的 sidecar 认证。
- 可选的平台强制 always-allow 与 always-deny 覆盖层。

Docker 将出站 sidecar 作为独立容器启动，并让主沙箱容器运行在 sidecar 的网络命名空间中。Kubernetes 容器提供者将出站 sidecar 追加到 Pod spec 中，并从主沙箱容器移除 `NET_ADMIN`，从而只有 sidecar 能修改网络规则。FastSandbox 使用共享出站进程（每个 Fastlet Pod 一个，位于 microVM guest 之外），通过 FastPath 接收策略动作。

## 6. 网络与访问

### 6.1 Ingress

`components/ingress/` 是一个面向 Kubernetes 的 HTTP/WebSocket 反向代理。它监视沙箱资源并将流量路由到沙箱端口。

支持的路由模式：

- 头部模式：`OpenSandbox-Ingress-To: <sandbox-id>-<port>` 或主机名解析。
- URI 模式：`/<sandbox-id>/<port>/<path>`。
- 通过服务端端点格式化实现的通配符主机模式。

对于 `BatchSandbox`，ingress 从 `sandbox.opensandbox.io/endpoints` 注解读取端点数据。对于 `agent-sandbox`，它读取 `status.serviceFQDN`。

对于 FastSandbox 路由，ingress 请求 FastPath 解析端点，并将流量转发到返回的 Fastlet 或中央代理地址。

### 6.2 安全访问

`secureAccess` 目前支持通过 ingress 网关模式暴露的 Kubernetes 沙箱。启用后，服务端会供给端点凭据，并在端点响应中返回所需的头部。当配置了网关安全访问签名密钥时，还支持签名的路由 token。

### 6.3 访问时自动续期

可选的续期意图集成在观测到访问时延长沙箱 TTL。它可以由服务端代理请求触发，也可以由通过 Redis 送达的 ingress 网关事件触发。每个沙箱的选择加入由创建参数 `extensions["access.renew.extend.seconds"]` 控制。

## 7. 核心流程

### 7.1 沙箱创建

```text
Client / SDK / CLI / MCP
  -> POST /v1/sandboxes
  -> FastAPI lifecycle server validates request and config
  -> selected service creates a Docker container, Kubernetes workload, or FastSandbox microVM
  -> runtime stages execd or uses a prepared guest image, with backend-specific networking and storage
  -> sandbox reaches Running or reports Failed with status reason/message
```

从 API 视角看，创建是异步的。客户端应轮询 `GET /v1/sandboxes/{sandboxId}` 或使用 SDK 的就绪辅助工具。

### 7.2 命令、文件与代码执行

```text
Client
  -> resolve execd endpoint from sandbox metadata or server proxy
  -> call execd API with X-EXECD-ACCESS-TOKEN when required
  -> execd runs command, file operation, session, PTY, or Jupyter code execution
  -> execd streams SSE/WebSocket output or returns structured responses
```

### 7.3 服务暴露

```text
Client
  -> GET /v1/sandboxes/{sandboxId}/endpoints/{port}
  -> server returns Docker-mapped, ingress-gateway, or server-proxy endpoint
  -> client includes returned headers when secure access or sidecar auth requires them
  -> HTTP/WebSocket traffic reaches the target sandbox port
```

### 7.4 出站策略

```text
Create request with networkPolicy
  -> server validates runtime-specific network policy settings
  -> runtime attaches egress sidecar or binds policy actions to the shared Fastlet egress process
  -> sandbox outbound DNS/network traffic is filtered
  -> FastSandbox clients manage policy via GET/PUT/PATCH/DELETE /v1/sandboxes/{sandboxId}/networkpolicy
  -> container clients resolve the egress endpoint and PATCH /policy
```

### 7.5 暂停、恢复与快照

```text
Pause / resume
  -> lifecycle server delegates to runtime provider
  -> Docker pauses/resumes the sandbox container and its egress sidecar when present
  -> BatchSandbox uses rootfs commit/recreate or the configured QEMU VM-state restore path
  -> FastSandbox delegates pause/resume to external FastPath v2 service

Public snapshot API
  -> server persists snapshot metadata
  -> Docker runtime commits the sandbox to a local OCI image
  -> Kubernetes runtime (scoped to BatchSandbox) creates a public SandboxSnapshot CR that commits and pushes to an OCI registry
  -> create-from-snapshot resolves that image and starts a new sandbox
```

## 8. 设计原则

### 协议优先

公共行为从 `specs/` 中的 OpenAPI 契约出发。SDK 和客户端应对齐这些契约，生成的产物应从源规范重新生成，而不是把打补丁当作唯一的修复手段。

### 控制平面与数据平面分离

生命周期服务端负责编排和校验。平台特定的供给属于运行时服务/提供者。沙箱内操作属于 `execd` 和出站 sidecar。

### 运行时无关的 API，运行时特定的执行

生命周期 API 使用共享概念，例如资源限制、卷、端点、网络策略和元数据。Docker 和 Kubernetes 可以用不同方式落地这些概念，同时保持 API 契约不变。

### 安全默认值与显式逃生通道

服务端支持 API-key 认证、针对未认证模式的启动防护、资源限制、能力移除、可选安全运行时、出站控制、端点头部以及平台特定的网络隔离。限制较少的模式面向本地开发或运维人员的显式选择。

### 可观测的失败

沙箱状态包含 `state`、`reason`、`message` 和状态转换时间。`execd` 暴露指标，服务端暴露诊断，ingress/egress/execd 在已实现的地方支持日志和 OpenTelemetry 指标，并且请求 ID 会被传播以便调试。

## 9. 常见使用场景

- **编码智能体**：在隔离的沙箱中运行 Claude Code、Gemini CLI、Codex CLI、Qwen Code、Kimi CLI 或其他智能体工具。
- **AI 代码执行**：通过命令/文件/代码解释器 API 执行模型生成的代码，并获得流式反馈。
- **浏览器自动化**：运行 Chrome 或 Playwright，文件系统与网络行为可控。
- **远程开发**：通过沙箱端点暴露 VS Code Web、桌面、VNC 或开发服务器。
- **RL 与评测工作负载**：使用 Kubernetes BatchSandbox、Pool 和任务编排实现高吞吐沙箱交付。
- **企业级隔离**：组合安全运行时、ingress、egress、端点访问头部和 Kubernetes 部署控制。
