---
title: 暂停与恢复
description: 将沙箱状态暂停为 OCI 镜像，并在 Kubernetes 上从快照恢复。
---

# 暂停与恢复指南

本指南介绍如何对基于 Kubernetes 的 OpenSandbox 沙箱使用暂停与恢复功能。暂停会将沙箱的根文件系统提交为 OCI 镜像并释放集群资源。恢复则从该镜像还原沙箱。

## 目录

- [概述](#概述)
- [架构](#架构)
- [前提条件](#前提条件)
- [控制器配置](#控制器配置)
- [镜像仓库与 Secret 设置](#镜像仓库与-secret-设置)
- [使用指南](#使用指南)
- [管理员指南](#管理员指南)
- [SandboxSnapshot 参考](#sandboxsnapshot-参考)
- [故障排除](#故障排除)

---

## 概述

### 暂停与恢复做什么

| | 行为 |
|--|---------|
| **暂停** | 创建内部 `SandboxSnapshot`，将运行中容器的根文件系统提交为 OCI 镜像，然后静默沙箱运行时并释放 Pod / Pool 分配 |
| **恢复** | 复用同一个 `BatchSandbox`，将其模板改写为最新快照镜像，并从该镜像重建运行时 |
| **sandboxId** | 在暂停/恢复周期之间保持稳定 —— 调用方在整个沙箱生命周期内使用同一个 ID |
| **副本支持** | 目前仅支持 `BatchSandbox.spec.replicas=1`。Server 创建的 Kubernetes 沙箱使用 `replicas: 1`；副本数不同的直接 CR 会被控制器暂停入口拒绝。 |

### 关键设计原则

**控制器级配置**：镜像仓库 URL 和推送/拉取 Secret 配置在 Kubernetes 控制器管理器上，而不是 `~/.sandbox.toml` 中。SDK 用户和 API 调用方使用暂停/恢复**无需任何代码修改** —— 只需对现有沙箱 ID 调用 `pause` 和 `resume`。

### 生命周期

```text
Time ---------------------------------------------------------------->

Sandbox lifecycle:   [Running]--[Pausing]--[Paused]--[Resuming]--[Running]
                         |                     |
                  commit rootfs          rewrite template images
                  push to registry       recreate runtime from snapshot
                  release pods/alloc
```

### 状态机细节

沙箱在稳定状态和中间状态之间流转：

| 状态 | 类型 | 说明 |
|-------|------|-------------|
| `Running` | 稳定 | 沙箱处于活跃状态，正在处理请求 |
| `Pausing` | 中间 | 暂停操作进行中。快照提交通过内部 `SandboxSnapshot` 资源协调。 |
| `Paused` | 稳定 | 沙箱已暂停，最新根文件系统快照就绪，运行时 Pod / Pool 分配已释放 |
| `Resuming` | 中间 | 恢复操作进行中。控制器正在将沙箱模板改写为最新快照镜像并重建运行时 |
| `Failed` | 稳定 | 操作失败（查看 `reason` 和 `message` 了解详情） |

Lifecycle API 只暴露上述粗粒度的沙箱状态。要查看详细的快照进度，请检查内部 `SandboxSnapshot` 资源：

在控制器报告 `Paused` 之后，Pod 终止仍可能继续进行。在此期间可以提交恢复请求；
控制器会等待旧 Pod 被移除后再重建其替代 Pod。带有删除时间戳的 Pod 不会产生新的
运行时失败条件，包括 Kubernetes 在删除过程中报告的终态退出状态。沙箱上已记录的
失败保持终态不变，而替代 Pod 的失败仍会正常上报。

- `Pending`：快照请求已接受，等待解析源 Pod / 创建提交 Job
- `Committing`：提交 Job 正在运行并推送快照镜像
- `Succeed`：快照就绪，可用于下一次恢复
- `Failed`：快照创建失败

### 保留与不保留的内容

| | 是否保留？ |
|--|-----------|
| 根文件系统内容 | ✅ 是 —— 提交为 OCI 镜像 |
| 环境变量 | ✅ 是 —— 来自 BatchSandbox 模板 |
| 运行中进程 / 内存 | Rootfs 模式：否。可选的 QEMU-in-runc 模式：QEMU 进程与 Guest 内存会被还原；其他外层进程重启。参见 [QEMU VMState 快照](/guides/qemu-vmstate-snapshots)。 |
| 显式挂载的卷 | 取决于卷类型 |
| Credential Vault 条目 | 否 —— 仅存储在 egress sidecar 内存中；恢复后需从可信控制平面重新注入 |

暂停/恢复目前仅支持单副本。内部暂停快照只记录一个源 Pod 的容器镜像，不存储按副本的状态，因此除非 `BatchSandbox.spec.replicas=1`，Kubernetes 控制器会拒绝暂停请求。

恢复后所需的凭据重新注入流程，请参见 [Credential Vault](/guides/credential-vault#persistence-across-pause-and-resume)。

### 持久性与异常终止

只有当快照达到 `Succeed` 且 OCI 镜像已推送到配置的镜像仓库后，根文件系统才是持久的。暂停请求被接受，或快照处于 `Pending` 或 `Committing`，都不能提供持久性保证。

最后一次成功快照之后的更改仍保留在容器的可写层中。当进程退出、容器被 OOM 杀死、Pod 进入 `Failed`、Pod 被驱逐或节点不可用时，这些更改不会被自动快照。Lifecycle API 目前不提供失败后的快照或恢复操作，由 Kubernetes 重启创建的新容器也不会继承旧的可写层。

当元数据和可写快照仍存在于源节点上时，镜像提交器有时可以从 containerd 直接读取已停止的容器。这是一种尽力而为的运维恢复路径，不是受支持的持久性契约：正常的快照 API 要求沙箱处于 `Running` 状态，而 containerd 或 kubelet 的垃圾回收、Pod 替换、节点丢失或容器身份被复用，都可能使旧层不可用或产生歧义。

对于必须在意外终止后仍然存留的数据，请将其存入持久卷或外部服务，并显式执行根文件系统快照以获得可复现的环境状态。挂载卷中的文件遵循该卷自身的持久性与备份行为；它们不会包含在 OCI rootfs 快照中。

---

## 架构

```
API caller
    │ POST /v1/sandboxes/{id}/pause
    ▼
OpenSandbox Server
    │ PATCH BatchSandbox.spec.pause=true
    ▼
BatchSandbox Controller (Kubernetes)
    │ validates lifecycle state
    │ creates internal SandboxSnapshot CR
    ▼
SandboxSnapshot Controller
    │ resolves running Pod
    │ creates commit Job on the same node
    ▼
commit Job Pod (image-committer)
    │ containerd API: commit container rootfs → OCI image
    │ OCI registry resolver: push image
    ▼
SandboxSnapshot.status.phase = Succeed
    │ BatchSandbox.status.phase = Paused
    │ deletes Pods or releases pooled allocation
    ▼
Cluster resources released

--- Later: resume ---

API caller
    │ POST /v1/sandboxes/{id}/resume
    ▼
OpenSandbox Server
    │ PATCH BatchSandbox.spec.pause=false
    ▼
BatchSandbox Controller
    │ reads internal SandboxSnapshot
    │ rewrites pod template images from snapshot
    │ clears poolRef for pooled sandboxes
    │ recreates runtime Pods
    ▼
Sandbox running again with restored filesystem
```

---

## 前提条件

1. 部署了 OpenSandbox 控制器的 **Kubernetes 集群**
2. 集群节点可访问（推送）且 Kubernetes API 可访问（拉取）的 **OCI 兼容容器镜像仓库**
3. 用于镜像仓库认证的 `kubernetes.io/dockerconfigjson` 类型 **Kubernetes Secret**
4. 配置了快照镜像仓库和 Secret 参数的 **控制器管理器**

---

## 控制器配置

为控制器管理器部署配置快照参数：

```yaml
- --snapshot-registry=registry.example.com/sandboxes
- --snapshot-registry-insecure=false
- --snapshot-push-secret=registry-snapshot-push-secret
- --resume-pull-secret=registry-pull-secret
```

### 配置参考

| 键 | 类型 | 默认值 | 说明 |
|-----|------|---------|-------------|
| `--snapshot-registry` | string | `""` | **必填。** OCI 镜像仓库前缀。默认情况下，镜像存储为 `<registry>/<sandboxName>-<container>:snap-gen<N>`。 |
| `--snapshot-image-uri-template` | string | `""` | 用于快照镜像 URI 的 Go 命名字段模板；为空时使用下方的默认命名规则。 |
| `--snapshot-registry-insecure` | bool | `false` | 为快照推送操作启用不安全镜像仓库模式。仅用于 HTTP 或自签名的本地仓库。 |
| `--snapshot-push-secret` | string | `""` | 用于推送快照的 Kubernetes Secret 名称。必须为 `kubernetes.io/dockerconfigjson` 类型。 |
| `--image-committer-pod-template-file` | string | `""` | 提交 Job Pod 的 PodTemplateSpec 覆盖文件路径。 |
| `--resume-pull-secret` | string | `""` | 注入到已恢复沙箱中用于拉取快照镜像的 Kubernetes Secret 名称。可与推送 Secret 相同。 |
| `--image-committer-image` | string | `"image-committer:dev"` | 提交 Job 使用的镜像。 |
| `--commit-job-timeout` | duration | `"10m"` | 提交 Job 的超时时间。 |

### Helm chart 支持

`opensandbox-controller` Helm chart 现在直接暴露快照相关的控制器 values：

- `controller.snapshot.imageCommitterImage`
- `controller.snapshot.imageCommitterPodTemplate`
- `controller.snapshot.commitJobTimeout`
- `controller.snapshot.registry`
- `controller.snapshot.imageURITemplate`
- `controller.snapshot.registryInsecure`
- `controller.snapshot.snapshotPushSecret`
- `controller.snapshot.resumePullSecret`

对于一体化 `opensandbox` chart，在 `opensandbox-controller.*` 前缀下使用相同的 values。

### 启动行为

Server 不再携带专门的暂停/恢复配置。缺失的镜像仓库或 Secret 设置由 Kubernetes 控制器在处理 `SandboxSnapshot` 时呈现，例如以 `SandboxSnapshot.status.conditions[type=Failed]` 的形式给出 `RegistryNotConfigured` 等原因。

---

## 镜像仓库与 Secret 设置

### 步骤 1：准备镜像仓库

任何 OCI 兼容的镜像仓库都可以（Docker Hub、GitHub Container Registry、Harbor、私有的 `registry:2` 实例等）。该仓库必须：

- **集群节点可访问**（供提交 Job 推送）
- **Kubernetes API server / kubelet 可访问**（供恢复时拉取镜像）

### 步骤 2：创建推送 Secret

```bash
kubectl create secret docker-registry registry-snapshot-push-secret \
  --docker-server=registry.example.com \
  --docker-username=<username> \
  --docker-password=<password-or-token> \
  --namespace=<sandbox-namespace>
```

### 步骤 3：创建拉取 Secret

拉取 Secret 由恢复后的 `BatchSandbox` Pod 用于拉取快照镜像。如果你的凭据同时具有读写权限，它可以是与推送 Secret 相同的 Secret：

```bash
kubectl create secret docker-registry registry-pull-secret \
  --docker-server=registry.example.com \
  --docker-username=<username> \
  --docker-password=<password-or-token> \
  --namespace=<sandbox-namespace>
```

### 使用私有 `registry:2`（开发环境）

在开发环境中使用集群内部的 `registry:2` 部署：

```bash
# Create a registry deployment
kubectl create deployment docker-registry \
  --image=registry:2 --port=5000

kubectl expose deployment docker-registry --port=5000

# No authentication needed for internal registry
# Leave snapshot push/pull secret flags empty on the controller manager
```

---

## 使用指南

控制器管理器配置完成且 server 运行后，暂停/恢复通过标准 Lifecycle API 工作。无需任何 SDK 修改。

### 暂停沙箱

```bash
curl -X POST http://localhost:8080/v1/sandboxes/{sandbox_id}/pause \
  -H "Content-Type: application/json"
```

**响应：** `202 Accepted`，响应体为空。

暂停是异步的。沙箱依次经过：
`running` → `pausing` → `paused`

### 查询暂停状态

```bash
curl http://localhost:8080/v1/sandboxes/{sandbox_id}
```

当 `status` 为 `paused` 时，文件系统已提交且集群资源已释放。

### 恢复沙箱

```bash
curl -X POST http://localhost:8080/v1/sandboxes/{sandbox_id}/resume \
  -H "Content-Type: application/json"
```

**响应：** `202 Accepted`，响应体为空。

沙箱依次经过：
`paused` → `resuming` → `running`

### 多次暂停/恢复循环

暂停和恢复可以重复执行。每个暂停周期都会产生一个新的快照镜像标签（`snap-gen1`、`snap-gen2` 等）。下一次恢复总是使用最新的快照。

---

## 管理员指南

### 控制器 RBAC

OpenSandbox 控制器执行暂停/恢复需要以下 RBAC 权限（已包含在 Helm chart 和 `make manifests` 的输出中）：

| 资源 | 动词 | 用途 |
|----------|-------|---------|
| `sandboxsnapshots` | get, list, watch, create, update, patch, delete | 管理 SandboxSnapshot CR |
| `jobs` / `jobs/status` | 全部 | 创建/监控提交 Job |
| `secrets` | get | 在创建提交 Job 之前验证推送 Secret 是否存在 |
| `pods` | get, list, watch | 查找用于提交的运行中 Pod |

### 快照镜像命名

内部暂停/恢复快照镜像命名为：
```
<snapshot-registry>/<sandboxName>-<containerName>:snap-gen<N>
```

例如，设置 `--snapshot-registry=registry.example.com/sandboxes`、沙箱 `my-sandbox`、容器 `sandbox` 时，首次暂停：
```
registry.example.com/sandboxes/my-sandbox-sandbox:snap-gen1
```

Server 托管的公共快照使用相同的仓库布局，但标签由 snapshot-id 稳定派生：
```
<snapshot-registry>/<sandboxName>-<containerName>:snap-<snapshotIdHex>
```

控制器通过 owner reference 区分这两种模式。暂停/恢复快照由 `BatchSandbox` 控制器创建，并带有指向所属 `BatchSandbox` 的 controller ownerReference；公共快照由 Lifecycle server 创建，不使用该 ownerReference。

#### 自定义镜像名称

在控制器管理器上设置 `--snapshot-image-uri-template`，或在控制器 Helm chart 中设置 `controller.snapshot.imageURITemplate`，即可替换命名规则。空值使用以下 Go `text/template`：

```text
{{.Registry}}/{{.SandboxName}}-{{.ContainerName}}:{{.SnapshotTag}}
```

模板接收以下命名字段：

| 字段 | 值 |
|-------|-------|
| `Registry` | 必填的 `--snapshot-registry` 前缀，包含任意仓库路径 |
| `Namespace` | 快照所在的 Kubernetes 命名空间 |
| `SandboxName` | 源 BatchSandbox 名称 |
| `SnapshotName` | SandboxSnapshot 名称 |
| `SnapshotUID` | SandboxSnapshot UID |
| `SnapshotCreationTime` | SandboxSnapshot 的 `metadata.creationTimestamp`，以控制器进程本地时区的 Go `time.Time` 表示 |
| `ContainerName` | 源容器名称；对于 QEMU VM-state 产物为 `vmstate` |
| `SnapshotTag` | 前文描述的现有 `snap-gen<N>` 或公共快照标签 |
| `ArtifactKind` | 容器镜像为 `rootfs`，QEMU VM-state 产物为 `vmstate` |

例如，为每个快照产物使用同一个仓库并配以不同标签：

```yaml
controller:
  snapshot:
    registry: registry.example.com/sandboxes
    imageURITemplate: '{{.Registry}}/snapshots:{{.SnapshotUID}}-{{.ArtifactKind}}-{{.ContainerName}}'
```

如需按快照创建日期（控制器本地时区）对镜像分组，同时保留沙箱、容器和快照标签：

```yaml
controller:
  snapshot:
    registry: registry.example.com/team
    imageURITemplate: '{{.Registry}}/library/snapshots-{{date "2006-01-02" .SnapshotCreationTime}}:{{.SandboxName}}-{{.ContainerName}}-{{.SnapshotTag}}'
```

日期函数遵循 Helm 的函数名称和参数顺序，并接受 Go `time.Time`，例如 `SnapshotCreationTime`：

```text
{{date "2006-01-02" .SnapshotCreationTime}}
{{dateInZone "2006-01-02" .SnapshotCreationTime "UTC"}}
{{dateInZone "2006-01-02-15-04-05" .SnapshotCreationTime "+08:00"}}
{{dateInZone "2006-01-02" .SnapshotCreationTime "America/New_York"}}
```

`date` 使用 Go 的 `time.Local`。`dateInZone` 接受 `Local` 或空时区（同样使用默认值）、`UTC` 或 `Z`、带符号的 `HH:MM` 偏移量（小时 00–23，分钟 00–59），或 IANA `Area/Location` 名称。固定偏移量（如 `+05:30`）不随夏令时变化；具名时区遵循快照创建时刻的规则。未知时区和格式错误的偏移量会导致模板执行失败。这些辅助函数会拒绝零值时间戳，而不是用当前时间代替。

在 Unix 上，本地时区来自 `TZ`，未设置时取 `/etc/localtime`。在 Kubernetes 中，这是控制器容器的时区；请设置其 `TZ` 环境变量或提供其时区文件来选择预期的本地时区。当操作系统没有时区数据时，控制器内嵌了具名时区的数据。

格式化使用 Go 的标准布局：`2006-01-02` 表示日期，`20060102-150405` 表示紧凑时间戳，`2006-01-02-15-04-05.000` 表示毫秒。不支持 Java 风格的 `yyyy-MM-dd` 模式。布局在其模板位置必须产生合法的镜像引用：空格、冒号或大写月份名称不适合出现在仓库名称中，会导致校验失败。

原生方法仍然可用：`.SnapshotCreationTime.Format "2006-01-02"` 使用本地时区，而 `.SnapshotCreationTime.UTC.Format "2006-01-02"` 显式选择 UTC。为早期仅 UTC 字段编写的模板，在控制器本地时区不是 UTC 时，应显式选择 UTC 以保持其日期不变。

源始终是快照创建时间戳，因此在相同的时区配置下，跨午夜重试会保持相同的日期。模板或时区设置变更时，已有的提交 Job 保留其已解析的目标。

对于 umbrella chart，将这些 values 嵌套在 `opensandbox-controller` 下。Helm 按字面传递模板，不会解析 Go 模板字段。对于 Kustomize 或直接的 Deployment，在 manager 容器上添加一个参数：

```yaml
- '--snapshot-image-uri-template={{.Registry}}/snapshots:{{.SnapshotUID}}-{{.ArtifactKind}}-{{.ContainerName}}'
```

控制器在创建提交 Job 之前渲染 URI。镜像提交器直接推送到该 URI，恢复使用记录的引用；不存在中间推送或重命名。QEMU rootfs 和 VM-state 产物使用相同的模板和原始快照标签。

模板必须产生合法的带标签且无 digest 的 OCI 镜像引用。自定义模板结果在记录并传给镜像提交器之前会被规范化为完全限定的引用：`snapshots:tag` 变为 `docker.io/library/snapshots:tag`，`team/snapshots:tag` 变为 `docker.io/team/snapshots:tag`。若要指向其他仓库，请包含显式的仓库主机（包含点号的主机名、带端口的主机，或 `localhost`）。空模板保持原始命名规则和仓库前缀不变。名称不会自动清洗或截断。请包含能让产物和快照彼此区分的字段；控制器会拒绝同一快照内的重复目标，但无法检测与其他快照的冲突。模板语法错误会阻止控制器启动；渲染错误、非法引用和重复目标会在创建提交 Job 之前使快照失败。变更在创建新的提交 Job 时生效；已有 Job 保留其已解析的目标，已完成的快照保留其记录的恢复引用。

### 运行时兼容性

内置的 `rootfs-v1` 镜像提交器要求容器运行时暴露兼容的 containerd 任务暂停/恢复和可写快照 API。handler 为 `runsc` 的 gVisor RuntimeClass 目前不满足该契约。Lifecycle server 会在持久化快照记录或创建 `SandboxSnapshot` 资源之前，拒绝为这些工作负载创建公共快照，同时保持运行中的沙箱不受影响。

原生 gVisor 的 checkpoint/restore 需要专门的快照后端，`rootfs-v1` 提交器不提供该能力。

### 提交 Job

控制器为每次暂停创建一个短生命周期的 Kubernetes `Job`：

- **Job 名称**：`<snapshotName>-commit`
- **节点亲和**：运行在**与源 Pod 相同的节点**上（需要访问 containerd socket）
- **超时**：10 分钟（`ActiveDeadlineSeconds`）
- **TTL**：完成后 5 分钟（`TTLSecondsAfterFinished`）
- **镜像**：`image-committer`（可通过控制器 `--image-committer-image` 参数配置）

提交 Job 从源节点挂载主机 containerd socket，并以 UID 0 运行。这赋予了 `image-committer` 镜像节点级容器运行时访问权限。请只使用可信镜像，最好通过 digest 固定或由准入策略控制。

内置镜像提交器直接使用 containerd API 进行容器查找、任务暂停/取消暂停、可写快照镜像创建和仓库推送。提交前，它会验证节点的内容存储中包含节点平台所需的每个基础镜像 blob，并从原始镜像 digest 拉取缺失的 blob；这避免了 CRI 在镜像解包期间丢弃压缩内容后的快照失败。源仓库默认使用 HTTPS。对于需要跳过 TLS 验证或使用纯 HTTP 的可信源仓库，通过 image-committer Pod 模板向 `commit` 容器注入 `SOURCE_IMAGE_REGISTRY_INSECURE=true`。可复用的 Go 契约和默认实现暴露在 [`pkg/imagecommitter`](https://github.com/opensandbox-group/OpenSandbox/tree/main/kubernetes/pkg/imagecommitter)；特定于供应商的二进制程序可以提供凭据提供程序并复用标准 CLI 契约。Job 还保留主机 `/run/containerd/fifo` 挂载，以便兼容的实现可以使用 containerd task exec。内置实现使用的任何准备命令都是尽力而为的，不会改变提交接口。

运维控制的 Pod 模板可以添加标签、注解、ServiceAccount、调度设置、init 容器、sidecar，以及必需 `commit` 容器上的资源、`env`、`envFrom` 等设置。模板是合并基底，控制器生成的不变量会覆盖冲突的模板值。控制器保留源 `nodeName`、重启策略、提交器镜像、镜像拉取策略、命令、参数、安全上下文、终止消息设置，以及必需的环境变量、挂载和卷。环境变量、挂载和卷按名称合并，冲突时以控制器必需值优先。额外的常规 sidecar 必须在 Job 完成前终止。

模板引用的任何 ServiceAccount 或准入配置必须存在于每个沙箱命名空间中。取消暂停 Job 不会接收提交 Pod 模板，因为它们不访问镜像仓库。

如果提交 Job 失败，控制器会在同一节点上创建一个尽力而为的 `<snapshotName>-unpause` Job，以取消暂停可能因提交器异常退出而遗留暂停状态的源容器。

删除 `SandboxSnapshot` 会清理 Kubernetes 的提交/取消暂停 Job，但不会从仓库中删除已推送的 OCI 镜像。反复的暂停周期会创建 `snap-gen<N>` 等标签；请在外部配置仓库的保留或垃圾回收策略。

### 监控

检查 SandboxSnapshot 状态：

```bash
kubectl get sandboxsnapshot -n <namespace>
# NAME          PHASE       SANDBOX_ID     AGE
# my-snapshot   Succeed     my-sandbox     5m

kubectl describe sandboxsnapshot my-snapshot -n <namespace>
```

需要关注的关键字段：

- `status.phase`：`Pending` → `Committing` → `Succeed` / `Failed`
- `status.conditions`：附带人类可读消息的就绪或失败原因
- `status.containers`：每个已提交容器的镜像 URI
- `status.sourcePodName` / `status.sourceNodeName`：快照解析出的执行源

#### 监控沙箱状态流转

通过 Lifecycle API 检查沙箱状态时，你会看到中间状态：

**暂停流程：**
```bash
curl http://localhost:8080/v1/sandboxes/{sandbox_id}
# Response during pause:
{
  "id": "my-sandbox",
  "status": {
    "state": "Pausing",
    "reason": "PAUSING",
    "message": "Pausing sandbox"
  }
}
```

**恢复流程：**
```bash
curl http://localhost:8080/v1/sandboxes/{sandbox_id}
# Response during resume:
{
  "id": "my-sandbox",
  "status": {
    "state": "Resuming",
    "reason": "RESUMING",
    "message": "Resuming sandbox"
  }
}
```

---

## SandboxSnapshot 参考

### Spec 字段

| 字段 | 类型 | 说明 |
|-------|------|-------------|
| `sandboxName` | string | 同一命名空间内目标 `BatchSandbox` 的名称 |

### Status 字段（由控制器设置）

| 字段 | 类型 | 说明 |
|-------|------|-------------|
| `phase` | string | `Pending` / `Committing` / `Succeed` / `Failed` |
| `conditions` | list | 带 reason 和 message 的 `Ready` / `Failed` 条件 |
| `sourcePodName` | string | 用于提交的 Pod 名称 |
| `sourceNodeName` | string | 提交 Job 运行所在的节点 |
| `containers` | list | 每个容器的 `{containerName, imageUri, imageDigest}`。`imageDigest` 是镜像配置 digest，保留了早期 image-committer 版本的镜像 ID 语义。 |
| `observedGeneration` | int | 最近处理的 spec 代数 |

---

## 故障排除

### 1. 快照卡在 `Failed` —— 找不到推送 Secret

**原因**：控制器管理器配置的 `--snapshot-push-secret` 在沙箱命名空间中不存在。

**解决方法**：
```bash
kubectl get secret registry-snapshot-push-secret -n <namespace>
# If missing:
kubectl create secret docker-registry registry-snapshot-push-secret \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<token> \
  -n <namespace>
```

控制器**在**创建提交 Job **之前**验证 Secret 是否存在（fail-fast）。Secret 创建后，触发一次新的暂停循环。

---

### 2. 快照长时间卡在 `Committing`

**检查提交 Job 及其 Pod：**

```bash
kubectl get job -n <namespace> -l sandbox.opensandbox.io/snapshot=<snapshotName>
kubectl describe pod <commit-pod-name> -n <namespace>
```

**常见原因：**

| 现象 | 原因 | 解决方法 |
|---------|-------|---------|
| `ContainerCreating` 超过 30 秒 | Secret 缺失或类型错误 | 以 `kubernetes.io/dockerconfigjson` 类型重建 Secret |
| `FailedMount` 事件 | 找不到 Secret | 参见上文问题 #1 |
| Pod 在运行但 Job 永不完成 | 节点无法访问镜像仓库 | 检查节点到仓库的网络连通性 |
| Pod 日志中出现 `unauthorized` | Secret 中的凭据错误 | 用 `kubectl get secret ... -o yaml` 核对 Secret 内容 |

---

### 3. Secret 类型错误

Docker 镜像仓库 Secret **必须**是 `kubernetes.io/dockerconfigjson` 类型。通用 Secret（`Opaque`）会导致 `FailedMount` 错误。

```bash
# Check secret type
kubectl get secret registry-snapshot-push-secret -o jsonpath='{.type}'
# Expected: kubernetes.io/dockerconfigjson

# If wrong type, delete and recreate:
kubectl delete secret registry-snapshot-push-secret
kubectl create secret docker-registry registry-snapshot-push-secret \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<token>
```

---

### 4. 镜像仓库不可达（`Committing` → 超时后 `Failed`）

**症状**：提交 Job Pod 启动、运行一段时间后因推送错误失败。

**检查：**

```bash
# Inspect commit Pod logs
kubectl logs <commit-pod-name> -n <namespace>

# Test registry connectivity from a node
kubectl run registry-test --rm -it --image=alpine -- \
  wget -O- https://<registry>/v2/ --timeout=5
```

**常见原因：**
- 镜像仓库位于防火墙后，集群节点无法访问
- 自签名 TLS 证书不被 containerd 信任
- 镜像仓库 URL 错误（http 与 https 混淆）

---

### 5. 恢复请求被接受但运行时 Pod 启动失败

**原因**：快照镜像无法拉取。

```bash
kubectl describe pod <resumed-pod-name> -n <namespace>
# Look for: ErrImagePull or ImagePullBackOff
```

**检查：**
- `--resume-pull-secret` 已正确配置且 Secret 存在于该命名空间
- 拉取镜像的节点可以访问镜像仓库
- 暂停期间快照镜像已成功推送（检查 `status.containers`）

---

### 6. SandboxSnapshot 未被处理（无状态）

**原因**：OpenSandbox 控制器未运行。

```bash
kubectl get pods -n opensandbox-system
kubectl logs -n opensandbox-system deployment/opensandbox-controller-manager
```

---

## 获取帮助

- **文档**：[OpenSandbox GitHub](https://github.com/opensandbox-group/OpenSandbox)
- **Issue**：[GitHub Issues](https://github.com/opensandbox-group/OpenSandbox/issues)
- **Kubernetes 控制器**：[Kubernetes 概览](/architecture/control-plane/operator)

## 测量恢复延迟

包含原始样本、P50/P95、失败报告和明确缓存条件的可选 rootfs 基准测试，请参见 [Kubernetes 恢复基准](/guides/kubernetes-resume-benchmark)。
