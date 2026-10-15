---
title: Playwright
description: 在 OpenSandbox 中使用 Playwright + Chromium 以无头模式访问网页。
---

# Playwright 示例

在 OpenSandbox 中使用 Playwright + Chromium 以无头模式访问网页，抓取标题/正文片段。

## 构建 Playwright 沙箱镜像

示例目录中的 Dockerfile 会构建一个预装 Playwright 和 Chromium 的沙箱镜像：

```shell
cd examples/playwright
docker build -t opensandbox/playwright:latest .
```

该镜像包含：
- Playwright Python 包
- Chromium 浏览器二进制文件
- Node.js 和 npm（用于 Playwright MCP）
- 出于安全考虑的非 root 用户（playwright）

## 启动 OpenSandbox 服务器 [本地]

预先拉取 Playwright 镜像：

```shell
docker pull sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/playwright:latest
```

启动本地 OpenSandbox 服务器：

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
opensandbox-server
```

## 创建并访问 Playwright 沙箱

```shell
# 安装 OpenSandbox 包
uv pip install opensandbox

uv run python examples/playwright/main.py
```

该脚本以无头模式启动 Chromium 访问目标 URL，打印标题/正文片段，并将整页截图保存到沙箱内的 `/home/playwright/screenshot.png`。它还会把该截图下载到本地工作目录，命名为 `./screenshot.png`。默认使用预构建的 Playwright 镜像。

![Playwright 截图](../public/images/playwright-screenshot.png)

## 参考

- [Playwright](https://playwright.dev/)
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/playwright)
