---
title: OpenClaw
description: 在 OpenSandbox 实例中启动 OpenClaw Gateway 并暴露其 HTTP 端点。
---

# OpenClaw Gateway 示例

在 OpenSandbox 实例中启动 [OpenClaw](https://github.com/openclaw/openclaw) Gateway 并暴露其 HTTP 端点。脚本会轮询网关直至其返回 HTTP 200，然后打印可访问的端点。

## 快速开始

```shell
# 安装依赖
uv pip install opensandbox requests

# 使用默认配置运行
uv run python examples/openclaw/main.py
```

## 配置选项

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `OPENCLAW_SERVER` | `http://localhost:8080` | OpenSandbox 服务器地址 |
| `OPENCLAW_TOKEN` | `dummy-token-for-sandbox` | 网关认证令牌 |
| `OPENCLAW_IMAGE` | `ghcr.io/openclaw/openclaw:latest` | 容器镜像 |
| `OPENCLAW_TIMEOUT` | `3600` | 沙箱超时时间（秒） |

## 网络策略

默认情况下，沙箱拒绝除 `pypi.org`（用于安装包）之外的所有网络访问。你可以在 `main.py` 中自定义：

```python
network_policy=NetworkPolicy(
    defaultAction="deny",
    egress=[
        NetworkRule(action="allow", target="pypi.org"),
        NetworkRule(action="allow", target="pypi.python.org"),
        # Add more allowed targets
    ],
)
```

## OpenClaw 的环境变量

向沙箱内的 OpenClaw Gateway 传递环境变量：

```python
env={
    "OPENCLAW_GATEWAY_TOKEN": token,
    "OPENCLAW_MODEL": "claude-sonnet-4-20250514",
    # Add more env vars as needed
},
```

## 启动 OpenSandbox 服务器 [本地]

你可以在[这里](https://github.com/openclaw/openclaw/pkgs/container/openclaw)找到最新的 OpenClaw 容器镜像。

::: info Docker 运行时要求
服务器默认使用 `runtime.type = "docker"`，因此它**必须**能够连接到正在运行的 Docker daemon。

- **Docker Desktop**：确保 Docker Desktop 正在运行，然后用 `docker version` 验证。
- **Colima (macOS)**：先启动它（`colima start`），并在启动服务器前导出 socket：

```shell
export DOCKER_HOST="unix://${HOME}/.colima/default/docker.sock"
```
:::

预先拉取 OpenClaw 镜像：

```shell
docker pull ghcr.io/openclaw/openclaw:latest
```

启动 OpenSandbox 服务器（日志将保留在终端中）：

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
opensandbox-server
```

## 创建并访问 OpenClaw 沙箱

本示例为快速上手做了硬编码：
- OpenSandbox 服务器：`http://localhost:8080`
- 镜像：`ghcr.io/openclaw/openclaw:latest`
- 网关端口：`18789`
- 超时：`3600s`
- 令牌：`OPENCLAW_GATEWAY_TOKEN`（默认：`dummy-token-for-sandbox`）

从项目根目录安装依赖：

```shell
uv pip install opensandbox requests
```

运行示例（如需认证访问，请设置真实的令牌）：

```shell
export OPENCLAW_GATEWAY_TOKEN="$(openssl rand -hex 32)"
uv run python examples/openclaw/main.py
```

你应该会看到类似以下的输出：

```text
Creating openclaw sandbox with image=ghcr.io/openclaw/openclaw:latest on OpenSandbox server http://localhost:8080...
[check] sandbox ready after 7.1s
Openclaw started finished. Please refer to 127.0.0.1:56123
```

## 进阶：自定义网关端口

要使用自定义端口，请修改 `main.py` 中的 `entrypoint`：

```python
entrypoint=["node dist/index.js gateway --bind=lan --port 19999 --allow-unconfigured --verbose"],
```

然后在 `get_endpoint()` 调用中更新端口：

```python
endpoint = sandbox.get_endpoint(19999)
```

## 参考

- [OpenClaw](https://github.com/openclaw/openclaw)
- [OpenSandbox Python SDK](https://pypi.org/project/opensandbox/)
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/openclaw)
