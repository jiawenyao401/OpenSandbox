---
title: SDKs
description: 选择 OpenSandbox SDK，比较生命周期、Client Pool、追踪和诊断支持。
---

# SDKs

OpenSandbox 提供五个 sandbox SDK，覆盖生命周期管理、命令执行、文件操作和资源指标。本站点跟随仓库的默认分支；合并到这里的特性可能需要比你已安装版本更新的软件包。升级时请查看[发布索引](https://github.com/opensandbox-group/OpenSandbox/releases)。

## 安装

| 语言 | 软件包 | 安装 |
| --- | --- | --- |
| [Python](/sdks/python) | `opensandbox` | `pip install opensandbox` |
| [JavaScript/TypeScript](/sdks/javascript) | `@alibaba-group/opensandbox` | `npm install @alibaba-group/opensandbox` |
| [Kotlin/Java](/sdks/kotlin) | `com.alibaba.opensandbox:sandbox` | Gradle/Maven |
| [Go](/sdks/go) | `github.com/alibaba/OpenSandbox/sdks/sandbox/go` | `go get github.com/alibaba/OpenSandbox/sdks/sandbox/go` |
| [C#/.NET](/sdks/csharp) | `Alibaba.OpenSandbox` | `dotnet add package Alibaba.OpenSandbox` |

## 能力覆盖

"Yes" 表示存在公开的 SDK 入口点。可用性还取决于 server 运行时以及 sandbox 中安装的 execd 版本。

| 能力 | Python async/sync | JavaScript/TypeScript | Kotlin/Java | Go | C#/.NET |
| --- | --- | --- | --- | --- | --- |
| 创建、连接、续期、暂停/恢复、终止 | Yes | Yes | Yes | Yes | Yes |
| 快照与模板管理 | Yes | Yes | Yes | Yes | Yes |
| 资源请求、卷、生命周期钩子 | Yes | Yes | Yes | Yes | Yes |
| 命令、文件、资源指标 | Yes | Yes | Yes | Yes | Yes |
| Egress 策略与 Credential Vault | Yes | Yes | Yes | Yes | Yes |
| 隔离会话 | Yes | Yes | Yes | Yes | Yes |
| [Client Pool](/guides/client-pool)，包括 Redis | Yes | Yes | Yes | Yes | No |
| [Pool warmup 追踪](/sdks/observability#pool-warmup-tracing) | Yes | Yes（属性较少） | Yes | No | No |
| [远程诊断日志/事件](/api/#diagnostics) | Yes | No | Yes | No | No |
| [创建延迟遥测](/sdks/observability#creation-metrics) | Yes | Yes | Yes | Yes | Yes |

### 影响应用代码的差异

- **Pool：** Go 暴露不同的 warmup 控制项；默认 pool 创建器接受的字段在各语言间也不同。参见 [pool 配置矩阵](/guides/client-pool#configuration)。
- **Go 命令控制：** 中断、命令状态和累计命令日志由 `ExecdClient` 暴露，而不是高层 `Sandbox` 包装器。`Sandbox.CreateSession` 和 `ExecdClient.CreateSession` 都不接受初始工作目录，请改为在会话中运行命令时设置 `Cwd`。
- **Go connect/resume 就绪：** 传入 `ReadyOptions` 以请求就绪检查。不传的话，连接只解析端点而不检查 sandbox 健康状况。
- **指标流：** Go 暴露 `ExecdClient.WatchMetrics`；其他 SDK 的稳定指标服务提供时点读取。[CLI](/cli/) 也通过 `osb sandbox metrics --watch` 提供指标流。
- **超时单位：** Python 使用 `timedelta`，JVM 使用 `Duration`；JavaScript 和 C# 的命令超时使用秒。Go 的 `RunCommandRequest.Timeout` 使用**毫秒**。

基于模板的 sandbox 需要显式 TTL，并从已发布的模板继承工作负载配置。其 egress 策略通过生命周期 API 管理；它们没有 sandbox 侧的 Credential Vault。

## 特性指南

- [Client Pool](/guides/client-pool)：维护就绪缓冲、选择获取策略、通过 Redis 共享状态，以及退役 pool 命名空间。
- [可观测性](/sdks/observability)：配置 pool warmup 追踪和创建指标，理解默认设置，并定位缓慢的启动阶段。

## 诊断

Python 和 Kotlin/Java 在 `Sandbox` 和 `SandboxManager` 上都暴露远程日志/事件。当 execd 尚未就绪时请使用 manager。Python 方法为 `get_diagnostic_logs` / `get_diagnostic_events`；JVM 方法为 `getDiagnosticLogs` / `getDiagnosticEvents`。向 manager 方法传入 sandbox ID，并传入显式作用域，如日志的 `container` 或事件的 `runtime`。

JavaScript、Go 和 C# 可以使用 [CLI](/cli/#collect-diagnostics) 或 [HTTP API](/api/#diagnostics)。C# 的 `SdkDiagnosticsOptions` 控制本地 SDK 日志。支持的作用域以及内联/URL 投递方式见 API 参考。

## 生命周期与清理

高层镜像/快照创建默认为 10 分钟 TTL。请为你的工作负载显式配置 TTL。要禁用过期，可使用 Python 的 `timeout=None`、JavaScript 的 `timeoutSeconds: null`、Kotlin 的 `timeout(null)`，或 Go/C# 的 `ManualCleanup` 选项。模板创建必须提供 TTL。

`close()`、`DisposeAsync()` 和上下文管理器退出只释放本地客户端资源；它们不会终止远程 sandbox。请在 `finally`/`defer` 块中终止它，或使用 Python 的 `destroy()` 辅助方法。pool 获取是一次性消耗的，不会归还到空闲缓冲。

## CLI

使用 [CLI](/cli/) 从终端管理 sandbox、运行命令和处理文件。

```bash
uv tool install opensandbox-cli
```

## MCP server

[MCP server](/sdks/mcp) 将 sandbox 操作暴露给支持 MCP 的客户端：

```bash
pip install opensandbox-mcp
```
