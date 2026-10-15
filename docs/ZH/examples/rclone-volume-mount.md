---
title: rclone Volume Mount
description: 使用现有的 volume API 将 rclone 支持的 Docker volume 或 Kubernetes PVC 挂载到 OpenSandbox。
---

# rclone Volume Mount

使用外部创建的 rclone volume 将远程存储暴露给沙箱。
OpenSandbox 通过其现有的 `pvc` 后端挂载该 volume；Docker
volume 插件或 Kubernetes CSI driver 负责管理远程配置、
凭据、FUSE 进程、缓存和卸载生命周期。

此集成不会向 OpenSandbox API 添加 `rclone` 后端或 `driver_opts` 字段。请在创建沙箱之前配置好存储 driver。

## Docker 配置

使用支持 FUSE 的 Linux Docker 宿主机、rclone CLI，以及配置为 Docker 运行时的 OpenSandbox 服务器。请在服务器所用的同一个 Docker daemon 上执行以下配置。沙箱镜像无需 rclone 或额外的 FUSE 权限。

### 配置 remote 并安装插件

创建插件目录并配置一个名为 `storage` 的 remote。在 rclone 交互式配置向导中选择你的后端和凭据：

```shell
sudo install -d -m 700 /var/lib/docker-plugins/rclone/config
sudo install -d -m 700 /var/lib/docker-plugins/rclone/cache
sudo rclone config --config /var/lib/docker-plugins/rclone/config/rclone.conf
sudo chmod 600 /var/lib/docker-plugins/rclone/config/rclone.conf

# This example uses the amd64 plugin; select the image for your host architecture.
docker plugin install rclone/docker-volume-rclone:amd64 --alias rclone
docker plugin ls
```

当 Docker 提示时，请审查插件请求的权限。该插件运行在沙箱之外，需要宿主机级别的 FUSE 访问权限。宿主机前置条件、架构选择和插件升级请遵循
[rclone Docker 插件文档](https://rclone.org/docker/)。

请将存储凭据保存在由运维人员管理的配置中。不要将密码或令牌放入沙箱环境变量，也不要在示例中提交它们。

### 创建命名 volume

在远程端使用一个专用的、可写的测试目录。将
`storage:opensandbox-demo` 替换为适合你后端的路径（例如
S3 bucket 及前缀，或 SFTP 目录）：

```shell
docker volume create --driver rclone \
  --opt remote=storage:opensandbox-demo \
  --opt allow-other=true \
  --opt vfs-cache-mode=writes \
  opensandbox-rclone-demo

docker volume inspect --format '{{.Driver}}' opensandbox-rclone-demo
docker run --rm --mount type=volume,src=opensandbox-rclone-demo,dst=/data \
  alpine:3.21 ls -la /data
```

确认 inspect 输出的 driver 是 `rclone`（或你安装的插件名），
并且临时容器可以访问预期目录。仅创建命名 volume 并不能验证远程连通性。

::: warning 必需的 Docker 设置
- 设置 `createIfNotExists=False`。否则，volume 名称缺失或拼写错误可能导致服务器创建一个不带 rclone 选项的普通 Docker volume。
- 省略 `subPath`：Docker 的 `subPath` 处理目前需要 `local` driver。请改用插件的 `remote` 选项来限定远程目录。
- 请在服务器的 Docker daemon 上创建该 volume，即使 SDK 运行在另一台机器上。
:::

## 从 SDK 挂载

同一个请求既适用于预先创建的 Docker 命名 volume，也适用于 Kubernetes PVC：

```python
from opensandbox import Sandbox
from opensandbox.models.sandboxes import PVC, Volume

sandbox = await Sandbox.create(
    image="python:3.11",
    volumes=[
        Volume(
            name="remote-storage",
            pvc=PVC(
                claimName="opensandbox-rclone-demo",
                createIfNotExists=False,
                deleteOnSandboxTermination=False,
            ),
            mountPath="/mnt/remote",
            readOnly=True,
        ),
    ],
)

async with sandbox:
    try:
        result = await sandbox.commands.run("ls -la /mnt/remote")
        if result.error or result.exit_code not in (None, 0):
            raise RuntimeError("Could not read the remote volume")
        print("".join(message.text for message in result.logs.stdout))
    finally:
        await sandbox.kill()
```

## 运行验证示例

在仓库根目录安装 Python SDK 并运行：

```shell
uv pip install -e sdks/sandbox/python
export OPEN_SANDBOX_DOMAIN=localhost:8080
export OPEN_SANDBOX_API_KEY=your-api-key
export SANDBOX_VOLUME_NAME=opensandbox-rclone-demo
python examples/rclone-volume-mount/main.py
```

该示例默认使用 `python:3.11`；可以通过 `SANDBOX_IMAGE` 选择其他带
`python3` 的 Linux 镜像。它会写入一个唯一命名的标记文件，检查内容，
删除第一个沙箱，然后从第二个沙箱以只读挂载读取标记文件。它还会验证
通过该挂载写入会因只读文件系统错误而失败。命令失败或内容不匹配会使示例失败，
而不是打印成功消息。

两个沙箱都会被清理，但外部 volume 和标记文件会保留。
脚本会打印标记文件名和预期内容。对于 Docker，请将该文件名代入以下命令，
独立地在远程端验证标记文件：

```shell
sudo rclone --config /var/lib/docker-plugins/rclone/config/rclone.conf \
  cat storage:opensandbox-demo/MARKER_FILENAME
```

通过第二个沙箱读取标记文件演示的是已挂载 volume 的复用，并不一定代表远程持久性：driver 可能仍在提供缓存数据。在删除测试文件或插件之前，请等待待上传的任务完成，并独立验证远程端。参见 rclone 的
[VFS 文件缓存文档](https://rclone.org/commands/rclone_mount/#vfs-file-caching)。

## Kubernetes 配置

请集群运维人员使用你平台支持的 CSI driver 创建 rclone 支持的 PV/PVC。运维人员必须按照该 driver 的文档配置其凭据
或 Secret 引用、远程路径、缓存和挂载选项；这些设置不是 OpenSandbox API 中的字段。

使用[自带 PVC 工作流](/examples/kubernetes-pvc-volume-mount#mode-1-bring-your-own-pvc)：

1. 在沙箱 workload 命名空间中创建 PVC，并确认它在所选 driver 下可用。
   `WaitForFirstConsumer` 的 claim 在 workload 被调度之前可能保持 Pending 状态。
2. 将 `SANDBOX_VOLUME_NAME` 设置为 PVC 名称，并将 SDK 指向 Kubernetes
   OpenSandbox 服务器。
3. 运行同一个验证示例。使用运维人员的凭据在远程端独立检查标记文件，并验证 driver 的清理情况。

对于 Pool 模式，请提前在 Pool pod 模板中挂载 PVC。每个沙箱的
`volumes` 不能与 `extensions.poolRef` 组合使用；上面的示例适用于按需创建的沙箱。参见[静态 pool 存储](/examples/kubernetes-pvc-volume-mount#pool-mode-pre-mount-a-shared-pvc)。

## 运维边界与清理

- 沙箱终止时，OpenSandbox 不会删除预先存在的 volume。
  其配置、凭据和清理由运维人员负责。
- 为每个租户使用专用的远程路径和范围适当的凭据。
  volume 名称不是授权边界。
- 后端语义各有差异：不要假设本地文件系统的锁、原子重命名
  或并发写入的立即可见性。请针对所选后端和缓存设置验证你的工作负载。
- 检查插件/CSI 日志以排查挂载失败、凭据过期或未完成的上传。
  仅凭沙箱就绪并不能证明远程端可写。

在独立确认远程端的标记文件之后，只删除本次运行创建的
标记文件。对于 Docker，请替换下面的 `MARKER_FILENAME`：

```shell
sudo rclone --config /var/lib/docker-plugins/rclone/config/rclone.conf \
  deletefile storage:opensandbox-demo/MARKER_FILENAME
docker volume rm opensandbox-rclone-demo
```

只有在所有使用该 volume 的容器都已停止且待上传的任务完成后，才删除 Docker volume。对于 Kubernetes，请遵循 CSI driver 的清理流程，并在删除 PVC 前检查 PV 回收策略。

## 参考

- [rclone Docker Volume 插件](https://rclone.org/docker/)
- [Docker 命名 volume 示例](/examples/docker-pvc-volume-mount)
- [Kubernetes PVC 示例](/examples/kubernetes-pvc-volume-mount)
- [示例源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/rclone-volume-mount)
