---
title: Code Interpreter
description: 使用 Code Interpreter SDK 在 OpenSandbox 中运行 Python 代码的完整演示。
---

# Code Interpreter 沙箱

使用 Code Interpreter SDK 运行 Python 代码的完整演示。

## 获取 Code Interpreter 镜像

::: info 镜像来源
容器镜像的来源和环境定义维护在 [opensandbox-group/sandbox-images](https://github.com/opensandbox-group/sandbox-images) 中。
:::

从镜像仓库拉取预构建镜像：

```shell
docker pull sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0

# use docker hub
# docker pull opensandbox/code-interpreter:v1.1.0
```

## 启动 OpenSandbox 服务器 [本地]

启动本地 OpenSandbox 服务器：

```shell
uv pip install opensandbox-server
opensandbox-server init-config ~/.sandbox.toml --example docker
opensandbox-server
```

## 创建并访问 Code Interpreter 沙箱

```shell
# Install OpenSandbox packages
uv pip install opensandbox opensandbox-code-interpreter

# Run the example (requires SANDBOX_DOMAIN / SANDBOX_API_KEY)
uv run python examples/code-interpreter/main.py
```

该脚本创建一个 Sandbox + CodeInterpreter，运行一段 Python 代码并打印 stdout/结果，然后终止远程实例。即使解释器初始化或代码执行抛出异常，也会在 `finally` 块中尝试清理。

## 环境变量

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `SANDBOX_DOMAIN` | `localhost:8080` | 沙箱服务地址 |
| `SANDBOX_API_KEY` | _（可选）_ | 服务器需要鉴权时使用的 API key |
| `SANDBOX_IMAGE` | `sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0` | 要使用的沙箱镜像 |

## 示例输出

```text
=== Python example ===
[Python stdout] Hello from Python!

[Python result] {'py': '3.14.2', 'sum': 4}

=== Java example ===
[Java stdout] Hello from Java!

[Java stdout] 2 + 3 = 5

[Java result] 5

=== Go example ===
[Go stdout] Hello from Go!
3 + 4 = 7


=== TypeScript example ===
[TypeScript stdout] Hello from TypeScript!

[TypeScript stdout] sum = 6
```

## 来自池的 Code Interpreter 沙箱

### 启动 OpenSandbox 服务器 [k8s]

安装 k8s OpenSandbox operator，并创建一个池：

```yaml
apiVersion: sandbox.opensandbox.io/v1alpha1
kind: Pool
metadata:
  labels:
    app.kubernetes.io/name: sandbox-k8s
    app.kubernetes.io/managed-by: kustomize
  name: pool-sample
  namespace: opensandbox
spec:
  template:
    metadata:
      labels:
        app: example
    spec:
      volumes:
        - name: sandbox-storage
          emptyDir: { }
        - name: opensandbox-bin
          emptyDir: { }
      initContainers:
        - name: task-executor-installer
          image: sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/task-executor:v0.1.0
          command: [ "/bin/sh", "-c" ]
          args:
            - |
              cp /workspace/server /opt/opensandbox/task-executor && 
              chmod +x /opt/opensandbox/task-executor
          volumeMounts:
            - name: opensandbox-bin
              mountPath: /opt/opensandbox
        - name: execd-installer
          image: sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/execd:v1.1.0
          command: [ "/bin/sh", "-c" ]
          args:
            - |
              cp ./execd /opt/opensandbox/execd && 
              cp ./bootstrap.sh /opt/opensandbox/bootstrap.sh &&
              chmod +x /opt/opensandbox/execd &&
              chmod +x /opt/opensandbox/bootstrap.sh
          volumeMounts:
            - name: opensandbox-bin
              mountPath: /opt/opensandbox
      containers:
        - name: sandbox
          image: sandbox-registry.cn-zhangjiakou.cr.aliyuncs.com/opensandbox/code-interpreter:v1.1.0
          command:
          - "/bin/sh"
          - "-c"
          - |
            /opt/opensandbox/task-executor \
              -listen-addr=0.0.0.0:5758 \
              -log-dir=/tmp
          env:
          - name: SANDBOX_MAIN_CONTAINER
            value: main
          - name: EXECD_ENVS
            value: /opt/opensandbox/.env
          - name: EXECD
            value: /opt/opensandbox/execd
          volumeMounts:
            - name: sandbox-storage
              mountPath: /var/lib/sandbox
            - name: opensandbox-bin
              mountPath: /opt/opensandbox
      tolerations:
        - operator: "Exists"
  capacitySpec:
    bufferMax: 3
    bufferMin: 1
    poolMax: 5
    poolMin: 0
```

#### 池的 entrypoint 注入是如何工作的

生命周期 API 从池中分配一个已在运行的 Pod，因此它不会替换该 Pod 的 `command`、`args` 或 `env`。当创建请求提供 `entrypoint` 或环境变量时，服务器会将它们记录到 `BatchSandbox.spec.taskTemplate` 中。然后，控制器将任务发送到所分配 Pod 的 IP 上的端口 `5758`。

池模板必须提供该执行路径的所有组成部分：

- 安装并运行 task-executor，监听 `0.0.0.0:5758`。请显式设置其
  `-log-dir`，以便故障排查路径是确定的；本示例
  写入 `/tmp/task-executor.log`。
- 在 Pod 启动之前，将 execd 和 `bootstrap.sh` 安装到共享卷中。
- 将 `bootstrap.sh` 保持在 `/opt/opensandbox/bootstrap.sh`。服务器生成的任务会调用这个确切路径。只有当 task-executor 环境相应地设置了 `EXECD` 时，execd 二进制文件才可以使用其他路径。
- 分配之后通过 `bootstrap.sh` 启动 execd，以便 `EXECD_ACCESS_TOKEN` 等请求特定的值可用。出于这个原因，上面的示例让 task-executor 作为热 Pod 的前台进程。

因此，Pod YAML 继续显示池模板是符合预期的。请改为检查 `BatchSandbox` 资源和 task-executor：

```shell
# Confirm that the server injected the requested process and environment.
kubectl get batchsandbox <sandbox-name> -n <namespace> \
  -o jsonpath='{.spec.taskTemplate}{"\n"}'

# Find the allocated Pod. The annotation value contains a JSON `pods` array.
kubectl get batchsandbox <sandbox-name> -n <namespace> \
  -o jsonpath='{.metadata.annotations.sandbox\.opensandbox\.io/alloc-status}{"\n"}'

# Replace <pool-pod> with the first Pod name from that array.
kubectl exec <pool-pod> -n <namespace> -- \
  sh -c 'test -x /opt/opensandbox/task-executor && test -x /opt/opensandbox/bootstrap.sh'
kubectl exec <pool-pod> -n <namespace> -- \
  tail -n 100 /tmp/task-executor.log

# Check the executor health endpoint from a second terminal while this runs.
kubectl port-forward pod/<pool-pod> -n <namespace> 5758:5758
curl http://127.0.0.1:5758/health
curl http://127.0.0.1:5758/getTasks

# The lifecycle server uses <sandbox-name>-0 as the task name. Check the task's
# captured output (adjust the path if task-executor uses a custom data directory).
kubectl exec <pool-pod> -n <namespace> -- \
  sh -c 'tail -n 100 /var/lib/sandbox/tasks/<sandbox-name>-0/stdout.log; tail -n 100 /var/lib/sandbox/tasks/<sandbox-name>-0/stderr.log'

# Check controller logs for delivery failures between the controller and port 5758.
kubectl logs -n opensandbox-system -l control-plane=controller-manager --tail=100
```

如果 `taskTemplate` 存在但健康检查无法访问端口 `5758`，请验证 task-executor 已安装并保持运行。生成的任务会有意在后台启动 `bootstrap.sh`，因此即使 `bootstrap.sh` 缺失或所请求的 entrypoint 稍后失败，其包装器也可能报告成功。对于这些失败，不要仅依赖 `taskFailed` 或 `taskLastErrorMessage`；请检查任务捕获的 `stderr.log` 和 `stdout.log`，然后直接验证 execd 或应用程序进程。

启动 k8s OpenSandbox 服务器：

```shell
uv pip install opensandbox-server

# replace with your k8s cluster config, kubeconfig etc.
opensandbox-server init-config ~/.sandbox.toml --example k8s
curl -o ~/batchsandbox-template.yaml https://raw.githubusercontent.com/opensandbox-group/OpenSandbox/main/server/opensandbox_server/examples/example.batchsandbox-template.yaml

opensandbox-server
```

### 创建并访问 Code Interpreter 沙箱（池）

```shell
# Install OpenSandbox packages
uv pip install opensandbox opensandbox-code-interpreter

# Run the example (requires SANDBOX_DOMAIN / SANDBOX_API_KEY)
uv run python examples/code-interpreter/main_use_pool.py
```

### 池示例输出

```text
=== Verify Environment Variable ===
[ENV Check] TEST_ENV value: test

[ENV Result] 'test'

=== Java example ===
[Java stdout] Hello from Java!

[Java stdout] 2 + 3 = 5

[Java result] 5

=== Go example ===
[Go stdout] Hello from Go!
3 + 4 = 7


=== TypeScript example ===
[TypeScript stdout] Hello from TypeScript!

[TypeScript stdout] sum = 6
```

## 参考资料

- [GitHub 上的源代码](https://github.com/opensandbox-group/OpenSandbox/tree/main/examples/code-interpreter)
- [Code Interpreter 镜像仓库](https://github.com/opensandbox-group/sandbox-images)
