---
title: 安全访问
description: 使用静态 Header 令牌或签名的短时效 URL 对访问沙箱端点的入站 HTTP 和 WebSocket 流量进行认证。
---

# 安全访问

安全访问（Secure Access）是 OpenSandbox 针对沙箱端点的入站访问控制。在沙箱上启用后，入口网关（以及 server 代理路径）会要求每个调用方在 HTTP 或 WebSocket 流量到达沙箱之前证明其已获得授权。

它的对应组件：

- [Credential Vault](/guides/credential-vault) —— 保护**出站**请求。
- [安全容器运行时](/guides/secure-container) —— 隔离**工作负载**。

## 要求

- Kubernetes 运行时（Docker 会以 `400` 拒绝 `secureAccess: true`）。
- `[ingress] mode = "gateway"`。
- 签名 URL 还需要 `[ingress.secure_access]` 中至少配置一个签名密钥。

## 工作原理

![Secure Access request flow](../public/images/secure-access.svg)

在**控制平面**上，以 `secureAccess: true` 创建沙箱并调用 `GetEndpoint`，会得到一个该沙箱专属的不透明 `SecureAccessToken`。传入 `?expires=<unix_seconds>` 还会使用 `[ingress.secure_access]` 下的活跃密钥铸造一个签名、有时效的路由令牌。

在**数据平面**上，入口网关按固定顺序评估两种机制：

1. **Header 存在** → 以常数时间比较 `SecureAccessToken`。不匹配立即返回 `401`；**不会**回退到签名 URL 分支。
2. **Header 不存在** → 解析签名令牌，检查 `now ≤ expires`，验证签名。

通过任一检查的请求会在剥离 Header 或签名前缀后转发到上游，工作负载永远不会看到凭据。

## 静态 Header 令牌

`GetEndpoint` 返回一个不透明的 `SecureAccessToken`。客户端以如下方式附加：

```http
OpenSandbox-Secure-Access: <token>
```

SDK 会自动从端点响应中获取该 Header，因此应用代码通常无需接触它。

## 签名的短时效 URL

通过添加 `?expires=<unix_seconds>`（十进制 `uint64` epoch 秒，**不是**毫秒）铸造签名 URL：

```
GET /sandboxes/{sandboxId}/endpoints/{port}?expires=<unix_seconds>
```

返回的路由令牌形如：

```
{sandbox_id}-{port}-{expires_b36}-{signature}
```

其中 `expires_b36` 是 base-36 的秒数，`signature` 是 8 位十六进制数字加上签名所用的 1 字符 `key_id`。网关在三种路由模式下接受它：

| 模式     | 令牌出现的位置 |
|----------|------------------------|
| wildcard | Host：`{token}.<parent>` |
| header   | 沙箱路由 Header 的值 |
| uri      | 路径前缀：`/{sandbox_id}/{port}/{expires_b36}/{signature}/…` |

使用签名 URL 可以发放可分享的、有时效的端点，而无需暴露静态令牌。

## Server 配置

签名 URL 需要签名密钥；静态 Header 令牌不需要。

```toml
# ~/.sandbox.toml
[ingress]
mode = "gateway"

[ingress.gateway]
# Wildcard mode requires a wildcard address (e.g. *.sandbox.example.com);
# header and uri modes take a plain host (e.g. sandbox.example.com).
address = "*.sandbox.example.com"

[ingress.gateway.route]
mode = "wildcard"   # or "header" or "uri"

[ingress.secure_access]
active_key = "a"    # 1 char [0-9a-z], must exist in keys

[[ingress.secure_access.keys]]
key_id = "a"
key = "<base64-encoded-secret>"   # raw base64, no "base64:" prefix
```

`opensandbox-server` Helm chart 在 `server.gateway.secureAccess` 下暴露同样的结构，并将密钥同时注入 server 和入口网关。

为避免密钥材料进入配置文件，server 还接受 `OPENSANDBOX_SECURE_ACCESS_KEYS`（`a=<base64-secret>[,b=...]`）和 `OPENSANDBOX_SECURE_ACCESS_ACTIVE_KEY`（`a`）环境变量，它们会覆盖 TOML 配置块；chart 的 `server.gateway.secureAccess.existingSecret` 选项可以从一个 Secret 提供这些变量。

**密钥轮换。** 先添加新的密钥条目并部署，然后切换 `active_key`。验证时加载所有已配置的密钥，因此旧密钥签发的令牌在其被移除前保持有效 —— 请保留旧条目，直到最长可能的令牌有效期结束。

## 启用方式

在创建时设置 `secureAccess: true`。SDK 等价写法：Python `secure_access=True`，JS/TS `secureAccess: true`，C# `SecureAccess = true`。

```bash
curl -X POST http://localhost:8080/v1/sandboxes \
  -H "Content-Type: application/json" \
  -d '{
    "image": {"uri": "python:3.11"},
    "resourceLimits": {"cpu": "500m", "memory": "512Mi"},
    "entrypoint": ["python", "/app/main.py"],
    "secureAccess": true
  }'
```

## 调用受保护的端点

使用静态 Header 令牌：

```bash
# Fetch endpoint; the response carries OpenSandbox-Secure-Access.
curl -s http://localhost:8080/v1/sandboxes/${SANDBOX_ID}/endpoints/8080

curl -H "OpenSandbox-Secure-Access: ${TOKEN}" \
  https://${SANDBOX_ID}-8080.sandbox.example.com/
```

使用签名 URL（有效期一小时）：

```bash
EXPIRES=$(( $(date +%s) + 3600 ))
curl -s "http://localhost:8080/v1/sandboxes/${SANDBOX_ID}/endpoints/8080?expires=${EXPIRES}"
# → https://{sandbox_id}-8080-{expires_b36}-{signature}.sandbox.example.com/
```

## 错误

| 状态码 | 原因 |
|------|-------|
| `400` | 在 Docker 上使用 `secureAccess: true`；`expires` 或路由令牌格式错误；非法的 `expires_b36` / `port` / `signature`。 |
| `401` | Header 不匹配；签名 URL 已过期；签名错误；未知密钥 ID；需要安全访问但未提供任何凭据。 |

在 `GetEndpoint` 上省略 `expires` 不会返回 `400` —— 它返回未签名的响应，仍附带静态的 `SecureAccessToken`。

## 注意事项

- 执行范围同时覆盖入口网关和 server 代理路径（`/v1/sandboxes/{id}/proxy/...`），包括 HTTP 和 WebSocket。
- 网关在转发前会剥离 `OpenSandbox-Secure-Access` 以及任何签名 URI 前缀。
- 在 URI 模式下，仅当沙箱确实启用了安全访问时，形似 `expires_b36/signature` 的路径段才会被解析为签名前缀；否则完整路径原样转发。
- 安全访问与 execd 访问令牌（`X-EXECD-ACCESS-TOKEN`）和 Credential Vault 相互独立；三者可以同时使用。

## 另请参阅

- [Credential Vault](/guides/credential-vault)
- [安全容器运行时](/guides/secure-container)
- [Ingress 组件](/architecture/network/ingress)
