---
title: 版本控制与发布
description: OpenSandbox 的统一伞形版本控制 —— 每个镜像、chart、CLI 和 SDK 共享同一个版本号。
---

# 版本控制与发布

OpenSandbox 以**统一伞形发布（umbrella release）**的形式交付：一次发布中的每个制品 —— server 与组件镜像、Kubernetes controller 与 task-executor 镜像、Helm chart、CLI 以及所有 SDK —— 都携带**相同的 `X.Y.Z`**，从单一的发布提交切出，并固定在一份签名的 BOM 中。完整设计见 [OSEP-0016](https://github.com/opensandbox-group/OpenSandbox/blob/main/oseps/0016-unified-umbrella-release-governance.md)。

::: warning 推进状态
伞形发布工具正在分阶段推进（dry-run → rc → GA，参见该 OSEP 的 Migration 部分）。在 GA 之前，按组件标签仍是实际生效的发布机制（[发布自动化](/community/release-automation)）。届时历史上的按组件标签将被冻结 —— 永不删除、永不扩展 —— 并永久保持可解析。
:::

## 首个版本：`1.1.0`

首个伞形发布为 **`release-1.1.0`**，同时充当 GA 宣告。`1.0.0` 被刻意跳过：Maven Central（`com.alibaba.opensandbox:sandbox` 已用到 `1.0.19`）和 Go module proxy（`sdks/sandbox/go` 已用到 `v1.0.5`）都已消耗掉这些版本，且两者均不可变。伞形发布从每个已被消耗版本之上的最低新**版本线**（`X.Y.0`）开始 —— `1.0.20` 虽能避开仓库中的既有版本，但不是有效的版本线诞生（`Z > 0` 被保留给线内快照），因此定为 `1.1.0`。

## 命名规则

| 制品 | 格式（`1.4.0` 时的示例） |
|---|---|
| Git 标签 | `release-1.4.0` |
| 容器镜像 | `opensandbox/{server,execd,ingress,egress,image-committer,controller,task-executor}:release-1.4.0` |
| Go SDK（VCS 标签） | `sdks/sandbox/go/v1.4.0`、`sdks/sandbox/go/poolredis/v1.4.0` —— 与伞形标签同属一个提交 |
| PyPI 上的 CLI / SDK / server | 裸版本 `1.4.0` |

Git 标签与镜像标签共享相同的 `release-` 字符串：`git checkout release-1.4.0` 检出的代码与你拉取的镜像属于同一发布。软件包仓库使用裸的 semver 核心，因为它们拒绝前缀。

## 覆盖范围

- **覆盖**：平台运行时镜像（server、execd、ingress、egress、image-committer、controller、task-executor；fast-sandbox 家族以 `opensandbox/fsb-*` 发布）、CLI 以及所有已发布的 SDK（Python、JavaScript、Kotlin/JVM、.NET、Go）。
- **Helm chart 不发布。** chart 存放在仓库内，在发布标签处确定版本；请自行渲染并部署：

  ```
  git checkout release-1.1.0
  helm template ./manifests/charts/opensandbox | kubectl apply -f -
  ```

  GitOps 平台可以直接指向仓库路径与标签。
- **不覆盖**：诸如 `opensandbox/code-interpreter` 之类的 sandbox 模板镜像。它们由用户在创建 sandbox 时选择，并在 [opensandbox-group/sandbox-images](https://github.com/opensandbox-group/sandbox-images) 中独立进行版本控制。

## 节奏与支持

- 每两周诞生一条版本线（`X.Y.0`）；线内快照（`X.Y.Z`，`Z > 0`）按需发布。预发布版本形如 `X.Y.0-rc.N`，仅发布镜像 —— 软件包在该版本线的稳定发布之前被暂扣。
- **仅支持最新版本线，无 LTS。** 当 `X.(Y+1).0` 发布时，`X.Y.*` 即刻 EOL，唯一的例外是紧急 CVE 窗口（CVSS ≥ 8.0，披露后 ≤ 72 小时，一次性的 `X.(Y-1).Z` 快照）。
- 没有按组件的热修复：回移是一次在新 `X.Y.Z` 处的完整伞形重建。

要验证你安装的发布版本，参见[发布验证](/community/release-verification)。
