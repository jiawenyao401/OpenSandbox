---
title: 实现兼容性
description: 已发布的 OpenSandbox 契约、参考实现边界，以及面向独立实现的当前 SDK 预期。
---

# 实现兼容性

OpenSandbox 将 sandbox 生命周期管理、sandbox 内执行以及工作负载专属服务相互分离。本页描述当前仓库的接口与客户端行为，并不定义新的 v1 一致性配置（conformance profile）或能力协商协议。请固定你针对的规范修订版本和 SDK 版本：某一个 API 的实现并不自动兼容 sandbox SDK 暴露的每一项特性。

## 契约与流程

[已发布的 OpenAPI 文件](/api/)是实现的起点：

| 契约 | 接口与使用者 |
| --- | --- |
| [`sandbox-lifecycle.yml`](https://github.com/opensandbox-group/OpenSandbox/blob/main/specs/sandbox-lifecycle.yml) | 控制平面：创建、检查、删除、过期、端点解析、暂停/恢复、快照、模板以及生命周期网络策略操作。供 sandbox SDK 和管理客户端使用。 |
| [`diagnostic-api.yml`](https://github.com/opensandbox-group/OpenSandbox/blob/main/specs/diagnostic-api.yml) | lifecycle 服务器上用于诊断日志/事件描述符的附加路由。当客户端使用诊断功能时需要；并非命令输出 API。 |
| [`execd-api.yaml`](https://github.com/opensandbox-group/OpenSandbox/blob/main/specs/execd-api.yaml) | 数据平面：健康检查、命令、文件、目录、代码上下文、指标以及隔离会话操作。供 sandbox SDK 中的执行服务和 Code Interpreter 客户端使用。 |
| [`egress-api.yaml`](https://github.com/opensandbox-group/OpenSandbox/blob/main/specs/egress-api.yaml) | 出口策略和 Credential Vault 管理。这是一个独立于 execd 的 API；参考部署通常通过一个 egress 组件提供服务。 |

线上契约并不要求独立实现运行 Go 编写的 `execd` 二进制文件。原生执行服务可以提供其客户端所需的 execd 操作。兼容性不仅要求路由名称一致，还要求行为一致：请求/响应 schema、认证请求头、文件传输语义、SSE 事件载荷与完成信号、错误以及上下文/命令生命周期都至关重要。仅实现 lifecycle 契约并不能提供 SDK 的命令、文件或代码执行能力；反过来，仅有一个 execd 兼容服务也无法实现 sandbox 创建或端点解析。

参考服务器的二进制注入、容器引导（bootstrap）、init/监督、Jupyter 集成以及代理布局都属于实现机制。后端可以使用不同的机制，但必须保留其声称支持的各项操作的可观测语义。OpenAPI 覆盖范围并非覆盖参考实现的每个扩展：例如，当前的 [execd 路由器](https://github.com/opensandbox-group/OpenSandbox/blob/main/components/execd/pkg/web/router.go)暴露了 `/pty` 的 HTTP/WebSocket 路由，但尚未纳入 `execd-api.yaml`。不要仅凭 OpenAPI 兼容性就推断支持这些扩展。

## 工作负载 API 与端点解析

`GET /v1/sandboxes/{sandboxId}/endpoints/{port}` 解析对请求的 sandbox 端口上服务的访问。它不会将该服务转换为 execd API，也不会把 execd 的命令/文件 schema 强加给它。在 [AIO 示例](https://github.com/opensandbox-group/OpenSandbox/blob/main/examples/aio-sandbox/main.py)中，OpenSandbox 管理 sandbox 并解析端口 `8080`，然后由 `AioSandboxClient` 使用镜像自身的 shell/文件/浏览器 API。

端点解析对工作负载的应用层 API 是无关的；传输支持仍取决于所选择的网络路径。例如，[Docker 内嵌代理](/architecture/network/single-host-network)转发 HTTP、SSE 和 WebSocket 流量。任意端口号并不意味着存在通用的 TCP/UDP 隧道。请保留返回的地址（包括代理路径）和所需请求头，并使用适合该服务和传输方式的客户端。经过 execd 的代理转发并不意味着目标进程就是 execd。

## 当前 Python SDK 的端点预期

以下描述的是当前代码树中的 [`SandboxSync`](https://github.com/opensandbox-group/OpenSandbox/blob/main/sdks/sandbox/python/src/opensandbox/sync/sandbox.py)，而非对所有客户端都适用的可移植协议要求：

| 流程 | 健康检查之前的端点解析 |
| --- | --- |
| 从镜像或快照执行 `create()` | 解析 execd 端口 `44772`，然后解析出口端口 `18080`。如果 execd 端点报告来源为模板，策略操作将使用 lifecycle 服务，但此创建路径已经解析了出口端点。 |
| `create_from_template()` | 解析 `44772`；使用 lifecycle 网络策略操作，不解析 `18080`。 |
| `connect()` | 解析 `44772` 并读取端点响应中的 `OPEN-SANDBOX-ORIGIN` 请求头。来源为模板时选择 lifecycle 网络策略；其他来源还会解析 `18080`。 |

自定义 `health_check` 会替换默认的 execd `/ping` 就绪探针，但**不会**移除这些端点查询或 SDK 服务适配器的构建。`skip_health_check=True` 也仍会等待端点发布。这解释了为什么 AIO 示例中的自定义健康检查并没有使其变成仅使用 lifecycle 的 SDK 客户端。解析端点与调用其 API 是两回事：服务适配器在应用选择操作之前就已构建完成。

针对这些高层流程的独立后端必须把这些查询、返回的请求头以及它将要暴露的操作都纳入考量。仅使用 lifecycle HTTP API 的客户端和工作负载专属客户端可以选择更窄的集成范围。其他 SDK 语言和版本需单独验证；该表并不承诺所有 SDK 都具有相同的启动行为。模板来源是一个既有的路由信号，并非通用的能力清单。

## 依赖后端的特性与不支持的操作

某个规范或 SDK 中存在某个操作，并不意味着每个运行时、镜像或部署都启用了它：

| 特性 | 当前边界 |
| --- | --- |
| 暂停/恢复与快照 | 依赖于工作负载提供方、运行时和存储配置。特别是，Kubernetes rootfs 快照提交器会拒绝 gVisor RuntimeClass。不要从基础的创建/执行支持推断快照支持。 |
| 模板 | 模板管理使用 Kubernetes Fast Sandbox 集成；参考 Docker 服务器会以 HTTP `501` 拒绝该操作。 |
| 出口策略与 Credential Vault | 依赖于出口配置和后端路由。基于模板的 Python sandbox 通过 lifecycle 服务管理策略；对该来源进行 `credential_vault` 访问会抛出 SDK 异常。支持策略并不代表支持 Vault。 |
| 代码执行 | 参考 execd 使用 Jupyter 和已安装的内核来提供代码上下文。仅有 shell 的镜像因此并不支持 notebook 风格的代码执行。原生实现若声称支持该特性，必须复现代码 API 的语义。 |
| PTY 与隔离会话 | 依赖于平台设施和参考扩展。`/v1/isolated/capabilities` 描述的是隔离器/加固环境，并非全部 lifecycle 和执行特性。隔离 diff/commit 当前报告为不支持且尚未实现。 |

当前契约中不存在单一的通用能力协商端点。请记录后端支持的操作及其前置条件，并在可用时使用特定于特性的发现机制。不支持请求应产生带可操作错误信息的显式失败，而不是伪造的成功。请使用该操作规定的错误形态，并将不支持的功能与缺少授权、缺少资源或服务暂时未就绪区分开来。

同样，也不存在可以到处替代的通用"不支持操作"状态码。当前的参考示例包括：不支持的模板管理使用 HTTP `501`，快照服务的不支持运行时预检使用 HTTP `409`，未实现的隔离 diff/commit 使用 HTTP `503`。参见 [模板处理器](https://github.com/opensandbox-group/OpenSandbox/blob/main/server/opensandbox_server/api/templates.py)、[快照服务](https://github.com/opensandbox-group/OpenSandbox/blob/main/server/opensandbox_server/services/snapshot_service.py)以及 [隔离 API 说明](/api/)。这些描述的是当前行为，并非新的统一错误约定。

## 选择初始兼容性目标

从一个明确的客户端版本和一组明确的操作集开始。对于命令和文件集成，请验证 lifecycle 的创建/检查/删除与端点解析、认证与就绪、命令 SSE 完成与错误，以及客户端所需的文件操作。只有当后端能够提供相应语义时，才添加代码上下文、快照或策略/Vault 操作。使用仓库中针对所选操作的 [SDK 端到端测试](https://github.com/opensandbox-group/OpenSandbox/tree/main/tests)，并记录未实现的特性。这是一个务实的集成目标，并非对完整协议一致性的声明。
