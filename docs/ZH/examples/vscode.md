---
title: VS Code
description: 在 OpenSandbox 中启动 code-server（VS Code Web），提供基于浏览器的 IDE 访问。
---

# VS Code 示例

在 OpenSandbox 中启动 code-server（VS Code Web），提供浏览器访问。

## 构建 VS Code 沙箱镜像

示例目录中的 Dockerfile 会构建一个预装 code-server 的沙箱镜像：

```shell
cd examples/vscode
docker build -t opensandbox/vscode:latest .
```

该镜像包含：
- 预装的 code-server（VS Code Web）
- 出于安全考虑的非 root 用户（vscode）
- 位于 `/workspace` 的工作区目录

## 启动 OpenSandbox 服务器 [本地]

预先拉取 VS Code 镜像：

```shell
docker pull sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/vscode:latest
```

启动本地 OpenSandbox 服务器：

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
opensandbox-server
```

## 创建并访问 VS Code 沙箱

```shell
# 安装 OpenSandbox 包
uv pip install opensandbox

uv run python examples/vscode/main.py
```

该脚本会启动 code-server（禁用认证），将其绑定到指定端口并输出可访问的地址。默认使用预构建的 VS Code 镜像。

![VS Code 截图 shell](../public/images/vscode-screenshot-shell.jpg)
![VS Code 截图 vscode](../public/images/vscode-screenshot-vscode.jpg)

## 参考

- [code-server（VS Code Web）](https://github.com/coder/code-server)
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/vscode)
