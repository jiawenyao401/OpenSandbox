---
title: 快照存储迁移
description: 将 server 快照存储从 SQLite 切换到 PostgreSQL 的迁移指南。
---

# 快照存储迁移指南

Feature：[#1653](https://github.com/opensandbox-group/OpenSandbox/pull/1653)

## 背景

server 将公共快照目录持久化在一个可配置的存储中。默认后端为 SQLite；PostgreSQL 后端面向需要外部持久化的运维人员，按需启用。PostgreSQL 后端**不会**读取既有的 SQLite 数据库，因此将 `store.type = "sqlite"` 切换为 `store.type = "postgresql"` 的 server 会以空的快照目录启动。

在切换存储类型之前，请使用 `migrate-snapshots` 命令将既有的快照记录从 SQLite 复制到 PostgreSQL。

## 开始之前

- 停止 server（或至少停止发起快照请求），以免在读取期间修改源 SQLite 数据库。
- 目标 PostgreSQL 数据库必须可达，且所配置的角色必须能够创建 `snapshots` 表。
- 请针对 server 切换后将使用的同一个目标数据库运行该命令。

## 迁移

先以 dry-run 查看将被复制的内容：

```bash
opensandbox-server migrate-snapshots \
  --from ~/.opensandbox/opensandbox.db \
  --to postgresql://user:password@localhost:5432/opensandbox \
  --dry-run
```

然后执行迁移：

```bash
opensandbox-server migrate-snapshots \
  --from ~/.opensandbox/opensandbox.db \
  --to postgresql://user:password@localhost:5432/opensandbox
```

该命令会打印一份摘要：

```text
Snapshots migrated: total=42, migrated=42, skipped=0
```

## 行为

- id 已在 PostgreSQL 中存在的记录会被跳过，因此该命令可以安全地重复执行。重复运行会报告 `migrated=0` 和 `skipped=42`。
- `--dry-run` 只报告数量而不写入任何内容，包括不创建 PostgreSQL 的 `snapshots` 表，因此可以使用只读的目标凭证运行。
- 源 SQLite 数据库以只读方式打开且从不被修改，因此只读挂载上的备份也可以迁移。
- PostgreSQL 的 schema 仅在真实迁移运行且表尚不存在时创建。
- SQLite 中以 naive UTC 存储的时间戳会以 UTC 的 `TIMESTAMPTZ` 写入。

## 切换 server

迁移完成后，更新 server 配置：

```toml
[store]
type = "postgresql"

[store.postgresql]
# 在生产环境中，请改用 OPENSANDBOX_STORE_POSTGRESQL_DSN 注入 DSN。
dsn = "postgresql://user:password@localhost:5432/opensandbox"
```

重启 server。快照查询和恢复请求现在会读取共享的 PostgreSQL 目录，包括已迁移的记录。

## 验证

- `GET /v1/sandboxes/{id}/snapshots` 返回已迁移的快照记录。
- 使用既有 `snapshotId` 恢复 sandbox 成功。
