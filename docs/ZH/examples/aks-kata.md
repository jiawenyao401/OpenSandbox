---
title: AKS + Kata
description: 将 OpenSandbox 部署到启用 Kata VM 隔离的 AKS 集群，并逐步与 Kata 隔离的沙箱交互，涵盖入口、出口和 Credential Vault。
---

# AKS Kata 示例

将 OpenSandbox 部署到启用 `kata-vm-isolation` 的 AKS 集群上，然后逐步与一个 Kata 隔离的沙箱交互。

::: warning 演示凭据
本示例中的 Helm 值使用了**不安全的、众所周知的演示凭据**
（`api_key = "aks-kata-demo-key"` 和固定的 `secureAccess` 签名密钥），以便
本演练在本地 `kubectl port-forward` 下开箱即用。**在将服务器或网关暴露到
本地端口转发之外之前，请务必更换它们。**参见
[安全模型](#security-model)。
:::

## 前提条件

- AKS 集群，且 `kubectl get runtimeclass` 能显示 `kata-vm-isolation`
- Helm 3
- Python 3.10+，并已 `pip install opensandbox requests`
- Azure OpenAI 资源（endpoint + API key）

## 文件

该示例位于 [`examples/aks-kata`](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/aks-kata)：

| 文件 | 用途 |
|------|---------|
| `main.py` | CLI 工具 — 逐步创建、检查和操作沙箱 |
| `controller-values.yaml` | OpenSandbox 控制器的 Helm 值 |
| `server-values.yaml` | 生命周期服务器和入口网关的 Helm 值 |
| `batchsandbox-template-configmap.yaml` | 将 Pod 固定到 Kata 节点的 BatchSandbox 模板 |

## 1. 安装 OpenSandbox

在仓库根目录下执行：

```bash
kubectl create namespace opensandbox-system --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace opensandbox --dry-run=client -o yaml | kubectl apply -f -

kubectl apply -f examples/aks-kata/batchsandbox-template-configmap.yaml

# The controller chart no longer ships the CRDs; install base first on a fresh cluster
helm upgrade --install base ./manifests/charts/base

helm upgrade --install opensandbox-controller ./manifests/charts/controller \
  --namespace opensandbox-system \
  -f examples/aks-kata/controller-values.yaml

helm upgrade --install opensandbox-server ./manifests/charts/server \
  --namespace opensandbox-system \
  -f examples/aks-kata/server-values.yaml
```

等待部署完成：

```bash
kubectl rollout status deploy/opensandbox-controller-manager -n opensandbox-system --timeout=180s
kubectl rollout status deploy/opensandbox-server -n opensandbox-system --timeout=180s
kubectl rollout status deploy/opensandbox-ingress-gateway -n opensandbox-system --timeout=180s
```

## 2. 启动端口转发

打开两个独立的终端：

```bash
# Terminal 1 — lifecycle server
kubectl port-forward -n opensandbox-system svc/opensandbox-server 18080:80

# Terminal 2 — ingress gateway
kubectl port-forward -n opensandbox-system svc/opensandbox-ingress-gateway 28080:80
```

## 3. 设置环境变量

```bash
export SANDBOX_DOMAIN=http://127.0.0.1:18080
export SANDBOX_API_KEY=aks-kata-demo-key

export AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com
export AZURE_OPENAI_API_KEY=your-real-key
# Optional:
# export AZURE_OPENAI_DEPLOYMENT=gpt-4o-mini
```

## 4. 启用暂停/恢复（可选）

暂停会将沙箱 rootfs 提交为 OCI 镜像并推送到镜像仓库。恢复则从该镜像重建沙箱。没有镜像仓库时，`pause` 和 `resume` 步骤会记录一条错误并继续。

以下步骤使用 Azure Container Registry（ACR）。任何 OCI 镜像仓库都可以——完整指南请参见[暂停 / 恢复指南](/guides/pause-resume)。

### 步骤 1：向 kubelet 身份授予 AcrPush 权限

```bash
KUBELET_ID=$(az aks show \
  -g <resource-group> -n <cluster-name> \
  --query "identityProfile.kubeletidentity.clientId" -o tsv)

ACR_ID=$(az acr show --name <acr-name> --query id -o tsv)

az role assignment create --assignee "$KUBELET_ID" --role AcrPush --scope "$ACR_ID"
```

### 步骤 2：创建推送/拉取密钥

```bash
ACR_PASSWORD=$(az acr credential show --name <acr-name> \
  --query "passwords[0].value" -o tsv)

kubectl create secret docker-registry acr-snapshot-push-secret \
  --docker-server=<acr-name>.azurecr.io \
  --docker-username=<acr-name> \
  --docker-password="$ACR_PASSWORD" \
  --namespace=opensandbox
```

### 步骤 3：升级控制器

```bash
helm upgrade opensandbox-controller ./manifests/charts/controller \
  --namespace opensandbox-system \
  --reuse-values \
  --set controller.snapshot.registry=<acr-name>.azurecr.io/opensandbox-snapshots \
  --set controller.snapshot.snapshotPushSecret=acr-snapshot-push-secret \
  --set controller.snapshot.resumePullSecret=acr-snapshot-push-secret

kubectl rollout status deploy/opensandbox-controller-manager \
  -n opensandbox-system --timeout=90s
```

::: tip
`controller.snapshot.containerdSocketPath` 在 chart 中默认为 `""`，这使得控制器无需传递 `--containerd-socket-path` 标志即可使用其内置默认值（`/var/run/containerd/containerd.sock`），除非显式配置。如果你的节点使用非默认的 containerd socket，并且部署的控制器镜像支持该标志，请相应地设置此值。
:::

## 5. 使用 `main.py`

下文所有 `main.py` 命令均在示例目录中运行：

```bash
cd examples/aks-kata
```

### 端到端运行全部步骤

```bash
python3 main.py all
```

这会创建一个沙箱，运行每个演示步骤，并在结束时将其删除。

### 逐个步骤运行

#### 创建沙箱

```bash
python3 main.py create
```

预期输出：

```
Creating Kata-isolated sandbox on AKS...
  OpenSandbox API: http://127.0.0.1:18080
  Sandbox image:   sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0

SANDBOX_ID=96045ee2-6614-435c-9fa8-d4b6f9592598

Use --sandbox-id 96045ee2-6614-435c-9fa8-d4b6f9592598 for subsequent steps.
```

保存该 ID 供后续步骤使用：

```bash
export SANDBOX_ID=96045ee2-6614-435c-9fa8-d4b6f9592598
```

验证 Pod 是否运行在 Kata 节点上：

```bash
kubectl get pod -n opensandbox -o wide
# NODE column should show aks-sandboxagent-*

kubectl get pod -n opensandbox -o jsonpath='{.items[0].spec.runtimeClassName}'
# Expected: kata-vm-isolation

python3 main.py exec --sandbox-id $SANDBOX_ID -c "uname -r"
# Expected: 6.6.137.mshv1-1.azl3 (mshv = Microsoft Hypervisor = Kata VM guest kernel)
```

#### 配置 Credential Vault

```bash
python3 main.py credentials --sandbox-id $SANDBOX_ID
```

预期输出：

```
[credentials] Credential Vault configured.
```

#### 向 LLM 提问

```bash
python3 main.py llm --sandbox-id $SANDBOX_ID -q "What is the capital of France? Reply in one word."
```

预期输出：

```
[llm] Question: What is the capital of France? Reply in one word.
[llm] Model:    gpt-4o-mini
[llm] Answer:   Paris
```

沙箱内只有一个假的 API key——真实的密钥由出口 sidecar 通过 Credential Vault 注入。你可以验证：

```bash
python3 main.py exec --sandbox-id $SANDBOX_ID -c "echo \$AZURE_OPENAI_API_KEY"
# Expected: fake-key-inside-sandbox
```

#### 在沙箱内运行命令

```bash
python3 main.py exec --sandbox-id $SANDBOX_ID -c "uname -a"
# [exec][stdout] Linux ...-0 6.6.137.mshv1-1.azl3 ... x86_64 GNU/Linux

python3 main.py exec --sandbox-id $SANDBOX_ID -c "cat /etc/os-release | head -3"
# [exec][stdout] PRETTY_NAME="Ubuntu 24.04.4 LTS"
# [exec][stdout] NAME="Ubuntu"
# [exec][stdout] VERSION_ID="24.04"

python3 main.py exec --sandbox-id $SANDBOX_ID -c "ls -la /tmp/www/"
# Shows the HTTP server root directory
```

#### 从沙箱 HTTP 服务器获取内容

沙箱运行 `python3 -m http.server 8080` 来服务 `/tmp/www/`。流量通过带 secure-access 头的入口网关转发。

```bash
# Write a file into the sandbox, then fetch it via the gateway
python3 main.py exec --sandbox-id $SANDBOX_ID -c "echo hello > /tmp/www/greeting.txt"
python3 main.py http --sandbox-id $SANDBOX_ID -p /greeting.txt
# [http] GET /greeting.txt -> 200 (6 bytes)
# hello

# Directory listing
python3 main.py http --sandbox-id $SANDBOX_ID -p /
# Shows HTML directory listing

# 404 for missing file
python3 main.py http --sandbox-id $SANDBOX_ID -p /nonexistent.txt
# [http] GET /nonexistent.txt -> 404
```

#### 检查沙箱状态

```bash
python3 main.py status --sandbox-id $SANDBOX_ID
# Sandbox:  96045ee2-6614-435c-9fa8-d4b6f9592598
# State:    Running
# Image:    ...code-interpreter:v1.1.0
```

#### 暂停与恢复

暂停会将容器 rootfs 快照到镜像仓库（1–5 分钟）。恢复则从该快照重建沙箱。需要上述第 4 步。

```bash
python3 main.py pause --sandbox-id $SANDBOX_ID
# [lifecycle] pausing sandbox...
# [lifecycle] sandbox is PAUSED

python3 main.py status --sandbox-id $SANDBOX_ID
# State: Paused

# Pod is gone, but the BatchSandbox CR and snapshot remain
kubectl get pods -n opensandbox
# No resources found
kubectl get sandboxsnapshot -n opensandbox
# Shows snapshot with phase Succeed

python3 main.py resume --sandbox-id $SANDBOX_ID
# [lifecycle] resuming sandbox...
# [setup] sandbox state: Resuming, waiting...
# [lifecycle] state after resume: Running
# [http] GET /index.html -> 200 (filesystem preserved!)

python3 main.py status --sandbox-id $SANDBOX_ID
# State: Running
```

#### 删除沙箱

```bash
python3 main.py delete --sandbox-id $SANDBOX_ID
# [lifecycle] sandbox 96045ee2-... deleted.

kubectl get pods -n opensandbox
# No resources found
```

不带 `-c`、`-q` 或 `-p` 时，`exec`、`llm` 和 `http` 步骤会运行其内置的演示序列。

## 安装了哪些组件

| 组件 | 命名空间 | 用途 |
|-----------|-----------|---------|
| `opensandbox-controller` | `opensandbox-system` | Kubernetes operator — 管理 BatchSandbox CRD、池和快照 |
| `opensandbox-server` | `opensandbox-system` | 生命周期 API — 创建/删除沙箱，代理 SDK 调用 |
| `opensandbox-ingress-gateway` | `opensandbox-system` | 将外部 HTTP 流量经鉴权路由到沙箱 Pod |
| BatchSandbox 模板 | `opensandbox-system` | 通过 `nodeSelector` 将沙箱 Pod 固定到支持 Kata 的 AKS 节点 |

服务器的配置包括：

- `workload_provider = "batchsandbox"`
- `secure_runtime.type = "kata"` / `k8s_runtime_class = "kata-vm-isolation"`
- 出口 sidecar（`dns+nft` 模式）
- 入口网关（带 `secureAccess` 的 `header` 路由）

## 安全模型

::: warning
`server-values.yaml` 附带的是**仅用于演示**的凭据：`api_key =
"aks-kata-demo-key"` 以及固定的 base64 `secureAccess` 签名密钥。它们的存在是为了
让本地端口转发演练无需额外配置即可运行。在将服务器或
网关暴露给 `127.0.0.1` 之外的任何访问之前，请将两者替换为唯一的保密值
（例如一个随机密钥，以及用于签名密钥的
`openssl rand -base64 32`）。
:::

### Kata VM 隔离

每个沙箱都运行在 AKS Kata 节点池（`nodeSelector: kubernetes.azure.com/kata-vm-isolation: "true"`）上专用的 Kata VM（`runtimeClassName: kata-vm-isolation`）内。沙箱看到的是自己的客户机内核，而非宿主机内核。

### 出口隔离

创建请求设置了默认拒绝的出站策略。只有显式允许的主机（Azure OpenAI、pypi.org、files.pythonhosted.org）可以被访问。

### Credential Vault

沙箱内只能看到一个假的 `AZURE_OPENAI_API_KEY`。真实密钥存储在 Credential Vault 中，由出口 sidecar 仅针对发往 `https://<your-resource>.openai.azure.com/openai/*` 的匹配出站请求注入 `api-key` 头。

### 入口网关 + secureAccess

所有到沙箱服务的外部流量都经过入口网关。网关校验 `OpenSandbox-Secure-Access` 令牌，并根据 `OpenSandbox-Ingress-To` 头进行路由。两者都由 SDK 的 `get_endpoint()` 调用返回——调用方无需手动构造。

### 每服务鉴权

即使有人绕过网关直接访问 Pod：
- **execd**（端口 44772）需要自己的访问令牌头
- **出口 sidecar**（端口 18080）需要 `OPENSANDBOX-EGRESS-AUTH` 头
- 只有用户的 HTTP 服务器（端口 8080）没有内置鉴权——这正是 secureAccess 所保护的

## 清理

移除本示例安装的所有内容：

```bash
# 1) Delete any running sandboxes
kubectl delete batchsandbox --all -n opensandbox

# 2) Uninstall Helm releases
helm uninstall opensandbox-server -n opensandbox-system
helm uninstall opensandbox-controller -n opensandbox-system

# 3) Remove the BatchSandbox template
kubectl delete configmap aks-kata-batchsandbox-template -n opensandbox-system

# 4) Remove the snapshot push/pull secret (if created)
kubectl delete secret acr-snapshot-push-secret -n opensandbox --ignore-not-found

# 5) Delete the namespaces
kubectl delete namespace opensandbox
kubectl delete namespace opensandbox-system
```
