---
title: Helm Chart 重构迁移
description: chart 迁移到 manifests/charts 之下、base chart 拆分以及独立 ingress-gateway chart 的迁移说明。
---

# Helm Chart 重构迁移指南

Feature：[#1857](https://github.com/opensandbox-group/OpenSandbox/pull/1857)

所有 Helm chart 均已迁移到 `manifests/charts/` 之下，并按组件拆分：

| Chart | 目录 | 说明 |
|---|---|---|
| base（新增） | `manifests/charts/base` | CRD + 面向用户的 CRD RBAC，集群作用域 |
| controller | `manifests/charts/controller` | 原为 `kubernetes/charts/opensandbox-controller` |
| server | `manifests/charts/server` | 原为 `kubernetes/charts/opensandbox-server` |
| ingress-gateway（新增） | `manifests/charts/ingress-gateway` | 从 server chart 中抽出 |
| node-agent | `manifests/charts/node-agent` | 原为 `kubernetes/charts/opensandbox-node-agent` |
| opensandbox（伞形） | `manifests/charts/opensandbox` | 聚合以上各项 |

## ⚠️ 升级风险 —— 请先阅读

### 1. 在 `crds.keep=false` 的情况下独立升级 controller 会删除所有自定义资源

CRD 已从 controller chart 中移出。对现有的 controller release 执行 `helm upgrade` 时，Helm 的三方合并发现 CRD 从 release 清单中消失并将其删除 —— 而 Kubernetes 会级联删除**所有** `BatchSandbox`、`Pool` 和 `SandboxSnapshot` 对象。

- 默认安装是安全的：旧 chart 默认的 `crds.keep=true` 会在 CRD 上加上 `helm.sh/resource-policy: keep`，而 Helm 拒绝删除带有该注解的资源。
- **如果你显式设置了 `crds.keep=false`，请勿就地执行 `helm upgrade`。** 改为：先安装 `base`，再升级；或在升级前确认 CRD 带有 `helm.sh/resource-policy: keep` 注解
  （`kubectl get crd batchsandboxes.sandbox.opensandbox.io -o jsonpath='{.metadata.annotations}'`）。

### 2. 伞形用户：server gateway 部署的 values 会被静默忽略

gateway 部署已从 server chart 移至新的 `ingress-gateway` chart。伞形 values 中位于 `opensandbox-server.server.gateway.*` 之下、只影响 gateway 工作负载的键（`image`、`replicaCount`、`port`、`dataplaneNamespace`、`providerType`、`logLevel`、`env`、`resources`、调度/安全相关键）不再被任何模板消费。升级后，gateway 要么保持关闭（沿用之前的默认值），要么以 `ingress-gateway` chart 的默认值运行 —— 你的覆盖值**不会**被迁移。

请在升级前把这些覆盖值移到 `ingress-gateway.*`（伞形 chart）或 `gateway.*`（独立 chart）。只有 `enabled`、`host`、`gatewayRouteMode` 和 `secureAccess` 仍保留在 `opensandbox-server.server.gateway.*`（它们渲染 server 的 `[ingress]` 配置公告）。

## 安装流程变化

- controller chart 不再安装 CRD。独立安装需要先安装 base chart：

  ```bash
  helm install base manifests/charts/base
  helm install opensandbox-controller manifests/charts/controller \
    --namespace opensandbox-system --create-namespace
  ```

- 伞形安装保持不变：`base` 是默认启用的依赖，因此 `helm install opensandbox ...` 仍会在单个 release 中提供 CRD。

## 发布顺序

文档中引用了 `helm/base/<version>/base-<version>.tgz` 发布资产。请在本变更合并后立即发布首个 `helm/base` 标签；在此之前这些链接会 404，独立安装应改用 chart 源码。

## 保持不变

- chart 名称、资源名称、selector 标签，以及配置侧的 `server.gateway.{enabled,host,gatewayRouteMode,secureAccess}` 键均予保留，因此升级无需重命名。
- 伞形 chart 仍在单个 release 中安装全部内容。
