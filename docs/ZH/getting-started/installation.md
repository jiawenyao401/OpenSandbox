---
title: 安装
description: 在所有支持的语言和平台上安装 OpenSandbox 服务端、SDK、CLI 和 MCP 服务器。
---

# 安装

## 服务端

OpenSandbox 服务端是一个基于 FastAPI 的服务，负责管理沙箱生命周期，支持 Docker 和 Kubernetes 运行时。

::: code-group

```bash [pip]
pip install opensandbox-server
```

```bash [uv]
uv pip install opensandbox-server
```

:::

**环境要求：**
- Python 3.10+
- Docker Engine 20.10+（Docker 运行时）或 Kubernetes 1.21.1+（Kubernetes 运行时）
- Linux、macOS，或带 WSL2 的 Windows

服务端配置请参见 [配置](/getting-started/configuration)。

## SDK

### 沙箱 SDK

用于沙箱生命周期管理、命令执行和文件操作的核心 SDK。

::: code-group

```bash [Python]
pip install opensandbox
```

```bash [JavaScript/TypeScript]
npm install @alibaba-group/opensandbox
```

```kotlin [Kotlin/Java (Gradle)]
dependencies {
    implementation(platform("com.alibaba.opensandbox:sandbox-bom:{latest_version}"))
    implementation("com.alibaba.opensandbox:sandbox")
}
```

```xml [Kotlin/Java (Maven)]
<dependency>
    <groupId>com.alibaba.opensandbox</groupId>
    <artifactId>sandbox</artifactId>
    <version>{latest_version}</version>
</dependency>
```

```bash [Go]
go get github.com/alibaba/OpenSandbox/sdks/sandbox/go
```

```bash [C#/.NET]
dotnet add package Alibaba.OpenSandbox
```

:::

SDK 的详细用法请参见 [SDK 文档](/sdks/)。

请为你的应用选择一个已发布的 SDK 版本，并将 JVM 示例中的 `{latest_version}` 替换为该版本；在同时使用 `sandbox`、`sandbox-pool-redis` 等 JVM 模块时建议使用 BOM。发布索引参见 [release index](https://github.com/opensandbox-group/OpenSandbox/releases)。

各语言的功能支持存在差异，请参见[能力矩阵](/sdks/#capability-coverage)、[Client Pool](/guides/client-pool) 和 [SDK 链路追踪](/sdks/observability#pool-warmup-tracing)。

## CLI

`osb` CLI 提供基于终端的沙箱管理能力。

::: code-group

```bash [pip]
pip install opensandbox-cli
```

```bash [uv]
uv tool install opensandbox-cli
```

:::

完整的命令集请参见 [CLI 参考](/cli/)。

## MCP 服务器

MCP 服务器将沙箱操作暴露给支持 MCP 的客户端，如 Claude Code 和 Cursor。

::: code-group

```bash [pip]
pip install opensandbox-mcp
```

```bash [uv]
uv pip install opensandbox-mcp
```

:::

```bash
opensandbox-mcp --domain localhost:8080 --protocol http
```

供 MCP 客户端使用的最小 stdio 配置：

```json
{
  "mcpServers": {
    "opensandbox": {
      "command": "opensandbox-mcp",
      "args": ["--domain", "localhost:8080", "--protocol", "http"]
    }
  }
}
```

各客户端的具体配置请参见 [MCP 文档](/sdks/mcp)。
