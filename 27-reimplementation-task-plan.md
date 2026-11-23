# Kimi Code CLI 重实现详细任务计划

> **供 agent 执行者使用**：按任务逐项执行；步骤用 checkbox（`- [ ]`）跟踪。本文拥有任务、依赖和验收细节，范围与里程碑以 `25-reimplementation-plan.md` 为准。

**目标**：用 TypeScript 清洁室重实现 Kimi Code CLI 核心，包括事件溯源会话、工具注册表、first-match-wins 权限链、危险命令分析、上下文安全，以及同一引擎上的 headless/TUI。ACP、KAP 和 SEA 是后续可选任务。

**架构**：`packages/core` 负责 loop、LLM 协议、工具执行和存储接口；`packages/agent-core` 负责会话、事件、工具、权限、MCP/Skills 与显式生命周期。`apps/cli` 和 `packages/tui` 只负责传输与渲染。首版不实现完整 DI Cascade；每个子系统只有一份 owning document。

**技术栈**：Node.js 24.15+（SEA），TypeScript 5.x strict ESM，pnpm 10 workspaces，zod，immer，picomatch，commander，Vitest；可选服务使用 Fastify + ws、@modelcontextprotocol/sdk、@agentclientprotocol/sdk。SQLite 仅在实现可重建会话索引时引入。

**规格**：本仓库根目录的 `01-product-and-repo.md` 至 `26-binary-vs-source-verification.md`。每个任务开始前阅读对应 owning doc；25 与本文件必须保持同步。

## 全局约束

- SEA 使用 Node `>=24.15.0`；开发可用 `>=22.19.0`，`engines` 保持 `>=22.19.0`。
- 仅使用 ESM（`"type": "module"`），不写 CommonJS 源文件。
- TypeScript 开启 `strict`、`noUncheckedIndexedAccess`、`exactOptionalPropertyTypes`、`verbatimModuleSyntax`。
- 公共函数和导出类型只在 owning doc 要求时写 JSDoc；不留死代码。
- 每个 durable event 都有 zod schema 和稳定 `type`；schema 变更必须有迁移。
- 工具消息按 **tool-call 顺序**追加，绝不按完成顺序。
- 权限策略顺序固定；`deny` 必须先于任何 `approve`。
- 敏感文件硬阻断在路径解析层执行，不能只放在策略层。
- 不声称有 OS 沙箱；`auto`/`yolo` 是非监督模式，不可信仓库必须在外部沙箱运行。
- 测试放在 `packages/*/test/**` 或 `apps/*/test/**`，每个测试只验证一个行为。
- 每个任务完成后做一次 conventional commit。

## 重点风险

以下问题必须由 owning task 的测试或明确的外部安全约束覆盖，不能只写在概述里。

1. **Tool call/result 配对**：缺失或重复 `toolCallId` 会使 provider 拒绝后续请求。
2. **压缩切在 tool exchange 内**：assistant tool call 与结果被切开会生成非法历史。
3. **权限链顺序反转**：把 `auto-mode-approve` 移到 `user-configured-deny` 前面会静默执行用户已拒绝的工具。
4. **Bash 绕过敏感文件边界**：`Read` 被阻断不代表 `Bash cat .env` 被阻断。首版必须明确该缺口并要求不可信任务使用外部沙箱；不能把文字说明当作防护。
5. **Wire 持久化**：撕裂尾、重放事件或追加失败不能损坏状态、重复 fold，或让调用方误以为事件已持久化。

---

## File Structure

```
packages/core/
  src/llm/{message,usage,protocol/openai,stream}
  src/loop/{loop,state,outcome}
  src/tool/{executor,contract}
  test/...
packages/agent-core/
  src/store/atomic
  src/event/{event2,dispatcher,state}
  src/session/{wire,sessionStore,stateJson,branch}
  src/context/{projection,compaction}
  src/tool/{registry,read,write,edit,grep,glob,bash,truncation,scheduler}
  src/permission/{types,modes,policy,rules,pathGuard,approval}
  src/session/subagent/*
  src/mcp/*
  src/skills/*
  test/...
packages/tree-sitter-bash/
  src/{parse,node,lexer,parser}
packages/acp/           # optional ACP
  src/{server,modes}
packages/tui/
  src/{app,input,stream,approval}
apps/cli/
  src/{main,run,commands}
apps/server/          # optional KAP
  src/{server,routes,auth}
```

规格文档位于仓库根目录，不使用 `docs/` 前缀。

---

## Task 0: Toolchain and workspace dependencies

**Files:**
- Modify: `package.json`
- Modify: `pnpm-workspace.yaml` (already correct)
- Test: `packages/core/test/smoke.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces: `pnpm test`, `pnpm typecheck` work from the root.

- [ ] **Step 1: Add runtime dependencies**

Modify root `package.json` dependencies:

```json
"dependencies": {
  "zod": "^4.0.0",
  "immer": "^10.1.0",
  "picomatch": "^4.0.0",
  "commander": "^14.0.0",
  "fastify": "^5.0.0",
  "ws": "^8.18.0",
  "@modelcontextprotocol/sdk": "^1.20.0",
  "@agentclientprotocol/sdk": "^0.4.0"
},
"devDependencies": {
  "@types/node": "^24.0.0",
  "@types/picomatch": "^4.0.0",
  "@types/ws": "^8.18.0",
  "oxlint": "^1.0.0",
  "postject": "^1.0.0-alpha.6",
  "tsx": "^4.20.0",
  "typescript": "^5.9.0",
  "vitest": "^4.0.0"
}
```

- [ ] **Step 2: Install and verify**

Run:

```bash
pnpm install
pnpm typecheck
```

Expected: install succeeds; typecheck exits 0.

- [ ] **Step 3: Add a smoke test**

Create `packages/core/test/smoke.test.ts`:

```ts
import { describe, expect, it } from "vitest";

describe("workspace", () => {
  it("runs vitest", () => {
    expect(1 + 1).toBe(2);
  });
});
```

- [ ] **Step 4: Run the smoke test**

Run: `pnpm vitest run packages/core/test/smoke.test.ts`  
Expected: 1 passed.

- [ ] **Step 5: Commit**

```bash
git add package.json pnpm-lock.yaml packages/core/test/smoke.test.ts
git commit -m "chore: add runtime dependencies and test harness"
```

---

## Task 1: Config, paths, and atomic storage

**Files:**
- Create: `packages/agent-core/src/config/schema.ts`
- Create: `packages/agent-core/src/config/configService.ts`
- Create: `packages/agent-core/src/config/paths.ts`
- Create: `packages/agent-core/src/store/atomic.ts`
- Test: `packages/agent-core/test/config/configService.test.ts`
- Test: `packages/agent-core/test/store/atomic.test.ts`

**Interfaces:**
- Consumes: `zod`.
- Produces:

```ts
interface ConfigService {
  get<T>(section: string): T | undefined;
  set(section: string, value: unknown): Promise<void>;
  diagnostics(): readonly { section?: string; severity: "warning" | "error"; message: string }[];
}
function resolveHome(env?: NodeJS.ProcessEnv): string; // KIMI_CODE_HOME ?? ~/.kimi-code
function atomicWriteFile(path: string, data: string, mode?: number): Promise<void>;
```

- [ ] **Step 1: Write failing tests**

`configService.test.ts`:

```ts
it("prefers memory over file over default", async () => {
  // default {a:1}, file {a:2}, memory {a:3}
  expect(service.get("demo")).toEqual({ a: 3 });
});

it("keeps the previous value when a section is invalid", async () => {
  // write invalid TOML for "demo"; expect previous valid value + diagnostic
});
```

`atomic.test.ts`:

```ts
it("leaves the original file intact on write failure", async () => {
  // write "old"; make rename fail; expect "old" still readable
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run packages/agent-core/test/config packages/agent-core/test/store`  
Expected: FAIL (modules not found).

- [ ] **Step 3: Implement minimal config and atomic write**

- `schema.ts`: section registry keyed by string, `z.object({}).passthrough()` default.
- `configService.ts`: read `config.toml` once, parse top-level keys, merge `default < file < memory`; on parse error keep last good and push a diagnostic.
- `paths.ts`: `resolveHome` honors `KIMI_CODE_HOME`, then `os.homedir()/.kimi-code`; create `0700`.
- `atomic.ts`: write `path.tmp.<pid>.<hex>`, `fsync`, `rename`; on error unlink temp and rethrow.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/agent-core/test/config packages/agent-core/test/store`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/agent-core/src/config packages/agent-core/src/store packages/agent-core/test/config packages/agent-core/test/store
git commit -m "feat: add layered config and atomic file storage"
```

---

## Task 2: Event2, replayable state, dispatcher

**Files:**
- Create: `packages/agent-core/src/event/event2.ts`
- Create: `packages/agent-core/src/event/state.ts`
- Create: `packages/agent-core/src/event/dispatcher.ts`
- Test: `packages/agent-core/test/event/dispatcher.test.ts`

**Interfaces:**
- Consumes: `zod`.
- Produces:

```ts
abstract class Event2<P> {
  static readonly type: string;
  static readonly durable: boolean;
  static readonly observable: boolean;
  static readonly schema?: z.ZodType<P>;
  readonly type: string;
  readonly time: number;
  serialize(): Record<string, unknown> & { type: string; time: number };
}

function defineState<T>(name: string, initial: () => T): StateKeyBuilder<T>;
// .replayable({schema}).on(EventClass, fold)

interface EventDispatcher {
  dispatch(event: Event2<unknown>): Promise<void>;
  restore(records: readonly unknown[]): Promise<void>;
  flush(): Promise<void>;
}
```

- [ ] **Step 1: Write failing tests**

```ts
it("folds a durable event into replayable state", async () => {
  // dispatch CounterAdd(2) twice from zero; expect count 4
});

it("is idempotent across restore", async () => {
  // serialize records; new dispatcher.restore(records); expect same state
});

it("ignores an unknown event type without throwing", async () => {
  // restore([{type:"unknown",time:1}]); expect state unchanged
});

it("does not commit state when a durable append fails", async () => {
  // injected wire rejects; expect dispatch to reject and state to remain unchanged
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run packages/agent-core/test/event/dispatcher.test.ts`  
Expected: FAIL.

- [ ] **Step 3: Implement event/state/dispatcher**

- `event2.ts`: static registry keyed by `type`; register rejects duplicate and durable-without-schema.
- `state.ts`: `defineState` returns builder; `.replayable` records a schema and fold map; `.on` rejects duplicate folds.
- `dispatcher.ts`: sequential queue; find folds by `type` and prepare drafts with immer. For durable events, await wire append before committing state or publishing; append failure rejects the dispatch and leaves state unchanged. The wire adapter is injected in Task 3.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/agent-core/test/event/dispatcher.test.ts`  
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add packages/agent-core/src/event packages/agent-core/test/event
git commit -m "feat: add event-sourced state and dispatcher"
```

---

## Task 3: Wire protocol and session layout

**Files:**
- Create: `packages/agent-core/src/session/wire.ts`
- Create: `packages/agent-core/src/session/sessionStore.ts`
- Create: `packages/agent-core/src/session/stateJson.ts`
- Test: `packages/agent-core/test/session/wire.test.ts`
- Test: `packages/agent-core/test/session/sessionStore.test.ts`

**Interfaces:**
- Consumes: `atomicWriteFile`, `EventDispatcher`.
- Produces:

```ts
const WIRE_PROTOCOL_VERSION = "1.5";

interface Wire {
  seal(): Promise<void>;                    // writes metadata if empty
  append(record: Record<string, unknown>): Promise<void>;
  read(): Promise<Record<string, unknown>[]>;
  readRestoreChains(): Promise<{ restorable: unknown[]; journal: unknown[] }>;
}

function sessionDir(home: string, workspaceId: string, sessionId: string): string;
function agentDir(session: string, agentId: string): string;
```

- [ ] **Step 1: Write failing tests**

```ts
it("writes metadata as the first line", async () => {
  // seal(); read(); expect [0].type === "metadata" && protocol_version === "1.5"
});

it("recovers from a torn last line", async () => {
  // append two records, truncate half of the last line, read(); expect 1 valid record
});

it("rejects a too-new min reader version", async () => {
  // metadata.min_protocol_version = "99.0"; expect WireVersionError
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run packages/agent-core/test/session`  
Expected: FAIL.

- [ ] **Step 3: Implement wire and layout**

- `wire.ts`: JSONL append; metadata on seal. Missing/malformed first-line metadata throws `STORAGE_CORRUPTED`; later malformed records are reported without crashing restore; on `min_protocol_version > 1.5` throw `WireVersionError`. Append errors propagate to the dispatcher.
- `sessionStore.ts`: `sessions/<workspaceId>/<sessionId>/agents/<agentId>/wire.jsonl`; `state.json` via `atomicWriteFile`; wire/state writes require the session lock.
- `stateJson.ts`: version 2 shape from doc 23; read legacy `workDir` fallback.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/agent-core/test/session`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/agent-core/src/session packages/agent-core/test/session
git commit -m "feat: add wire protocol and session directory layout"
```

---

## Task 4: LLM message model and OpenAI encode/decode (non-streaming)

**Files:**
- Create: `packages/core/src/llm/message.ts`
- Create: `packages/core/src/llm/usage.ts`
- Create: `packages/core/src/llm/protocol/openai.ts`
- Test: `packages/core/test/llm/openai.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces:

```ts
interface Message { role: "system" | "user" | "assistant" | "tool"; content: ContentPart[]; toolCalls?: ToolCall[]; toolCallId?: string; }
interface ToolCall { id: string; name: string; arguments: string | null; }
function encodeOpenAI(messages: Message[], tools: ToolDef[], model: string): unknown;
function decodeOpenAI(response: unknown): AssistantMessage;
```

- [ ] **Step 1: Write failing tests**

```ts
it("lowers a tool result to a tool message with the same id", () => {
  const body = encodeOpenAI(history, [], "gpt");
  expect(body.messages[1]).toMatchObject({ role: "tool", tool_call_id: "call_1" });
});

it("parses tool_calls arguments as JSON strings", () => {
  expect(decodeOpenAI(fixture).toolCalls?.[0]).toMatchObject({ name: "Read" });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run packages/core/test/llm/openai.test.ts`  
Expected: FAIL.

- [ ] **Step 3: Implement message types and OpenAI mapping**

- Roles, content parts (`text`, `think`, `image_url`), `ToolCall`.
- Encode: system/user/assistant/tool; tool results require `tool_call_id`; tools filtered by `deferred !== true`.
- Decode: `choices[0].message`, `tool_calls`, `finish_reason`, `usage`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/core/test/llm/openai.test.ts`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/core/src/llm packages/core/test/llm
git commit -m "feat: add LLM message model and OpenAI protocol"
```

---

## Task 5: Streaming and tool-call ID normalization

**Files:**
- Create: `packages/core/src/llm/stream.ts`
- Create: `packages/core/src/llm/toolIds.ts`
- Test: `packages/core/test/llm/stream.test.ts`
- Test: `packages/core/test/llm/toolIds.test.ts`

**Interfaces:**
- Consumes: `Message`, `ToolCall`.
- Produces:

```ts
class MessageAccumulator { pushDelta(part): void; finish(reason): AssistantMessage; }
function sanitizeToolCallId(id: string): string;
function dedupeToolCallIds(calls: readonly ToolCall[]): ToolCall[];
function verifyToolExchanges(messages: readonly Message[]): void; // throws on orphan/duplicate
```

- [ ] **Step 1: Write failing tests**

```ts
it("concatenates text deltas and tool argument fragments", () => {
  // push "He", "llo"; tool arg "{" then "}"; expect "Hello" and "{}"
});

it("suffixes duplicate tool-call ids", () => {
  expect(dedupeToolCallIds([{id:"x"},{id:"x"}]).map(c=>c.id)).toEqual(["x","x_2"]);
});

it("rejects an orphan tool result", () => {
  expect(() => verifyToolExchanges([{role:"tool",toolCallId:"missing"}])).toThrow();
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run packages/core/test/llm`  
Expected: FAIL.

- [ ] **Step 3: Implement accumulator and ID rules**

- Accumulator: append text/think; tool args by `_streamIndex`; finish maps reason.
- `sanitizeToolCallId`: `[^a-zA-Z0-9_-]` → `_`, max 64, empty → `randomUUID`.
- `dedupeToolCallIds`: `_2`, `_3`; truncate the base so the result stays within 64 chars, and skip suffixes already present in the batch.
- `verifyToolExchanges`: every tool message has a preceding open call; no duplicate results; no unanswered calls.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/core/test/llm`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/core/src/llm packages/core/test/llm
git commit -m "feat: add streaming accumulator and tool-call id rules"
```

---

## Task 6: Turn/step loop, tool executor, and context safety

**Files:**
- Create: `packages/core/src/loop/state.ts`
- Create: `packages/core/src/loop/loop.ts`
- Create: `packages/core/src/tool/executor.ts`
- Create: `packages/agent-core/src/context/projection.ts`
- Create: `packages/agent-core/src/context/compaction.ts`
- Test: `packages/core/test/loop/loop.test.ts`
- Test: `packages/agent-core/test/context/compaction.test.ts`

**Interfaces:**
- Consumes: `MessageAccumulator`, `verifyToolExchanges`.
- Produces:

```ts
interface LoopRequester { generate(messages: Message[], tools: ToolDef[]): Promise<AssistantMessage>; }
interface ToolExecutor { execute(call: ToolCall): Promise<ToolResult>; }
interface Loop {
  submit(text: string): Promise<void>;
  run(): Promise<TurnResult>;
  abort(): void;
}
function projectHistory(messages: readonly Message[]): Message[];
function canSplitAfter(messages: readonly Message[], index: number): boolean;
function compactHistory(messages: readonly Message[], summarize: (input: Message[]) => Promise<string>): Promise<Message[]>;
```

- [ ] **Step 1: Write failing tests**

```ts
it("runs a second step after a tool result", async () => {
  // requester returns tool call then text; expect two LLM calls
});

it("appends tool messages in call order even when completion order differs", async () => {
  // two tools with reversed delays; expect results [call1, call2]
});

it("aborts an in-flight request and keeps the turn resumable", async () => {
  // abort during generate; expect status "interrupted"
});

it("does not compact between a tool call and its result", async () => {
  // assistant tool call + tool result; every split inside the exchange is invalid
});

it("rejects an empty compaction summary without replacing history", async () => {
  // summarize returns ""; expect rejection and original history unchanged
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run packages/core/test/loop/loop.test.ts packages/agent-core/test/context/compaction.test.ts`
Expected: FAIL.

- [ ] **Step 3: Implement turn/step**

- State: `idle | running | interrupted`, `turn {id, step}`, `messages`.
- Loop: request → if tool calls, execute all, append results in call order, next step; else finish.
- Max steps; abort controller.
- Projection: pair calls/results, drop orphan results, and synthesize interrupted results as specified in doc 07.
- Manual compaction: choose only a legal split, require a non-empty summary, preserve the original history on failure. Automatic overflow recovery is explicitly out of scope for this plan.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/core/test/loop/loop.test.ts packages/agent-core/test/context/compaction.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/core/src/loop packages/core/src/tool packages/core/test/loop packages/agent-core/src/context packages/agent-core/test/context
git commit -m "feat: add loop, tool execution, and safe manual compaction"
```

---

## Task 7: Tool contract, registry, and file tools

**Files:**
- Create: `packages/agent-core/src/tool/contract.ts`
- Create: `packages/agent-core/src/tool/registry.ts`
- Create: `packages/agent-core/src/tool/read.ts`
- Create: `packages/agent-core/src/tool/write.ts`
- Create: `packages/agent-core/src/tool/edit.ts`
- Create: `packages/agent-core/src/permission/pathGuard.ts`
- Test: `packages/agent-core/test/tool/fileTools.test.ts`

**Interfaces:**
- Consumes: host `fs` interface.
- Produces:

```ts
interface ExecutableTool<I> {
  name: string;
  description: string;
  parameters: Record<string, unknown>;
  resolveExecution(input: I): ToolExecution;
}

class ToolRegistry {
  register(tool: ExecutableTool<unknown>): void;
  resolve(name: string): ExecutableTool<unknown> | undefined;
}
```

- [ ] **Step 1: Write failing tests**

```ts
it("rejects Write before Read for an existing file", async () => {
  // write existing file without read; expect isError
});

it("rejects Edit when the file changed since the last read", async () => {
  // read, mutate on disk, edit; expect isError
});

it("hard-blocks .env for Read", async () => {
  // expect PathSecurityError
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run packages/agent-core/test/tool/fileTools.test.ts`  
Expected: FAIL.

- [ ] **Step 3: Implement contract and tools**

- `contract.ts`: `ExecutableTool`, `ToolExecution`, `ToolAccesses`.
- `registry.ts`: name → tool; duplicate registration throws.
- `read.ts`: path resolution, line/column/char paging (start with simple whole-file + max chars).
- `write.ts`: read-before-write; `overwrite`/`append`.
- `edit.ts`: unique `old_string`; read-before-write; `replace_all`.
- `pathGuard.ts`: own canonical path resolution and sensitive-file hard block; throw `PATH_SENSITIVE`. Task 12 extends this same module with workspace rules instead of reimplementing it.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/agent-core/test/tool/fileTools.test.ts`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/agent-core/src/tool packages/agent-core/src/permission/pathGuard.ts packages/agent-core/test/tool
git commit -m "feat: add file tools and sensitive path guard"
```

---

## Task 8: Grep/Glob, truncation, and access-aware scheduling

**Files:**
- Create: `packages/agent-core/src/tool/grep.ts`
- Create: `packages/agent-core/src/tool/glob.ts`
- Create: `packages/agent-core/src/tool/truncation.ts`
- Create: `packages/agent-core/src/tool/scheduler.ts`
- Test: `packages/agent-core/test/tool/search.test.ts`
- Test: `packages/agent-core/test/tool/truncation.test.ts`
- Test: `packages/agent-core/test/tool/scheduler.test.ts`

**Interfaces:**
- Consumes: `rg`, `fd` subprocesses (or injectable).
- Produces:

```ts
function truncateForModel(text: string, maxChars = 50_000): { output: string; spillPath?: string };
class ToolScheduler { run<T>(accesses: ToolAccesses, work: () => Promise<T>): Promise<T>; }
```

- [ ] **Step 1: Write failing tests**

```ts
it("spills output above 50k chars and returns a readable path", async () => {
  const r = truncateForModel("x".repeat(60_000));
  expect(r.output).toContain("[...truncated]");
  expect(r.spillPath).toBeTruthy();
});

it("serializes two writers to the same path", async () => {
  // start two writes; expect max concurrency 1
});

it("filters sensitive files from Grep results", async () => {
  // repo contains .env; expect it absent from matches
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run packages/agent-core/test/tool`  
Expected: FAIL.

- [ ] **Step 3: Implement**

- Grep: spawn `rg --json`; map matches; filter `.env`/keys; paging.
- Glob: `fd` or `fs` walk; mtime desc; `.gitignore`.
- Truncation: 50k inline, 10M retained, head 4096/tail 1024, spill file.
- Scheduler: conflict if same path and either operation writes; queue.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/agent-core/test/tool`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/agent-core/src/tool packages/agent-core/test/tool
git commit -m "feat: add grep/glob, spill truncation, and tool scheduler"
```

---

## Task 9: Bash and background tasks

**Files:**
- Create: `packages/agent-core/src/tool/bash.ts`
- Create: `packages/agent-core/src/task/taskService.ts`
- Create: `packages/agent-core/src/tool/taskTools.ts`
- Test: `packages/agent-core/test/tool/bash.test.ts`
- Test: `packages/agent-core/test/task/taskService.test.ts`

**Interfaces:**
- Consumes: `child_process.spawn`, `ToolScheduler`.
- Produces:

```ts
interface BashInput { command: string; cwd?: string; timeout?: number; run_in_background?: boolean; description?: string; }
interface Task { id: string; status: "running" | "completed" | "failed" | "killed"; outputPath: string; }
```

- [ ] **Step 1: Write failing tests**

```ts
it("moves a foreground command to background after the timeout by default", async () => {
  // sleep 5, timeout 1; expect a task id and a still-running process
});

it("kills on foreground timeout when bash_auto_background_on_timeout is false", async () => {
  // disable auto-background; expect SIGTERM, then SIGKILL after the grace period
});

it("returns a task id for a background command", async () => {
  // run_in_background; expect /task_\w+/ and completion notification
});

it("injects the session id into the child environment", async () => {
  // echo $KIMI_CODE_SESSION_ID; expect the test session id
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run packages/agent-core/test/tool/bash.test.ts packages/agent-core/test/task`  
Expected: FAIL.

- [ ] **Step 3: Implement**

- Spawn via shell; `stdio: pipe`; close stdin; collect output with cap.
- Foreground timeout: by default detach and keep running as a background task; only when `bash_auto_background_on_timeout=false` use SIGTERM → 5s → SIGKILL.
- Background: detach, write `tasks/<id>/output.log`, emit completion; default timeout 600s except print mode defaults to no timeout.
- `TaskList/TaskOutput/TaskStop/WaitFor` minimal.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/agent-core/test/tool/bash.test.ts packages/agent-core/test/task`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/agent-core/src/tool packages/agent-core/src/task packages/agent-core/test
git commit -m "feat: add bash tool and background tasks"
```

---

## Task 10: Dangerous command analysis

**Files:**
- Create: `packages/tree-sitter-bash/package.json`
- Create: `packages/tree-sitter-bash/src/{parse,node,lexer,parser}.ts`
- Create: `packages/agent-core/src/permission/dangerousCommand.ts`
- Modify: `packages/agent-core/package.json` (workspace dependency on `@reimpl/tree-sitter-bash`)
- Test: `packages/agent-core/test/permission/dangerousCommand.test.ts`

**Interfaces:**
- Consumes: `parse` from the local `@reimpl/tree-sitter-bash` package created by this task; no unregistered external parser is assumed.
- Produces:

```ts
type Verdict = { kind: "dangerous"; command: string } | { kind: "unanalyzable" } | undefined;
function analyzeBash(source: string): Verdict;
```

- [ ] **Step 1: Write failing tests (the vector table from doc 15A §9)**

```ts
it.each([
  ["rm -rf /", "dangerous"],
  ["rm -rf /tmp/x", "safe"],
  ["rm -rf $DIR", "dangerous"],
  ["sudo -u root rm -rf /", "dangerous"],
  ["bash -c 'rm -rf /'", "dangerous"],
  ["bash -c \"$CMD\"", "unanalyzable"],
  ["command -v git", "safe"],
])("%s -> %s", (cmd, expected) => { ... });
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run packages/agent-core/test/permission/dangerousCommand.test.ts`  
Expected: FAIL.

- [ ] **Step 3: Implement analyzer**

- Start with a tokenizer that produces `command`, `command_name`, `word`, `raw_string`, `string`.
- Implement `collectCommands`, `analyzeCommand`, `analyzeInvocation`, `literalText`, `normalizeCommandName`, `dropLeadingOptions`, `dropLaunchWrapperOperands`.
- Rules: sudo/doas, env/command/exec/nohup/nice, sh -c, eval, busybox, rm, dd, systemctl, init, mkfs.
- Parse error → `unanalyzable`; depth ≥4 → `unanalyzable`. The permission mode owns the final result: manual asks, yolo allows unanalyzable commands, auto skips this policy.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/agent-core/test/permission/dangerousCommand.test.ts`  
Expected: all vectors pass.

- [ ] **Step 5: Commit**

```bash
git add packages/tree-sitter-bash packages/agent-core/package.json packages/agent-core/src/permission packages/agent-core/test/permission
git commit -m "feat: add local bash parser and dangerous command analysis"
```

---

## Task 11: Permission modes and the 13-policy chain

**Files:**
- Create: `packages/agent-core/src/permission/types.ts`
- Create: `packages/agent-core/src/permission/modes.ts`
- Create: `packages/agent-core/src/permission/rules.ts`
- Create: `packages/agent-core/src/permission/policy.ts`
- Create: `packages/agent-core/src/permission/policies/*.ts`
- Test: `packages/agent-core/test/permission/policyChain.test.ts`

**Interfaces:**
- Consumes: `analyzeBash`, `picomatch`, and config; this task implements the rule matcher before assembling the chain.
- Produces:

```ts
type PermissionMode = "manual" | "yolo" | "auto";
interface PermissionRule { decision: "allow"|"deny"|"ask"; scope: "turn-override"|"session-runtime"|"project"|"user"; pattern: string; reason?: string; }
function matchRule(rule: PermissionRule, toolName: string, matchesArgs?: (s:string)=>boolean): boolean;
interface PermissionPolicy { name: string; evaluate(ctx): PermissionResult | undefined; }
class PermissionPolicyService {
  evaluate(ctx): Promise<{ policyName: string; result: PermissionResult } | undefined>;
}
```

- [ ] **Step 1: Write failing tests (the scenario table from doc 14 §10)**

```ts
it.each([
  ["auto", "AskUserQuestion", "deny", "auto-mode-ask-user-question-deny"],
  ["yolo", "Bash:rm -rf /", "ask", "dangerous-command-ask"],
  ["manual", "Write:src/a.ts", "approve", "git-cwd-write-approve"],
  ["manual", "Bash:ls", "ask", "fallback-ask"],
])("%s %s -> %s via %s", ...);
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run packages/agent-core/test/permission/policyChain.test.ts`  
Expected: FAIL.

- [ ] **Step 3: Implement rule matching and the chain in the exact order**

`rules.ts` parses `ToolName(arg-pattern)`, matches MCP wildcards with picomatch, and lets each tool interpret argument patterns. Then assemble:

```
auto-mode-ask-user-question-deny
user-configured-deny
dangerous-command-ask
auto-mode-approve
session-approval-history
user-configured-ask
user-configured-allow
sensitive-file-access-ask
git-control-path-access-ask
yolo-mode-approve
default-tool-approve
git-cwd-write-approve
fallback-ask
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/agent-core/test/permission/policyChain.test.ts`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/agent-core/src/permission packages/agent-core/test/permission
git commit -m "feat: add permission modes and 13-policy chain"
```

---

## Task 12: Workspace guard and approval service

**Files:**
- Modify: `packages/agent-core/src/permission/pathGuard.ts`
- Create: `packages/agent-core/src/permission/approval.ts`
- Test: `packages/agent-core/test/permission/rules.test.ts`
- Test: `packages/agent-core/test/permission/pathGuard.test.ts`
- Test: `packages/agent-core/test/permission/approval.test.ts`

**Interfaces:**
- Consumes: `picomatch`, config.
- Produces:

```ts
function resolvePathAccess(path: string, cwd: string, workspace: WorkspaceConfig): string; // throws PATH_SENSITIVE
interface ApprovalService { request(req): Promise<ApprovalResponse>; }
```

- [ ] **Step 1: Write failing tests**

```ts
it("hard-blocks .env even when an allow rule matches", async () => {
  // expect PathSecurityError
});

it("lets a session approval suppress the next identical call", async () => {
  // approve with scope session; second call -> approve without prompt
});

it("denies when non-interactive and no resolver", async () => {
  // expect deny
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run packages/agent-core/test/permission`  
Expected: FAIL.

- [ ] **Step 3: Implement**

- Reuse Task 11's rule parser/matcher; `deny > ask > allow` is enforced by policy order.
- Extend Task 7's `pathGuard.ts`: sensitive list from doc 16 throws before policy; relative outside → reject; absolute outside → allow to policy.
- Approval: event/request, non-interactive deny, `scope: session` stores the pattern.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/agent-core/test/permission`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/agent-core/src/permission packages/agent-core/test/permission
git commit -m "feat: add permission rules, path guard, and approval"
```

---

## Task 13: Subagents and task tools

**Files:**
- Create: `packages/agent-core/src/subagent/profiles.ts`
- Create: `packages/agent-core/src/subagent/subagent.ts`
- Create: `packages/agent-core/src/subagent/agentTool.ts`
- Test: `packages/agent-core/test/subagent/subagent.test.ts`

**Interfaces:**
- Consumes: `Loop`, `ToolRegistry`, task service.
- Produces:

```ts
interface AgentInput { prompt: string; description: string; subagent_type?: string; run_in_background?: boolean; }
interface SubagentResult { result: string; usage: TokenUsage; }
```

- [ ] **Step 1: Write failing tests**

```ts
it("does not expose the parent history to a subagent", async () => {
  // parent messages contain secret; subagent prompt only; expect no secret in child
});

it("rejects a subagent type outside the allowlist", async () => {
  // profile allowlist ["coder"]; ask for "plan"; expect error
});

it("returns partial output on timeout", async () => {
  // subagent writes then hangs; timeout; expect marker
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run packages/agent-core/test/subagent`  
Expected: FAIL.

- [ ] **Step 3: Implement**

- Profiles: `agent`, `coder`, `explore`, `plan` with tool allowlists from doc 18.
- Spawn: new loop + registry; inherit permission mode; no parent history.
- Timeout + partial output; background task notification.
- `Agent` tool schema.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/agent-core/test/subagent`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/agent-core/src/subagent packages/agent-core/test/subagent
git commit -m "feat: add subagents and agent tool"
```

---

## Task 14: MCP stdio and tool naming

**Files:**
- Create: `packages/agent-core/src/mcp/config.ts`
- Create: `packages/agent-core/src/mcp/manager.ts`
- Create: `packages/agent-core/src/mcp/tools.ts`
- Test: `packages/agent-core/test/mcp/mcp.test.ts`

**Interfaces:**
- Consumes: `@modelcontextprotocol/sdk`.
- Produces:

```ts
interface McpServerConfig { transport: "stdio"; command: string; args?: string[]; env?: Record<string,string>; }
function qualifiedName(server: string, tool: string): string; // mcp__<server>__<tool>
class McpManager { connect(config): Promise<void>; tools(): readonly ToolDef[]; close(): Promise<void>; }
```

- [ ] **Step 1: Write failing tests**

```ts
it("merges user < project-root < project mcp.json", async () => { ... });
it("names tools mcp__server__tool", () => { expect(qualifiedName("a","b")).toBe("mcp__a__b"); });
it("keeps the first tool on a name collision", async () => { ... });
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run packages/agent-core/test/mcp`  
Expected: FAIL.

- [ ] **Step 3: Implement**

- Three-layer config load; schema validation.
- stdio connect; `listTools`; register as `mcp__server__tool`.
- Collision handling; status `connected|failed|disabled`.
- Tool result text/image mapping; 10 MiB cap.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/agent-core/test/mcp`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/agent-core/src/mcp packages/agent-core/test/mcp
git commit -m "feat: add MCP stdio, config layers, and tool naming"
```

---

## Task 15: Skills

**Files:**
- Create: `packages/agent-core/src/skills/parser.ts`
- Create: `packages/agent-core/src/skills/catalog.ts`
- Create: `packages/agent-core/src/skills/skillTool.ts`
- Test: `packages/agent-core/test/skills/skills.test.ts`

**Interfaces:**
- Consumes: host fs.
- Produces:

```ts
interface Skill { name: string; description: string; type: "prompt"|"inline"|"flow"|"reference"; body: string; dir: string; }
function discoverSkills(roots: readonly string[]): Promise<readonly Skill[]>;
function renderSkillListing(skills: readonly Skill[]): string;
```

- [ ] **Step 1: Write failing tests**

```ts
it("requires name and description for a directory SKILL.md", async () => { ... });
it("prefers project over user over builtin", async () => { ... });
it("rejects a disableModelInvocation skill from the tool", async () => { ... });
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run packages/agent-core/test/skills`  
Expected: FAIL.

- [ ] **Step 3: Implement**

- Frontmatter parse (`---` block), fields from doc 12.
- Roots precedence; same-name override.
- Listing injection.
- `Skill` tool: load body, depth ≤3, args interpolation.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/agent-core/test/skills`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/agent-core/src/skills packages/agent-core/test/skills
git commit -m "feat: add skills discovery and invocation"
```

---

## Task 16: TUI

**Files:**
- Create: `packages/tui/src/app.ts`
- Create: `packages/tui/src/input.ts`
- Create: `packages/tui/src/stream.ts`
- Create: `packages/tui/src/approval.ts`
- Create: `apps/cli/src/{main,run}.ts`
- Test: `packages/tui/test/render.test.ts`
- Test: `apps/cli/test/e2e.test.ts`

**Interfaces:**
- Consumes: `Loop`, `PermissionGate`.
- Produces: a terminal app with input box, streamed assistant text, tool cards, approval panel, plus an executable CLI entry for real-process tests.

- [ ] **Step 1: Write failing test**

```ts
it("renders a streamed assistant message without duplicating text", () => {
  // render("Hel"), render("Hello"); expect output contains "Hello" once
});

it("persists and resumes a real CLI session across processes", async () => {
  // spawn the built CLI with a fake provider: prompt + tool approval, exit, resume, then continue
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm vitest run packages/tui/test/render.test.ts`  
Expected: FAIL.

- [ ] **Step 3: Implement**

- Raw-mode input, history, `!` shell mode.
- Stream rendering with diff updates.
- Approval panel: `1/2/3`, Esc; "approve for session".
- Status bar: model, cwd, permission mode.
- CLI entry: wire headless/TUI to the engine; use a scripted local provider in E2E so no external model service is required.
- E2E must exercise the real binary/process boundary, including approval, wire persistence, process exit, resume, and a subsequent tool call.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/tui apps/cli/test/e2e.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/tui apps/cli
git commit -m "feat: add terminal UI"
```

---

## Task 17: Optional KAP server (REST/WS)

**Files:**
- Create: `apps/server/src/server.ts`
- Create: `apps/server/src/routes/meta.ts`
- Create: `apps/server/src/routes/sessions.ts`
- Create: `apps/server/src/routes/prompts.ts`
- Create: `apps/server/src/auth.ts`
- Test: `apps/server/test/auth.test.ts`
- Test: `apps/server/test/sessions.test.ts`

**Interfaces:**
- Consumes: Fastify, ws, `SessionStore`, `Loop`.
- Produces:

```
GET  /api/v1/meta
GET  /api/v1/healthz
POST /api/v1/sessions
POST /api/v1/sessions/:id/prompts
GET  /api/v1/sessions/:id/context
WS   /api/v1/ws
```

- [ ] **Step 1: Write failing tests**

```ts
it("returns 401 without a bearer token", async () => { ... });
it("returns 403 for an unlisted Host", async () => { ... });
it("accepts a WS upgrade only with a valid token", async () => { ... });
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run apps/server/test`  
Expected: FAIL.

- [ ] **Step 3: Implement**

- token file `server.token` (32 bytes, 0600).
- host allowlist and origin check.
- session create/prompt/context; WS event stream.
- envelope `{code,msg,data,request_id}`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run apps/server/test`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/server
git commit -m "feat: add KAP-compatible server"
```

---

## Task 18: Optional ACP server

**Files:**
- Create: `packages/acp/src/server.ts`
- Create: `packages/acp/src/modes.ts`
- Test: `packages/acp/test/acp.test.ts`

**Interfaces:**
- Consumes: `@agentclientprotocol/sdk`, engine.
- Produces: `initialize`, `session/new`, `session/prompt`, `session/cancel`, `session/set_mode`.

- [ ] **Step 1: Write failing test**

```ts
it("maps ACP mode auto to permission mode auto", () => {
  expect(toPermissionMode("auto")).toBe("auto");
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm vitest run packages/acp/test`  
Expected: FAIL.

- [ ] **Step 3: Implement**

- NDJSON stdio loop.
- Capabilities from doc 21.
- Mode mapping: `default→manual`, `plan→plan+manual`, `auto→auto`, `yolo→yolo`.
- Prompt → loop; updates → `session/update`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/acp/test`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/acp
git commit -m "feat: add ACP server"
```

---

## Task 19: Optional SEA packaging

**Files:**
- Create: `apps/cli/scripts/build-sea.mjs`
- Create: `apps/cli/scripts/verify-assets.mjs`
- Test: `apps/cli/test/sea.test.ts`

**Interfaces:**
- Consumes: `postject`, Node `--experimental-sea-config`.
- Produces: a `kimi` binary with an asset manifest and SHA-256 verification.

- [ ] **Step 1: Write failing test**

```ts
it("rejects a tampered asset", async () => {
  // flip one byte in an extracted asset; verifyAssets() -> false
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm vitest run apps/cli/test/sea.test.ts`  
Expected: FAIL.

- [ ] **Step 3: Implement**

- `sea-config.json` with `main`, `assets`, `useCodeCache` only on host.
- Inject with `postject`; sign (optional).
- Runtime extract: verify manifest version/target/SHA-256; atomic write to cache.
- `verifyAssets` returns false on mismatch.

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm vitest run apps/cli/test/sea.test.ts`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/cli/scripts apps/cli/test
git commit -m "build: add SEA packaging and asset verification"
```

---

## Acceptance Matrix

| Area | Command | Expected |
| --- | --- | --- |
| Config | `pnpm vitest run packages/agent-core/test/config` | pass |
| Events | `pnpm vitest run packages/agent-core/test/event` | pass, including append-failure atomicity |
| Wire | `pnpm vitest run packages/agent-core/test/session` | pass, including metadata corruption and locks |
| LLM | `pnpm vitest run packages/core/test/llm` | pass |
| Loop | `pnpm vitest run packages/core/test/loop` | pass |
| Context | `pnpm vitest run packages/agent-core/test/context` | legal splits; empty summary leaves history unchanged |
| Tools | `pnpm vitest run packages/agent-core/test/tool` | pass |
| Bash | `pnpm vitest run packages/agent-core/test/tool/bash.test.ts` | pass |
| Dangerous | `pnpm vitest run packages/agent-core/test/permission/dangerousCommand.test.ts` | pass |
| Policy | `pnpm vitest run packages/agent-core/test/permission/policyChain.test.ts` | pass |
| Rules/guard | `pnpm vitest run packages/agent-core/test/permission` | pass |
| Subagents | `pnpm vitest run packages/agent-core/test/subagent` | pass |
| MCP | `pnpm vitest run packages/agent-core/test/mcp` | pass |
| Skills | `pnpm vitest run packages/agent-core/test/skills` | pass |
| CLI E2E | `pnpm vitest run apps/cli/test/e2e.test.ts` | real process: prompt/approval/exit/resume/continue |
| Optional Server | `pnpm vitest run apps/server/test` | pass when KAP is selected |
| Optional ACP | `pnpm vitest run packages/acp/test` | pass when ACP is selected |
| Optional SEA | `pnpm vitest run apps/cli/test/sea.test.ts` | pass when SEA is selected |
| Types | `pnpm typecheck` | exit 0 |
| Lint | `pnpm lint` | exit 0 |

## 完成定义

核心 CLI 必须同时满足：

- 真实 CLI 进程能创建会话、prompt、调用工具、完成审批、退出、恢复并继续对话。
- `wire.jsonl` 重放得到相同状态；撕裂尾可恢复；追加失败不会先提交内存状态。
- 工具结果按调用顺序排列；重复 ID 归一化；压缩不会切开 tool exchange，空摘要不替换历史。
- `manual/yolo/auto` 符合 14 篇场景表；危险命令向量以 15A 为准。
- 文件工具硬阻断敏感文件。Bash 不具备同等边界，产品必须明确不支持直接执行不可信仓库；这类任务只能在外部 OS/容器沙箱中运行。
- 子代理隔离且默认不能递归；MCP 工具命名和冲突可见；Skills 优先级正确。
- `pnpm typecheck`、`pnpm lint` 和核心测试全部通过。

KAP、ACP、SEA 只有选择相应可选任务时才需要验收，不阻塞核心 CLI 完成。

## 本计划明确不做

- Desktop/VS Code 外壳。
- 完整 DI Cascade（03）；首版使用显式生命周期。
- Tower 和 Swarm（19）。
- Hooks（17）、插件（13）、远程环境与远程控制（20/21）。
- MCP/Kimi OAuth（先使用静态 key）。
- 动态工具 `select_tools`（10）。
- 自动溢出压缩与完整恢复策略（07）；本计划只实现安全投影和手动压缩。
- 会话索引 SQLite、完整 fork/undo/export、全部内置工具与四协议 parity；当前任务只覆盖明确列出的子集。

## 覆盖与复核

| 文档 | 本计划覆盖 |
| --- | --- |
| 01/02/04/05/06/08/09 | 部分：CLI、事件、循环、OpenAI、核心工具；其余见非目标 |
| 03 | 仅保留显式生命周期；完整 DI/Cascade 不做 |
| 07 | 部分：安全投影、合法切割、手动压缩 |
| 10/13/17/19/20 | 不做，或只做研究引用 |
| 11/12/14/14A/15/15A/16/18/22/23/24 | 部分覆盖，以各任务列出的接口和测试为准 |
| 21 | 核心覆盖 TUI/headless；ACP/KAP/远程为可选或不做 |
| 25 | 范围与里程碑 owner；本文件不得与其冲突 |
| 26 | 仅可选 SEA 任务使用资产校验方法，不继承过强的等价结论 |

- 五类重点风险分别由 Tasks 5、6、11、12/外部沙箱约束、2/3 覆盖；第四项是明确产品限制，不伪装成已防护。
- 类型名与 06/08/14 对齐；任务只给签名和验收，具体实现由执行者完成。

## 执行交接

本文件位于仓库根目录 `27-reimplementation-task-plan.md`。

按 Task 0–16 顺序实现核心 CLI；每个任务先运行失败测试，再实现并通过验收。Tasks 17–19 只在选择对应可选能力时执行。完成核心任务后，必须另做一次跨任务审查，重点检查事件持久化、tool 配对、压缩切割和权限顺序。
