---
title: Docker OSSFS 卷
description: 在 Docker 运行时上使用 OSSFS 卷模型将阿里云 OSS 挂载到沙箱中。
---

# Docker OSSFS 卷挂载示例

本示例演示如何使用 SDK 的 `ossfs` 卷模型，在 Docker 运行时上将阿里云 OSS 挂载到沙箱中。

## 本示例涵盖的内容

1. OSSFS 后端上的**基本读写挂载**。
2. 同一 OSSFS 后端路径上的**跨沙箱共享**。
3. 通过 `subPath` 实现的**两个挂载使用不同的 OSS 前缀**。

## 前提条件

### 1. 启动 OpenSandbox 服务器（Docker 运行时）

确保你的服务器主机具备：

- Linux 宿主操作系统（当 OpenSandbox Server 运行在 Windows 上时不支持 OSSFS 后端）
- 已安装 `ossfs`
- 已启用 FUSE 支持
- OSSFS 的可写本地挂载根目录（默认 `storage.ossfs_mount_root=/mnt/ossfs`）

如果使用默认的 `/mnt/ossfs`，`storage.ossfs_mount_root` 是**可选**的。
即使采用按需挂载，运行时仍然需要一个确定的宿主机侧
基础目录来放置动态挂载（`<mount_root>/<bucket>/<subPath?>`）。

可选配置示例：

```toml
[runtime]
type = "docker"

[storage]
ossfs_mount_root = "/mnt/ossfs"
```

然后启动服务器：

```bash
opensandbox-server
```

### 2. 安装 Python SDK

```bash
uv pip install opensandbox
```

如果你的 PyPI 版本尚不包含 OSSFS 卷模型，请从源码安装：

```bash
pip install -e sdks/sandbox/python
```

### 3. 准备 OSS 凭据和目标路径

```bash
export SANDBOX_DOMAIN=localhost:8080
export SANDBOX_API_KEY=your-api-key
export SANDBOX_IMAGE=ubuntu

export OSS_BUCKET=your-bucket
export OSS_ENDPOINT=oss-cn-hangzhou.aliyuncs.com
export OSS_ACCESS_KEY_ID=your-ak
export OSS_ACCESS_KEY_SECRET=your-sk
```

## 运行

```bash
uv run python examples/docker-ossfs-volume-mount/main.py
```

## 最小化 SDK 用法片段

```python
from opensandbox import Sandbox
from opensandbox.models.sandboxes import OSSFS, Volume

sandbox = await Sandbox.create(
    image="ubuntu",
    volumes=[
        Volume(
            name="oss-data",
            ossfs=OSSFS(
                bucket="your-bucket",
                endpoint="oss-cn-hangzhou.aliyuncs.com",
                # version="2.0",   # optional, default is "2.0"
                accessKeyId="your-ak",
                accessKeySecret="your-sk",
            ),
            mountPath="/mnt/data",
            subPath="train",      # optional
            readOnly=False,       # optional
        )
    ],
)
```

## 注意事项

::: info 实现细节
- 当前实现仅支持**内联凭据**（`accessKeyId`/`accessKeySecret`）。
- Docker 运行时中的挂载是**按需**的（挂载或复用），而不是为所有存储桶预先挂载。
- `ossfs.version` 存在于 API/SDK 中，枚举值为 `"1.0" | "2.0"`，省略时默认为 `"2.0"`。
- Docker 运行时现在应用**特定于版本的挂载参数编码**：
  - `1.0`：通过 `ossfs ... -o <option>` 挂载。
  - `2.0`：通过 `ossfs2 mount ... -c <config-file>` 挂载，其中选项以 `--<option>` 配置行的形式书写。
- `options` 的值必须是**原始负载**，不能带前导 `-`（例如：`allow_other`、`umask=0022`）。
:::

## 参考资料

- [沙箱生命周期 API 规范](https://github.com/opensandbox-group/OpenSandbox/blob/main/specs/sandbox-lifecycle.yml)
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/docker-ossfs-volume-mount)
