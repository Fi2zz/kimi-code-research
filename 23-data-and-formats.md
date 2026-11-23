# 23 · 数据目录与文件格式

> 目标：把磁盘上"到底存了什么、什么格式、谁是真相"讲清楚。重实现时，格式一旦定下就很难改，务必先对齐。

---

## 1. 根目录

```
KIMI_CODE_HOME ?? ~/.kimi-code
```

- 目录 `0700`；
- 文件默认 `0600`；
- 可用 `KIMI_CODE_HOME` 覆盖（测试/多实例）。

存储 scope：

```
config: ""          sessions: "sessions"
blobs: "blobs"      store: "store"
logs: "logs"        cache: "cache"
credentials: "credentials"
```

---

## 2. 目录树

```
~/.kimi-code/
├── config.toml                       # 用户配置
├── config.toml.*.bak                 # 备份/迁移
├── mcp.json                          # MCP 服务器
├── projects.json                     # 工作区目录
├── device_id                         # 遥测设备 id（UUID）
├── region                            # mainland-cn / global
├── region.json                       # 区域缓存
├── credentials/                      # 0700
│   ├── kimi-code.json                # OAuth token
│   └── mcp/<key>-{tokens,client,discovery,meta}.json
├── oauth/                            # 旧版 token 路径
├── sessions/
│   └── <workspaceId>/<sessionId>/
│       ├── state.json
│       ├── upcoming-goals.json
│       ├── logs/kimi-code.log
│       └── agents/<agentId>/
│           ├── wire.jsonl
│           ├── blobs/
│           ├── file-history/
│           └── tasks/<taskId>/output.log
├── blobs/                            # App 级 blob
├── store/                            # App 存储
├── cache/query-store/                # minidb 会话索引（16 shard）
├── telemetry/failed_*.jsonl          # 失败重试队列（7 天）
├── plugins/
│   ├── installed.json
│   ├── availability.json (+ .lock)
│   ├── managed/<id>/
│   └── cache/archives/<sha256>.zip
├── server/
│   ├── server.token                  # 0600
│   ├── instances/<serverId>.json
│   └── rc.json                       # 远程控制锁
├── search-index/
├── locks/sessions/
└── bin/                              # rg / fd
```

---

## 3. workspaceId

| 类型 | 规则 |
| --- | --- |
| 托管 | `wm_<uuid>` |
| 旧版/按目录 | `wd_<slug>_<sha256(path)[:12]>` |

持久化 scope：`sessions/<workspaceId>`。
会话路径：`sessions/<workspaceId>/<sessionId>`。
Agent 路径：`<session>/agents/<agentId>`。

---

## 4. state.json

`SESSION_META_VERSION = 2`：

```json
{
  "id": "session_...",
  "version": 2,
  "title": "...",
  "titleKind": "replaceable|generated|custom",
  "lastPrompt": "...",
  "createdAt": 1783559908844,
  "updatedAt": 1783559908844,
  "archived": false,
  "archivedAt": null,
  "cwd": "/path",
  "environmentId": "local",
  "environmentCwd": "/path",
  "forkedFrom": null,
  "agents": {
    "main": {
      "homedir": "...",
      "type": "main",
      "parentAgentId": null,
      "labels": {},
      "swarmItem": null
    }
  },
  "custom": {},
  "lastTurnReason": "completed"
}
```

持久化时额外写 `isCustomTitle`。
兼容旧字段：`workDir`、`titleSource`、`customTitle`。
时间接受 ISO 字符串 → epoch ms。

**state.json 不是事件日志**：它是元数据 + 索引种子。

---

## 5. wire.jsonl

### 5.1 行格式

第一行：

```json
{"type":"metadata","protocol_version":"1.5","min_protocol_version":"1.5","created_at":1783559908844}
```

之后每行一个事件：

```json
{"type":"context.append_message","agentId":"main","message":{...},"time":1783559908844}
```

- 无额外信封；
- `durable` 事件才写；
- 未知/畸形记录在恢复时报告但不致命；
- 尾部撕裂可修复；
- 需要 session lock 才能写。

### 5.2 分支记录

```json
{"type":"agent.switched","branch":"b1","base":{"branch":"main","line":123},"reason":"fork"}
{"type":"context.undo"}
{"type":"context.undone"}
```

### 5.3 版本迁移

```
1.0 → 1.1 → 1.2 → 1.3 → 1.4 → 1.5
```

版本过新 → `WIRE_VERSION_TOO_LOW`。

---

## 6. 会话索引

### 6.1 旧版

`session_index.jsonl`：

```json
{"sessionId":"session_...","sessionDir":"/...","workDir":"/..."}
{"sessionId":"session_...","deleted":true}
```

只是发现日志，不是真相。

### 6.2 新版

`cache/query-store/` 下的 minidb：

- `ClusterDb`，16 shard；
- `valueCodec: json`、`valueMode: memory`、`fsyncPolicy: everysec`；
- collection：`session:g<gen>`、`sessionCounters:g<gen>`；
- 索引：`updatedAt` 排序、`custom.parent_session_id`；
- manifest checkpoint `sessionIndex`，`SCHEMA_VERSION = 3`；
- 权威扫描：逐个 `state.json`，缓存 `(mtime,size)` 于 `sessions/.index-cache/scan.json`；
- 脏标记：`sessions/.index-dirty/<sessionId>.<ts>`；
- 损坏 → `wipeCluster` + 重建，`QueryStoreRebuiltError`；
- 回退：目录扫描。

---

## 7. working directory 相关

`projects.json`：

```json
{
  "schema_version": 2,
  "projects": [...],
  "deleted_project_ids": [...],
  "idempotency_keys": [...]
}
```

- 原子写 + `.projects-write-lock`。

---

## 8. 凭据

`credentials/kimi-code.json`：

```json
{
  "access_token": "...",
  "refresh_token": "...",
  "expires_at": 1783559908844,
  "scope": "...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

- 目录 `0700`，文件 `0600`；
- 原子写（tmp → fsync → rename）；
- MCP 凭据见 11 篇。

---

## 9. 日志

- 全局：`logs/kimi-code.log`，默认 6 MiB ×5；
- 会话：`sessions/.../logs/kimi-code.log`，5 MiB ×3；
- 级别 `KIMI_LOG_LEVEL`；
- 轮转参数：
  ```
  KIMI_LOG_GLOBAL_MAX_BYTES / KIMI_LOG_GLOBAL_FILES
  KIMI_LOG_SESSION_MAX_BYTES / KIMI_LOG_SESSION_FILES
  ```

---

## 10. 文件历史与 blob

- 前缀 `file-history`；
- 内容寻址：`<hash>@v<n>`；
- 每个 turn 前后 checkpoint；
- `blobs/` 存媒体，wire 里只存引用；
- 导出时排除 file-history blob。

---

## 11. 服务器文件

`server/server.token`：

- base64url(32 random bytes)；
- `0600`；
- KAP 鉴权用。

`server/instances/<id>.json`：

```json
{ "pid", "host", "port", "startedAt", "version", "product" }
```

`server/rc.json`：远程控制单实例锁。

---

## 12. 导出 zip

`manifest.json`：

```json
{
  "sessionId": "...",
  "exportedAt": 1783559908844,
  "kimiCodeVersion": "2.7.2",
  "wireProtocolVersion": "1.5",
  "os": "darwin",
  "nodejsVersion": "24.15.0",
  "sessionFirstActivity": 1783559908844,
  "sessionLastActivity": 1783559908844,
  "title": "...",
  "workspaceDir": "/path",
  "logPaths": [...],
  "desktopVersion": "...",
  "installSource": "...",
  "shellEnv": {...}
}
```

zip 内：会话目录（排除 file-history blob）+ manifest + 可选日志。

**没有 zip 导入**；`import-sessions` 是 Claude Code/Codex 迁移。

---

## 13. 不变量

1. `wire.jsonl` 是对话真相；`state.json` 是元数据。
2. 索引可重建，不能成为唯一来源。
3. 目录 `0700`、敏感文件 `0600`。
4. 写入必须原子（tmp+rename+fsync）。
5. 事件 schema 变更必须迁移。
6. 媒体进 blob，不进 wire。
7. 删除会话写 tombstone。
8. 不要在日志/遥测里写 token/密钥。
9. 文件名/路径必须可跨平台（避免 `:` 等）。
10. 同一目录只允许一个写者（session lock）。

---

## 14. 重实现最小版

1. `~/.kimi-code` + `sessions/<ws>/<sid>`。
2. `state.json` 原子读写。
3. `wire.jsonl` 追加 + metadata + 恢复。
4. blob 目录 + 引用。
5. 日志轮转。
6. SQLite 索引（可重建）。
7. 导出 zip + manifest。
8. 最后加 file-history 与插件/服务文件。

测试重点：

- 重启后 wire 恢复；
- state.json 损坏不丢会话（索引回退扫描）；
- 撕裂尾修复；
- 权限位正确；
- 导出 zip 可解压且 manifest 完整；
- 删除后索引不复活。

---

## 15. 证据索引

- `packages/agent-core-v2/src/app/bootstrap/{bootstrap,bootstrapService}.ts`
- `packages/agent-core-v2/src/wire/{record,wireService,migration/*,tree/*}.ts`
- `packages/agent-core-v2/src/session/sessionMetadata/*`
- `packages/agent-core-v2/src/app/sessionIndex/*`、`app/workspace/workspaceAlias.ts`
- `packages/agent-core-v2/src/persistence/**`、`packages/minidb/**`
- `packages/agent-core-v2/src/app/sessionExport/*`
- `packages/oauth/src/storage.ts`
- `packages/kap-server/src/services/auth/*`
- `docs/en/configuration/data-locations.md`
