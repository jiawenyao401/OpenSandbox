---
title: Kubernetes 恢复基准测试
description: 复现标准 Kubernetes BatchSandbox 后端的端到端 rootfs 恢复延迟测量。
---

# Kubernetes 恢复基准测试

可选启用的[运行器](https://github.com/opensandbox-group/OpenSandbox/blob/main/tests/python/benchmarks/kubernetes_resume.py)测量标准 Kubernetes **独立、rootfs** 的暂停/恢复。它每次试验创建一个沙箱，在卷挂载之外写入唯一标记，暂停它，等待其 Pod 消失，恢复它，并通过前台命令检查标记。它会在每次试验后删除沙箱。QEMU VMState、池化创建和 Fast Sandbox 不在此基线范围内。

## 前提条件

- 一个为 Kubernetes BatchSandbox provider 配置的生命周期服务器，控制器和 OCI 快照注册表按[暂停与恢复](/guides/pause-resume)中的方式配置。
- `kubectl` 连接到同一集群，并具有读取沙箱命名空间中 BatchSandbox、SandboxSnapshots、Pod、节点和集群版本的权限。传递显式 context 以避免意外对不同集群进行测量。
- 一个包含 `/bin/sh`、`cat` 和 `sleep` 的不可变镜像摘要。服务器必须注入 execd。不要把标记目录放在卷挂载之下。
- 一个能通过网络访问生命周期服务器和沙箱端点的运行器；`--server-proxy` 测量服务器代理路由而非直接访问。

这是一个显式的基准测试，不属于默认 E2E 套件。请使用专用测试集群。运行器不会驱逐任何节点缓存。

## 运行

创建一个环境文件，包含复现实验所需的版本和条件。例如 `environment.json`：

```json
{
  "server_revision": "<git SHA or image digest>",
  "controller_image": "<image digest>",
  "execd_image": "<image digest>",
  "image_committer_image": "<image digest>",
  "runtime_class": "runc (default)",
  "node_resources": "<CPU, RAM, disk type>",
  "registry": "<implementation, placement, network conditions>",
  "snapshot_configuration": "<non-secret controller flags>",
  "cache_evidence": "Uncontrolled; no cache intervention",
  "background_load": "<other workloads and concurrency>"
}
```

不要包含凭证。输出还会记录运行器版本、Python 和 SDK 版本、Kubernetes 版本、节点运行时版本、context、镜像、资源限制、代理模式、超时和轮询间隔。

```bash
cd tests/python
uv sync --frozen
export OPEN_SANDBOX_DOMAIN=localhost:8080
export OPEN_SANDBOX_API_KEY='<your API key>'
uv run python benchmarks/kubernetes_resume.py \
  --image '<image>@sha256:<digest>' \
  --namespace default --context '<test-context>' \
  --environment environment.json --scenario uncontrolled \
  --warmups 3 --samples 30 --output resume-uncontrolled.jsonl
```

TLS 服务器使用 `--protocol https`，直接端点不可访问时使用 `--server-proxy`。运行 `--help` 查看资源、轮询和超时选项。脚本为此实验禁用了 SDK 重试和遥测。轮询保持显式，其延迟计入测量的观察时间。

## 测量边界

所有时长使用运行器的单调时钟，并在 `SandboxManager.resume_sandbox` 之前立即开始。恢复部分有一个共享的超时：

| 字段 | 区间结束 |
| --- | --- |
| `api_ack_ms` | 恢复 API 返回；接受不等于就绪 |
| `running_ms` | 第一个报告 `Running` 的生命周期 GET |
| `ready_ms` | 一次全新的 `Sandbox.connect` 解析端点并通过 execd 健康检查 |
| `command_ms` | 一个前台命令成功返回并打印确切的持久化标记 |

这些是累计时间，不是相加的阶段。`command_ms` 包含一次 execd 往返，是主要的用户可见结果。它不测量应用特定的 Agent 初始化；在单独的实验中添加该工作负载。生命周期观察和 SDK 就绪是顺序进行的，因此轮询间隔和 HTTP 开销会影响结果。不会尝试从这些客户端计时推断调度器或镜像拉取时长。

创建、标记写入、暂停、Pod 删除、可选的缓存准备和证据收集以及清理都在恢复区间之外。等待源 Pod 消失可以避免把一个正在终止的源实例误认为恢复出的实例。运行器在暂停前会验证所创建的 ID 作为独立 rootfs BatchSandbox 存在于所选命名空间中。

## 缓存条件

`--scenario` 是实验标签，不是缓存状态的自动证明。可选择 `uncontrolled`、`snapshot-cache-hit` 或 `snapshot-cache-miss`。后两者需要 `--prepare-script` 指向你提供的 Python 准备/验证脚本。它在暂停和 Pod 删除之后、计时器开始之前以如下方式运行：

```text
python prepare.py <sandbox-id> <namespace> <context> <scenario>
```

脚本只有在建立并检查了所声明的条件之后才必须以零退出。它可以检查沙箱拥有的 SandboxSnapshot CR 以找到已提交的镜像摘要。其 stdout 会作为该试验的 `preparation_evidence` 保留；保持简洁且不含密钥。准备有有限的超时。失败会阻止恢复，并作为失败试验保留。

对于缓存命中运行，将**已提交的快照镜像**（而不仅仅是基础镜像）预拉取到每个符合条件的节点上并验证其存在。对于缓存未命中运行，使用全新的隔离 worker 或受控缓存设置，并在恢复前验证快照不存在。节点固定、镜像拉取策略、注册表缓存和层共享都会影响结果；将它们记录在环境文件中。每次试验都会创建新快照，因此仅靠预热试验无法确立快照缓存命中。比较百分位数时不要混合场景、资源配置或代理模式。运行器不验证准备脚本的声明；应检查其证据以及保留的恢复 Pod 节点/镜像信息。如果需要镜像拉取或调度归因，请单独收集 kubelet 事件。

## 结果与解读

新的输出路径以独占方式打开，以避免覆盖先前的结果。JSONL 包含一条环境记录、每条完成试验（包括排除的预热）的刷写记录，以及一条摘要。试验包括其沙箱 ID、失败阶段、完成的计时、快照工件、恢复的 Pod 节点/镜像信息和清理错误。失败的清理会停止运行并打印 ID 供手动删除；创建响应失败也会停止运行，并保留 `benchmark_run` 元数据值以定位创建结果不明确的沙箱；它绝不会默默继续累积泄漏的沙箱。

P50 和 P95 使用**成功的测量试验**上的最近秩百分位数。摘要单独报告测量失败率和清理失败。较晚阶段的失败不会将部分计时纳入百分位数总体。预热从不进入测量统计。全部失败的运行其百分位数为 null。非零退出状态表示存在试验或清理失败（包括预热）。中断会把已刷写的记录留在磁盘上；只有带 `completed_requested_trials: true` 的最终摘要才表示一次完成的运行。

三十个样本是一个起点，不是稳定的尾延迟结论。增加样本数量、重复独立运行、保留原始样本，并在百分位数旁报告成功、超时、失败和环境。在相同条件下比较基线和候选版本，CI 中不要使用机器速度阈值。本指南提供了一个测试框架；它不包含任何测得的性能声明。
