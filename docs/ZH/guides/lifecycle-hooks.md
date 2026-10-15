---
title: 生命周期钩子
description: 在沙箱入口点之前运行设置工作，并在其运行期间调度周期性工作。
---

# 生命周期钩子

生命周期钩子让沙箱在其运行时的定义点执行声明式命令。OpenSandbox 目前支持一个阻塞的 `preStart` 钩子和一个或多个非阻塞的 `periodic` 钩子。

## 受支持的钩子

| 钩子 | 数量 | 运行时机 | 失败行为 |
|---|---:|---|---|
| `preStart` | 零个或一个 | execd 开始监听之后、用户入口点启动之前 | 失败或超时会阻止用户入口点启动 |
| `periodic` | 零个或多个 | 调度器在 `preStart` 成功后启动，并按各自的计划运行每个钩子 | 失败或超时的运行会被记录，后续运行正常继续；如果超时的进程无法被终止，该钩子会被禁用 |

两个钩子都在沙箱内以沙箱环境运行。命令是参数数组，直接执行。要使用管道、重定向或变量展开等 shell 语法，请显式调用 shell，例如 `['sh', '-c', 'command > file']`。

![bootstrap 与 execd-as-init 模式下的生命周期钩子时序](../public/images/lifecycle-hooks-startup.png)

两种启动模式下的顺序相同：

1. execd 启动，其 HTTP 服务器开始监听。
2. 如果配置了，`preStart` 运行至完成。
3. 如果配置了，周期调度器启动。
4. 用户入口点启动。

在 execd-as-init 模式下，execd 保持为 PID 1，并将入口点作为其受监管的子进程启动。在 bootstrap 模式下，bootstrap 进程在 execd 报告生命周期启动完成后启动入口点。

## 配置

生命周期钩子是沙箱创建请求的一部分：

```json
{
  "image": { "uri": "ubuntu:24.04" },
  "entrypoint": ["tail", "-f", "/dev/null"],
  "resourceLimits": {
    "cpu": "1",
    "memory": "1Gi"
  },
  "lifecycle": {
    "preStart": {
      "command": ["sh", "-c", "echo ready > /tmp/prestart.done"],
      "timeoutSeconds": 120
    },
    "periodic": [
      {
        "name": "checkpoint",
        "schedule": "@every 5m",
        "command": ["sh", "-c", "date -u >> /tmp/checkpoints.log"],
        "timeoutSeconds": 120
      }
    ]
  }
}
```

### `preStart`

| 字段 | 必填 | 描述 |
|---|---:|---|
| `command` | 是 | 非空的命令与参数数组 |
| `timeoutSeconds` | 否 | 省略时默认 60 秒；显式值必须在 1 到 10800 秒（3 小时）之间 |

`preStart` 在每次容器启动时运行。让命令保持幂等，以便重试或重启沙箱不会破坏其状态。

### `periodic`

| 字段 | 必填 | 描述 |
|---|---:|---|
| `name` | 是 | 非空名称，在沙箱内唯一 |
| `schedule` | 是 | 标准五字段 cron 表达式或描述符，例如 `@hourly` 或 `@every 30s` |
| `command` | 是 | 非空的命令与参数数组 |
| `timeoutSeconds` | 否 | 省略时默认 60 秒；显式值必须在 1 到 300 秒之间 |

`@every` 间隔必须是整秒且至少一秒。同名钩子的运行绝不重叠：如果上一次运行仍在活跃，下一次计划运行会被跳过。

## 当前可用性

生命周期钩子目前需要 Kubernetes provider。它们会被 Docker provider 和 Fast Sandbox 后端拒绝。请求不能将 `lifecycle` 与 `poolRef` 组合使用。

SDK 在其沙箱创建 API 上暴露生命周期字段，但不强制超时范围。服务器是请求验证的权威。特定语言的示例参见 [SDK 文档](/sdks/)。

## 运维指南

- 保持 `preStart` 有界且确定，因为入口点会等待它。
- 使周期性工作幂等且可安全重试。
- 通过平台支持的密钥机制存储凭证，而不是将它们嵌入命令中。
- 将必须在节点替换后存活的数据写入持久存储。生命周期钩子负责调度恢复和同步工作；它们本身不会使沙箱文件系统持久化。
