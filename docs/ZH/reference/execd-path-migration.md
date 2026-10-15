---
title: Execd 路径迁移
description: execd 安装路径从 /opt/opensandbox/bin/ 变更为 /opt/opensandbox/ 的迁移指南。
---

# Execd 路径迁移指南

PR：[#1014](https://github.com/opensandbox-group/OpenSandbox/pull/1014)

## 背景

OpenSandbox 此前将 execd 和 bootstrap.sh 安装到 `/opt/opensandbox/bin/`，形成了不必要的嵌套。execd 镜像本身已将这两个文件放在根目录：

```
/execd
/bootstrap.sh
```

虽然 bootstrap.sh 默认使用 `EXECD=/opt/opensandbox/execd`（无 `bin/`），但 Kubernetes provider 和 Pool CR 模板历史上一直把文件注入到 `bin/` 子目录。本 PR 将目录结构扁平化：

| 变更前 | 变更后 |
|---|---|
| `/opt/opensandbox/bin/execd` | `/opt/opensandbox/execd` |
| `/opt/opensandbox/bin/bootstrap.sh` | `/opt/opensandbox/bootstrap.sh` |
| `/opt/opensandbox/bin/task-executor` | `/opt/opensandbox/task-executor` |

此外，Docker 运行时现在使用 execd 镜像中的**完整版** `bootstrap.sh`，而不是最小化的 15 行内联生成的 shim。这使 Docker sandbox 获得了与 Kubernetes 相同的能力：MITM CA 信任配置、SIGTERM 转发、pre-script 加载以及链式命令。

## 受影响的用户

| 用户类型 | 影响 |
|---|---|
| **code-interpreter 镜像 `<=v1.0.2`** | **必须升级到 `v1.1.0`** —— `/opt/opensandbox` 处的 emptyDir 会遮蔽 `/opt/opensandbox/` 下的旧脚本 |
| **带自定义模板的 Pool CR** | 必须更新 execd init 容器路径和卷挂载 |
| **自定义 execd 镜像** | 现在必须包含与 `/execd` 并列的 `/bootstrap.sh` |
| **Docker 运行时用户（默认）** | 无需手动操作 —— bootstrap.sh 会自动安装 |

## 迁移步骤

### 1. code-interpreter 镜像

**必须将镜像从 `v1.0.2` 升级到 `v1.1.0`。**

在 code-interpreter `v1.0.2` 中，入口脚本（`code-interpreter.sh`、`code-interpreter-env.sh`）直接位于 `/opt/opensandbox/` 下。在本 PR 之前，emptyDir 卷挂载在 `/opt/opensandbox/bin/`，仅遮蔽该子目录。现在 emptyDir 挂载在 `/opt/opensandbox/`，会遮蔽整个目录 —— 包括那些脚本。Pod 能启动，但找不到自己的入口点。

[PR #1012](https://github.com/opensandbox-group/OpenSandbox/pull/1012) 将所有 code-interpreter 脚本从 `/opt/opensandbox/` 移到了 `/opt/code-interpreter/`，避免了冲突。`v1.1.0` 是包含该变更的首个镜像。

```yaml
# 在你的 Pool CR 或 sandbox 创建请求中：
image: sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0
```

Python SDK（`opensandbox-code-interpreter`）无需升级 —— 它通过网络与 execd 通信，不引用文件系统路径。

### 2. Pool CR 模板

如果你维护着定义了 execd init 容器的自定义 Pool CR，请更新它：

```yaml
# Before
initContainers:
  - name: execd-installer
    image: opensandbox/execd:latest
    args:
      - |
        cp ./execd /opt/opensandbox/bin/execd &&
        cp ./bootstrap.sh /opt/opensandbox/bin/bootstrap.sh &&
        chmod +x /opt/opensandbox/bin/execd &&
        chmod +x /opt/opensandbox/bin/bootstrap.sh
    volumeMounts:
      - name: opensandbox-bin
        mountPath: /opt/opensandbox/bin

# After
initContainers:
  - name: execd-installer
    image: opensandbox/execd:latest
    args:
      - |
        cp ./execd /opt/opensandbox/execd &&
        cp ./bootstrap.sh /opt/opensandbox/bootstrap.sh &&
        chmod +x /opt/opensandbox/execd &&
        chmod +x /opt/opensandbox/bootstrap.sh
    volumeMounts:
      - name: opensandbox-bin
        mountPath: /opt/opensandbox
```

如果使用了 task-executor init 容器，也需要同样的更新：

```yaml
# Before
cp /workspace/server /opt/opensandbox/bin/task-executor && chmod +x /opt/opensandbox/bin/task-executor
# After
cp /workspace/server /opt/opensandbox/task-executor && chmod +x /opt/opensandbox/task-executor
```

以及主容器的环境变量和卷挂载：

```yaml
# Before
- name: EXECD
  value: /opt/opensandbox/bin/execd
volumeMounts:
  - name: opensandbox-bin
    mountPath: /opt/opensandbox/bin

# After
- name: EXECD
  value: /opt/opensandbox/execd
volumeMounts:
  - name: opensandbox-bin
    mountPath: /opt/opensandbox
```

### 3. 自定义 execd 镜像（Docker）

如果你用自定义镜像覆盖 `[runtime].execd_image`，该镜像现在必须**除了** `/execd` 之外还包含 `/bootstrap.sh`。此前只要求 `/execd`，因为 Docker 运行时会生成最小化的内联 bootstrap 脚本。

如果你的自定义 execd 镜像由仅复制 execd 二进制文件的 Dockerfile 构建，请添加 bootstrap.sh：

```dockerfile
COPY bootstrap.sh /bootstrap.sh
```

官方 `opensandbox/execd` 镜像已包含这两个文件，无需更改。

### 4. 自定义入口脚本

如果你在入口脚本或构建流水线中硬编码了 `/opt/opensandbox/bin/execd` 或 `/opt/opensandbox/bin/bootstrap.sh`，请将这些引用更新为 `/opt/opensandbox/`。

## 兼容性

### 旧路径不再可用

此变更之后，路径 `/opt/opensandbox/bin/execd` 和 `/opt/opensandbox/bin/bootstrap.sh` 不再存在。任何引用它们的脚本、工具或模板都会因"找不到文件"而失败。

### 失败的表现形式

| 场景 | 失败模式 |
|---|---|
| 旧 code-interpreter 镜像（`<=v1.0.2`） | `/opt/opensandbox` 处的 emptyDir 遮蔽了 `code-interpreter.sh`；Pod 启动但找不到入口点 |
| 旧 Pool CR 模板 | execd init 容器成功（它复制到自己的挂载点），但主容器找不到 `/opt/opensandbox/bin/bootstrap.sh` |
| 旧自定义 execd 镜像（无 bootstrap.sh） | Docker sandbox 创建失败，报"bootstrap.sh not found in execd image cache" |

### 验证

迁移完成后，确认 sandbox 能正常启动：

```bash
# 在运行中的 sandbox 内：
ls /opt/opensandbox/execd /opt/opensandbox/bootstrap.sh
# 两者都应存在。验证 /opt/opensandbox/bin/ 不存在：
test -d /opt/opensandbox/bin && echo "STALE PATH EXISTS" || echo "OK"
```

## 发布说明

面向发布说明作者：请在其中加入链接到本文档的摘要。

```markdown
### Breaking: execd install path flattened

execd and bootstrap.sh are now installed to `/opt/opensandbox/` instead of
`/opt/opensandbox/bin/`. code-interpreter image users must upgrade to `>=v1.1.0`.
Custom Pool CR templates and custom execd images require updates.

See [execd Path Migration Guide](/reference/execd-path-migration) for full details.
```
