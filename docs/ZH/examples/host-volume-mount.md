---
title: Host Volume Mount
description: 使用 OpenSandbox Volume API 将宿主机目录挂载到沙箱容器中，实现双向文件共享。
---

# Host Volume Mount 示例

本示例演示如何使用 OpenSandbox Volume API 将宿主机目录挂载到沙箱容器中。Host volume 挂载支持宿主机与沙箱环境之间的双向文件共享——非常适合共享数据集、模型检查点、配置文件，或收集沙箱的输出。

## 场景

| # | 场景 | 说明 |
|---|----------|-------------|
| 1 | **读写挂载** | 挂载宿主机目录，进行双向文件交换 |
| 2 | **只读挂载** | 提供沙箱无法修改的共享数据 |
| 3 | **SubPath 挂载** | 从宿主机路径挂载特定的子目录 |

## 前置条件

### 1. 启动 OpenSandbox 服务器

```shell
git clone git@github.com:opensandbox-group/OpenSandbox.git
cd OpenSandbox/server
cp opensandbox_server/examples/example.config.toml ~/.sandbox.toml
uv sync && uv run python -m opensandbox_server.main
```

### 2. 配置允许的宿主机路径

出于安全考虑，服务器会限制可挂载的宿主机路径。在 `~/.sandbox.toml` 中添加 `[storage]` 配置段：

```toml
[storage]
# Allowlist of host path prefixes permitted for bind mounts.
# Only paths under these prefixes can be mounted into sandboxes.
# If empty, all host paths are allowed (not recommended for production).
allowed_host_paths = ["/tmp/opensandbox-data", "/data/shared"]
```

::: warning 安全提示
在生产环境中，务必设置明确的 `allowed_host_paths`，防止沙箱访问敏感的宿主机目录。空列表表示允许所有路径，这对本地开发很方便，但在共享环境中并不安全。
:::

### 3. 创建宿主机目录

```shell
# Create a directory to share with sandboxes
mkdir -p /tmp/opensandbox-data
echo "hello-from-host" > /tmp/opensandbox-data/marker.txt

# Create a subdirectory for the subpath demo
mkdir -p /tmp/opensandbox-data/datasets/train
echo -e "id,value\n1,100\n2,200\n3,300" > /tmp/opensandbox-data/datasets/train/data.csv
```

### 4. 从源码安装 SDK

Volume 支持需要从源码构建的最新 SDK（已发布的包中尚不可用）：

```shell
# From the project root (recommended: use uv)
uv pip install -e sdks/sandbox/python

# Or use pip inside a virtual environment
# python3 -m venv .venv && source .venv/bin/activate
# pip install -e sdks/sandbox/python
```

### 5. 拉取沙箱镜像

```shell
docker pull ubuntu:latest
```

## 运行

```shell
HOST_VOLUME_PATH=/tmp/opensandbox-data uv run python examples/host-volume-mount/main.py
```

## 预期输出

```text
Using HOST_VOLUME_PATH: /tmp/opensandbox-data

OpenSandbox server : localhost:8080
Sandbox image      : ubuntu
Host volume path   : /tmp/opensandbox-data

============================================================
Scenario 1: Read-Write Host Volume Mount
============================================================
  Host path : /tmp/opensandbox-data
  Mount path: /mnt/shared

  [1] Listing files visible from inside the sandbox:
  ...

  [2] Writing a file from inside the sandbox:
  -> Written: /mnt/shared/sandbox-greeting.txt

  [3] Reading back the file:
  Hello from sandbox!

  Scenario 1 completed.

============================================================
Scenario 2: Read-Only Host Volume Mount
============================================================
  ...

============================================================
Scenario 3: SubPath Host Volume Mount
============================================================
  ...

============================================================
All scenarios completed successfully!
============================================================
```

## SDK 用法速查

### Python (async)

```python
from opensandbox import Sandbox
from opensandbox.models.sandboxes import Host, Volume

sandbox = await Sandbox.create(
    image="ubuntu",
    volumes=[
        Volume(
            name="my-data",
            host=Host(path="/data/shared"),
            mountPath="/mnt/data",
            readOnly=False,       # optional, default is False
            subPath="subdir",     # optional, mount a subdirectory
        ),
    ],
)
```

### Python (sync)

```python
from opensandbox import SandboxSync
from opensandbox.models.sandboxes import Host, Volume

sandbox = SandboxSync.create(
    image="ubuntu",
    volumes=[
        Volume(
            name="my-data",
            host=Host(path="/data/shared"),
            mountPath="/mnt/data",
        ),
    ],
)
```

### JavaScript / TypeScript

```typescript
import { Sandbox } from "@alibaba-group/opensandbox";

const sandbox = await Sandbox.create({
  image: "ubuntu",
  volumes: [
    {
      name: "my-data",
      host: { path: "/data/shared" },
      mountPath: "/mnt/data",
      readOnly: false,
    },
  ],
});
```

### Java / Kotlin

```java
Volume volume = Volume.builder()
    .name("my-data")
    .host(Host.of("/data/shared"))
    .mountPath("/mnt/data")
    .readOnly(false)
    .build();

Sandbox sandbox = Sandbox.builder()
    .image("ubuntu")
    .volume(volume)
    .build();
```

## 参考

- [Sandbox Lifecycle API 规范](https://github.com/opensandbox-group/OpenSandbox/blob/main/specs/sandbox-lifecycle.yml) —— volume 定义的 OpenAPI schema
- [服务器配置](https://github.com/opensandbox-group/OpenSandbox/blob/main/server/opensandbox_server/examples/example.config.toml) —— `allowed_host_paths` 所属的 `[storage]` 配置段
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/host-volume-mount)
