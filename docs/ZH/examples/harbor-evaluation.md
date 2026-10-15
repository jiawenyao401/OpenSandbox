---
title: Harbor Evaluation
description: 在 OpenSandbox 上运行 Harbor 智能体评估，每次 trial 配置一个沙箱。
---

# 在 OpenSandbox 上运行 Harbor Evaluation

在 OpenSandbox 基础设施上运行 [Harbor](https://github.com/harbor-framework/harbor) 智能体评估。Harbor 为每次 trial 配置一个 OpenSandbox 容器，在其中运行智能体，执行任务的验证器（verifier），并收集奖励（reward）以及日志和产物（artifacts）。

Harbor 在
[harbor-framework/harbor#2054](https://github.com/harbor-framework/harbor/pull/2054)
中获得了对 OpenSandbox 的原生支持，该 PR 添加了 `opensandbox` 环境后端以及 `harbor[opensandbox]` 安装 extra。

本示例附带一个最小化的、自包含的任务（`hello-opensandbox`），以及一个选择 `opensandbox` 环境的 job 配置。该任务使用预构建镜像和离线验证器，因此无需额外依赖即可在任何地方运行。

## 启动 OpenSandbox 服务器 [本地]

启动本地 OpenSandbox 服务器（Docker 运行时）：

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
opensandbox-server
```

## 安装 Harbor

OpenSandbox 后端已合并进 Harbor 的 `main` 分支
（[#2054](https://github.com/harbor-framework/harbor/pull/2054)），但尚未包含在已发布的版本中。在包含该功能的版本发布之前，请从 git 安装 Harbor：

```shell
uv pip install "harbor[opensandbox] @ git+https://github.com/harbor-framework/harbor"
```

一旦发布的 Harbor 版本包含该后端，直接使用普通 extra 即可：

```shell
uv pip install "harbor[opensandbox]"   # 在 Harbor 下一次发布之后
```

## 配置连接

Harbor 从环境变量（或 `config.yaml` 中的 `domain` / `api_key` kwargs）读取服务器连接信息：

```shell
export OPENSANDBOX_DOMAIN="localhost:8080"   # OpenSandbox 服务器地址
export OPENSANDBOX_API_KEY=""                # API key（如服务器要求）
```

对于本地 / 无认证的服务器，`OPENSANDBOX_API_KEY` 可以留空。

## 运行评估

```shell
# 在 examples/harbor-evaluation 目录下
harbor run -c config.yaml
```

Harbor 使用任务预构建的镜像（`ubuntu:24.04`）创建 OpenSandbox 沙箱，运行 `oracle` 智能体（它会执行参考解决方案），运行验证器，记录奖励，然后销毁沙箱。

## 环境变量

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `OPENSANDBOX_DOMAIN` | `localhost:8080` | OpenSandbox 服务器地址 |
| `OPENSANDBOX_API_KEY` | _（可选）_ | 若服务器要求认证，则提供 API key |

## 输出

结果写入 `config.yaml` 中的 `jobs_dir`
（`jobs/opensandbox/<job-id>/<trial-id>/`），包括 `agent/` 日志、`verifier/`
输出（`reward.txt`、`ctrf.json`）、收集的 `artifacts/` 以及 `results.json`。
成功的运行会产生 `reward = 1.0`。

## 使用你自己的任务

自带的 `hello-opensandbox` 任务签入本仓库，只是为了让示例自包含，无需先发布或下载任何内容即可运行。对于真实评估，建议使用来自外部 Harbor 任务注册中心的任务，并从命令行覆盖示例任务：

```shell
harbor run -c config.yaml --task <org>/<task>@latest
```

在本地任务开发过程中，你也可以让 Harbor 指向一个任务目录：

```shell
harbor run -c config.yaml --path /path/to/task
```

## 评估真实智能体

将 `oracle` 智能体替换为真实智能体 + 模型：

```shell
harbor run -c config.yaml -a claude-code -m "anthropic/claude-sonnet-4-5"
```

## OpenSandbox 特有说明

- **仅支持预构建镜像** — 每个任务必须在 `task.toml` 中设置 `[environment].docker_image`；OpenSandbox 后端不会构建 Dockerfile。私有镜像仓库请使用 `image_auth` kwargs。
- **工作目录** — 设置 `[environment].workdir`（本任务使用 `/app`），以便智能体的相对路径解析到验证器期望的位置。
- **资源** — `cpus` 和 `memory_mb` 映射为沙箱硬性限制；`storage_mb` 会被忽略；GPU 通过 `gpus` / `gpu_types` 支持。
- **产物** — 写入 `/logs/artifacts` 下的文件会被下载到该 trial 的 `artifacts/` 目录。
- **多容器** — 不支持 `docker-compose.yaml` 任务；请使用单容器任务。

## 参考

- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/harbor-evaluation)
- [Harbor](https://github.com/harbor-framework/harbor)
