---
title: Credential Vault（凭证保险库）
description: 为沙箱出站请求安全注入凭证，无需向工作负载暴露密钥。
---

# Credential Vault（凭证保险库）

Credential Vault 是 OpenSandbox 面向沙箱化 agent 和开发者工具的出站凭证代理。真实凭证由宿主侧 SDK 写入 egress sidecar，而沙箱进程只会收到虚假或空的凭证值。当 Claude Code、Git、curl、包管理器或模型 API 客户端等工具发起允许的出站 HTTPS 请求时，sidecar 会将请求与 Credential Vault 绑定进行匹配，并在出站方向注入所需的认证头。这让现有工具保持正常工作流的同时，将真实密钥挡在沙箱环境、命令行、文件系统和日志之外，降低了来自提示注入或不可信代码的凭证外泄风险。

## 前提条件

- `opensandbox-server` >= 0.2.0
- `egress` >= 1.1.1
- Python SDK >= 0.1.11
- JavaScript/TypeScript SDK >= 0.1.9
- Kotlin SDK >= 1.0.13
- Go SDK >= 1.0.3
- C# SDK >= 0.1.3
- 服务器配置设置了 `[egress].image`。
- 服务器配置设置了 `[egress].mode = "dns+nft"`。Credential Vault 拒绝在仅 DNS 模式下启用，因为直连 IP 的连接可以绕过 DNS 策略。
- 沙箱创建请求包含出站网络策略。
- 出站网络策略应使用 `defaultAction="deny"`，并显式允许凭证绑定引用的每个主机。默认允许（default-allow）仍被暂时支持以保持向后兼容，但会发出安全警告。
- 沙箱创建请求启用了 Credential Proxy。
- 沙箱 Pod 在同一网络命名空间中没有运行额外的透明服务网格 sidecar（例如 Istio/Envoy 注入）。Credential Vault 目前假定 OpenSandbox egress sidecar 是 Pod 中唯一的透明出站拦截层。
- 沙箱镜像包含你想运行的工具。对于 Claude Code，使用带有 Node.js 和 npm 的镜像，例如 OpenSandbox code-interpreter 镜像。

::: warning 迁移通知
Credential Proxy 仍然要求服务器 `[egress].mode = "dns+nft"`；无法提供 nft 强制执行的部署无法安全启用凭证注入。
默认允许策略在兼容期内仍被接受，但会发出安全警告，应在未来版本收紧强制执行之前迁移到 `defaultAction="deny"`。
:::

## 工作原理

![Credential Vault 请求流程](../public/images/credential-vault.png)

Credential Vault 由 egress sidecar 实现。沙箱必须同时带有出站 `network_policy` / `networkPolicy` 并启用 Credential Proxy 才能创建。生命周期 API 字段名是 `credentialProxy.enabled`；SDK 使用各自语言的命名约定暴露该字段。

从高层来看：

1. 生命周期服务器将 egress sidecar 附加到沙箱。
2. SDK 将凭证和绑定写入 sidecar 的 Credential Vault API。
3. 沙箱进程使用虚假或空的凭证环境变量运行。
4. 当沙箱发起 HTTPS 请求时，sidecar 中的透明 MITM 会检查请求元数据。
5. 如果恰好有一个绑定匹配请求的 scheme、主机、端口、方法和路径，sidecar 会注入配置的认证头和作用域占位符替换。
6. 密钥值会从 vault 响应和响应头中被脱敏。

当活动 vault 检查成功时，不匹配任何凭证绑定的请求原样转发。凭证路径安全检查仅在绑定匹配、且请求本会获得凭证之后才适用。活动 vault 检查本身在每个被拦截的流的绑定选择之前运行，包括发往不会匹配绑定的主机的请求。

MITM 进程使用的活动 vault 通过 sidecar 内部的本地 Unix domain socket 提供服务。沙箱工作负载无法通过正常的服务器代理路径获取此活动状态。

每条新流会条件性地用缓存的不可读快照标签在 `If-None-Match` 中读取该私有端点。`304 Not Modified` 响应会重用同一不可变的进程内快照，而不渲染或传输密钥值；`200 OK` 响应携带一个完全验证过的替换快照和新的不可读 `ETag`。成功的创建和补丁响应只有在存储更改了该标签后才会被确认，因此下一条流会立即观察到新凭证，没有缓存 TTL 延迟。

该不可读标签与公共 vault revision 是分开的。删除一个 vault 再重新创建它可以将公共 revision 重置为 `1`，但重新创建的 vault 会收到不同的标签，无法复用被删除快照中的明文。私有端点返回 `404 Not Found` 是正常的"无活动 vault"结果：MITM 进程清除其缓存的快照并继续运行，不进行凭证注入。

::: warning 运行时可用性依赖
私有活动 vault 套接字是 Credential Proxy 拦截的所有流量的硬可用性依赖，而不仅仅是最终匹配凭证绑定的流量。超时、连接被拒、意外的 EOF、`5xx` 响应、格式错误的快照或无效的 `ETag` 都会清除受影响的明文缓存，并在任何上游转发之前以失败关闭该请求。主体被安全缓冲的小请求会收到本地 `503 Service Unavailable` 响应。已经在流式传输、可能跨过流式阈值或主体长度未知的请求则会被终止，因为 mitmproxy 无法在流式传输开始后安全地合成一个本地响应。运维人员应将 sidecar 和私有套接字作为 egress 数据平面可用性的一部分进行监控。
:::

## 跨暂停与恢复的持久化

::: warning 内存中的状态
Credential Vault 条目位于 egress sidecar 的进程本地内存中；它们不是沙箱根文件系统或 `BatchSandbox` Pod 模板的一部分。Kubernetes pause 在快照后会删除 Pod，因此恢复时新创建的 egress sidecar 从空 vault 开始。在受信任的客户端再次创建凭证和绑定之前，凭证注入不会恢复。

在沙箱之外的受信任控制平面中保留原始的 vault 请求或等效的密钥管理器引用。在沙箱回到 `Running` 之后，在允许依赖这些凭证的工作之前，再次调用 Credential Vault 创建 API。不要在沙箱元数据、环境变量、快照或日志中持久化真实凭证值。

Docker pause/unpause 保留现有容器进程，但任何 egress sidecar 替换或重启也会创建新的内存 vault，并需要同样的重新注入过程。
:::

## 服务网格兼容性

Credential Vault 依赖 egress sidecar 的透明重定向和 MITM 路径。如果沙箱 Pod 还被注入了诸如 Istio/Envoy 之类的透明服务网格 sidecar，两个层会在同一网络命名空间中尝试拦截出站流量。OpenSandbox 目前不支持该组合用于 Credential Vault。

请改用以下运维模式之一：

- 对需要 Credential Vault 的沙箱 Pod 禁用网格 sidecar 注入
- 保持网格注入启用，但不要为这些 Pod 启用 `credentialProxy` / Credential Vault
- 如果网格注入是强制性的，则将出站策略和凭证处理移到沙箱 Pod 之外的平台机制

关于底层 egress sidecar 的限制，参见 [Egress](/architecture/network/egress#service-mesh-compatibility)。

凭证绑定被有意设计得精确。默认拒绝的 egress 策略是必需的。使用狭窄的路径匹配，例如对 Anthropic API 调用使用 `/v1/*`。

## 认证类型

每个绑定使用 `auth` 规则描述被引用的凭证如何被渲染进出站请求：

- `bearer`：注入 `Authorization: Bearer <credential>`。
- `basic`：注入 `Authorization: Basic <credential>`。凭证值必须已经是 base64 编码的 `username:password`。
- `apiKey`：将凭证值注入到配置的头名称中。
- `customHeaders`：注入多个配置的头，每个头由各自的凭证支撑。
- `passthrough`：不注入认证头。当上游 API 要求在路径、查询字符串或主体占位符中而非头中提供凭证时，将它与 `substitutions` 一起使用。

简单示例：

```python
auth={"type": "bearer", "credential": "github-token"}
```

```http
Authorization: Bearer <github-token>
```

```python
auth={"type": "basic", "credential": "registry-basic"}
```

```http
Authorization: Basic <base64(username:password)>
```

```python
auth={"type": "apiKey", "name": "x-api-key", "credential": "anthropic-api-key"}
```

```http
x-api-key: <anthropic-api-key>
```

```python
auth={
    "type": "customHeaders",
    "headers": [
        {"name": "X-Client-Id", "credential": "client-id"},
        {"name": "X-Client-Secret", "credential": "client-secret"},
    ],
}
```

```http
X-Client-Id: <client-id>
X-Client-Secret: <client-secret>
```

### 作用域占位符替换

一些上游 API 要求在请求 URL 或主体中而非专用认证头中提供凭证。Credential Vault 可以处理这些 API，而无需将真实凭证放入沙箱进程。所有认证类型都接受一个可选的 `substitutions` 列表。每个替换指定一个凭证、一个字面占位符，以及允许替换的请求表面：

```python
auth={
    "type": "passthrough",
    "substitutions": [
        {
            "credential": "client-secret",
            "placeholder": "__client_secret__",
            "in": ["body", "query"],
        }
    ],
}
```

当绑定只执行替换而不应注入认证头时，使用 `type="passthrough"`。当同一上游请求同时需要头注入和占位符替换时，你也可以将替换与 `bearer`、`basic`、`apiKey` 或 `customHeaders` 组合使用。

替换默认禁用，并且是精确的、字面的、大小写敏感的。只有配置的表面会被重写：

| 表面 | 行为 |
| --- | --- |
| `path` | 替换请求路径中的占位符并对凭证值进行 URL 编码。如果重写后的路径包含歧义路径段、编码分隔符或类似目录遍历的内容，sidecar 会拒绝该请求，而不是将携带密钥的 URL 转发到匹配范围之外。 |
| `query` | 替换查询字符串中的占位符并对凭证值进行 URL 编码。 |
| `header` | 替换原始请求头中的占位符，排除逐跳头和安全敏感头，例如 `Host`、`Content-Length` 和转发头。替换在 Credential Vault 注入认证头之前运行，因此 sidecar 不会重写它自己创建的凭证头。 |
| `body` | 替换 UTF-8 请求主体中的占位符。对于 `application/json`，替换被编码为 JSON 字符串内容，因此请将占位符放在带引号的 JSON 字符串内。对于 `application/x-www-form-urlencoded`，替换以表单编码方式写入。压缩和 multipart 主体会被跳过。 |

sidecar 针对一个表面，基于原始请求文本一次性应用所有替换。插入的凭证值不会再次被扫描以查找后续占位符，这防止了一个密钥意外重写另一个密钥。当主体被重写时，sidecar 会更新 `Content-Length` 并移除 `Transfer-Encoding`，因为转发的请求现在具有固定大小的缓冲主体。

占位符、原始凭证值、URL 编码值、表单编码值和 JSON 转义值都会加入活动脱敏集。一个匹配了请求但没有找到任何占位符的带替换绑定会发出替换未命中日志，而不暴露凭证值。

示例配置：

```python
await sandbox.credential_vault.create(
    credentials=[
        Credential(name="tenant-id", source={"value": "tenant 42"}),
        Credential(name="api-key", source={"value": "query secret+value"}),
        Credential(name="client-secret", source={"value": 'body "secret" value'}),
    ],
    bindings=[
        CredentialBinding(
            name="token-request",
            match={
                "schemes": ["https"],
                "hosts": ["api.example.com"],
                "methods": ["POST"],
                "paths": ["/tenants/__tenant_id__/token"],
            },
            auth={
                "type": "passthrough",
                "substitutions": [
                    {
                        "credential": "tenant-id",
                        "placeholder": "__tenant_id__",
                        "in": ["path"],
                    },
                    {
                        "credential": "api-key",
                        "placeholder": "__api_key__",
                        "in": ["query"],
                    },
                    {
                        "credential": "client-secret",
                        "placeholder": "__client_secret__",
                        "in": ["body"],
                    },
                ],
            },
        )
    ],
)
```

沙箱可以使用占位符代替真实密钥：

```bash
curl -X POST \
  "https://api.example.com/tenants/__tenant_id__/token?api_key=__api_key__" \
  -H "content-type: application/json" \
  --data '{"client_secret":"__client_secret__"}'
```

上游收到重写后的请求：

```http
POST /tenants/tenant%2042/token?api_key=query%20secret%2Bvalue HTTP/1.1
content-type: application/json

{"client_secret":"body \"secret\" value"}
```

## Egress Sidecar 配置

| 环境变量 | 默认值 | 描述 |
| --- | --- | --- |
| `OPENSANDBOX_EGRESS_CREDENTIAL_VAULT_REQUIRE_TLS` | 关闭 | 启用时（`true`/`1`/`on`），credential vault 写操作（创建、补丁、删除）要求 TLS、回环传输，或来自已配置受信任代理的 `X-Forwarded-Proto: https`。禁用时（默认），任何已认证的请求都会被接受，无论传输方式如何。在 egress sidecar 可从不可信网络直接访问且没有 TLS 终止反向代理的部署中启用此项。 |
| `OPENSANDBOX_EGRESS_CREDENTIAL_VAULT_TRUSTED_PROXY_CIDRS` | 空 | 允许声明 `X-Forwarded-Proto: https` 的逗号分隔 IP 地址或 CIDR 列表。所有其他对端的转发传输头将被忽略。当 TLS 在 egress sidecar 之前的反向代理处终止时配置此项。 |
| `OPENSANDBOX_EGRESS_CREDENTIAL_VAULT_REQUIRE_SCOPED_MATCH` | 关闭 | 启用时，每个凭证绑定必须显式提供非空的 `match.methods` 和 `match.paths`，且路径不能包含主机范围的 `/*` 模式。省略任一字段的 vault 创建和补丁请求将被拒绝，而不是收到兼容性默认值。在将高层 API 工具定义转换为 Credential Vault 绑定的控制平面中启用此项。 |

scoped-match 设置是一个 fail-closed 的集成防护。它不会从其他平台的 API 工具定义推断方法或路径。创建 vault 的调用者仍然必须将预期的方法和路径复制到每个绑定中，并验证脱敏后的绑定读回。


## SDK 快速参考

所有沙箱 SDK 使用相同的线缆契约。主要差异是命名和语言风格：

| SDK | 在沙箱创建时启用代理 | Vault 入口点 | 创建 / 补丁方法 |
| --- | --- | --- | --- |
| Python | `credential_proxy=CredentialProxyConfig(enabled=True)` | `sandbox.credential_vault` | `create(...)`、`patch(...)` |
| Go | `CredentialProxy: &opensandbox.CredentialProxyConfig{Enabled: true}` | `sandbox.CredentialVault(ctx)` 或 sandbox 辅助方法 | `CreateCredentialVault(ctx, req)`、`PatchCredentialVault(ctx, req)` |
| JavaScript/TypeScript | `credentialProxy: { enabled: true }` | `sandbox.credentialVault` | `create(request)`、`patch(request)` |
| Kotlin/JVM | `.credentialProxyEnabled(true)` 或 `.credentialProxy { enabled(true) }` | `sandbox.credentialVault()` | `create(request)`、`patch(request)` |
| C#/.NET | `CredentialProxy = new CredentialProxyConfig { Enabled = true }` | `sandbox.CredentialVault` 或 sandbox 辅助方法 | `CreateCredentialVaultAsync(...)`、`PatchCredentialVaultAsync(...)` |

Vault API 返回脱敏的元数据。明文凭证值是只写的，不会由 `get`、`list` 或补丁响应返回。

## Claude Code 与 Anthropic

此示例在沙箱中安装 Claude Code 并调用官方 Anthropic API 端点。真实 API key 在宿主上读取并写入 Credential Vault。沙箱只能看到虚假的 `ANTHROPIC_API_KEY`。

在运行脚本之前：

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
# Optional: export ANTHROPIC_MODEL="<a Claude Code supported Anthropic model>"
```

运行：

```python
import os
from datetime import timedelta

from opensandbox import SandboxSync
from opensandbox.models.sandboxes import (
    Credential,
    CredentialBinding,
    CredentialProxyConfig,
    NetworkPolicy,
    NetworkRule,
    SandboxImageSpec,
)


ANTHROPIC_HOST = "api.anthropic.com"
ANTHROPIC_BASE_URL = "https://api.anthropic.com"
REAL_API_KEY = os.environ["ANTHROPIC_API_KEY"]


sandbox_env = {
    "ANTHROPIC_BASE_URL": ANTHROPIC_BASE_URL,
    "ANTHROPIC_API_KEY": "fake-key-inside-sandbox",
}
if os.getenv("ANTHROPIC_MODEL"):
    sandbox_env["ANTHROPIC_MODEL"] = os.environ["ANTHROPIC_MODEL"]


sandbox = SandboxSync.create(
    image=SandboxImageSpec(
        os.getenv("SANDBOX_IMAGE", "opensandbox/code-interpreter:latest")
    ),
    timeout=timedelta(minutes=15),
    env=sandbox_env,
    network_policy=NetworkPolicy(
        defaultAction="deny",
        egress=[
            NetworkRule(action="allow", target=ANTHROPIC_HOST),
            NetworkRule(action="allow", target="registry.npmjs.org"),
        ],
    ),
    credential_proxy=CredentialProxyConfig(enabled=True),
)

try:
    sandbox.credential_vault.create(
        credentials=[
            Credential(
                name="anthropic-api-key",
                source={"value": REAL_API_KEY},
            )
        ],
        bindings=[
            CredentialBinding(
                name="anthropic-api",
                match={
                    "schemes": ["https"],
                    "hosts": [ANTHROPIC_HOST],
                    "methods": ["GET", "POST"],
                    "paths": ["/v1/*"],
                },
                auth={
                    "type": "apiKey",
                    "name": "x-api-key",
                    "credential": "anthropic-api-key",
                },
            )
        ],
    )

    sandbox.commands.run(
        "npm install -g @anthropic-ai/claude-code --no-audit --no-fund"
    )
    result = sandbox.commands.run("claude -p '1+1'")
    output = "".join(part.text for part in result.logs.stdout)
    print(output)
finally:
    sandbox.kill()
    sandbox.close()
```

Claude Code 进程从 `ANTHROPIC_API_KEY` 读取虚假 key，但发往 `api.anthropic.com/v1/*` 的出站 HTTPS 请求会从 Credential Vault 获得真实的 `x-api-key` 头。如果你的环境使用私有 npm 镜像，请将网络策略和 `npm install` 命令中的 `registry.npmjs.org` 替换为该镜像主机。

## Git 与 Curl 使用 Vault 注入的凭证

Credential Vault 还可以保护 `git` 和 `curl` 等命令行工具使用的凭证。保持命令不含真实密钥，并将请求形态绑定到 Vault 中的凭证。

对于私有 Git 仓库，存储 base64 编码的 `username:token` 值并用 `basic` 认证绑定：

```python
Credential(name="git-basic", source={"value": "<base64(username:token)>"})

CredentialBinding(
    name="git-basic",
    match={
        "schemes": ["https"],
        "hosts": ["git.example.com"],
        "paths": ["/org/private-repo.git*"],
    },
    auth={"type": "basic", "credential": "git-basic"},
)
```

然后运行普通 URL 而不嵌入凭证：

```bash
GIT_TERMINAL_PROMPT=0 git clone https://git.example.com/org/private-repo.git
```

对于期望 token 头的 API 请求，将路径和方法绑定到 `apiKey` 认证规则：

```python
Credential(name="api-token", source={"value": "<token>"})

CredentialBinding(
    name="api-token",
    match={
        "schemes": ["https"],
        "hosts": ["api.example.com"],
        "methods": ["GET"],
        "paths": ["/v1/projects/123/variables"],
    },
    auth={"type": "apiKey", "name": "PRIVATE-TOKEN", "credential": "api-token"},
)
```

沙箱命令保持不含密钥：

```bash
curl -fsS https://api.example.com/v1/projects/123/variables
```

## 绑定指南

- 使用 `defaultAction="deny"`，只允许工具所需的服务主机。默认允许策略已弃用，因为它们可能允许凭证目标绕过，并且会发出安全警告。
- 尽可能按路径限定绑定范围，例如 `/v1/*`。
- 避免在同一优先级出现重叠绑定；歧义匹配会被拒绝。
- 被拒绝的请求（歧义路径、跨越绑定边界的编码分隔符）在请求主体小且完全已知时返回 `403`。主体超过 mitmproxy 流式传输阈值（约 1 MiB）或长度未知（chunked）的请求在主体流式传输期间无法以 `403` 应答，因此 sidecar 会改为断开连接——无论哪种方式，请求都绝不向上游转发。
- 不要将真实密钥放在沙箱 `env`、命令参数、文件或元数据中。
- 当 CLI 没有密钥就拒绝启动时，保留虚假环境变量；vault 注入的头才是为出站请求认证的凭据。

## 路径歧义防护

sidecar 只对会获得注入凭证的请求检查请求路径。当请求路径在任意百分号解码深度包含点段（`..`）或反斜杠（`\`、`%5c`），或当百分号解码未在防护的迭代边界内收敛时，请求会被 `403` 拒绝——这些是合法客户端不会发送的遍历或路径混淆原语。

编码斜杠（`%2f`，包括 `%252f` 等嵌套形式）在路径的每个解码深度都匹配同一凭证绑定时是允许的。这保持了合法的编码线缆格式可用——npm 作用域注册表路径（`/@scope%2fname`）以及坐标被双重编码的构件存储下载 URL（例如某些内部 pypi 代理生成的 `pkg%252F1.0`）——而解码后会跨越绑定边界的路径仍在注入任何凭证之前被拒绝。在所有绑定范围之外的请求原样通过。

## 从 `ports` 迁移

`match.ports` 字段已弃用。端口现在从 scheme 派生（`https`→443，`http`→80）。仅支持端口 80 和 443；非标准值会被验证错误拒绝。

如果你有使用 `ports` 缩小范围的现有绑定，请将它们迁移到等效的 `schemes` 限制：

| 之前 | 之后 |
|--------|-------|
| `schemes: ["http", "https"], ports: [443]` | `schemes: ["https"]` |
| `schemes: ["http", "https"], ports: [80]` | `schemes: ["http"]` |
| `schemes: ["https"], ports: [443]` | `schemes: ["https"]`（移除 `ports`） |

## 实验性实时凭证绑定准入（OSEP-0023）

::: warning 实验性边界
实时凭证绑定循环是一个内部 sidecar 实验，不是受支持的部署模式。此处记录它以界定当前实现实际做什么；开关默认保持关闭，且不添加公共配置接口。
:::

当实验性 revision 运行时启用时，sidecar 通过私有认证 IPC 通道向 mitmproxy 子进程安装一个不可变的决策快照，并按连接、按请求准入流量：

- **决策只来自已安装的快照。** 未绑定的 HTTPS 不透明地通过；绑定主机仅在注册表准入后才解密。未知、引导中或不匹配的状态都会拒绝——该模式绝不会回退到全解密或动态放行。
- **粘性恢复，同一 sidecar 不可重启。** 任何具有未知外部影响的终止结果（远程结果无法确认的变更、会话清理失败、意外的子进程退出）都会锁定一个永久的"需要恢复"状态：就绪性下降、未完成的引导票据失效、nft 写入静默，并尝试隔离。只有在隔离确认后才建立包隔离。同一 sidecar 之后无法引导新的 revision 会话；整个 sidecar 重启不是已验证的安全恢复过程。
- **请求准入与切换。** vault 变更首先应用于活跃接收器，只有在精确确认后才公开发布。超过 IPC 大小限制的渲染快照会在传输前以 HTTP 400 拒绝；当前 vault 和 sidecar 保持可用。进行中的请求在其被准入的 revision 上完成；主机被移除的连接上的新请求立即被围栏（403），退役的传输在 drain 截止时被强制关闭。删除 vault 影响新请求和新连接；退役的连接绝不能通过重新添加主机而复活。已准入的请求只能在有界的 drain 窗口内完成——删除不会撤回远程源已收到的请求，也不会撤销已远程交付的凭证。
- **策略更新共享 revision 屏障。** 现有的 `POST`/`PUT`/`PATCH`/`DELETE /policy` 操作和自动 `deny.always` / `allow.always` 重载会验证完整的 Policy/Vault 对。会拒绝任何活动绑定的策略在文件、nft 或接收器影响之前被拒绝。仅策略更新保留公共 Vault revision、固定的凭证字节和绑定；它不会重新读取凭证源。其提交顺序是可选的用户策略文件持久化、nft 应用、精确的接收器确认，然后是 Go proxy/base 发布和 API 成功。重载跳过用户策略持久化，并且仅在接收器确认后发布已接受的加载器视图。后续的 Vault 写入和引导使用已提交的内部 Policy epoch，而不是将其重置为零。
- **策略可见性与失败。** 磁盘、nft 和接收器更新不是瞬时原子的：新的 nft 规则可能在接收器确认之前影响流量。nft 应用成功后的接收器失败需要粘性恢复、隔离和精确的子进程清理，即使接收器明确拒绝了候选方案。重放旧 nft 规则或恢复文件无法证明新连接或请求已被撤销。不可用或未确认的 revision 更新和被恢复阻塞的写入返回固定的、脱敏的 503；初始的文件/nft 错误可能仍是 500。影响之前的验证失败本身不需要恢复。
- **策略空操作与重试。** 没有匹配目标的 DELETE 和未改变的 always-rule 重载不会推进内部 Policy epoch，但不能绕过恢复检查。其他已提交的写入，包括相同内容或加载器检测到的重写，可以推进 epoch 并发起另一次 drain。不添加公共 epoch、Policy 幂等键或新 SDK 方法。`GET /policy` 报告最后由 Go 发布的用户策略，而不是进行中的事务、always-rule 覆盖或包隔离状态。
- **Drain 计量。** 退役的传输带有一个有界的退役截止时间（默认 30 秒，内部范围 1–300）。drain 扫描会分页已过期的传输并关闭它们；关闭失败会在下一次扫描时重试，而不是延长截止时间。HTTP 101 协议升级（包括 WebSocket）在连接断开前保留其请求准入，因此凭证轮换和仅策略变更也会 drain 一个已升级的传输，即使其主机仍被绑定。API 成功等待发布和退役截止注册，而不是等待 30 秒窗口过去。重复更新绝不会延长较早请求的最早截止时间。仍被覆盖的连接可以准入新 revision 的请求，但为过期的旧请求关闭它也会中断那些较新的请求。仍被覆盖、没有旧请求的空闲连接不会仅因 Policy epoch 变化而退役。扫描调度和失败关闭重试可能延迟实际关闭；对于注册表之外的不透明或直接 IP 连接，这不是截止时间。
- **ECH 是不透明透传。** 携带 `encrypted_client_hello` 扩展（0xfe0d）的 ClientHello（包括 GREASE 形式）作为不透明隧道原样转发而不解密；无法检查的 ClientHello 数据会被拒绝。实现不解密内部 ClientHello，也不保证启用 ECH 的源握手成功。
- **HTTP/1 authority 门。** 在解密连接上，每个请求必须携带恰好一个语法严格的 `Host` 头，等于被拦截的 SNI 和准入绑定（端口省略或为 443）。冲突的绝对形式目标、流内的 CONNECT authority 形式、缺失/重复/格式错误的 Host 值以及 HTTP/2 请求都会在注入任何凭证之前被拒绝。
- **`ssl_insecure` 在实时模式下被拒绝。** 当实时准入包激活时，启动器和插件拒绝不安全的上游配置；仅安装模式的遗留接收器保持其现有行为。
- **范围。** 此模式限定于内部 sidecar 配置：Docker DNS + nft 强制执行、端口 443 上的精确主机 HTTPS、HTTP/1.1 origin-form 或匹配的绝对形式请求，以及作为头注入的内联凭证。HTTP/2、fast-sandbox 配置和跨完整 sidecar 替换的恢复仍不在范围内。动态 DNS 学习和 Kubernetes 部署未被本实验的证据覆盖。策略参与是本实现的一部分；其验收证据与下文的早期 live-Vault 检查分开记录。

此实验的任务级验收记录位于 [Live Vault Acceptance](/guides/live-vault-acceptance)。
