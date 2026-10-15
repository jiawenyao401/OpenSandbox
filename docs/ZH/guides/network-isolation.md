---
title: Kubernetes 上的网络隔离
description: 为什么 Kubernetes NetworkPolicy 不足以实现沙箱隔离，以及 OpenSandbox 如何改用 egress sidecar 控制来实现。
---

# Kubernetes 上的网络隔离

## 问题

在 Kubernetes 集群上，每个 OpenSandbox 沙箱都作为一个独立的 Pod 运行，并由 CNI 插件分配专用的 Pod IP。默认情况下，任何沙箱都可以通过 Pod IP 直接访问同一集群中的其他沙箱。这带来了若干安全风险：

- **内网扫描**：沙箱内的恶意代码可以扫描集群 Pod CIDR 范围，发现其他沙箱。
- **未授权访问**：攻击者可以直接连接其他沙箱的监听端口，绕过 OpenSandbox 的认证与鉴权机制。
- **数据泄露**：当不同租户的沙箱共存于同一集群时，缺乏隔离可能导致跨租户的未授权数据访问。

因此需要一种机制，在阻止沙箱之间基于 IP 的直接通信的同时，保留合法的外部访问路径（通过 OpenSandbox Ingress）。

## 原生 Kubernetes NetworkPolicy 的局限

Kubernetes NetworkPolicy 通过 **Pod 标签选择器**控制流量，在沙箱隔离场景下面临根本性限制：

- **标签不可预测**：沙箱 Pod 的标签由平台自动注入，用户无法控制。不同租户或不同安全等级的沙箱可能共享相同的标签集合，无法通过标签选择器定义精确的隔离边界。
- **生命周期动态**：沙箱创建和销毁非常频繁。NetworkPolicy 是一种静态声明，无法实时跟踪每个沙箱的预期隔离关系。
- **粒度不匹配**：NetworkPolicy 作用于 Pod 集合（namespace + 标签选择器），而沙箱隔离要求每个沙箱都是独立的安全域，默认拒绝所有其他沙箱的访问。用 NetworkPolicy 表达"拒绝所有其他 Pod 对我的访问"需要为每个沙箱单独编写规则，且无法覆盖未来创建的沙箱。
- **无出站控制**：NetworkPolicy 的 Ingress 规则可以阻止入站流量，但无法阻止沙箱进程主动发起出站连接（例如 curl 另一个沙箱的 Pod IP）。阻止出站直接连接需要为每个沙箱配置双向（Ingress + Egress）策略，这又会遇到同样的标签不可预测和动态性问题。

因此，OpenSandbox 不依赖原生 Kubernetes NetworkPolicy，而是采用**沙箱级出站控制**来实现沙箱间隔离。控制点位于沙箱内部（egress sidecar），而不是集群网络层。

## 方案一：通过 Egress Sidecar `deny.always` 实现全局强制隔离

### 工作原理

OpenSandbox 的 egress sidecar 提供了 `deny.always` 机制：在 sidecar 容器内的 `/var/egress/rules/deny.always` 路径放置一个文件，其中声明的规则将**无条件**生效，优先级高于用户通过 SDK 或 API 设置的任何 allow 规则。

实现位于 `components/egress/pkg/policy/always_rules.go`：

```go
func MergeAlwaysOverlay(user *NetworkPolicy, alwaysDeny, alwaysAllow []EgressRule) *NetworkPolicy {
    // alwaysDeny is prepended so it matches first, achieving unconditional denial
    merged = append(merged, alwaysDeny...)
    merged = append(merged, alwaysAllow...)
    merged = append(merged, out.Egress...)
}
```

每次策略变更（包括初始启动和运行时更新）都会调用 `MergeAlwaysOverlay`，按优先级合并 always-deny、always-allow 和用户策略。由于规则按顺序匹配，`deny.always` 规则拥有最高优先级，用户无法通过 SDK 或 API 覆盖。

此外，`deny.always` 文件每分钟自动重新加载一次，更新无需重启容器即可生效。

### 配置步骤

#### 1. 确定集群 Pod CIDR 和 Service CIDR

获取 Kubernetes 集群的 Pod 和 Service CIDR 范围：

```bash
# Service CIDR: usually set by kube-controller-manager's --service-cluster-ip-range
# Pod CIDR: usually set by the CNI plugin or kube-controller-manager's --pod-network-cidr
```

常见示例值：
- Pod CIDR：`10.244.0.0/16`
- Service CIDR：`10.96.0.0/12`

#### 2. 创建 `deny.always` 规则文件

```text
10.244.0.0/16
10.96.0.0/12
```

支持的目标类型：IP 地址（如 `10.0.0.5`）、CIDR 范围（如 `10.0.0.0/8`）、域名（如 `internal.service.local`，支持 `*.` 通配符前缀）。

#### 3. 基于官方 Egress 镜像构建自定义镜像

创建一个 `Dockerfile`，将 `deny.always` 文件嵌入镜像：

```dockerfile
FROM opensandbox/egress:latest

COPY deny.always /var/egress/rules/deny.always
```

构建并推送到集群可访问的镜像仓库：

```bash
docker build -t registry.example.com/opensandbox/egress:hardened .
docker push registry.example.com/opensandbox/egress:hardened
```

#### 4. 更新 Server 配置

在 OpenSandbox server 配置中，将 `[egress]` 的 `image` 指向自定义镜像：

```toml
[egress]
image = "registry.example.com/opensandbox/egress:hardened"
mode = "dns+nft"
```

该配置生效后，所有新建沙箱都会使用包含 `deny.always` 规则的 egress sidecar 镜像，无需修改单个沙箱的创建请求。

> 注意：`deny.always` 文件每分钟热加载一次。如需修改规则（例如集群 CIDR 变更后），请更新 `deny.always` 文件、重建镜像，并对 server 配置执行滚动更新。

### 效果

- **Pod IP 被阻断**：沙箱无法通过 Pod IP 直接访问同集群内的其他 Pod。
- **Service ClusterIP 被阻断**：沙箱无法通过 Service ClusterIP 访问集群内服务。
- **强制访问路径**：合法的跨沙箱通信必须走 `GetEndpoint()` API，该 API 返回经 OpenSandbox Ingress 代理并附带认证鉴权的外部访问端点。
- **对用户透明**：用户无需在 SDK 调用中声明任何额外参数，隔离在平台层面自动强制执行。

## 放行合法的集群内服务

启用 `defaultAction: deny` 后，要允许沙箱访问某个 Kubernetes Service，需要 egress sidecar 的**两层**同时放行：

1. **DNS 层**：放行服务主机名，使 DNS 代理不返回 `NXDOMAIN`。
2. **网络层**（`dns+nft` 模式）：放行 Service CIDR（或更窄的 ClusterIP 范围），使 nftables 不丢弃已解析的 TCP 连接。

例如，如果沙箱需要访问 `postgres.opensandbox.svc.cluster.local`，仅以下规则是不够的：

```json
{ "action": "allow", "target": "postgres.opensandbox.svc.cluster.local" }
```

当 Service ClusterIP 落在被拒绝的 CIDR（如 `10.96.0.0/12`）内时，请求在 DNS 解析之后仍会失败。在 `dns+nft` 模式下，你还必须放行 Service CIDR（或希望暴露的特定 ClusterIP 范围）：

```json
{ "action": "allow", "target": "10.96.0.0/12" }
```

### 推荐模式

- **平台托管白名单**：如果多个沙箱需要访问相同的内部服务，可将 Service CIDR 放行规则保留在 `allow.always` 中，让用户只请求自己需要的 DNS 名称。
- **按沙箱白名单**：如果访问需要严格限定范围，可在沙箱的 `network_policy` / `networkPolicy` 中同时包含服务 FQDN 和所需的 Service CIDR。
- **不要以为 CoreDNS 豁免就足够了**：sidecar 会自动放行 nameserver IP 以保证 DNS 转发正常工作，但它**不会**自动放行任意 Service ClusterIP 用于应用流量。

## 方案二：通过按沙箱 NetworkPolicy 实现按需隔离

如果不希望全局强制隔离，或某些沙箱需要更宽松的访问权限，可以在创建沙箱时通过 `network_policy` 参数显式拒绝内网访问：

::: warning Pool 模式限制
当 Lifecycle API 通过 `extensions.poolRef` 从预热 Pool 创建沙箱时，不支持按请求传递 `networkPolicy`。Pool Pod 已经存在，server 无法为单个分配注入 egress sidecar，因此会拒绝同时包含这两个字段的请求。请在 Pool Pod 模板创建之前配置所需的网络控制，或者在每次请求需要独立策略时使用非 Pool 沙箱。
:::

```python
from opensandbox import Sandbox, NetworkPolicy, EgressRule

sandbox = await Sandbox.create(
    image="python:3.11",
    network_policy=NetworkPolicy(
        default_action="deny",
        egress=[
            EgressRule(action="deny", target="10.244.0.0/16"),   # Deny Pod CIDR
            EgressRule(action="deny", target="10.96.0.0/12"),   # Deny Service CIDR
            EgressRule(action="allow", target="*.example.com"),  # Allow external domains
        ],
    ),
)
```

### 局限性

- **按沙箱声明**：每个沙箱在创建时必须显式传入 `network_policy` 参数，平台无法默认强制执行。
- **CIDR 暴露**：用户需要知道集群的 Pod CIDR 和 Service CIDR。在多租户场景下，这是不应暴露给用户的敏感信息。
- **可被覆盖**：用户可以在同一个 `network_policy` 中声明更高优先级的 allow 规则来覆盖自己的 deny 规则，因此安全性不如方案一中 `deny.always` 的强制执行。

## 对比

| 维度 | 全局 `deny.always` | 按沙箱 NetworkPolicy |
|-----------|---------------------|---------------------------|
| 强制性 | 是（用户无法覆盖） | 否（用户可修改策略） |
| 用户感知 | 透明（平台级） | 需显式声明 |
| 运维成本 | 低（一次性构建镜像，全局生效） | 高（每个沙箱单独声明） |
| 集群 CIDR 暴露 | 不暴露给用户 | 必须暴露给用户 |
| 适用场景 | 平台级默认隔离（推荐） | 白名单模式、细粒度控制 |

## 运行时兼容性

方案一（通过 egress sidecar 的 `deny.always`）和方案二（按沙箱 `network_policy`）都依赖 egress sidecar，后者使用 iptables `nat` 表的 REDIRECT 规则拦截 DNS。这适用于 `runc`（默认）以及所有 Kata Containers 变体（`kata-qemu`、`kata-clh`、`kata-fc`），但**不适用于 gVisor** —— gVisor 的 netstack 没有实现 `nat` 表。

如果同时需要 gVisor 的 syscall 隔离和 FQDN 出站控制：
- 改用 `kata-qemu` —— 它提供相当的隔离能力，并支持 egress sidecar。
- 或者在使用 gVisor 的同时，采用 CNI 层的 FQDN 策略（例如 Cilium `toFQDNs`）实现网络隔离。

同样的架构约束也适用于 Istio/Envoy sidecar 注入等服务网格：OpenSandbox egress 期望独占 Pod 网络命名空间内的出站拦截。如果网格 sidecar 也在同一命名空间内改写出站流量，则目前不支持与按沙箱网络策略、透明 MITM、Credential Vault 等 egress sidecar 特性共存。建议将沙箱 Pod 排除在网格注入之外，或在平台网络层实施出站策略。

完整的功能支持表请参阅安全容器运行时指南中的[兼容性矩阵](/guides/secure-container#compatibility-matrix)。

## 建议

1. **默认完全隔离**：使用 `deny.always` 阻断集群内部 CIDR 范围，作为平台默认安全基线。
2. **用 `allow.always` 做白名单**：对于需要跨沙箱通信的场景（例如 Agent 向沙箱分发任务），使用 `allow.always` 文件（`/var/egress/rules/allow.always`）开放特定 Pod IP 或 Service DNS 名称。`allow.always` 的优先级高于用户策略但低于 `deny.always`，使平台可以精确控制开放范围。
3. **外部访问入口**：沙箱对外暴露服务的唯一途径应当是 `GetEndpoint()` API，由 OpenSandbox Ingress 代理。不应将 Pod IP 用作对外服务端点。
