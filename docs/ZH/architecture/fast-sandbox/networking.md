---
title: Fast Sandbox 网络
description: 流量如何进出 Fast Sandbox 沙箱 — 入站是已签名的路由作用域和 FastPath 解析；出站是基于主体的出口强制执行。
---

# 网络

Fast Sandbox 的流量流经本仓库中的两个适配器：

- **入站**（客户端 → 沙箱）：入口网关的 `FastSandboxProvider` 验证已签名的路由作用域，并通过 FastPath 端点解析将流量转发到 Fastlet（`components/ingress/`）。
- **出站**（沙箱 → 外部）：每个 Fastlet pod 的一个共享出口进程为 N 个沙箱强制执行每主体策略，由 Sandbox Actions 协议驱动（`components/egress/`）。

## 设计要点

- **已认证的作用域取代原始路由键**：一条路由是一个已签名的 `f1.` 作用域，绑定租户命名空间、沙箱 ID 和端口。入口先验证再路由；不存在未经认证即可到达沙箱的路径。
- **惰性解析**：上游地址只在流量到达时解析，因此路由在 Fastlet 重新调度后依然有效，无需失效协议。
- **fail-closed 出口**：每个主体从注册起即拒绝流量，直到其数据平面就绪 — 策略交付可以迟到，但绝不能提前开放。
- **策略即数据**：出口策略随 Sandbox CRD 带版本的 `actionBindings` 传递；只有凭据走带外通道，且仅存于内存。
- **单一分派点**：DNS 过滤和网络强制执行是共享的 pod-netns 服务，按源 IP 对每个数据包分派。

## 入站：从路由作用域到 Fastlet

### 路由作用域

Fast Sandbox 路由是一个经过认证的作用域，而非普通的 host 或路径。服务器以 `opensandbox-fsb-route-v1` 方案对租户命名空间、沙箱 ID 和端口签名；`f1.` 前缀标识作用域格式版本，入口与服务器共享签名密钥环。

- Header 模式将该作用域表达为 `OpenSandbox-Ingress-To` 头；URI 模式将其编码在路径中。
- 通配符 host 作用域被有意不支持：命名空间、沙箱 ID 和 MAC 无法安全地放进单个 DNS 标签。
- 客户端必须原样保留返回的路由和头 — 作用域就是凭据。

服务器签发这些路由时无需调用 FastPath，也无需等待就绪：对租户命名空间、沙箱 ID 和端口签名是一项以共享入口密钥环为密钥的本地操作。因此签发与沙箱状态解耦，端点发现永远不能证明存在性 — 入口在流量到达时才解析并验证。

### 端点解析

请求到达时，入口通过 FastPath 的 `ResolveEndpoint` RPC 解析上游地址。解析返回目标端点、要转发经过的代理地址以及所需的头。

失败语义是契约的一部分：

- `FailedPrecondition` — 包括尚未发布的路由 — 以 `503` 加 `Retry-After` 呈现。它绝不会作为"沙箱缺失"上报：入口不得泄露"未就绪"与"不存在"之间的区别。
- 策略端口（`18080`）为 SDK 兼容性可被解析，但对流量返回 `501`；策略操作属于服务器的网络策略路由，不属于数据路径。

### 访问模式

| 模式 | 路径 | 权衡 |
|---|---|---|
| `direct-fastlet-proxy`（默认） | 入口 → 直接到达 Fastlet pod | 延迟更低；要求入口能到达 Fastlet pod IP |
| `central-proxy` | 入口 → fast-sandbox 中央代理 → Fastlet | 无 pod IP 可达性要求；多一跳 |

直连模式绕过中央代理，因此 pod 网络必须将 Fastlet 数据平面端口限制为可信的入口工作负载，并且当两套系统分开部署时，FastPath 必须准入入口命名空间。该模式按部署选择，而不是按请求选择。

### 传输与互操作性

入口只使用 FastPath v2 的 `ResolveEndpoint` 子集，其线上修订版本与服务器生成的客户端固定一致，因此随着 fast-sandbox 演进，组件之间保持互操作。gRPC 传输为明文；入口、Fastlet 与 FastPath 之间的隔离是部署层面的考量（网络策略），TLS 推迟到双方都支持之后。

## 出站：一个出口控制平面，N 个主体

Fastlet pod 在一个网络域中承载多个沙箱，因此每沙箱出口 sidecar 模式不适用。出口组件运行可选的 **fast-sandbox profile**（`OPENSANDBOX_EGRESS_PROFILE=fast-sandbox`）：每个 Fastlet pod 的一个出口进程以独立**主体**的形式服务 N 个沙箱，每个主体拥有自己的策略、凭据和内核规则。默认 sidecar profile 保持不变；两种 profile 是互斥的部署形态。

策略以数据形式到达主体：服务器通过共享生命周期策略路由将其写入 Sandbox CRD 带版本的 `actionBindings`，语义与 sidecar 兼容 — 替换是全量的，合并遵循 sidecar 规则（同目标规则原地替换，每目标先到先得），删除按目标进行且幂等。写入以沙箱 UID 和代数加栅栏（失去竞争时返回 `409`；其他租户与缺失不可区分，返回 `404`），成功的写入只提交意图 — 强制执行在 Fastlet 上异步收敛，而清空的绑定会将主体重置为默认拒绝。

### 网络槽位

每个沙箱在存在之前就在 Fastlet pod netns 中被预置了一个**网络槽位**：netns、veth 和伪装在任何创建请求之前就已设置好，附属身份随 `SET_BINDING` 信封一起交付。沙箱永远不会等待网络设置，出口从它的第一个数据包起就能强制执行策略。

![Fast Sandbox network slots](../../public/images/fast-sandbox-network-slot.svg)

槽位的 IP 是分派键：共享 DNS 代理按源 IP 为每个数据包选择主体，因此 Fastlet 的一个服务就能以每主体策略服务所有沙箱。

| `attachment.network` 字段 | 作用 |
|---|---|
| `ip` | 分派键 — `ip saddr` 匹配选中该主体的规则 |
| `gateway` | DNS REDIRECT 的目标 |
| `privateCidr` | 沙箱之间的同侪隔离规则 |
| `hostVeth` | 保留 — 在桥接拓扑上，策略钩子看到的是桥接设备，而非 veth |

### 主体生命周期

**主体**是策略、凭据和规则所有权的单元 — 每个沙箱一个，以沙箱 UID 标识。出口进程维护一个每主体状态机：

```text
absent ──SET_BINDING──► denying ──data-plane-ready──► active
   ▲                        │                            │
   └──────REMOVE_BINDING────┴────────────────────────────┘
```

主体生命周期和策略由 Fastlet 通过公开的 **Sandbox Actions Handler** 协议（`sandbox.fast.io/actions/v1`）交付。出口进程就是处理器；两个 Pod 环回 HTTP 端点为其提供服务：

- `POST /_fastlet/v1/actions` — `SET_BINDING`、`LIFECYCLE_HOOK`、`REMOVE_BINDING`。
- `GET /_fastlet/v1/actions/status` — 进程化身探针。`instanceId` 发生变化会使 Fastlet 重放最新绑定和所有已到达的钩子。

每个操作都携带栅栏字段（`runtimeInstanceId`、`attachmentId`），因此控制平面重置永远不会被误认为是一次重新绑定。未知的协议版本、操作或钩子名称都是验证错误，绝不会被静默忽略。

### 凭据交付

凭据保管库的修订版本由服务器通过 fast-sandbox 的代理路由机制推送，并按 `X-Fast-Sandbox-Uid` 头路由到各主体。推送在出口中保持仅存于内存 — 没有 Secret 卷，没有磁盘状态。

- 对一个绑定尚未出现的 UID 的推送会被短暂缓存，并在注册时应用。
- 携带不匹配的 `X-Fast-Sandbox-Generation` 的推送会被丢弃。

### 强制执行

强制执行由 Fastlet pod 网络命名空间中的 nftables 实现，按源 IP 分派 — 每个操作信封的附属块提供主体的 IP、网关、主机 veth 和私有 CIDR。

- **DNS**：环回 `127.0.0.1:15353` 上的一个共享代理。每主体的 prerouting REDIRECT 将发往附属网关的沙箱 DNS 转发出去，同时保留源 IP，以便按查询将策略分派到正确的主体。
- **转发路径**：每主体的 `hook prerouting` 链标记允许的目的地（`meta mark 0x2`）；主转发链丢弃未标记的流量。该路径刻意从不发出显式的 `accept` — 在 `net.bridge.bridge-nf-call-iptables=1`（fast-sandbox 桥接拓扑）下，accept 判定会把帧交回桥接 L2 而丢失。
- **动态集合**：DNS 学习到的 IP 获得有界租约；每主体的连接刷新循环为活跃 TCP 连接续租。UDP/QUIC 依赖 DNS TTL。
- **加密 DNS**：DoT（853）始终被丢弃；可选的 443 端口 DoH 阻断对所有主体全局生效。
- **链式上游代理**：当 `OPENSANDBOX_EGRESS_UPSTREAM_PROXY` 与透明 MITM 同时设置时，所有共享 mitmproxy 的出口流量都会通过 `CONNECT` 链接到该代理。该端点是基础设施，不是沙箱出口：它在主分派链中被全 profile 范围丢弃（forward 和 input 路径，位于每条每主体规则之前），因此任何主体策略 — 包括默认放行 — 都无法直接 CONNECT 该代理。主机名端点在共享 DNS 代理上注册为基础设施域名，因此其应答永远不会进入动态放行集合；共享的 mitmdump 通过 Fastlet pod 自己的解析器解析该名称，因此它必须能通过集群 DNS 解析。由于 DNS 代理的转发上游与 pod 解析器可能返回不同的地址集合（split-horizon 或运维配置的 DNS），出口的自解析刷新会同时查询两个权威方，并以并集填充丢弃集合 — 仅由 pod 解析器返回的地址正是沙箱可能直接 CONNECT 的地址。丢弃元素是永久的（无内核超时）：与表中其他规则一样，封堵在出口停机期间持续有效，过期由出口自己负责 — 刷新循环会修剪两个权威方都不再返回的地址，而一次失败的解析会保留所有元素（fail-closed 保留）。首次填充在启动时以有界退避重试，如果主机名无法解析则使出口启动失败，这发生在任何沙箱操作被服务之前。
- **上游代理刷新与恢复**：两个解析权威方并发运行。部分失败会被记录，且只会添加已返回的地址；修剪要求在两个权威方各自两次成功刷新中均缺席，而沙箱的 DNS 查询会为保留续期。在重置 nft 表之前，启动会以有界重试解析主机名并填充内存中的地址。两个权威方都必须成功完成并得到非空并集；失败则保留先前的内核表不变。DNS 解析和 nft 应用分别有独立的超时。
- **原子性**：静态集合以原子方式交换；重启时，出口在安装已填充的代理地址的同时清除过期的主体规则，发布新的 `instanceId`，Fastlet 会重放每一个活跃绑定。
