---
title: Pod 供给失败恢复
description: Kubernetes 控制器如何在初始启动期间自动替换卡在供给失败（如镜像拉取）中的 BatchSandbox Pod。
---

# Pod 供给失败恢复

当 BatchSandbox Pod 被调度到不健康的节点上时，供给可能在 Pod 达到 Ready 之前就停滞。最常见的一类是主容器镜像拉取持续失败 —— 例如由节点磁盘满导致的 `ImagePullBackOff`：

```text
failed to prepare extraction snapshot ...: no space left on device
```

沙箱会永远停留在 `Pending`：kubelet 在同一个坏节点上不断重试，没有任何机制恢复沙箱。控制器会在初始启动阶段自动替换此类 Pod，让调度器有机会把重建的 Pod 调度到健康的节点上。

## 行为

控制器监视**从未 Ready 过**的沙箱（`Phase` 为 `Pending` 或为空）。当 Pod 在某个被跟踪的失败条件中停留时间超过阈值时，即视为卡住。当前跟踪的条件：

- **镜像拉取失败** —— `ImagePullBackOff` / `ErrImagePull`，通常由坏节点（例如磁盘满）引起。
- **Kubelet 准入拒绝** —— 调度器已放置 Pod，但 kubelet 的本地资源台账不同意（CPU、内存或临时存储不足；节点被封锁或未就绪；节点压力驱逐）。这些 Pod 以 `OutOfcpu`、`OutOfmemory`、`OutOfephemeral-storage`、`Evicted`、`NodeNotSchedulable`、`KubeletNotReady` 或 `UnexpectedAdmissionError` 等原因进入 `Failed`，而 Kubernetes 自己永远不会重试它们。被跟踪的拒绝原因集合可通过 `pod-recovery-admission-reasons` 配置（参见[配置](#configuration)）。

对于卡住的 Pod：

1. 控制器删除它。沙箱根据模板重建 Pod，该 Pod 像任何新 Pod 一样被重新调度。
2. 替换尝试有上限：在 `pod-recovery-max-attempts` 次替换之后，控制器停止并记录一条 `PodRecoveryLimitReached` 警告事件。沙箱保持 `Pending`，需要运维人员介入。
3. 更新沙箱模板（例如修复镜像）会重置替换计数。

::: tip 有意排除的情况
`CreateContainerConfigError` 是重调度无法解决的 spec 问题。终态失败（`PodFailed`、`ContainerFailed`、`InitContainerFailed`）保持既有的冻结语义。`CrashLoopBackOff` 不在覆盖范围内：移动一个崩溃的 Pod 通常无济于事，还可能掩盖应用缺陷。调度失败（`Unschedulable`）留给 kube-scheduler 自动重试。Pool 模式沙箱有独立的生命周期，不在本功能范围内。
:::

### 坏镜像

一个**在任何节点上**都无法拉取的镜像（缺少 tag 或 manifest、仓库不存在、或访问被拒绝）在每次替换后都会以相同方式失败。控制器能识别这些情况，并**不会**浪费替换次数：Pod 保持原样，同时记录一条 `ImagePullPermanentFailure` 警告事件说明镜像错误。请修复镜像或凭据；修复后，Pod 会在下一次拉取尝试时自行恢复，无需重启。

::: warning 与上层 Pending 看门狗的相互作用
如果上层管理者会在自己的超时（例如约 7.5 分钟）后清理卡在 `Pending` 的沙箱，请注意在默认设置（`1m` 阈值、3 次尝试）下，控制器会在大约 4 分钟内完成恢复尝试 —— 早于此类看门狗触发。如果把 `pod-recovery-stuck-threshold` 提高到超过看门狗超时的大约三分之一，沙箱被清理时替换尝试可能仍在进行中。
:::

::: warning 大镜像与冷拉取
阈值过低会替换那些镜像拉取仅仅超过阈值时长的 Pod。当 kubelet 正在积极拉取时，容器显示为 `ContainerCreating`（不计入统计）；只有显式的拉取失败才会计时。如果你的集群从较慢的仓库拉取非常大的镜像，请调高 `pod-recovery-stuck-threshold`。
:::

## 配置

功能配置位于控制器命名空间中的 `feature-flags` ConfigMap（普通的 `data` 键值对）。修改会热加载，无需重启控制器：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
  namespace: opensandbox-system  # the controller's namespace
data:
  # How long a pod must stay stuck in a tracked failure condition during
  # initial startup before it is replaced. Defaults to 1m.
  pod-recovery-stuck-threshold: "1m"
  # Maximum number of stuck-pod replacements per BatchSandbox template.
  # Defaults to 3.
  pod-recovery-max-attempts: "3"
  # Optional: comma-separated kubelet admission rejection reasons that
  # replacement can recover. When set, it replaces the built-in set
  # (NodeNotSchedulable, KubeletNotReady, UnexpectedAdmissionError, Evicted);
  # the OutOf* family (OutOfcpu, OutOfmemory, ...) always applies. Useful for
  # vendor-specific kubelet rejection reasons.
  # pod-recovery-admission-reasons: "KubeletNotReady,Evicted"
```

缺失或非法的条目会回退到内置默认值（`1m` / `3`），删除 ConfigMap 也会恢复默认值。

Helm（控制器 chart）从 values 渲染该 ConfigMap：

```yaml
controller:
  podRecovery:
    stuckThreshold: "1m"
    maxAttempts: 3
    # Optional; leave empty for the built-in admission reasons.
    admissionReasons: ""
```

## 监控

记录在 BatchSandbox 上的事件：

| 事件 | 类型 | 含义 |
| --- | --- | --- |
| `ReplacedStuckPod` | Normal | 卡住的 Pod 已被删除，将在另一个节点上重建。 |
| `PodRecoveryLimitReached` | Warning | 替换次数已耗尽；沙箱保持 `Pending`，需要运维人员介入。 |
| `ImagePullPermanentFailure` | Warning | 镜像本身无法拉取（缺少 manifest/仓库或访问被拒绝）；请修复镜像或凭据。 |

控制器指标端点上的 Prometheus 指标：

| 指标 | 标签 | 说明 |
| --- | --- | --- |
| `opensandbox_pod_recovery_replacements_total` | `condition` | 初始启动期间被替换的卡住 Pod 数量。 |
| `opensandbox_pod_recovery_skips_total` | `condition`, `reason` | 检测到但未替换的卡住 Pod；`reason` 为 `permanent_failure`（坏镜像）或 `budget_exhausted`（需要运维介入）。 |

查找受影响的沙箱：

```bash
kubectl get events --field-selector reason=ReplacedStuckPod,reason=PodRecoveryLimitReached,reason=ImagePullPermanentFailure
```
