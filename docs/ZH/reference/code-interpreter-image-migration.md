---
title: Code Interpreter 镜像迁移
description: code-interpreter sandbox 环境镜像的迁移指南与仓库拆分详情。
---

# Code Interpreter 镜像迁移指南

Issue：[#1764](https://github.com/opensandbox-group/OpenSandbox/issues/1764)

## 背景

此前，官方 `opensandbox/code-interpreter` 容器镜像的源 Dockerfile、环境配置脚本和构建工具位于 OpenSandbox monorepo 的 `sandboxes/code-interpreter/` 目录下。

为了将平台运行时与 SDK 治理同具体的 sandbox 用户态环境实现解耦，容器镜像的实现已被抽离到独立仓库：

- **新仓库**：[opensandbox-group/sandbox-images](https://github.com/opensandbox-group/sandbox-images)

## 拆分范围

### 迁移至独立仓库

以下组件和制品现在从 [opensandbox-group/sandbox-images](https://github.com/opensandbox-group/sandbox-images) 进行开发、版本控制和发布：

- Dockerfile（`Dockerfile`、`Dockerfile_base`）
- 环境引导与生命周期脚本（`code-interpreter.sh`、`code-interpreter-env.sh`、`jupyter_notebook_config.py`）
- 镜像构建脚本与测试套件（`build.sh`、`tests/test_code_interpreter_node_setup.sh`）
- 独立的镜像发布与 release 工作流

### 保留在 OpenSandbox Monorepo 中

OpenSandbox 继续在 monorepo 中为 code-interpreter 体验提供完整支持：

- **Code Interpreter SDK**：面向 Python、JavaScript/TypeScript、C#/.NET、Kotlin/Java 和 Go 的多语言 SDK（`sdks/sandbox/go/code_interpreter.go`）保留在 `sdks/` 中
- **执行守护进程与契约**：sandbox 内守护进程（`components/execd`）和公共 API 契约（`specs/execd-api.yaml`、`specs/sandbox-lifecycle.yml`）
- **可运行示例与测试**：端到端测试和示例应用（`examples/code-interpreter/`）

## 兼容性与延续性

此次迁移对 monorepo 维护而言纯属结构性调整。容器镜像坐标、固定版本、启动路径和 SDK API 保持完全兼容：

| 属性 | 状态 | 详情 |
|---|---|---|
| **镜像名称** | 不变 | 发布到 `docker.io/opensandbox/code-interpreter`、`ghcr.io/opensandbox-group/opensandbox/code-interpreter` 和 `sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter` |
| **既有标签** | 有效 | 既有的固定标签（如 `v1.1.0`）继续可解析并运行 |
| **启动路径** | 不变 | 入口点 `/opt/code-interpreter/code-interpreter.sh` 保持标准 |
| **环境变量** | 不变 | 通过 `PYTHON_VERSION`、`JAVA_VERSION`、`NODE_VERSION`、`GO_VERSION` 选择语言版本的机制完全一致 |
| **SDK 使用** | 不变 | 使用 `CodeInterpreter` 类的客户端代码无需修改 |

### 签名与来源验证

验证容器镜像签名或构建来源（provenance）的运维人员和用户请注意工作流身份：

- **已有的镜像发布**（如 `v1.1.0` 及更早版本）由 OpenSandbox monorepo 发布，继续使用 OpenSandbox 工作流身份（`publish-components.yml`）。
- 来自 [opensandbox-group/sandbox-images](https://github.com/opensandbox-group/sandbox-images) 的**新发布**使用该仓库的 `release.yml` 工作流身份。
- 需要为新发布验证 Cosign 签名或 provenance 证明的用户，请遵循 [opensandbox-group/sandbox-images](https://github.com/opensandbox-group/sandbox-images) 中的验证文档。

## 参与贡献与报告问题

- **容器镜像问题或需求**：如果你需要在 `code-interpreter` 环境镜像中新增预装软件包、更新语言版本或自定义内核，请在 [opensandbox-group/sandbox-images](https://github.com/opensandbox-group/sandbox-images) 提交 issue 或 pull request。
- **SDK 或生命周期平台问题**：对于 SDK 缺陷、生命周期控制面特性或 `execd` 守护进程问题，请继续向 [opensandbox-group/OpenSandbox](https://github.com/opensandbox-group/OpenSandbox) 贡献。
