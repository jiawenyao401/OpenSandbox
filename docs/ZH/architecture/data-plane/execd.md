---
title: Execd
description: OpenSandbox 沙箱内执行 API 的参考守护进程——命令、代码执行、终端、文件与指标，内置分层隔离。
---

# Execd

Execd 是 OpenSandbox 沙箱内执行 API 的参考守护进程。沙箱 SDK 的命令和文件操作使用该 API；沙箱创建和生命周期管理则使用独立的生命周期服务端。客户端也可以通过已解析的沙箱端点访问工作负载特定的服务，使用这些服务自身的协议和客户端。

例如，[AIO Sandbox 示例](/examples/aio-sandbox)解析端口 `8080` 并使用 `AioSandboxClient` 访问该镜像的 shell、文件和浏览器 API。这些请求并不属于 execd 的命令或文件操作，即使流量经由 execd 的反向代理承载。已发布的契约、当前 SDK 的端点假设以及依赖后端的功能，请参见[实现兼容性](/api/implementation-compatibility)。

![execd 总览](../../public/images/execd-overview.svg)

参考平台会在你的 entrypoint 旁边供给 execd——在 Docker 中是暂存的二进制文件，在 Kubernetes 中是 init 容器，在 Fast Sandbox 中则预置于模板镜像内。Execd 提供一个 HTTP API（默认端口 `44772`）；当启用了访问 token 保护时，客户端需转发端点解析所提供的 token。已发布的 [execd API 规范](https://github.com/opensandbox-group/OpenSandbox/blob/main/specs/execd-api.yaml)描述了执行契约；部分参考实现扩展（包括 PTY 路由）尚未纳入其中。

Execd 还内置一个小型反向代理（`/proxy/{port}`），因此一个暴露的宿主机端口即可访问沙箱内的所有端口——参见[单机网络（Docker）](/architecture/network/single-host-network)。

## 能力一览

| 能力 | 你能得到什么 |
|---|---|
| 命令执行 | 运行任意命令并实时流式输出；前台或后台执行，支持状态轮询与日志获取 |
| 代码执行 | 基于 Jupyter 的代码上下文和 kernel——Code Interpreter SDK 的基础 |
| 交互式终端 | 基于 WebSocket 的真实终端（PTY），可分享给只读观察者 |
| 文件与目录 | 上传、下载、列举、搜索、移动、chmod、删除、原地内容替换——沙箱文件系统完全可脚本化 |
| 隔离会话 | 在私有命名空间内运行 shell，适用于不可信或探索性工作 |
| 指标 | 以时间点快照或实时流的形式获取沙箱 CPU 和内存 |

## API 面

Execd 暴露的一切都在同一套 API 之下。当平台配置了访问 token 时，下表中的每个端点都要求在 `X-EXECD-ACCESS-TOKEN` 头中携带它——只有存活与就绪检查永远开放。契约位于 [specs/execd-api.yaml](https://github.com/opensandbox-group/OpenSandbox/blob/main/specs/execd-api.yaml)。

![execd 请求管道](../../public/images/execd-request-pipeline.svg)

| 领域 | 端点 | 说明 |
|---|---|---|
| 健康 | `GET /ping`、`GET /ready` | 存活与就绪检查；无需 token |
| 命令 | `/command` | 前台或后台运行、中断、轮询状态、获取日志 |
| Shell 会话 | `/session` | 跨调用的持久有状态 shell |
| 代码 | `/code`、`/code/contexts` | 上下文与基于 Jupyter 的执行 |
| 终端 | `/pty` | PTY 会话，附 WebSocket 接入点 |
| 文件 | `/files`、`/directories` | 完整的文件系统能力面 |
| 隔离会话 | `/v1/isolated/*` | 拥有独立文件操作的私有命名空间 shell |
| 指标 | `/metrics`、`/metrics/watch` | 快照与实时流 |
| 反向代理 | `/proxy/{port}` | 通过 execd 访问任意沙箱端口 |

## 命令执行

命令可以两种形式提交：

- **Shell 语法** — 管道、重定向、环境变量展开；也就是你在命令行提示符下会输入的内容。
- **直接 argv** — 一个程序加上字面量参数，不经过 shell 解析。当命令的部分内容来自不可信输入时更安全，并且能精确保留空字符串和特殊字符。

执行模式：

- **前台** 在命令运行期间以实时 Server-Sent-Events 流返回输出。
- **后台** 立即返回；你轮询状态并获取增量日志。已完成的后台输出在 24 小时内可继续获取，运行中的命令永远不会被清理。

前台的 `stdout` 和 `stderr` 事件文本保留命令本身的行尾（`\n`、`\r` 或 `\r\n`）。将每个流的事件文本拼接起来即可重建其输出；最后一个事件可能是未以换行结束的片段。

会话在镜像提供 Bash 时使用 Bash，在极简镜像上回退到 POSIX `sh`——发送给回退会话的命令必须兼容 `sh`。支持 Windows 沙箱（环境变量名不区分大小写；批处理文件需要 shell 语法）。

## 代码执行

代码上下文是由 execd 管理的持久化 Jupyter kernel。你在上下文中执行代码，并以流式方式接收结果——标准输出、执行结果和错误——跨多次调用共享状态，与 notebook 完全一致。

官方[代码解释器镜像](https://github.com/opensandbox-group/sandbox-images)内置 Python、Java、Node.js 和 Go 运行时及配套的 Jupyter kernel——参见[代码解释器示例](/examples/code-interpreter)。代码执行也可以通过普通沙箱 SDK 和原始 API 使用。

## 交互式终端

每个终端会话都是通过 WebSocket 提供的真实 PTY——不是包装的命令，因此交互式程序（编辑器、REPL、智能体 CLI）表现正常。

![execd PTY 模型](../../public/images/execd-pty.svg)

共享模型是经过深思的：恰好一个**持有者**拥有键盘，任意数量的**观察者**可以观看。观察者首先回放已经打印的内容，然后跟随实时输出——适用于监督、调试助手，或让第二个人观看智能体工作而不给它输入。

## 文件与目录

一套 API 覆盖整个文件系统树：流式读写文件、列举和搜索目录、查看和修改权限、替换文件内容、移动和删除路径。每个沙箱 SDK 中的文件 API 都是对这些端点的薄封装。

### 文件系统执行身份

在 Linux 上，受信任的后端可以通过在文件 API 路径前加上 `/v1/filesystem/{uid}/{gid}`，
来选择执行普通文件操作的身份。
两个 ID 都必须是 0 到 4294967294 之间的十进制整数。例如，
`GET /v1/filesystem/1001/1001/files/download?path=/workspace/session-b/result.txt`
会以 UID 1001、主 GID 1001 读取该文件。同一前缀支持所有
普通文件和目录操作，包括上传、元数据、搜索、
替换、重命名、删除和权限修改。

每个请求都在一个独立的 worker 进程中以所选凭据运行。
补充组来自所选用户的系统账户；未知的
数字 UID 没有补充组。除未知 UID 之外的账户或组查询失败
都会使请求失败。worker 绝不会以 Execd 的
补充组作为回退继承。Execd 必须拥有建立所
请求凭据的权限；失败时不会以 Execd 身份重试该操作。
相对路径从 `/` 解析，`~` 使用所选账户的
home 目录（未知 UID 时为 `/`）。Worker 不继承 Execd 的环境变量。

Linux 会以该身份检查路径穿越、父目录访问、文件访问和元数据
修改。上传时的 `owner`、`group` 和 `mode` 仍然是目标元数据，
不能为 worker 带来额外特权。携带不同身份的请求可以并发运行，而不会改变 Execd 的凭据。

要在多个会话之间共享一个沙箱，请为每个会话配置独立的用户和工作区，
授予所需的读访问权限，并将目录写权限保留给所有者。
删除和重命名权限取决于父目录，
因此仅有只读文件权限是不够的。沙箱 API
凭据应保存在选择这些身份的受信任后端中。该功能
不提供独立的挂载命名空间，也不是租户隔离边界。

现有的 `/files` 和 `/directories` 路由保留其默认身份。
带身份前缀的路由需要 Linux 和支持的 Execd 版本。较旧的
服务端会返回不支持的路由，而不是静默忽略身份
选项；客户端不得回退到无前缀的路由。

## 隔离会话

隔离会话在每次执行独立的 [bubblewrap](https://github.com/containers/bubblewrap) 命名空间中运行 shell——一个私有的挂载与身份视图，为每个会话全新创建：

- 显式 bind mount 以只读或读写方式引入宿主机路径。
- 写入被限制在允许列表内，并针对完全解析后的路径强制执行，因此符号链接无法把 bind 重定向到允许范围之外。
- 会话创建会报告当前环境支持什么，让调用者可以适配而不是猜测。

完整工作流请参见[隔离会话指南](/guides/isolation-sessions)。

## 分层安全

Execd 对沙箱运行的一切应用纵深防御：

![execd 安全分层](../../public/images/execd-security-layers.svg)

- **沙箱边界** — 资源限制、身份和隔离运行时（runc 容器、gVisor、Kata、Firecracker microVM）在创建时选择（参见[安全容器指南](/guides/secure-container)）。
- **加固底座** — 当运维者启用时，每个用户进程都通过一个原生启动器启动，该启动器移除能力、设置 `no_new_privs`、安装系统调用拒绝列表，并用 Landlock 限制文件系统写入。
- **隔离会话** — 上文所述的每次执行命名空间，适用于需要自己边界的工作。
- **可选 eBPF 审计** — 记录进程 exec、网络连接和特权变更，范围限定在沙箱自己的 cgroup。以独立的 `execd-ebpf` 构建变体发布，不包含在默认镜像中。

每一层都会报告其状态——激活、降级或不受支持——让客户端可以验证实际生效的内容而不是假设。需要记住的一个边界：execd 的生命周期钩子（平台配置的 setup 命令）作为受信任代码在沙箱内运行；它们是便利功能，不是安全控制。

## 监管与退出码

Execd 可以充当沙箱的 init 进程。它回收孤儿子进程、向你的 entrypoint 转发信号，并将其退出码传播给运行时——因此 `Terminated` 和 `Failed` 等沙箱状态反映的是你的进程的真实行为，长时间运行的会话也不会积累僵尸进程。

## 每次分配的全新身份

沙箱可以在池中预热并在多次分配间复用。Execd 让这变得安全：平台在指派时交付一个按分配的绑定——沙箱 ID、环境变量、访问 token、生命周期钩子——execd 在 API 开放之前原子地应用它。在预热沙箱等待其绑定期间（以及启动完成之前），所有业务端点都返回 `503`；只有 `GET /ping` 和 `GET /ready` 会响应。环境变量、token 和遥测归属绝不会在上一个沙箱与你的沙箱之间泄漏。

![execd 分配绑定流程](../../public/images/execd-binding-flow.svg)

## 可观测性

Execd 以快照和实时流两种形式暴露本地 CPU 和内存指标，并在平台配置了端点时导出 OpenTelemetry 指标——归属到你的沙箱 ID 和分配。指标清单和导出配置方式参见[遥测](/guides/telemetry#execd)。
