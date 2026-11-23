# Kimi Code Reimplementation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a clean-room TypeScript reimplementation of the Kimi Code CLI architecture: event-sourced sessions, a tool registry with a first-match-wins permission chain, a pure-TS dangerous-command analyzer, and a headless/TUI/server surface over one engine.

**Architecture:** `packages/core` owns the loop, LLM protocols, tool executor, and storage interfaces with no DI. `packages/agent-core` owns DI×Scope, sessions, events, tools, permissions, MCP/skills/plugins. `apps/cli`, `packages/tui`, and `apps/server` are transport/render shells that call the engine. Every subsystem has a single owning document in `docs/`.

**Tech Stack:** Node.js 24.15+ (SEA), TypeScript 5.x strict ESM, pnpm 10 workspaces, zod, immer, picomatch, commander, Fastify + ws, Vitest, SQLite (better-sqlite3), @modelcontextprotocol/sdk, @agentclientprotocol/sdk.

**Spec:** `docs/01-product-and-repo.md` … `docs/26-binary-vs-source-verification.md` (this repo). Read the owning doc before each task.

## Global Constraints

- Node `>=24.15.0` for SEA; development may use `>=22.19.0`. `engines` stays `>=22.19.0`.
- ESM only (`"type": "module"`); no CommonJS source files.
- TypeScript `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `verbatimModuleSyntax`.
- Public functions and exported types require JSDoc only where the doc says so; no dead code.
- Every durable event has a zod schema and a stable `type` string. Changing a schema requires a migration.
- Tool messages are appended in **tool-call order**, never completion order.
- Permission policy order is fixed; `deny` is always evaluated before any `approve`.
- Sensitive-file hard block is enforced in path resolution, not only in a policy.
- No OS sandbox is claimed; `auto`/`yolo` are documented as unsupervised modes.
- Tests live under `packages/*/test/**` or `apps/*/test/**`; one behavior per test.
- Commit after every task with a conventional message.

## Review Focus

These are the five failure modes most likely to bite a user; each has a dedicated test in the task that owns the code.

1. **Tool call/result pairing** — a missing or duplicate `toolCallId` makes every provider reject the next request.
2. **Compaction split inside a tool exchange** — cutting between an assistant tool call and its result produces an invalid history.
3. **Permission chain inversion** — moving `auto-mode-approve` above `user-configured-deny` silently lets denied tools run.
4. **Sensitive-file bypass via Bash** — `Read` is blocked, but `Bash cat .env` is not; the user expects one boundary.
5. **Wire durability** — a torn last line or a replayed event must not corrupt state or double-apply a fold.

---

## File Structure

```
packages/core/
  src/llm/{message,usage,protocol/openai,stream}
  src/loop/{loop,state,outcome}
  src/tool/{executor,contract}
  src/store/{appendLog,atomic}
  test/...
packages/agent-core/
  src/di/{service,scope,collection}
  src/event/{event2,dispatcher,state}
  src/session/{wire,sessionStore,stateJson,branch}
  src/tool/{registry,read,write,edit,grep,glob,bash,truncation,scheduler}
  src/permission/{types,modes,policy,rules,pathGuard,approval}
  src/session/subagent/*
  src/mcp/*
  src/skills/*
  test/...
packages/tui/
  src/{app,input,stream,approval}
apps/cli/
  src/{main,run,commands}
apps/server/
  src/{server,routes,auth}
docs/                  # this specification
```

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
  "@agentclientprotocol/sdk": "^0.4.0",
  "better-sqlite3": "^12.0.0"
},
"devDependencies": {
  "@types/node": "^24.0.0",
  "@types/picomatch": "^4.0.0",
  "@types/ws": "^8.18.0",
  "@types/better-sqlite3": "^7.6.0",
  "oxlint": "^1.0.0",
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
  // dispatch CounterAdd(2) twice; expect count 3
});

it("is idempotent across restore", async () => {
  // serialize records; new dispatcher.restore(records); expect same state
});

it("ignores an unknown event type without throwing", async () => {
  // restore([{type:"unknown",time:1}]); expect state unchanged
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run packages/agent-core/test/event/dispatcher.test.ts`  
Expected: FAIL.

- [ ] **Step 3: Implement event/state/dispatcher**

- `event2.ts`: static registry keyed by `type`; register rejects duplicate and durable-without-schema.
- `state.ts`: `defineState` returns builder; `.replayable` records a schema and fold map; `.on` rejects duplicate folds.
- `dispatcher.ts`: sequential queue; for each event, find folds by `type`, produce draft with immer, commit, then append if `durable` (wire injected later); publish if `observable`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/agent-core/test/event/dispatcher.test.ts`  
Expected: PASS (3 tests).

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

- `wire.ts`: JSONL append; metadata on seal; read skips malformed lines and reports them; on `min_protocol_version > 1.5` throw `WireVersionError`.
- `sessionStore.ts`: `sessions/<workspaceId>/<sessionId>/agents/<agentId>/wire.jsonl`; `state.json` via `atomicWriteFile`.
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
- `dedupeToolCallIds`: `_2`, `_3`.
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

## Task 6: Turn/step loop and minimal tool executor

**Files:**
- Create: `packages/core/src/loop/state.ts`
- Create: `packages/core/src/loop/loop.ts`
- Create: `packages/core/src/tool/executor.ts`
- Test: `packages/core/test/loop/loop.test.ts`

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
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run packages/core/test/loop/loop.test.ts`  
Expected: FAIL.

- [ ] **Step 3: Implement turn/step**

- State: `idle | running | interrupted`, `turn {id, step}`, `messages`.
- Loop: request → if tool calls, execute all, append results in call order, next step; else finish.
- Max steps; abort controller.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/core/test/loop/loop.test.ts`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/core/src/loop packages/core/src/tool packages/core/test/loop
git commit -m "feat: add turn/step loop and tool executor"
```

---

## Task 7: Tool contract, registry, and file tools

**Files:**
- Create: `packages/agent-core/src/tool/contract.ts`
- Create: `packages/agent-core/src/tool/registry.ts`
- Create: `packages/agent-core/src/tool/read.ts`
- Create: `packages/agent-core/src/tool/write.ts`
- Create: `packages/agent-core/src/tool/edit.ts`
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
- Path guard: sensitive files throw `PATH_SENSITIVE`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/agent-core/test/tool/fileTools.test.ts`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/agent-core/src/tool packages/agent-core/test/tool
git commit -m "feat: add tool registry and read/write/edit"
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
it("kills a foreground command after the timeout", async () => {
  // sleep 5, timeout 1; expect isError and process gone
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
- Foreground timeout: SIGTERM → 5s → SIGKILL.
- Background: detach, write `tasks/<id>/output.log`, emit completion.
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
- Create: `packages/agent-core/src/permission/dangerousCommand.ts`
- Test: `packages/agent-core/test/permission/dangerousCommand.test.ts`

**Interfaces:**
- Consumes: `parse` from `@reimpl/tree-sitter-bash` (create in this task or vendor a parser).
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
- Fail-closed: parse error → `unanalyzable`; depth ≥4 → `unanalyzable`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/agent-core/test/permission/dangerousCommand.test.ts`  
Expected: all vectors pass.

- [ ] **Step 5: Commit**

```bash
git add packages/agent-core/src/permission packages/agent-core/test/permission
git commit -m "feat: add dangerous command analysis"
```

---

## Task 11: Permission modes and the 13-policy chain

**Files:**
- Create: `packages/agent-core/src/permission/types.ts`
- Create: `packages/agent-core/src/permission/modes.ts`
- Create: `packages/agent-core/src/permission/policy.ts`
- Create: `packages/agent-core/src/permission/policies/*.ts`
- Test: `packages/agent-core/test/permission/policyChain.test.ts`

**Interfaces:**
- Consumes: `analyzeBash`, rule matcher.
- Produces:

```ts
type PermissionMode = "manual" | "yolo" | "auto";
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

- [ ] **Step 3: Implement the chain in the exact order**

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

## Task 12: Rules, path guard, and approval service

**Files:**
- Create: `packages/agent-core/src/permission/rules.ts`
- Create: `packages/agent-core/src/permission/pathGuard.ts`
- Create: `packages/agent-core/src/permission/approval.ts`
- Test: `packages/agent-core/test/permission/rules.test.ts`
- Test: `packages/agent-core/test/permission/pathGuard.test.ts`
- Test: `packages/agent-core/test/permission/approval.test.ts`

**Interfaces:**
- Consumes: `picomatch`, config.
- Produces:

```ts
interface PermissionRule { decision: "allow"|"deny"|"ask"; scope: "turn-override"|"session-runtime"|"project"|"user"; pattern: string; reason?: string; }
function matchRule(rule: PermissionRule, toolName: string, matchesArgs?: (s:string)=>boolean): boolean;
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

- Rule parsing `ToolName(arg-pattern)`; picomatch; `deny > ask > allow`.
- Sensitive list from doc 16; throw before policy.
- Workspace guard: relative outside → reject; absolute outside → allow to policy.
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
interface Skill { name: string; description: string; type: "prompt"|"inline"|"flow"; body: string; dir: string; }
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
- Test: `packages/tui/test/render.test.ts`

**Interfaces:**
- Consumes: `Loop`, `PermissionGate`.
- Produces: a terminal app with input box, streamed assistant text, tool cards, approval panel.

- [ ] **Step 1: Write failing test**

```ts
it("renders a streamed assistant message without duplicating text", () => {
  // render("Hel"), render("Hello"); expect output contains "Hello" once
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

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run packages/tui`  
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/tui apps/cli
git commit -m "feat: add terminal UI"
```

---

## Task 17: KAP server (REST/WS)

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

## Task 18: ACP server

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

## Task 19: SEA packaging

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
| Events | `pnpm vitest run packages/agent-core/test/event` | pass |
| Wire | `pnpm vitest run packages/agent-core/test/session` | pass |
| LLM | `pnpm vitest run packages/core/test/llm` | pass |
| Loop | `pnpm vitest run packages/core/test/loop` | pass |
| Tools | `pnpm vitest run packages/agent-core/test/tool` | pass |
| Bash | `pnpm vitest run packages/agent-core/test/tool/bash.test.ts` | pass |
| Dangerous | `pnpm vitest run packages/agent-core/test/permission/dangerousCommand.test.ts` | pass |
| Policy | `pnpm vitest run packages/agent-core/test/permission/policyChain.test.ts` | pass |
| Rules/guard | `pnpm vitest run packages/agent-core/test/permission` | pass |
| Subagents | `pnpm vitest run packages/agent-core/test/subagent` | pass |
| MCP | `pnpm vitest run packages/agent-core/test/mcp` | pass |
| Skills | `pnpm vitest run packages/agent-core/test/skills` | pass |
| Server | `pnpm vitest run apps/server/test` | pass |
| ACP | `pnpm vitest run packages/acp/test` | pass |
| SEA | `pnpm vitest run apps/cli/test/sea.test.ts` | pass |
| Types | `pnpm typecheck` | exit 0 |
| Lint | `pnpm lint` | exit 0 |

## Definition of Done

- A new session can be created, prompted, tool-called, persisted, and resumed.
- `wire.jsonl` replays to the same state after restart.
- Tool results are in call order; duplicate IDs are normalized.
- `manual/yolo/auto` match the scenario table in doc 14.
- Dangerous-command vectors in doc 15A pass.
- Sensitive files are hard-blocked for file tools; the Bash bypass is documented.
- Subagents are isolated and cannot recurse by default.
- MCP tools are named `mcp__server__tool` and collisions are visible.
- Skills respect project > user > extra > builtin.
- KAP rejects missing token / bad Host / bad Origin.
- SEA binary starts offline and rejects a tampered asset.

## Out of Scope (for this plan)

- Tower missions and worktree merge gates (doc 19).
- Desktop/VS Code shells.
- OAuth flows for MCP/Kimi providers (start with static keys).
- Dynamic tool loading (`select_tools`) — add after the core loop is stable.
- Full compaction algorithm — start with manual truncation, then add auto.

## Self-Review

- Spec coverage: each of docs 01–26 has at least one owning task, except Tower (explicitly out of scope) and telemetry (not required for behavior).
- Type consistency: `Message`, `ToolCall`, `ToolResult`, `PermissionMode`, `PermissionRule` names match docs 06/08/14.
- Review Focus: five failure modes each have a test in Tasks 5, 6, 11, 12, and 3.
- Proportion: tasks specify signatures and acceptance, not full bodies; bodies belong to the implementer.

## Execution Handoff

Plan complete and saved to `docs/27-reimplementation-task-plan.md`.

Two execution options:

- **Subagent-driven** — a fresh subagent implements each task, a fresh reviewer checks it, then a whole-branch review. Best when tasks touch shared interfaces (Tasks 2–12) and a mistake is expensive.
- **Native** — implement all tasks in one session, then one fresh review. Cheaper and faster; suitable if you are following the docs closely.

Recommended: **Subagent-driven** for Tasks 2–12 (event/wire/loop/permissions), then **Native** for surfaces (Tasks 16–19), because the core interfaces are where drift hurts most.
