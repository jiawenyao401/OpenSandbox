---
title: Egress Mitmproxy SSE 截断
description: 为什么当 HTTP/1.1 上游以 TCP RST 关闭连接时大型 SSE 主体可能被截断，如何确认，以及如何在网关侧修复。
---

# Egress Mitmproxy：大型 SSE 块在尾部被截断

## 症状

通过 egress sidecar 调用 LLM API 的沙箱有时会收到**被截断的 SSE 流**：大于约 1 MB 的 `data:` 事件在接近末尾处被切断，SDK 报告 JSON 格式错误或连接中断。丢失量不等（几个字节到数百 KB）。

## 根因

对完整数据路径的字节级插桩表明这是**标准的 TCP RST 语义**，而不是 mitmproxy 或 OpenSandbox 的 bug：

1. 上游在其接收缓冲区中仍有未读数据时关闭了连接（例如流式发送响应但不消费客户端请求主体的网关）。内核发送 **TCP RST** 而不是干净的 FIN。
2. 根据 [RFC 793](https://www.rfc-editor.org/rfc/rfc793)，RST 会丢弃接收方内核接收缓冲区中未读的数据——响应的未读尾部在任何应用代码看到它之前就丢失了。
3. mitmproxy 的 h11 解析器随后硬错误（`incomplete chunked read`），流被杀死，客户端连接被关闭。

受控 A/B/C 实验确认了这一点（相同的服务器代码、负载、代理和客户端；仅关闭行为不同）：

| 关闭行为 | mitmproxy TCP 签名 | 客户端结果 |
|---|---|---|
| A. 关闭时请求有未读数据 → 内核 RST | `ConnectionResetError` | 截断，5/5 |
| B. 请求已消费，关闭 → 干净的 FIN | 干净的 EOF | 完整，5/5 |
| C. 请求已消费 + `SO_LINGER=0` 强制 RST（空接收缓冲区） | `ConnectionResetError` | 截断，4/5 |

C 单元是铁证：**仅 RST 就能重现截断，即使没有未读数据；FIN 从不导致截断**。直接慢读的客户端对服务器 A 会丢失整个主体，对服务器 B 则毫无损失，因此丢失纯粹是 RST 落地时尾部还有多少未读数据的函数。HTTP/2 上游和 keep-alive 连接不受影响。

## 复现

[`components/egress/tests/mitm-sse-truncation/`](https://github.com/opensandbox-group/OpenSandbox/blob/main/components/egress/tests/mitm-sse-truncation/run.sh) 中的脚本针对一个合成的 RST 关闭 SSE 上游驱动本地 `mitmdump`（与 egress 透明模式相同的选项）：

```bash
cd components/egress
./tests/mitm-sse-truncation/run.sh            # reproduce the truncation (TLS, 4 MiB, RST close)
./tests/mitm-sse-truncation/run.sh --plain     # controls that must pass
./tests/mitm-sse-truncation/run.sh --delay-close 1
./tests/mitm-sse-truncation/run.sh --read-request
./tests/mitm-sse-truncation/run.sh --probe --keep-workdir   # show the error hook in the mitmdump log
```

需要 `python3`、`openssl`、`mitmdump`（如果不在 `PATH` 中，传 `--mitmdump PATH`）。选项：`--size`、`--iterations`、`--delay-close`、`--plain`、`--read-request`、`--probe`、`--mitmdump`、`--port`、`--upstream-port`、`--workdir`、`--keep-workdir`。

## 修复（网关侧）

mitmproxy 或 OpenSandbox 中没有代码修复——字节已从内核消失。修复属于上游服务器。

**先确认是 RST**（在复现时运行于网关上/前方）：

```bash
tcpdump -ni any 'tcp port 443 and tcp[tcpflags] & (tcp-rst) != 0'   # RST right after the last response byte, no FIN
netstat -s | grep -i reset
# from a client: "Recv failure: Connection reset by peer" / curl: (56) instead of a clean EOF
```

**然后修复关闭行为**——规则：关闭前先消费请求；对于 SSE，尽量不关闭：

- **nginx**：保持 `lingering_close on`（默认）与 `lingering_time 30s` / `lingering_timeout 5s`；如果设置了 `proxy_request_buffering off`，尤其要检查这一点。
- **FastAPI / uvicorn / Starlette**：在流式响应前读取请求主体（`await request.body()`），否则流式响应后紧跟关闭可能让主体未被读取 → RST。
- **Go net/http**：显式排空主体（`io.Copy(io.Discard, r.Body)`）；Go 不会替你排空。绝不要对流式响应使用 `SO_LINGER` 0（`SetLinger(0)`）。
- **通用**：优先使用 keep-alive（以 `0\r\n\r\n` 结束分块主体）；如果必须关闭，先排空接收缓冲区；HTTP/2 连接可避免这一整类失败。

如果网关无法更改：将沙箱到网关的路由改走 HTTP/2，或在它前面加一个消费请求主体并转发 keep-alive 的代理。

## 状态

- mitmproxy 上游：[mitmproxy/mitmproxy#8364](https://github.com/mitmproxy/mitmproxy/issues/8364) — 以非 bug（TCP RST 语义）关闭。
- OpenSandbox 跟踪：[opensandbox-group/OpenSandbox#1461](https://github.com/opensandbox-group/OpenSandbox/issues/1461) — 等待现场确认。

曾考虑并测量过更大的 egress 侧读取缓冲区：它在突发速率下能缩小丢失窗口，但从不改变用户可见的结果（尾随 SSE 事件无论如何都会丢失），因此没有发布补丁。
