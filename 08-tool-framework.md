# 08 · 工具框架

> 目标：解释一个工具从"注册"到"被模型调用"到"结果进入上下文"的完整契约。重实现时，这是权限、并发、截断、UI 展示的共同基础。

---

## 1. 工具契约

`packages/agent-core-v2/src/tool/toolContract.ts`：

```ts
interface ExecutableTool<Input = unknown> {
  name: string;
  description: string;
  parameters: Record<string, unknown>; // JSON Schema
  resolveExecution(input: Input): ToolExecution | Promise<ToolExecution>;
}

type ToolExecution = RunnableToolExecution | ExecutableToolErrorResult;

interface RunnableToolExecution {
  isError?: false;
  accesses?: ToolAccesses;       // 读/写/搜索声明
  display?: ToolInputDisplay;    // 审批/UI 展示
  description?: string;          // "Running: ..." 这类摘要
  approvalRule: string;          // 会话规则模式
  matchesRule?: (ruleArgs: string) => boolean;
  execute(ctx: ExecutableToolContext): Promise<ExecutableToolResult>;
  stopping?(context): void;
}
```

结果：

```ts
interface ExecutableToolSuccessResult {
  output: string | ContentPart[];
  isError?: false;
  truncated?: boolean;
  note?: string;
  spill?: { outputPath?; totalChars?; suffix? };
  spillExempt?: true;
  stopTurn?: boolean;
  stopTurnReason?: string;
}
```

`resolveExecution` 的职责：

1. 解析/校验输入；
2. 决定展示（command、diff、plan_review…）；
3. 声明访问（供并发/权限使用）；
4. 返回 `approvalRule`（供"本会话允许"缓存）；
5. 延迟到 `execute` 才真正干活。

**权限系统只看 `display/accesses/approvalRule`，不看工具内部实现**。这是解耦关键。

---

## 2. 注册

`agent/toolRegistry/toolContribution.ts`：

```ts
registerAgentToolService(id, ctor, {
  name: "Bash",
  source: "builtin",          // builtin | user | mcp
  disclosure: "inline",       // inline | deferred
  when?: (accessor) => boolean,
  requiredEnvironmentCapabilities?: ("fs" | "process")[],
  domain: "os",
});
```

- 工具是 Agent scope 的 `OnDemand` 服务；
- 同时登记到全局 `_agentToolContributions`；
- `BuiltinToolAssemblyService` 在 App scope 把 contributions 提供给 collection；
- `ToolRegistry`（Agent scope）按需实例化，支持 `register/unregister/list/resolve`。

`ToolRegistry`：

```ts
interface IAgentToolRegistryService {
  register(tool, options?): ToolRegistration;
  unregister(name): boolean;
  list(): readonly ToolInfo[];
  listReferences(): readonly ToolReference[];
  resolve(name): ExecutableTool | undefined;
  onDidChange(listener): IDisposable;
}
```

---

## 3. 工具可见性（tool policy）

`agent/toolPolicy/evaluate.ts`：

```ts
function isToolActiveComposed(layers, name, source): boolean {
  return (
    isToolActive({ disallowedTools: layers.workspaceDisabledTools }, name, source) &&
    isToolActive(layers.profile, name, source) &&
    isToolActive({ tools: global.enabled || undefined, disallowedTools: global.disabled }, name, source) &&
    isToolActive({ disallowedTools: layers.sessionDisabledTools }, name, source)
  );
}
```

四层：

| 层 | 来源 |
| --- | --- |
| workspace | 工作区禁用列表 |
| profile | agent 文件的 `tools` / `disallowedTools` |
| global | `[tools].enabled/disabled` |
| session | 运行时禁用 |

MCP 工具名支持 `picomatch` 通配（`mcp__*`）；非 MCP 工具只支持精确名（避免 `Bash*` 这类危险通配）。

`findInactiveToolPatterns` 会诊断：

- `wildcard-not-mcp`：非 MCP 用了通配；
- `incomplete-mcp-name`：`mcp__server` 缺 `__tool`；
- `unknown-tool`：未知工具名。

---

## 4. 执行器

`packages/core/src/tool/tool-executor.ts`（983 行）+ Agent 适配层。

### 4.1 阶段

一次工具调用经历：

```
resolve → resolveExecution
  → starting hook（权限）
  → before-execute
  → execute
  → output accumulate
  → did-execute hook（PostToolUse）
  → truncate/spill
  → 返回
```

`ToolExecutionOutcome`：

```
executed | preflight-rejected | resolution-failed | vetoed
| aborted | synthetic | skipped
```

### 4.2 钩子

```ts
interface ToolExecutorHook {
  starting?(ctx): void;        // 权限
  beforeExecute?(ctx): void;
  afterExecute?(ctx): void;
  didExecute?(ctx): void;
}
```

Agent 层还注册：

- `AgentPermissionGate`（starting）
- `externalHooks`（PreToolUse/PostToolUse）
- 工具策略、MCP 加载等待

### 4.3 参数校验

- `args-validator.ts`：JSON Schema → 校验；
- `tool-args-parse.ts`：从模型返回的字符串解析 JSON（容忍前后空白/代码块）；
- 解析失败 → `preflight-rejected`，错误文本回给模型，不执行。

### 4.4 中断

- 每个调用有 `AbortSignal`；
- `stopping()` 允许工具在中断时清理；
- 已 detach 的工具不被 turn abort 杀掉。

---

## 5. 输出累积与截断

`output-accumulator.ts` 收集 stdout/stderr/progress，支持 `replace` 和 `fields`。

`ToolResultTruncationService`：

| 常量 | 值 |
| --- | --- |
| 内联上限 | 50,000 字符 |
| 落盘保留上限 | 10,000,000 字符 |
| 预览头 | 4,096 字符 |
| 预览尾 | 1,024 字符 |
| 单行上限 | 2,000 字符 |

流程：

1. 若 `spillExempt` → 原样返回；
2. 文本 ≤ 50k → 原样；
3. 否则保留最多 10M；
4. 按行 shape（超长行截断）；
5. 落盘到 agent scope 的 `tool-results`；
6. 返回"头 + 尾 + 落盘路径"的预览；
7. 若落盘失败 → 返回"未持久化"的降级文本。

模型看到的是预览 + 文件指针；需要全文时用 `Read` 读文件。

---

## 6. 去重与重复调用

`agent/toolDedupe/`：

- 记录跨 step 的相同调用；
- 达到阈值 → `REPEAT_BREAKER_STOP_REASON`，结束 turn；
- 避免模型陷入"反复读同一文件"。

重实现时可以先做简单的 `(tool, args)` 计数 + 阈值；不要一上来做语义去重。

---

## 7. 后台任务与 detach

`ExecutableToolContext` 提供：

```ts
detach?: (text, brief?) => void;
onDetach?: (acknowledge) => { content; brief? };
taskId?: string;
```

流程：

1. 工具启动子任务；
2. `detach()` 立即返回 ack（task id、phase）；
3. 工具的结果稍后通过 `completeTool` 进入 notify；
4. `TaskList/TaskOutput/TaskStop/WaitFor` 管理这些任务。

`Bash` 的默认行为：前台超时**不杀**，转为后台任务（`bash_auto_background_on_timeout` 可关）。后台超时默认 600s（`bash_task_timeout_s`，0 = 无超时）。

---

## 8. 并发与访问冲突

`ToolScheduler`：

- 每个 `RunnableToolExecution` 声明 `ToolAccesses`；
- `ToolAccesses.file(operation, path, {recursive})`；
- 冲突规则：同一路径的读/写/搜索，按操作组合判断；
- 冲突 → 排队；不冲突 → 并行；
- lease 按 `turnId:step` 批次获取。

工具应尽量声明**最小**访问：

- `Read`：`readFile(path)`；
- `Grep`：`searchTree(path)`；
- `Write`：`writeFile(path)`；
- `Bash`：`ToolAccesses.all()`（无法静态确定）。

---

## 9. 结果进入上下文

执行器返回 `ExecutableToolResult`，由 loop 转成 `tool` 消息：

- `renderToolResultForModel` 处理错误/空输出/耗时；
- `toolCallId` 必须与 call 匹配；
- 结果顺序按 call 顺序；
- 大结果由截断服务处理。

---

## 10. 不变量

1. 每个工具必须有唯一 `name`。
2. `parameters` 必须是 object schema；工具参数校验失败不能执行。
3. `resolveExecution` 不能有副作用（除了读文件等只读准备）。
4. `approvalRule` 必须能唯一标识"这类调用"（Bash 用命令，Read 用路径）。
5. `accesses` 必须真实；谎报会导致并发冲突漏检。
6. 工具不能直接写 wire；只能返回结果。
7. 大输出必须走 spill，不能把 10MB 塞进模型上下文。
8. detach 后不能再依赖 turn 的 signal。

---

## 11. 重实现最小版

1. 定义 `ExecutableTool` + `ToolExecution` + `ToolAccesses`。
2. 实现注册表（name → ctor，按需实例化）。
3. 实现 executor：校验 → resolve → execute → truncate。
4. 加 `accesses` 与串行调度。
5. 加权限钩子（先只做 allow/ask）。
6. 加 detach/后台任务。
7. 加去重与重复熔断。
8. 最后加 MCP/动态工具。

测试重点：

- 参数非法 → 不执行，错误回模型；
- 两个写同一文件 → 串行；
- 50k 以上输出落盘且可 Read；
- detach 后 turn 结束，任务完成仍通知；
- 重复调用触发熔断；
- 工具抛错 → `isError` 结果而不是崩溃。

---

## 12. 证据索引

- `packages/agent-core-v2/src/tool/{toolContract,path-access,output-accumulator,args-validator,tool-args-parse,rule-match,input-schema,toolInputDisplay}.ts`
- `packages/agent-core-v2/src/tool/executor/*`
- `packages/core/src/tool/{tool-executor,args,validate,output-store,types}.ts`
- `packages/agent-core-v2/src/agent/toolRegistry/*`
- `packages/agent-core-v2/src/agent/toolPolicy/*`
- `packages/agent-core-v2/src/agent/toolExecutor/{toolScheduler,toolExecutorService,toolHooks}.ts`
- `packages/agent-core-v2/src/agent/toolResultTruncation/*`
- `packages/agent-core-v2/src/agent/toolDedupe/*`
