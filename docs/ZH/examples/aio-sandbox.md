---
title: AIO Sandbox
description: 通过 OpenSandbox 创建并访问 All-in-One（AIO）沙箱。
---

# All-in-One（AIO）沙箱示例

本示例演示如何通过 OpenSandbox 创建并访问 [All-in-One（AIO）沙箱](https://github.com/agent-infra/sandbox)。

## 启动 OpenSandbox 服务器 [本地]

你可以在[这里](https://github.com/agent-infra/sandbox/pkgs/container/sandbox)找到最新版本。

你可以预先拉取示例中使用的目标镜像。

::: info Docker 运行时要求
服务器默认配置为 `runtime.type = "docker"`，因此它**必须**能够连接到正在运行的 Docker 守护进程。

- **Docker Desktop**：确保 Docker Desktop 正在运行，然后用 `docker version` 验证。
- **Colima（macOS）**：先启动它（`colima start`），并在启动服务器之前导出 socket：

```shell
export DOCKER_HOST="unix://${HOME}/.colima/default/docker.sock"
```
:::

```shell
# 预先拉取目标镜像
docker pull ghcr.io/agent-infra/sandbox:latest
```

然后启动 OpenSandbox 服务器，你可以从终端获取 stdout 日志。

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
opensandbox-server
```

::: tip
`opensandbox-server` 在前台运行，会一直占用当前终端会话。示例代码位于本仓库中——请克隆它，并在新的终端窗口/标签页中 `cd` 到项目根目录，然后执行下方的 AIO 沙箱创建步骤。

如果你看到来自 `docker/transport/unixconn.py` 的 `FileNotFoundError: [Errno 2] No such file or directory` 之类的错误，通常意味着 Docker unix socket 缺失或 Docker 守护进程未运行。
:::

## 创建并访问 AIO 沙箱实例

本示例使用固定配置以便快速上手：
- OpenSandbox 服务器：`http://localhost:8080`
- 镜像：`ghcr.io/agent-infra/sandbox:latest`
- AIO 端口：`8080`
- 超时时间：`300s`

在项目根目录下使用 uv 安装依赖：

```shell
uv pip install opensandbox agent-sandbox==0.0.18
```

运行示例（它会通过 OpenSandbox 创建沙箱，等待其进入 Running 状态，然后通过 agent-sandbox 连接到它）：

```shell
uv run python examples/aio-sandbox/main.py
```

随后，你将实例化一个 AIO 沙箱，导航到 Google，截取屏幕截图，并将其下载到本地环境。

```text
Creating AIO sandbox with image=ghcr.io/agent-infra/sandbox:latest on OpenSandbox server http://localhost:8080...
[check] sandbox ready after 7.1s
AIO portal endpoint: 127.0.0.1:56123
...
Screenshot saved to sandbox_screenshot.png
```

![AIO Sandbox screenshot](../public/images/aio-sandbox-screenshot.png)

## 更多示例

有关使用 AIO Sandbox 的更多示例，请参阅 agent-infra/sandbox 的[示例](https://github.com/agent-infra/sandbox/tree/main/examples)。

## 参考资料

- [AIO Sandbox](https://github.com/agent-infra/sandbox/tree/main)
- [AIO Sandbox Python SDK](https://github.com/agent-infra/sandbox/tree/main/sdk/python)
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/aio-sandbox)
