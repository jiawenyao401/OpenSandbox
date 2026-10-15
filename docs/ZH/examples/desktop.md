---
title: Desktop（VNC）
description: 在 OpenSandbox 中启动 Xvfb + x11vnc + fluxbox，提供可通过 VNC 访问的桌面环境。
---

# Desktop（VNC）示例

在 OpenSandbox 中启动 Xvfb + x11vnc + fluxbox，提供可通过 VNC 访问的桌面环境。

## 构建 Desktop 沙箱镜像

示例目录中的 Dockerfile 构建一个预装了桌面和 VNC 组件的沙箱镜像：

```shell
cd examples/desktop
docker build -t opensandbox/desktop:latest .
```

该镜像包含：
- Xvfb（虚拟帧缓冲 X 服务器）
- x11vnc（VNC 服务器）
- XFCE 桌面（面板、文件管理器、终端）
- 出于安全考虑的非 root 用户（desktop）

## 启动 OpenSandbox 服务器 [本地]

预先拉取 desktop 镜像：

```shell
docker pull opensandbox/desktop:latest
```

启动本地 OpenSandbox 服务器：

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
opensandbox-server
```

## 创建并访问 Desktop 沙箱

```shell
# Install OpenSandbox package
uv pip install opensandbox

uv run python examples/desktop/main.py
```

该脚本启动桌面组件栈（Xvfb + XFCE + x11vnc），同时启动 noVNC/websockify。它会打印：
- VNC 端点（`endpoint.endpoint`），在启用直接端点模式时供原生 VNC 客户端使用，密码来自 `VNC_PASSWORD`（默认：`opensandbox`）
- 供浏览器使用的 noVNC URL（`/vnc.html?host=...&port=...&path=...`）

沙箱默认保持存活 5 分钟；可提前用 Ctrl+C 中断。默认使用预构建的 desktop 镜像。

### 在 HTTPS 页面中嵌入 noVNC

浏览器会阻止 HTTPS 页面中的 `http://` noVNC iframe。请在 OpenSandbox 服务器前面的
反向代理处终止 TLS，然后让示例使用
该 HTTPS origin 和服务器代理：

```shell
export SANDBOX_DOMAIN=sandbox.example.com
export SANDBOX_PROTOCOL=https
export SANDBOX_USE_SERVER_PROXY=true
export VNC_PASSWORD=opensandbox

uv run python examples/desktop/main.py
```

生成的 URL 对 noVNC 页面及其 WebSocket 使用相同的 HTTPS origin，例如：

```text
https://sandbox.example.com/v1/sandboxes/<sandbox-id>/proxy/6080/vnc.html?host=sandbox.example.com&port=443&path=v1/sandboxes/<sandbox-id>/proxy/6080
```

TLS 反向代理必须将普通 HTTP 请求和 WebSocket 升级
转发到 OpenSandbox 服务器。对于 Nginx，相关设置为：

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 443 ssl;
    server_name sandbox.example.com;
    # Configure ssl_certificate and ssl_certificate_key here.

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
    }
}
```

当 OpenSandbox 代理路由不要求浏览器提供的鉴权头时，这个直接浏览器 URL 可以工作，包括默认的
单租户模式。在多租户模式下，代理路由要求
在初始 noVNC HTTP 请求和 WebSocket
升级上都带有 `OPEN-SANDBOX-API-KEY`。SDK 会在它自己的请求中添加该头，但浏览器导航
或 WebSocket 无法添加它。

对于多租户浏览器访问，请在 OpenSandbox 前面放置一个受信任的、经浏览器鉴权的反向
代理。它必须授权对所请求沙箱的访问、将浏览器身份映射到正确的租户，并为 HTTP 和 WebSocket 流量注入
该租户的 `OPEN-SANDBOX-API-KEY` 头。不要
将 API key 放入 noVNC URL 中，也不要暴露一个向未鉴权流量
添加共享密钥的反向代理。

::: warning
不要在 HTTPS iframe 中暴露直接的 `http://<sandbox-host>:<mapped-port>` URL。该端点没有 TLS 终止，浏览器会将其作为混合内容拒绝。此 SDK 配置对管理 API
和沙箱服务请求使用同一协议，因此只要管理 API 使用 HTTPS，本示例就要求
`SANDBOX_USE_SERVER_PROXY=true`。这样可以将生命周期、execd、noVNC 页面和 WebSocket 流量都保持在 HTTPS 服务器
origin 上，而不是错误地将 HTTPS 发送到直接的沙箱端点。
服务器代理模式不支持原生 VNC 客户端使用的原始 TCP 协议，因此本示例在此模式下省略了原生 VNC 端点。
:::

![Desktop shell](../public/images/desktop-screenshot-shell.jpg)
![noVNC connect](../public/images/desktop-screenshot-connect.jpg)
![noVNC password](../public/images/desktop-screenshot-password.jpg)
![Desktop UI](../public/images/desktop-screenshot-desktop.jpg)

## 环境变量

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `SANDBOX_DOMAIN` | `localhost:8080` | 沙箱服务地址；可以包含 `http://` 或 `https://` 协议头 |
| `SANDBOX_PROTOCOL` | 域名的协议头，否则为 `http` | 当 `SANDBOX_DOMAIN` 没有协议头时使用的协议（`http` 或 `https`） |
| `SANDBOX_USE_SERVER_PROXY` | `false` | 通过 OpenSandbox 服务器路由沙箱服务、noVNC HTTP 和 WebSocket 流量；使用 HTTPS 时必须开启 |
| `SANDBOX_API_KEY` | _（本地可选）_ | 示例专用的 API key；优先于 `OPEN_SANDBOX_API_KEY` |
| `OPEN_SANDBOX_API_KEY` | _（本地可选）_ | `SANDBOX_API_KEY` 未设置时的 SDK 标准 API key 回退 |
| `SANDBOX_IMAGE` | `opensandbox/desktop:latest` | 要使用的沙箱镜像 |
| `VNC_PASSWORD` | `opensandbox` | VNC 访问密码 |

## 参考资料

- [noVNC](https://github.com/novnc/noVNC)
- [x11vnc](https://github.com/LibVNC/x11vnc)
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/desktop)
