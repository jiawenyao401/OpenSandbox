---
title: MCP Sandbox Server
description: OpenSandbox MCP Server 将 sandbox 操作作为 MCP 工具暴露给 Claude Code、Cursor 和其他支持 MCP 的客户端。
---

# OpenSandbox MCP Sandbox Server

## 概述

OpenSandbox MCP Server 将 OpenSandbox Python SDK 作为 MCP 工具暴露给 Claude Code、Cursor 和其他支持 MCP 的客户端。它提供聚焦的 sandbox 生命周期管理、命令执行和文本文件操作。

## 安装与启动

### 源码

```bash
cd sdks/mcp/sandbox/python
uv sync
uv run opensandbox-mcp
```

### 软件包

```bash
pip install opensandbox-mcp
opensandbox-mcp
```

### 配置

环境变量：

- `OPEN_SANDBOX_API_KEY`
- `OPEN_SANDBOX_DOMAIN`

CLI 覆盖项：

```bash
opensandbox-mcp --api-key ... --domain ... --protocol https
```

配置字段：

- `api_key`：用于认证的 OpenSandbox API 密钥。
- `domain`：OpenSandbox API 域，例如 `api.opensandbox.io`。
- `protocol`：API 请求使用的 `http` 或 `https`。
- `request_timeout_seconds`：HTTP 请求超时（秒）。
- `transport`：默认 `stdio`，或 `streamable-http`。

### Streamable HTTP

```bash
opensandbox-mcp \
  --transport streamable-http \
  --host 0.0.0.0 \
  --port 8000
```

HTTP 传输默认监听 `127.0.0.1:8000`。当 MCP server 需要被另一个容器访问时，请设置 `--host 0.0.0.0`，并使用 `--port` 选择不同的监听端口。除非容器网络或上游网关能保护暴露的端点，否则请保持默认的环回主机。

## 集成

### Claude Code stdio

```bash
claude mcp add opensandbox-sandbox --transport stdio -- \
  opensandbox-mcp --api-key "$OPEN_SANDBOX_API_KEY" --domain "$OPEN_SANDBOX_DOMAIN"
```

### Claude Code http

```bash
claude mcp add opensandbox-sandbox --transport http http://localhost:8000/mcp
```

### Cursor stdio

```json
{
  "mcpServers": {
    "opensandbox-sandbox": {
      "command": "opensandbox-mcp",
      "args": [
        "--api-key",
        "${OPEN_SANDBOX_API_KEY}",
        "--domain",
        "${OPEN_SANDBOX_DOMAIN}"
      ]
    }
  }
}
```

### Cursor http

```json
{
  "mcpServers": {
    "opensandbox-sandbox": {
      "url": "http://localhost:8000/mcp"
    }
  }
}
```

## 工具

::: info
- 与 sandbox 交互的工具使用由 `sandbox_create` 或 `sandbox_connect` 注册的 ID。`sandbox_list` 列出 server 侧的 sandbox，不依赖既有的本地 sandbox 注册。
- `file_read`/`file_write` 仅支持文本；大文件请使用 `encoding` 和 `range_header`。
:::

MCP server 暴露以下工具集，而非完整的 Python SDK。它目前不为 Client Pool、pool 追踪、快照、模板、Credential Vault 或稳定的远程诊断日志/事件注册工具。在受支持的情况下，这些工作流请使用 [SDK 指南](/sdks/)或 [CLI](/cli/)。

### Sandbox

- `sandbox_create`：创建新 sandbox 并在本地注册
- `sandbox_connect`：附加到既有 sandbox 并在本地注册
- `sandbox_kill`：按 ID 终止 sandbox
- `sandbox_get_info`：按 ID 获取 sandbox 信息
- `sandbox_list`：以可选的 `filter` 对象列出 sandbox
- `sandbox_renew`：延长 sandbox 过期时间
- `sandbox_healthcheck`：检查 sandbox 是否健康
- `sandbox_get_metrics`：获取资源指标
- `sandbox_get_endpoint`：获取某个端口的网络端点

### 命令执行

- `command_run`：在 sandbox 内运行命令
- `command_interrupt`：中断运行中的命令

### 文件系统

- `file_read`：读取文本文件
- `file_write`：写入文本文件
- `file_delete`：删除文件
- `file_search`：按 glob 搜索文件
- `file_create_directories`：创建目录
- `file_delete_directories`：删除目录
- `file_move`：移动/重命名文件或目录
- `file_replace_contents`：替换文件内容

## 最小工作流

1. `sandbox_create` -> 保存 `sandbox_id`。
2. `file_write` 将代码或资产写入 sandbox。
3. `command_run` 执行、安装依赖或启动服务。
4. 如需暴露端口，使用 `sandbox_get_endpoint`。
5. 完成后 `sandbox_kill`。

## 使用示例

以下是一些可以让 LLM 去做的事：

- "创建一个 Python sandbox 并运行一条快速健康检查命令。"
- "把一个 Python 脚本写进 sandbox 并运行它。"
- "下载一个 GitHub 仓库，安装依赖并运行其测试。"
- "生成一个包含伪造销售数据的 CSV 文件并运行一个简单的汇总脚本。"
- "在 8000 端口启动一个微型 web 服务器并返回公共 URL。"
- "构建一个最小 REST API（hello + health）并暴露在 8000 端口。"
- "为 /app 创建 tar.gz 并报告文件大小。"
- "构建一个简单的贪吃蛇游戏并返回可访问它的 web 端点。"
