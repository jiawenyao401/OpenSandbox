---
title: 发布验证
description: 针对源码归档、容器镜像、软件包和 Helm chart 验证签名的 OpenSandbox 发布制品。
---

# 发布验证

OpenSandbox 在不改变常规安装命令的前提下对公开发布产物进行签名。验证对于日常使用是可选的，但为需要进行供应链完整性检查的用户提供了支持。

此流程适用于签名工作流引入之后产生的发布版本。更早的发布可能没有证明（attestation）或签名。

## 签名模型

OpenSandbox 使用以下签名路径：

- 源码发布：Generic Release 工作流将显式的 `opensandbox-<tag>.tar.gz` 源码归档和 `SHA256SUMS` 文件上传到 GitHub Release，然后为这两个文件创建 GitHub/Sigstore 来源证明（provenance attestation）。
- 容器镜像：组件与 server 镜像工作流使用 `cosign` 无密钥（keyless）签名对 Docker Hub、GitHub Container Registry（GHCR）和阿里云容器镜像服务（ACR）上的镜像摘要进行签名，并将来源证明发布到这三个仓库。
- Python 与 CLI 软件包：wheel 和源码分发包在 `uv publish` 之前生成证明。
- JavaScript 软件包：工作流运行 `pnpm pack`，为生成的 npm tarball 生成证明，并发布同一 tarball。
- C# 软件包：NuGet `.nupkg` 文件在发布之前生成证明。
- Go SDK 模块：`sdks/sandbox/go/v<version>` 源码发布归档由 Generic Release 工作流生成证明。
- 由当前发布工作流创建的 Helm chart：打包的 chart `.tgz` 文件生成证明，并连同已证明的 `SHA256SUMS` 文件一起发布，以便使用者验证确切的发布字节。
- Java/Kotlin 软件包：Maven Central 上的发布由 Gradle Maven 发布签名配置进行签名。请下载 Maven 制品旁的 `.asc` 签名文件，并使用 OpenPGP 工具进行验证。

当发布操作者配置了本地 git 签名密钥时，发布标签也可以通过 `manifests/release/create-release.sh --sign-tag` 进行签名。不要仅依赖签名标签来验证生成的交付物；请验证你正在安装的制品本身。

## 信任根与密钥

OpenSandbox 的大多数发布签名是由 GitHub Actions OpenID Connect（OIDC）创建的无密钥 Sigstore 签名。这些签名没有长期存在的 OpenSandbox 私钥，因此也没有可下载的项目公钥文件。公钥证书和签名包由 `gh` 或 `cosign` 从 GitHub 的证明服务、OCI 镜像仓库以及 Sigstore 透明性基础设施获取。

预期的身份值：

- Repository：`opensandbox-group/OpenSandbox`
- OIDC issuer：`https://token.actions.githubusercontent.com`

::: warning 旧版工作流身份（仅适用于伞形发布之前的版本）
OpenSandbox 迁移到统一伞形发布时，以下按目标发布的工作流已被移除。对于在此时间点之前发布的制品，它们仍是验证身份。新的伞形发布由 `release-umbrella.yml` 和 `release-packages.yml` 产生。
:::

- 组件镜像工作流：`opensandbox-group/OpenSandbox/.github/workflows/publish-components.yml`
- Server 镜像工作流：`opensandbox-group/OpenSandbox/.github/workflows/publish-server.yml`
- CLI 软件包工作流：`opensandbox-group/OpenSandbox/.github/workflows/publish-cli.yml`
- Python 软件包工作流：`opensandbox-group/OpenSandbox/.github/workflows/publish-python-sdks.yml`
- JavaScript 软件包工作流：`opensandbox-group/OpenSandbox/.github/workflows/publish-js-sdks.yml`
- C# 软件包工作流：`opensandbox-group/OpenSandbox/.github/workflows/publish-csharp-sdks.yml`
- Helm chart 工作流：`opensandbox-group/OpenSandbox/.github/workflows/publish-helm-chart.yml`

为你要验证的发布版本设置仓库身份：

```bash
REPOSITORY="opensandbox-group/OpenSandbox"
WORKFLOW_REPOSITORY="${REPOSITORY}"
WORKFLOW_REPOSITORY_URL="https://github.com/${WORKFLOW_REPOSITORY}"
```

对于在 GitHub 组织迁移之前产生的发布版本，请改用历史身份：

```bash
REPOSITORY="alibaba/OpenSandbox"
WORKFLOW_REPOSITORY="${REPOSITORY}"
WORKFLOW_REPOSITORY_URL="https://github.com/${WORKFLOW_REPOSITORY}"
```

如果你从下游 fork 运行发布工作流，请将验证命令中的 `opensandbox-group/OpenSandbox` 替换为该 fork 的 `owner/repository` 身份。

私有签名材料不会存储在 GitHub Releases、Docker Hub、GHCR、ACR、PyPI、npm、Maven Central、NuGet 或 Helm chart 下载中。Java/Kotlin 的 Maven Central 签名密钥仅保存在 GitHub Actions secrets 中。

## 验证源码发布

> 源码归档仅针对 2026-08 之前的发布版本生成（当时 `release-generic.yml` 工作流被移除）。更新的发布版本没有源码归档；对它们请跳过本节。

先设置发布标签：

```bash
TAG="server/v0.1.13"
SAFE_TAG="${TAG//\//-}"
```

下载已签名的源码归档和校验和文件：

```bash
gh release download "$TAG" \
  --repo "$REPOSITORY" \
  --pattern "opensandbox-${SAFE_TAG}.tar.gz" \
  --pattern "SHA256SUMS"
```

检查归档摘要：

```bash
sha256sum -c SHA256SUMS
```

验证源码归档证明：

```bash
gh attestation verify "opensandbox-${SAFE_TAG}.tar.gz" \
  --repo "$REPOSITORY" \
  --signer-workflow "${WORKFLOW_REPOSITORY}/.github/workflows/release-generic.yml"
```

验证校验和文件证明：

```bash
gh attestation verify SHA256SUMS \
  --repo "$REPOSITORY" \
  --signer-workflow "${WORKFLOW_REPOSITORY}/.github/workflows/release-generic.yml"
```

Generic Release 工作流通过 `workflow_dispatch` 启动，因此其 provenance 的 `source-ref` 是触发工作流时选择的 ref（通常是 `refs/heads/main`），而不是该任务创建的发布标签。

## 验证容器镜像

安装 `cosign` 和 `gh`，然后解析镜像摘要。请始终按摘要进行验证，而不是仅按可变的标签。

发布镜像以相同的组件名和摘要发布到全部三个官方仓库：

| 仓库 | 镜像名模式 |
| --- | --- |
| Docker Hub | `docker.io/opensandbox/<component>` |
| GitHub Container Registry | `ghcr.io/opensandbox-group/opensandbox/<component>` |
| 阿里云容器镜像服务 | `sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/<component>` |

组件可以是 `execd`、`ingress`、`egress`、`controller`、`task-executor`、`image-committer` 或 `nodeagent`。server 镜像使用的组件名为 `server`。

::: tip Code Interpreter 镜像验证
独立的 `code-interpreter` 环境镜像从 [opensandbox-group/sandbox-images](https://github.com/opensandbox-group/sandbox-images) 发布并维护。
- **已有的镜像发布**（如 `v1.1.0` 及更早版本）继续使用 OpenSandbox 工作流身份（`.github/workflows/publish-components.yml`）。
- 来自 [opensandbox-group/sandbox-images](https://github.com/opensandbox-group/sandbox-images) 的**新发布**使用该仓库的 `release.yml` 工作流身份。
需要为新发布验证签名或 provenance 的用户和运维人员，请遵循 [opensandbox-group/sandbox-images](https://github.com/opensandbox-group/sandbox-images) 中的验证文档。
:::

```bash
IMAGE="docker.io/opensandbox/execd"
TAG="v1.0.15"
DIGEST="$(docker buildx imagetools inspect "${IMAGE}:${TAG}" --format '{{.Manifest.Digest}}')"
IMAGE_REF="${IMAGE}@${DIGEST}"
```

验证 cosign 无密钥签名：

```bash
cosign verify "$IMAGE_REF" \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp "^${WORKFLOW_REPOSITORY_URL}/.github/workflows/publish-components.yml@refs/tags/(docker|k8s)/[^/]+/v?[0-9].*$"
```

验证仓库来源证明：

```bash
gh attestation verify "oci://${IMAGE_REF}" \
  --repo "$REPOSITORY" \
  --bundle-from-oci \
  --signer-workflow "${WORKFLOW_REPOSITORY}/.github/workflows/publish-components.yml"
```

对于 server 镜像，使用 `docker.io/opensandbox/server` 和 server 工作流：

```bash
IMAGE="docker.io/opensandbox/server"
TAG="v0.1.13"
DIGEST="$(docker buildx imagetools inspect "${IMAGE}:${TAG}" --format '{{.Manifest.Digest}}')"
IMAGE_REF="${IMAGE}@${DIGEST}"

cosign verify "$IMAGE_REF" \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp "^${WORKFLOW_REPOSITORY_URL}/.github/workflows/publish-server.yml@refs/tags/server/v[0-9].*$"
```

GHCR 和 ACR 镜像使用相同的摘要与身份检查，只需替换为各自的镜像名，例如：

```bash
IMAGE="ghcr.io/opensandbox-group/opensandbox/execd"
# 或
IMAGE="sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/execd"
```

## 验证软件包

从常规的软件包仓库下载包文件，然后对照 OpenSandbox 仓库和预期的发布标签验证证明。

Python 与 CLI 软件包：

```bash
python -m pip download opensandbox-server==0.1.13 --no-deps
gh attestation verify opensandbox_server-0.1.13*.whl \
  --repo "$REPOSITORY" \
  --signer-workflow "${WORKFLOW_REPOSITORY}/.github/workflows/publish-server.yml" \
  --source-ref refs/tags/server/v0.1.13
```

JavaScript 软件包：

```bash
npm pack @alibaba-group/opensandbox@0.1.7
gh attestation verify alibaba-group-opensandbox-0.1.7.tgz \
  --repo "$REPOSITORY" \
  --signer-workflow "${WORKFLOW_REPOSITORY}/.github/workflows/publish-js-sdks.yml" \
  --source-ref refs/tags/js/sandbox/v0.1.7
```

C# 软件包：

```bash
gh attestation verify Alibaba.OpenSandbox.1.0.0.nupkg \
  --repo "$REPOSITORY" \
  --signer-workflow "${WORKFLOW_REPOSITORY}/.github/workflows/publish-csharp-sdks.yml" \
  --source-ref refs/tags/csharp/sandbox/v1.0.0
```

Helm chart：

```bash
HELM_CHART="opensandbox"
HELM_VERSION="<chart-version>"
HELM_TAG="helm/${HELM_CHART}/${HELM_VERSION}"
HELM_PACKAGE="${HELM_CHART}-${HELM_VERSION}.tgz"

gh release download "$HELM_TAG" \
  --repo "$REPOSITORY" \
  --pattern "$HELM_PACKAGE" \
  --pattern SHA256SUMS
sha256sum -c SHA256SUMS
gh attestation verify "$HELM_PACKAGE" \
  --repo "$REPOSITORY" \
  --signer-workflow "${WORKFLOW_REPOSITORY}/.github/workflows/publish-helm-chart.yml" \
  --source-ref "refs/tags/${HELM_TAG}"
gh attestation verify SHA256SUMS \
  --repo "$REPOSITORY" \
  --signer-workflow "${WORKFLOW_REPOSITORY}/.github/workflows/publish-helm-chart.yml" \
  --source-ref "refs/tags/${HELM_TAG}"
```

校验和将下载的包绑定到发布工作流记录的摘要。证明验证该摘要对应的包确实由预期的 OpenSandbox 工作流产生。更改 `HELM_CHART` 和 `HELM_VERSION` 即可验证不同的 Helm 发布。

手动 Helm 发布必须选择确切的既有发布标签作为其触发 ref。因此，与由标签触发的 Helm 发布一样，其 provenance 的 `source-ref` 标识该受保护的 Helm 标签。

### Helm 发布状态

只有当发布工作流将发布摘要所标识的确切打包 `.tgz` 安装到临时 Kind 集群之后，稳定的 `opensandbox` 伞形 chart 才会被标记为 `production-ready`。工作流在运行时门禁与发布之间不会重建 chart。该门禁必须验证：

- controller 与 server 工作负载变为 Ready。
- 所需的 OpenSandbox CRD 已建立并可用。
- 经过认证的 server 访问成功，而缺失或无效的凭证被拒绝。
- 一个 BatchSandbox 完成创建、命令执行和删除的完整生命周期。

此状态仅覆盖发布说明中列出的默认核心配置（profile）。除非另行列出，否则它不为可选的 ingress、egress 策略、快照/暂停恢复、node-agent、升级或多架构链路作担保。

独立的 `opensandbox-controller`、`opensandbox-server` 和 `opensandbox-node-agent` chart 发布仅为 `package-verified`。它们的验证覆盖打包的 chart 本身，不包括稳定伞形发布所具备的集成 Kind 运行时验证。

每个 Helm GitHub Release 在其发布说明中记录了包的 SHA-256 摘要、源 ref 与提交、验证工作流运行、验证配置，以及在经过运行时测试时，所请求的核心镜像引用、仓库 RepoDigests 和镜像 ID。该配置将稳定伞形的 Kind 运行时门禁与仅包验证区分开来。由于 chart 默认值仍以文档中的镜像版本标签为准，`production-ready` 记录的是发布时门禁的结果；它不保证仓库标签在之后永远不会被更改。

Java/Kotlin Maven 制品：

```bash
curl -O https://repo1.maven.org/maven2/com/alibaba/opensandbox/sandbox/1.0.10/sandbox-1.0.10.jar
curl -O https://repo1.maven.org/maven2/com/alibaba/opensandbox/sandbox/1.0.10/sandbox-1.0.10.jar.asc
KEY_ID="$(gpg --list-packets sandbox-1.0.10.jar.asc | awk '/keyid/ { print $NF; exit }')"
gpg --keyserver hkps://keys.openpgp.org --recv-keys "$KEY_ID"
gpg --verify sandbox-1.0.10.jar.asc sandbox-1.0.10.jar
```

如果验证过程无法为早于此流程的发布版本找到证明，请以更新版本的发布作为已签名发布的证据。
