---
title: 单机网络（Docker）
description: Docker 运行时如何通过 execd 的反向代理，让每个沙箱只用一个宿主机端口即可暴露全部端口——host 与 bridge 网络模式。
---

# 单机网络（Docker）

本页描述单台主机上的 **Docker 运行时**。它在 Kubernetes 上的对应物是 [Ingress 网关](/architecture/network/ingress)，负责在整个集群内路由沙箱流量。

Docker 运行时让每个沙箱只用一个宿主机端口即可暴露全部端口：`execd` 内置一个反向代理，运行时只把该代理端口映射到宿主机。

![单机沙箱路由](../../public/images/single-host-network.svg)

## 单机路由模型
- 每个沙箱容器启动时运行监听容器端口 `44772` 的 `execd`。`execd` 内置一个轻量反向代理，拦截带 `/proxy/{port}` 前缀的请求，并将其转发到同一容器内的 `127.0.0.1:{port}`。
- Docker 运行时只绑定 execd 代理端口的宿主机一侧（标注为 `opensandbox.io/embedding-proxy-port`）。调用者使用 `get_endpoint(..., port=X)` 获得 `{public_host}:{host_proxy_port}/proxy/{X}`，execd 将请求透明地路由回端口 `X` 上的沙箱服务。
- 由于代理保留了 `Upgrade`、`Connection` 等其他 HTTP 头，HTTP、Server-Sent Events 和 WebSocket 流量无需额外配置即可共享同一个映射的宿主机端口。
- 有了这种设置，每个沙箱只需一个宿主机端口即可访问**所有**容器端口。你可以在一台机器上安全运行大量沙箱，而不必担心宿主机端口分配冲突。
- 当调用者确实能够路由到沙箱的 Docker 网络时，使用 `get_endpoint(..., resolve_internal=True)` 绕过宿主机映射，直接返回沙箱 IP（例如 `172.17.0.3:5900`）。仅仅把生命周期服务端跑在容器里是不够的：挂接到 Compose 网络的服务端通常无法路由到创建在 Docker 独立默认网桥上的沙箱。
- 上图展示了路由路径：宿主机流量命中代理端口，execd 将请求改写并指向目标容器端口，上游服务保持在沙箱内隔离。

## 网络模式

### Host 网络模式（单机约束）
- 容器共享宿主机网络栈（`network_mode=host`），因此沙箱端口在宿主机上直接可访问。
- 由于每个沙箱都在宿主机上绑定自己的端口，除非你为每个沙箱预留专用端口，否则该模式实际上限制每台主机只能运行一个沙箱实例。
- `get_endpoint(..., port=X)` 返回 `{public_host}:{X}`，没有 `/proxy/` 前缀，因此调用者需要知道确切的宿主机端口，且宿主机必须为每个沙箱端口管理防火墙规则。

### Bridge 网络模式（单机部署的默认值）
- Docker 将沙箱置于一个隔离的桥接网络上，使容器端口在未显式映射时不可达。
- 为了在单机上扩展，OpenSandbox 只映射 execd 的代理端口（`44772`），以及可选的端口 `8080`。任何其他容器端口保持私有，并通过代理访问。
- 反向代理标签（`opensandbox.io/embedding-proxy-port`）标识一个位于 `execd` 之前的宿主机端口。`get_endpoint(..., port=X)` 返回 `{public_host}:{host_proxy_port}/proxy/{X}`，因此所有内部端口可以共享同一个宿主机绑定。
- 端口 `8080` 也可以获得直接的宿主机绑定（`opensandbox.io/http-port`），在需要时提供一个不带代理路径的常规 HTTP 端点。
- 这种 bridge 设置让单机可以承载大量沙箱而不产生端口冲突，因为同一个宿主机代理端口可以为 HTTP、SSE、WebSocket、VNC 等复用请求。

### 用户态网络模式（podman：`pasta`、`slirp4netns`）
- 一些容器运行时可以把容器挂接到用户态网络命名空间，而不是桥接网络。Podman 将其用于 rootless 容器，而在桥接网络不可用的主机上（例如 netavark 无法应用其 nftables 规则集的 rootless podman machine），`network_mode = "pasta"` 是实用的选择。
- 这些模式**不涉及**桥接，因此不存在可从宿主机路由的容器 IP：Docker 兼容的 inspect 会报告空的 `NetworkSettings.IPAddress`，且没有 `NetworkSettings.Networks` 条目。宿主机只能通过已发布的端口访问沙箱。
- 因此 `get_endpoint(..., resolve_internal=True)` 在这些模式下返回服务端本地的宿主机映射端点（`{proxy_host}:{host_proxy_port}/proxy/{X}`）——与共享 egress sidecar 网络命名空间的沙箱使用相同的回退——而不是以 `NETWORK_MODE_ENDPOINT_UNAVAILABLE` 失败。
- 其余一切不变：`host` / `bridge` / `none` 和用户自定义网络保持既有行为，且 `networkPolicy`（egress sidecar）仍要求 `bridge`。

## 容器化的生命周期服务端

当生命周期服务端运行在 Compose 网络上、并通过挂载的 Docker socket 在 Docker 默认网桥上创建沙箱时，请通过宿主机发布的端口路由服务端代理流量：

```toml
[server]
host = "0.0.0.0"

[proxy]
resolve_internal = false

[docker]
network_mode = "bridge"
host_ip = "host.docker.internal"
```

服务端容器必须能将 `host.docker.internal` 解析到 Docker 宿主机。在 Linux 上，为它的 Compose 服务添加 `extra_hosts: ["host.docker.internal:host-gateway"]`，如[Compose 示例](https://github.com/opensandbox-group/OpenSandbox/blob/main/server/docker-compose.example.yaml)所示。此后 HTTP 和 WebSocket 代理请求都会使用宿主机映射的端点。已配置的 `[server].eip` 仍然是公共端点地址；服务端侧的代理改用本地可达的宿主机。

::: warning 升级旧版服务端镜像
旧的 `server/v0.2.3` 版本在两条代理路径中都硬编码了内网 IP 解析。仅设置 `resolve_internal = false` 无法修复该版本：即使沙箱的宿主机映射 `/ping` 端点可以响应，SDK 就绪检查仍可能超时。

请使用 `opensandbox/server:release-1.1.0`，其中包含了可配置的代理解析和本地代理主机修复。OpenSandbox 现在使用[统一发布标签](/community/versioning)；冻结的 `server/v*` 标签不再标识更新的服务端版本。Compose 示例显式固定了修复后的镜像，而不是依赖 `latest`。
:::

更新镜像和配置后，拉取并重建服务端容器。在仓库根目录下：

```bash
docker compose -f server/docker-compose.example.yaml pull opensandbox-server
docker compose -f server/docker-compose.example.yaml up -d --force-recreate opensandbox-server
```

服务端启动后，通过服务端代理重试 SDK 就绪检查。

## 运维说明
- 如果 execd 的代理端口（`44772`）或可选的 `8080` 宿主机映射缺失，`get_endpoint` 会以 HTTP 500 响应，并附带一条说明哪个映射不可用的消息。
- 在基于浏览器的客户端或 SDK 中嵌入 URL 时，务必保留 `/proxy/{port}` 前缀（包括任何附加路径或查询字符串），以便 execd 能正确分发请求。
- 这种基于代理的方式意味着额外的端口无需发布到宿主机上，简化了防火墙管理并提升了安全性。
- 当生命周期服务端运行在 Docker 中并挂载了宿主机 socket 时，设置 `[proxy] resolve_internal = false`，配置 `[docker] host_ip`，并确保该主机名可以从服务端容器解析到 Docker 宿主机。仓库中的 `server/docker-compose.example.yaml` 演示了这一拓扑。
