---
title: Windows 沙箱
description: 通过 KVM/QEMU 在 Linux 容器内运行 Windows Guest，并经由 OpenSandbox lifecycle API 进行管理。
---

# Windows 沙箱

Windows 沙箱通过 KVM/QEMU 在 Linux 容器中运行 Windows Guest，并通过 OpenSandbox lifecycle API（创建、暴露端点、删除）进行端到端管理。当前的 Windows profile 基于 [`dockur/windows`](https://github.com/dockur/windows)，由 OpenSandbox 提供服务集成和能力封装。

## 范围

本指南介绍 OpenSandbox Server 的 Windows profile（`platform.os=windows`）：

- 创建 Windows 沙箱
- 查询状态和端点
- 常用资源和卷设置
- 删除与清理

## 工作原理

- **运行时**：[`dockur/windows`](https://github.com/dockur/windows) 在 Linux 容器中运行 Windows Guest（需要 KVM/QEMU），使用 `/storage`、`/boot.iso` 等挂载点存放系统盘和安装介质。  
- **控制平面**：创建时，OpenSandbox Server 注入 Windows profile 配置（设备、能力、`execd`、端口映射），并执行资源检查、状态管理和端点发布。  
- **客户端**：调用 Lifecycle API 或 Python SDK 并指定 `platform.os=windows`，然后按 创建 → 轮询 `Running` → 获取端点 → 使用 → 删除 的流程操作。

## 前提条件

- 可访问的 OpenSandbox Server（例如 `http://localhost:8080`）
- 主机满足 Windows profile 要求：
  - 存在 `/dev/kvm`
  - 存在 `/dev/net/tun`
- Server 已配置 `storage.allowed_host_paths` 以允许主机路径
- 一个用于绑定到 `/storage` 的主机目录（推荐）
- 可选：一个本地 Windows ISO 用于 `/boot.iso`（可省去重复下载时间）

## 关键设置与约束

### 平台与镜像

- 设置 `platform: {"os":"windows","arch":"amd64"}`（`arm64` 取决于主机）
- 建议镜像：`dockurr/windows:latest`；参见 [`dockur/windows` releases](https://github.com/dockur/windows/releases) 和 [Docker Hub 标签](https://hub.docker.com/r/dockurr/windows/tags)

### 最低资源

Windows profile 强制要求：

- `resourceLimits.cpu >= 2`
- `resourceLimits.memory >= 4G`
- `resourceLimits.disk >= 64G`

### `resourceLimits` 与环境变量

如果设置了 `resourceLimits`，server 会将其映射并覆盖以下环境变量：

- `CPU_CORES`
- `RAM_SIZE`
- `DISK_SIZE`

请只使用 `resourceLimits`；不要手动设置这三个环境变量。

### 默认端口与 `USER_PORTS`

Windows profile 默认接通以下端口：

- `44772`（execd）
- `8080`
- `3389/tcp`、`3389/udp`（RDP）
- `8006/tcp`（典型 Web 控制台）

它还会构建或合并 `USER_PORTS`（通常为 `44772,8080,3389,8006`）。

### 存储挂载

- 将可写的主机目录绑定到 `/storage`
- 可选：将 ISO 文件以只读方式绑定到 `/boot.iso`（`readOnly=true`）
- 主机路径必须位于 `storage.allowed_host_paths` 之下，否则请求被拒绝

## 端到端示例（Python）

该流程：

1. 创建 Windows 沙箱  
2. 等待其变为 `Running`  
3. 获取 `44772` 端点  
4. 通过 Python SDK 操作沙箱  
5. 删除沙箱  

### 最小示例

```python
import os
from datetime import timedelta

from opensandbox import SandboxSync
from opensandbox.config import ConnectionConfigSync
from opensandbox.models.sandboxes import PlatformSpec

BASE_URL = os.getenv("OPEN_SANDBOX_BASE_URL", "http://localhost:8080")
API_KEY = os.getenv("OPEN_SANDBOX_API_KEY", "")


def main() -> None:
    cfg = ConnectionConfigSync(
        domain=BASE_URL,
        api_key=API_KEY or None,
        use_server_proxy=True,
    )

    sbx = None
    try:
        sbx = SandboxSync.create(
            image="dockurr/windows:latest",
            timeout=timedelta(hours=12),
            ready_timeout=timedelta(minutes=30),
            resource={
                "cpu": "4",
                "memory": "8G",
                "disk": "64G",
            },
            env={"VERSION": "11"},
            entrypoint=["cmd", "/c", "echo OpenSandbox Windows profile"],
            platform=PlatformSpec(os="windows", arch="amd64"),
            connection_config=cfg,
        )
        print("created:", sbx.id)

        endpoint = sbx.get_endpoint(44772)
        print("execd endpoint:", endpoint.endpoint)
        print("sdk health:", sbx.is_healthy())
    finally:
        if sbx is not None:
            try:
                sbx.kill()
                print("deleted:", sbx.id)
            finally:
                sbx.close()


if __name__ == "__main__":
    main()
```

> 该示例保留 SDK 健康检查等待（`ready_timeout=30m`）。Windows 冷启动可能较慢；可按需增大 `ready_timeout`。若需要自定义就绪逻辑，请使用 `skip_health_check=True` 并在你的应用中轮询 `Running`。

### 高级选项

#### 持久化 `/storage`

将主机路径绑定到 `/storage`，即可得到持久化的系统/用户数据目录。该目录**无需预先存在**：它可以在绑定挂载时创建（全新），也可以在已存在时复用。无论目录是否存在，`host.path` 都必须位于允许的 `storage.allowed_host_paths` 前缀之下。

```python
from opensandbox.models.sandboxes import Host, Volume

volumes = [
    Volume(
        name="win-storage",
        host=Host(path="/data/opensandbox/windows-storage"),
        mount_path="/storage",
        read_only=False,
    ),
]
# Pass as SandboxSync.create(..., volumes=volumes, ...)
```

#### 使用主机上已有的 ISO

将本地 Windows 安装 ISO 以只读方式绑定到 `/boot.iso`，避免每次下载介质（同样需要 `allowed_host_paths`；路径必须是**文件**）：

```python
from opensandbox.models.sandboxes import Host, Volume

volumes = [
    Volume(
        name="win-iso",
        host=Host(path="/data/iso/Win11_23H2.iso"),
        mount_path="/boot.iso",
        read_only=True,
    ),
]
```

#### 自定义 `dockur/windows`

[`dockur/windows`](https://github.com/dockur/windows) 通过**环境变量**配置。OpenSandbox 会按上游文档将你的 `env` 传入容器。**安装 / 用户体验**相关的常见设置包括：

- `VERSION`：Windows 变体代码（例如 `11`、`11l`、`10` —— 完整表格见项目 README），某些情况下也可以是自定义 ISO 的 `https://` URL  
- `USERNAME` / `PASSWORD`：首个用户及其密码（省略时使用镜像内默认值 —— 见上游文档）  
- `LANGUAGE`：安装语言（例如 `English`、`Chinese`）  
- `REGION` / `KEYBOARD`：区域和键盘布局（例如 `en-US`、`zh-CN`）  
- 高级场景可能用到 `DHCP`、`ARGUMENTS`（额外的 QEMU 参数）等 —— 见 [`dockur/windows` FAQ](https://github.com/dockur/windows)

如果你已经通过 `resourceLimits` 设置了 CPU、内存和磁盘，就**不要**像 dockur 文档那样在 `env` 中重复设置这些**资源**键；参见上文的 [`resourceLimits` 与环境变量](#resourcelimits-与环境变量)。其他非资源键可以自由组合。

```python
env = {
    "VERSION": "11l",
    "USERNAME": "Docker",
    "PASSWORD": "your-secure-password",
    "LANGUAGE": "Chinese",
    "REGION": "zh-CN",
    "KEYBOARD": "zh-CN",
}
```

完整列表、默认值和交互行为，请查阅 [`dockur/windows` 仓库](https://github.com/dockur/windows)和镜像标签。

#### 使用你自己的 Windows 镜像

在 [`dockur/windows` 文档](https://github.com/dockur/windows)中，**"你自己的"** 通常指**使用哪个 Windows 安装 ISO** —— 而不是与上游无关的自定义配方。[自定义安装 FAQ](https://github.com/dockur/windows#how-do-i-install-a-custom-image) 说明：

- 对于不在项目版本表中的 ISO，将 `VERSION` 设置为该 ISO 的**下载 URL**（`https://...`）；容器会拉取并安装它。  
- 如果你**已有本地 ISO 文件**，请像上文[使用主机上已有的 ISO](#使用主机上已有的-iso)那样将它绑定到 **`/boot.iso`**；上游文档说明此时 `VERSION` 会被忽略。

在 OpenSandbox 中，在 `env` 里设置 `VERSION`，需要时为本地 ISO 在 `/boot.iso` 处添加一个卷，与上游流程保持一致；`image` 保持为 `dockurr/windows` 或其他兼容的上游镜像，与其他示例相同。

**`VERSION` = ISO URL** 场景的代码片段（与最小示例中传入 `env` 的方式相同）：

```python
env = {
    "VERSION": "https://example.com/path/to/your.iso",
    # same pattern as the minimal example for resource / platform / etc.
}
```

## FAQ

- 创建时出现 `Unsupported platform.os 'windows'` —— server 构建版本不含 Windows profile；请升级到包含该功能的版本。

- `resourceLimits.*` 报 `INVALID_PARAMETER` —— 请确保 `cpu >= 2`、`memory >= 4G`、`disk >= 64G`。

- 长时间停留在 `Pending` —— 首次 Windows 安装较慢；检查主机资源和 `/storage` 空间，并增大等待超时。

- 状态为 `Running` 但端点不可达 —— 验证 `GET /v1/sandboxes/{id}/endpoints/44772` 返回有效地址；如需转发更多端口，请设置 `env.USER_PORTS`。
