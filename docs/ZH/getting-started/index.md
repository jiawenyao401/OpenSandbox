---
title: 快速开始
description: 使用 Docker 和 Python，几分钟内在本地跑起 OpenSandbox。
---

# 快速开始

OpenSandbox 是一个面向 AI 应用的通用沙箱平台，提供多语言 SDK、统一的沙箱 API，以及 Docker/Kubernetes 运行时。

## 前提条件

- **Docker** Engine 20.10+（用于本地执行）
- **Python** 3.10+（用于服务端和 Python SDK）
- **uv**（推荐）或 pip

## 1. 启动服务端

```bash
# 生成初始配置
uvx opensandbox-server init-config ~/.sandbox.toml --example docker

# 启动服务端
uvx opensandbox-server
```

验证服务端是否已启动：

```bash
curl http://127.0.0.1:8080/health
# → {"status": "healthy"}
```

## 2. 安装 SDK

::: code-group

```bash [Python]
pip install opensandbox
```

```bash [JavaScript]
npm install @alibaba-group/opensandbox
```

```bash [Go]
go get github.com/alibaba/OpenSandbox/sdks/sandbox/go
```

```bash [C#]
dotnet add package Alibaba.OpenSandbox
```

:::

Kotlin/Java 请参见 [安装](/getting-started/installation)。

## 3. 创建并使用沙箱

本示例使用沙箱 SDK 和标准 Python 镜像。如果服务端已配置 API Key，请在客户端环境中设置 `OPEN_SANDBOX_API_KEY`。

```python
import asyncio
from datetime import timedelta

from opensandbox import Sandbox
from opensandbox.config import ConnectionConfig
from opensandbox.models import WriteEntry


async def main() -> None:
    sandbox = await Sandbox.create(
        "python:3.12",
        timeout=timedelta(minutes=10),
        connection_config=ConnectionConfig(
            domain="localhost:8080",
            use_server_proxy=True,
        ),
    )
    try:
        execution = await sandbox.commands.run("python -c 'print(1 + 1)'")
        for output in execution.logs.stdout:
            print(output.text)

        await sandbox.files.write_files([
            WriteEntry(path="/tmp/hello.txt", data="Hello World", mode=644)
        ])
        print(await sandbox.files.read_file("/tmp/hello.txt"))
    finally:
        await sandbox.destroy()


if __name__ == "__main__":
    asyncio.run(main())
```

`use_server_proxy=True` 会让沙箱流量经由生命周期服务端转发，适用于客户端无法直接访问容器地址的场景。`destroy()` 会终止远程沙箱并释放本地 SDK 资源。仅关闭客户端并不会终止远程沙箱。

## 4. 试用 CLI

```bash
pip install opensandbox-cli

osb config init
osb config set connection.domain localhost:8080
osb config set connection.protocol http
osb config set connection.use_server_proxy true
osb sandbox create --image python:3.12 --timeout 30m -o json
osb command run <sandbox-id> -o raw --argv -- python -c "print(1 + 1)"
osb sandbox kill <sandbox-id> -o json
```

## 后续步骤

- [安装](/getting-started/installation) — 全部 SDK 与运行时的详细安装说明
- [配置](/getting-started/configuration) — 服务端配置参考
- [架构](/architecture/) — OpenSandbox 的底层工作原理
- [指南](/guides/) — Client Pool、链路追踪、诊断与安全特性
- [SDK 能力矩阵](/sdks/#capability-coverage) — 选择语言前先确认支持情况
- [示例](/examples/) — 实际使用示例
