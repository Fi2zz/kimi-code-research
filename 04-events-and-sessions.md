# 04 · 事件、状态与会话持久化

> 目标：解释 Kimi Code 的"真相层"——事件如何定义、派发、折叠成状态、写入 `wire.jsonl`、在恢复时重放，以及会话如何创建/恢复/分叉/导出。重实现时这一层决定你能不能做撤销、回放、分支和多端一致。

---

## 1. 为什么是事件溯源

普通 agent 只存"消息列表"。Kimi 存的是**事件流**：

```
context.append_message      （消息）
context.append_loop_event   （流式增量、工具调用状态）
permission.set_mode         （权限模式）
environment.set_binding     （环境绑定）
file_history.checkpoint     （撤销点）
task.started / task.terminated
```

状态（当前消息列表、当前权限模式、当前工具表）是事件的**投影**。好处：

- 恢复 = 重放，不需要额外快照格式；
- 撤销 = 回滚到某个 checkpoint；
- 分叉 = 从某一行切一条新分支；
- 多端 = 同一个事件流投影到 TUI/Web/IDE。

代价：事件必须严格版本化、可校验、可迁移。

---

## 2. Event2：事件基类

`packages/agent-core-v2/src/app/event/event2.ts`：

```ts
abstract class Event2<P = Record<string, unknown>> {
  static readonly type: string;
  static readonly durable = false;    // 是否写入 wire
  static readonly observable = false; // 是否发布到事件总线
  static readonly agentDomain = false;// 是否属于某个 agent
  static readonly schema?: ZodType;

  readonly type: string;
  readonly time: number;

  constructor(payload: P, time?: number); // Object.assign + Date.now()
  serialize(): { type: string; time: number; [k: string]: unknown };
}

abstract class AgentEvent2<P extends { agentId: string }> extends Event2<P> {
  static readonly agentDomain = true;
  readonly agentId: string;
}
```

注册表：

```ts
const EVENT2_REGISTRY = new Map<string, Event2Class>();
registerEvent2Class(cls); // 重复 type 报错；durable 必须有 schema
```

反序列化：

```ts
event2FromRecord(cls, record) // 去掉 type/time，用 schema.safeParse 校验，失败返回 undefined
```

### durable 与 observable 正交

| 组合 | 行为 |
| --- | --- |
| durable + observable | 持久化 + 推送（如 `permission.approval.requested`） |
| durable + !observable | 只持久化（如 `context.append_message`） |
| !durable + observable | 只推送（如 `plugin_command.activated`） |
| 都不是 | 纯内部 |

这是重实现时最容易搞混的一点：**"写进 wire"和"前端能看到"是两件事**。

---

## 3. 状态：defineState / replayable / undoable

`packages/agent-core-v2/src/state/state.ts`：

```ts
const permissionModeKey = defineState("permissionMode", (): PermissionMode => "manual")
  .replayable({ schema: z.custom<PermissionMode>() })
  .on(PermissionSetMode, (_state, event) => event.mode);
```

- `defineState(name, initial)`：普通状态，不参与重放。
- `.replayable({ schema, durable?, blobs?, mutableReplay? })`：状态由事件折叠而来；`snapshotExcluded = true`。
- `.on(EventClass, fold)`：
  - 同一事件重复 fold 报 `STATE_DUPLICATE_FOLD`；
  - durable 事件不能折叠进 transient 状态（`STATE_DURABILITY_MISMATCH`）；
  - fold 用 immer draft。
- `.undoable(opts?)`：增加 checkpoint/undo 协议；compaction/clear 会清 checkpoint；`context.undo` 回滚 N 个 checkpoint 或调用自定义 `onUndo`。
- `StateFold = (draft, event, ctx) => S | void`；`ctx` 提供 `silent`、`checkpoint()`、`clearCheckpoints()`、`undoToCheckpoint(n)`、`emit(event)`。

重放时 `silent = true`，这样 fold 里的 checkpoint/emit 不会产生真实副作用。

---

## 4. 状态注册表

`_base/state/stateRegistry.ts`：

```ts
interface StateKey<T> { name: string; initial(): T; snapshotExcluded?: boolean }

class StateRegistry {
  contributeState(key);
  get/set;
  snapshot();
  onDidChange(key);
  onDidChangeAny();
  inspect(); // scope 链
}
```

- 重放 key 必须贡献到 **Agent scope** 的 registry；普通 registry 会抛错。
- Session 也有自己的 `ISessionStateService`。
- `IAgentStateService` 额外提供 `replayableKeys()`、`onDidContributeReplayable`、`onDidWithdrawReplayable`。
- 重放 key 必须在 `restore()` 之前贡献；否则恢复后新增的 key 没有历史。

---

## 5. 派发管线

`state/eventDispatcherService.ts`：

```ts
interface IEventDispatcher {
  dispatch(event): Promise<void>;
  attach(participant);
  attachLate(participant);
  restore(): Promise<void>;
  flush(): Promise<void>;
  restorePhase: "new" | "restoring" | "ready" | "failed";
}
```

`dispatch`：

1. 若 `agentDomain` 且 `event.agentId !== 当前 agent` → 拒绝。
2. 若已在派发中 → 入队，返回 promise（30s 卡住会告警）。
3. 否则同步 `runDispatch`，再 `drainQueue`；drain 深度 > 100 抛 `CycleError`。

`executeEvent(event, silent, replayUndoable?)`：

1. 按 `event.type` 找 state fold、agent model、durable participant。
2. 用 immer `produce` 准备每个 key 的新状态。
3. 提交状态（undoable 会记录前值 checkpoint）。
4. 提交 participant。
5. 若 `durable` → `wire.appendRecord(event.serialize(), dehydrator)`。
6. 若 `observable` → `eventBus.publish(event, agentContext)`。
7. fold 里的 `ctx.emit()` 只入同一队列，因此级联有界。

错误策略：写入失败记录日志并上报，不向调用方抛（避免一个持久化错误打断整个 turn）。

> 风险：状态提交发生在 wire append 之前，因此追加失败会造成"内存状态已更新、持久化日志缺事件"。重启后这部分状态会丢失。重实现不能只复制这一容错策略：应让调用方和会话状态明确感知持久化失败，并阻止继续产生未落盘的副作用，或将会话标记为 degraded。

---

## 6. wire.jsonl：物理格式

`wire/record.ts`：

```ts
const WIRE_PROTOCOL_VERSION = "1.5";
const WIRE_MIN_READER_VERSION = "1.5";

interface WireMetadataRecord {
  type: "metadata";
  protocol_version: string;
  min_protocol_version?: string;
  created_at: number;
}
```

> 注意：这里说的是 **wire 协议 1.5**。CLI `kimi web` 使用的 KAP（REST/WS 协议）是 **1.0.0**，不要混用。二进制核验（26 篇）在 `/api/v1/meta` 看到了 `kimi_agent_protocol_version: "1.0.0"`，并在 bundle 内找到了 `WIRE_PROTOCOL_VERSION` 1.5。

每个 agent 一个文件：

```
<home>/sessions/<workspaceId>/<sessionId>/agents/<agentId>/wire.jsonl
```

- 第一行必须是 metadata；否则 `STORAGE_CORRUPTED`。
- 之后每行一个 `event.serialize()`（`{ type, ...payload, time }`），**没有额外信封**。
- 版本太新：`WIRE_VERSION_TOO_LOW`。
- 迁移链：1.0 → 1.1 → 1.2 → 1.3 → 1.4 → 1.5。
- 追加日志支持撕裂尾修复（`repairWireJournal`）与重写（`rewrite`）。
- 写入需要持有 session lock；否则拒绝。

### 分支记录

```
agent.switched   { branch, base: {branch, line}, reason? }
context.undo     （旧版兼容）
context.undone
```

`parseTree(entries)` 把 `agent.switched` 解析成 `SwitchEdge` 与 `TreeSegment`；`activeChain` 沿 base 递归还原当前分支。分叉时追加三元组并校验落盘。

---

## 7. 恢复与投影

`restore()`：

```
new → restoring → ready | failed
```

1. `resetReplayState`
2. `replayRecords`：
   - 先 `restorable`（可撤销状态/participant）
   - 再 `journal`（全部记录）
   - 每条记录只解析一次；未知/畸形记录报告但不致命
3. `freezeReplayedStates`
4. `rehydrateStates`（媒体 blob → content parts）
5. `hooks.onDidRestore`
6. 可选 drain 队列

恢复后，对话由 `context.append_message` / `context.append_loop_event` 折叠成 `ContextMessage[]`。另有一套 `AgentModel` 投影用于 UI 状态（todo、工具卡等）。

---

## 8. 会话目录与 state.json

```
sessions/<workspaceId>/<sessionId>/
├── state.json
├── logs/kimi-code.log
├── agents/<agentId>/
│   ├── wire.jsonl
│   ├── blobs/
│   ├── file-history/
│   └── tasks/
└── attachments/
```

`state.json`（`SESSION_META_VERSION = 2`）：

```ts
interface SessionMeta {
  id: string;
  version?: number;
  title?: string;
  titleKind?: "replaceable" | "generated" | "custom";
  lastPrompt?: string;
  createdAt: number;
  updatedAt: number;
  archived: boolean;
  archivedAt?: number;
  cwd?: string;
  environmentId?: string;
  environmentCwd?: string;
  forkedFrom?: string;
  agents?: Record<string, AgentMeta>;
  custom?: Record<string, unknown>;
  lastTurnReason?: "completed" | "cancelled" | "failed";
}

interface AgentMeta {
  homedir?: string;
  type?: "main" | "sub" | "independent";
  parentAgentId?: string | null;
  forkedFrom?: string;
  labels?: Record<string, string>;
  swarmItem?: string;
}
```

持久化时额外写 `isCustomTitle`。`state.json` **不是**事件日志，只是元数据 + 索引种子。

---

## 9. session_index 与 minidb

- 旧版：`~/.kimi-code/session_index.jsonl`，逐行 `{ sessionId, sessionDir, workDir }`，删除写 `{ deleted: true }`。
- 新版：`cache/query-store/` 下的 minidb（16 shard，`valueMode: memory`，`fsyncPolicy: everysec`）。
  - collection：`session:g<gen>`、`sessionCounters:g<gen>`；
  - 索引：`updatedAt` 排序、`custom.parent_session_id`；
  - 权威来源仍是逐个 `state.json` 扫描；
  - 索引损坏时清空重建（`QueryStoreRebuiltError`），并回退到目录扫描。
- 脏标记：`sessions/.index-dirty/<sessionId>.<ts>`；扫描缓存：`sessions/.index-cache/scan.json`。

**重实现建议**：先用 SQLite 做索引，保持"索引可重建"这一契约；不要把它当真相。

---

## 10. Agent 生命周期

`session/agentLifecycle/`：

```ts
const MAIN_AGENT_ID = "main";

interface IAgentLifecycleService {
  create(opts?): Promise<AgentContext>;
  fork(source, opts?): Promise<AgentContext>;
  get(agentId): AgentContext | undefined;
  list(filter?): readonly AgentContext[];
  broadcastPermissionMode(mode): void;
  remove(agent): Promise<void>;
  adopt(handle): AgentContext;
}
```

创建流程（简化）：

1. 计算 `agentScope = sessionScope + "/agents/" + agentId`；
2. 建 `AgentContext`（冻结）；
3. `eventBus.activateAgent`；
4. 创建 blob service、`WireService`；
5. `wire.seal()` 写 metadata（新文件）；
6. 创建 Agent scope，seed 上下文；
7. 注册到 roster，写 `state.json.agents`；
8. 触发 `onDidCreate`；
9. 远程环境则派发 `EnvironmentSetBinding`；
10. `dispatcher.restore()` 重放；
11. 绑定 profile/权限，激活工具，`loop.start()`。

Agent id：main 固定 `main`；子代理自动 `agent-0`、`agent-1`…（扫描 roster + `state.json` + 磁盘目录取最大）。

移除：先 `onWillClose` → 取消 prompt/压缩 → 等 loop settle（最多 3s）→ `loop.close()` → `flush()` → 杀 space → dispose handle → 触发 `onDidClose`。

---

## 11. 分叉

两种：

| 类型 | 行为 |
| --- | --- |
| Agent fork | 新建 agent scope；把源 agent 的历史**复制到内存**；不复制 wire |
| Session fork | 复制/slice 会话目录与所有 agent 的 wire；写新 `state.json`；更新索引；`forkedFrom` |

Session 分叉需要计算切割行（`computeForkLine`），处理 compaction 边界、未完成的 turn、legacy undo 对。失败原因：`compaction_boundary` / `insufficient`。

重实现时，先做 session fork（复制目录 + 重写 metadata），再做 agent fork（内存历史）。

---

## 12. 导出

`app/sessionExport/`：

```ts
export(input, options) → { zipPath, entries, sessionDir, manifest }
```

zip 内容：

- 会话目录递归（排除 file-history blob）；
- `manifest.json`：sessionId、exportedAt、version、wireProtocolVersion、os、node、首末活动时间、title、workspaceDir、可选日志；
- 可选 session log / global log / web log；`desktop log` 仅属于上游 Desktop 导出，不是 CLI 必需项。

注意：**没有对应的 zip 导入**。`import-sessions` 是另一件事——导入 Claude Code / Codex 的会话，不是恢复 Kimi 的 zip。

---

## 13. 不变量与坑

1. `wire.jsonl` 是真相；`state.json` 是元数据；索引可重建。
2. durable 事件必须带 schema；改 schema 必须加迁移。
3. 重放顺序：先 undoable，再 journal；否则 checkpoint 会错。
4. 事件 type 全局唯一；重复注册直接报错。
5. agent-domain 事件必须有匹配的 `agentId`，否则被丢弃。
6. 写 wire/state.json 必须持 session lock；并发写会破坏追加日志。
7. 不要在 fold 里做 I/O；fold 可能被 silent 重放多次。
8. 媒体走 blob 引用，不要把 base64 写进 wire。
9. 删除会话要写 tombstone，不能只删目录（索引会复活）。
10. 分叉不是"复制文件"这么简单：要重写 metadata、处理 compaction 边界。

---

## 14. 重实现最小版

1. 定义 `Event2`（type/durable/observable/agentDomain/schema）。
2. 定义 `StateKey` + `replayable().on()`。
3. 实现 `EventDispatcher`：dispatch → fold → append → publish。
4. 实现 `wire.jsonl` 追加 + metadata + 恢复。
5. 实现 `state.json` 读写（原子写）。
6. 实现 session/agent 目录与 id 规则。
7. 再补 undoable checkpoint 与 fork。
8. 最后做索引（SQLite）与导出。

测试重点：

- 写 3 条事件 → 重启 → 状态一致；
- 同一事件重放两次不改变结果（幂等 fold）；
- 未知事件不导致恢复失败；
- 撕裂尾（最后一行不完整）可修复；
- 分叉后两条分支互不影响；
- `state.json` 损坏时索引能回退扫描。

---

## 15. 证据索引

- `packages/agent-core-v2/src/app/event/{event2,eventBus,eventBusService}.ts`
- `packages/agent-core-v2/src/state/{state,eventDispatcher,eventDispatcherService,agentModel}.ts`
- `packages/agent-core-v2/src/_base/state/stateRegistry.ts`
- `packages/agent-core-v2/src/wire/{record,wireService,migration/*,tree/*,repair}.ts`
- `packages/agent-core-v2/src/persistence/**`
- `packages/agent-core-v2/src/session/{sessionContext,sessionMetadata,agentLifecycle}/**`
- `packages/agent-core-v2/src/app/{sessionManager,sessionIndex,sessionExport}/**`
- `packages/minidb/**`
