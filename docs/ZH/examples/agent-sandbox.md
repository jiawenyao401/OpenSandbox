---
title: Agent Sandbox
description: 创建一个 kubernetes-sigs/agent-sandbox 实例，并使用 OpenSandbox Python SDK 运行一条命令。
---

# Agent-Sandbox 示例

本示例创建一个由 `kubernetes-sigs/agent-sandbox` 支持的沙箱，并通过 OpenSandbox Python SDK 执行 `echo hello world`。

## 前提条件

- 一个已安装 agent-sandbox >= v0.5.0 控制器和 CRD 并提供 `agents.x-k8s.io/v1beta1` 的 Kubernetes 集群。
- OpenSandbox 服务器已配置 Kubernetes 运行时，且 `workload_provider = "agent-sandbox"`。
- 沙箱镜像应包含 `bash`（默认示例使用 `ubuntu:22.04`）。

## 兼容性与迁移

服务器 provider 和 ingress informer 仅使用 `agents.x-k8s.io/v1beta1`。
该 API 自 agent-sandbox v0.5.0 起可用；v1.0.0 移除了
`v1alpha1`。使用此集成之前，请先升级较旧的控制器和 CRD 安装；
不存在 `v1alpha1` 回退路径。

对于已有安装，首先将 agent-sandbox 升级到同时提供两个版本的发行版
（例如 v0.5.0），并确保其转换 webhook 健康运行，然后
将 OpenSandbox 服务器和 ingress 一起升级。在安装移除
alpha API 的 agent-sandbox 发行版之前，请先完成上游的
资源/存储迁移。

更新自定义 AgentSandbox 模板以使用 `apiVersion: agents.x-k8s.io/v1beta1`，
并移除 `spec.replicas`。服务器会设置 `spec.operatingMode: Running` 和
`spec.service: true`，包括在合并自定义模板时也是如此。必须创建 Service
才能通过 `status.serviceFQDN` 进行 ingress 路由。

服务器在创建新沙箱时应用这些字段；它不会重写
既有资源。在 v0.5.0 上，alpha 到 beta 的转换会保留被省略的
`service` 字段。如果既有沙箱没有 Service/FQDN，请在该资源上显式启用
`spec.service: true`，并等待 `Ready=True` 以及 `status.serviceFQDN`
被填充，然后再使用 ingress。这样可以无需重建 Sandbox 即可恢复路由。
对于有意不带 Service 的资源，请保持不变。

已配置的 `shutdownPolicy`、Pod 模板合并以及绝对 `shutdownTime`
语义均保持不变。没有过期时间的请求会省略 `shutdownTime`，
包括模板提供的任何值。状态和端点解析仍然
使用 `Ready`、`selector`、`podIPs` 和 `serviceFQDN`。

OpenSandbox 自身的 BatchSandbox API 仍为 `sandbox.opensandbox.io/v1alpha1`。

## 暂停与恢复

该 provider 支持标准的 `POST /v1/sandboxes/{id}/pause` 和
`POST /v1/sandboxes/{id}/resume` 端点。两者都通过修补
`Sandbox` CRD 的 `spec.operatingMode` 字段来实现。

暂停会将 `spec.operatingMode` 修补为 `Suspended`。agent-sandbox 控制器
会终止并销毁 Pod，但保留 `Sandbox` 对象，因此
`sandboxId` 在暂停/恢复周期内保持稳定。沙箱状态依次为
`Running` → `Pausing` → `Paused`；当 Pod 正在终止时，状态
`reason` 来自 `Suspended` 条件（例如 `PodTerminating`）。

恢复会将 `spec.operatingMode` 修补为 `Running`。控制器
根据 Sandbox 规格重建 Pod，因此在沙箱再次可用之前，
预计会有调度和镜像拉取延迟。沙箱状态依次为 `Paused` → `Resuming` →
`Pending` → `Running`。

| | 是否保留？ |
|--|-----------|
| 持久卷（PVC）数据 | ✅ 是 — 卷随 Sandbox 对象一起保留 |
| `sandboxId` 和沙箱元数据 | ✅ 是 |
| 内存 / 运行中的进程 | ❌ 否 — Pod 已被销毁 |
| 根文件系统运行时状态 | ❌ 否 — 恢复时 Pod 会从原始镜像重新创建 |

任何未写入持久卷的数据在暂停后都会丢失。

已暂停的沙箱仍会在其原始超时时间到期；一个保留但已过期的
沙箱会报告 `Terminated`，且无法再被暂停或恢复。

## 启动 OpenSandbox 服务器

1. 安装服务器软件包并获取 agent-sandbox 的示例配置：

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
```

2. 用以下配置段更新 `~/.sandbox.toml`：

```toml
[runtime]
type = "kubernetes"
execd_image = "opensandbox/execd:v1.1.0"

[kubernetes]
namespace = "default"
# kubeconfig_path = "/absolute/path/to/kubeconfig"  # optional if running in-cluster
workload_provider = "agent-sandbox"

[agent_sandbox]
shutdown_policy = "Delete"
```

3. 启动服务器：

```shell
opensandbox-server
```

## 运行示例

```shell
uv pip install opensandbox
uv run python examples/agent-sandbox/main.py
```

## 预期输出

```text
command output: hello world
```

## 参考资料

- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/agent-sandbox)
