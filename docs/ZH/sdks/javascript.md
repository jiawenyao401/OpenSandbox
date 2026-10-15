---
title: JavaScript/TypeScript SDK
description: 用于创建、管理安全 OpenSandbox 环境并与之交互的 TypeScript/JavaScript SDK。
---

# OpenSandbox JavaScript/TypeScript SDK

一个用于与 OpenSandbox 进行低层交互的 TypeScript/JavaScript SDK。它提供了创建、管理安全 sandbox 环境并与之交互的能力，包括执行 shell 命令、管理文件和读取资源指标。

## 安装

### npm

```bash
npm install @alibaba-group/opensandbox
```

### pnpm

```bash
pnpm add @alibaba-group/opensandbox
```

### yarn

```bash
yarn add @alibaba-group/opensandbox
```

## 快速开始

此示例使用 Node.js 20+，配合 ES modules 和顶层 `await`。它会创建一个 sandbox、运行一条 shell 命令，并释放远程与本地资源。

::: tip
运行此示例前，请确保 OpenSandbox 服务已在运行。启动说明参见[快速上手](/getting-started/)指南。
:::

```ts
import { ConnectionConfig, Sandbox, SandboxException } from "@alibaba-group/opensandbox";

const config = new ConnectionConfig({
  domain: "api.opensandbox.io",
  apiKey: "your-api-key",
  // protocol: "https",
  // requestTimeoutSeconds: 60,
});

try {
  const sandbox = await Sandbox.create({
    connectionConfig: config,
    image: "ubuntu",
    timeoutSeconds: 10 * 60,
  });

  try {
    const execution = await sandbox.commands.run("echo 'Hello Sandbox!'");
    console.log(execution.logs.stdout[0]?.text);
  } finally {
    try {
      await sandbox.kill();
    } finally {
      await sandbox.close();
    }
  }
} catch (err) {
  if (err instanceof SandboxException) {
    console.error(
      `Sandbox Error: [${err.error.code}] ${err.error.message ?? ""}`,
    );
    console.error(`Request ID: ${err.requestId ?? "N/A"}`);
  } else {
    console.error(err);
  }
}
```

## 生命周期钩子

在 `Sandbox.create` 中设置 `lifecycle`。`preStart` 在入口点启动之前完成，而 `periodic` 钩子在启动后按各自的计划运行。

```ts
const sandbox = await Sandbox.create({
  connectionConfig: config,
  image: "ubuntu:24.04",
  lifecycle: {
    preStart: {
      command: ["sh", "-c", "echo ready > /tmp/prestart.done"],
      timeoutSeconds: 120,
    },
    periodic: [
      {
        name: "checkpoint",
        schedule: "@every 5m",
        command: ["sh", "-c", "date -u >> /tmp/checkpoints.log"],
        timeoutSeconds: 120,
      },
    ],
  },
});
```

Server 会校验 `timeoutSeconds`；`preStart` 接受 1–10800 秒，而 `periodic` 接受 1–300 秒。两者在省略时均默认为 60 秒。计时、失败行为和 provider 限制参见[生命周期钩子](/guides/lifecycle-hooks)。

### 空闲策略（自动暂停与访问唤醒）

`lifecycle.idlePolicy`（OSEP-0024，仅 fast-sandbox）告诉控制面在 ingress 观测到连续空闲流量达到 `idleTimeoutSeconds`（30–86400）后自动暂停 sandbox，并在下一个请求到来时唤醒它。它属于控制面状态：永远不会到达 guest，在模板模式（`Sandbox.createFromTemplate({ idlePolicy })`）下被允许，并被非 fast-sandbox 后端拒绝。省略它则完全保持非自动化行为。

```ts
const sandbox = await Sandbox.create({
  connectionConfig: config,
  image: "ubuntu:24.04",
  timeoutSeconds: 86400,
  lifecycle: {
    idlePolicy: {
      idleTimeoutSeconds: 300,
      onIdle: "pause",
      wakeOnAccess: true,
    },
  },
});
```

`activeProbe` 让应用在真实工作运行但不产生 ingress 流量时否决暂停：声明一个以 `{"active": bool, "message"?: string}` 应答的端点，当 sandbox 除此之外处于空闲状态时 server 会轮询它；忙碌的应答会重置空闲时钟。

### 唤醒重试

对已暂停 sandbox 的请求可能在其恢复仍在进行时到达。网关以 `OpenSandbox-Wake: expired` 头应答 `503`；传输层识别该标记并在 `Retry-After` 延迟之后重发原始请求（对任何方法都安全 —— 该标记证明请求从未被转发）。唤醒预算默认在所有唤醒周期内为 5 次尝试 / 60 秒；预算耗尽时 SDK 抛出 `SandboxWakeTimeoutException`（携带 `attempts` 和 `waitedMs`），而 server 侧的恢复在后台继续。裸 `503` 响应遵循既有规则、不受影响。可在连接配置上调整：`wakeRetryEnabled`（默认 true）、`wakeMaxAttempts`（5）、`wakeOverallDeadlineSeconds`（60，`null` 表示无上限）。SSE 传输也被包装，因此 `commands.run()` 会像其他请求一样唤醒已暂停的 sandbox（已建立的流绝不会被重新分发 —— 只有带标记的 503 连接会重放）；上传传输保持裸露，因为上传请求体是流，无法重放。

## Client Pool 与可观测性

`SandboxPool` 提供内存与 Redis 存储四种获取策略以及分阶段 warmup 控制。Redis 存储从 `@alibaba-group/opensandbox/pool-redis` 导出。示例、配置和命名空间退役参见 [Client Pool](/guides/client-pool)。

在 `ConnectionConfig` 中设置 `enableTracing: true` 以启用 [pool warmup 追踪](/sdks/observability#pool-warmup-tracing)。JavaScript 会输出阶段 span，但诊断属性少于 Python/JVM。创建延迟[遥测](/sdks/observability#creation-metrics)有单独的退出设置。远程诊断日志/事件可通过 [CLI 或 HTTP](/api/#diagnostics) 获取，而无法通过 JavaScript SDK。

## 使用示例

以下代码片段使用快速开始中的 `config` 和存活的 `sandbox`。请在快速开始的清理块之前运行它们。示例在 Node.js 中使用带顶层 `await` 的 TypeScript；在 JavaScript 中省略类型注解。各创建示例互为替代。用完的 sandbox 以 `kill()` 终止，并以 `close()` 释放其客户端。

### 1. 生命周期管理

管理 sandbox 生命周期，包括续期、暂停和恢复。

```ts
const info = await sandbox.getInfo();
console.log("State:", info.status.state);
console.log("Created:", info.createdAt);
console.log("Expires:", info.expiresAt); // null when manual cleanup mode is used

await sandbox.pause();

const deadline = Date.now() + 120_000;
while (true) {
  if (Date.now() >= deadline) throw new Error("Sandbox did not pause within 120 seconds");
  const current = await sandbox.getInfo();
  if (current.status.state === "Paused") break;
  if (current.status.state === "Failed") throw new Error(current.status.message);
  await new Promise((resolve) => setTimeout(resolve, 1000));
}

// Resume returns a fresh handle; close both handles when done.
const resumed = await sandbox.resume();
try {
  await resumed.renew(30 * 60); // expiresAt = now + timeoutSeconds
} finally {
  await resumed.close();
}
```

暂停是异步的且取决于运行时。参见[暂停与恢复](/guides/pause-resume)。使用 `Sandbox.connect({ sandboxId, connectionConfig: config })` 可以附加到已在运行的 sandbox。

通过传入 `timeoutSeconds: null` 创建不过期的 sandbox：

```ts
const manual = await Sandbox.create({
  connectionConfig: config,
  image: "ubuntu",
  timeoutSeconds: null,
});
```

### 2. 自定义健康检查

解析端点只能确认路由存在；它不能确认该端口上的应用是健康的。对于服务就绪，应对应用的健康端点发起有边界的请求，并携带返回的端点头。

自定义判断 sandbox 是否就绪/健康的逻辑。这会覆盖默认的 ping 检查。检查不得阻塞事件循环，且可在超时后继续运行。

```ts
const sandbox = await Sandbox.create({
  connectionConfig: config,
  image: "nginx:latest",
  entrypoint: ["nginx", "-g", "daemon off;"],
  healthCheck: async (sbx) => {
    const ep = await sbx.getEndpoint(80);
    const url = await sbx.getEndpointUrl(80);
    try {
      const response = await fetch(url, {
        headers: ep.headers,
        signal: AbortSignal.timeout(2000),
      });
      await response.body?.cancel();
      return response.status === 200;
    } catch {
      return false;
    }
  },
});
```

### 3. 命令执行与流式输出

实时执行命令并处理输出流。

```ts
import type { ExecutionHandlers } from "@alibaba-group/opensandbox";

const handlers: ExecutionHandlers = {
  onStdout: (m) => console.log("STDOUT:", m.text),
  onStderr: (m) => console.error("STDERR:", m.text),
  onExecutionComplete: (c) =>
    console.log("Finished in", c.executionTimeMs, "ms"),
};

await sandbox.commands.run(
  'for i in 1 2 3; do echo "Count $i"; sleep 0.2; done',
  undefined,
  handlers,
);
```

要执行不带 shell 解析的原生程序，请传入参数列表。在 Linux 上，此示例打印字面的 `$HOME` 并将 `hello world` 作为一个参数保留：

```ts
await sandbox.commands.run(["printf", "%s\n", "$HOME", "hello world"]);
```

原生 argv 执行需要更新版本的 execd。可执行文件查找与平台行为参见[命令执行模式](/architecture/data-plane/execd#command-execution)。

#### 后台命令

轮询状态和增量日志。命令超时与 sandbox TTL 相互独立。

```ts
const execution = await sandbox.commands.run(
  'for i in 1 2 3; do echo "step $i"; sleep 1; done',
  { background: true, timeoutSeconds: 30 },
);
if (!execution.id) throw new Error("No command ID returned");
let cursor = 0;
const deadline = Date.now() + 45_000;
while (true) {
  if (Date.now() >= deadline) {
    await sandbox.commands.interrupt(execution.id);
    throw new Error("Command did not finish");
  }
  const status = await sandbox.commands.getCommandStatus(execution.id);
  const logs = await sandbox.commands.getBackgroundCommandLogs(execution.id, cursor);
  process.stdout.write(logs.content);
  cursor = logs.cursor ?? cursor;
  if (status.running === false) {
    if (status.exitCode !== 0) throw new Error(`Command failed: ${status.exitCode}, ${status.error}`);
    break;
  }
  await new Promise((resolve) => setTimeout(resolve, 500));
}
```

#### 持久 shell 会话

Bash 会话可跨命令保留 shell 变量和工作目录。

```ts
const sessionId = await sandbox.commands.createSession({ workingDirectory: "/tmp" });
try {
  await sandbox.commands.runInSession(sessionId, "export DEMO=hello");
  const result = await sandbox.commands.runInSession(sessionId, 'echo "$DEMO"; pwd');
  console.log(result.logs.stdout.map((message) => message.text).join(""));
} finally {
  await sandbox.commands.deleteSession(sessionId);
}
```

#### 持久环境变量

设置由运行时注入到每个后续命令和会话中的环境变量 —— 无需手写 shell 转义去操作 sandbox env 文件。

```ts
await sandbox.commands.setEnv("MY_TOKEN", "it's a safe value");
```

键必须匹配 `[A-Za-z_][A-Za-z0-9_]*`。不含单引号的值按原样存储；含单引号的值使用 env 文件的双引号形式，其中 shell 风格的 `$NAME` 序列可能在运行时加载该文件时被展开。env 文件是追加式的：同一键的最后一次写入生效。如果 sandbox 未能持久化该变量，将抛出异常。

关于 sandbox 内的文件系统/进程隔离，参见[隔离会话](/guides/isolation-sessions)。它们与 Bash 会话是相互独立的概念。

### 4. 文件操作

管理文件和目录，包括读取、写入、列出/搜索和删除。

```ts
await sandbox.files.createDirectories([{ path: "/tmp/demo", mode: 755 }]);

await sandbox.files.writeFiles([
  { path: "/tmp/demo/hello.txt", data: "Hello World", mode: 644 },
]);

const content = await sandbox.files.readFile("/tmp/demo/hello.txt");
console.log("Content:", content);

const entries = await sandbox.files.listDirectory({ path: "/tmp/demo", depth: 1 });
console.log(entries.map((entry) => entry.path));

const files = await sandbox.files.search({
  path: "/tmp/demo",
  pattern: "*.txt",
});
console.log(files.map((f) => f.path));

await sandbox.files.deleteDirectories(["/tmp/demo"]);
```

对于二进制文件，请向 `writeFiles()` 传入 `Uint8Array`，并用 `readBytes()` 或 `readBytesStream()` 读取。读取选项支持 `offset` 和 `limit` 以实现部分下载。

#### 带显式身份的文件操作

将文件系统客户端绑定到 Linux UID 和 GID：

```ts
const files = sandbox.filesWithIdentity(1001, 2000);
```

绑定的客户端暴露常规文件操作，并共享 sandbox 连接、认证和端点路由头。既有文件客户端保持其默认身份。命令执行时请单独选择 UID/GID。

这需要 Linux 以及支持文件系统执行身份的 Execd 版本。不支持的服务器会返回错误；请求绝不会回退到默认文件系统路由。请将 sandbox 凭证和身份选择保留在可信后端。Linux 文件系统权限依然生效；该特性并不创建租户隔离边界。权限与 workspace 供给详情参见 [Execd](/architecture/data-plane/execd)。

### 5. 端点

`getEndpoint()` 返回**不带协议**的端点（例如 `"localhost:44772"`）。如需可直接使用的绝对 URL（例如 `"http://localhost:44772"`），请使用 `getEndpointUrl()`。

```ts
const endpoint = await sandbox.getEndpoint(44772);
const url = await sandbox.getEndpointUrl(44772);
console.log(url, Object.keys(endpoint.headers ?? {}));
```

向 sandbox 服务发起 HTTP 请求时，请转发 `endpoint.headers`，包括 secure access 所要求的凭证。上面的健康检查示例展示了向确实在目标端口上监听的应用发起请求的方式。

### 6. 卷挂载

`volumes` 支持 `host`、`pvc` 和 `ossfs` 三种后端。每个卷必须且只能指定一种后端。

```ts
const sandbox = await Sandbox.create({
  connectionConfig: config,
  image: "ubuntu",
  volumes: [
    {
      name: "oss-data",
      ossfs: {
        bucket: "bucket-a",
        endpoint: "oss-cn-hangzhou.aliyuncs.com",
        accessKeyId: process.env.OSS_ACCESS_KEY_ID!,
        accessKeySecret: process.env.OSS_ACCESS_KEY_SECRET!,
        version: "2.0",
      },
      mountPath: "/mnt/oss",
      subPath: "prefix",
    },
  ],
});
```

### 7. Sandbox 管理（管理面）

使用 `SandboxManager` 执行管理任务和查找既有 sandbox。

```ts
import { SandboxManager } from "@alibaba-group/opensandbox";

const manager = SandboxManager.create({ connectionConfig: config });
try {
  // First page only; increase page for subsequent pages.
  const list = await manager.listSandboxInfos({
    states: ["Running"], pageSize: 10, page: 1,
  });
  console.log(list.items.map((s) => s.id));
} finally {
  await manager.close();
}
```

### 资源指标

使用 `await sandbox.getMetrics()` 读取当前 sandbox 的资源使用。这与 [SDK 创建遥测](/sdks/observability#creation-metrics)相互独立。

## 快照、模板与元数据

| 操作 | 公共 API |
| --- | --- |
| 为 sandbox 创建快照 | `manager.createSnapshot(sandboxId, { name })` |
| 查看/列出/删除快照 | `manager.getSnapshot`、`listSnapshots`、`deleteSnapshot` |
| 恢复快照 | `Sandbox.create({ snapshotId, connectionConfig })` |
| 管理 Fsb 模板 | `manager.createTemplate`、`getTemplate`、`listTemplates`、`deleteTemplate` |
| 从已发布模板创建 | `Sandbox.createFromTemplate({ templateId, timeoutSeconds, connectionConfig })` |
| 修补元数据 | `sandbox.patchMetadata` 或 `manager.patchSandboxMetadata` |

恢复之前请轮询快照状态。模板构建是异步的；使用前请等待 `status.phase === "Succeeded"`。基于模板的创建需要 TTL，并从已发布的模板继承工作负载配置。元数据修补值会新增/替换键；`null` 删除键。后端约束参见[生命周期契约](/api/#1-sandbox-lifecycle-yml)。

快照支持取决于运行时与 server 配置。如果源 sandbox 的剩余 TTL 可能在快照创建期间到期，请先续期。恢复之前请等待 `Ready`；此示例结束后快照仍然保留：

```ts
import { SandboxManager } from "@alibaba-group/opensandbox";

const manager = SandboxManager.create({ connectionConfig: config });
try {
  let snapshot = await manager.createSnapshot(sandbox.id, { name: "demo" });
  console.log("Snapshot:", snapshot.id);
  const deadline = Date.now() + 900_000;
  while (true) {
    if (Date.now() >= deadline) throw new Error(`Snapshot ${snapshot.id} is not ready`);
    snapshot = await manager.getSnapshot(snapshot.id);
    if (snapshot.status.state === "Ready") break;
    if (snapshot.status.state === "Failed") throw new Error(snapshot.status.message);
    await new Promise((resolve) => setTimeout(resolve, 2000));
  }
  const restored = await Sandbox.create({ snapshotId: snapshot.id, connectionConfig: config });
  try {
    console.log(restored.id);
  } finally {
    try { await restored.kill(); } finally { await restored.close(); }
  }
  // When no longer needed: await manager.deleteSnapshot(snapshot.id);
} finally {
  await manager.close();
}
```

在模板构建达到 `Succeeded` 后使用既有模板：

```ts
const templated = await Sandbox.createFromTemplate({
  templateId: "your-published-template-id",
  timeoutSeconds: 600,
  connectionConfig: config,
});
```

新增/替换一个元数据键并移除另一个：

```ts
await sandbox.patchMetadata({ project: "demo", "obsolete-key": null });
```

## 配置

### 1. 连接配置

`ConnectionConfig` 类管理 API server 的连接设置。

::: info 运行时说明
- 在浏览器中，SDK 使用全局的 `fetch` 实现。
- `requestTimeoutSeconds` 限制响应等待时间：一旦响应头到达即不再生效（流式响应不会被切断）。它永远不会限制健康的文件上传请求体 —— 后者合法地可能超过该超时；在 Node.js 上，停滞的上传、或请求体已发出但在超时内未收到响应头的上传，会被中止而不是一直挂起（即 Go SDK 的 `ResponseHeaderTimeout` 契约）。
- 在 Node.js 中，每个 `Sandbox` 和 `SandboxManager` 都会通过 `withTransportIfMissing()` 克隆基础 `ConnectionConfig`，因此每个实例都会获得隔离的 `undici` keep-alive 连接池。使用完毕后请调用 `sandbox.close()` 或 `manager.close()`，以便 SDK 释放关联的 agent。
:::

| 参数 | 说明 | 默认值 | 环境变量 |
| ----------------------- | ------------------------------------------------------------------------------------------------------------ | ---------------- | ---------------------- |
| `apiKey` | 用于认证的 API 密钥 | 可选 | `OPEN_SANDBOX_API_KEY` |
| `domain` | sandbox 服务域（`host[:port]`） | `localhost:8080` | `OPEN_SANDBOX_DOMAIN`  |
| `protocol` | HTTP 协议（`http`/`https`） | `http`           | -                      |
| `requestTimeoutSeconds` | 应用于 SDK HTTP 调用的请求超时 | `30`             | -                      |
| `debug` | 启用基础 HTTP 调试日志 | `false`          | -                      |
| `headers` | 应用于每个请求的额外头 | `{}`             | -                      |
| `useServerProxy` | 将 sandbox server 用作 execd/端点请求的代理（例如客户端无法直接访问 sandbox 时） | `false`          | -                      |
| `enableTracing` | 启用 [pool warmup 追踪](/sdks/observability#pool-warmup-tracing) | `false` | - |
| `disableMetrics` | 禁用 SDK 创建延迟遥测（参见 [SDK 遥测](/sdks/observability#creation-metrics)） | `false`          | `OPENSANDBOX_DISABLE_METRICS` |

```ts
import { ConnectionConfig } from "@alibaba-group/opensandbox";

// 1. Basic configuration
const config = new ConnectionConfig({
  domain: "api.opensandbox.io",
  apiKey: "your-key",
  requestTimeoutSeconds: 60,
});

// 2. Advanced: custom headers
const config2 = new ConnectionConfig({
  domain: "api.opensandbox.io",
  apiKey: "your-key",
  headers: { "X-Custom-Header": "value" },
});
```

### 2. Sandbox 创建配置

`Sandbox.create()` 允许配置 sandbox 环境。

| 参数 | 说明 | 默认值 |
| ---------------------------- | ------------------------------------------------ | ---------------------------- |
| `image` | 要使用的 Docker 镜像 | image 或快照 ID 二选一 |
| `timeoutSeconds` | 自动终止超时（server 侧 TTL） | 10 分钟 |
| `entrypoint` | 容器入口命令 | `["tail","-f","/dev/null"]`  |
| `resource` | CPU 与内存限制（字符串映射） | `{"cpu":"1","memory":"2Gi"}` |
| `env` | 环境变量 | `{}`                         |
| `metadata` | 自定义元数据标签 | `{}`                         |
| `networkPolicy` | 可选的出站网络策略（egress） | -                            |
| `credentialProxy` | 可选的 Credential Vault 代理启动设置 | -                            |
| `extensions` | 额外的 server 定义字段 | `{}`                         |
| `skipHealthCheck` | 跳过就绪检查（`Running` + 健康检查） | `false`                      |
| `healthCheck` | 自定义就绪检查 | -                            |
| `readyTimeoutSeconds` | 等待就绪的最长时间 | 30 秒 |
| `healthCheckPollingInterval` | 等待期间的轮询间隔（毫秒） | 200 ms |
| `snapshotId` | 恢复快照而不是传 `image` | - |
| `resourceRequests` | Kubernetes 资源请求；不得超过限制 | - |
| `lifecycle` | 启动前与周期性钩子 | - |
| `platform` | OS/架构约束 | - |
| `volumes` | Host、PVC 或 OSSFS 挂载 | - |
| `secureAccess` | 要求端点访问凭证 | `false` |

::: warning
`opensandbox.io/` 下的元数据键保留给系统管理的标签，将被 server 拒绝。
:::

```ts
const sandbox = await Sandbox.create({
  connectionConfig: config,
  image: "python:3.11",
  networkPolicy: {
    defaultAction: "deny",
    egress: [{ action: "allow", target: "pypi.org" }],
  },
});
```

### 3. 运行时 Egress 策略更新

运行时 egress 策略路由取决于 sandbox 的来源。对基于镜像的 sandbox，SDK 解析端口 `18080` 并调用 sidecar 的 `/policy` API。对基于模板的 sandbox（包括恢复自模板的快照），SDK 检测 `OPEN-SANDBOX-ORIGIN: template` 并通过生命周期 `/sandboxes/{sandboxId}/networkpolicy` API 路由策略操作。

修补使用合并语义：
- 传入规则优先于具有相同 `target` 的既有规则。
- 其他目标的既有规则保持不变。
- 在单个修补载荷内，某个 `target` 的第一条规则生效。
- 当前的 `defaultAction` 被保留。

```ts
const policy = await sandbox.getEgressPolicy();

await sandbox.patchEgressRules([
  { action: "allow", target: "www.github.com" },
  { action: "deny", target: "pypi.org" },
]);
```

### 4. Credential Vault

Credential Vault 需要 sandbox 侧的 egress 服务，对基于模板的 sandbox 不可用。它从 egress sidecar 注入出站凭证，同时让真实密钥远离 sandbox 的环境变量、命令、文件和日志。以启用 `credentialProxy` 的方式创建 sandbox，然后通过 `sandbox.credentialVault` 写入凭证和绑定。

```ts
const sandbox = await Sandbox.create({
  connectionConfig: config,
  image: "python:3.11",
  networkPolicy: {
    defaultAction: "deny",
    egress: [{ action: "allow", target: "api.example.com" }],
  },
  credentialProxy: { enabled: true },
});

await sandbox.credentialVault.create({
  credentials: [{ name: "api-token", source: { value: "<token>" } }],
  bindings: [
    {
      name: "api-token",
      match: {
        schemes: ["https"],
        hosts: ["api.example.com"],
        paths: ["/v1/*"],
      },
      auth: { type: "apiKey", name: "x-api-key", credential: "api-token" },
    },
  ],
});
```

认证类型、绑定指南以及 Git/curl 示例参见 [Credential Vault](/guides/credential-vault)。

### 5. 资源清理

在 Node.js 上运行时，`Sandbox` 和 `SandboxManager` 都持有各自作用域内的 HTTP agent，因此你可以安全地复用同一个 `ConnectionConfig`。在完成与 sandbox 或管理 API 的交互后，请调用 `sandbox.close()` / `manager.close()` 以释放底层 agent。

## 浏览器注意事项

::: warning
- SDK 可以在浏览器中运行，但**流式文件上传仅支持 Node**。
- 如果你为 `writeFiles` 传入 `ReadableStream` 或 `AsyncIterable`，浏览器会在上传前回退为**内存缓冲**。
- 原因：浏览器不支持带自定义 boundary 的流式 `multipart/form-data` 请求体（execd 上传 API 所要求）。
:::
