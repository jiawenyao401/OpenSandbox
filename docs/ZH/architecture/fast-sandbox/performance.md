---
title: Fast Sandbox 性能
description: fast-sandbox 集成的 ACK 测量 — 串行与并发创建、请求延迟、实时快照以及同节点暂停/恢复。
---

# 性能

ACK（Alibaba Cloud Container Service for Kubernetes）是阿里云的托管
Kubernetes 服务。本页报告 2026-10-08 在一个双节点 ACK 集群上收集的测量数据：串行与并发创建、请求延迟、实时快照，
以及通过 OpenSandbox SDK 的同节点暂停/恢复。运行时日志在 SDK 结果之外
提供了阶段计时。这些是工程测量值，
不是生产延迟保证。

热创建路径将工作负载接纳进预热的容量（参见
[调度](/architecture/fast-sandbox/scheduling)），并还原快照而非启动内核（参见 [Firecracker](/architecture/fast-sandbox/firecracker)）。
暂停/恢复使用提交并收敛的变更路径（参见
[高可用](/architecture/fast-sandbox/ha)）。容量、运行时工作
和就绪边界仍然决定观测到的延迟和成功率。

## 如何解读这些数字

- **SDK 就绪**包含 Python SDK、生命周期服务器、FastPath、
  入口网关以及 guest `/ping` 返回 200。运行时创建在此之前结束。
- **热创建**的模板制品已缓存且 Fastlet 容量已备好。
  首次使用的制品分发是另一项开销。
- **样本数与失败**与延迟一同报告。成功尝试的
  百分位数不能描述失败或超时的请求，也不能证明吞吐量。
- **批次差异**很重要：客户端资源与放置，以及制品存储的
  持久化，在最初的串行基线和后续运行之间发生了变化。
  两个批次分开报告，不是受控的 A/B 对比。

[下载原始 SDK 报告](/benchmarks/fast-sandbox-ack-2026-10-08.json)以获取
单个样本、失败、运行时阶段、镜像摘要和设置。

## 环境

| 维度 | ACK 环境（2026-10-08） |
|---|---|
| 集群 | 两个 ACK 节点，Kubernetes `1.36.2-aliyun.1`，containerd 2.1.9 |
| CPU | 每节点：AMD EPYC 9T95，暴露 192 CPU，可分配 191.45 |
| 内存 | 每节点约 742 GiB 可分配 |
| 操作系统 / 内核 | ContainerOS/Lifsea，`5.10.134-19.6.1.lifsea8.x86_64` |
| StateRoot | 每节点 50 GiB XFS `reflink=1` loop 文件系统，由 ext4 ESSD 数据盘支撑 |
| 池 | 五个 Fastlet，每个 4 个槽位：共 20 个槽位 |
| 快照工作负载 | Alpine 3.19 + execd，1 vCPU，2 GiB 内存，2 GiB rootfs |
| 制品存储 | 后期运行期间为其中一个节点持久 hostPath 上的 RustFS |
| Firecracker | 1.16.1 |

SDK 部署使用 server、controller 和 runtime agent 的
`release-1.1.1-rc.1`。其派生的 Fastlet 镜像添加了 GNU coreutils `cp` 9.11；
可用的 reflink 支持需要文件系统和复制工具同时满足。XFS
启动拒绝的变更在测量期间未部署。模板
制品摘要为
`d78575470094c6fe1a2da2a71b7578971dff9e77cd5381dee236dd53e01b54cb`，已缓存于
两个节点。SDK 版本：`1.1.1rc2.dev106+gc7dc78a4.d20261008`。

## 创建

### SDK 创建

每次 SDK 创建运行包含两次不计时的预热和 100 次计时尝试，模板
制品已缓存。计时从 `Sandbox.create_from_template()`
之前开始，到通过网关的 guest `/ping` 返回 200 为止。每个沙箱在
就绪后被杀死。并发批次之间有五秒的回收暂停；删除
和批次暂停不计入单次延迟。后续的串行和
并发运行保持相同的池配置。

集群内客户端禁用 metrics，使用十秒 HTTP 请求超时，
并每 10 ms 轮询一次就绪（SDK 默认：200 ms）。百分位数使用
nearest-rank 方法。最初的串行基线与后续对比运行之间的
客户端差异在下方说明。

| 测量项 | 成功 | p50 | p90 | p99 | 均值 |
|---|---:|---:|---:|---:|---:|
| SDK 创建，初始串行基线 | 100/100 | **64.0 ms** | 93.8 ms | 108.3 ms | 70.6 ms |
| SDK 创建，与并发运行同时进行的串行重复 | 100/100 | 77.9 ms | 105.3 ms | 112.5 ms | 78.2 ms |
| SDK 创建，并发 10 | 100/100 | 102.7 ms | 120.1 ms | 125.4 ms | 102.4 ms |
| SDK 创建，并发 20 | 51/100 | 136.7 ms | 163.0 ms | 197.0 ms | 134.2 ms |

最初的 100 次创建验证保留为热串行基线。
其精确 p50 为 63.995 ms，均值 70.587 ms，范围 53.811–113.326 ms；
全部 100 次创建均达到 guest 就绪。它使用了相同的集群、模板、
池形态、SDK 版本、超时和 10 ms 就绪标准。其客户端
Pod 有两个 CPU 的限额，放置未显式固定；后续
对比客户端为四个 CPU 限额，运行在选定节点上。RustFS
持久化在两次运行之间也发生了变化，不过这些热创建使用的是
缓存的模板制品。

77.9 ms 的重复结果仍然列出，因为它是在并发运行及其客户端配置下同时收集的。
两个串行批次都在原始报告中，且未合并。
它们的差异不是隔离主机、客户端 CPU 或存储变更的
受控实验。

相同创建窗口的运行时日志给出如下分解。
这些包含每次运行的两次不计时预热，因此样本数与 SDK 表不同。
它们止于运行时创建，早于 SDK 就绪：

| 负载 | 运行时创建 p50 | Rootfs 准备 p50 | VM 状态加载/恢复 p50 | 运行时日志样本数 |
|---|---:|---:|---:|---:|
| 串行重复 | 50.0 ms | 2.31 ms | 0.233 ms | 102 |
| 并发 10 | 34.1 ms | 2.76 ms | 0.234 ms | 102 |
| 并发 20 | 33.5 ms | 2.63 ms | 0.225 ms | 53 |

初始串行验证的独立运行时窗口在 103 次创建中
（100 次计时、两次预热和一次后续生命周期创建）的运行时创建中位数为 39.43 ms，
rootfs 准备中位数为 2.286 ms。

::: warning 饱和结果
并发 20 的延迟分布仅包含**成功的尝试**。
有 48 次 `SandboxRateLimitException` 失败（HTTP 429 / Fastlet
`CapacityRejected`）和一次就绪超时并伴随重复的网关 HTTP 503。
还有四个测试 CR 因分配注解/状态投影冲突而卡在删除中。
它们的过期放置投影已被清除，以便控制器在下次实验前执行清理。这次运行
并未建立可靠的 20 路准入或可持续的每秒请求数率。
准入/投影失败的实现原因需要单独修复；仅更换主机磁盘并不能解决它。
:::

## 请求延迟

SDK 样本使用一个预热沙箱，每路径 100 次顺序调用：

| 路径 | 成功 | p50 | p90 | p99 | 均值 |
|---|---:|---:|---:|---:|---:|
| 经网关的热 SDK `/ping` | 100/100 | 0.92 ms | 1.15 ms | 5.91 ms | 1.07 ms |
| SDK `commands.run('echo ack-perf')`，直至完成和流 EOF | 100/100 | 202.8 ms | 204.3 ms | 207.1 ms | 203.2 ms |

[execd 命令处理器](https://github.com/opensandbox-group/OpenSandbox/blob/c7dc78a4090e5de2b9119e9bd93952cae24f87bd/components/execd/pkg/web/controller/command.go)
有 200 ms 的默认流关闭宽限期；观测到的约
203 ms 与该行为一致。它并不是对 shell
进程启动本身的测量。

## 实时快照

测量了一次实时快照：

| 阶段 | 测量值 |
|---|---:|
| 快照请求被接受 | 14.4 ms |
| 请求 → 快照可用（SDK 中的 `Ready`） | 60.63 s |
| 转储期间 VM 冻结 | 11.14 s |
| 运行时制品发布 | 46.22 s |
| 制品集 | 2 GiB rootfs + 2 GiB 内存 |

之后 SDK 源沙箱保持健康，其文件标记和启动 ID
得以保留。冻结期间未对请求全程采样，因此这
不构成零中断的声明。最初的探测使用了错误的终态
检查，已从计时中排除。

## 从快照恢复

本次 ACK 运行未测量从已发布的实时快照创建新沙箱。
缓存的 checkpoint 还原作为[同节点暂停/恢复](#pause-resume)的一部分进行了测量；模板还原涵盖于
[创建](#create)。这些操作有不同的完成边界。

## 暂停 / 恢复

仅针对此测试，空闲池被限制为一个既有节点，保留
五个 Fastlet，每个 Fastlet 四个槽位。每个样本的源和目标
Kubernetes 节点都通过 CR 的放置和 Fastlet Pod 的
`spec.nodeName` 进行了验证。全部三次恢复都迁移到该**同一节点**上不同的 Fastlet，并使用了该节点的 checkpoint 缓存。之后恢复了原始池选择器。

| 测量项 | 中位数 | 观测范围（3 个样本） |
|---|---:|---:|
| SDK 暂停请求被接受 | 22.4 ms | 22.0–25.5 ms |
| 暂停请求 → 持久的 `Paused`，包括上传和释放 | 61.14 s | 61.13–61.23 s |
| checkpoint 转储期间 VM 冻结 | 11.15 s | 11.13–11.18 s |
| 发布约 4 GiB（2 GiB rootfs + 2 GiB 内存） | 45.61 s | 45.61–45.63 s |
| 恢复运行时创建，在缓存制品可用后 | 35.49 ms | 33.64–35.50 ms |
| 恢复 rootfs 准备 | 2.92 ms | 2.64–3.26 ms |
| SDK 同节点恢复 → guest `/ping` 返回 200 | **62.66 s** | **58.29–65.16 s** |

全部三个样本都保留了文件系统标记、tmpfs 标记、启动 ID、
后台 PID 和进程启动时间。沙箱身份也保持不变。
相应 Fastlet 日志中没有出现 checkpoint 制品分发尝试。Rootfs checkpoint 克隆耗时 0.88–0.94 ms，零完整复制时间。三个样本建立的是这个观测范围，不是可靠的 p99。

::: warning 同节点恢复仍然很慢
缓存的运行时还原是几十毫秒的量级，但在此部署中可用的 SDK 恢复
约为 1 分钟。运行时日志在还原后立即发布路由，
大部分耗时留在端点/guest 可用性和客户端等待上。这些测量
没有隔离出是哪种网络、网关、端点解析或重试机制造成了这个差距。
该差距无法用复制 2 GiB rootfs 或将 checkpoint 取到另一个节点来解释。不要用运行时数字替代 SDK 恢复。

探测使用了四分钟的恢复预算和十秒 HTTP 请求超时；
SDK 默认恢复预算为 30 秒。报告的结果
包含真实的就绪等待，且不跳过健康检查。
:::

## 制品分发（DART P2P）

本次 ACK 运行未验证。模板制品已缓存在两个
节点上，同节点恢复使用了 checkpoint 缓存。冷分发延迟、
源站/对等块计数以及 P2P 效率需要单独测量。

## 启动特性

本次 ACK 运行未验证冷内核启动。模板创建
还原 `vmstate.snap` 而不是启动内核；其测得的 VM 状态加载/恢复
阶段见[创建](#create)。

## 结果解读

XFS reflink 支持和 GNU `cp` 使热的 rootfs 准备保持在
毫秒量级：初始串行运行时窗口测得 2.286 ms 的
中位数。这测量的是存储/复制阶段，不是 SDK 的总就绪时间。

并发改变准入竞争；持久暂停包含转储、
哈希和上传 checkpoint；SDK 就绪包含端点解析、
HTTP 超时和重试。[创建](#create)下的饱和失败是
控制平面的证据，不能仅用主机解释。同节点恢复的差距
同样仍未解决。热的 `/ping`、完成的命令流、运行时
还原和 SDK 恢复各有不同的完成边界。

## 复现

### SDK 测量

使用与记录版本匹配的源码构建 SDK。为目标集群设置 `DOMAIN`、`API_KEY`
和 `TEMPLATE_ID`，且不要把凭据放进报告。从集群内客户端运行以保持这一网络边界：

```bash
REQUEST_TIMEOUT=10 DISABLE_METRICS=1 PING_INTERVAL_MS=10 READY_TIMEOUT_WARM=180 \
N=100 CONCURRENCY=1 BATCH_PAUSE=0 REPORT_PATH=serial.json \
  python scripts/fast-sandbox-env/create-bench.py

REQUEST_TIMEOUT=10 DISABLE_METRICS=1 PING_INTERVAL_MS=10 READY_TIMEOUT_WARM=180 \
N=100 CONCURRENCY=10 BATCH_PAUSE=5 REPORT_PATH=concurrent10.json \
  python scripts/fast-sandbox-env/create-bench.py

# Saturation diagnostic: record failures as well as successful latencies.
REQUEST_TIMEOUT=10 DISABLE_METRICS=1 PING_INTERVAL_MS=10 READY_TIMEOUT_WARM=180 \
N=100 CONCURRENCY=20 BATCH_PAUSE=5 REPORT_PATH=concurrent20.json \
  python scripts/fast-sandbox-env/create-bench.py
```

`REPORT_PATH` 保存原始尝试、设置、成功/失败计数和
nearest-rank 百分位数。存在失败创建的运行会返回非零退出状态。将部署的镜像摘要、主机/存储细节、模板
摘要、池形态以及带时间戳的 Fastlet/controller 日志与之一同保存。
SDK 运行使用了等效的探测脚本，同时也收集了生命周期
和请求样本；报告保留了那些原始样本。

对于其他测量，保持相同的 SDK 连接设置：

1. 在一个预热沙箱上，计时 100 次顺序的 `sandbox.is_healthy()` 调用，
   断言成功；然后计时 100 次 `sandbox.commands.run('echo ack-perf')`
   调用直至流 EOF，断言预期输出。
2. 在限制到一个节点的空闲池上，为三个生命周期样本中的每一个创建一个全新沙箱。
   在暂停之前，等待 CR
   `status.runtime.state == Ready`；SDK 的早期就绪结果可能先于
   该投影出现。向文件系统和 tmpfs 写入标记，并记录
   启动 ID 和一个后台进程的 PID/启动时间。
3. 在 `sandbox.pause()` 之前启动暂停计时器。每 200 ms 轮询
   `manager.get_sandbox_info(id).status.state == 'Paused'`；仅被接受的
   暂停响应不是持久的 checkpoint。
4. 计时 `Sandbox.resume(id, resume_timeout=timedelta(minutes=4),
   health_check_polling_interval=timedelta(milliseconds=10),
   connection_config=config)` 直至其正常的就绪检查。在杀死
   沙箱之前验证在原节点上的放置和所有 guest 标记。恢复池的原始选择器，并等待五个热、
   就绪的 Fastlet。
5. 对于实时快照，将 `sandbox.create_snapshot()` 的接受时间与
   `manager.get_snapshot(id).status.state == 'Ready'`（200 ms 轮询）分开计时。
   公共快照 API 使用 `Ready`；模板构建使用 `Succeeded`。
   验证源沙箱并删除测试快照和沙箱。

捕获运行时转储、发布、还原和路由发布的日志时间戳，
以区分 checkpoint I/O 与之后的 SDK 就绪。仅在测试创建的
沙箱/快照资源消失之后，再清理该测试创建的制品。

在比较负载形态时，保持硬件、内核、Firecracker 版本、模板摘要、池形态、客户端
放置和制品存储放置不变。
