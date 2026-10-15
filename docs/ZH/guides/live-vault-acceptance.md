---
title: Live Vault 验收（实验性）
description: OSEP-0023 实验性实时凭证绑定 Vault 循环在 egress sidecar 中的任务规格、计划与验收证据矩阵。
---

# OSEP-0023 实时凭证绑定 Vault：验收

::: warning 内部任务文档
本页将当前 Policy revision 验收与早于 PR #2176 的历史 live-Vault 证据分开。更早的 `fresh` 或 `pass` 结果不能验证 Policy 集成，也不能确立一次通过的当前 CI 运行。
运行时保持默认关闭，且不作为公共部署模式受支持。
:::

## Policy revision 集成验收

- 分支：`feat/osep0023-policy-revision`。
- 实现起始提交：`1adfcecec408c2732071befb16f2412463bc3315`。
- 设计基线：`12b77ddefdeb7b35cd933bad18cf253e2c02fcbf`（PR #2176 合并）。
- 范围：现有 Policy API 处理器和自动 always-rule 重载加入共享的 Policy/Vault 变更屏障。后续 Vault 更新和引导保留已提交的内部 Policy epoch。不添加公共方法、模式字段、受支持模式、HTTP/2 或 Fast Sandbox 支持。

成功要求可选的用户策略文件持久化、nft 应用、精确的接收器确认，然后是 Go proxy/base 发布。重载省略用户策略持久化。这不是原子的预 ACK 可见性：nft 可能先影响流量。nft 成功后的接收器失败需要粘性恢复和隔离，即使是明确的拒绝。API 成功等待发布和旧请求截止注册，而不是默认的 30 秒 drain。到期会关闭整个连接，可能影响较新的请求；重复的 revision 不会延长最早的旧请求截止时间。

以下各行将已执行的检查与被阻塞的验收层分开。选定的单元测试通过不是完整套件或端到端验收。下方的命令摘要适用于当前实现工作树；最终提交身份应记录在相应的验证交接中。

| ID | 所需证据 | 当前结果 |
|---|---|---|
| P1 | Go owner：Policy 方法/重置、空操作、绑定预检、精确候选/身份、文件/nft 失败、取消、串行重载和生命周期清理 | 新鲜通过：选定的 owner 单元和聚焦竞态检查，包括权威 Policy 遮蔽和所有三条关闭路径。更广的允许选择：18 个包，594 个顶层通过；排除与跳过详见下文 |
| P2 | 真实 Go → Unix → Python：Policy 与重载确认、丢失 ACK 的对账、Policy → Vault epoch 连续性、引导围栏 | 阻塞：此环境中 AF_UNIX 套接字创建返回 EPERM |
| P3 | 真实 mitmdump：仅策略 keepalive、仍被覆盖的空闲复用、长请求/升级退役、最早截止与整连接关闭 | 注册表单元：69/69 通过。真实运行时阻塞：AF_UNIX EPERM 导致 14 个设置失败；一个负向启动检查通过，不是聚合的运行时通过 |
| P4 | 特权 Linux 文件/nft/隔离：不变的启动失败、已提交效果后接收器拒绝/未知、实际包围栏 | 阻塞：nft 和所需能力不可用；unshare 返回 EPERM |
| P5 | 当前检出镜像：Policy 方法、绑定拒绝的拒绝、真实周期 always 重载、默认 30 秒 drain、先 Policy 后 Vault 轮换；记录镜像身份与所拥有资源清理 | 运行器单元：33/33 通过。镜像运行未通过前提检查（退出 1，零跳过）：Docker/nft 缺失且 unshare 返回 EPERM |
| P6 | 现有 Go SDK PatchPolicy/DeletePolicy 200 结构与 503 传播；窄化规范生成；docs-site 构建 | 新鲜通过：Go SDK 完整套件和聚焦 Policy 用例；docs-site 构建。Python/TypeScript 生成仅限于已记录的响应；两个 Python 处理器均通过 200/503 响应检查，且生成的 TypeScript 契约通过严格类型检查 |

当前检查（2026-10-11 UTC）在起始提交加 Policy 集成工作树上运行：

- Egress Go 测试使用了包范围的 `go test -json` 选择，明确排除了 64 个命名的 Unix 套接字或特权测试。允许的选择在 18 个包中通过，共 594 个顶层测试通过，四个特权测试被跳过，两个 ACL 子测试被跳过。这**不是完整的 egress 套件通过**。
- 选定的 owner 单元和 `go test -race` 检查通过，包括权威 Policy 遮蔽回归和 Policy API、always-rule 重载及 Vault 关闭路径。更广的聚焦竞态选择通过 75 个顶层测试，零跳过或失败；它不覆盖被阻塞的 IPC 或特权路径。
- `go vet ./...` 和 `go build .` 在 `GOFLAGS=-buildvcs=false` 下通过，因为环境无法执行 VCS 打标子进程。Go 依赖检查使用预置的模块缓存，`GOPROXY=off` 且 `GOSUMDB=off`。
- Python 发现 363 个测试；53 个运行时/Unix 套接字用例被明确排除在纯 Python 选择之外。全部 310 个选定测试通过，零跳过，包括 69 个注册表测试。这**不是完整的发现验收**，不能替代真实 mitmdump 证据。
- `components/egress` 下的 `python -m unittest discover -v -s scripts -p 'test_*.py'` 通过 85 个测试，包括 33 个镜像运行器单元测试。
- `cd sdks/sandbox/go && go test ./...`：通过；主 SDK 包和 `poolredis` 包（无测试）。
- `go test -v -run 'Test(PatchPolicy|DeletePolicy|PolicyMutationServiceUnavailable)$'`：通过，包括单独的 PATCH 和 DELETE 503 用例。
- 使用 `openapi-python-client 0.28.0` 和 `openapi-typescript 7.13.0` 的仅 egress 重新生成：基线输出与检入文件匹配；更新后的输出只更改两个 Python Policy 处理器和 TypeScript Policy 响应类型/描述。公共模式和方法未变。
- 生成的 Python `sync_detailed` 处理器：PATCH 和 DELETE 的模拟 HTTP 200 和纯文本 503 响应通过，包括严格的意外状态模式。
- TypeScript 5.9.3：生成的 Policy 200/503 契约的严格 no-emit 检查通过。这不是完整的 Python 或 JavaScript SDK 套件结果。
- `cd docs && pnpm docs:build`：在 Node 22.23.3 和 pnpm 9.15.0 下通过；VitePress 报告一个包大小警告，零构建错误。

镜像运行器包括 `TestPolicyRevisionAPIMutations`、`TestPolicyRevisionRejectsBoundDeny`、`TestAlwaysReloadRevision`、`TestPolicyOnlyDrain` 和 `TestPolicyThenVaultMutation`。Policy drain 用例使用生产 30 秒默认值而无覆盖；always-rule 用例等待实际的一分钟重载作业。它们的存在和运行器单元覆盖在前提失败时不能确立镜像验收。此环境中未构建任何镜像，也没有运行任何真实镜像用例。真实 IPC、mitmdump、nft/数据包和镜像验收仍被阻塞；不声称有任何完整验收或当前 GitHub CI 通过。

单元测试和环回运行时测试都不能替代镜像/数据包证据。缺失的 Docker、nft、命名空间权限或依赖必须报告为阻塞，而不是通过性跳过。镜像证据必须标识实际提交/树、脏差异指纹、镜像 ID、执行的用例、返回码和清理结果。不要暴露凭证或添加公共 epoch 调试 API。

## 历史 live-Vault 证据（PR #2176）

其余记录保留自更早的 live-Vault 工作。其分支、基线、会话相关证据状态和推迟的检查属于那项工作，不是上述 Policy revision 集成的状态。

- 分支：`feat/osep0023-live-vault`
- 验证基线：`f46ff91ef572589daeeb67b453ea4e8714a68a26` 是干净的起始提交。本地检查针对该基线加修复工作树运行，而不是仅针对未变更的基线。
- 保留的实现基座：`447a7be3bd3c0ee0ac2e9da5412c281aa44b1ca9`。本发布不会 rebase 到上游 main；PR 创建和基座选择仍由用户决定。
- Owner：当前 Devin 会话（实现 + 聚焦测试）。独立评审在此上下文中不可用 —— **阻塞**；对完整 diff 的 lead 评审已完成，不算独立。
- 写入集：`components/egress/**`、`docs/guides/credential-vault.md`、`docs/guides/live-vault-acceptance.md`、`components/egress/docs/live-vault-acceptance.md`、`oseps/0023-credential-bound-tls-interception.md`。
- 历史证据注记：更早的运行报告"18 Go packages ok"、"309 Python tests, 0 skips"、"6/6 image cases PASS" —— 这些是自我报告且未验证的；在本会话的新鲜日志落入证据目录之前，不声称任何当前通过。

### 任务规格

在实验性实时凭证绑定循环可以被信任之前对其进行加固：

1. 未知的变更结果必须锁定粘性恢复，而不是从可能陈旧的先前状态重启（AC1）。
2. 无论实时准入包在哪里激活，`ssl_insecure` 都必须被拒绝（AC2）。
3. 必须从实际扩展列表检测携带 ECH 的 ClientHello 并不透明地透传；无法检查的数据被拒绝（AC3）。
4. 解密的 HTTP/1 请求必须证明其 authority 等于被准入的 SNI，然后才能注入任何凭证（AC4）。
5. 真实的 Vault 生命周期（轮换、keepalive、陈旧 revision、移除/重加）在真实运行时下成立（AC5）。
6. Drain 必须分页所有过期传输、重试失败的关闭，并以记录的结果清理所有运行器创建的资源（AC6）。
7. 遗留的仅安装行为保持不变（AC7）。

### 计划

- Go：在 `mutateRevisionVault` 的每次终止分离前锁定 `requireRevisionRecoveryLocked(ExternalEffectsUnknown)`；在 `validateRevisionIPCConfig` 中拒绝 `LiveAdmission && ssl_insecure`。
- Python 插件：从 `ClientHello.extensions` 检测 ECH、严格的 H1 authority 验证、load/configure/每请求中的 ssl_insecure 围栏、基于游标的 drain 分页。
- 测试框架：live-vault 镜像运行器记录每资源清理结果和脱敏的证据模式；真实 mitmproxy 和原始线缆测试提供边界证据。
- 验证：聚焦的离线 Go 测试、聚焦的 Python 单元测试、真实环回 mitmdump 测试；仅在显式授权后进行 Docker 镜像测试。

### 验收标准与证据矩阵

证据状态按 AC 跟踪：`fresh` = 由本会话的运行产生，`reused` = 已有的未变更覆盖，`missing` = 尚未运行，`blocked` = 需要此上下文缺少的权限或资源。仅对本会话实际执行的检查记录通过/失败。

| AC | 要求 | 证据层 | 证据状态 | 结果 |
|----|-------------|-----------------|----------------|--------|
| AC1 | 不确定的生命周期：每次终止分离都锁定恢复；新引导、陈旧票据和 watcher 重启被拒绝；旧凭证绝不重新授权；已知的拒绝/取消/关闭路径非粘性 | Go 单元 + Python-IPC 集成测试（`TestRevision*`） | fresh | 通过 —— `go test ./...` 退出 0，18 个包（日志 `rework-20261010/go-test-all.txt`）；Go 测试用例跳过数未记录（非 verbose） |
| AC2 | 证书/安全配置：在 Go 启动验证和真实 mitmdump 启动中拒绝 live bundle + `ssl_insecure`；安全设置和遗留配置被接受；lazy 握手验证上游 | Go 环境测试、真实 mitmdump 启动测试 | fresh | 通过 —— 真实 mitmdump 下 3/3 `LiveSslInsecureStartupTest` |
| AC3 | ECH 不透明：从真实扩展列表检测扩展 0xfe0d（含 GREASE）；不透明的 ignore_connection、无 token、零凭证可见性；格式错误的数据被拒绝 | 由真实 `mitmproxy.tls.ClientHello` 解析的子进程隔离原始线缆测试 + 真实 mitmdump 边界测试 | fresh | 通过 —— 2/2 `test_ech_wire`，包括真实 mitmdump 的源与 mitm 证书对比 |
| AC4 | H1 authority：恰好一个严格 Host 等于 SNI 和准入 token；绝对目标一致；CONNECT/重复/缺失/格式错误/H2 在注入前被拒绝 | 单元 + 子进程隔离的真实 `read_request_head` 测试 | fresh（单元/解析器）；推迟（Docker 原始 H1 输入、镜像层） | 通过 —— 17 个真实解析器 worker 用例；镜像层推迟 |
| AC5 | 真实 Vault 生命周期：keepalive 上的轮换、陈旧 expectedRevision 冲突、移除围栏、移除/重加 | 真实 mitmdump 运行时测试；Docker 镜像运行 | fresh（环回）；推迟（镜像运行、用户决定） | 通过（环回）—— 344 个 Python 测试，包括 6 个运行时用例，0 跳过；Vault HTTP API 的陈旧/移除重加在真实 Docker 中仍推迟 |
| AC6 | 空闲/多页 drain：after_serial 分页、失败关闭重试、无饥饿、无截止延长、完整清理计量 | 单元测试 + 真实 mitmdump 空闲关闭环回测试 + 运行器单元测试；镜像空闲 drain 用例 | fresh（单元+环回）；推迟（镜像层） | 通过 —— 空闲传输仅在截止时关闭；运行器模拟清理单元通过 |
| AC7 | 遗留不变：仅安装接收器、拉取式 vault 路径和非 live 环境保持现有行为 | 保留现有的绿色测试 | fresh | 通过 —— 包含在 344 测试发现中 |

Docker 镜像层（真实的透明 443 端口拦截、容器生命周期）**因用户决定而推迟**：镜像验收推迟到 GitHub CI 或后续会话。它尚未运行，没有 CI 通过，且此推迟不是验收降级 —— 代码已准备就绪，一旦授权镜像套件即可原样运行。docs-site 构建也保持推迟（离线依赖存储不完整）。

### 证据层

单元测试不能替代真实运行时运行，任何环境跳过都不会被报告为通过。本会话的新鲜逐命令日志写入会话证据目录（`/tmp/live-vault-evidence-final/`）之下，每次运行一个子目录，并从交接报告中链接而不是复制到此页面。

- Go 单元和跨语言 IPC 集成：`cd components/egress && umask 022 && GOPROXY=off GOSUMDB=off go test . -run 'TestRevision(VaultMutation|WriteHandler|Recovery)'` 和 `go test ./pkg/mitmproxy ./pkg/revisionruntime`。
- Python 插件单元测试：在预置的 mitmproxy 11.0.2 venv 下运行 `python -m unittest discover -v -s tests -p "test_*.py"`。
- 真实 mitmdump 运行时测试（仅环回、一次性）：`test_credential_bound_runtime.py`、`test_mitmproxy_runtime.py`、ECH/H1 线缆测试。
- 真实 Docker 镜像端到端：`components/egress/scripts/test_live_vault_image.py --image <egress image> --artifacts <evidence dir>` —— **因用户决定推迟**（到 GitHub CI 或后续会话）；本会话未运行。

### 范围排除

- HTTP/2 GOAWAY/流 drain、fast-sandbox 主体配置和跨完整 sidecar 替换的恢复仍在范围之外。动态 Policy 和 always-rule 重载参与被排除在此历史切片之外；它们在上文有单独的验收矩阵。
- 动态 DNS 学习和 Kubernetes 部署未被本实验当前证据验证；Docker 网络运行器固定源地址并使用静态允许规则（本地 Docker 内嵌 DNS 是运行器的便利，与运行时 DNS 或 Kubernetes 证据无关）。
- Dockerfile 构建旋钮 `GOPROXY` 和 `APT_MIRROR` 保持可选且默认为空；它们仅是受限构建网络的本地镜像便利。覆盖设置错误会使构建失败；`APT_MIRROR` 还要求至少一个受支持的 APT 源文件。

### 证据卫生

测试输出和本文档不包含凭证值：fixture 掩蔽注入的头，源仅记录头名称，运行器在记录的命令中对已知的 fixture 密钥进行脱敏，运行时失败通过固定的错误类报告。

### 提交身份重写

用户仅授权重写四个 live-Vault 切片提交。其作者和提交者现在是 `hpliStartAgain`，使用已验证账号的 noreply 地址 `118508408+hpliStartAgain@users.noreply.github.com`。每个提交保留其原始树、时间戳和 Claude co-author 尾注。证据提交的消息现在引用重写后的实现提交。

| 原始 SHA | 重写后 SHA |
|---|---|
| `a90a41330029fb646be5c63de86b3d4d8f4ac40a` | `362e7136f14621e5015ab7447b9fb7eff19feeae` |
| `a6354dea8f8df4c523fa68b146e1b8d2e4df9341` | `6c6619d38b00b6672680ac4ec69a430c7959e1a9` |
| `205d6c8601997415a34b3159b6a7e027dffe8559` | `87a3133b94202480d737057bc3043684ab24c57a` |
| `f46ff91ef572589daeeb67b453ea4e8714a68a26` | `437ff10c43cf830bd3cf91e762f33b78cd1a45ad` |

旧的基线引用标识原始验证历史。安全修复是一个单独的子提交，不属于此仅元数据的重写。本地备份 ref 保留旧的已发布 tip。
