# OpenSandbox 文档站点

本目录承载 OpenSandbox 的 VitePress 文档站点。

## 本地开发

```bash
nvm use 22
cd docs
pnpm install
pnpm docs:dev
```

## 构建

```bash
nvm use 22
cd docs
pnpm install
pnpm docs:build
```

## 说明

- 站点内容直接维护在 `docs/` 目录下——并非由 monorepo 中的 README 自动生成。
- 内容归属与规范参见 `AGENTS.md` → "Documentation Rules"。
