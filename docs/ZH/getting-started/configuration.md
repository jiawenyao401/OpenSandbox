---
title: 配置
description: OpenSandbox 服务端配置参考，涵盖 TOML 配置、运行时、网络与认证。
---

# 配置

OpenSandbox 服务端读取 **TOML** 配置文件，默认路径为 `~/.sandbox.toml`。可通过环境变量 `SANDBOX_CONFIG_PATH` 或 `--config` 参数覆盖。

## 生成配置文件

```bash
# Docker 运行时（默认）
opensandbox-server init-config ~/.sandbox.toml --example docker

# Kubernetes 运行时
opensandbox-server init-config ~/.sandbox.toml --example k8s

# 仅包含 schema 的骨架（无默认值）
opensandbox-server init-config ~/.sandbox.toml

# 覆盖已有配置
opensandbox-server init-config ~/.sandbox.toml --example docker --force
```

## 运行服务端

```bash
opensandbox-server
# 或指定配置路径
opensandbox-server --config /path/to/sandbox.toml
```

服务端监听的地址来自 TOML 配置中的 `server.host` / `server.port`。

## 认证

设置 `server.api_key` 后即启用认证。除 `/health`、`/version`、`/docs`、`/redoc` 和 `/openapi.json` 外，所有 API 端点都要求携带 `OPEN-SANDBOX-API-KEY` 请求头：

```bash
curl -H "OPEN-SANDBOX-API-KEY: your-secret-api-key" http://localhost:8080/v1/sandboxes
```

::: warning
如果 `server.api_key` 为空，服务端将在无认证模式下运行。在非交互式环境（Docker/Kubernetes/CI）中，需设置 `OPENSANDBOX_INSECURE_SERVER=YES` 以确认已知悉此风险。**生产环境务必设置 API Key。**
:::

## 主要配置区域

| 配置段 | 说明 |
|---------|-------------|
| `[server]` | 主机、端口、API Key 及服务端通用设置 |
| `[runtime]` | 运行时选择与 execd 镜像配置 |
| `[otel]` | 用于 SDK 创建延迟指标的 OpenTelemetry 导出 |
| `[proxy]` | 服务端沙箱代理目标选择；当服务端无法路由到沙箱 IP 时使用主机映射 |
| `[docker]` | Docker 运行时的网络、主机地址与镜像拉取设置 |
| `[kubernetes]` | Kubernetes 运行时：`workload_provider`、`batchsandbox_template_file` |
| `[egress]` | 用于实施 `networkPolicy` 的 egress sidecar |
| `[ingress]` | Ingress 网关配置 |
| `[secure_runtime]` | 安全容器运行时（gVisor、Kata、Firecracker） |
| `[store]` | 持久化后端（默认：位于 `~/.opensandbox/opensandbox.db` 的 SQLite） |
| `[renew_intent]` | 访问时自动续期（实验性） |
| `[agent_sandbox]` | Kubernetes 下的 Agent Sandbox 设置 |

包含所有键与默认值的完整配置参考，请参见 [server configuration.md](https://github.com/opensandbox-group/OpenSandbox/blob/main/server/configuration.md)。

## 沙箱过期

`server.max_sandbox_timeout_seconds` 用于限制创建沙箱时传入的 `timeout`。若已配置，其值必须至少为 `60` 秒；省略该项则不启用创建时的上限。它不会限制续期请求，也不会限制沙箱的总存活时间。

`POST /v1/sandboxes/{sandboxId}/renew-expiration` 用于设置一个绝对的 `expiresAt`。尽管端点名称如此，只要请求的时间戳仍在未来，它就可以缩短、保持或延长当前过期时间。超过 `now + max_sandbox_timeout_seconds` 的时间戳同样会被接受。

基于时长的 SDK `renew()` 方法将目标时间计算为 `now + duration`，而非 `当前 expiresAt + duration`。例如，在剩余三十分钟时续期五分钟，会把过期时间提前。只希望延长过期时间的调用方，在确定目标时间前必须先考虑当前过期时间。

## Kubernetes 创建等待

对于 BatchSandbox 和 agent-sandbox provider，服务端会等待工作负载状态变为 `Running` 或 `Allocated` 后才返回创建成功的响应。这并不检查沙箱内的应用是否已就绪。

创建请求会共享既有的资源 watch。工作负载的变更会立即唤醒对应的等待请求，该请求直接评估事件携带的对象，而无需再次调用 API 读取。初始检查和周期性兜底仍然使用常规的缓存/API 读取。与读取操作重叠的事件、以及描述删除的事件会触发一次全新检查。不支持订阅机制的 provider 继续采用轮询方式。

| `[kubernetes]` 配置项 | 默认值 | 含义 |
|------------------------|---------|---------|
| `sandbox_create_timeout_seconds` | `60` | 整体创建等待上限 |
| `pool_acquisition_timeout_seconds` | `30` | Pool 容量耗尽时的累计等待上限 |
| `sandbox_create_poll_interval_seconds` | `1.0` | 无通知到达时的兜底状态检查间隔 |

默认兜底间隔保持不变。增大该值（例如设为 `5.0`）可以在无变化时减少重复检查，但在 watch 通知不可用时会延迟检测。整体与 Pool 容量的截止时间仍然独立于该间隔生效。缓存读取不可用时，会继续回退到 Kubernetes API。

## API 文档

服务端启动后，可通过以下地址访问交互式 API 文档：

- **Swagger UI**：`http://localhost:8080/docs`
- **ReDoc**：`http://localhost:8080/redoc`

## 环境变量

| 变量 | 说明 |
|----------|-------------|
| `SANDBOX_CONFIG_PATH` | 覆盖配置文件路径 |
| `DOCKER_HOST` | 自定义 Docker daemon 地址 |
| `OPENSANDBOX_INSECURE_SERVER` | 设为 `YES` 以在非交互模式下无 API Key 运行 |

## 相关内容

- [安全容器运行时](/guides/secure-container) — gVisor、Kata 与 Firecracker 配置
- [凭证保险库](/guides/credential-vault) — 安全的凭证注入
- [Kubernetes 上的网络隔离](/guides/network-isolation) — Egress 策略设计
- [Kubernetes 部署](/deployment/) — Kubernetes 专属部署
