---
title: API 规范
description: 定义 OpenSandbox 完整 API 接口与数据模型的 OpenAPI 规范文档。
---

# OpenSandbox API 规范

本节包含 OpenSandbox 项目的 OpenAPI 规范文档，定义了完整的 API 接口与数据模型。在构造请求时，请使用各规范中定义的服务器基础 URL（例如 lifecycle API 使用 `http://localhost:8080/v1`，execd 使用 `http://localhost:44772`，egress 使用 `http://localhost:18080`）。

## 规范文件

打算实现替代后端，或将工作负载自身的 API 整合进来？请阅读 [实现兼容性](/api/implementation-compatibility)，了解这些契约之间的边界、参考守护进程以及当前 SDK 的预期行为。

### 1. sandbox-lifecycle.yml

[OpenAPI 源文件](https://github.com/opensandbox-group/OpenSandbox/blob/main/specs/sandbox-lifecycle.yml)

**Sandbox 生命周期管理 API**

定义了从容器镜像或快照创建、管理和销毁 sandbox 环境的完整生命周期接口。

**核心特性：**
- **Sandbox 管理**：创建、列出、查询和删除 sandbox 实例，支持元数据过滤与分页
- **状态控制**：暂停和恢复 sandbox 执行
- **生命周期状态**：支持 Pending -> Running -> Pausing -> Paused -> Stopping -> Terminated 之间的状态流转，并通过 `Failed` 进行错误处理
- **资源与运行时配置**：指定资源限制和可选的 Kubernetes `resourceRequests`、镜像启动 `entrypoint`、`platform`、生命周期钩子、可选的 `secureAccess`、卷、环境变量以及不透明的 `extensions`
- **镜像支持**：从公共或私有镜像仓库创建 sandbox，包括镜像仓库认证
- **超时管理**：创建时可选择设置 `timeout`（省略或设为 `null` 可禁用自动过期），并可通过 API 显式续期
- **端点访问**：检索 sandbox 内运行服务的公共访问端点，在启用安全访问时包含所需的请求头；端点查询会通过 `OPEN-SANDBOX-ORIGIN` 响应头报告 sandbox 来源（fsb golden-image sandbox 使用 `template`）
- **模板管理**：创建、列出、检查和删除 fsb golden-image 模板；模板构建是异步的（轮询直至 `Succeeded`）
- **快照管理**：从 sandbox 创建快照，按来源/名称过滤列出快照，以及删除快照

**主要端点（基础路径 `/v1`）：**
- `POST /sandboxes` - 从镜像、快照或模板创建 sandbox，支持超时和资源限制
- `GET /sandboxes` - 列出 sandbox，支持状态/元数据过滤与分页
- `GET /sandboxes/{sandboxId}` - 获取 sandbox 完整详情（包括启动来源和 entrypoint）
- `DELETE /sandboxes/{sandboxId}` - 删除 sandbox
- `POST /sandboxes/{sandboxId}/snapshots` - 从 sandbox 创建快照
- `GET /snapshots` - 列出快照，支持按来源 sandbox、精确名称和状态过滤，以及分页
- `GET /snapshots/{snapshotId}` - 获取快照状态与元数据
- `DELETE /snapshots/{snapshotId}` - 删除快照
- `POST /sandboxes/{sandboxId}/pause` - 暂停 sandbox（异步）；如果已处于暂停状态则返回 `409 Conflict`
- `POST /sandboxes/{sandboxId}/resume` - 恢复已暂停的 sandbox
- `POST /sandboxes/{sandboxId}/renew-expiration` - 设置 sandbox 的绝对过期时间
- `PATCH /sandboxes/{sandboxId}/metadata` - 修改 sandbox 元数据（JSON Merge Patch，RFC 7396）
- `GET /sandboxes/{sandboxId}/endpoints/{port}` - 获取访问端点及所需请求头；支持 `use_server_proxy` 和签名端点 `expires` 参数
- `GET/PUT/PATCH/DELETE /sandboxes/{sandboxId}/networkpolicy` - 检查并管理 sandbox 的出口网络策略（Fsb 将策略意图持久化到 Sandbox CR；其他后端则代理 sandbox 侧的出口服务）
- `POST /templates` - 创建 fsb 模板（异步 golden-image 构建）
- `GET /templates` - 列出模板，支持元数据过滤与分页
- `GET /templates/{templateId}` - 获取模板状态和制品引用
- `DELETE /templates/{templateId}` - 删除模板

**过期时间更新：**
`renew-expiration` 将 `expiresAt` 设置为所提供的未来时间戳，该时间戳可以早于、等于或晚于当前过期时间。配置的 `server.max_sandbox_timeout_seconds` 仅作用于创建时的 `timeout`，不会限制过期时间更新。参见 [Sandbox 过期时间](/getting-started/configuration#sandbox-expiration)。

**可选的 `Sandbox.allocation` 响应字段：**
- 仅当运行时确认了 sandbox 当前具体的 Pool 分配时才会返回。
- 对于未确认的分配、非 Pool 的 sandbox 以及正在释放的分配，该字段会被省略。
- 此字段不是请求回显、分配历史或就绪信号，也不会暴露 Pod 名称或其他 Kubernetes 内部字段。

**认证：**
- HTTP 请求头：`OPEN-SANDBOX-API-KEY: your-api-key`
- 环境变量：`OPEN_SANDBOX_API_KEY`（供 SDK 客户端使用）

### 2. diagnostic-api.yml {#diagnostics}

[OpenAPI 源文件](https://github.com/opensandbox-group/OpenSandbox/blob/main/specs/diagnostic-api.yml)

**Sandbox 诊断 API**

定义了用于 sandbox 诊断日志和事件尽力而为的故障排查描述符。这些描述符要么以纯文本形式内嵌诊断内容，要么返回内容的下载 URL。本规范不定义结构化的审计或可观测性模型。

**主要端点（基础路径 `/v1`）：**
- `GET /sandboxes/{sandboxId}/diagnostics/logs` - 检索诊断日志内容描述符；`scope` 为必填
- `GET /sandboxes/{sandboxId}/diagnostics/events` - 检索诊断事件内容描述符；`scope` 为必填

**认证：**
- HTTP 请求头：`OPEN-SANDBOX-API-KEY: your-api-key`
- 环境变量：`OPEN_SANDBOX_API_KEY`（供 SDK 客户端使用）

`scope` 为必填项。Docker 和 Kubernetes 对日志支持 `container`/`all`，对事件支持 `runtime`/`all`。Fast Sandbox 支持事件，但对日志收集尚未实现。使用不受支持的 scope 将返回 `DIAGNOSTICS_SCOPE_UNSUPPORTED`。

响应使用带 `content` 的 `delivery: inline`，或带 `contentUrl` 及可选过期时间的 `delivery: url`。SDK 不会下载 URL 内容；在将结果视为完整结果之前，请先检查 `truncated` 和 `warnings`。Python 模型和 CLI 的 JSON/YAML 输出使用 snake_case 字段，例如 `content_url`。CLI 的原始输出会直接打印内容或 URL，而不会访问该 URL。

关于各语言支持情况，参见 [SDK 诊断](/sdks/#diagnostics)；命令示例参见 [CLI 诊断](/cli/#collect-diagnostics)。

### 3. execd-api.yaml

[OpenAPI 源文件](https://github.com/opensandbox-group/OpenSandbox/blob/main/specs/execd-api.yaml)

**Sandbox 内代码执行 API**

定义了在 sandbox 环境中执行代码、命令和文件操作的接口，提供完整的代码解释器和文件系统管理能力。请转发由 lifecycle 端点解析返回的请求头，必要时包括 `X-EXECD-ACCESS-TOKEN`。不要硬编码 execd 地址，也不要假设仅凭 lifecycle API key 就能授权直接访问 sandbox。

**核心特性：**
- **代码执行**：支持 Python、JavaScript 等多种语言的有状态代码执行，并带有上下文生命周期管理
- **命令执行**：执行 shell 命令，支持前台/后台模式，并提供用于查询状态/输出的轮询端点
- **文件操作**：对文件和目录的完整 CRUD 操作
- **实时流式输出**：通过 SSE（Server-Sent Events）实现实时输出流
- **系统监控**：实时监控 CPU 和内存等指标
- **访问控制**：通过 `X-EXECD-ACCESS-TOKEN` 进行基于令牌的 API 认证

**主要端点分类：**

**健康检查：**
- `GET /ping` - 服务健康检查

**代码解释器：**
- `GET /code/contexts` - 列出活跃的代码执行上下文（可按语言过滤）
- `DELETE /code/contexts` - 删除某种语言的所有上下文
- `DELETE /code/contexts/{context_id}` - 删除特定上下文
- `POST /code/context` - 创建代码执行上下文
- `POST /code` - 在上下文中执行代码（流式输出）
- `DELETE /code` - 中断代码执行

**命令执行：**
- `POST /command` - 执行 shell 命令（流式输出）
- `DELETE /command` - 中断命令执行
- `GET /command/status/{id}` - 获取前台/后台命令状态
- `GET /command/{id}/logs` - 获取后台命令累积的 stdout/stderr

**Bash 会话：**
- `POST /session` - 创建 bash 会话
- `POST /session/{sessionId}/run` - 在 bash 会话中运行命令（流式输出）
- `DELETE /session/{sessionId}` - 删除 bash 会话

**文件系统：**
- `GET /files/info` - 获取文件的元数据
- `DELETE /files` - 删除文件（不含目录）
- `POST /files/permissions` - 修改文件权限
- `POST /files/mv` - 移动/重命名文件
- `GET /files/search` - 搜索文件（支持 glob 模式）
- `POST /files/replace` - 批量替换文件内容
- `POST /files/upload` - 上传文件（multipart）
- `GET /files/download` - 下载文件（支持范围请求）

**目录操作：**
- `GET /directories/list` - 列出目录内容，可选深度控制
- `POST /directories` - 创建目录及权限（mkdir -p 语义）
- `DELETE /directories` - 递归删除目录

**系统指标：**
- `GET /metrics` - 获取系统资源指标
- `GET /metrics/watch` - 实时监视系统指标（SSE 流）

**隔离执行（基础路径 `/v1/isolated`）：**
- `POST /session` - 创建隔离 bash 会话
- `GET /sessions` - 列出隔离会话
- `GET /capabilities` - 获取隔离器能力
- `GET /session/{sessionId}` - 获取隔离会话状态
- `DELETE /session/{sessionId}` - 删除隔离会话
- `POST /session/{sessionId}/run` - 在隔离会话中运行命令（前台 SSE 或后台执行）
- `GET/DELETE /session/{sessionId}/runs/{runId}` - 获取隔离运行的状态或中断运行
- `GET /session/{sessionId}/runs/{runId}/logs` - 检索隔离运行的日志
- `GET /session/{sessionId}/files/info` - 获取文件信息
- `GET /session/{sessionId}/files/download` - 下载文件
- `POST /session/{sessionId}/files/upload` - 上传文件
- `DELETE /session/{sessionId}/files` - 移除文件
- `POST /session/{sessionId}/files/mv` - 重命名或移动文件
- `POST /session/{sessionId}/files/permissions` - 修改文件权限
- `POST /session/{sessionId}/files/replace` - 替换文件内容
- `GET /session/{sessionId}/files/search` - 搜索文件
- `GET /session/{sessionId}/directories/list` - 列出目录内容
- `POST /session/{sessionId}/directories` - 创建目录
- `DELETE /session/{sessionId}/directories` - 删除目录

`GET /session/{sessionId}/diff` 和 `POST /session/{sessionId}/commit` 出现在契约中，但当前 execd 并未实现：两者均返回 `503` 及不支持错误，且能力报告为 `diff_supported: false` 和 `commit_supported: false`。在使用依赖运行时的隔离功能之前，请先检查 `/v1/isolated/capabilities`。

### 4. egress-api.yaml

[OpenAPI 源文件](https://github.com/opensandbox-group/OpenSandbox/blob/main/specs/egress-api.yaml)

**Sandbox 出口运行时 API**

定义由 sandbox 内部 [egress sidecar](/architecture/network/egress) 直接暴露的运行时出口策略接口。与 lifecycle 操作不同，此 API 需要先解析出口端口对应的 sandbox 端点，然后直接调用 sidecar 端点。

**核心特性：**
- **策略检查**：检索当前生效的出口策略及派生的运行时模式
- **策略变更**：在运行时使用 sidecar 合并语义修改出口规则
- **直接访问 Sidecar**：通过 sandbox 端点解析访问，而非服务端 lifecycle 转发
- **可选 Sidecar 认证**：当 egress sidecar 需要认证时，支持端点专属的请求头

**主要端点：**
- `GET /policy` - 获取当前出口策略
- `PATCH /policy` - 将新的出口规则合并到当前策略
- `DELETE /policy` - 按 target 从当前策略中移除特定出口规则
- `POST/GET/PATCH/DELETE /credential-vault` - 创建、检查、修改或删除 vault 状态
- `GET /credential-vault/credentials` 与 `GET /credential-vault/credentials/{credential_name}` - 读取脱敏后的凭据元数据
- `GET /credential-vault/bindings` 与 `GET /credential-vault/bindings/{binding_name}` - 读取绑定元数据

凭据值是只写的。在创建 vault 之前，需先启用 Credential Proxy 和出口策略。基于模板的 sandbox 没有 egress sidecar 和 Credential Vault；其策略操作使用 lifecycle 的 `networkpolicy` 端点。参见 [Credential Vault](/guides/credential-vault)。

## 技术特性

### 流式输出（Server-Sent Events）

代码执行和命令执行接口使用 SSE 进行实时流式输出，支持以下事件类型：
- `init` - 初始化事件
- `status` - 状态更新
- `stdout` / `stderr` - 标准输出/错误流
- `result` - 执行结果
- `execution_complete` - 执行完成
- `execution_count` - 执行计数
- `error` - 错误信息

### 资源限制

支持灵活的资源配置（类似于 Kubernetes）：
```json
{
  "cpu": "500m",
  "memory": "512Mi",
  "gpu": "1"
}
```

### 文件权限

支持 Unix 风格的文件权限管理：
- 所有者（Owner）
- 所属组（Group）
- 权限模式值，如 644 或 755
