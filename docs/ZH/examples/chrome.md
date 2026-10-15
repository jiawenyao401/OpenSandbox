---
title: Chrome
description: 使用 OpenSandbox 运行时运行 Chrome 浏览器，通过 DevTools 进行远程调试。
---

# OpenSandbox 中的 Chrome 浏览器

本示例使用 OpenSandbox 运行时运行 Chrome 浏览器。

该镜像启动一个 VNC 服务器（`Xtigervnc :1`），并以远程调试模式启动 Chromium，调试端口为 `9222`。

## 获取 Chrome 镜像

你可以从源码构建镜像，或从 Docker Hub 拉取。

### 从源码构建

```shell
cd examples/chrome
docker build -t opensandbox/chrome .
```

### 拉取现有镜像

```shell
docker pull opensandbox/chrome:latest

# use acr from china
# docker pull sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/chrome:latest
```

## 启动 OpenSandbox 服务器

启动 OpenSandbox 服务器，并从终端查看 stdout：

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
opensandbox-server
```

## 创建并访问 Chrome 沙箱

构建/拉取上述镜像，然后使用镜像 `opensandbox/chrome:latest` 和一个保持沙箱
存活的 entrypoint（例如 `["/bin/sh", "-c", "sleep infinity"]`，或复用 `tail -f /dev/null`）创建沙箱。确保运行时暴露端口
`5901` 和 `9222` 以供 VNC/DevTools 使用。

```shell
uv pip install opensandbox
uv run python examples/chrome/main.py
```

然后获取 5901/9222 的端点，用 VNC 客户端或 DevTools 连接，例如：

```text
execd daemon running with endpoint='127.0.0.1:48379/proxy/44772'
VNC running with endpoint='127.0.0.1:48379/proxy/5901'
DevTools running with endpoint='127.0.0.1:48379/proxy/9222'/json
```

```text
[ {
   "description": "",
   "devtoolsFrontendUrl": "https://chrome-devtools-frontend.appspot.com/serve_rev/@.../inspector.html?ws=127.0.0.1:52302/devtools/page/...",
   "faviconUrl": "https://www.gstatic.com/images/branding/searchlogo/ico/favicon.ico",
   "id": "2215AF60AC345E4BA6D822389CFC743B",
   "title": "Google",
   "type": "page",
   "url": "https://www.google.com.hk/",
   "webSocketDebuggerUrl": "ws://127.0.0.1:52302/devtools/page/2215AF60AC345E4BA6D822389CFC743B"
} ]
```

你也可以通过 MCP 客户端使用它，更多信息请参阅：
[chrome-devtools-mcp](https://github.com/ChromeDevTools/chrome-devtools-mcp)。

## 参考资料

- [chrome-devtools-mcp](https://github.com/ChromeDevTools/chrome-devtools-mcp)
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/chrome)
