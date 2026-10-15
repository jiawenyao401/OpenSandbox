<!--
Release notes template (docs/releases/TEMPLATE.md).

Usage: copy this file to docs/releases/X.Y.Z.md, fill it in, remove all
HTML comments, and commit it on the release branch BEFORE triggering
release-umbrella.yml — preflight fails if the file is missing or empty.

Conventions:
- Sections are organized BY COMPONENT; use only the components that
  actually shipped changes in this release, and delete empty sections.
- Within a component, list Features first, then Bug Fixes; reference
  PRs as (#NNNN).
- Mark experimental or unstable items with **[EXPERIMENTAL]** /
  **[UNSTABLE]**.
- Anything that does not belong to a listed component (CLI, node-agent,
  image-committer, docs, CI, deps) goes under Misc.
- The GitHub Release mirrors this file verbatim.
-->

# OpenSandbox X.Y.Z

<!-- One paragraph: the headline of this release and who should upgrade. -->

## 亮点（Highlights）

<!-- 1-3 bullets max. The things a user scanning for 10 seconds must see. -->

-

## Server

<!-- Lifecycle API, proxy, snapshot store, FastPath fleets backend. -->

-

## SDKs

<!-- Python / JavaScript / Kotlin-JVM / C# / Go — group by language when
     a change is language-specific; cross-SDK changes go first. -->

-

## Controller

<!-- BatchSandbox reconciliation, CRD changes, capacity/OTLP metrics.
     Includes the task-executor and image-committer images — they ship
     from the same codebase. -->

-

## Execd

<!-- In-sandbox execution daemon, lifecycle hooks, runtime init. -->

-

## Networking（网络）

<!-- Egress (policy sidecar, TLS interception, credential snapshots)
     and Ingress (gateway routing, endpoints, upstream readiness).
     Prefix entries with egress: / ingress: when the split matters. -->

-

## Fast Sandbox

<!-- fsb-* images: controller, fastlet, fastlet-proxy, janitor,
     firecracker-runtime, sandboxtemplate-builder. -->

-

## 其他（Misc）

<!-- CLI, node-agent, docs, CI, dependencies. -->

-

## 升级与兼容性（Upgrade & Compatibility）

<!-- Everything an operator needs to move from the previous line. -->

- 镜像：`opensandbox/<component>:release-X.Y.Z`（全部三个仓库）
- 软件包：server / CLI / SDKs 均为 `X.Y.Z`
- Chart：从该标签渲染（`helm template ./manifests/charts/opensandbox`）
- Kubernetes：支持范围 vX.Y – vW.Z
- 版本偏差：server ↔ CLI/SDK 同一版本线受支持；±1 个 minor 会告警

## 👥 贡献者

感谢以下贡献者 ❤️

-

## 制品（Artifacts）

<!-- The BOM is the audit record: every image digest of this release,
     one hop from tag -> BOM -> source commit. On GitHub it is attached
     to this release as an asset with the same name. -->

- BOM：[docs/releases/X.Y.Z.yaml](./X.Y.Z.yaml)
- 镜像：`opensandbox/<component>:release-X.Y.Z`（Docker Hub / GHCR / ACR）
- 软件包：PyPI ×5、npm ×2、Maven Central ×5、NuGet ×2 —— 均为 `X.Y.Z`
- Go module：`github.com/alibaba/OpenSandbox/sdks/sandbox/go@vX.Y.Z`

## 安装（Installation）

```bash
# Platform (Kubernetes) — render the chart at this tag and apply
git clone https://github.com/opensandbox-group/OpenSandbox
cd OpenSandbox
git checkout release-X.Y.Z
helm dependency build manifests/charts/opensandbox  # package file:// sub-charts (not committed)
helm template ./manifests/charts/opensandbox | kubectl apply -f -
# or point your GitOps platform (Argo / Flux) at the repo path + tag

# SDKs
pip install opensandbox==X.Y.Z            # Python
npm install @alibaba-group/opensandbox@X.Y.Z   # JavaScript
# Kotlin/JVM: implementation("com.alibaba.opensandbox:sandbox:X.Y.Z")
dotnet add package Alibaba.OpenSandbox --version X.Y.Z
go get github.com/alibaba/OpenSandbox/sdks/sandbox/go@vX.Y.Z

# Run the server locally
uvx opensandbox-server==X.Y.Z
```
