---
title: Kubernetes PVC
description: 将 Kubernetes PersistentVolumeClaims 挂载到 OpenSandbox 容器中，实现持久化存储。
---

# Kubernetes PVC Volume Mount

本示例展示如何使用 Kubernetes [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)（PVC）为沙箱提供存储。写入 PVC 的数据在沙箱进程结束后依然存在，因此当后续沙箱挂载同一个 claim 时，文件仍然可用。

OpenSandbox 支持两种获取 PVC 的模式，请根据 claim 生命周期的归属方来选择：

| 模式 | `createIfNotExists` | `deleteOnSandboxTermination` | PVC 归属方 | 适用场景 |
|------|---------------------|------------------------------|------------------|-------------|
| **自带（BYO）** | `false` | _被忽略_ | 你（在带外预先创建） | 长期共享存储；模型缓存；多个沙箱复用的基础数据集 |
| **服务器管理、持久化** | `true` | `false`（默认） | 你（首次创建之后） | 首次使用时创建，并在沙箱生命周期之外保留 |
| **服务器管理、临时** | `true` | `true` | 服务器 | 限定单个沙箱使用的临时存储；沙箱终止时自动清理 |

两种模式挂载 PVC 的方式完全相同，区别仅在于创建与清理。清理机制的详细说明见 [PVC 生命周期](#pvc-lifecycle)。

## 前置条件

### CSI Driver

Kubernetes PVC 需要 [Container Storage Interface (CSI)](https://kubernetes-csi.github.io/docs/drivers.html) driver 来创建和挂载存储。请安装与你存储后端匹配的 driver。例如，[Alibaba Cloud CSI Driver](https://github.com/kubernetes-sigs/alibaba-cloud-csi-driver) 覆盖：

- **Cloud Disk (EBS)** —— 块存储；高性能单节点读写
- **NAS** —— 共享文件存储；多节点读写（`ReadWriteMany`）
- **OSS** —— 对象存储；大规模共享读
- **CPFS** —— 高性能并行文件系统
- **LVM** —— 本地卷管理

### OpenSandbox 服务器

服务器必须运行在 Kubernetes 运行时上，并使用 BatchSandbox workload provider。官方 Helm chart 已授予 BYO 挂载和服务器管理创建所需的 RBAC（对 `persistentvolumeclaims` 的 `get`/`create`/`list`/`delete`/`patch` 权限）。

### Python SDK

```shell
uv pip install opensandbox
```

## 模式 1：自带 PVC（BYO）

当 PVC 是你平台设施的一部分时使用此模式——例如多个沙箱复用的共享 NAS，或存放模型权重的预热磁盘。

### 1. 创建 PVC

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: opensandbox
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: <your-storage-class>
  resources:
    requests:
      storage: 10Gi
```

```shell
kubectl apply -f pvc.yaml
kubectl get pvc my-pvc -n opensandbox        # should be Bound
```

### 2. 从沙箱挂载

```python
from opensandbox import Sandbox
from opensandbox.models.sandboxes import PVC, Volume

sandbox = await Sandbox.create(
    image="python:3.11",
    volumes=[
        Volume(
            name="data-volume",
            pvc=PVC(
                claimName="my-pvc",
                createIfNotExists=False,      # never auto-provision a BYO claim
            ),
            mountPath="/mnt/data",
            readOnly=False,
        ),
    ],
)

result = await sandbox.commands.run("ls -la /mnt/data")
print("\n".join(msg.text for msg in result.logs.stdout))
```

OpenSandbox 永远不会删除 BYO PVC：沙箱终止时，claim 保持绑定状态，数据可供下一个挂载它的沙箱使用。

### 运行端到端示例

```shell
export OPEN_SANDBOX_API_KEY=your-api-key
export OPEN_SANDBOX_BASE_URL=http://localhost:8080
export SANDBOX_PVC_NAME=my-pvc

python examples/kubernetes-pvc-volume-mount/main.py
```

![Kubernetes PVC Volume Mount demo](../public/images/kubernetes-pvc-volume-mount-demo.png)

该脚本会创建一个沙箱，在 `/mnt/data` 下写入一个标记文件，然后终止该沙箱；接着创建第二个绑定同一 PVC 的沙箱，并确认标记文件仍然存在。

## Pool 模式：预挂载共享 PVC

Pool pod 在沙箱分配之前就已创建，因此 pool 共享的存储必须是 Pool pod 模板的一部分。先创建 PVC，然后通过 `Pool.spec.template` 将其挂载到每个预热 pod 中：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-workspace-pvc
  namespace: opensandbox
spec:
  accessModes: [ReadWriteMany]
  storageClassName: <your-rwx-storage-class>
  resources:
    requests:
      storage: 100Gi
---
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: Pool
metadata:
  name: shared-workspace-pool
  namespace: opensandbox
spec:
  template:
    spec:
      containers:
        - name: sandbox-container
          image: python:3.11
          command: ["sleep", "3600"]
          volumeMounts:
            - name: shared-workspace
              mountPath: /workspace
      volumes:
        - name: shared-workspace
          persistentVolumeClaim:
            claimName: shared-workspace-pvc
  capacitySpec:
    bufferMax: 10
    bufferMin: 2
    poolMax: 20
    poolMin: 5
```

在从 pool 分配沙箱之前，先将 Pool manifest 应用到集群。
之后沙箱通过 `extensions.poolRef` 选择预配置的 Pool，且不传递每个沙箱各自的
`volumes` 列表。

::: warning 仅支持静态 pool 存储
PVC 必须已经存在，并且其访问模式和存储后端必须允许所有已调度的预热 pod 挂载它。OpenSandbox 不会创建、修改或删除 Pool 模板引用的 PVC。沙箱创建请求不能在使用 `extensions.poolRef` 的同时添加 `volumes`，因为所选的预热 pod 已经存在。
:::

## 模式 2：服务器管理的 PVC

当希望沙箱拥有自己的存储时使用此模式。服务器在首次引用该 claim 名称时按需创建 PVC。你可以通过 `deleteOnSandboxTermination` 控制 claim 是否在沙箱终止后保留。

### 创建并持久化（默认）

经典的"第一个沙箱创建、后续沙箱复用"模式——例如一个能在崩溃和重启后幸存、且永远不需要手动 `kubectl apply` 的热缓存。

```python
sandbox = await Sandbox.create(
    image="python:3.11",
    volumes=[
        Volume(
            name="cache",
            pvc=PVC(
                claimName="agent-cache",
                createIfNotExists=True,                   # auto-provision on first use
                deleteOnSandboxTermination=False,         # default: keep the PVC
                storageClass="alibaba-cloud-disk-ssd",    # optional; defaults to cluster default
                storage="20Gi",                           # optional; defaults to server config
                accessModes=["ReadWriteOnce"],            # optional; defaults to ReadWriteOnce
            ),
            mountPath="/mnt/cache",
        ),
    ],
)
```

第一次调用会创建 `agent-cache`；后续使用相同 `claimName` 的调用会挂载现有 PVC 并跳过创建。PVC 会一直保留，直到你用 `kubectl` 删除它。

### 创建并清理

限定单个沙箱使用的临时存储——沙箱终止（包括 TTL 过期）时，服务器会回收 PVC。

```python
sandbox = await Sandbox.create(
    image="python:3.11",
    timeout=600,                                   # 10-minute sandbox
    volumes=[
        Volume(
            name="scratch",
            pvc=PVC(
                claimName=f"scratch-{run_id}",   # unique per run; opted-in PVCs are owned exclusively
                createIfNotExists=True,
                deleteOnSandboxTermination=True,
                storage="5Gi",
            ),
            mountPath="/mnt/scratch",
        ),
    ],
)
```

::: tip 清理范围
服务器只会删除它以这种 opt-in 方式创建的 PVC。预先存在的 PVC 以及以 `deleteOnSandboxTermination=false` 创建的 PVC 永远不会被触碰。 opted-in 的 PVC 由创建它的沙箱独占——第二个沙箱尝试挂载相同 `claimName` 会被 `409 CONFLICT` 拒绝。请为每个沙箱使用唯一的 `claimName`，如果需要共享存储，请使用非 opt-in / 预先存在的 PVC。
:::

## PVC 生命周期

清理行为遵循创建时使用的模式：

| 来源 | 沙箱终止时的清理 |
|--------|-------------------------------|
| 预先存在的 PVC（BYO） | 服务器永远不会触碰。 |
| 自动创建、`deleteOnSandboxTermination=false`（默认） | PVC 保留；由调用方负责清理。 |
| 自动创建、`deleteOnSandboxTermination=true` | 服务器删除该 PVC。 |

对于 opt-in 的 PVC，服务器会为其打上 `opensandbox.io/volume-managed-by=server` 和 `opensandbox.io/id=<sandbox-id>` 标签，然后通过两条分层路径执行清理：

1. **`ownerReferences`** —— PVC 在创建后立即被 patch 为指向沙箱的 workload 自定义资源。当该 CR 被删除时，Kubernetes 垃圾回收会级联删除 PVC，包括永远不会触达 `DELETE /sandboxes/{id}` API 的控制器驱动的 TTL 过期。
2. **标签选择器清扫** —— 在 `DELETE /sandboxes/{id}` 时，服务器按这些标签列出 PVC 并尽力删除。这可以覆盖 `ownerReferences` patch 失败（例如 RBAC 问题）的情况，并确保 API 返回时 PVC 已被删除。

这两条路径只匹配带服务器标签的 PVC，因此 BYO 和未 opt-in 的 claim 永远不会被回收。PVC 被删除后，底层 PV 遵循其 `StorageClass.reclaimPolicy`。

## 重要说明

::: warning
- 每个沙箱的 `volumes` 不能与 `extensions.poolRef` 组合使用。Pool 模式请按上文描述在 `Pool.spec.template` 中预挂载已存在的共享 PVC。
- 如果访问模式允许（例如 `ReadWriteMany`），多个沙箱可以挂载同一 PVC——但前提是该 PVC **未** opt-in 自动清理。以 `deleteOnSandboxTermination=true` 创建的 PVC 由创建它的沙箱独占；服务器会以 `409 CONFLICT` 拒绝其他沙箱挂载它们的尝试。
- 单个请求中对同一 `claimName` 的所有挂载在 `createIfNotExists` 和 `deleteOnSandboxTermination` 上必须一致；不一致会以 `400 INVALID_PARAMETER` 拒绝。
:::

## 参考

- [Kubernetes CSI Drivers](https://kubernetes-csi.github.io/docs/drivers.html)
- [Alibaba Cloud CSI Driver](https://github.com/kubernetes-sigs/alibaba-cloud-csi-driver)
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/kubernetes-pvc-volume-mount)
