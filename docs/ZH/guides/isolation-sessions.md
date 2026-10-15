---
title: 隔离会话
description: 沙箱内按执行隔离的 bubblewrap bash 会话，实现快速、受控的代码运行。
---

# 隔离会话

隔离会话在 [bubblewrap](https://github.com/containers/bubblewrap) 命名空间中运行一个**长驻的 `bash`**，这样一个沙箱 Pod 就能承载多个相互隔离的任务运行，而无需启动新容器。每个会话拥有自己的 PID、挂载、tmpfs 和环境命名空间；启动时间为亚毫秒级。

## 目录

- [前提条件](#requirements)
- [概述](#overview)
- [工作原理](#how-it-works)
- [快速开始](#quick-start)
- [会话生命周期](#session-lifecycle)
- [工作区模式](#workspace-modes)
- [绑定挂载与允许列表](#bind-mounts-and-allowlist)
- [配置档与默认值](#profiles-and-defaults)
- [环境、UID 与网络](#environment-uid-and-networking)
- [文件系统代理](#filesystem-proxy)
- [能力与探测](#capabilities-and-probing)
- [服务器配置](#server-configuration)
- [限制](#limitations)
- [另请参阅](#see-also)

---

## 前提条件

本指南所述功能所需的组件版本：

- `execd` >= 1.0.20 提供基础隔离会话支持；`binds`、`List sessions`、`uid_mode: "userns"` 和默认可写允许列表（`/workspace`、`/mnt`、`/media`、`/data`）**建议 >= 1.0.21**
- `opensandbox-server` >= 0.2.1 —— 当 execd 镜像声明 `bootstrap.execd.isolation` 时，服务器会注入 `CAP_SYS_ADMIN`、不受限的 AppArmor、seccomp 和受保护系统路径设置，以及 `bwrap` 所需的 tmpfs 挂载
- Python SDK >= 0.1.14（`isolation.run_once` / `isolation.session` 上下文管理器）；仅使用生成的隔离客户端需 >= 0.1.13
- JavaScript / TypeScript SDK >= 0.1.10（`isolation.runOnce` / `isolation.withSession`）
- Kotlin / Java SDK >= 1.0.16（`isolation.runOnce()` / `isolation.withSession { ... }`）；仅使用生成的隔离客户端需 >= 1.0.15
- C# SDK >= 0.1.4（`RunOnceAsync` / `WithSessionAsync`）
- Go SDK >= 1.0.4（`IsolationRunOnce` / `IsolationWithSession`）

[`/capabilities`](#capabilities-and-probing) 上的 `setpriv_available` / `userns_available` 字段是在 execd v1.0.21 之后添加的，撰写本文时仅存在于来自 `main` 的 execd 构建中；较旧的 execd 会省略这两个字段（客户端必须容忍其缺失）。

主机要求（`bwrap` 二进制、`CAP_SYS_ADMIN`、`overlayfs` 等）列在[服务器配置 → 主机要求](#server-configuration)下。

发布的 execd 镜像包含会话启动期间使用的 fail-closed 原生工作负载门。对于 Linux 源码构建，`make build` 需要 C 编译器加上静态 libc，并生成 `bin/opensandbox-session-gate`；在启动 execd 之前运行 `make build-session-gate`，然后 `sudo make install-session-gate`，将该辅助程序安装到 `/opt/opensandbox/opensandbox-session-gate`。保持该路径及其父目录由 root 所有，且不允许组或全局可写。

---

## 概述

| 概念 | 边界 | 启动 | 典型用途 |
|---|---|---|---|
| **隔离会话** | 一个沙箱内的 bubblewrap 命名空间 | 创建约 100 毫秒；在既有会话中后续 `run` 的开销接近零 | 复用一个长驻会话运行许多短的、相互隔离的任务 |
| Bash 会话（`/session`） | bash 进程，无额外命名空间 | 毫秒级 | 交互式 REPL 式的命令序列 |
| 沙箱 | 容器或 Pod | 数百毫秒到秒级 | 租户、工作区或用户边界 |
| 安全运行时（gVisor / Kata） | 每个沙箱一个用户态内核或虚拟机 | 10–500 毫秒 | 防御容器逃逸的硬件级保护 |

::: info 会话创建与单次运行成本
创建会话约需 100 毫秒，因为 execd 在启动 `bwrap` 后会短暂等待以检测立即退出的子进程。会话存在后，每次 `POST /run` 都复用同一个 bash 进程，因此单次运行开销可忽略。设计工作负载时，应将创建成本分摊到一个会话中的多次 `run` 调用上。
:::

适合场景：**RL rollout**、**批量代码评分**、**多工具 agent 运行** —— 每个 worker 一个沙箱，内部运行多个隔离任务。

不适合场景：跨语言内核（使用 `/code`）、交互式 REPL（使用 `/session`），或对抗内核漏洞利用的硬信任边界（使用 gVisor / Kata；参见[安全容器运行时](/guides/secure-container)）。

---

## 工作原理

`execd` 为每个会话 fork 一个 `bwrap` 子进程；`bwrap` 建立 Linux 命名空间，然后在其中 `exec` 一个长驻的 `bash`。

![隔离会话运行时布局](../public/images/isolation-sessions-layout.svg)

图中未展示的两点：

- `bash` 是长驻的，因此一次 `run` 中的 `export X=1` 对**同一会话**中的下一次 `run` 可见（在其他会话中永不可见）。
- FS 代理从命名空间**外部**读写合并的工作区视图，因此即使 `bash` 忙碌，上传和下载也能工作。

### `run` 请求流程

![隔离会话 run 流程](../public/images/isolation-sessions-run-flow.svg)

非正常路径：

| 事件 | 发生什么 |
|---|---|
| 运行 `timeout_seconds` 到期 | execd 取消运行上下文，向 bwrap 进程组发送 `SIGINT` 并发出 `IsolatedError` SSE 事件；会话本身保持存活。 |
| `bwrap` 进程已退出 | 下一次 `run` 返回带 `session process has exited` 的 `IsolatedError` SSE 事件；只有 `ErrContextNotFound`（会话 ID 未知）会成为 HTTP 层错误。`DELETE` 后重建。 |
| 达到空闲超时 | GC 执行与 `DELETE` 相同的清理。 |
| 客户端在 SSE 中途断开 | Gin 请求上下文被取消，execd 向运行中的命令发送 `SIGINT`；运行**不会**在后台继续。 |

---

## 快速开始

### curl

```bash
# Probe.
curl -s http://localhost:44772/v1/isolated/capabilities

# Create.
SESSION=$(curl -s -X POST http://localhost:44772/v1/isolated/session \
  -H "Content-Type: application/json" \
  -d '{
    "profile": "strict",
    "workspace": {"path": "/workspace", "mode": "overlay"},
    "idle_timeout_seconds": 300
  }' | jq -r .session_id)

# Run (SSE: stdout / error / complete).
curl -N -X POST "http://localhost:44772/v1/isolated/session/$SESSION/run" \
  -H "Content-Type: application/json" \
  -d '{"code": "export X=1; echo $X", "timeout_seconds": 30}'

# Second run reuses shell state.
curl -N -X POST "http://localhost:44772/v1/isolated/session/$SESSION/run" \
  -H "Content-Type: application/json" \
  -d '{"code": "echo $X"}'  # prints 1

# Delete.
curl -X DELETE "http://localhost:44772/v1/isolated/session/$SESSION"
```

### Python SDK

```python
from opensandbox import Sandbox
from opensandbox.models.isolated import (
    CreateIsolatedSessionRequest, IsolatedWorkspaceSpec, IsolatedRunOpts,
)

# Sandbox.create is an async classmethod, so await it before entering
# the async context manager.
async with (await Sandbox.create("python:3.11")) as sandbox:
    # One-shot.
    await sandbox.isolation.run_once(
        "python -c 'print(42)'", workspace="/workspace", profile="strict",
    )

    # Persistent session.
    async with sandbox.isolation.session(
        CreateIsolatedSessionRequest(
            workspace=IsolatedWorkspaceSpec(path="/workspace", mode="overlay"),
            profile="strict",
            idle_timeout_seconds=300,
        )
    ) as session:
        await session.run("export STAGE=train")
        await session.run("python train.py", opts=IsolatedRunOpts(timeout_seconds=600))
        await session.files.write_file("/workspace/data.csv", data_bytes)

    # Reattach after a client restart.
    handle = await sandbox.isolation.attach(known_session_id)
```

### JavaScript / TypeScript SDK

```ts
await sandbox.isolation.runOnce("node -e 'console.log(1)'", "/workspace", {
  profile: "strict",
});

await sandbox.isolation.withSession(
  { profile: "strict", workspace: { path: "/workspace", mode: "overlay" } },
  async (session) => { await session.run("npm test"); },
);
```

### Kotlin SDK

```kotlin
sandbox.isolation().runOnce(code = "python -c 'print(1)'", workspace = "/workspace")

sandbox.isolation().withSession(request) { session ->
    session.run("ls /workspace")
}
```

---

## 会话生命周期

位于 execd 的 `/v1/isolated/` 下。当 execd 配置了访问令牌时使用 `X-EXECD-ACCESS-TOKEN`。

| 方法 | 路径 | 用途 |
|---|---|---|
| `POST` | `/session` | 创建；返回 `{session_id, created_at}`。 |
| `GET` | `/sessions` | 列出活跃会话。 |
| `GET` | `/session/{id}` | 完整状态；回显创建参数，以便无状态客户端重建句柄。 |
| `POST` | `/session/{id}/run` | 前台（默认）：SSE 流 `stdout` / `error` / `complete`。后台（`"background": true`）：返回带运行句柄的 `202`。同一会话上的运行是串行的。 |
| `GET` | `/session/{id}/runs/{runId}` | 后台运行状态：`running`、`exit_code`、`error`、时间戳。 |
| `GET` | `/session/{id}/runs/{runId}/logs` | 后台运行的组合输出，纯文本，支持字节游标分页。 |
| `DELETE` | `/session/{id}` | 销毁。 |
| `GET` | `/capabilities` | 探测。 |

`idle_timeout_seconds > 0` 自动销毁空闲会话；设为 `0` 可禁用空闲 GC，并始终显式 `DELETE`。

---

## 后台运行

后台运行在会话内以分离方式启动代码并立即返回；运行的组合 stdout/stderr 和退出码由 execd 捕获，客户端在其他工作在会话上继续的同时轮询它们。

```bash
# Start detached; 202 returns {"session_id", "run_id", "started_at"}.
RUN=$(curl -s -X POST "http://localhost:44772/v1/isolated/session/$SESSION/run" \
  -H "Content-Type: application/json" \
  -d '{"code": "sleep 5 && echo done", "background": true}' | jq -r .run_id)

# Poll status until running=false.
curl -s "http://localhost:44772/v1/isolated/session/$SESSION/runs/$RUN"

# Read output (plain text). The response header EXECD-ISOLATED-TAIL-CURSOR
# carries the next byte offset for incremental reads.
curl -s "http://localhost:44772/v1/isolated/session/$SESSION/runs/$RUN/logs"
curl -s "http://localhost:44772/v1/isolated/session/$SESSION/runs/$RUN/logs?cursor=123"
```

Python SDK：

```python
session = await sandbox.isolation.create(CreateIsolatedSessionRequest(
    workspace=IsolatedWorkspaceSpec(path="/workspace", mode="rw"),
))

run = await session.run_background("sleep 5 && echo done")
while (status := await session.run_status(run.run_id)).running:
    await asyncio.sleep(0.2)
print(status.exit_code)
logs = await session.run_logs(run.run_id)   # .text + .cursor for pagination
```

后台运行语义：

- `timeout_seconds` 仅适用于前台；后台运行不受时间限制。
- 后台运行活跃期间空闲 GC 暂停；运行结束后，正常空闲窗口适用。删除会话会杀死该运行。
- 后台运行需要可写的日志位置，因此只读（`ro`）工作区的会话会以 `400` 拒绝它们；支持 `rw` 和 `overlay` 工作区。
- 运行共享会话的进程组，因此会话级信号（例如前台运行超时或被取消时发送的 `SIGINT`）也会到达它们。
- 每次 `logs` 请求最多返回 16 MiB；使用返回的游标对大输出分页。每次运行的日志保留上限为 16 MiB —— 运行完成时超出上限的输出会被丢弃，因此如果需要超过第一页的内容，请在运行活跃时增量排空。
- 会话中途死亡的运行会报告 `running: false` 和 `error: "session terminated"`；会话被删除或垃圾回收时，运行记录及其日志会被移除。

---

## 工作区模式

| 模式 | 语义 |
|---|---|
| `rw` | 以读写方式绑定挂载；写入持久保存在主机上。 |
| `overlay`（默认） | 通过 overlayfs 写时复制；写入进入每会话的 upper 目录，`DELETE` 时消失。 |
| `ro` | 以只读方式绑定挂载；写入以 `EROFS` 失败。 |

Overlay upper 目录位于 `upper_root`（默认 `/var/lib/execd/isolation`）下。

因为隔离会话状态只存在于 execd 的内存中，execd 退出时遗留在 `upper_root` 下的每个会话目录都是陈旧的。在发布新会话之前，execd 会在 `<upper_root>/.execd-cleanup/<session-id>` 创建一个持久的清理记录；启动时它会恢复这些记录并移除精确匹配的会话目录。只有在整个会话子树被删除后，记录才会退役，因此部分删除和瞬时文件系统失败会在重启间保留可重试的身份。当前无法移除的残留——例如仍被上一生命周期中的挂载引用的 upper——仍计入 `upper_max_bytes`，并由空闲收集器重试。

只会移除有记录的会话目录。`upper_root` 下未记录的子项（包括分配器形态的目录树）被视为运维数据，绝不自动回收。因此，早于清理注册表的 execd 版本留下的目录需要显式迁移或手动清理；保持 `upper_root` 专用于 execd，不要修改保留的 `.execd-cleanup` 注册表。在池化/预配置沙箱中，这防止了某个占用者的会话数据泄露给下一个占用者。

```json
{ "workspace": { "path": "/workspace", "mode": "overlay" } }
```

---

## 多重 Overlay 挂载

会话可以通过 `overlays` 请求字段携带**多个独立的 overlay 挂载**（单一 `workspace` 仍受支持，作为单元素列表的语法糖；两者同时存在时，`workspace` 会被前置）：

```json
{
  "overlays": [
    { "path": "/", "mode": "overlay" },
    { "path": "/workspace", "mode": "overlay", "persist": true },
    { "path": "/data/scratch", "mode": "overlay", "persist": false }
  ]
}
```

- 每个条目有自己的挂载语义，因此工作区文件、系统级更改（`apt install` → `/usr`、配置 → `/etc`）和额外的项目目录会获得**独立的写时复制 upper**。
- 挂载按浅层优先应用；嵌套 overlay（例如在 `/` 根 overlay 之上的 `/workspace`）在其自身子树内遮蔽其祖先。路径必须是绝对且唯一的。
- `persist`（仅 overlay 模式，默认 `true`）：`true` 在 `upper_root` 下分配一个主机 upper 目录；`false` 使用临时的 tmpfs upper，其写入在会话结束时被丢弃。`rw`/`ro` 条目不得设置 `persist`。临时 upper 只存在于命名空间内，因此文件 API 从其主机侧内容提供 `persist=false` 的 overlay：此类 overlay 下的会话内写入无法通过文件 API 观察，且文件 API 向其写入会被拒绝。
- 文件 API 将每个请求路由到挂载路径是请求路径最长前缀的 overlay；相对路径相对于第一个 overlay 解析。
- 后台运行使用**第一个** overlay 作为其日志位置：它必须是 `rw`，或 `persist: true` 的 `overlay`；否则后台运行被拒绝。

---

## 绑定挂载与允许列表

- **`extra_writable`** —— 在相同路径（`source == destination`）以读写方式绑定挂载的路径。
- **`binds`** —— 显式 `source` → `dest` 映射，可选 `readonly`。`source` 必须已存在；`dest` 必须在命名空间内已存在（烘焙进沙箱镜像）。

```json
{
  "workspace": { "path": "/workspace", "mode": "rw" },
  "extra_writable": ["/data/scratch"],
  "binds": [
    { "source": "/data/in",  "dest": "/mnt/in",  "readonly": true },
    { "source": "/data/out", "dest": "/mnt/out" }
  ]
}
```

每个 `source` 在符号链接解析**之后**对照运维配置的 `allowed_writable` 允许列表进行检查，因此符号链接无法逃逸它。默认允许列表：`/workspace`、`/mnt`、`/media`、`/data`（允许子路径）。空允许列表拒绝所有额外绑定。

---

## 配置档与默认值

`profile` 字段目前只控制 `/tmp` 在命名空间内的暴露方式：

| 配置档 | `/tmp` |
|---|---|
| `strict`（默认） | 私有 tmpfs（`--tmpfs /tmp`） |
| `balanced` | 与沙箱共享（`--bind /tmp /tmp`） |

`workspace.mode` 和 `env_passthrough` **独立于**配置档：当它们被省略时，无论选择哪个配置档，execd 都会将 `workspace.mode` 规范化为 `overlay`，将 `env_passthrough.mode` 规范化为 `deny`。如果想要持久的工作区写入（`"rw"`）或主机环境变量透传（`"allow"`），请显式设置这些字段。

---

## 环境、UID 与网络

- **`env_passthrough`** —— `mode: "allow"` + `keys` 白名单主机环境变量；默认 `deny`。每次运行的覆盖放在 `IsolatedRunRequest.envs` 中。
- **`uid` / `gid`** 搭配 **`uid_mode: "setpriv"`**（默认，真实的 setuid/setgid 降权）或 **`"userns"`**（用户命名空间重映射）。在请求某个模式之前，先检查 [`/capabilities`](#capabilities-and-probing) 的 `setpriv_available` / `userns_available`。
- **`share_net: true`** 共享沙箱的网络命名空间。沙箱级的出站和 Credential Vault 策略仍然适用。
- **`share_net: false`** 创建私有网络命名空间。在释放工作负载启动门之前，execd 打开经过认证的 NetNS，用 `NS_GET_USERNS` 获取其真实所属的 UserNS，并将两者绑定固定在 `/run/execd/namespaces/<opaque-id>/` 之下。任何验证或固定失败都会中止会话创建。Execd 在启动失败、进程退出、显式删除、空闲收集和运行器关闭时尝试清理固定；可重试的失败保留会话所有权，并在 execd 存活期间重试。这适用于两种 UID 模式；加固的网络后端将要求 `uid_mode: "userns"`。

遗留默认值在本阶段保持不变：省略 `share_net` 继续共享沙箱网络命名空间。仅命名空间固定不会启用会话的出站或入站。

私有会话无法跨 execd 重启恢复。启用它们的部署必须将 execd 视为沙箱关键组件，在 execd 退出时重建整个沙箱/容器；不得在存活的挂载命名空间内启动替代 execd。销毁沙箱/容器会拆除该挂载命名空间并释放所有固定。Execd 有意在启动时不扫描并采用不透明的命名空间固定目录，因为它们不携带持久的沙箱代际，这样做可能会卸载另一个活跃进程的资源。

---

## 文件系统代理

从命名空间外部读写会话的**合并**工作区视图 —— 这就是 SDK 无需生成 shell 即可 `upload`、`download` 和 `list` 的方式。所有路径都是 `/v1/isolated/session/{id}/` 下的按会话路径：

| 方法 | 路径 |
|---|---|
| `GET` | `files/info?path=...` |
| `GET` | `files/download?path=...`（支持 `Range` 和 `offset`/`limit`） |
| `POST` | `files/upload`（multipart：`metadata` + `file`） |
| `DELETE` | `files?path=...` |
| `POST` | `files/mv` |
| `POST` | `files/permissions` |
| `POST` | `files/replace?verbose=true` |
| `GET` | `files/search?path=...&pattern=...` |
| `GET` | `directories/list?path=...&depth=N` |
| `POST` | `directories` |
| `DELETE` | `directories?path=...` |

`ro` 工作区上的写入会失败；`overlay` 上的写入进入 upper 目录，`DELETE` 时消失。

---

## 能力与探测

```bash
curl -s http://localhost:44772/v1/isolated/capabilities
```

```json
{
  "available": true,
  "isolator": "bwrap",
  "version": "0.9.0",
  "setpriv_available": true,
  "userns_available": false,
  "commit_supported": false,
  "diff_supported": false
}
```

- `available: false` —— 受信任的原生工作负载门缺失或不受信任、bubblewrap 缺失，或主机无法创建所需的命名空间（缺少 `CAP_SYS_ADMIN`、受限的 user-ns sysctl 等）。
- `setpriv_available` / `userns_available` —— 是否可以创建 `uid_mode: "setpriv"` 或 `"userns"` 的会话。`setpriv_available` 仅反映 execd 的**默认** UID/GID；请求不同 UID/GID 的会话在身份切换不可用时仍可能返回 `503 NOT_SUPPORTED`。
- `commit_supported` / `diff_supported` —— 阶段 2 的占位实现，当前返回 `503`。

---

## 服务器配置

让 execd 指向一个可选的 TOML 文件：

| 标志 | 环境变量 |
|---|---|
| `--isolation-config` | `EXECD_ISOLATION_CONFIG` |

```toml
# Parent directory for per-session overlay upper dirs.
upper_root = "/var/lib/execd/isolation"

# Allocation-time threshold for total overlay upper-directory size (bytes).
# Existing sessions can write beyond this value.
# Default: 8 GiB. Set to 0 to disable the allocation check.
upper_max_bytes = 8589934592  # 8 GiB

# Sources allowed for extra_writable / binds (symlink-resolved).
# Default: ["/workspace", "/mnt", "/media", "/data"]. Empty = reject all.
allowed_writable = ["/workspace", "/mnt", "/media", "/data"]
```

示例：`components/execd/configs/isolation.example.toml`。

`upper_max_bytes` 在创建 `overlay` 工作区（默认模式）时检查。如果一次成功的用量扫描报告的总用量达到或超过配置的正限额，新会话会被拒绝。该设置不限制已有会话的写入，也不适用于 `rw` 或 `ro` 工作区。

删除一个 overlay 会话可以在其清理成功且总用量降到阈值以下后恢复准入。删除会丢弃该会话的私有 upper 数据，因此删除前请保留任何需要的数据。

**主机要求：** execd 镜像中的 `bwrap` 和受信任的原生工作负载门；`CAP_SYS_ADMIN`（`uid_mode: "userns"` 还需要 `kernel.unprivileged_userns_clone=1`）；`overlay` 工作区需要内核中的 `overlayfs`。发布的镜像会自动安装该门。Linux 源码构建必须在启动 execd 前从 `components/execd` 运行 `make build-session-gate`，然后 `sudo make install-session-gate`。

注意：当原生工作负载门无法作为受信任的可执行文件打开，或 bwrap 本身无法启动（缺少二进制或缺少命名空间能力）时，`/capabilities` 报告 `available: false`。缺少 `overlayfs` **不会**翻转 `available` —— overlay 探测只影响阶段 2 的 `commit`/`diff` 支持，且默认 overlay 模式的会话创建在此类主机上运行时仍可能失败。如果你依赖 `workspace.mode: "overlay"`，请直接在主机上验证 `overlayfs` 支持。

---

## 限制

- **`diff` / `commit` 是阶段 2 的占位实现**，当前返回 `503`。
- **无硬件级保证。** 仅有命名空间 + seccomp；搭配安全运行时可防御内核漏洞利用。
- **仅限 Linux。** 非 Linux 构建返回 `available: false`。
- **每会话串行运行** —— 需要并行时创建多个会话。
- **绑定目标必须预先存在**于沙箱镜像中。

---

## 另请参阅

- [execd](/architecture/data-plane/execd)
- [安全容器运行时](/guides/secure-container)
- [execd OpenAPI 规范](/api/)
