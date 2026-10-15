---
title: QEMU VMState 快照
description: 配置并运维在 runc 沙箱内运行的 QEMU 的进程级暂停与恢复。
---

# QEMU VMState 快照

本指南介绍如何准备、部署和验证支持进程级暂停与恢复的 QEMU-in-runc 工作负载。该工作负载在普通的 runc 容器内运行 QEMU；不使用虚拟化的 Kubernetes `RuntimeClass`。

该模式为实验性功能。它会还原 QEMU Guest 的内存、vCPU 和可迁移的设备状态。外层 runc 容器中的其他进程按照 `BatchSandbox` Pod 模板重新启动。

最初的 `qemu-v1` 实现支持对同一个 `BatchSandbox` 的暂停与恢复。公共快照克隆 API 尚不支持还原 QEMU VMState，因为其快照记录不持久化完整的 Pod 模板和 QEMU 启动计划。独立的公共快照操作在发布产物后会恢复其源 VM。

## 状态与产物模型

| 状态 | 存储位置 |
|---|---|
| 外层容器 rootfs | 现有的 rootfs 快照镜像 |
| 可写的 Guest qcow2 overlay | Rootfs 快照镜像；overlay 不得位于 Kubernetes 卷挂载之下 |
| Guest 内存、vCPU 和模拟设备状态 | VMState 镜像，包含 zstd 压缩的 QEMU 迁移流 |
| 不可变的 Guest 基础镜像 | 原始镜像层，或一个独立可用的只读卷 |
| PVC、hostPath、云盘、网络对端状态 | OpenSandbox 不复制 |

两类镜像都会推送到配置的镜像仓库，并在同一个 `SandboxSnapshot.status` 中以 manifest digest 记录。恢复从不依赖可变标签。

## 准备 QEMU 工作负载镜像

QEMU 支持是工作负载镜像与 OpenSandbox 之间的契约。镜像必须：

1. 以 QMP Unix socket 启动 QEMU；
2. 从实际生效的 QEMU 设置生成 OpenSandbox 启动清单（launch manifest）；
3. 将 v1 捕获的每个可写 Guest overlay 保留在容器 rootfs 内；
4. 识别 OpenSandbox 恢复环境并添加 QEMU `-incoming`；
5. 只有在 QMP socket 和 Guest 服务可用之后才变为 Ready；
6. 对于内部 BatchSandbox 暂停快照，确保工作负载控制的进程不会通过 QMP（例如 `cont`）独立恢复源 VM，也不会在迁移导出之后、源 Pod 被删除之前重启源 QEMU 进程（这不限制快照 worker 在失败时或在独立公共快照期间恢复源 VM）。

### 声明 Pod 模板契约

在创建 QEMU 容器的 Pod 模板上设置以下注解。对于独立沙箱，使用 `BatchSandbox.spec.template.metadata`；对于 Pool 沙箱，使用 `Pool.spec.template.metadata`，使分配出的 Pod 携带该契约。

```yaml
annotations:
  sandbox.opensandbox.io/checkpoint-provider: qemu
  sandbox.opensandbox.io/qemu-container: qemu
  sandbox.opensandbox.io/qemu-qmp-socket: /run/qemu/qmp.sock
  sandbox.opensandbox.io/qemu-launch-manifest: /run/qemu/launch.json
  # Optional: constrain restore to compatible nodes.
  sandbox.opensandbox.io/qemu-required-node-class: shenlong-v1
```

| 注解 | 必填 | 用途 |
|---|---:|---|
| `checkpoint-provider` | 是 | 选择 `qemu` 提供方。OpenSandbox 不扫描进程名称。 |
| `qemu-container` | 是 | 指定拥有 QEMU、QMP 和启动清单的 Pod 容器。 |
| `qemu-qmp-socket` | 是 | 该容器内 QMP Unix socket 的干净绝对路径。 |
| `qemu-launch-manifest` | 是 | 该容器内 OpenSandbox 启动清单的干净绝对路径。 |
| `qemu-required-node-class` | 否 | 将恢复限制在带有匹配 OpenSandbox QEMU node-class 标签的节点上。 |

这些路径是容器内路径，不需要挂载进控制器 Pod。快照 worker 通过节点的容器运行时访问目标容器。

### 生成 OpenSandbox 启动清单

启动清单由 OpenSandbox 定义。它不是原生 QEMU 文件，QEMU 也不会自动创建它。工作负载镜像的拥有者负责在 Pod 变为 Ready 之前生成它。

请在容器入口点中根据构建 QEMU 命令所用的同一组实际生效的值生成该文件。当 CPU、内存、磁盘或设备可能通过环境变量或 Pod 模板变化时，不要把静态文件烘焙进 Docker 镜像。

启动清单示例：

```json
{
  "formatVersion": "qemu-v1",
  "architecture": "amd64",
  "qemuVersion": "6.2.0",
  "machineType": "pc-q35-6.2",
  "cpuModel": "host",
  "vcpus": 2,
  "memoryBytes": 536870912,
  "qemuConfigDigest": "sha256:...",
  "disks": [
    {
      "id": "osdisk",
      "overlayPath": "/vm/state.qcow2",
      "capture": "rootfs"
    }
  ]
}
```

| 字段 | 含义 |
|---|---|
| `formatVersion` | 必须为 `qemu-v1`。 |
| `architecture` | QEMU 使用的 Guest 宿主架构，例如 `amd64`。 |
| `qemuVersion` | 运行中 QEMU 进程报告的版本。worker 通过 QMP 校验该值。 |
| `machineType` | 显式带版本的机器类型，例如 `pc-q35-6.2`。 |
| `cpuModel` | 实际生效的 QEMU CPU 模型。`host` 通常要求恢复节点同构。 |
| `vcpus` | 实际生效的 vCPU 数量。 |
| `memoryBytes` | 实际生效的 Guest 内存大小（字节）。 |
| `qemuConfigDigest` | 工作负载生成的、标识兼容性敏感 QEMU 配置的 SHA-256 值。 |
| `disks` | 可写 overlay 及其捕获策略。v1 仅支持 `capture: rootfs`。 |

`qemuConfigDigest` 在 v1 中是一个不透明的兼容性值。请基于机器、CPU、内存、固件、磁盘、网络和设备设置的规范化表示确定性地计算它。排除 PID、时间戳、QMP 路径和生成的 socket 名称等瞬态值。OpenSandbox 记录该 digest，但不会用它重建 QEMU 命令行。

请原子性地写入清单，避免并发快照读到不完整的 JSON 文件：

```bash
runtime_dir=/run/qemu
manifest_tmp="$runtime_dir/launch.json.tmp"
manifest="$runtime_dir/launch.json"

mkdir -p "$runtime_dir"
cat >"$manifest_tmp" <<EOF
{
  "formatVersion": "qemu-v1",
  "architecture": "amd64",
  "qemuVersion": "$qemu_version",
  "machineType": "$machine_type",
  "cpuModel": "$cpu_model",
  "vcpus": $vcpus,
  "memoryBytes": $memory_bytes,
  "qemuConfigDigest": "$qemu_config_digest",
  "disks": [
    {"id":"osdisk","overlayPath":"/vm/state.qcow2","capture":"rootfs"}
  ]
}
EOF
mv "$manifest_tmp" "$manifest"
```

参考 E2E 入口点展示了如何用同一组变量构建清单和 QEMU 参数。参见[参考入口点](https://github.com/alibaba/OpenSandbox/blob/main/kubernetes/test/e2e_qemu/testdata/entrypoint.sh)。

### 暴露 QMP 并支持恢复

在 QEMU 启动时创建声明的 QMP Unix socket：

```bash
qemu_args+=(
  -qmp "unix:/run/qemu/qmp.sock,server=on,wait=off"
)
```

QMP 控制运行中的 QEMU 进程并承载迁移流。启动清单则单独声明兼容性与磁盘捕获意图，这些信息无法仅凭 QMP 可靠地还原。

工作负载入口点必须检测 `OPENSANDBOX_RESTORE_MODE=qemu-v1`，并使用 `OPENSANDBOX_VMSTATE_DIR` 中提供的加载器流启动 QEMU：

```bash
if [[ "${OPENSANDBOX_RESTORE_MODE:-}" == "qemu-v1" ]]; then
  test -x "$OPENSANDBOX_VMSTATE_DIR/vmstate-loader"
  qemu_args+=(
    -incoming
    "exec:$OPENSANDBOX_VMSTATE_DIR/vmstate-loader stream --dir $OPENSANDBOX_VMSTATE_DIR"
  )
fi
```

OpenSandbox 注入恢复目录和加载器；完整的 QEMU 命令行仍由工作负载负责。它必须在消费 incoming 流之前重建相同的机器、CPU、内存、固件、磁盘、网络和设备拓扑。

### 将可写 overlay 保留在 rootfs 内

每个以 `capture: rootfs` 声明的磁盘都必须是 QEMU 容器可写 rootfs 中的文件。它不得解析到 Kubernetes `volumeMount` 或 `volumeDevice` 之下，包括 PVC、hostPath、projected 卷或 `emptyDir`。`nerdctl commit` 不会捕获这些挂载，快照 worker 会拒绝该配置，而不是创建不完整的快照。

不可变的基础镜像可以来自原始镜像层，或另一个独立可用的只读源。已有的用户 init 容器会在恢复时先于注入的 VMState 加载器再次运行，因此它们必须是幂等的，且不得覆盖已还原的可写 overlay。

## 暂停与恢复序列

1. 控制器解析带注解的 QEMU 容器，并在同一节点上调度快照 Job。
2. worker 校验每个声明为 `capture: rootfs` 的可写 overlay 都位于容器的 Kubernetes 卷挂载路径之外。
3. worker 使用 QMP 迁移导出 VM 状态，用 zstd 压缩后放入标准容器镜像。
4. QEMU 进入迁移后状态后，worker 冻结配套的外层容器（QEMU 容器保持不冻结，以避免在 QEMU 处于 postmigrate 静默状态时出现内核 worker 冻结死锁），提交它们的根文件系统，并推送两类镜像。
5. 控制器原子性地发布 rootfs 和 VMState manifest digest，然后删除源 Pod。
6. 恢复时，一个加载器 init 容器在 `emptyDir` 中还原并校验 VMState 文件。它被追加在所有用户 init 容器之后。
7. 重建的 QEMU 容器消费迁移流。其他外层容器进程按 Pod 模板正常启动。

不使用 `savevm`。它创建的是与 qcow2 存储耦合的内部快照，不是这里使用的实时 Guest 内存传输方式。

## 使用 Helm 进行生产部署

没有 QEMU 专用的 Helm 功能开关。Helm 安装集群级的快照能力；每个 `BatchSandbox` 或 `Pool` 的 Pod 模板通过上述注解选择加入。仅 rootfs 的工作负载继续沿用现有行为。

### 前提条件

- Linux 节点将 `/dev/kvm` 暴露给 QEMU 容器。
- 镜像仓库必须接受 Docker schema 2 或 OCI 镜像 manifest，并且快照 Job 和 kubelet/containerd 都能访问。
- 恢复时 QEMU 版本、带版本的机器类型、CPU 模型、vCPU 数量、内存大小、设备配置 digest 以及可选的 node class 必须匹配。
- 快照 Job 需要主机 containerd 运行时目录、主机 PID 命名空间和 `SYS_PTRACE`。它已经在节点级信任边界内运行，因为它控制着 containerd socket。Pod 安全与准入策略必须显式允许该 Job。
- VMState 可能包含来自 Guest 内存的凭据和用户数据。请按敏感持久存储的同等标准实施访问控制、加密、保留和删除策略。
- 根据实测的压缩负载大小规划仓库容量。压缩效果取决于 Guest 内存内容，不保证很小。

控制器镜像和 image-committer 镜像都必须来自支持 `qemu-v1` 的版本。特别是，不要依赖 chart 中较旧的 `controller.snapshot.imageCommitterImage` 默认值；请显式覆盖。

### 创建仓库凭据

提交 Job 和恢复后的 Pod 运行在沙箱命名空间中，因此引用的 Secret 必须存在于每个使用快照与恢复的命名空间中：

```bash
kubectl -n sandbox-team create secret docker-registry snapshot-registry \
  --docker-server=registry.example.com \
  --docker-username='<username>' \
  --docker-password='<password-or-token>'
```

当凭据同时具备推送、image-committer 拉取和恢复拉取三种权限时，一个 Secret 可以三者共用。生产部署可以在每个沙箱命名空间中使用同名但最小权限分离的 Secret。

### 配置并安装 chart

创建 values 文件：

```yaml
controller:
  image:
    repository: registry.example.com/opensandbox/controller
    tag: qemu-v1
  snapshot:
    imageCommitterImage: registry.example.com/opensandbox/image-committer:qemu-v1
    containerdSocketPath: /var/run/containerd/containerd.sock
    commitJobTimeout: 15m
    registry: registry.example.com/opensandbox-snapshots
    registryInsecure: false
    snapshotPushSecret: snapshot-registry
    imageCommitterPullSecret: snapshot-registry
    resumePullSecret: snapshot-registry
```

安装或升级控制器。在全新集群上，先安装基础 chart —— 控制器 chart 不再随附 CRD：

```bash
helm upgrade --install base ./manifests/charts/base

helm upgrade --install opensandbox-controller \
  ./manifests/charts/controller \
  --namespace opensandbox-system \
  --create-namespace \
  --values qemu-snapshot-values.yaml
```

将 CRD 同步到基础 chart（`make -C manifests helm-gen-crds`）并升级 `base`，使更新后的 `SandboxSnapshot` CRD 生效。在创建工作负载之前，验证已部署的能力：

```bash
kubectl -n opensandbox-system rollout status \
  deployment/opensandbox-controller-manager
kubectl -n opensandbox-system get deployment,pod
kubectl get crd sandboxsnapshots.sandbox.opensandbox.io
kubectl get crd sandboxsnapshots.sandbox.opensandbox.io \
  -o jsonpath='{.spec.versions[0].schema.openAPIV3Schema.properties.status.properties.virtualMachine.type}{"\n"}'
```

确认集群准入允许节点级信任的快照 Job。QEMU 模式需要主机 PID 命名空间、`SYS_PTRACE` 和主机 containerd 运行时目录。QEMU 工作负载本身需要 `/dev/kvm`，并且通常以特权方式运行，除非平台提供了更细粒度的设备权限。

对于异构集群，为兼容的恢复节点打标签，并在工作负载上声明匹配的 class：

```bash
kubectl label node <node> \
  sandbox.opensandbox.io/qemu-node-class=shenlong-v1
```

```yaml
metadata:
  annotations:
    sandbox.opensandbox.io/qemu-required-node-class: shenlong-v1
```

默认情况下，快照镜像名称在配置的仓库前缀下生成。QEMU 沙箱会产生普通的容器 rootfs 镜像（如 `<prefix>/<sandbox>-<container>:<tag>`）和一个 VMState 镜像（如 `<prefix>/<sandbox>-vmstate:<tag>`）。两类产物名称都可以通过 [`--snapshot-image-uri-template`](/guides/pause-resume#custom-image-names) 自定义；VM-state 产物的 `ContainerName` 和 `ArtifactKind` 为 `vmstate`。恢复时解析并使用它们不可变的 manifest digest。

## 验证

专门的 E2E 会从 `Pool` 分配一个预热的 QEMU-in-runc Pod，向匿名 mmap、裸 Guest 磁盘和外层 rootfs 写入三个独立值，对两类镜像做快照，并还原出一个独立 Pod。它验证 Pool 的补充与解绑、不可变的 rootfs 和 VMState 镜像 digest、加载器完成情况、快照清理、全部三个值、Guest boot ID 以及实时计数器：

```bash
cd kubernetes
make test-e2e-qemu
```

该测试需要 Linux amd64、Docker、Kind、kubectl 和 `/dev/kvm`。默认情况下，它会在 Kind 集群中部署一个隔离的 `registry:2` Pod。设置 `QEMU_E2E_SNAPSHOT_REGISTRY` 和 `QEMU_E2E_DOCKER_CONFIG` 可针对一个需要认证的外部仓库验证相同的流程；两个仓库分别是 `<prefix>/qemu-rootfs` 和 `<prefix>/qemu-vmstate`。设置 `KEEP_QEMU_E2E_CLUSTER=true` 可保留专用 Kind 集群以便诊断。所有可覆盖项见 [E2E README](https://github.com/alibaba/OpenSandbox/tree/main/kubernetes/test/e2e_qemu)。

### 可复用的 Kind 验证环境

E2E 测试框架可以构建镜像、创建启用 KVM 的专用 Kind 集群、部署控制器和仓库、运行参考 Pool 流程，并保留环境供人工检查：

```bash
cd kubernetes
KEEP_QEMU_E2E_CLUSTER=true make test-e2e-qemu
```

生成的 context 为 `kind-opensandbox-qemu-vmstate-e2e`。控制器运行在 `opensandbox-system`，而集群内的 `registry:2` Pod 和自动化测试对象运行在 `qemu-vmstate-e2e`：

```bash
kubectl --context kind-opensandbox-qemu-vmstate-e2e \
  -n opensandbox-system get deploy,pod
kubectl --context kind-opensandbox-qemu-vmstate-e2e \
  -n qemu-vmstate-e2e get pod,service,pool,batchsandbox
```

如需改为使用需要认证的外部仓库，请提供其仓库前缀和本地 Docker `config.json`；测试框架会创建临时的 Kubernetes 拉取/推送 Secret，且不打印其内容：

```bash
QEMU_E2E_SNAPSHOT_REGISTRY=registry.example.com/team \
QEMU_E2E_DOCKER_CONFIG=/path/to/config.json \
KEEP_QEMU_E2E_CLUSTER=true \
make test-e2e-qemu
```

以下命令使用独立的 `qemu-manual` 命名空间，因此不需要先删除保留的 E2E 对象。

### 在准备好的 Kind 集群中手动验证

该流程有意将生命周期操作留给运维人员。它假设控制器、快照仓库、image-committer 镜像和演示 QEMU 镜像已安装在一个带 `/dev/kvm` 的 Kind 集群中。从仓库根目录开始，选择该集群并只创建独立的 `BatchSandbox`：

```bash
export OSB_QEMU_CONTEXT=kind-opensandbox-qemu-vmstate-e2e
export OSB_QEMU_NAMESPACE=qemu-manual
export OSB_QEMU_SANDBOX=qemu-standalone
export OSB_QEMU_POD=qemu-standalone-0

test -c /dev/kvm
kubectl --context "$OSB_QEMU_CONTEXT" get nodes
kubectl --context "$OSB_QEMU_CONTEXT" -n opensandbox-system \
  rollout status deployment/opensandbox-controller-manager
kubectl --context "$OSB_QEMU_CONTEXT" apply \
  -f kubernetes/config/samples/alibaba/qemu-vmstate/standalone.yaml
kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  wait --for=condition=Ready pod/"$OSB_QEMU_POD" --timeout=180s
```

暂停之前，核对快照 worker 将使用的确切契约：

```bash
kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  get pod "$OSB_QEMU_POD" \
  -o jsonpath='{.metadata.annotations}{"\n"}'
kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  exec "$OSB_QEMU_POD" -c qemu -- \
  test -S /run/qemu-e2e/qmp.sock
kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  exec "$OSB_QEMU_POD" -c qemu -- \
  cat /run/qemu-e2e/launch.json
```

如果带注解的容器、socket 路径、清单路径或实际生效的 QEMU 设置有任何不一致，请不要继续。

演示 Guest 在同一个 PID 1 进程中运行 HTTP 服务和可变的内存映射。QEMU 用户态网络将外层容器的回环端口 `18080` 转发到 Guest 端口 `8080`；该服务不会在 Pod IP 上暴露。

| 端点 | 用途 |
| --- | --- |
| `GET /healthz` | 验证恢复后的 Guest 进程正在服务请求 |
| `PUT /value` | 在进程内存映射中存储一个最多 512 字节的值 |
| `GET /value` | 读取当前内存映射值 |
| `GET /status` | 读取 PID、Guest boot ID、基于 mmap 的实时计数器和值 |
| `PUT /disk` | 在裸可写 Guest 磁盘中存储一个标记 |
| `GET /disk` | 读取裸 Guest 磁盘标记 |

定义一个阶段等待函数，并在 Guest 内存、Guest 磁盘和外层容器 rootfs 中写入三个独立标记：

```bash
wait_for_sandbox_phase() {
  local wanted=$1
  local phase=
  for _ in $(seq 1 180); do
    phase=$(kubectl --context "$OSB_QEMU_CONTEXT" \
      -n "$OSB_QEMU_NAMESPACE" get batchsandbox "$OSB_QEMU_SANDBOX" \
      -o jsonpath='{.status.phase}' 2>/dev/null || true)
    if [[ "$phase" == "$wanted" ]]; then
      return 0
    fi
    sleep 2
  done
  echo "timed out waiting for phase=$wanted; last phase=$phase" >&2
  return 1
}

export OSB_MEMORY_TOKEN="MANUAL-MMAP-$(date -u +%Y%m%dT%H%M%SZ)"
export OSB_DISK_TOKEN="MANUAL-DISK-$(date -u +%Y%m%dT%H%M%SZ)"
export OSB_ROOTFS_TOKEN="MANUAL-ROOTFS-$(date -u +%Y%m%dT%H%M%SZ)"

kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  exec "$OSB_QEMU_POD" -c qemu -- \
  curl --fail --silent --show-error --request PUT \
  --data-binary "$OSB_MEMORY_TOKEN" http://127.0.0.1:18080/value
kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  exec "$OSB_QEMU_POD" -c qemu -- \
  curl --fail --silent --show-error --request PUT \
  --data-binary "$OSB_DISK_TOKEN" http://127.0.0.1:18080/disk
kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  exec "$OSB_QEMU_POD" -c qemu -- sh -c \
  'mkdir -p /var/lib/opensandbox && printf "%s" "$1" > /var/lib/opensandbox/rootfs-marker' \
  sh "$OSB_ROOTFS_TOKEN"

OSB_BEFORE=$(kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  exec "$OSB_QEMU_POD" -c qemu -- \
  curl --fail --silent --show-error http://127.0.0.1:18080/status)
OSB_SOURCE_UID=$(kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  get pod "$OSB_QEMU_POD" -o jsonpath='{.metadata.uid}')
printf 'before: %s\nsource uid: %s\n' "$OSB_BEFORE" "$OSB_SOURCE_UID"
```

在另一个终端中，观察暂停与恢复涉及的资源：

```bash
watch -n 1 kubectl --context "$OSB_QEMU_CONTEXT" \
  -n "$OSB_QEMU_NAMESPACE" get batchsandbox,sandboxsnapshot,pod,job
```

初始清单有意省略 `spec.pause`：对全新对象将其设置为 `false` 会被解释为显式的恢复请求。通过添加 `spec.pause: true` 暂停沙箱，然后检查两个不可变的镜像产物：

```bash
kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  edit batchsandbox "$OSB_QEMU_SANDBOX"
wait_for_sandbox_phase Paused

export OSB_SNAPSHOT_NAME="$OSB_QEMU_SANDBOX-pause"
kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  get sandboxsnapshot "$OSB_SNAPSHOT_NAME" -o yaml
kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  wait --for=delete pod/"$OSB_QEMU_POD" --timeout=120s
```

快照必须报告 `status.format: qemu-v1`、`status.containers` 下的 rootfs 镜像 digest，以及 `status.virtualMachine` 下不同的 VMState 镜像 digest 和非零压缩大小。

通过把 `spec.pause` 改回 `false` 来恢复，然后验证进程级连续性和两个文件系统层：

```bash
kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  edit batchsandbox "$OSB_QEMU_SANDBOX"
wait_for_sandbox_phase Succeed
kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  wait --for=condition=Ready pod/"$OSB_QEMU_POD" --timeout=180s

OSB_AFTER=$(kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  exec "$OSB_QEMU_POD" -c qemu -- \
  curl --fail --silent --show-error http://127.0.0.1:18080/status)
OSB_AFTER_DISK=$(kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  exec "$OSB_QEMU_POD" -c qemu -- \
  curl --fail --silent --show-error http://127.0.0.1:18080/disk)
OSB_AFTER_ROOTFS=$(kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  exec "$OSB_QEMU_POD" -c qemu -- cat /var/lib/opensandbox/rootfs-marker)
OSB_RESTORED_UID=$(kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  get pod "$OSB_QEMU_POD" -o jsonpath='{.metadata.uid}')

OSB_BEFORE_BOOT_ID=$(printf '%s\n' "$OSB_BEFORE" | sed -n 's/.*"boot_id":"\([^"]*\)".*/\1/p')
OSB_AFTER_BOOT_ID=$(printf '%s\n' "$OSB_AFTER" | sed -n 's/.*"boot_id":"\([^"]*\)".*/\1/p')
OSB_BEFORE_COUNTER=$(printf '%s\n' "$OSB_BEFORE" | sed -n 's/.*"counter":\([0-9]*\).*/\1/p')
OSB_AFTER_COUNTER=$(printf '%s\n' "$OSB_AFTER" | sed -n 's/.*"counter":\([0-9]*\).*/\1/p')

[[ "$OSB_SOURCE_UID" != "$OSB_RESTORED_UID" ]]
[[ "$OSB_BEFORE_BOOT_ID" == "$OSB_AFTER_BOOT_ID" ]]
((OSB_AFTER_COUNTER > OSB_BEFORE_COUNTER))
[[ "$OSB_AFTER" == *"\"value\":\"$OSB_MEMORY_TOKEN\""* ]]
[[ "$OSB_AFTER_DISK" == *"\"value\":\"$OSB_DISK_TOKEN\""* ]]
[[ "$OSB_AFTER_ROOTFS" == "$OSB_ROOTFS_TOKEN" ]]

printf 'after: %s\ndisk: %s\nrootfs: %s\nrestored uid: %s\n' \
  "$OSB_AFTER" "$OSB_AFTER_DISK" "$OSB_AFTER_ROOTFS" "$OSB_RESTORED_UID"
kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  get pod "$OSB_QEMU_POD" -o jsonpath='{range .status.initContainerStatuses[*]}{.name}{"="}{.state.terminated.reason}{"\n"}{end}'
```

所有断言都必须返回零。Pod UID 必须变化、Guest boot ID 必须保持不变、实时计数器必须增加、全部三个标记必须匹配，且 VMState 加载器 init 容器必须报告 `Completed`。恢复成功后，内部 `SandboxSnapshot` 会被删除。结束后只删除测试对象；Kind 基础设施保持可用：

```bash
kubectl --context "$OSB_QEMU_CONTEXT" delete \
  -f kubernetes/config/samples/alibaba/qemu-vmstate/standalone.yaml
```

### 使用 Pool 手动验证

先创建 Pool 并等待其预热的 QEMU Pod。然后创建一个分配该 Pod 的 `BatchSandbox`：

```bash
kubectl --context "$OSB_QEMU_CONTEXT" apply \
  -f kubernetes/config/samples/alibaba/qemu-vmstate/pool.yaml
kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  get pool qemu-pool -w
```

Pool 报告就绪容量后，停止 watch 并执行：

```bash
kubectl --context "$OSB_QEMU_CONTEXT" apply \
  -f kubernetes/config/samples/alibaba/qemu-vmstate/pooled-sandbox.yaml
kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  get batchsandbox qemu-pooled -w
```

从分配注解中读取分配的 Pod 名称，并写入与独立场景相同的三个标记：

```bash
export OSB_POOLED_POD=$(kubectl --context "$OSB_QEMU_CONTEXT" \
  -n "$OSB_QEMU_NAMESPACE" get batchsandbox qemu-pooled \
  -o jsonpath='{.metadata.annotations.sandbox\.opensandbox\.io/alloc-status}' \
  | jq -r '.pods[0]')

export OSB_POOL_MEMORY_TOKEN="POOL-MMAP-$(date -u +%Y%m%dT%H%M%SZ)"
export OSB_POOL_DISK_TOKEN="POOL-DISK-$(date -u +%Y%m%dT%H%M%SZ)"
export OSB_POOL_ROOTFS_TOKEN="POOL-ROOTFS-$(date -u +%Y%m%dT%H%M%SZ)"

kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  exec "$OSB_POOLED_POD" -c qemu -- \
  curl --fail --silent --show-error --request PUT \
  --data-binary "$OSB_POOL_MEMORY_TOKEN" http://127.0.0.1:18080/value
kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  exec "$OSB_POOLED_POD" -c qemu -- \
  curl --fail --silent --show-error --request PUT \
  --data-binary "$OSB_POOL_DISK_TOKEN" http://127.0.0.1:18080/disk
kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  exec "$OSB_POOLED_POD" -c qemu -- sh -c \
  'mkdir -p /var/lib/opensandbox && printf "%s" "$1" > /var/lib/opensandbox/rootfs-marker' \
  sh "$OSB_POOL_ROOTFS_TOKEN"
```

运行 `kubectl edit batchsandbox qemu-pooled -n qemu-manual` 并添加 `spec.pause: true`。暂停成功后，验证 Pool 特有的交接行为：

```bash
kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  get batchsandbox qemu-pooled \
  -o jsonpath='phase={.status.phase}{" poolRef="}{.spec.poolRef}{" templateContainers="}{.spec.template.spec.containers[*].name}{"\n"}'
kubectl --context "$OSB_QEMU_CONTEXT" -n "$OSB_QEMU_NAMESPACE" \
  get sandboxsnapshot qemu-pooled-pause -o yaml
```

阶段必须是 `Paused`。控制器会把分配 Pod 的模板物化到 `BatchSandbox` 中并清除 `spec.poolRef`；这一解绑是有意为之，因此恢复时会创建独立 Pod 而不是回到 Pool。把 `spec.pause` 改回 `false`，等待 `qemu-pooled-0` 变为 Ready，然后对它调用 `GET /status`、`GET /disk` 并执行 rootfs 标记检查。

清理 Pool 示例但不删除 Kind 集群：

```bash
kubectl --context "$OSB_QEMU_CONTEXT" delete \
  -f kubernetes/config/samples/alibaba/qemu-vmstate/pooled-sandbox.yaml
kubectl --context "$OSB_QEMU_CONTEXT" delete \
  -f kubernetes/config/samples/alibaba/qemu-vmstate/pool.yaml
```

## 故障排除

| 现象 | 检查项 |
|---|---|
| `InvalidCheckpointContract` | 确认全部四个必填注解都存在于实际的 Pod 模板上，且指向已存在的容器。路径必须是干净且绝对的。 |
| 启动清单复制或解码失败 | exec 进入带注解的容器，读取确切路径，并验证入口点在就绪探测成功之前写入了完整的 JSON。 |
| QMP 探测失败 | 验证声明的路径是 Unix socket，且 QEMU 使用了 `server=on,wait=off`。检查是否有监管进程移除或替换了 socket。 |
| QEMU 版本不匹配 | 对比清单中的 `qemuVersion` 与运行中的二进制。应在容器启动时生成清单，而不是把过期版本烘焙进镜像。 |
| 可写磁盘被拒绝 | 检查 QEMU 容器上的每个 `volumeMount` 和 `volumeDevice`。`capture: rootfs` 的 overlay 不能位于任何挂载路径之下。 |
| 快照 Job 被准入拒绝 | 允许 image-committer Job 身份在支持快照的节点上使用主机 PID、`SYS_PTRACE` 和主机 containerd 运行时目录。 |
| 快照 Job `ImagePullBackOff` | 确保 `imageCommitterPullSecret` 存在于沙箱命名空间中，且能拉取配置的 image-committer 镜像。 |
| 恢复 Pod `ImagePullBackOff` | 确保 `resumePullSecret` 存在于沙箱命名空间中，且能拉取 rootfs 和 VMState 两个镜像仓库。 |
| QEMU 在消费 `-incoming` 时退出 | 将 QEMU 版本、机器类型、CPU 模型、vCPU 数量、内存、固件、磁盘、网络和设备拓扑与捕获的兼容性数据逐一对比。 |
| 恢复后 Pod 无法调度 | 对照节点的 `qemu-node-class` 标签，检查 `/dev/kvm`、节点亲和性和 `qemu-required-node-class`。 |

从快照状态和提交 Job 日志开始诊断：

```bash
kubectl -n <namespace> get sandboxsnapshot <name> -o yaml
kubectl -n <namespace> get job,pod \
  -l sandbox.opensandbox.io/sandbox-snapshot-name=<name>
kubectl -n <namespace> logs job/<commit-job-name> --all-containers
```
