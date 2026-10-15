---
title: Docker PVC 卷
description: 使用 OpenSandbox 的 pvc 后端将 Docker 命名卷挂载到沙箱容器中。
---

# Docker PVC（命名卷）挂载示例

本示例演示如何使用 OpenSandbox 的 `pvc` 后端将 Docker 命名卷挂载到沙箱容器中。在 Docker 运行时中，`pvc.claimName` 映射到一个 Docker 命名卷——为跨沙箱共享数据提供了一种比宿主机路径绑定挂载更便捷、更安全的替代方案。

::: info 什么是 `pvc`？
`pvc` 后端是一个与运行时无关的抽象。在 Kubernetes 中它映射到 PersistentVolumeClaim；在 Docker 中它映射到命名卷。同一个 API 请求在两种运行时上都可以工作。
:::

## 为什么选择命名卷而非宿主机路径？

| | 宿主机路径（`host` 后端） | 命名卷（`pvc` 后端） |
|---|---|---|
| **安全性** | 暴露宿主机文件系统路径 | Docker 管理存储位置；不暴露宿主机路径 |
| **配置** | 需要 `allowed_host_paths` 白名单 | 无需白名单 |
| **跨沙箱共享** | 所有容器必须就同一宿主机路径达成一致 | 引用同一个卷名即可 |
| **可移植性** | 依赖宿主机目录结构 | 在任何 Docker 主机上都能工作 |
| **生命周期** | 用户自行管理宿主机目录 | `docker volume create/rm` |

## 场景

| # | 场景 | 说明 |
|---|----------|-------------|
| 1 | **读写挂载** | 挂载一个命名卷，用于双向文件 I/O |
| 2 | **只读挂载** | 挂载一个沙箱无法修改的命名卷 |
| 3 | **跨沙箱共享** | 两个沙箱通过同一个命名卷共享数据 |
| 4 | **SubPath 挂载** | 仅挂载命名卷的一个子目录（与 K8s PVC subPath 一致） |

## 前提条件

### 1. 启动 OpenSandbox 服务器

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
opensandbox-server
```

### 2. 创建 Docker 命名卷

```shell
# Create the named volume
docker volume create opensandbox-pvc-demo

# Seed it with a marker file via a temporary container
docker run --rm -v opensandbox-pvc-demo:/data alpine \
  sh -c "echo 'hello-from-named-volume' > /data/marker.txt"
```

### 3. 安装 Python SDK

```shell
uv pip install opensandbox
```

### 4. 拉取沙箱镜像

```shell
docker pull ubuntu:latest
```

## 运行

```shell
uv run python examples/docker-pvc-volume-mount/main.py
```

该脚本会自动创建命名卷并写入测试数据。你也可以指定自定义卷名或镜像：

```shell
SANDBOX_IMAGE=ubuntu SANDBOX_DOMAIN=localhost:8080 uv run python examples/docker-pvc-volume-mount/main.py
```

## 预期输出

```text
OpenSandbox server : localhost:8080
Sandbox image      : ubuntu
Docker volume      : opensandbox-pvc-demo
  Ensuring Docker named volume 'opensandbox-pvc-demo' exists...
  Created volume 'opensandbox-pvc-demo' with marker.txt

============================================================
Scenario 1: Read-Write PVC (Named Volume) Mount
============================================================
  ...

============================================================
Scenario 2: Read-Only PVC (Named Volume) Mount
============================================================
  ...

============================================================
Scenario 3: Cross-Sandbox Sharing via PVC (Named Volume)
============================================================
  ...

============================================================
Scenario 4: SubPath PVC (Named Volume) Mount
============================================================
  ...

============================================================
All scenarios completed successfully!
============================================================
```

## SDK 用法速查

### Python（异步）

```python
from opensandbox import Sandbox
from opensandbox.models.sandboxes import PVC, Volume

sandbox = await Sandbox.create(
    image="ubuntu",
    volumes=[
        Volume(
            name="my-data",
            pvc=PVC(claimName="my-named-volume"),
            mountPath="/mnt/data",
            readOnly=False,       # optional, default is False
            subPath="datasets/train",  # optional, mount a subdirectory
        ),
    ],
)
```

### Python（同步）

```python
from opensandbox import SandboxSync
from opensandbox.models.sandboxes import PVC, Volume

sandbox = SandboxSync.create(
    image="ubuntu",
    volumes=[
        Volume(
            name="my-data",
            pvc=PVC(claimName="my-named-volume"),
            mountPath="/mnt/data",
            subPath="datasets/train",  # optional
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
      pvc: { claimName: "my-named-volume" },
      mountPath: "/mnt/data",
      readOnly: false,
      subPath: "datasets/train",  // optional
    },
  ],
});
```

### Java / Kotlin

```java
Volume volume = Volume.builder()
    .name("my-data")
    .pvc(PVC.of("my-named-volume"))
    .mountPath("/mnt/data")
    .readOnly(false)
    .subPath("datasets/train")  // optional
    .build();

Sandbox sandbox = Sandbox.builder()
    .image("ubuntu")
    .volume(volume)
    .build();
```

## 清理

```shell
docker volume rm opensandbox-pvc-demo
```

## 参考资料

- [沙箱生命周期 API 规范](https://github.com/opensandbox-group/OpenSandbox/blob/main/specs/sandbox-lifecycle.yml) -- 卷定义的 OpenAPI schema
- [宿主机卷挂载示例](/examples/host-volume-mount) -- 宿主机路径绑定挂载示例（另一种方案）
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/docker-pvc-volume-mount)
