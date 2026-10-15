---
title: Qwen Code
description: 通过 OpenAI 兼容端点在 OpenSandbox 容器中运行 Qwen Code。
---

# Qwen Code 示例

通过 OpenAI 兼容端点在 OpenSandbox 容器中运行 [Qwen Code](https://github.com/QwenLM/qwen-code)。

## 启动 OpenSandbox 服务器 [本地]

预先拉取 code-interpreter 镜像（包含 Node.js）：

```shell
docker pull sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0

# 使用 docker hub
# docker pull opensandbox/code-interpreter:v1.1.0
```

然后启动本地 OpenSandbox 服务器，stdout 日志将显示在终端中：

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
opensandbox-server
```

## 创建并访问 Qwen 沙箱

```shell
# 安装 OpenSandbox 包
uv pip install opensandbox

# 导出 provider 配置
export API_KEY=your-api-key
export BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
export MODEL_NAME=qwen3-coder-plus

# 运行示例
uv run python examples/qwen-code/main.py
```

该脚本会在运行时安装 Qwen Code（`npm install -g @qwen-code/qwen-code@latest`），在沙箱内写入项目本地的 `.qwen/settings.json`，并以无头模式运行 `qwen -p "Compute 1+1 and reply with only the final number."`。API key 仅通过 `API_KEY` 环境变量注入，不会写入仓库。

## 环境变量

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `SANDBOX_DOMAIN` | `localhost:8080` | 沙箱服务地址 |
| `SANDBOX_API_KEY` | _（本地可选）_ | 若服务器要求认证，则提供 API key |
| `SANDBOX_IMAGE` | `sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0` | 要使用的沙箱镜像 |
| `API_KEY` | _（必需）_ | Qwen Code 使用的 OpenAI 兼容 provider 的 API key |
| `BASE_URL` | `https://dashscope.aliyuncs.com/compatible-mode/v1` | OpenAI 兼容的 base URL |
| `MODEL_NAME` | `qwen3-coder-plus` | Qwen Code 的模型名称 |

## 参考

- [Qwen Code](https://github.com/QwenLM/qwen-code) - 官方仓库
- [Qwen Code 认证](https://qwenlm.github.io/qwen-code-docs/en/users/configuration/auth/) - provider 配置参考
- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/qwen-code)
