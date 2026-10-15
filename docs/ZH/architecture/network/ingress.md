---
title: Ingress
description: 将客户端流量路由到沙箱端口的 HTTP/WebSocket 网关——基于头部和路径的路由、原生 WebSocket 代理，以及精确的错误语义。
---

# Ingress

Ingress 网关是 Kubernetes 上沙箱流量的前门：一个反向代理站在所有沙箱之前，无需按沙箱配置。它监视工作负载已经发布的状态——端点注解、Service FQDN 或 FastPath 控制平面——并用实时缓存解析每一个进来的请求。

![Ingress 路由](../../public/images/ingress-routing.svg)

## 一个请求如何到达沙箱

到达沙箱的路由是一个由沙箱 ID 和端口组成的对，通过以下两种模式之一发现：

- **头部模式**（默认）：`OpenSandbox-Ingress-To` 头携带 `<sandbox-id>-<port>`，或者主机名标签编码同样的组合（`my-sandbox-8080.example.com`）。更名前的旧头部仍被接受。
- **URI 模式**：路径的前两个段携带该组合——`/<sandbox-id>/<port>/<path>`——因此无法设置头部的客户端也能工作。

使用[安全访问](/guides/secure-access)时，路由可以改为一个签名的路由 token：沙箱 ID、端口、过期时间和 MAC 编码在一个标签里，无需查询即可验证。

解析从拥有该沙箱的工作负载读取端点——BatchSandbox 端点注解、agent-sandbox 的 Service FQDN，或 Fast Sandbox 路由对应的 FastPath。在转发之前，ingress 会剥离路由头部，添加 `X-Forwarded-*`，过滤逐跳头部，并原样保留请求路径和查询。

## 错误语义

当路由无法继续时，失败是精确的——客户端可以按状态码分支处理：

| 状态码 | 含义 |
|---|---|
| `404` | 沙箱不存在（或已不存在） |
| `503` | 沙箱存在但尚未就绪——尚无可达端点 |
| `502` | 基础设施层面的失败（集群 API 错误之类） |

一旦到达沙箱，其响应会**原样**透传，包括重定向和错误。ingress 自己从不跟随重定向，因此一个陈旧的 `Location` 永远不会把凭据泄漏到无关的端点。

## WebSocket 支持

WebSocket 是一等公民路径，而不是挂在 HTTP 代理之上的升级补丁：

- 升级遵循 RFC 6455 over HTTP/1.1，后端握手预算为 45 秒。
- 客户端选择的子协议和后端 `Set-Cookie` 头都会跨越握手。
- 应用层关闭码原样传播——策略违规（`1008`）或应用自定义码（`4001+`）绝不会改写为 `1000`。
- 单条消息最大 64 MiB，双向皆可，足以承载终端和 Jupyter 流量。
- `Connection` 头的 token 匹配不区分大小写，且两个方向都流式转发，不缓冲对端。

## 在 Ingress 之前部署 TLS

Ingress 只说明文 HTTP/1.1，从不终结 TLS。请在它前面放一个 L7 代理——但要注意 WebSocket 约束：

![TLS 之后的 Ingress](../../public/images/ingress-tls-l7.svg)

WebSocket-over-HTTP/2（RFC 8441）**不被**接受，而 RFC 7540 禁止在普通 h2 请求上使用 `Upgrade` 头。因此前端代理必须以经典 HTTP/1.1 交付 WebSocket 升级。按优先顺序：

1. 从面向浏览器的 ALPN 列表中移除 `h2` —— 可移植、经过验证的回退方案。
2. 针对 WebSocket 的 HTTP/1.1 回退——如果代理支持，同时其他流量仍走 h2。
3. RFC 8441 → RFC 6455 转换——因代理而异；在信任它之前请用真实浏览器验证。

绝不向 ingress 本身转发 HTTP/2：升级永远到不了 WebSocket 处理器，普通代理路径也无法完成握手。

## 健康与网络就绪

- `/status.ok` — 存活：进程已启动并在服务。
- `/status.ok/network-readiness` — 一个**影子**信号，面向关注节点到沙箱网络健康状况的运维者。它对 ingress 在最近一个时间窗口内发起的 TCP 连接进行分类（成功、超时、不可达、拒绝……），当超时和不可达在足够多的不同目标上占主导时报告 `DEGRADED`。

影子端点始终以 `200` 应答 `OK` 或 `DEGRADED`，且从不作为流量闸门——它是运维者的决策输入，不是负载均衡器钩子。窗口长度和阈值可调；其指标中有意不包含目标和沙箱 ID。

## 访问时延长沙箱过期时间

启用后，ingress 会为每个代理请求（在沙箱解析之后）发布一条 **renew-intent** 事件，生命周期服务端则为创建时选择加入的沙箱延长过期时间。发布按沙箱节流，投递是尽力而为的——空闲的沙箱仍会过期；繁忙的沙箱无需客户端改动即可保持存活。

### 按请求退出

单个请求可以通过一个控制头部退出 renew-intent 发布：

```text
OpenSandbox-Access-Renew: skip
```

- 两条代理路径都遵守它——ingress 网关和服务端的 `/sandboxes/{id}/proxy/{port}/...` 路由——因此行为不取决于由哪条路径承载流量。
- 精确的哨兵值 `skip` 抑制该请求的意图（完全绕过按沙箱节流）；未知值出于向前兼容被忽略。
- 该头部在向上游转发之前会被剥离，如同 `OpenSandbox-Ingress-To`，因此后端应用永远不会观察到它。
- 典型用途：健康探针和其他命中已选择加入沙箱的后台流量，而不延长其生命周期。

WebSocket 升级在握手请求上携带该头部；长连接本来也不会产生按请求的意图。

## Fast Sandbox 路由

`FastSandboxProvider` 通过 FastPath v2 解析端点并将流量转发到 Fastlet Pod，为经过认证的 `f1.` 路由范围提供服务。它可以独立运行，也可以与传统 Kubernetes 提供者共享同一个 ingress 进程；路由范围校验使用与服务端相同的签名密钥环。参见[Fast Sandbox：网络](/architecture/fast-sandbox/networking)。
