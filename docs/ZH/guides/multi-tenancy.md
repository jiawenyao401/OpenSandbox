---
title: 多租户
description: 跨 Kubernetes 命名空间隔离沙箱工作负载，提供每租户 API 密钥和自动命名空间路由。
---

# 多租户

多租户让单个 OpenSandbox Server 部署可以服务多个独立消费者，每个消费者隔离在自己的 Kubernetes 命名空间中。每个租户获得专用的 API 密钥，所有沙箱生命周期操作（创建、列出、获取、删除）自动限定在该租户的命名空间内。

## 前提条件

- `opensandbox-server` >= 0.2.2
- Kubernetes 运行时（`server.toml` 中 `runtime.type = "kubernetes"`）
- 每租户一个 Kubernetes 命名空间，**由集群管理员预先创建**
- 默认 Helm chart RBAC（已包含跨命名空间访问的 ClusterRole）

::: tip 多租户模式下的代理路由
与单租户模式不同，启用多租户后代理路由需要 `OPEN-SANDBOX-API-KEY`。确保客户端在代理请求中包含该头。
:::

::: warning 不支持 Docker
多租户要求 Kubernetes 命名空间隔离。如果配置了 `runtime.type = "docker"` 且启用了租户，服务器会以致命错误拒绝启动。
:::

## 工作原理

```
Request with OPEN-SANDBOX-API-KEY header
       │
       ├── Tenant provider configured?
       │       ├── YES → lookup key in tenant provider
       │       │       ├── found  → inject tenant context, route to tenant namespace
       │       │       └── not found → 401 Unauthorized
       │       └── NO  → validate against server.api_key (single-tenant legacy)
```

当多租户激活时：

1. 认证中间件通过配置的 provider 将 API 密钥解析为 `TenantEntry`。
2. 租户的命名空间被注入到请求作用域的 `ContextVar` 中。
3. 沙箱生命周期操作（创建/列出/获取/删除）使用解析出的命名空间。
4. 列出操作只返回经过认证的租户命名空间内的沙箱。
5. 代理路由（`/sandboxes/{id}/proxy/...`）在多租户模式下也需要 `OPEN-SANDBOX-API-KEY`。
6. 创建时在工作负载上标记 `opensandbox.io/tenant=<name>`。当多个租户共享一个命名空间时，list/get/delete/proxy 会隐藏其他租户带标签的沙箱（`404`）。无标签的遗留沙箱保持可见，直到它们过期。

## 配置

通过在 `server.toml` 中添加 `[tenants]` 部分启用多租户。有两种 provider 后端可用：**file**（本地 TOML）和 **http**（远程端点）。

### File Provider（默认）

添加到 `server.toml`：

```toml
[tenants]
provider = "file"
```

然后在以下路径之一创建 `tenants.toml`（按顺序检查）：

1. `$SANDBOX_TENANTS_CONFIG_PATH` 环境变量
2. `~/.opensandbox/tenants.toml`

`tenants.toml` 示例：

```toml
[[tenants]]
name = "team-alpha"
namespace = "sandbox-alpha"
api_keys = ["sk-alpha-key-1", "sk-alpha-key-2"]

[[tenants]]
name = "team-beta"
namespace = "sandbox-beta"
api_keys = ["sk-beta-key-1"]
```

每个租户条目需要：

| 字段 | 描述 |
|-------|-------------|
| `name` | 唯一的人类可读租户标识符 |
| `namespace` | 目标 Kubernetes 命名空间（服务器启动时必须已存在） |
| `api_keys` | 该租户的一个或多个 API 密钥（支持密钥轮换） |

约束：
- API 密钥在所有租户间必须唯一（重复 → 启动错误）。
- 每个租户必须至少有一个密钥。
- 使用 `[tenants]` 时必须移除 `server.toml` 中的 `server.api_key`。

### HTTP Provider

对于租户由外部 IAM 或租户管理系统管理的环境：

```toml
[tenants]
provider = "http"
endpoint = "https://iam.internal/opensandbox/tenant-lookup"
max_stale_seconds = 300.0
timeout_seconds = 5.0
auth_header = "X-Internal-Auth"
auth_token = "bearer-token-here"
```

| 字段 | 默认值 | 描述 |
|-------|---------|-------------|
| `provider` | `"file"` | `"file"` 或 `"http"` |
| `endpoint` | — | HTTP 必填。服务器 GET 请求以解析密钥的 URL。 |
| `max_stale_seconds` | `300.0` | 端点不可达时提供陈旧缓存的最长秒数。 |
| `timeout_seconds` | `5.0` | HTTP 请求超时。 |
| `auth_header` | — | 可选的 provider 级认证头名称。 |
| `auth_token` | — | 该头的可选 token 值。 |

**HTTP 端点契约：**

请求：
```
GET {endpoint}
Header: OPEN-SANDBOX-API-KEY: <api_key>
```

响应 (200)：
```json
{
  "namespace": "sandbox-alpha",
  "ttl": 60
}
```

响应 (401)：
```json
{
  "code": "UNAUTHORIZED",
  "message": "Invalid API key"
}
```

HTTP provider 使用服务器建议的 `ttl` 按密钥缓存结果。TTL 过期时同步重新获取。如果端点不可达，陈旧条目最多提供 `max_stale_seconds`，之后请求以 503 失败。

::: warning HTTP provider 跳过启动时命名空间验证
HTTP provider 按 API 密钥解析租户，无法在启动时枚举所有租户，因此会跳过它的 fail-fast 命名空间检查（改为记录警告）。file provider 在启动时加载完整的 `tenants.toml`，当任何租户命名空间缺失或不可访问时仍然快速失败。使用 HTTP provider 时，在签发租户 API 密钥之前请确保命名空间存在且可访问。
:::

## 命名空间设置

在接入租户之前，集群管理员必须准备目标命名空间。

### 1. 创建命名空间

```bash
kubectl create namespace sandbox-alpha
```

### 2. ResourceQuota（推荐）

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  namespace: sandbox-alpha
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    pods: "20"
```

#### 配额耗尽行为

当命名空间配额耗尽时，`POST /v1/sandboxes` 会快速失败并返回 `403 KUBERNETES::QUOTA_EXCEEDED`，而不是等待沙箱创建超时并返回 `KUBERNETES::POD_READY_TIMEOUT`：

- **数量配额**（`count/sandboxes.agents.x-k8s.io`）：Sandbox 创建立即被 Kubernetes 准入拒绝。
- **计算配额**（`requests.cpu`、`requests.memory`、`pods`）：Sandbox 被创建，但其 Pod 无法被接纳；控制器记录后（通常在一秒内），服务器会从工作负载状态（`ReconcilerError`）呈现该拒绝。

响应体使用标准的 `ErrorResponse` 形状，`message` 携带 Kubernetes 准入详情，例如 `exceeded quota: tenant-quota, requested: requests.cpu=1, used: requests.cpu=8, limited: requests.cpu=8`。SDK 客户端将其呈现为 HTTP 状态 403 的类型化错误。提高配额或释放容量，然后重试 —— 无需重启服务器。

此快速失败契约目前适用于 `agent-sandbox` 工作负载提供程序（`[kubernetes] workload_provider = "agent-sandbox"`）。使用默认的 `batchsandbox` provider 时，控制器尚未将准入拒绝传播到 CR 状态，因此配额耗尽仍会等待沙箱创建超时并返回 `KUBERNETES::POD_READY_TIMEOUT`。

### 3. LimitRange（推荐）

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-limits
  namespace: sandbox-alpha
spec:
  limits:
  - default:
      cpu: "1"
      memory: 2Gi
    defaultRequest:
      cpu: 250m
      memory: 512Mi
    type: Container
```

`Container` 默认值适用于每个省略相应资源配置的容器，包括 OpenSandbox egress sidecar。在生命周期服务器中为 sidecar 配置更小的专用值，使 egress 容器不会继承为沙箱工作负载设计的默认值。

### 4. Egress sidecar 资源（推荐）

在服务器的 `[egress]` 配置中添加 Kubernetes 资源设置：

```toml
[egress]
image = "opensandbox/egress:v1.1.7"
requests = { cpu = "25m", memory = "64Mi" }
limits = { cpu = "250m", memory = "256Mi" }
```

Requests 和 limits 可以独立配置。这些值是基本 DNS/nft 强制执行的起点；请对你的工作负载进行基准测试，并在使用 Credential Vault 或透明 mitmproxy 时留出更多余量。如果你的 `LimitRange` 定义了最小值，egress 设置必须满足它们。

### 5. 服务器 RBAC

默认 Helm chart（`opensandbox-server`）已经部署了具有跨命名空间访问能力的 ClusterRole + ClusterRoleBinding。多租户不需要额外的 RBAC 更改 —— 服务器的 ServiceAccount 已经可以在任何命名空间中操作。

## 热重载

### File provider

file provider 会监视 `tenants.toml` 的更改（每 2 秒轮询一次）。更改无需重启服务器即生效：

- **新增密钥** → 下次请求立即生效。
- **移除密钥** → 立即失效（401）。
- **重载期间解析错误** → 保留先前配置，记录警告。
- **文件被删除** → 所有租户密钥失效。

监视器会监控父目录，以处理 Kubernetes ConfigMap 的原子符号链接切换。

### HTTP provider

无需显式重载 —— 缓存条目按其 `ttl` 过期，并在下次查找时重新获取。

## 密钥轮换

多租户支持每个租户多个 API 密钥，专门用于零停机密钥轮换：

1. 将新密钥添加到 `tenants.toml`（或通过 HTTP provider 签发）。
2. 等待热重载（file）或 TTL 过期（HTTP）。
3. 更新客户端使用新密钥。
4. 从配置中移除旧密钥。

过渡窗口内新旧密钥同时有效。

## 从单租户迁移

1. 为你的租户创建目标命名空间。
2. 编写 `tenants.toml`，将现有 API 密钥作为租户条目，指向现有命名空间：
   ```toml
   [[tenants]]
   name = "default"
   namespace = "your-existing-namespace"
   api_keys = ["your-existing-api-key"]
   ```
3. 在 `server.toml` 中添加 `[tenants]` 部分：
   ```toml
   [tenants]
   provider = "file"
   ```
4. 从 `server.toml` 的 `[server]` 中移除 `api_key`。
5. 部署。现有密钥继续工作，现在作为租户密钥。
6. 按需添加更多租户。

**回滚：** 从 `server.toml` 中移除 `[tenants]` 部分，恢复 `server.api_key`，重启。

## 隔离模型

服务器本身不强制资源配额或网络策略。隔离委托给 Kubernetes：

| 维度 | Kubernetes 机制 | 范围 |
|-----------|---------------------|-------|
| 资源配额 | `ResourceQuota` | 每命名空间 CPU、内存、存储 |
| 默认限制 | `LimitRange` | 每命名空间默认容器资源 |
| 网络隔离 | Egress sidecar | 通过 egress 代理的每沙箱出站策略 |
| 沙箱数量 | `ResourceQuota`（Pod 数量） | 每命名空间 Pod 限制 |
| RBAC | `RoleBinding` | 每命名空间 API 访问 |

::: info Pool API 不按租户隔离
池管理路由（`/pools`）目前在服务器配置的默认命名空间中操作，而不是经过认证的租户命名空间。池是共享资源。如果需要每租户池隔离，请创建单独的服务器部署，或等待未来的每租户池支持。
:::

## 故障排查

| 症状 | 原因 | 修复 |
|---------|-------|-----|
| 服务器启动时退出："Multi-tenancy requires Kubernetes namespaces" | 配置了 `[tenants]` 且 `runtime.type = "docker"` | 使用 Kubernetes 运行时或移除 `[tenants]` |
| 服务器退出："Remove server.api_key from server.toml" | `server.api_key` 和 `[tenants]` 同时存在 | 从 `[server]` 部分移除 `api_key` |
| 服务器退出：找不到命名空间 | 租户命名空间不存在 | 启动服务器前 `kubectl create namespace <ns>` |
| 配置更改后有效密钥返回 401 | 热重载尚未获取更改 | 等待 2 秒（file）或 TTL 秒数（HTTP） |
| 503 TENANT_PROVIDER_UNAVAILABLE | HTTP 端点不可达且缓存过期超过 `max_stale_seconds` | 修复端点连接；增加 `max_stale_seconds` |
| 启动时重复 api_key 错误 | 同一密钥分配给多个租户 | 确保每个密钥在所有租户间唯一 |
