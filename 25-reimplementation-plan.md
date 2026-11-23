# 25 · 重新实现蓝图

> 目标：把前 24 篇的结论收敛成一份可执行的复刻计划：边界、接口、里程碑、验收测试、风险与工作量。  
> 这不是实现代码，而是"照着做就能重建"的路线图。

---

## 1. 目标与非目标

### 目标

- 与 Kimi Code 同构：事件溯源会话、工具注册表、权限策略链、危险命令分析、多表面。
- 先跑通"单机编码闭环"，再扩 MCP/子代理/远程。
- 每个里程碑都有可运行的验收。

### 非目标（第一版不做）

- Tower（仓库级多 mission 协作）。
- 桌面/VS Code 外壳。
- 单文件 SEA 分发（放到最后）。
- 自研 bash 解析器（可先用官方 WASM，再替换）。
- 自研 minidb（先用 SQLite）。

---

## 2. 技术栈（建议）

| 关注点 | 选择 |
| --- | --- |
| 运行时 | Node 24.15+（SEA）；开发 Node ≥22.19 |
| 语言 | TypeScript strict, ESM |
| 包管理 | pnpm workspaces |
| 构建 | tsc → tsdown；SEA 用 postject |
| Schema | zod |
| 状态 | immer |
| Glob | picomatch |
| CLI | commander |
| TUI | pi-tui（或 Ink） |
| Server | Fastify + ws |
| Web | Vue 3 + Vite |
| MCP | @modelcontextprotocol/sdk |
| ACP | @agentclientprotocol/sdk |
| 存储 | SQLite（better-sqlite3）+ JSONL 事件日志 |
| 测试 | Vitest + v8 coverage |

---

## 3. 包布局与依赖

```
apps/
  cli/            → 依赖 agent-core, tui
  server/         → 依赖 agent-core
packages/
  core/           → 无依赖：loop, llm, tool executor, store
  agent-core/     → 依赖 core：DI, sessions, events, tools, permissions, mcp
  tui/            → 依赖 agent-core
  sdk/            → 依赖 agent-core（可选）
docs/
```

规则：

- `core` 不 import `agent-core`；
- `agent-core` 不 import `tui`/`server`；
- 所有 I/O 通过接口（fs/process/clock/log），便于测试。

---

## 4. 核心接口（先定这些）

### 4.1 Event

```ts
abstract class Event2<P> {
  static type: string;
  static durable: boolean;
  static observable: boolean;
  static agentDomain: boolean;
  static schema?: ZodType;
  readonly type: string;
  readonly time: number;
  constructor(payload: P, time?: number);
  serialize(): Record<string, unknown> & { type: string; time: number };
}
```

### 4.2 State

```ts
interface StateKey<T> { name: string; initial(): T; snapshotExcluded?: boolean }
defineState<T>(name, initial): StateKeyBuilder<T>;
StateKeyBuilder.replayable({schema, durable?, blobs?}): ReplayableStateKey<T>;
ReplayableStateKey.on(EventClass, fold): this;
ReplayableStateKey.undoable(opts?): this;
```

### 4.3 Dispatcher

```ts
interface EventDispatcher {
  dispatch(event: Event2): Promise<void>;
  restore(): Promise<void>;
  flush(): Promise<void>;
  restorePhase: "new" | "restoring" | "ready" | "failed";
}
```

### 4.4 Wire

```ts
interface Wire {
  seal(): Promise<void>;
  appendRecord(record, dehydrate?): Promise<void>;
  readRestoreChains(): Promise<{restorable, journal}>;
  switchBranch(input): Promise<void>;
}
```

### 4.5 Tool

```ts
interface ExecutableTool<I> {
  name: string;
  description: string;
  parameters: Record<string, unknown>;
  resolveExecution(input: I): ToolExecution | Promise<ToolExecution>;
}

interface RunnableToolExecution {
  description?: string;
  display?: ToolInputDisplay;
  approvalRule: string;
  matchesRule?(argPattern: string): boolean;
  accesses?: ToolAccesses;
  execute(ctx: ExecToolContext): Promise<ToolResult>;
}
```

### 4.6 Permission

```ts
interface PermissionPolicy {
  name: string;
  evaluate(ctx: PermissionContext): PermissionResult | undefined | Promise<...>;
}

type PermissionResult =
  | {kind:"approve"; reason?}
  | {kind:"deny"; reason?; message?}
  | {kind:"ask"; reason?; resolveApproval?; resolveNonInteractive?};
```

### 4.7 Loop

```ts
interface LoopRequester {
  generate(req, opts): Promise<LlmOutcome>;
}
interface ToolExecutor {
  execute(call, ctx): ToolHandler;
}
interface Loop {
  submit(msg): Promise<void>;
  notify(msg): Promise<void>;
  abort(reason?): Promise<void>;
  continue(opts?): Promise<void>;
}
```

### 4.8 Environment

```ts
interface Environment {
  identity: {environmentId: string; generation: string};
  capabilities: Set<"fs"|"process">;
  fs?: HostFileSystem;
  process?: HostProcessService;
  status: EnvironmentStatus;
  whenReady(): Promise<void>;
}
interface EnvironmentLease { environment: Environment; track(r): r; dispose(): void }
```

---

## 5. 里程碑

### M0 · 骨架（1–2 天）

- pnpm 工作区、tsconfig、vitest、lint。
- `core`/`agent-core` 空包可编译。
- 验收：`pnpm typecheck && pnpm test` 通过。

### M1 · 配置 + 日志 + 存储（3–5 天）

- section 注册 + zod + 默认值 + 文件读取。
- `~/.kimi-code` 目录、权限位、原子写。
- JSONL 日志 + 轮转。
- 验收：配置文件读写保留注释；权限位正确；日志轮转。

### M2 · 事件 + 会话（5–7 天）

- Event2、StateKey、Dispatcher（fold + append）。
- `wire.jsonl` metadata/append/restore。
- `state.json` + 目录布局。
- 验收：写 3 条事件 → 重启 → 状态一致；未知事件不崩；撕裂尾修复。

### M3 · LLM + 循环（7–10 天）

- 统一 Message/ContentPart/ToolCall/Usage。
- OpenAI chat 协议（先非流式，再流式）。
- turn/step 循环、tool batch（先串行）、maxSteps、abort。
- 验收：两步对话（工具后回答）；abort 后可继续；空响应重试。

### M4 · 工具框架 + 5 个核心工具（5–7 天）

- Tool contract/registry/executor/truncation/accesses。
- `Read/Write/Edit/Grep/Glob`。
- 验收：参数校验；大输出落盘；写前必须读；敏感文件硬阻断。

### M5 · Bash + 危险命令（5–7 天）

- `Bash`（前台/后台、超时、kill 策略）。
- 危险命令分析（先用黑名单 + 包装递归，再换解析器）。
- 验收：第 15 篇的测试向量。

### M6 · 权限链 + 审批（5–7 天）

- 3 模式 + 13 条策略 + 规则引擎。
- 敏感/ Git 控制/ Git cwd 策略。
- 审批事件 + session 缓存 + 非交互 deny。
- 验收：第 14 篇场景表；顺序/短路测试。

### M7 · 子代理 + 任务（4–6 天）

- `Agent`、profile、allowlist、超时、partial。
- `TaskList/TaskOutput/TaskStop/WaitFor` + 后台通知。
- 验收：隔离；explore 只读；超时 partial；后台通知。

### M8 · MCP + Skills + 插件（5–8 天）

- MCP stdio + 命名 + 状态机 + deferred。
- Skills 发现/解析/调用/参数。
- 插件清单 + 安装 + 贡献（先 skills/MCP）。
- 验收：三层配置；工具冲突；skill 覆盖；插件路径逃逸拒绝。

### M9 · 表面（TUI + KAP）（7–10 天）

- TUI：输入、流、审批、状态栏。
- KAP：会话 CRUD、prompt、审批、WS、token/host/origin。
- ACP：initialize/session/prompt/审批。
- 验收：第 21 篇鉴权与能力矩阵。

### M10 · 打包（3–5 天）

- tsdown + SEA + 资源 manifest。
- 签名/校验（macOS 可选）。
- 验收：无 Node 环境可启动；篡改资源拒绝。

### 可选 M11 · 动态工具

- `select_tools` + deferred + 公告。
- 验收：deferred 不出现；加载后可调用；恢复后状态在。

### 可选 M12 · Swarm / Tower

- Swarm 优先；Tower 单独排期。

**总量估算**：单人全职约 8–12 周到 M10；加 MCP/Skills/插件到 12–16 周；Tower 再加 4–6 周。

---

## 6. 任务分解（可勾选）

> 每个任务应能独立测试、独立提交。

### 基础设施

- [ ] T1 `ServiceCollection` + `createDecorator` + 构造注入。
- [ ] T2 三层 scope + ledger + dispose。
- [ ] T3 配置 section 注册 + zod。
- [ ] T4 原子文件写 + 权限位。
- [ ] T5 JSONL 日志 + 轮转。

### 事件/会话

- [ ] T6 `Event2` + registry + schema。
- [ ] T7 `StateKey` + replayable + fold。
- [ ] T8 Dispatcher：fold/append/publish。
- [ ] T9 wire metadata/append/restore。
- [ ] T10 state.json + 目录 + id。
- [ ] T11 撕裂尾修复。
- [ ] T12 undoable checkpoint + undo。
- [ ] T13 session fork（复制目录 + 重写 metadata）。

### LLM/循环

- [ ] T14 Message/ContentPart/ToolCall/Usage。
- [ ] T15 OpenAI encode/decode（非流式）。
- [ ] T16 流式 + accumulator。
- [ ] T17 tool-call ID 归一化 + 结构校验。
- [ ] T18 turn/step 状态机。
- [ ] T19 tool batch + 串行调度。
- [ ] T20 abort/steer/queue。
- [ ] T21 重试/退避。

### 工具

- [ ] T22 Tool contract + registry。
- [ ] T23 Executor：校验/resolve/execute。
- [ ] T24 截断 + 落盘 + 预览。
- [ ] T25 accesses + 冲突调度。
- [ ] T26 `Read`。
- [ ] T27 `Write`。
- [ ] T28 `Edit`。
- [ ] T29 `Grep`（rg）。
- [ ] T30 `Glob`。
- [ ] T31 `Bash`（前后台/超时/终止）。
- [ ] T32 危险命令分析。
- [ ] T33 后台任务 + 通知。

### 权限

- [ ] T34 模式 + 状态 + 广播。
- [ ] T35 规则解析/匹配。
- [ ] T36 13 条策略（严格顺序）。
- [ ] T37 审批服务 + 事件 + session 缓存。
- [ ] T38 敏感文件硬阻断。
- [ ] T39 路径守卫 + git 控制路径 + git cwd。
- [ ] T40 非交互 deny。
- [ ] T41 Plan 审阅强制审批。

### 子代理

- [ ] T42 profile + allowlist。
- [ ] T43 spawn + 隔离 + 权限继承。
- [ ] T44 超时 + partial。
- [ ] T45 resume。
- [ ] T46 后台 + 通知。

### 扩展

- [ ] T47 MCP stdio + 命名 + 注册。
- [ ] T48 MCP 状态机 + 重连。
- [ ] T49 MCP http/sse + OAuth。
- [ ] T50 Skills 发现/解析/调用。
- [ ] T51 Skills 参数 + slash。
- [ ] T52 插件清单 + 安装 + 贡献。
- [ ] T53 动态工具 + 公告。

### 表面

- [ ] T54 TUI 输入/流/审批。
- [ ] T55 KAP 会话/prompt/审批。
- [ ] T56 KAP token/host/origin/WS。
- [ ] T57 ACP。
- [ ] T58 SEA 打包。

---

## 7. 验收测试矩阵

| 层 | 测试 |
| --- | --- |
| 事件 | 写入/恢复/未知事件/撕裂尾/分叉 |
| 状态 | fold 幂等/undo/checkpoint |
| 循环 | 两步/tool 顺序/abort/maxSteps/重试 |
| LLM | 四协议编码/tool ID/流式/空响应/usage |
| 工具 | 校验/截断/并发冲突/去重/后台 |
| 权限 | 每策略/顺序/场景表/非交互/子代理 |
| 危险命令 | 第 15 篇向量 + 误报 |
| 路径 | 敏感/工作区外/git/symlink |
| MCP | 配置/冲突/重连/401/大结果 |
| Skills | 解析/覆盖/参数/嵌套 |
| 插件 | 清单/路径逃逸/安装回滚/贡献 |
| 表面 | 鉴权/host/origin/WS/ACP 映射 |
| 安全 | 第 24 篇矩阵 |

---

## 8. 风险登记

| 风险 | 影响 | 缓解 |
| --- | --- | --- |
| 事件 schema 反复改 | 历史不可恢复 | 先冻结 1.0，加迁移框架 |
| tool ID 处理不严 | provider 400 | 先做结构校验测试 |
| 压缩切在 tool 中间 | 请求失败 | 切割点合法性测试 |
| 权限链顺序错 | 安全/体验事故 | 场景表 + 顺序单测 |
| 没有沙箱 | 高危执行 | 容器/VM；不要宣传为沙箱 |
| SEA 资源 key 不一致 | 启动失败 | 用 manifest 常量共享 |
| 动态工具历史不可回放 | 恢复后模型看不到 | schema 进 system 消息 |
| MCP 生命周期复杂 | 工具时有时无 | 状态机 + 超时 + 错误可见 |
| 配置写入破坏注释 | 用户投诉 | 行级写回 + 重解析校验 |

---

## 9. 复刻检查表（完成度自评）

- [ ] 会话可恢复、可分叉、可导出。
- [ ] 工具调用顺序稳定、结果可配对。
- [ ] 大输出不会塞爆上下文。
- [ ] 权限链可按策略名解释每次决策。
- [ ] 危险命令能拆 `sudo/sh -c/eval`。
- [ ] 敏感文件硬阻断不可被规则绕过。
- [ ] 子代理隔离且不能无限递归。
- [ ] MCP 工具名稳定、冲突可见。
- [ ] Skills 优先级正确。
- [ ] Web/ACP 与 TUI 行为一致。
- [ ] 单二进制可离线启动。

---

## 10. 证据与延伸阅读

按顺序读前 24 篇；每篇末尾都有"证据索引"。  
优先精读：

1. `14-permissions.md`
2. `15-dangerous-command-analysis.md`
3. `08-tool-framework.md` + `09-builtin-tools.md`
4. `04-events-and-sessions.md`
5. `05-agent-loop.md` + `06-llm-layer.md`

这五组决定系统是否"像一个 agent"，其余是扩展与外壳。
