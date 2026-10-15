---
title: 定时客户端池扩缩
description: 使用应用级控制器和 Resize，在业务时段内调整 Go SandboxPool 的空闲目标。
---

# 定时客户端池扩缩

在业务时段保持更多沙箱就绪，在夜间或周末减少空闲
容量。这个 Go 示例评估一个小型的应用自有时间表，并
调用现有的 `SandboxPool.Resize` API。SDK 的协调器负责创建
和移除空闲沙箱。

时间表使用 **周一至周五，09:00（含）到 21:00（不含）**，采用
IANA 时区，例如 `Asia/Shanghai` 或 `America/New_York`。其默认目标为
业务时段内五个空闲沙箱，时段外为零。编辑示例中的
`desiredMaxIdle` 以实现你自己的需求或预算策略。

源代码：[`examples/client-pool-schedule/`](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/client-pool-schedule)。

## 运行

安装 Go 1.20 或更新版本，检出本仓库，并启动一个
[OpenSandbox 服务器](/getting-started/)。该示例的 Go 模块通过本地 `replace` 指令
使用同一检出中的 SDK。

```bash
cd examples/client-pool-schedule
export OPEN_SANDBOX_DOMAIN=localhost:8080
export OPEN_SANDBOX_API_KEY=your-api-key
go run . -timezone Asia/Shanghai -peak-idle 5 -offpeak-idle 0
```

对于 HTTPS 服务器，使用 `OPEN_SANDBOX_PROTOCOL=https`。示例通过服务器代理路由沙箱
健康检查。它使用 `ubuntu:22.04` 和一个保持沙箱
存活的 entrypoint；使用 `-image` 可选择其他兼容镜像。
运行 `go run . -h` 查看所有标志。

::: warning 沙箱资源
当定时目标为正数时，此命令会创建真实的沙箱。试用期间请
使用较小的目标值。Ctrl+C 会停止协调，然后
清空示例剩余的空闲沙箱，并等待其尽力而为的终止请求
完成后才退出。失败的终止可能导致沙箱一直运行到其
TTL 过期。已被你的应用获取的沙箱仍由调用方负责。
:::

## 扩缩的工作方式

- 池启动时使用当前时间对应的目标，避免夜间启动时的初始
  业务时段预热。
- 每 30 秒，控制器将期望目标与
  `Snapshot().MaxIdle` 进行比较。仅当两者不同时才调用 `Resize`，因此
  临时性的空闲缺口不会触发重复的扩缩调用。
- 目标更新是异步的。协调间隔、沙箱启动和
  就绪时间以及预热并发度决定了容量收敛的速度。
  如果必须在流量到达之前容量就绪，请将业务时段开始时间提前。
- 缩容会移除多余的**空闲**沙箱。已被获取的沙箱不属于空闲
  缓冲区。目标为零只会停止补充；它不会禁用获取时的
  直接创建，也不是沙箱总配额。
- 每次检查时都会重新评估时区，包括夏令时变更。
  该示例内嵌了 Go 的时区数据库，因此即使在没有
  系统 zoneinfo 文件的主机上也能工作。
- 快照或扩缩错误会停止示例并运行清理。它不会
  调用 `Start` 或清除命名空间销毁状态来从错误中恢复。

## 多个应用进程

这个可运行示例使用**内存存储**，因此它的池属于单个
进程。对于分布式池，请为每个参与者配置相同的 Redis
存储和 `pool_name`，以及唯一的 `owner_id`；参见
[客户端池指南](/guides/client-pool)。

每个共享命名空间只运行一个应用级调度控制器，
或在你的应用中选出一个。`Resize` 更新的是共享目标，因此
你不需要在每个 worker 上都运行 cron 任务。池的预热领导者选举不会
为你的外部调度策略选举领导者。避免手动的和定时的写入者相互竞争：本示例会在下一次检查时重新应用定时目标。
请将 worker 的启动和重启与控制器协调，因为启动
池时可能会将其配置的初始目标写入存储。

示例的关机清空逻辑针对其单进程池设计。不要把
这个清空逻辑复制到共享池某个 worker 的关机流程中：它会在其他 worker 可能仍在补充共享空闲缓冲区时
将其清空。停用整个分布式池时，请使用文档中的
命名空间退役流程。

## 无服务器测试

```bash
go test ./...
go vet ./...
```

测试涵盖工作日与周末边界、时区转换、夏令时
切换，以及通过真实 SDK 的内存状态存储进行的目标更新。
一个本地 HTTP 测试服务器用于验证关机时会等待未完成的终止请求。
