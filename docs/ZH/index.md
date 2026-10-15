---
layout: home

hero:
  name: OpenSandbox
  text: 面向 AI 应用的通用沙箱基础设施
  tagline: 在隔离环境中安全地运行命令、代码解释器、浏览器和开发者工具，并提供多语言 SDK。
  actions:
    - theme: brand
      text: 快速开始
      link: /getting-started/
    - theme: alt
      text: 架构
      link: /architecture/
    - theme: alt
      text: SDK
      link: /sdks/

features:
  - title: 沙箱生命周期管理
    details: 基于 Docker 与 Kubernetes 运行时，对沙箱实例进行创建、监控、续期、暂停/恢复和终止等管理。
  - title: 多语言 SDK
    details: 在标准化的生命周期与执行协议之上，使用 Python、Java/Kotlin、JavaScript/TypeScript、C#/.NET 和 Go SDK 进行构建。
  - title: 沙箱内执行
    details: 执行 shell 命令、管理文件、运行多语言代码解释器、暴露端口，以及流式获取日志与指标。
  - title: 为 AI 工作负载而生
    details: 支持编码智能体、浏览器自动化、远程开发、代码执行和强化学习等场景。
---

## 典型场景

OpenSandbox 已收录于 [CNCF Landscape](https://landscape.cncf.io/?item=orchestration-management--scheduling-orchestration--opensandbox)。

<div class="scenario-grid">
  <a class="scenario-card" href="/examples/claude-code">
    <h3>编码智能体</h3>
    <p>在隔离沙箱中运行 Claude Code、Gemini CLI、Codex 等编码智能体。</p>
  </a>
  <a class="scenario-card" href="/examples/playwright">
    <h3>浏览器自动化</h3>
    <p>在可控的运行时、文件系统和网络环境下执行 Chrome 与 Playwright 工作负载。</p>
  </a>
  <a class="scenario-card" href="/examples/vscode">
    <h3>远程开发</h3>
    <p>托管 VS Code Web 及类桌面环境，支撑安全的云端开发工作流。</p>
  </a>
  <a class="scenario-card" href="/examples/code-interpreter">
    <h3>AI 代码执行</h3>
    <p>安全运行模型生成的代码，流式输出结果，并在可复现的环境中快速迭代。</p>
  </a>
</div>

在 [示例](/examples/) 中查看全部示例。

## 快速安装

::: code-group

```bash [Python SDK]
pip install opensandbox
```

```bash [JavaScript SDK]
npm install @alibaba-group/opensandbox
```

```kotlin [Kotlin (Gradle)]
dependencies {
    implementation("com.alibaba.opensandbox:sandbox:{latest_version}")
}
```

```bash [Go SDK]
go get github.com/alibaba/OpenSandbox/sdks/sandbox/go
```

```bash [C# SDK]
dotnet add package Alibaba.OpenSandbox
```

:::

完整的安装指南请参见 [快速开始](/getting-started/)。
