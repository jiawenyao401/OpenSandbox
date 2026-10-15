---
title: 发布自动化
description: OpenSandbox 的发布机制 —— 一个伞形发布、一个标签，覆盖所有制品。
---

# 发布自动化

OpenSandbox 以**统一伞形发布（umbrella release）**（OSEP-0016）的形式交付：一次发布中的每个制品 —— 平台镜像、Helm chart（仓库内交付）、CLI 以及所有 SDK —— 都携带相同的 `X.Y.Z`，由单一工作流驱动。按组件和按 SDK 的发布入口已被移除；伞形发布是唯一的发布途径。命名与节奏参见[版本控制](/community/versioning)。

## 发布准备（手动，在发布分支上）

```bash
# 1. 一条命令完成版本提升：chart 版本、镜像引用、SDK 版本和依赖范围
#    在一次提交中统一改写为目标版本
manifests/release/create-umbrella-release.sh --version 1.1.0 --bump-only

# 2. 手工撰写发布说明（复制 docs/releases/TEMPLATE.md），然后推送
cp docs/releases/TEMPLATE.md docs/releases/1.1.0.md
$EDITOR docs/releases/1.1.0.md
git push
```

## 预发布版本（rc）

rc 始终基于**该版本线的诞生版本**发布（`X.Y.0-rc.N` —— `Z > 0` 时不存在 `X.Y.Z-rc`）。它用于验证平台可部署且扇出流程可运行：镜像会被发布，**所有软件包被暂扣（不发布）**，且只生成伞形标签（在 GitHub 上标记为 `--prerelease`）。rc 运行无需审批：没有 `release` 环境门禁，BOM PR 也无需等待评审即可合并（如果发布分支的 ruleset 仍强制要求评审，为发布机器人添加一条 ruleset 旁路 —— 工作流会快速失败而不是等待）。

```bash
# 1. 版本提升（channel=rc 由后缀推导得出）
manifests/release/create-umbrella-release.sh --version 1.1.0-rc.1 --bump-only

# 2. 撰写发布说明，然后推送
cp docs/releases/TEMPLATE.md docs/releases/1.1.0-rc.1.md
$EDITOR docs/releases/1.1.0-rc.1.md
git push

# 3. 触发工作流
gh workflow run release-umbrella.yml \
  --ref release-1.1 \
  -f version=1.1.0-rc.1 -f dry_run=false
```

随着修复落地，可以使用 `-rc.2`、`-rc.3`……重复上述步骤 —— 每个 rc 都是一次完整重建，之前的 rc 制品在仓库中保持冻结。该版本线的稳定版本（`1.1.0`，无后缀）随后会真正发布软件包。预发布版本默认不会被解析（PyPI / NuGet / Maven / Go 会隐藏它们；npm 则以 `rc` dist-tag 发布）。

## 发布执行（工作流）

触发 `.github/workflows/release-umbrella.yml`（`dry_run=false`、`version`、`channel`、`release_branch`）。稳定发布需要由一位非触发者的发布审批人批准 `release` 环境；rc 发布和 dry run 无需审批。工作流随后执行：

| 阶段 | 发生的事情 |
|---|---|
| preflight | 提交可达性检查 + 发布说明存在性检查（`docs/releases/<version>.md`） |
| scan | 版本一致性扫描（阻断发布） |
| build | 构建 13 个镜像并直接推送到 `release-X.Y.Z`（在 `UMBRELLA_PUBLISH_ENABLED` 门禁关闭期间使用 run 作用域的 staging 标签）；构建并暂扣所有软件包 |
| BOM | 将摘要固定写入 `docs/releases/<version>.yaml`，推送到 `release/<version>` 分支；工作流向发布分支发起 PR 并自行合并（`C_bom`）—— 稳定发布等待 code-owner 批准（分支 ruleset 要求），rc 立即合并 |
| publish | BOM PR 合并后（所有镜像变绿 + code-owner 批准），各仓库并行发布软件包（PyPI、npm、NuGet、Maven Central）；镜像在 build 阶段已以 `release-X.Y.Z` 公开 |
| tag | 在 `C_bom` 上生成 `release-X.Y.Z` + `sdks/sandbox/go/vX.Y.Z` 标签，并附带发布说明和 BOM 创建 GitHub Release |

发布失败绝不会生成对应的 git 标签或 GitHub Release，因此发布信息流不会出现不完整的发布；在发布窗口之间可以触发 dry run（`dry_run=true`）以保持扇出流程的健康。

### Dry run

`dry_run=true` 会演练除发布之外的一切：镜像在**本地**构建（单架构、`--load` —— 不推送仓库，无需凭证），软件包被构建并作为工作流制品暂扣，BOM 通过 BOM PR 落在发布分支上（需要一次 code-owner 批准，与保护分支上的任何变更一样），并以本地镜像 ID 代替仓库摘要，同时不生成任何 git 标签。要在 fork 中演练发布，请准备好发布分支（版本提升 + 发布说明），并以 `dry_run=true` 触发工作流 —— 该 fork 中无需存在 `UMBRELLA_PUBLISH_ENABLED` 变量。fork 必须启用 **Allow GitHub Actions to create and approve pull requests**（Settings → Actions → General → Workflow permissions）；仅靠工作流中的 `pull-requests: write` 无法覆盖该设置的关闭状态。

## 发布制品

- `docs/releases/<version>.yaml` —— BOM，镜像摘要的权威来源
- `docs/releases/<version>.md` —— 手工撰写的发布说明（同步镜像到 GitHub Release）
- 容器镜像：Docker Hub、GHCR 和 ACR 上的 `opensandbox/<component>:release-X.Y.Z`
- 软件包：PyPI（server、CLI、3 个 SDK）、npm（2 个）、Maven Central（5 个 Kotlin/JVM 坐标）、NuGet（2 个）—— 全部为伞形版本
- Helm chart：**不发布**；它们在发布标签处以仓库内形式交付

## 发布操作手册

### 发布与回滚

各仓库链路（PyPI、npm、NuGet、Maven Central）并行且相互独立地运行 —— 不存在跨生态系统的顺序依赖。发布步骤是幂等的：已在某仓库上可见的版本会被跳过，因此重跑永远不会与仓库的不可变性冲突（已发布的 Maven 版本不可变；`sandbox-bom` POM 出现在 Central 上即为跳过标记）。上传成功后不会重新对照公共索引验证 —— 各仓库的 simple index 可能比上传滞后数分钟。发生故障时，请对已发布的链路采取补救措施：

| 生态系统 | 撤销方式 |
|---|---|
| Maven Central | 若其链路失败则不会发布；发布后不可变 —— 仅能弃用（deprecate） |
| NuGet | `dotnet nuget delete <pkg> <version>`（72 小时内），否则取消列出（unlist） |
| npm | `npm unpublish <pkg>@<version>`（72 小时内），否则 `npm deprecate` |
| PyPI | 通过项目设置手动 yank（无可脚本化的 API） |
| 镜像 | `crane delete <image>:release-X.Y.Z` |

在采取补救措施之前，务必附带运行链接提交 P0 发布事故。全局镜像缓存可能仍保留被撤销的制品；撤销仅在上游具有权威性。

### N-1 紧急回移（CVSS ≥ 8.0，≤ 72 小时，每个版本线一次性）

1. 将修复合入 `main`（标准 PR 评审）。
2. `git checkout release-X.(Y-1) && git cherry-pick <sha> && git push`。
3. 以 `version=X.(Y-1).Z`、`release_branch=release-X.(Y-1)`、`dry_run=false` 触发 `release-umbrella.yml`。完整扇出流程会重新运行；发布说明必须链接 CVE 并指明当前版本线中对应的等价快照。

任何不符合条件的 CVE 都要等待下一个版本线诞生。

### Staging 清理

run 作用域的 staging 镜像标签（`staging-<sha>-<runid>`）仅在某次运行不发布 release 标签时存在（dry run，或 `UMBRELLA_PUBLISH_ENABLED` 门禁仍关闭时的真实运行）。它们有独立的命名空间，永远不会被面向用户的界面引用；一个定时任务会删除其运行未在 14 天内完成的 staging 标签。被暂扣的软件包制品使用默认的 14 天工作流制品保留期。

## 旧版发布

历史上的按组件标签（`server/v0.2.3`、`docker/execd/v1.1.0`……）已被冻结并永久保持可解析。这些旧制品的验证说明（及其工作流身份）见[发布验证](/community/release-verification)。
