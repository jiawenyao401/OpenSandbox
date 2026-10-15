---
title: Egress
description: 按沙箱的出站控制——在 DNS 层和网络层强制执行的 FQDN 允许列表、透明凭据注入，以及快速失败的启动。
---

# Egress

Egress 让沙箱获得的是网络策略，而不是对互联网的完全开放。它以 sidecar 形式运行，与沙箱共享网络命名空间，因此每一个出站数据包都会经过它——无需应用配置，也无法从沙箱内部绕过。策略在创建时（`networkPolicy`）或由平台声明，并被持续执行。

![Egress 控制流](../../public/images/egress-control-flow.svg)

## 能力一览

| 能力 | 你能得到什么 |
|---|---|
| FQDN 规则 | 按域名、通配符（`*.pypi.org`）、IP 或 CIDR 允许或拒绝 |
| DNS 过滤（默认） | 被拒绝的域名解析失败；允许的域名正常解析 |
| 网络强制（`dns+nft`） | 在数据包层实施严格的默认拒绝，由 DNS 解析出的地址驱动，租约有界 |
| 平台覆盖层 | 运维者设定的 `deny.always` / `allow.always` 底线，用户策略无法覆盖 |
| Credential Vault | 密钥留在 sidecar 中；凭据注入到匹配的出站请求里 |
| 透明 TLS 拦截（L7） | 查看、控制并认证出站 HTTPS——机制可在生产使用，配置面为实验性 |
| 快速失败启动 | 如果强制规则无法安装，sidecar 直接退出——绝不会以空转方式运行 |

## 拦截如何工作

- **第 1 层 — DNS。** 所有 53 端口的流量都被重定向进 sidecar 的 DNS 代理。被拒绝的域名应答 `NXDOMAIN`；允许的域名继续向上游真实解析，其解析出的地址会被记录。
- **第 2 层 — 网络（`dns+nft` 模式）。** nftables 丢弃所有未显式允许的流量。被允许的流量通过静态规则或 DNS 学习到的地址集放行：一个解析出的 IP 会获得有界租约（对活跃 TCP 连接有宽限窗口），因此"被允许"绝不会悄悄变成"永远被允许"。UDP 和 QUIC 流仅依赖 DNS 租约的时间。
- **优先级。** 首条匹配规则生效：先是平台 `deny.always`，再是平台 `allow.always`，最后是你的策略——平台的拒绝永远压过你的允许。覆盖层存放在 sidecar 镜像内，每分钟热重载一次；只有当 nftables 接受了相应的静态策略后，重载才会发布。解析或应用失败时保留最后已接受的内存规则，并保持可重试状态。
- **运行时修改。** 你的策略是按沙箱设置的，并可在运行时通过 SDK 暴露的 API 修改（按目标添加、替换、删除）。

## 设计决策

| 决策 | 理由 |
|---|---|
| 强制执行位于数据路径中 | sidecar 在共享命名空间中安装重定向规则；工作负载不持有 `NET_ADMIN`，因此没有任何东西可以绕过，也不需要应用配合。 |
| 两层，两种成本 | 仅 DNS 过滤便宜且通用，但只能约束域名。`dns+nft` 增加了包级默认拒绝，以及 DNS 学习到的、有租约的例外。 |
| 加密 DNS 是逃生通道 | DoT（853 端口）始终被丢弃，在 443 上阻断 DoH 是一个开关选项。 |
| 宁可快速失败也不尽力而为 | 如果重定向规则无法安装，sidecar 会退出而不是以静默直通方式运行；监督进程以退避和崩溃循环熔断器重启它。 |
| 通过回退实现可移植，对限制保持诚实 | 重定向在可用时通过 iptables 安装，否则回退到原生 nftables（Firecracker 风格的 guest 内核）。gVisor 的 netstack 无法支持这一点——Kata 是受支持的安全运行时——而透明的 mesh sidecar 在结构上与其冲突。 |

## 透明 TLS 拦截（L7）

第 1 层和第 2 层决定一条连接*是否*可以发生；透明 MITM 层增加了对*什么东西通过它*的控制与可见性。启用后，出站 HTTP 和 HTTPS 被重定向进 sidecar 内的 mitmproxy 监听器，它从连接状态恢复真实目的地，并代表沙箱终结 TLS。

**状态：实验性，但可在生产使用。** 拦截、凭据注入和 CA 分发已完成，并在生产环境中运行；配置面——额外端口和诊断开关——在版本之间可能仍有变化。

::: warning 内部 OSEP-0023 revision 运行时
`OPENSANDBOX_EGRESS_EXPERIMENTAL_REVISION_RUNTIME=true` 是一个仅用于开发的 sidecar 开关（默认关闭），面向 OSEP-0023 revision 协议。它不增加任何公共配置面、没有 `interceptionMode`，也不支持 HTTP/2 或 Fast Sandbox；范围限定为已绑定的 HTTPS/443 HTTP/1.1。关键属性：

- 准入是按连接、按请求进行的，基于通过已认证信道安装的不可变快照；未知或不匹配的状态会被拒绝——该模式绝不会回退到解密一切。
- 任何不确定的结果（未确认的变更、会话清理失败、子进程意外退出）都会锁存一个粘性的 recovery-required 状态：就绪下降、nft 写入静默，并在进程存续期间安装一张独立的隔离表。重启 sidecar 不是经过验证的恢复方式——应当销毁并重建沙箱。
- 更新不是单一的原子可见性边界：nft 规则可能在接收方确认之前影响流量，且如果 API 响应丢失，已提交的更新不会被回滚。退役连接在一段有界窗口内（默认 30 秒）排空。

完整契约，包括设置 `OPENSANDBOX_EGRESS_POLICY_FILE` 时由 egress 侧拥有的策略文件存储要求：[凭据保管库指南](/guides/credential-vault#experimental-live-credential-bound-admission-osep-0023)与 [OSEP-0023](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0023-credential-bound-tls-interception.md)。
:::

**信任是交付的，而不是禁用的。** sidecar 导出它的 CA，沙箱引导过程尽力将其安装进系统、NSS 和 JDK 信任库——客户端保持证书校验开启（不带 `-k` 的 `curl`）。Chromium 系镜像应内置原生 `certutil` 包，以便更新按用户的 NSS 存储。

它带来的能力：

- **Credential Vault** — 凭据注入发生在读取请求头时，因此无论请求体多大都适用，包括完全流式化的上传。
- **TLS 级可观测性** — 可选的请求级结果指标（解密 / 直通），用于定向的诊断窗口。

**可在 L7 扩展。** 内置的系统插件总是最先加载，随后按顺序加载 `OPENSANDBOX_EGRESS_MITMPROXY_SCRIPT` 中以逗号分隔的任意 mitmproxy 插件。插件是标准的 mitmproxy 脚本，拥有完整的 L7 钩子面（主机名、方法、URI 路径、头部、请求体），因此运维者可以以普通 Python 插件的形式添加主机名/方法/头部过滤、头部注入或请求改写——叠加在内置行为之上，绝不会先于内置行为执行。

**链式上游代理。** 设置 `OPENSANDBOX_EGRESS_UPSTREAM_PROXY` 为 `http(s)://host[:port]`——可选配合 `OPENSANDBOX_EGRESS_UPSTREAM_PROXY_AUTH`（一个完整的 `Proxy-Authorization` 值）——即可让所有由 mitmproxy 处理的出站流量通过该代理经由 `CONNECT` 链式转发：

- 该端点只接受 `host[:port]` 或单个 `/` 路径，且 host 必须是字面 IP 或带点的域名（不带点的名称会通过 Pod 解析器的 search list 产生不同的解析结果，可能导致包含集合错过实际拨号地址）。凭据、查询、片段及其他路径都会被拒绝。
- 快速失败：无法链式处理的流（无 SNI、`ignore_hosts`/`tcp_hosts`/`udp_hosts` 匹配、UDP）会被拒绝，而不是直连。链式要求透明 MITM，且在 fast-sandbox profile 之外还要求 `dns+nft`；无效组合会导致启动失败。
- 在 `dns+nft` 下，代理端点仅可从 mitmproxy 进程访问（按 UID、IP 和端口限定的 nft 规则），绝不会通过沙箱允许集合访问。字面代理 IP 会被永久播种进这些集合；DNS 学习到的则使用有界 TTL。
- Fast-sandbox profile：该端点在按主体规则之前被 profile 级丢弃，其主机名注册为基础设施域名。共享的 mitmproxy 并发查询 Pod 解析器及其转发上游，并将并集播种为永久丢弃集合；除非两者都以非空并集成功，否则启动失败（此前内核表保持完好）。
- 配置：Docker 和 Kubernetes 上的服务端 `[egress.upstream_proxy]` 配置段——参见[服务端配置](https://github.com/opensandbox-group/OpenSandbox/blob/main/server/configuration.md#egress)与[链式上游代理运维](/guides/egress-upstream-proxy)。同一配置段还可以交付一份额外的 CA bundle（Docker 宿主机 PEM 路径，或只读挂载进 egress sidecar 的 Kubernetes Secret），mitmproxy 会在系统根证书之外——而非替代系统根证书——信任它。

**范围说明：**

- 端口 80 和 443 始终被拦截；可以额外添加一份有界的端口列表。Vault 绑定匹配目前仅在 80/443 上触发——额外端口会被解密并记录日志，但不注入凭据。
- IPv6 目的地与 IPv4 一起被拦截（监听器和重定向规则覆盖两个地址族）。
- 尚未初始化完成的代理在健康检查中可见：sidecar 会应答 `503 mitmproxy not ready`，而不是不加检查地放行流量。
- 当源端在响应体中段以 TCP RST 关闭连接时，超过约 1 MB 的 SSE 响应体可能被截断——这是标准 TCP 语义，不是拦截缺陷。参见[Egress SSE 截断](/guides/egress-sse-truncation)。

## 凭据不进入沙箱

在启用透明 TLS 拦截后，sidecar 可以向匹配 vault 绑定的出站请求注入凭据——bearer token、basic auth、API key 或自定义头部。Vault 只存在于 sidecar 的内存中：沙箱工作负载看到的是已认证的流量，但从不持有密钥本身。参见[Credential Vault](/guides/credential-vault)。

## 部署形态

| 部署方式 | 形态 |
|---|---|
| Docker | 一个独立的 sidecar 容器；沙箱加入 sidecar 的网络命名空间 |
| Kubernetes | sidecar 被追加到沙箱 Pod 中，并持有该 Pod 唯一的 `NET_ADMIN` |
| 池化沙箱 | 预热 Pod 无法按请求追加 sidecar——请将所需的控制放入 Pool 模板（按请求的 `networkPolicy` + 池引用会被拒绝） |
| Fast Sandbox | 每个 Fastlet Pod 一个共享出站进程，每个沙箱一个主体——参见[Fast Sandbox：网络](/architecture/fast-sandbox/networking) |

## Service Mesh 兼容性

::: warning 不支持与透明 mesh sidecar 共存
Egress 被设计为沙箱网络命名空间中唯一的透明出站拦截层。向同一 Pod 注入 service mesh sidecar（Istio/Envoy 及类似）的部署，当前不支持 egress 功能。
:::

两层都在同一命名空间中改写出站流量，因此在注入 mesh 的前提下，无法依赖按沙箱策略、透明 MITM 和 Credential Vault。建议将沙箱 Pod 排除在 mesh 注入之外，或用 CNI 级机制执行出站策略。同样的约束适用于 gVisor——请使用 Kata，它提供相当的隔离能力并完整支持 egress（参见[兼容性矩阵](/guides/secure-container#compatibility-matrix)）。

## 关闭

关闭时，sidecar 会让 DNS 和网络规则在在途投递完成（一段有界窗口）之前保持工作，然后在传统路径中移除网络规则并冲刷遥测；实验性包含所有者则改为保留普通规则。投递是尽力而为的——强制终止可能丢失事件。Docker 在删除时给 sidecar 9 秒的停止预算。崩溃由监督进程以指数退避重启，带有崩溃循环熔断器，并在全新启动前清理残留的重定向规则。

## 可观测性（OpenTelemetry）

当平台配置了 OTLP 端点时，sidecar 导出按沙箱的指标：被拒绝的请求计数、DNS 查询结果（被策略拒绝与解析器失败被有意分为两个计数器——对错误的计数器告警会颠倒诊断结论）、nftables 更新失败，以及进程资源占用。基数有界：被查询的域名或目的地永远不会成为标签。指标清单和导出配置参见[遥测](/guides/telemetry#egress-sidecar)。
