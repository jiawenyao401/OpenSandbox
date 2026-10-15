---
title: 增强提案（OSEPs）
description: OpenSandbox 增强提案索引，涵盖重大特性、架构变更和 API 修改。
---

# OpenSandbox 增强提案

OpenSandbox 增强提案（OpenSandbox Enhancement Proposals，OSEPs）是提出重大特性、架构变更，或对核心 API 与安全模型进行修改的机制。小型缺陷修复和细微改进无需提交 OSEP。

有关如何创建和合并提案，请参阅 [OSEP 贡献指南](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/CONTRIBUTING.md)。

## 提案索引

| OSEP | 标题 | 状态 | 最后更新 |
|:----:|:-----:|:------:|:------------:|
| [OSEP-0001](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0001-fqdn-based-egress-control.md) | 基于 FQDN 的出站流量控制 | implemented | 2026-01-22 |
| [OSEP-0002](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0002-kubernetes-sigs-agent-sandbox-support.md) | kubernetes-sigs/agent-sandbox 支持 | implemented | 2026-01-23 |
| [OSEP-0003](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0003-volume-and-volumebinding-support.md) | Volume 支持 | implementing | 2026-02-11 |
| [OSEP-0004](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0004-secure-container-runtime.md) | 可插拔安全容器运行时支持 | implemented | 2026-07-27 |
| [OSEP-0005](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0005-client-side-sandbox-pool.md) | 客户端 Sandbox 池 | implemented | 2026-07-27 |
| [OSEP-0006](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0006-developer-console.md) | 面向 Sandbox 运维的开发者控制台 | implementable | 2026-03-06 |
| [OSEP-0007](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0007-fast-sandbox-runtime-support.md) | Fast Sandbox 运行时支持 | provisional | 2026-02-08 |
| [OSEP-0008](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0008-pause-resume-rootfs-snapshot.md) | 基于 Rootfs 快照的暂停与恢复 | implementing | 2026-03-13 |
| [OSEP-0009](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0009-auto-renew-sandbox-on-ingress-access.md) | 在 Ingress 访问时自动续期 Sandbox | implemented | 2026-03-23 |
| [OSEP-0010](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0010-opentelemetry-instrumentation.md) | OpenTelemetry 指标与日志（execd、egress 与 ingress） | implementing | 2026-04-12 |
| [OSEP-0011](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0011-secure-access-endpoint.md) | GetEndpoint 上的安全访问与签名 Endpoint | implemented | 2026-04-25 |
| [OSEP-0012](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0012-credential-vault.md) | Credential Vault 与凭证代理 | implemented | 2026-06-23 |
| [OSEP-0013](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0013-isolated-execution-api.md) | 隔离执行 API | implementing | 2026-06-23 |
| [OSEP-0014](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0014-multi-tenancy.md) | Kubernetes 运行时的多租户支持 | implemented | 2026-07-27 |
| [OSEP-0015](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0015-pod-snapshot.md) | 基于 Spec 的 Pod 快照暂停与恢复 | draft | 2026-06-27 |
| [OSEP-0016](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0016-unified-umbrella-release-governance.md) | 统一伞形发布治理 | implemented | 2026-09-21 |
| [OSEP-0017](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0017-resilient-sdk-transport.md) | 高弹性 SDK 传输层 | implementing | 2026-07-22 |
| [OSEP-0019](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0019-node-agent-sandbox-collection.md) | 用于节点级 Sandbox 采集的 Node Agent | implementing | 2026-08-11 |

## 状态定义

| 状态 | 含义 |
|--------|---------|
| **draft** | 提案处于初始开发阶段，尚未被接受 |
| **provisional** | 提案在原则上已被接受，设计仍可能演进 |
| **implementable** | 设计已定稿，可以开始实施 |
| **implementing** | 实施进行中 |
| **implemented** | 已完整实现并可用 |

## 提交提案

::: tip
在撰写完整的 OSEP 之前，建议先创建一个 GitHub Issue 或 Discussion，以了解社区的兴趣并获得对你想法的早期反馈。
:::

要提交新的 OSEP：

1. Fork 该仓库
2. 复制 OSEP 模板
3. 填写提案详情
4. 提交 pull request

完整流程请参阅 [OSEP 贡献指南](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/CONTRIBUTING.md)。
