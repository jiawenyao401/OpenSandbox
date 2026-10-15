---
title: 应用模型补丁
description: 在使用自定义镜像的 OpenSandbox 沙箱中应用 Codex 风格的文件补丁。
---

# 应用模型补丁

OpenSandbox 暴露了文件操作和命令执行能力，但默认的
`execd` 镜像并不包含补丁解释器。`/files/replace` API
执行的是字面字符串替换；它不理解上下文 hunk、
文件创建、删除或移动。

如果你的编码智能体输出的是 OpenAI Codex 使用的补丁格式，可以在自定义沙箱镜像中安装独立的
`apply_patch` 可执行文件，并在沙箱内运行它。
这样可以将补丁方言保留在智能体镜像中，而不是向 OpenSandbox 文件 API
添加特定厂商的行为。

## 构建自定义镜像

以下示例从一个固定的 Codex 版本构建该可执行文件，并将其复制到 OpenSandbox
`execd` 镜像中。在生产环境中，请将 `CODEX_REF` 固定到一个
经过审查的提交上，以确保镜像构建可复现。

```dockerfile
FROM rust:alpine AS apply-patch-builder

ARG CODEX_REF=<reviewed-commit>
RUN apk add --no-cache build-base git musl-dev
WORKDIR /src
RUN git clone --depth 1 https://github.com/openai/codex.git . \
    && git fetch --depth 1 origin ${CODEX_REF} \
    && git checkout ${CODEX_REF}
WORKDIR /src/codex-rs
RUN cargo build --release --locked -p codex-apply-patch

FROM opensandbox/execd:v1.1.0
COPY --from=apply-patch-builder /src/codex-rs/target/release/apply_patch /usr/local/bin/apply_patch
RUN chmod 0755 /usr/local/bin/apply_patch
```

构建并发布镜像，然后在创建沙箱时使用其名称：

```shell
docker build --build-arg CODEX_REF=<reviewed-commit> \
  -t my-registry/opensandbox-execd-with-patch:1 .
docker push my-registry/opensandbox-execd-with-patch:1
```

构建阶段和运行时阶段必须产出兼容的二进制文件。如果你的基础
镜像不是 Alpine，请使用具有匹配 C 库和工具链的构建器。

## 从 Python SDK 应用补丁

将模型输出写入临时文件，然后通过现有的命令 API 调用
`apply_patch`。该可执行文件既接受单个补丁参数，
也接受标准输入上的补丁；使用文件可以避免 shell 引号和参数
长度问题。

在 async 函数中运行以下代码片段。它假设
`/workspace/hello.py` 已存在且包含 `print("hello")`。

```python
import sys

from opensandbox import Sandbox

patch = """*** Begin Patch
*** Update File: /workspace/hello.py
@@
-print("hello")
+print("hello from OpenSandbox")
*** End Patch
"""

async with (await Sandbox.create(image="my-registry/opensandbox-execd-with-patch:1")) as sandbox:
    await sandbox.files.write_file("/tmp/change.patch", patch)
    result = await sandbox.commands.run(
        "cd /workspace && apply_patch < /tmp/change.patch"
    )
    stdout = "".join(message.text for message in result.logs.stdout)
    stderr = "".join(message.text for message in result.logs.stderr)
    print(stdout, end="")
    print(stderr, end="", file=sys.stderr)
    if result.exit_code != 0:
        raise RuntimeError(f"apply_patch failed (exit code: {result.exit_code})")
```

除非补丁使用绝对路径，否则补丁路径相对于命令工作目录解析。
请将工作目录和补丁路径保持在
沙箱工作区内，并将补丁内容视为来自模型的
不可信输入。

## 运维注意事项

- `apply_patch` 可以添加、更新、删除和移动文件，并使用上下文匹配更新
  hunk。
- 一个失败的补丁可能导致先前的文件更改已被提交。重试前请检查命令
  退出状态并检查受影响的文件。
- 默认的 OpenSandbox 镜像保持不变。仅为需要
  此补丁格式的智能体使用自定义镜像。
- 对于已经自行管理编辑工具的智能体，请继续使用常规的文件和命令
  API，而不是安装此可执行文件。

参见 [OpenAPI 规范](/api/)和
[Codex CLI 示例](/examples/codex-cli)了解相关的沙箱搭建方式。
