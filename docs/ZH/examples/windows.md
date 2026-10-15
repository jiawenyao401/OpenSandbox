---
title: Windows
description: 使用 dockur/windows 镜像通过 KVM/QEMU 在 OpenSandbox 沙箱中运行 Windows 客户机。
---

# Windows 沙箱示例

使用 [`dockur/windows`](https://github.com/dockur/windows) 镜像，通过 KVM/QEMU 在 OpenSandbox 沙箱中运行 Windows 客户机。

## 工作原理

OpenSandbox 创建一个运行 KVM/QEMU 的 Linux 容器，在其中启动 Windows 客户机操作系统。Windows 配置文件（`platform.os=windows`）会自动配置所需的设备、能力、OEM 脚本和端口映射——你只需在 SDK 调用中指定 `platform` 和 `resource`。

## 前置条件

- OpenSandbox 服务器已运行（例如 `http://localhost:8080`）
- 宿主机存在 `/dev/kvm` 和 `/dev/net/tun`
- 对于任何宿主机 bind mount，服务器已配置 `storage.allowed_host_paths`

## 启动 OpenSandbox 服务器 [本地]

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
opensandbox-server
```

## 运行示例

```shell
uv pip install opensandbox
cd examples/windows
python main.py
```

该脚本会：

1. 使用 `dockurr/windows:latest` 和 Windows 11 创建 Windows 沙箱
2. 等待沙箱变为健康状态（首次启动可能需要几分钟）
3. 打印 execd、RDP（3389）和 Web 控制台（8006）端点
4. 执行一个测试命令并打印输出

## 环境变量

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `SANDBOX_DOMAIN` | `localhost:8080` | 沙箱服务地址 |
| `SANDBOX_API_KEY` | _（本地可选）_ | 若服务器要求认证，则提供 API key |

## 自定义

### 资源限制

Windows 配置文件强制最小资源：**cpu >= 2，memory >= 4G，disk >= 64G**。示例使用 4 CPU、8G 内存和 64G 磁盘。你可以在 `main.py` 的 `resource` 字典中调整这些值。

### 持久化存储

将宿主机目录绑定到 `/storage` 以获得持久化的系统磁盘（添加到 `SandboxSync.create` 调用中）：

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
```

### 本地 ISO

将 Windows 安装 ISO 绑定到 `/boot.iso`，以避免重复下载：

```python
volumes = [
    Volume(
        name="win-iso",
        host=Host(path="/data/iso/Win11_23H2.iso"),
        mount_path="/boot.iso",
        read_only=True,
    ),
]
```

### Windows 客户机配置

通过 `env` 参数传递 [dockur/windows 环境变量](https://github.com/dockur/windows)：

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

::: warning
不要手动设置 `CPU_CORES`、`RAM_SIZE` 或 `DISK_SIZE`——它们会从 `resourceLimits` 自动派生。
:::

## 暴露的端口

| 端口 | 服务 |
|------|---------|
| 44772 | execd（沙箱执行 API） |
| 8080 | HTTP 服务 |
| 3389 | RDP（原生远程桌面） |
| 8006 | Web 控制台（noVNC） |

## 故障排查

- **`Unsupported platform.os 'windows'`**：服务器构建不包含 Windows 配置文件；请升级 OpenSandbox 服务器。
- **resourceLimits 报 `INVALID_PARAMETER`**：确保 cpu >= 2，memory >= 4G，disk >= 64G。
- **长时间停留在 Pending**：首次安装 Windows 较慢；检查宿主机资源和 `/storage` 空间，并增大 `ready_timeout`。
- **状态为 Running 但端点不可达**：验证端点解析返回有效地址；如需转发额外端口，请检查 `USER_PORTS`。

### ENI CNI 网络问题（阿里云 ACK）

在使用基于 ENI 的 CNI（例如 ENI 模式下的阿里云 ACK Terway）的集群上，dockur/windows 启动时会失败，报错：

```
> ERROR: This container does not support host mode networking!
```

或：

```
> ERROR: Status 1 while: ethtool -i "$VM_NET_DEV"
```

**根本原因**：镜像的 `network.sh` 使用 `ethtool -i` 检查网络接口。ENI 接口带有真实的 PCI bus-info，会触发错误的"host 模式"判定。基于标准 veth 的 CNI（Calico、Flannel、Cilium）不会有此问题。

**解决方案**：使用提供的 `main_fix_net.py` 示例，它在运行时修补脚本并为 QEMU 用户态 NAT 设置 `NETWORK=slirp`：

```shell
cd examples/windows
python main_fix_net.py
```

完整实现见 [`main_fix_net.py`](https://github.com/opensandbox-group/OpenSandbox/blob/main/examples/windows/main_fix_net.py)。

**工作原理**：

1. `sed` 将 `/run/network.sh` 中的三行替换为空变量赋值（`result=""`、`nic=""`、`bus=""`），防止 ethtool 检查中止脚本。
2. `NETWORK=slirp` 告知脚本使用 QEMU 的 SLIRP 网络（用户态 NAT），它不需要真实网卡。
3. `exec /usr/bin/tini -s /run/entry.sh` 在修补后启动原始镜像入口。

这种方式保留了 Pod 的独立 IP，且无需重建镜像或使用 `hostNetwork`。

## 从 Pool 启动 Windows 沙箱

使用预热的 K8s pool 来加快 Windows 沙箱的启动。

### 1. 创建 pool

应用 pool manifest（镜像、资源、设备挂载和 OEM 脚本均已预配置）：

```shell
cd examples/windows
kubectl apply -f pool-win-example.yaml
```

### 2. 启动 OpenSandbox 服务器 [k8s]

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example k8s
opensandbox-server
```

### 3. 运行 pool 示例

```shell
uv pip install opensandbox
cd examples/windows
python main_use_pool.py
```

该脚本会从 `pool-win-example` 获取一个沙箱，打印端点，并运行一个命令。

### 环境变量（pool）

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `SANDBOX_DOMAIN` | `localhost:8080` | 沙箱服务地址 |
| `SANDBOX_API_KEY` | _（可选）_ | 若服务器要求认证，则提供 API key |

## 参考

- [Windows 沙箱指南](/guides/windows-sandbox)
- [dockur/windows](https://github.com/dockur/windows)
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/windows)
