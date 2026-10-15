---
title: Kubernetes 部署
description: 在 Kubernetes 上部署 OpenSandbox —— CRD、controller、生命周期服务端及可选组件，包含 Helm 部署顺序与配置。
---

# Kubernetes 部署

本指南介绍在 Kubernetes 上部署 OpenSandbox：集群基础（CRD）、controller、生命周期服务端，以及可选组件（ingress 网关、节点 agent、fast-sandbox 运行时）。

Chart 是 [OpenSandbox 仓库](https://github.com/opensandbox-group/OpenSandbox/tree/main/manifests/charts)中的版本化源码，需从 `release-X.Y.Z` 标签的检出中安装——官方不发布独立的 chart 包。

## 前提条件

- Kubernetes 1.21.1+
- Helm 3.x
- 已针对你的集群配置好的 `kubectl`

## 部署顺序

按以下顺序安装 chart：

```text
base → opensandbox-controller → fast-sandbox* → ingress-gateway → opensandbox-server → 可选组件
```

\* `fast-sandbox` 为可选组件，但一旦启用，必须在 ingress 网关和服务端**之前**安装。Kubernetes 部署必须安装 `ingress-gateway`——沙箱 Pod 仅有 ClusterIP，客户端流量需经网关路由。它要安装在**服务端之前**，以便服务端从首次安装起就公告网关。

| 步骤 | Chart | 为何安排在此处 |
|------|-------|-------------------|
| 1 | `base` | 拥有 `sandbox.opensandbox.io` 的 CRD、fast-sandbox 的 CRD（`sandbox.fast.io`）及其 RBAC。其余一切都依赖这些对象存在。每个集群安装一次。 |
| 2 | `opensandbox-controller` | 调和你（用户）和服务端创建的 `BatchSandbox`、`Pool`、`SandboxSnapshot` 对象。依赖 `base` 提供的 CRD。 |
| 3 | `fast-sandbox`（可选） | Firecracker 运行时（`sandbox.fast.io`）。必须安装在**服务端之前**：服务端的 `[runtime]`/fsb 配置指向 FastPath gRPC 端点（`fast-sandbox-fastpath...svc:9090`），并在启动时 watch `sandbox.fast.io` 对象。它还消费 `base` 提供的 ServiceAccounts——命名空间 values 需与其保持一致。如果只使用默认 Kubernetes 运行时，可跳过。 |
| 4 | `ingress-gateway` | Kubernetes 部署必需：沙箱 Pod 仅有 ClusterIP，客户端流量经网关路由。安装在服务端之前，以便从首次安装起就完成公告配置。通过 fsb 运行时服务沙箱时，需将 `gateway.fastpathEndpoint` 指向上一步的 FastPath 服务。 |
| 5 | `opensandbox-server` | 创建和删除沙箱的生命周期 REST API。依赖 `base` 的 CRD、运行中的 controller，以及（通过 fsb 运行时服务沙箱时）fast-sandbox 发布的 FastPath 端点。通过 `server.gateway.*` 公告 ingress 网关。 |
| 6 | `opensandbox-node-agent`（可选） | 节点级沙箱数据采集。与顺序无关。 |

如果使用 umbrella chart，上述顺序会在一次发布中自动处理。

## 安装

检出你要部署的版本：

```bash
git clone https://github.com/opensandbox-group/OpenSandbox.git
cd OpenSandbox
git checkout release-1.1.0   # 开发环境可用 main
```

### 方式一：Umbrella chart（推荐）

一次发布即按正确顺序安装所有组件：

```bash
cd manifests/charts

# 打包子 chart（charts/ 已被 git 忽略，每次都会重新构建）
helm dependency build opensandbox

helm install opensandbox opensandbox \
  --namespace opensandbox-system \
  --create-namespace
```

可选组件默认关闭。对于 Kubernetes 部署，请启用 ingress 网关——沙箱 Pod 仅有 ClusterIP，客户端流量需经其路由：

```bash
helm install opensandbox opensandbox \
  --namespace opensandbox-system \
  --create-namespace \
  --set ingress-gateway.enabled=true
```

### 方式二：按组件发布

```bash
# 1. 集群基础：CRD + RBAC
helm install base manifests/charts/base

# 2. Controller
helm install opensandbox-controller manifests/charts/controller \
  --namespace opensandbox-system \
  --create-namespace

# 3. Ingress 网关（Kubernetes 上必需；见部署顺序）
helm install ingress-gateway manifests/charts/ingress-gateway \
  --namespace opensandbox-system

# 4. 生命周期服务端
helm install opensandbox-server manifests/charts/server \
  --namespace opensandbox-system \
  --create-namespace
```

### 配置 API 认证

默认情况下，在非交互式容器中缺少 API Key 时服务端会拒绝启动。先创建控制面命名空间和默认沙箱工作负载命名空间，再将 Key 存入 Kubernetes `Secret`：

```bash
kubectl create namespace opensandbox-system --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace opensandbox --dry-run=client -o yaml | kubectl apply -f -

read -s OPENSANDBOX_API_KEY
kubectl create secret generic opensandbox-api-key \
  --namespace opensandbox-system \
  --from-literal=api-key="${OPENSANDBOX_API_KEY}" \
  --dry-run=client -o yaml | kubectl apply -f -
unset OPENSANDBOX_API_KEY
```

在 values 文件中引用该 Secret：

```yaml
# values-server.yaml
server:
  replicaCount: 1
  env:
    - name: OPENSANDBOX_SERVER_API_KEY
      valueFrom:
        secretKeyRef:
          name: opensandbox-api-key
          key: api-key
```

生产环境建议使用外部密钥管理系统，而非手动创建 Secret。

chart 会将服务端安装到 `opensandbox-system`，而默认的 `configToml` 在 `opensandbox` 命名空间中创建沙箱和 pool 资源。如果你修改了 `configToml` 中的 `[kubernetes].namespace`，请在提交工作负载前创建对应命名空间而非 `opensandbox`。

::: warning 默认单活跃 Server
chart 默认 `server.replicaCount: 1`。请保持仅一个活跃的生命周期服务端，除非你有意采用下文介绍的、基于 PostgreSQL 的 Kubernetes 公共快照拓扑。该例外只协调公共快照，并不提供通用的多副本 Server 高可用。Server Deployment 使用 `Recreate` 策略，因此升级时会先停止活跃 Server 再启动替代实例；升级期间预计会出现短暂 API 中断。
:::

### 使用 PostgreSQL 作为服务端持久化

创建包含 PostgreSQL 连接串的 Secret：

```bash
read -s OPENSANDBOX_POSTGRESQL_DSN
kubectl create secret generic opensandbox-postgresql \
  --namespace opensandbox-system \
  --from-literal=dsn="${OPENSANDBOX_POSTGRESQL_DSN}" \
  --dry-run=client -o yaml | kubectl apply -f -
unset OPENSANDBOX_POSTGRESQL_DSN
```

在 `values-server.yaml` 中，保持默认的 `server.replicaCount` 为 `1`，添加下方基于 Secret 的环境变量，并将所示 `[store]` 表配置补充到完整的 `configToml` 值中：

```yaml
server:
  replicaCount: 1
  env:
    - name: OPENSANDBOX_STORE_POSTGRESQL_DSN
      valueFrom:
        secretKeyRef:
          name: opensandbox-postgresql
          key: dsn

configToml: |
  # 此处保留 chart 完整服务端配置的其余部分。
  [store]
  type = "postgresql"

  [store.postgresql]
  min_pool_size = 1
  max_pool_size = 10
  snapshot_recovery_interval_seconds = 15
```

::: info
chart 默认仍为单 Server 副本。只有当两个副本都使用同一 PostgreSQL 数据库和 Kubernetes 运行时时，才可为多活跃公共快照处理显式设置 `server.replicaCount: 2`。SQLite 和 Docker 快照执行不支持这种多活跃拓扑。
:::

### 安装服务端（含 API Key）并验证

引用你的 values 文件安装服务端：

```bash
helm install opensandbox-server manifests/charts/server \
  --namespace opensandbox-system \
  --create-namespace \
  --values values-server.yaml
```

等待 Deployment 就绪并验证 API 健康端点：

```bash
kubectl rollout status deployment/opensandbox-server \
  --namespace opensandbox-system \
  --timeout=180s

kubectl port-forward \
  --namespace opensandbox-system \
  service/opensandbox-server 8080:80
```

在另一个终端中：

```bash
curl --fail http://127.0.0.1:8080/health
```

### 关键 values

| Value | 用途 | 说明 |
|-------|---------|-------|
| `server.image.repository` | 服务端镜像的 registry 与仓库 | 使用私有镜像源或自定义构建时覆盖。 |
| `server.image.tag` | 服务端镜像版本 | 默认为随 chart 版本一起发布的 `release-<appVersion>` 镜像；仅自定义构建时覆盖。 |
| `server.replicaCount` | 服务端 Pod 数量 | 默认 `1`；暂不支持通用的多副本 Server 高可用。 |
| `server.env` | 额外的容器环境变量 | 配合 `secretKeyRef` 用于 `OPENSANDBOX_SERVER_API_KEY`。 |
| `configToml` | 完整的服务端配置 | 挂载到 `/etc/opensandbox/config.toml`；覆盖它会替换整个默认 TOML，包括工作负载命名空间。 |
| `server.gateway.enabled` | 向客户端公告 ingress 网关 | 默认 `false`。网关本身由 [ingress-gateway chart](https://github.com/opensandbox-group/OpenSandbox/tree/main/manifests/charts/ingress-gateway) 部署。 |
| `server.service.type` | 服务端 Service 类型 | 默认 `ClusterIP`。需从集群外部访问时使用 `NodePort` 或 `LoadBalancer`，并用 `server.service.nodePort` 固定端口。 |
| `namespaceOverride` | chart 资源使用的命名空间 | 默认 `opensandbox-system`。 |

服务端容器及其 Service 使用端口 `80`。替换 `configToml` 时保持 `[server].port = 80`，除非 chart 模板也已同步改为其他端口。Service 默认为 `ClusterIP`；需从集群外部访问服务端时设置 `server.service.type`。

### 配置 egress sidecar 资源

当创建请求包含 `networkPolicy` 时，生命周期服务端会为每个非池化的沙箱 Pod 添加一个 egress sidecar。该容器未声明资源时，会应用命名空间的 `LimitRange` 默认值，这可能预留远超基本 DNS/nft 执行所需的容量。

在 `configToml` 的 `[egress]` 配置段中添加可选的资源设置：

```toml
[egress]
image = "opensandbox/egress:release-1.1.0"
requests = { cpu = "25m", memory = "64Mi" }
limits = { cpu = "250m", memory = "256Mi" }
```

`requests` 和 `limits` 可任选其一省略。请将这些值作为起点，并根据实际观测用量进行调优；Credential Vault 和透明 mitmproxy 通常比基本 DNS/nft 执行需要更多余量。

### Ingress 网关（Kubernetes 上必需）

Kubernetes 上的沙箱 Pod 仅有 ClusterIP——客户端流量需经 ingress 网关到达沙箱（`[ingress] mode = "gateway"`，即面向 Kubernetes 的模式；Docker 运行时使用 `direct`，无需网关）。先安装网关（**在服务端之前**），再在服务端安装中包含公告配置：

```bash
helm install ingress-gateway manifests/charts/ingress-gateway \
  --namespace opensandbox-system \
  --set gateway.fastpathEndpoint=fast-sandbox-fastpath.opensandbox-system.svc:9090
```

启用公告安装服务端（或对已有 server release 执行带相同参数的 `helm upgrade`）：

```bash
helm install opensandbox-server manifests/charts/server \
  --namespace opensandbox-system \
  --set server.gateway.enabled=true \
  --set server.gateway.host=gateway.example.com \
  --values values-server.yaml
```

保持 `server.gateway.gatewayRouteMode` 与网关 chart 的 `gateway.gatewayRouteMode` 一致。对于带签名和过期时间的沙箱路由，需在两个 chart 上配置共享的 secure-access 密钥环——参见 [secure-access keys](https://github.com/opensandbox-group/OpenSandbox/blob/main/manifests/HELM-DEPLOYMENT.md#secure-access-keys-osep-0011)。

### 可选：fast-sandbox 运行时

`fast-sandbox` chart 提供 Firecracker（`sandbox.fast.io`）运行时。需将其安装在 **ingress 网关和生命周期服务端之前**（参见[部署顺序](#deployment-order)）：服务端的 `[runtime]`/fsb 配置指向此 chart 创建的 FastPath gRPC 端点。它还要求先安装 `base`、具备 KVM 能力的节点，以及基于固定上游 commit 构建的配套镜像——参见 [fast-sandbox runtime 部署指南](https://github.com/opensandbox-group/OpenSandbox/blob/main/manifests/HELM-DEPLOYMENT.md#fast-sandbox-runtime-firecracker)。

## 升级

从更新的检出升级 umbrella release：

```bash
git fetch --tags
git checkout release-1.2.0

cd manifests/charts
helm dependency build opensandbox
helm upgrade opensandbox opensandbox --namespace opensandbox-system
```

对于按组件发布的 server：

```bash
git checkout release-1.2.0
helm upgrade opensandbox-server manifests/charts/server \
  --namespace opensandbox-system \
  --values values-server.yaml
```

完整的 values 参考和本地开发安装，请参见 [`opensandbox-server` chart README](https://github.com/opensandbox-group/OpenSandbox/tree/main/manifests/charts/server)。

## Operator 指标

operator（controller-manager）暴露标准的 [controller-runtime](https://book.kubebuilder.io/reference/metrics) Prometheus 指标——reconcile 速率与延迟（`controller_runtime_reconcile_*`）、工作队列深度、client-go 请求计数以及 Go 运行时统计。该端点**默认关闭**（`--metrics-bind-address=0`）。

通过 `opensandbox-controller` chart values 启用：

| Value | 默认值 | 用途 |
|-------|---------|---------|
| `controller.metrics.enabled` | `false` | 暴露 `/metrics` 端点（设置 `--metrics-bind-address`） |
| `controller.metrics.port` | `8080` | 指标端点的端口 |
| `controller.metrics.secure` | `false` | 通过 HTTPS 提供，带 authn/authz（`--metrics-secure`）；设为 `false` 则使用纯 HTTP 抓取 |

```yaml
controller:
  metrics:
    enabled: true
    port: 8080
    secure: false   # 纯 HTTP，例如供 PodMonitoring/ServiceMonitor 抓取
```

- `secure: false` 时端点为纯 HTTP，可直接抓取（无需 TLS 或 bearer token）。
- `secure: true` 时 controller-runtime 过滤器通过 `TokenReview`/`SubjectAccessReview` 对每次抓取进行认证和授权。chart 会自动创建两个 `ClusterRole`：
  - `opensandbox-metrics-auth-role`（绑定到 manager）——允许 controller 执行认证检查。
  - `opensandbox-metrics-reader`（chart **不会**自动绑定）——授予对 `/metrics` 非资源 URL 的 `get` 权限。需将其绑定到你抓取器（如 Prometheus）的 `ServiceAccount`，并让抓取器出示该账号的 bearer token。

将你的 Prometheus 栈指向 `metrics` 容器端口（例如通过 `ServiceMonitor` 或 `PodMonitoring`）。

### 业务容量指标

当选出的 controller 检测到设置了 `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` 或 `OTEL_EXPORTER_OTLP_ENDPOINT` 时，还会通过 OTLP/HTTP 导出低基数的业务容量指标。这独立于 controller-runtime 的 Prometheus 端点；两个变量均未配置时保持关闭。

```yaml
extraEnv:
  - name: OTEL_EXPORTER_OTLP_METRICS_ENDPOINT
    value: http://otel-collector.observability:4318/v1/metrics
```

| 指标 | 单位 | 属性 | 说明 |
|--------|------|------------|-------------|
| `controller.pool.pods` | `{pod}` | `namespace`、`pool_name`、`state` | 当前 Pool Pod 数量，`state` 为 `total`、`allocated`、`available` 或 `updated` |
| `controller.pool.cpu.requested` | `{cpu}` | `namespace`、`pool_name`、`state` | 由 total、allocated 或 available 的 Pool Pod 代表的、与调度器口径一致的 CPU request |
| `controller.pool.memory.requested` | `By` | `namespace`、`pool_name`、`state` | 由 total、allocated 或 available 的 Pool Pod 代表的、与调度器口径一致的内存 request |
| `controller.batchsandbox.count` | `{batchsandbox}` | `namespace`、`phase`、`allocation_mode` | 按生命周期阶段及 pool/direct 模式统计的当前 BatchSandbox 对象数量 |
| `controller.batchsandbox.pods` | `{pod}` | `namespace`、`state`、`allocation_mode` | BatchSandbox Pod 的 desired、current、allocated 和 ready 数量 |
| `controller.capacity.collect.duration` | `s` | 无 | 读取缓存对象并采集一次容量快照所花费的时间 |

这些指标有意省略沙箱、BatchSandbox 和 Pod 标识。只有 leader 会导出它们，因此多个 controller 副本不会重复统计集群总量。BatchSandbox 初始阶段未设置时导出为 `Unknown`。Pool 利用率可由 `allocated / total` 推导，峰值、谷值或百分位容量可在遥测后端中计算。实际的 CPU 和内存用量仍可从 kubelet/cAdvisor 获取，此处不做重复。

每次采集都会从 controller manager 的 informer 缓存中读取所有 Pool 和 BatchSandbox，然后对每个非删除状态的 Pool 执行一次基于 owner-UID 索引的缓存 Pod 列表查询。因此采集的 CPU 与内存开销随缓存中的 Pool、BatchSandbox 及 Pool 拥有的 Pod 数量线性增长，而不会对每个 Pool 都发起一次 API server 的 list 请求。OpenTelemetry 周期性 reader 默认每 60 秒导出一次；可通过 `OTEL_METRIC_EXPORT_INTERVAL`（毫秒）修改该间隔。缩短该间隔前，请监控 `controller.capacity.collect.duration` 并验证目标集群的规模。在配置了端点后若 OTLP 初始化失败，controller 会继续执行 reconcile，并记录失败的初始化阶段、相关端点环境变量以及已剥离凭证的端点信息。

## 为 Kubernetes 配置服务端

生成面向 Kubernetes 的服务端配置：

```bash
opensandbox-server init-config ~/.sandbox.toml --example k8s
```

Kubernetes 专属的关键配置段：

| 配置段 | 用途 |
|---------|---------|
| `[kubernetes]` | 工作负载 provider、BatchSandbox 模板文件 |
| `[agent_sandbox]` | Agent Sandbox 设置 |
| `[ingress]` | 用于沙箱流量路由的 ingress 网关 |
| `[secure_runtime]` | 安全容器运行时（gVisor、Kata） |

完整参考请参见[配置](/getting-started/configuration)。

## Kubernetes 上的组件

| 组件 | 部署方式 | 用途 |
|-----------|-----------|---------|
| CRD + RBAC（`base`） | 集群级 | `BatchSandbox`、`Pool`、`SandboxSnapshot` 及 `sandbox.fast.io` API 类型 |
| Server | Deployment | 生命周期控制面 |
| Controller（operator） | Deployment | 管理 BatchSandbox/Pool CRD |
| Ingress 网关 | Deployment | 将流量路由到沙箱；Kubernetes 部署必需 |
| Egress | Sidecar | 每沙箱的 egress 策略执行 |
| Execd | 内置于沙箱镜像 | 沙箱内执行 |
| Node agent | DaemonSet | 可选的节点级沙箱数据采集 |
| fast-sandbox | Deployment + DaemonSet | 可选的 Firecracker 控制面与节点运行时 |

## 卸载

```bash
helm uninstall opensandbox -n opensandbox-system
```

CRD 带有 `helm.sh/resource-policy: keep` 注解，会在卸载后保留；`opensandbox-dataplane` 命名空间（由 `base` 为 fast-sandbox 创建）同样会保留。当其数据不再需要时，可手动删除：

```bash
kubectl delete crd batchsandboxes.sandbox.opensandbox.io \
  pools.sandbox.opensandbox.io \
  sandboxsnapshots.sandbox.opensandbox.io
kubectl delete namespace opensandbox-dataplane
```

## 相关内容

- [Kubernetes 概述](/architecture/control-plane/operator) — Operator 特性与 CRD
- [暂停与恢复](/guides/pause-resume) — Kubernetes 上基于快照的暂停/恢复
- [安全容器](/guides/secure-container) — Kubernetes 上的 gVisor 与 Kata
- [Kubernetes 上的网络隔离](/guides/network-isolation) — 面向 Kubernetes 的 egress 策略设计
- [Helm charts 源码](https://github.com/opensandbox-group/OpenSandbox/tree/main/manifests/charts) — chart 源码与 values 参考
