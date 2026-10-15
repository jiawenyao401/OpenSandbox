---
title: Egress 链式上游代理运维
description: 在 OpenSandbox 中配置、运维、验证和排查链式上游 HTTP CONNECT 代理。
---

# Egress 链式上游代理运维

OpenSandbox 为以出站网络策略运行的沙箱提供了管理员管理的链式上游代理能力。本指南说明如何在 Docker 和 Kubernetes 环境中配置、运维、验证和回滚上游转发代理的下一跳。

## 概述与前提条件

链式上游代理通过标准 HTTP CONNECT 隧道将出站流量路由到外部 HTTP/HTTPS 转发代理。该配置是生命周期服务器全局且仅管理员可用的；请求环境无法覆盖代理目的地、凭证或自定义 CA 包。

前提条件：
- 通过以 `networkPolicy` 配置沙箱来启用 egress sidecar。
- 服务器 `[egress]` 配置设置为 `mode = "dns+nft"`。
- 通过 `credentialProxy.enabled=true` 或 `OPENSANDBOX_EGRESS_MITMPROXY_TRANSPARENT=true` 启用透明拦截。
- Docker：`[docker] network_mode = "bridge"`。
- Kubernetes：非池化的 AgentSandbox 或 BatchSandbox 工作负载。

::: warning
注入到同一 Pod 网络命名空间的透明服务网格 sidecar（如 Envoy 或 Istio）不受支持，因为两层都会拦截并重写出站数据包。请改用外部链式上游代理跳。
:::

## Docker 与 Kubernetes 配置示例

在生命周期服务器配置中配置 `[egress.upstream_proxy]` 部分。完整的配置模式和字段定义请参考[服务器配置参考](https://github.com/opensandbox-group/OpenSandbox/blob/main/server/configuration.md#egress)。

### Docker 配置

对于 Docker 中的私有代理 CA，`ca_cert_path` 必须引用 Docker 守护进程主机上的绝对 POSIX 路径。该文件以只读方式绑定挂载，且仅挂载到 egress sidecar 容器。

```toml
[egress]
image = "opensandbox/egress:<version>"
mode = "dns+nft"

[egress.upstream_proxy]
url = "https://proxy.example.com:8443"
# Optional complete Proxy-Authorization value:
# authorization = "Basic <base64>"
ca_cert_path = "/etc/ssl/certs/proxy-ca.pem"
```

### Kubernetes 配置

对于 Kubernetes 中的私有代理 CA，`ca_secret_name` 引用沙箱命名空间中的一个 Secret。该 Secret 必须包含固定键 `ca.crt`，它以只读方式投影到 egress sidecar 容器。

```toml
[egress]
image = "opensandbox/egress:<version>"
mode = "dns+nft"

[egress.upstream_proxy]
url = "https://proxy.example.com:8443"
# Optional complete Proxy-Authorization value:
# authorization = "Basic <base64>"
ca_secret_name = "upstream-proxy-ca"
```

## 流量与安全模型

关于出站数据包流的详细概述，参见 [/architecture/network/egress](/architecture/network/egress)。

1. **解析与路由**：在 `dns+nft` 下，即使处于 deny-all 策略之下，代理主机名也由基础设施 DNS 解析。出站可达性由严格限定于 mitmproxy UID、代理 IP 和代理端口的专用 nftables 集合强制执行，将代理流量与沙箱动态允许集合隔离。
2. **协议与下一跳**：`http://` 和 `https://` 代理端点都受支持。对于 HTTPS 端点，强制执行 SNI 和服务器主机名验证。配置的 CA 包扩充系统信任存储，并应用于所有 mitmproxy 上游 TLS 验证。
3. **CONNECT authority**：被拦截的 TLS 流量在可用时使用 SNI 或从 Host 派生的 FQDN 作为 CONNECT authority，回退到 `IP:port`。启用链式转发时严格拒绝直接拨号以防止静默绕过。
4. **凭证隔离**：来自 Credential Vault 的业务凭证被注入到内部目标请求，而 `authorization` 仅作为 `Proxy-Authorization` 附加在外部 CONNECT 请求上。两个值都不会写入日志。密钥管理详情参见 [/guides/credential-vault](/guides/credential-vault)。
5. **密钥暴露边界**：代理授权值以字面环境变量的形式出现在 egress 容器内，并可通过 `docker inspect` 或 Kubernetes Pod 规格查看。请相应地限制主机和集群的检查权限。授权环境和 CA 挂载只添加到 egress 容器，而不是沙箱容器。

## 运行时兼容性矩阵

兼容性仅反映当前上游 `main`：

| 运行时 / 平台 | 支持状态 | 运维约束 |
|---|---|---|
| Docker (`runc`) | 支持 | 要求 `[docker] network_mode = "bridge"`；仅在需要私有 CA 时使用 `ca_cert_path`。 |
| Kubernetes (`runc`) | 支持 | 非池化的 AgentSandbox 和 BatchSandbox 工作负载；仅在需要私有 CA 时使用 `ca_secret_name`。 |
| Kubernetes (`Kata`) | 支持 | 被 egress 校验接受；请在你的部署中验证所选的 RuntimeClass、CNI 和内核。 |
| gVisor (`secure_runtime.type = "gvisor"`) | 拒绝 | 沙箱创建返回 HTTP 400，因为 gVisor netstack 缺少所需的 iptables nat 重定向。 |
| Kubernetes 池模式 | 不支持 | 拒绝每请求 `networkPolicy`；无法附加每分配的 egress sidecar。 |
| Fast Sandbox | 不支持 | 当前 fast-sandbox 配置在设置了 `upstream_proxy` 时拒绝 `networkPolicy`。 |

## 协议与流式行为

| 协议 / 流量类型 | 受支持 | 行为与约束 |
|---|---|---|
| HTTP/1.1 和 HTTP/2 应用流量 | 是 | Mitmproxy 可以与应用目标协商 HTTP/2。上游 web 代理跳使用 HTTP/1 CONNECT；mitmproxy 11.0.2 不支持 HTTP/2 CONNECT。 |
| 端口 80 和 443 | 是 | 始终拦截；Credential Vault 绑定适用于规范端口 80/443。 |
| 额外 TCP 端口 | 有界 | 配置时会被拦截，但 Credential Vault 绑定不适用。 |
| 流式和大型请求主体 | 有限保留 | 凭证头注入发生在主体流式传输之前的 `requestheaders` 阶段。流式主体的主体占位符替换会被跳过。 |
| Server-Sent Events (SSE) | 注意事项 | 被强制进入流式模式。如果源以 TCP RST 关闭则可能发生截断；参见 [/guides/egress-sse-truncation](/guides/egress-sse-truncation)。 |
| TLS 透传 | 否 | 非空的 `ignore_hosts`、`tcp_hosts` 或 `udp_hosts` 会使上游插件加载失败；无 SNI 透传在请求时被拒绝。 |
| UDP 和 QUIC | 否 | 这些流无法使用 HTTP CONNECT，会被拒绝而不是直接发送。 |
| WebSocket (WSS) | 未验证 | 专门的链式代理 WSS 兼容性未经独立验证。 |
| IPv6 出站 | 不完整 | Kubernetes 双栈支持不完整；`egress.disable_ipv6` 默认为 `true`。 |

## 部署、轮换与回滚

- **应用边界**：代理配置更改仅适用于新创建的沙箱。现有 sidecar 保留其创建时的代理设置。
- **代理或授权轮换**：更新生命周期服务器配置，重启生命周期服务器，然后重建受影响的沙箱。
- **CA 轮换**：更新 Docker CA 文件或 Kubernetes Secret。如果其配置的路径或 Secret 名称未变，生命周期服务器不需要重启，但受影响的沙箱仍需重建。重建可以避免依赖 bind-mount 或 Secret `subPath` 的更新行为。
- **回滚过程**：移除 `[egress.upstream_proxy]` 部分，重启生命周期服务器，并重建受影响的沙箱。现有沙箱在重建前保持不变，且代理失败绝不会触发自动直接回退。

## 验证清单

1. 确认生命周期服务器以新配置启动；无效的 URL、模式和运行时特定 CA 字段在沙箱创建之前就会验证失败。
2. 创建一个启用了 `networkPolicy` 和透明拦截的新沙箱。
3. 确认 egress 健康端点返回 `200 OK`；`503 mitmproxy not ready` 表示拦截进程未初始化。
4. 检查启动日志中的 `credential proxy: upstream proxy chaining enabled` 消息，且不打印环境值。
5. 从沙箱中，以正常证书验证访问一个允许的 HTTPS 目的地。不要使用 `curl -k` 或设置 `ssl_insecure`。
6. 确认被拒绝的目的地仍被拒绝，并检查外部代理的遥测中预期的 FQDN CONNECT authority。
7. 如果启用了 Credential Vault，确认目标收到业务凭证，而转发代理只收到 `Proxy-Authorization`。

## 故障排查表

| 故障类别 | 观察到的症状 | 根因与解决方法 |
|---|---|---|
| 配置加载错误 | 服务器启动失败；配置解析错误 | 无效的代理 URL、非绝对的 Docker CA 路径，或混用 Docker 和 Kubernetes CA 设置。 |
| 请求 400 错误 | 沙箱创建被 HTTP 400 拒绝 | 缺少透明拦截、选择了 `secure_runtime.type = "gvisor"`，或将 Pool/Fast Sandbox 与每请求 `networkPolicy` 组合。 |
| Pod 或 bind 设置失败 | Kubernetes Pod 一直 pending，或 Docker 容器创建失败 | Kubernetes Secret/键缺失，或 Docker 守护进程无法读取配置的主机路径。更正管理员管理的源并重建沙箱。 |
| Sidecar 启动失败 | egress 容器重启或未通过就绪检查 | 自定义透传设置不兼容，或挂载的 CA 包格式错误或不可读。 |
| Sidecar 503 错误 | egress 健康检查返回 HTTP 503 | Mitmproxy 尚未初始化。检查 sidecar 启动日志；运行时代理中断影响请求，但本身不会使本地监听器未就绪。 |
| TLS / SAN 失败 | 握手失败；证书验证错误 | 上游代理证书 SAN 与代理主机名不匹配，或缺少自定义 CA 包。 |
| 代理认证失败 | 通过代理的请求失败；代理遥测报告 407 | 在服务器配置中提供完整有效的 `authorization` 值并重建沙箱。 |
| 直接拨号错误 | 日志包含 `upstream proxy required: direct egress dial refused` | 对无法链式的流的预期 fail-closed 行为；不要通过禁用 TLS 验证来绕过它。 |
| SSE / 流截断 | 流式连接过早中断 | 源在仍有未读字节时以 TCP RST 关闭连接；检查上游网关 keep-alive 设置。 |

## 已知限制与参考

- **无每请求代理**：代理设置不能按请求或通过环境变量修改。
- **无热重载**：现有沙箱 sidecar 无法动态重载代理配置。
- **无专用 OTLP 指标**：没有专用的链式代理指标族；依靠 egress 健康检查、DNS/nftables 指标和代理日志。
- **运行时范围**：不对每种 Kata 实现、自定义 RuntimeClass、CNI 或内核组合声明生产认证。
- **参考**：
  - [Egress 架构规范](/architecture/network/egress)
  - [Credential Vault 运维指南](/guides/credential-vault)
  - [Server-Sent Events 截断指南](/guides/egress-sse-truncation)
  - [服务器配置参考](https://github.com/opensandbox-group/OpenSandbox/blob/main/server/configuration.md#egress)
