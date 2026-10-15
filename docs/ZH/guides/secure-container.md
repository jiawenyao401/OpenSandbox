---
title: 安全容器运行时
description: 使用 gVisor、Kata Containers 和 Firecracker microVM 实现硬件级沙箱隔离。
---

# 安全容器运行时指南

本指南介绍如何在 OpenSandbox 中使用安全容器运行时，为执行不可信的 AI 生成代码提供硬件级隔离。

## 目录

- [概述](#概述)
- [Server 配置](#server-配置)
- [Kubernetes 上的多种运行时配置（Profile）](#kubernetes-上的多种运行时配置profile)
- [Docker 模式](#docker-模式)
- [Kubernetes 模式](#kubernetes-模式)
- [用户指南](#用户指南)
- [管理员指南](#管理员指南)
- [故障排除与最佳实践](#故障排除与最佳实践)

---

## 概述

### 什么是安全容器运行时？

安全容器运行时比 Docker 和 containerd 使用的标准 runc 运行时提供更强的隔离。它们通过不同机制叠加额外的安全层：

| 运行时 | 隔离机制 | 启动开销 | 内存开销 | 最适合 |
|---------|---------------------|------------------|-----------------|----------|
| **runc**（默认） | 进程级 cgroups | ~0ms | 极小 | 可信工作负载、本地开发 |
| **gVisor** | 用户态内核（syscall 拦截） | ~10-50ms | ~50MB | 低开销的通用工作负载 |
| **Kata (QEMU)** | 基于 QEMU hypervisor 的完整 VM | ~500ms | ~20-50MB | 最大兼容性和隔离性 |
| **Kata (Firecracker)** | 基于 Firecracker hypervisor 的 microVM | ~125ms | ~5MB | 高密度、占用极小 |
| **Kata (CLH)** | Cloud Hypervisor | ~200ms | ~10-20MB | 性能与隔离兼顾 |

### 为什么要使用安全运行时？

OpenSandbox 的设计目标是执行 AI 模型（Claude、GPT-4、Gemini 等）生成的不可信代码。安全运行时提供：

1. **容器逃逸防护**：阻止恶意代码逃出容器
2. **内核级隔离**：每个沙箱拥有自己的内核上下文
3. **多租户安全**：不同用户的沙箱被强隔离
4. **合规性**：满足受监管行业的安全要求

### 支持的运行时类型

OpenSandbox 通过 server 级配置支持以下安全运行时类型：

- `"gvisor"` - 使用 runsc 的 Google gVisor
- `"kata"` - 使用 QEMU hypervisor 的 Kata Containers（默认）
- `"firecracker"` - 使用 Firecracker hypervisor 的 Kata Containers
- `""`（空） - 标准 runc（默认，无安全运行时）

### 关键设计原则

**Server 级配置**：安全运行时由管理员在 server 级配置一次，该 server 上的所有沙箱透明地使用配置的运行时。SDK 用户和 API 调用方**无需任何代码修改**。

---

## Server 配置

安全运行时通过 `~/.sandbox.toml` 配置文件配置。server 在启动时校验配置的运行时，如果运行时不可用将拒绝启动。

### 配置文件

编辑 `~/.sandbox.toml`：

```toml
[runtime]
type = "docker"  # or "kubernetes"
execd_image = "opensandbox/execd:latest"

# Secure container runtime configuration
# When enabled, ALL sandboxes on this server use the specified runtime
[secure_runtime]
# Runtime type: "", "gvisor", "kata", "firecracker"
type = ""

# Docker mode: OCI runtime name (e.g., "runsc" for gVisor, "kata-runtime" for Kata)
# Required when runtime.type = "docker" and type is not empty
docker_runtime = "runsc"

# Kubernetes mode: RuntimeClass name (e.g., "gvisor", "kata-qemu", "kata-fc")
# Required when runtime.type = "kubernetes" and type is not empty
k8s_runtime_class = "gvisor"
```

### 配置示例

#### 示例 1：Docker 上的 gVisor

```toml
[runtime]
type = "docker"
execd_image = "opensandbox/execd:latest"

[secure_runtime]
type = "gvisor"
docker_runtime = "runsc"
k8s_runtime_class = "gvisor"
```

#### 示例 2：Kubernetes 上的 Kata Containers

```toml
[runtime]
type = "kubernetes"
execd_image = "opensandbox/execd:latest"

[secure_runtime]
type = "kata"
docker_runtime = "kata-runtime"
k8s_runtime_class = "kata-qemu"
```

#### 示例 3：Kubernetes 上的 Kata + Firecracker

```toml
[runtime]
type = "kubernetes"
execd_image = "opensandbox/execd:latest"

[secure_runtime]
type = "firecracker"
docker_runtime = ""  # Not supported in Docker mode
k8s_runtime_class = "kata-fc"
```

### 启动校验

server 启动时自动校验配置的安全运行时是否可用：

```bash
$ opensandbox-server
INFO     Validating secure runtime for Docker backend
INFO     Docker OCI runtime 'runsc' is available: {...}
INFO     Application startup complete.
```

如果运行时不可用，server 会拒绝启动并给出明确的错误信息：

```
ERROR    Configured Docker runtime 'runsc' is not available.
        Available runtimes: runc.
        Please install and configure it in /etc/docker/daemon.json.
```

---

## Kubernetes 上的多种运行时配置（Profile）

`[secure_runtime]` 是一个 server 启动设置，而不是沙箱创建字段。因此一个 Lifecycle Server 进程会把它创建的每个模板模式沙箱都应用同一个运行时配置。如果要在同一个 Kubernetes 集群中同时提供 runc 和 Kata 这类更强的运行时，请为每个配置运行一个独立寻址的 Lifecycle Server 发行版，并把每个客户端路由到对应的 Server API。

| Server 配置 | API Service | `[secure_runtime]` | 生成的 Pod 字段 |
|----------------|-------------|--------------------|---------------------|
| 标准 | `opensandbox-runc` | `type = ""` | 省略 `runtimeClassName` |
| 隔离 | `opensandbox-kata` | `type = "kata"`, `k8s_runtime_class = "kata-qemu"` | `runtimeClassName: kata-qemu` |

两种配置可以使用同一个 OpenSandbox 控制器和同一个沙箱工作负载命名空间。RuntimeClass 应带有自己的调度约束，使 Kata Pod 只落在安装了 Kata 的节点上。

::: warning 资源名称必须唯一
Server chart 默认把 `fullnameOverride` 设为 `opensandbox-server`，因此只修改 Helm release 名称是不够的：两个默认 release 会渲染出相同的 Deployment、ServiceAccount、RBAC、ConfigMap 和 Service 名称。请为每个配置设置不同的 `fullnameOverride`。独立的 ingress-gateway chart 也默认使用固定的 `fullnameOverride`（`opensandbox-ingress-gateway`）；请为每个网关赋予不同的值，例如 `opensandbox-runc-gateway` 和 `opensandbox-kata-gateway`。其 ClusterRole 和 ClusterRoleBinding 名称由该值派生，必须在集群内唯一。仅分离控制平面命名空间无法避免这些 RBAC 冲突。
:::

### 部署两个 Server 配置

创建 `opensandbox` 工作负载命名空间，并为每个配置创建一个控制平面命名空间。按 [Kubernetes 部署](/deployment/#configure-api-authentication)所述，在对应的控制平面命名空间中创建 API-key Secret。使用 `opensandbox-runc-system` 中的 `opensandbox-runc-api-key` 和 `opensandbox-kata-system` 中的 `opensandbox-kata-api-key`，以匹配下文的引用。

两个 release 都使用下方的 values 文件。它们会完整替换 `configToml`，包括沙箱网络策略所需的 `[egress]` 镜像和模式。示例使用直连沙箱端点：客户端必须能够访问集群内的沙箱 Pod IP。对于外部客户端，请按下文所述配置网关。请使 execd 和 egress 镜像标签与所检出的发行版匹配。

::: code-group

```yaml [values-runc.yaml]
fullnameOverride: opensandbox-runc
namespaceOverride: opensandbox-runc-system

server:
  replicaCount: 1
  gateway:
    enabled: false
  env:
    - name: OPENSANDBOX_SERVER_API_KEY
      valueFrom:
        secretKeyRef:
          name: opensandbox-runc-api-key
          key: api-key

configToml: |
  [server]
  host = "0.0.0.0"
  port = 80
  api_key = ""

  [log]
  level = "INFO"

  [runtime]
  type = "kubernetes"
  execd_image = "sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/execd:release-1.1.1-rc.1"

  [kubernetes]
  namespace = "opensandbox"
  workload_provider = "batchsandbox"

  [egress]
  image = "sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/egress:release-1.1.1-rc.1"
  mode = "dns+nft"

  [secure_runtime]
  type = ""
```

```yaml [values-kata.yaml]
fullnameOverride: opensandbox-kata
namespaceOverride: opensandbox-kata-system

server:
  replicaCount: 1
  gateway:
    enabled: false
  env:
    - name: OPENSANDBOX_SERVER_API_KEY
      valueFrom:
        secretKeyRef:
          name: opensandbox-kata-api-key
          key: api-key

configToml: |
  [server]
  host = "0.0.0.0"
  port = 80
  api_key = ""

  [log]
  level = "INFO"

  [runtime]
  type = "kubernetes"
  execd_image = "sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/execd:release-1.1.1-rc.1"

  [kubernetes]
  namespace = "opensandbox"
  workload_provider = "batchsandbox"

  [egress]
  image = "sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/egress:release-1.1.1-rc.1"
  mode = "dns+nft"

  [secure_runtime]
  type = "kata"
  k8s_runtime_class = "kata-qemu"
```

:::

在一个已安装基础 CRD 和控制器的 OpenSandbox 检出目录中，使用位于 `manifests/charts/server` 的 Server chart 安装两个配置。chart 默认提供与之匹配的 Server 镜像标签：

```bash
kubectl create namespace opensandbox --dry-run=client -o yaml | kubectl apply -f -

helm upgrade --install opensandbox-runc manifests/charts/server \
  --namespace opensandbox-runc-system \
  --create-namespace \
  --values values-runc.yaml

helm upgrade --install opensandbox-kata manifests/charts/server \
  --namespace opensandbox-kata-system \
  --create-namespace \
  --values values-kata.yaml
```

在不同的内部 DNS 名称、主机名或 API 网关路由下暴露两个 Service。不要把它们放在一个无差别的轮询（round-robin）路由后面：运行时的选择就是通过选择 Server 端点完成的。后续的 get、renew、pause、resume、snapshot 和 delete 调用都应保持在创建该沙箱的同一配置端点上。

### 网关访问

`server.gateway.enabled=true` 只是让 Server 声明一个网关地址，并不会部署网关。请为每个配置安装一个独立的 `manifests/charts/ingress-gateway` release，并使用上文不同的 `fullnameOverride` 值。将 `gateway.dataplaneNamespace` 设置为该配置的工作负载命名空间（这些示例中为 `opensandbox`）。

通过各自客户端可访问的主机暴露每个网关，并将匹配的 Server 的 `server.gateway.host` 设置为该主机。使 `server.gateway.gatewayRouteMode` 与网关的 `gateway.gatewayRouteMode` 保持一致。如果启用了签名访问，请按[安全访问](/guides/secure-access)所述，在每个 Server/网关对上配置相同的签名密钥环。

例如，安装两个网关并更新已有的 Server release：

```bash
helm upgrade --install opensandbox-runc-gateway manifests/charts/ingress-gateway \
  --namespace opensandbox-runc-system \
  --set namespaceOverride=opensandbox-runc-system \
  --set fullnameOverride=opensandbox-runc-gateway \
  --set gateway.dataplaneNamespace=opensandbox \
  --set gateway.gatewayRouteMode=header

helm upgrade --install opensandbox-kata-gateway manifests/charts/ingress-gateway \
  --namespace opensandbox-kata-system \
  --set namespaceOverride=opensandbox-kata-system \
  --set fullnameOverride=opensandbox-kata-gateway \
  --set gateway.dataplaneNamespace=opensandbox \
  --set gateway.gatewayRouteMode=header

helm upgrade --install opensandbox-runc manifests/charts/server \
  --namespace opensandbox-runc-system --values values-runc.yaml \
  --set server.gateway.enabled=true \
  --set-string server.gateway.host=runc-gateway.example.com \
  --set server.gateway.gatewayRouteMode=header

helm upgrade --install opensandbox-kata manifests/charts/server \
  --namespace opensandbox-kata-system --values values-kata.yaml \
  --set server.gateway.enabled=true \
  --set-string server.gateway.host=kata-gateway.example.com \
  --set server.gateway.gatewayRouteMode=header
```

网关 chart 使用 `namespaceOverride` 作为其资源命名空间；仅设置 Helm 的 `--namespace` 不会改变该默认值。上述网关 Service 默认为 `ClusterIP`。请通过集群的 ingress 或负载均衡器暴露它们，并在执行 Server 升级之前把 `*.example.com` 主机替换为可访问的地址。网关主机路由沙箱流量；生命周期操作仍使用对应的 Server API 端点。

### 验证配置选择

通过每个 Server 各创建一个沙箱，然后检查生成的 BatchSandbox Pod 模板：

```bash
kubectl get batchsandbox --namespace opensandbox <runc-sandbox-id> \
  -o jsonpath='{.spec.template.spec.runtimeClassName}'
# Empty output: the cluster's default OCI runtime is used.

kubectl get batchsandbox --namespace opensandbox <kata-sandbox-id> \
  -o jsonpath='{.spec.template.spec.runtimeClassName}'
# kata-qemu
```

::: tip Pool 模式
基于 Pool 的沙箱复用已存在的 Pod，因此 Server 的 `[secure_runtime]` 设置不会改写其运行时。请为每个配置创建独立的 Pool，在每个 Pool 上显式设置 `spec.template.spec.runtimeClassName`，并只通过该配置暴露匹配的 `poolRef`。
:::

::: info 运维边界
多种 Server 配置提供确定性的运行时选择，而不是安全或租户边界。面向同一工作负载命名空间的 Server 可以观察到相同的 Kubernetes 资源。当各配置之间也需要相互隔离时，请使用独立的工作负载命名空间（以及范围合适的 RBAC 或独立集群）。分离的网关名称和主机可以避免资源和路由冲突；它们不会限制对共享命名空间中工作负载的访问。
:::

---

## Docker 模式

Docker 模式完全支持安全容器运行时。

### 前提条件

- Docker daemon 已安装并运行
- 安全运行时已安装在主机上

### Docker 上的 gVisor 设置

#### 步骤 1：安装 gVisor runsc

对于 Docker 模式，你只需安装 **runsc** OCI 运行时：

```bash
# Ubuntu/Debian
curl -fsSL https://gvisor.dev/archive.key | sudo gpg --dearmor -o /usr/share/keyrings/gvisor-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/gvisor-archive-keyring.gpg] https://storage.googleapis.com/gvisor/releases release main" | \
  sudo tee /etc/apt/sources.list.d/gvisor.list
sudo apt-get update && sudo apt-get install -y runsc

# Verify installation
runsc --version
```

> **注意**：对于 Docker 模式，只需要 `runsc`。`containerd-shim-runsc-v1` 仅在 Kubernetes/containerd 中需要。
>
> **参考**：其他发行版和安装方法请参见 [gVisor 安装指南](https://gvisor.dev/docs/user_guide/install/)。

#### 步骤 2：配置 Docker daemon

使用 `runsc install` 命令自动配置 Docker daemon：

```bash
sudo runsc install
```

或手动编辑 `/etc/docker/daemon.json`：

```json
{
  "runtimes": {
    "runsc": {
      "path": "/usr/bin/runsc",
      "runtimeArgs": [
        "--platform=systrap",
        "--network=host",
        "--overlay2=none"
      ]
    }
  }
}
```

::: warning Docker 快照需要 `--overlay2=none`
runsc 通常把根文件系统更改保存在自己的 `root:self` overlay 中。Docker 无法在 `docker commit` 中包含这些更改，因此基于 Docker 的 OpenSandbox 快照可能报告成功，但恢复出的只是原始镜像。当你使用 snapshot、fork 或 restore 工作流时，请在运行时参数中保留 `--overlay2=none`。这样沙箱的写入对 Docker 的存储驱动可见，因此能进入快照镜像。

未带此参数创建的现有 runsc 安装，必须先更新 `daemon.json` 并重启 Docker，才能进行快照。根文件系统 overlay 行为请参见 gVisor 的[文件系统文档](https://gvisor.dev/docs/user_guide/filesystem/)。
:::

重启 Docker：

```bash
sudo systemctl restart docker
```

> **参考**：更多细节请参见 [gVisor Docker 快速入门](https://gvisor.dev/docs/user_guide/quick_start/docker/)。

#### 步骤 3：配置 OpenSandbox Server

编辑 `~/.sandbox.toml`：

```toml
[runtime]
type = "docker"
execd_image = "opensandbox/execd:latest"

[secure_runtime]
type = "gvisor"
docker_runtime = "runsc"
```

#### 步骤 4：启动 Server 并验证

```bash
opensandbox-server
```

创建一个测试沙箱：

```bash
curl -X POST http://localhost:8080/v1/sandboxes \
  -H "Content-Type: application/json" \
  -d '{
    "image": {"uri": "python:3.11"},
    "timeout": 3600,
    "resourceLimits": {"cpu": "500m", "memory": "512Mi"},
    "entrypoint": ["python", "-u", "-c", "import time\nwhile True: print('hello from gVisor!'); time.sleep(1)"],
    "metadata": {
      "name": "gvisor-docker-sandbox"
    }
  }'
```

验证运行时：

```bash
docker ps --format "{{.ID}}\t{{.Image}}\t{{.Names}}"
docker inspect <container_id> | grep -A2 Runtime
# Expected output:
# "Runtime": "runsc",
```

### Docker 上的 Kata Containers 设置

#### 系统要求

Kata Containers 需要硬件虚拟化支持。请验证系统满足以下要求：

**硬件虚拟化支持：**
```bash
# Check if CPU supports hardware virtualization (VT-x for Intel, AMD-V for AMD)
lscpu | grep Virtualization
# Expected output: Virtualization: VT-x (Intel) or AMD-V (AMD)

# Alternatively on Intel
grep -E --color=auto 'vmx|svm' /proc/cpuinfo
# Expected: vmx (Intel) or svm (AMD) flags present
```

**KVM 模块：**
```bash
# Check if KVM module is loaded
lsmod | grep kvm
# Expected: kvm_intel (Intel) or kvm_amd (AMD)

# If not loaded, load KVM module
sudo modprobe kvm_intel  # For Intel
# or
sudo modprobe kvm_amd    # For AMD
```

**内核要求：**
- 推荐 Linux 内核 5.10 或更高
- 内核配置中启用 KVM

**Docker 要求：**
- Docker 20.10 或更高
- `/etc/docker/daemon.json` 已为 Kata 运行时配置

#### 安装

从 GitHub releases 下载并安装 Kata Containers 静态二进制：

```bash
# Find the latest release at https://github.com/kata-containers/kata-containers/releases
KATA_VERSION="3.27.0"
wget https://github.com/kata-containers/kata-containers/releases/download/${KATA_VERSION}/kata-static-${KATA_VERSION}-amd64.tar.zst

# Extract to root directory - Kata will be installed in /opt/kata
zstd -d kata-static-${KATA_VERSION}-amd64.tar.zst
tar -xvf kata-static-${KATA_VERSION}-amd64.tar -C /

# Create symbolic links for PATH access
sudo ln -sf /opt/kata/bin/kata-runtime /usr/local/bin/kata-runtime
sudo ln -sf /opt/kata/bin/containerd-shim-kata-v2 /usr/local/bin/containerd-shim-kata-v2

# Verify installation
kata-runtime --version
```

#### 配置 Docker Daemon

编辑 `/etc/docker/daemon.json`，将 Kata 注册为运行时：

```json
{
  "default-runtime": "runc",
  "runtimes": {
    "kata": {
      "runtimeType": "io.containerd.kata.v2"
    }
  }
}
```

重启 Docker 以应用更改：

```bash
sudo systemctl restart docker

# Verify Kata is available in Docker
docker info | grep -A5 Runtimes
# Expected output should include "io.containerd.runc.v2 kata"
```

#### 配置 OpenSandbox Server

编辑 `~/.sandbox.toml`：

```toml
[runtime]
type = "docker"
execd_image = "opensandbox/execd:latest"

[secure_runtime]
type = "kata"
docker_runtime = "kata"
```

#### 验证安装

**通过 OpenSandbox API 测试**

创建一个沙箱，通过检查内核来验证它运行在 VM 中：

```bash
# Create a test sandbox
curl --location 'http://127.0.0.1:8080/v1/sandboxes' \
  --header 'Content-Type: application/json' \
  --data '{
    "image": {"uri": "ubuntu:latest"},
    "timeout": 3600,
    "resourceLimits": {"cpu": "500m", "memory": "512Mi"},
    "entrypoint": ["/bin/bash", "-c", "while true; do uname -a; sleep 1; done"],
    "metadata": {
      "name": "kata-sandbox"
    }
  }'
```

检查容器内的内核以验证 VM 隔离：

```bash
# Get the container ID
docker ps | grep kata-sandbox

# Check the kernel inside the container (should be different from host)
docker exec <container_id> uname -a
# Expected output: Linux <hostname> 5.10.x-generic #x86_64 ... (Kata VM kernel)

# Compare with host kernel
uname -a
# Host kernel might be different version or have different hostname
```

**Kata VM 的关键特征：**
- 容器运行在独立的内核中，主机名不同
- 内核版本通常为 `5.10.x`（Kata 的 Guest 内核）
- 主机进程列表中出现 `qemu-system-x86_64` 或类似的 hypervisor 进程
---

## Kubernetes 模式

Kubernetes 模式通过 RuntimeClass 资源支持安全运行时。

### 前提条件

- 使用 containerd 运行时的 Kubernetes 集群
- 安全运行时已安装在所有节点上
- RuntimeClass CRD 已创建

### Kubernetes 上的 gVisor 设置

#### 步骤 1：在所有节点上安装 gVisor 组件

对于使用 containerd 的 Kubernetes，需要安装**两个**组件：

1. **runsc** - gVisor 的 OCI 运行时
2. **containerd-shim-runsc-v1** - gVisor 的 containerd shim

```bash
# On each node - Ubuntu/Debian
curl -fsSL https://gvisor.dev/archive.key | sudo gpg --dearmor -o /usr/share/keyrings/gvisor-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/gvisor-archive-keyring.gpg] https://storage.googleapis.com/gvisor/releases release main" | \
  sudo tee /etc/apt/sources.list.d/gvisor.list
sudo apt-get update

# Install both gVisor components
sudo apt-get install -y runsc containerd-shim-runsc-v1

# Verify installation
runsc --version
containerd-shim-runsc-v1 --version
```
> **参考**：完整的安装说明和其他发行版请参见 [gVisor 安装指南](https://gvisor.dev/docs/user_guide/containerd/configuration/)。

#### 步骤 2：配置 containerd

编辑 `/etc/containerd/config.toml`：

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
          runtime_type = "io.containerd.runsc.v1"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc.options]
            TypeUrl = "io.containerd.runsc.v1.options"
            ConfigPath = "/etc/containerd/runsc.toml"
```

```bash
sudo tee /etc/containerd/runsc.toml > /dev/null <<'EOF'
[runsc]
  platform = "ptrace"
EOF
```

重启 containerd：

```bash
sudo systemctl restart containerd
```

#### 步骤 3：创建 RuntimeClass CRD

```yaml
# gvisor-runtimeclass.yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
scheduling:
  nodeSelector:
    kubernetes.io/arch: amd64
```

```bash
kubectl apply -f gvisor-runtimeclass.yaml
```

#### 步骤 4：配置 OpenSandbox Server

编辑 `~/.sandbox.toml`：

```toml
[runtime]
type = "kubernetes"
execd_image = "opensandbox/execd:latest"

[secure_runtime]
type = "gvisor"
k8s_runtime_class = "gvisor"
```

#### 步骤 5：验证安装

```bash
# Test the RuntimeClass
kubectl run test-gvisor --restart=Never --image=hello-world --runtime-class=gvisor
kubectl logs test-gvisor
kubectl delete pod test-gvisor
```

### Kubernetes 上的 Kata Containers 设置

#### 步骤 1：安装 Kata Containers

请遵循 [Kata Containers 官方安装指南](https://github.com/kata-containers/kata-containers/blob/main/tools/packaging/kata-deploy/helm-chart/README.md)。

使用 Helm 快速安装：

```bash
# Install kata-deploy which will set up Kata Containers via DaemonSet
helm install kata-deploy "oci://ghcr.io/kata-containers/kata-deploy-charts/kata-deploy" --version "3.27.0" --namespace kube-system --create-namespace

# Wait for kata-deploy pods to be ready
kubectl wait --for=condition=ready pod -l name=kata-deploy -n kube-system --timeout=300s
```

> **注意**：`kata-deploy` DaemonSet 会自动配置所有节点上的 containerd。使用 kata-deploy 时无需手动配置 containerd。

#### 步骤 2：验证安装

检查 Kata Containers 是否已安装、RuntimeClass 是否已创建：

```bash
# Check RuntimeClasses
kubectl get runtimeclass

# Expected output:
# NAME         HANDLER     AGE
# kata         kata-qemu   10m
# kata-qemu    kata-qemu   10m
# kata-clh     kata-clh    10m
# kata-fc      kata-fc     10m

# Test Kata with a simple pod
kubectl run test-kata --restart=Never --image=hello-world --runtime-class=kata-qemu
kubectl logs test-kata
kubectl delete pod test-kata
```

### 为不同运行时创建 Pool（可选）

使用 Pool CRD 预热沙箱时，请为每种运行时类型创建独立的 Pool：

```yaml
# gvisor-pool.yaml
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: Pool
metadata:
  name: gvisor-pool
  labels:
    runtime: gvisor
spec:
  template:
    spec:
      runtimeClassName: gvisor
      containers:
        - name: sandbox-container
          image: opensandbox/code-interpreter:v1.1.0
  capacitySpec:
    bufferMax: 10
    bufferMin: 2
    poolMax: 20
    poolMin: 5
```

---

## 用户指南

本节面向使用 OpenSandbox 的 AI 应用开发者。

### 无需修改代码

**重要**：安全运行时在 server 级配置。你的代码不需要更改。

只需通过 OpenSandbox Lifecycle API 创建沙箱 —— server 会自动应用配置的安全运行时：

**创建一个测试沙箱：**

```bash
curl -X POST http://localhost:8080/v1/sandboxes \
  -H "Content-Type: application/json" \
  -d '{
    "image": {"uri": "python:3.11"},
    "timeout": 3600,
    "resourceLimits": {"cpu": "500m", "memory": "512Mi"},
    "entrypoint": ["python", "-u", "-c", "import time\nwhile True: print(\"hello from secure sandbox!\"); time.sleep(1)"],
    "metadata": {
      "name": "my-secure-sandbox"
    }
  }'
```

**响应：**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "running"
}
```

沙箱将自动使用 server 上配置的安全运行时（gVisor、Kata 或 runc）。

### 工作原理

1. **管理员**在 `~/.sandbox.toml` 中配置安全运行时
2. **Server** 在启动时校验运行时
3. **Server** 自动将运行时注入每个沙箱：
   - Docker 模式：在 HostConfig 中添加 `runtime`
   - Kubernetes 模式：在 Pod spec 中添加 `runtimeClassName`
4. **用户**通过 API 创建沙箱 —— 无需运行时参数

### 验证运行时隔离

创建沙箱后，验证所使用的运行时：

**Docker 模式：**
```bash
docker ps --format "{{.ID}}\t{{.Image}}\t{{.Names}}"
docker inspect <container_id> | grep -A2 Runtime
# Expected output for gVisor:
# "Runtime": "runsc",
```

**Kubernetes 模式：**
```bash
kubectl get pod <pod-name> -o jsonpath='{.spec.runtimeClassName}'
# Expected output for gVisor:
# gvisor
```

---

## 管理员指南

本节面向管理安全运行时基础设施的平台运维和 SRE。

### 前提条件

安全运行时必须在配置 OpenSandbox **之前**安装并配置到基础设施上。OpenSandbox 不会自动安装运行时。

### 安装概览

| 运行时 | Docker | Kubernetes |
|---------|--------|------------|
| gVisor | 安装 runsc → 配置 daemon.json | 安装 runsc → 配置 containerd → 创建 RuntimeClass |
| Kata (QEMU) | 安装 kata-runtime → 配置 daemon.json | 安装 Kata → 配置 containerd → 创建 RuntimeClass |
| Kata (Firecracker) | 不支持 | 安装 Kata → 配置 containerd → 创建 RuntimeClass |

### 配置校验

server 在启动时校验安全运行时配置：

1. **Docker 模式**：检查运行时是否存在于 Docker daemon 的运行时列表中
2. **Kubernetes 模式**：检查 RuntimeClass 是否存在于集群中

校验失败时，server 会拒绝启动并给出明确的错误信息。

### 安全最佳实践

1. **默认使用 gVisor**：对大多数工作负载以可接受的性能提供良好的安全性
2. **对不可信代码使用 Kata**：对完全未知的代码提供最大隔离
3. **定期更新**：保持运行时更新以获取安全补丁
4. **测试兼容性**：生产前用选定的运行时验证你的工作负载
5. **监控资源**：安全运行时的内存开销更高

### 运行时选择指南

| 使用场景 | 推荐运行时 | 理由 |
|----------|---------------------|-----------|
| 开发/测试 | runc（默认） | 启动最快、开销最低 |
| 生产 AI 代码执行 | gVisor | 安全与性能的良好平衡 |
| 高安全要求 | Kata (QEMU) | 最大隔离、完全兼容 |
| 高密度多租户 | Kata (Firecracker) | 每个沙箱的内存开销最小 |
| 不可信网络代码 | gVisor 或 Kata | Syscall 过滤可防御网络攻击 |

---

## 故障排除与最佳实践

### 常见问题

#### 1. 找不到运行时（Docker）

**错误**：`Configured Docker runtime 'runsc' is not available.`

**解决方法**：确保运行时已配置在 `/etc/docker/daemon.json` 中并已重启 Docker：

```bash
sudo systemctl restart docker
docker info | grep -A5 Runtimes
```

#### 2. 找不到 RuntimeClass（Kubernetes）

**错误**：`RuntimeClass 'gvisor' does not exist.`

**解决方法**：创建 RuntimeClass CRD：

```bash
kubectl get runtimeclass
kubectl apply -f gvisor-runtimeclass.yaml
```

#### 3. Syscall 兼容性问题

**错误**：容器以退出码 1 退出，无日志

**原因**：gVisor 没有实现全部 syscall，某些应用可能不兼容。

**解决方法**：查看 [gVisor 兼容性指南](https://gvisor.dev/docs/user_guide/compatibility/)。可尝试兼容性更好的 Kata (QEMU)。

#### 4. Pod 卡在 ContainerCreating

**原因**：节点上未配置 RuntimeClass handler。

**解决方法**：验证 containerd 配置：

```bash
# On the node
sudo containerd config dump
sudo systemctl restart containerd
```

#### 5. Egress Sidecar 与 gVisor 不兼容

**错误**：沙箱 Pod 出现 CrashLoopBackOff，egress 容器日志为：
```
iptables: Failed to initialize nft: Protocol not supported
```
或使用 iptables-legacy 时：
```
iptables v1.8.9 (legacy): can't initialize iptables table 'nat': Table does not exist (do you need to insmod?)
```

**原因**：gVisor 的 netstack 实现了 `filter` 和 `mangle` iptables 表，但没有实现 `nat` 表。egress sidecar 使用 `nat` 表中的 REDIRECT 规则拦截 DNS 查询（端口 53 → 15353），因此在 gVisor 下无法启动。这是 gVisor 的上游限制（[gvisor#170](https://github.com/google/gvisor/issues/170)）。

**解决方法**：
- 使用 `secure_runtime.type = "kata"` 并设置 `k8s_runtime_class = "kata-qemu"` —— Kata 为每个 Pod 提供完整的 Linux 内核，因此 `nat` 表可用，egress sidecar 无需改动即可工作。
- 在 gVisor 下改用 CNI 层的 FQDN 策略（例如 Cilium `toFQDNs`）代替 egress sidecar 实现网络隔离。
- 如果不需要出站控制，从沙箱创建请求中移除 `network_policy`。

> **注意**：server 会在请求时校验该组合，当 `secure_runtime.type = "gvisor"` 与 `network_policy` 同时使用时返回 HTTP 400 和明确的错误信息。

### 兼容性矩阵

| 特性 | runc | gVisor | Kata (QEMU) | Kata (CLH) | Kata (FC) |
|---------|------|--------|-------------|------------|-----------|
| Syscall 兼容性 | 完整 | 部分 | 完整 | 完整 | 有限 |
| GPU 支持 | 是 | 否 | 是 | 是 | 否 |
| IPv6 | 是 | 是 | 是 | 是 | 是 |
| 特权模式 | 是 | 否 | 是 | 是 | 否 |
| Docker 卷 | 是 | 是 | 是 | 是 | 是 |
| Systemd | 是 | 否 | 是 | 是 | 否 |
| iptables `nat` 表（egress sidecar） | 是 | **否** | 是 | 是 | 是 |

### 获取帮助

- **文档**：[OpenSandbox GitHub](https://github.com/opensandbox-group/OpenSandbox)
- **Issue**：通过 [GitHub Issues](https://github.com/opensandbox-group/OpenSandbox/issues) 报告缺陷
