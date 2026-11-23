# 14A · 权限系统：逐函数实现笔记

> 配套 `14-permissions.md`。本文按**文件 → 类 → 方法**展开，给出签名、依赖、判定顺序、边界与测试点。  
> 目标：读者可以照着写出同样的类与方法，而不只是理解概念。

---

## 0. 调用链总览

```
ToolExecutor
  └─ hook "permitting"
      └─ runLegacyStarting
          └─ AgentPermissionGate.adjudicate(event)
              ├─ policyService.evaluate(context)
              │   └─ for policy of policies: policy.evaluate(context) → first non-undefined
              ├─ telemetry: permission_policy_decision
              ├─ result.kind === "ask"
              │   └─ event.waitUntil(() => toolApproval.requestToolApproval(...))
              ├─ result.kind === "approve"
              │   └─ event.pass(result.executionMetadata)
              └─ result.kind === "deny" | "result"
                  └─ toolApproval.resolvePermissionResolution(...)
                      └─ event.veto(...)
```

`AgentPermissionGate.authorize(context)` 是同一套逻辑的非事件版本，供其它调用方直接使用。

---

## 1. 文件清单

| 文件 | 职责 |
| --- | --- |
| `agent/permissionMode/permissionMode.ts` | 模式服务接口 |
| `agent/permissionMode/permissionModeService.ts` | 模式读写、广播、遥测 |
| `agent/permissionMode/permissionModeOps.ts` | `PermissionSetMode` 事件与 replayable state |
| `agent/permissionMode/injection/permissionModeInjection.ts` | auto 进入/退出提醒 |
| `agent/permissionPolicy/types.ts` | 模式/规则/决策/审批类型 |
| `agent/permissionPolicy/permissionPolicy.ts` | 策略服务接口 |
| `agent/permissionPolicy/permissionPolicyService.ts` | 策略链组装与短路 |
| `agent/permissionPolicy/policies/*.ts` | 13 条策略 |
| `agent/permissionRules/permissionRules.ts` | 规则/服务接口 |
| `agent/permissionRules/permissionRulesService.ts` | 规则合并、写入、审批记录 |
| `agent/permissionRules/permissionRulesOps.ts` | 规则事件与 replayable state |
| `agent/permissionRules/configSection.ts` | `[permission]` schema/TOML/env |
| `agent/permissionRules/matchesRule.ts` | 模式解析与匹配 |
| `agent/permissionRules/projectPermissionRules.ts` | 项目规则接口 |
| `agent/permissionRules/projectPermissionRulesService.ts` | 项目规则加载/追加 |
| `agent/permissionGate/permissionGate.ts` | 门面接口 |
| `agent/permissionGate/permissionGateService.ts` | 钩子注册与裁决 |
| `agent/toolApproval/toolApproval.ts` | 审批接口 |
| `agent/toolApproval/toolApprovalService.ts` | 人工审批、会话缓存、遥测 |
| `features/plan/exitPlanModeReview.ts` | 计划审阅强制审批 |

---

## 2. 类型（逐字段）

```ts
type PermissionMode = "manual" | "yolo" | "auto";

type PermissionDecision = "approve" | "deny" | "ask";

interface PermissionRule {
  decision: "allow" | "deny" | "ask";
  scope: "turn-override" | "session-runtime" | "project" | "user";
  pattern: string;
  reason?: string;
}

interface PermissionData {
  mode: PermissionMode;
  rules: PermissionRule[];
}

interface ApprovalRequest {
  id?: string;
  toolCallId: string;
  toolName: string;
  action: string;
  display: ToolInputDisplay;
}

interface ApprovalResponse {
  decision: "approved" | "rejected" | "cancelled";
  scope?: "session";
  feedback?: string;
  selectedLabel?: string;
}

type PermissionPolicyResult =
  | { kind: "approve"; reason?: Record<string, ...>; executionMetadata?: unknown }
  | { kind: "deny"; reason?: Record<string, ...>; message?: string }
  | {
      kind: "ask";
      reason?: Record<string, ...>;
      resolveApproval?: (r: ApprovalResponse) => PermissionPolicyResolution | undefined;
      resolveError?: (e: unknown) => PermissionPolicyResolution | undefined;
      resolveNonInteractive?: () => Exclude<PermissionPolicyResolution, { kind: "ask" }>;
    };
```

`AuthorizationRequest.display` 是 `ToolInputDisplay`，让 UI 能渲染 command/diff/plan_review 等。

---

## 3. 模式服务

### 3.1 `AgentPermissionModeService`

**构造依赖**：`IEventDispatcher`、`IAgentScopeContext`、`IAgentLifecycleService`、`IAgentReminderService`、`ITelemetryService`、`IAgentStateService`、`IBootstrapService`。

**构造行为**：

1. `agentState.contributeState(permissionModeKey)`；
2. `agentState.contributeState(permissionModeConfiguredKey)`；
3. 若 `KIMI_CODE_PERMISSION_MODE_REMINDER !== false`，注册 `PermissionModeInjection`。

**`get mode()`**：`agentState.get(permissionModeKey)`。

**`setMode(mode)`**：

```
previous = this.mode
changed = mode !== previous
if (!changed && configuredKey) return
dispatch(PermissionSetMode { agentId, mode })
dispatch(AgentStatusUpdated { agentId, permission: mode })
if (changed) emit onDidChangeMode({ mode, previousMode })
```

**`setModeAndBroadcast(mode)`**：

```
wasYolo/wasAuto 记录
setMode(mode)
if (agentId === MAIN_AGENT_ID) agentLifecycle.broadcastPermissionMode(mode)
yolo 变化 → telemetry "yolo_toggle"
auto 变化 → telemetry "afk_toggle"
```

注意：`setMode` 在"模式没变但从未显式配置"时仍会派发一次，用于标记 `configured=true`。这影响恢复时是否覆盖默认值。

### 3.2 `permissionModeOps`

```ts
class PermissionSetMode extends AgentEvent2<{agentId, mode}> {
  static type = "permission.set_mode";
  static durable = true;
}

const permissionModeKey = defineState("permissionMode", () => "manual")
  .replayable({ schema })
  .on(PermissionSetMode, (_s, e) => e.mode);

const permissionModeConfiguredKey = defineState("permissionMode.configured", () => false)
  .replayable({ schema })
  .on(PermissionSetMode, () => true);
```

### 3.3 `PermissionModeInjection`

状态：`permissionMode.lastMode`（非 replayable 的普通 state）。

`reminder({ injectedPositions })`：

```
current = mode
previous = lastMode
if (current === previous) {
  if (injectedPositions.length > 0 || current !== "auto") return undefined
  return AUTO_MODE_ENTER_REMINDER
}
lastMode = current
if (current === "auto") return AUTO_MODE_ENTER_REMINDER
if (previous === "auto") return AUTO_MODE_EXIT_REMINDER
return undefined
```

用途：auto 模式进入/退出时给模型一次明确的行为提示（"不要问人/可以问了"）。

---

## 4. 策略服务

### 4.1 `AgentPermissionPolicyService`

**构造依赖**：`IInstantiationService`、`IBootstrapService`。

**`policies` 数组（顺序即语义）**：

```ts
[
  AutoModeAskUserQuestionDeny,
  UserConfiguredDeny,
  ...(bootstrap.args.nonInteractive ? [] : [DangerousCommandAsk]),
  AutoModeApprove,
  SessionApprovalHistory,
  UserConfiguredAsk,
  UserConfiguredAllow,
  SensitiveFileAccessAsk,
  GitControlPathAccessAsk,
  YoloModeApprove,
  DefaultToolApprove,
  GitCwdWriteApprove,
  FallbackAsk,
]
```

**`evaluate(context)`**：

```
for (const policy of this.policies) {
  const result = await policy.evaluate(context)
  if (result !== undefined) return { policyName: policy.name, result }
}
return undefined
```

`undefined` 表示"没有策略处理"，通常不应发生，因为最后一条 `fallback-ask` 永远返回 `ask`。

---

## 5. 13 条策略逐函数

### 5.1 `AutoModeAskUserQuestionDenyPermissionPolicyService`

```ts
name = "auto-mode-ask-user-question-deny"

evaluate(context) {
  if (modeService.mode !== "auto") return undefined
  if (context.toolCall.name !== "AskUserQuestion") return undefined
  return {
    kind: "deny",
    message: "AskUserQuestion is disabled while auto permission mode is active; decide and continue.",
  }
}
```

边界：只针对 `AskUserQuestion`；其它"提问型"工具不受影响。

### 5.2 `UserConfiguredDenyPermissionPolicyService`

```ts
evaluate(context) {
  return evaluateUserConfiguredRule(context, "deny", rulesService)
}
```

### 5.3 `DangerousCommandAskPermissionPolicyService`

**依赖**：`IBashParserService`、`IAgentPermissionModeService`、`IConfigService`。

**常量**：

```ts
PARSE_OPTIONS = { timeoutMs: 500, maxNodes: 10_000 }
MAX_NESTED_SHELL_DEPTH = 4
UNSAFE_OPERAND = /[$`*?[\]~]/
```

**`evaluate`**：

```
if (!isDangerousCommandGuardEnabled(config)) return undefined
if (mode === "auto") return undefined
if (toolCall.name !== "Bash") return undefined
command = bashCommandText(args)
verdict = command === undefined
  ? { kind: "unanalyzable" }
  : analyzeSource(command, 0, s => parser.parse(s, PARSE_OPTIONS))
if (verdict === undefined) return undefined
if (verdict.kind === "dangerous") return { kind: "ask", reason: { dangerous_command: verdict.command } }
if (mode === "yolo") return undefined
return { kind: "ask", reason: { unanalyzable_command: true } }
```

详细分析算法见 `15A`。

### 5.4 `AutoModeApprovePermissionPolicyService`

```ts
evaluate() {
  return modeService.mode === "auto" ? { kind: "approve" } : undefined
}
```

### 5.5 `SessionApprovalHistoryPermissionPolicyService`

**依赖**：`IAgentPermissionRulesService`。

```ts
evaluate(context) {
  for (const pattern of rulesService.sessionApprovalRulePatterns) {
    const match = matchPermissionRule({
      rule: { decision: "allow", scope: "session-runtime", pattern, reason: "approve for session" },
      toolName: context.toolCall.name,
      execution: context.execution,
    })
    if (match) return { kind: "approve", reason: { has_rule_args: match.hasRuleArgs, match_strategy: match.strategy } }
  }
  return undefined
}
```

注意：这里用 `sessionApprovalRulePatterns`（字符串数组），不是完整规则对象；`scope` 仅用于传给匹配器。

### 5.6 `UserConfiguredAskPermissionPolicyService`

```ts
evaluate(context) {
  return evaluateUserConfiguredRule(context, "ask", rulesService)
}
```

### 5.7 `UserConfiguredAllowPermissionPolicyService`

```ts
evaluate(context) {
  return evaluateUserConfiguredRule(context, "allow", rulesService)
}
```

### 5.8 `evaluateUserConfiguredRule`（共享）

```ts
const USER_CONFIGURED_SCOPES = new Set(["turn-override", "project", "user"])

function evaluateUserConfiguredRule(context, decision, rulesService) {
  const match = firstMatchingRule(context, decision, rulesService, USER_CONFIGURED_SCOPES)
  if (!match) return undefined
  if (decision === "deny") return { kind: "deny", message: defaultPermissionRuleDenyMessage(tool, match.rule.reason) }
  if (decision === "ask") return { kind: "ask" }
  return { kind: "approve" }
}
```

关键：**`session-runtime` 被排除**，由第 5 条单独处理。这样"本会话允许"不会被用户 allow/ask 规则覆盖。

`firstMatchingRule` 遍历 `rulesService.rules`，按声明顺序，第一条 `decision` 匹配且 `matchPermissionRule` 命中的规则胜出。

`defaultPermissionRuleDenyMessage`：

```
Tool "<name>" was denied by permission rule.[ Reason: <reason>]
```

### 5.9 `SensitiveFileAccessAskPermissionPolicyService`

```ts
evaluate(context) {
  if (!isDangerousCommandGuardEnabled(config)) return undefined
  const access = fileAccesses(context).find(a => isSensitiveFile(a.path))
  return access === undefined ? undefined : { kind: "ask" }
}
```

注意：真正的**硬阻断**在 `resolvePathAccess`（见 16 篇），这里只是给"路径守卫放行但敏感检查被关闭"的场景兜底。默认 `checkSensitive=true` 时，工具会在到达策略前就抛错。

### 5.10 `GitControlPathAccessAskPermissionPolicyService`

**依赖**：`IAgentEnvironmentService`、`ISessionWorkspaceContext`、`IConfigService`。

```ts
evaluate(context) {
  if (!guardEnabled) return undefined
  cwd = workspace.workDir
  if (!cwd) return undefined
  lease = acquireEnvironmentLease(environment)
  if (!lease) return undefined
  try {
    pathClass = lease.environment.host?.pathClass
    if (!pathClass) return undefined
    accesses = fileAccesses(context)
    if (empty) return undefined

    // 1) 路径里直接含 .git 组件
    if (accesses.find(a => hasGitPathComponent(a.path, cwd, pathClass))) return { kind: "ask" }

    // 2) 规范化到 worktree 的 dotGitPath / controlDirPath
    fs = lease.environment.fs
    if (!fs) return undefined
    marker = await findGitWorkTree(fs, cwd)
    if (!marker) return undefined
    return accesses.find(a => isGitControlPath(a.path, marker, pathClass))
      ? { kind: "ask" } : undefined
  } finally { lease.dispose() }
}
```

`path-utils.ts`：

```ts
function fileAccesses(context) {
  return context.execution.accesses?.filter(a => a.kind === "file") ?? []
}

function writeFileAccesses(context) {
  return fileAccesses(context).filter(a => a.operation === "write" || a.operation === "readwrite")
}

function hasGitPathComponent(target, cwd, pathClass) {
  return relativePathParts(target, cwd, pathClass).some(p => p.toLowerCase() === ".git")
}

function isGitControlPath(target, marker, pathClass) {
  return isWithinDirectory(target, marker.dotGitPath, pathClass)
      || isWithinDirectory(target, marker.controlDirPath, pathClass)
}
```

注意：这条策略对**所有**文件访问生效（读也包括），且排在 yolo 放行之前。

### 5.11 `YoloModeApprovePermissionPolicyService`

```ts
evaluate() {
  return modeService.mode === "yolo" ? { kind: "approve" } : undefined
}
```

### 5.12 `DefaultToolApprovePermissionPolicyService`

```ts
const DEFAULT_APPROVE_TOOLS = new Set([
  "Read","Grep","Glob","ReadMediaFile","SetTodoList","TodoList",
  "TaskList","TaskOutput","WaitFor","CronList","WebSearch","FetchURL",
  "Agent","AgentSwarm","AskUserQuestion","NotifyUser","Skill",
  "ExitPlanMode","CreateGoal","GetGoal","SetGoalBudget","UpdateGoal",
  "select_tools",
])

evaluate(context) {
  return DEFAULT_APPROVE_TOOLS.has(context.toolCall.name) ? { kind: "approve" } : undefined
}
```

注意：`SetTodoList` 是旧名，保留兼容；`TaskStop`/`CronCreate`/`CronDelete` 不在白名单。

### 5.13 `GitCwdWriteApprovePermissionPolicyService`

**依赖**：`IAgentEnvironmentService`、`ISessionWorkspaceContext`。

```ts
evaluate(context) {
  if (tool !== "Write" && tool !== "Edit") return undefined
  lease = acquireEnvironmentLease(environment)
  if (!lease) return undefined
  try {
    if (lease.environment.host?.pathClass !== "posix") return undefined
    fs = lease.environment.fs
    if (!fs) return undefined
    cwd = workspace.workDir
    if (!cwd) return undefined
    writes = writeFileAccesses(context)
    if (empty) return undefined
    if (!writes.every(a => isWithinWorkspace(a.path, {workspaceDir: cwd, additionalDirs}, "posix")))
      return undefined
    return (await findGitWorkTree(fs, cwd)) === null ? undefined : { kind: "approve" }
  } finally { lease.dispose() }
}
```

边界：

- 仅 POSIX（Windows 不走这条）；
- 仅 Write/Edit；
- 仅工作区内（含 additionalDirs）；
- 必须有 git worktree；
- `.git` 控制路径已在第 9 条被 ask，不会到这里；
- 敏感文件已在路径解析时硬阻断。

### 5.14 `FallbackAskPermissionPolicyService`

```ts
evaluate() { return { kind: "ask" } }
```

---

## 6. 规则引擎逐函数

### 6.1 `parsePattern(pattern)`

```
trim
if empty → VALIDATION_FAILED
openIdx = indexOf("(")
if -1 → { toolName: trimmed }
if !endsWith(")") → VALIDATION_FAILED
toolName = slice(0, openIdx)
argPattern = slice(openIdx+1, -1)
if toolName empty → VALIDATION_FAILED
if argPattern empty → { toolName }
return { toolName, argPattern }
```

注意：只处理**第一对**括号；参数模式里可以再含括号（`Bash(echo (x))` 会得到 `argPattern = "echo (x"`？实际上 `endsWith(")")` + `slice` 会保留内部括号，行为是"外层第一对"）。实现时应与测试对齐。

### 6.2 `matchPermissionRule({rule, toolName, execution})`

```
try parsePattern(rule.pattern) catch → undefined
if (parsed.toolName !== "*" && !picomatch.isMatch(toolName, parsed.toolName)) return undefined
if (parsed.argPattern === undefined) return { rule, strategy: "tool_name_only", hasRuleArgs: false }
return execution.matchesRule?.(parsed.argPattern) === true
  ? { rule, strategy: "matches_rule", hasRuleArgs: true }
  : undefined
```

### 6.3 `configSection.ts`

```ts
PermissionRuleDecisionSchema = z.enum(["allow","deny","ask"])
PermissionRuleScopeSchema = z.enum(["turn-override","session-runtime","project","user"])

PermissionRuleSchema = z.object({
  decision: ...,
  scope: ...default("user"),
  pattern: z.string().min(1).refine(isValidPermissionPattern),
  reason: z.string().optional(),
})

PersistedPermissionRuleSchema = PermissionRuleSchema.extend({
  scope: z.enum(["project","user"]).default("user"),
})

PermissionConfigSchema = z.object({
  rules: z.array(PermissionRuleSchema).optional(),
  dangerousCommandGuard: z.boolean().optional(),
})
```

`readUserPermissionRules(config)`：

```
rules = config.get("permission")?.rules ?? []
return rules.filter(r => r.scope === "project" || r.scope === "user")
```

`isDangerousCommandGuardEnabled(config)`：

```
return config.get("permission")?.dangerousCommandGuard ?? true
```

`permissionFromToml(rawSnake)`：

- 把 `rules`、`deny`、`allow`、`ask` 合并成统一 `rules`；
- `{tool, match}` → `${tool}(${match})`；无 match → `${tool}`；
- `transformPermissionRule` 保留 decision/scope/reason。

`permissionToToml(value, rawSnake)`：

- 删除旧 `deny/allow/ask`；
- 写回 `rules`；
- 保留其它字段。

### 6.4 `AgentPermissionRulesService`

**`get rules()`**：

```
dedupe([
  ...agentState.get(permissionRulesKey).rules,
  ...projectRules.rules,
  ...readUserPermissionRules(config),
])
```

**`addRules(rules)`**：

```
runtime = rules.filter(scope in {turn-override, session-runtime})
user = rules.filter(scope === "user")
project = missing(projectRules.rules, rules.filter(scope === "project"))

if runtime.length: dispatch(PermissionRulesAdd)
if user.length: enqueueUserConfigWrite → config.replace("permission", {..., rules: [...existing, ...appended]})
if project.length: projectRules.appendRules(project)
await Promise.all(writes)
```

`enqueueUserConfigWrite` 用 `WeakMap<IConfigService, Promise>` 串行化写入，避免并发覆盖。

**`recordApprovalResult(record)`**：`dispatch(PermissionRecordApprovalResult({...record, agentId}))`。

### 6.5 `permissionRulesOps`

```ts
class PermissionRulesAdd extends AgentEvent2<{agentId, rules}> {
  static type = "permission.rules.add";
  // 注意：没有 durable = true
}

class PermissionRecordApprovalResult extends AgentEvent2<...> {
  static type = "permission.record_approval_result";
  static durable = true;
  static schema = ...;
}

const permissionRulesKey = defineState("permissionRules", () => ({
  rules: [],
  sessionApprovalRulePatterns: [],
}))
  .replayable({ schema })
  .on(PermissionRulesAdd, (s, e) => { if (e.rules.length) s.rules = [...s.rules, ...e.rules] })
  .on(PermissionRecordApprovalResult, (s, e) => {
    const p = e.sessionApprovalRule
    if (e.result.decision !== "approved" || e.result.scope !== "session" || p === undefined || s.sessionApprovalRulePatterns.includes(p)) return
    s.sessionApprovalRulePatterns = [...s.sessionApprovalRulePatterns, p]
  })
```

**重要**：运行时规则（`PermissionRulesAdd`）不持久化；只有"本会话允许"的 pattern 通过 durable 的 `PermissionRecordApprovalResult` 进入 replayable state。用户/项目规则通过配置文件持久化。

### 6.6 `ProjectPermissionRulesService`

**构造依赖**：`ISessionContext`、`ISessionWorkdir`、`IProjectLocalConfigService`、`IWorkspaceTrust`。

```
cwd = workdir.cwd ?? session.cwd
local = workdir.environmentId === "local"
ready = local ? enqueue(loadFromDisk) : Promise.resolve()
```

**`get rules()`**：`currentRules`。

**`appendRules(rules)`**：

```
if (!local) reject("Project permission rules are not supported in remote environments")
return enqueue(async () => {
  parseProjectPermissionRules(rules)   // 校验
  appended = await localConfig.appendPermissionRules(cwd, rules)
  if (trusted) currentRules = parseProjectPermissionRules(appended.rules)
})
```

**`loadFromDisk()`**：

```
trusted = await trust.get()
if (!trusted) return
loaded = await localConfig.readPermissionRules(cwd)
currentRules = parseProjectPermissionRules(loaded.rules)
```

**`parseProjectPermissionRules(raw)`**：

```
try raw.map(entry =>
  PersistedPermissionRuleSchema.parse(
    isObject(entry) && !Array.isArray(entry) ? { scope: "project", ...entry } : entry
  ))
catch → CONFIG_INVALID "Invalid project permission rule"
```

注意：项目规则只接受 `project`/`user` scope；省略 scope 默认 `project`。

---

## 7. 门面与审批逐函数

### 7.1 `AgentPermissionGate`

**构造**：

```ts
toolExecutor.registerHook({
  permitting: async (context) => {
    const decision = await runLegacyStarting(context, (event) => this.adjudicate(event))
    return decision !== undefined && "skipPermitting" in decision ? undefined : decision
  },
})
```

`data()`：`{ mode, rules: [...rulesService.rules] }`。

**`adjudicate(event)`**：

```
evaluation = await policyService.evaluate(event)
if (!evaluation) return
telemetry.track2("permission_policy_decision", {
  turn_id, tool_call_id, policy_name, tool_name, permission_mode, decision, ...reason
})
switch (evaluation.result.kind) {
  case "ask":
    event.waitUntil(() => toolApproval.requestToolApproval(event, result, policyName))
    return
  case "approve":
    event.pass(result.executionMetadata)
    return
  default:
    resolved = await toolApproval.resolvePermissionResolution(result, event, policyName)
    if (resolved?.veto !== undefined) event.veto(resolved.veto)
}
```

**`authorize(context)`**：同上，但返回 `BeforeExecuteDecision | undefined`，不操作 event。

### 7.2 `AgentToolApprovalService`

**`resolvePermissionResolution(result, context, origin)`**：

```
approve → executionMetadata ? { executionMetadata } : undefined
deny    → { veto: denyToolExecution(formatDenyMessage(...)) }
ask     → requestToolApproval(context, result, origin)
result  → { veto: result.result }
```

**`requestToolApproval(context, result, origin)`**：

```
if (bootstrap.args.nonInteractive) {
  return resolvePermissionResolution(
    result.resolveNonInteractive?.() ?? {
      kind: "deny",
      message: `Tool "<name>" needs approval, but nobody can approve it in a non-interactive run.`,
    }, context, origin)
}

action = execution.description ?? `Approve <name>`
display = execution.display ?? { kind: "generic", summary: action, detail: args }
approvalRequest = {
  id: `approval_${randomUUID()}`,
  sessionId, agentId, turnId, toolCallId, toolName: name, action, display,
}
approvalContext = { ...approvalRequest, toolInput: args }
tags = { agentId, sessionId, turnId, toolCallId }

dispatch(PermissionApprovalRequested(approvalContext))
try {
  response = await abortable(
    interactions.request({ id, kind: "approval", payload: approvalRequest, tags }),
    signal,
  )
  signal.throwIfAborted()
} catch (error) {
  if (isUserCancellation(error)) throw error
  telemetry(... result: "error")
  dispatch(PermissionApprovalResolved({...decision:"error", error}))
  resolved = result.resolveError?.(error)
  if (resolved) return resolvePermissionResolution(resolved, context, origin)
  throw error
}

sessionApprovalRule =
  response.decision === "approved" && response.scope === "session"
    ? execution.approvalRule
    : undefined

dispatch(PermissionApprovalResolved({...approvalContext, ...response}))
rulesService.recordApprovalResult({ turnId, toolCallId, toolName, action, sessionApprovalRule, result: response })
telemetry(...)

resolved = result.resolveApproval?.(response)
if (resolved) return resolvePermissionResolution(resolved, context, origin)
if (response.decision === "approved") return undefined
return { veto: denyToolExecution(formatApprovalRejectionMessage(name, response)) }
```

**`formatApprovalRejectionMessage`**：

```
prefix = cancelled
  ? `Tool "<name>" was not run because the approval request was cancelled.`
  : `Tool "<name>" was not run because the user rejected the approval request.`
suffix = feedback ? ` Reason: <feedback>` : ""
return prefix + suffix
```

**`formatDenyMessage`**：原样返回，子代理追加"换个办法"。

**`usesWorkerRejectionGuidance()`**：`agentId !== "main"`。

### 7.3 `ExitPlanModeReview`

**`requestApproval(context)`**：

```
display = context.execution.display
if (display?.kind !== "plan_review") return undefined
if (!display.plan.trim()) return undefined
telemetry plan_submitted
return toolApproval.requestToolApproval(
  context,
  {
    kind: "ask",
    reason: { has_options: display.options?.length >= 2 },
    resolveApproval: r => this.approvalResult(r, display),
    resolveNonInteractive: () => ({ kind: "deny", message: PLAN_REVIEW_NON_INTERACTIVE_MESSAGE }),
  },
  "exit-plan-mode-review-ask",
)
```

**`approvalResult(response, display)`**：

```
if (decision !== "approved") return rejectedApprovalResult(response)
selected = options.find(o => o.label === selectedLabel)
plan.exit()
telemetry plan_resolved
optionPrefix = selected ? `Selected approach: <label>\nExecute ONLY the selected approach...` : ""
savedTo = display.path ? `Plan saved to: <path>\n\n` : ""
return { kind: "result", result: { isError: false, output:
  `Exited plan mode. ${optionPrefix}${savedTo}## Approved Plan:\n${display.plan}` } }
```

**`rejectedApprovalResult`**：

- `cancelled` → 结果 "Plan approval dismissed. Plan mode remains active."
- `selectedLabel === "Reject and Exit"` → `plan.exit()` + `isError:true` "Plan rejected by user. Plan mode deactivated."
- `selectedLabel === "Revise"` 或有 feedback → Plan 保持，返回反馈文本。
- 其它 → Plan 保持，返回 "Plan rejected..."。

---

## 8. 状态与事件总表

| 事件/状态 | durable | 作用 |
| --- | --- | --- |
| `PermissionSetMode` | 是 | 记录模式 |
| `permissionMode` | replayable | 当前模式 |
| `permissionMode.configured` | replayable | 是否显式配置 |
| `PermissionRulesAdd` | **否** | 运行时规则 |
| `permissionRules` | replayable | 规则 + session patterns |
| `PermissionRecordApprovalResult` | 是 | 审批结果；session 规则来源 |
| `PermissionApprovalRequested` | 否（observable） | UI 审批请求 |
| `PermissionApprovalResolved` | 否（observable） | UI 审批结果 |

---

## 9. 单元测试清单（按函数）

| 测试 | 断言 |
| --- | --- |
| `setMode` 相同值 + 未配置 | 仍派发一次 |
| `setMode` 相同值 + 已配置 | 不派发 |
| `setModeAndBroadcast` 非 main | 不广播 |
| 策略链顺序 | 每个场景命中预期 `policyName` |
| auto + AskUserQuestion | deny #1 |
| auto + Bash rm | approve #4（危险策略不跑） |
| yolo + Bash rm | ask #3 |
| yolo + 敏感文件 | ask #8 |
| yolo + `.git/config` | ask #9 |
| manual + git 内 Write | approve #12 |
| manual + 工作区外 Write | ask #13 |
| manual + Bash | ask #13 |
| deny 规则 + yolo | deny #2 |
| ask 规则 + allow 规则 | ask #6 |
| session pattern 命中 | approve #5 |
| `parsePattern` 空/缺括号 | 抛 `VALIDATION_FAILED` |
| `matchPermissionRule` 通配 | `*`/picomatch 行为 |
| 项目规则未信任 | 不加载 |
| 项目规则远端 | append 拒绝 |
| 非交互 ask | deny |
| 非交互 + `resolveNonInteractive` | 用其结果 |
| session 批准 | 写入 pattern，下次命中 #5 |
| plan review 非交互 | deny，Plan 保持 |

---

## 10. 实现顺序

1. 类型 + 模式 state/event。
2. 规则解析/匹配 + 规则服务。
3. 13 条策略（先 4/10/13，再 2/5/6/7，再 3/8/9/12，最后 1）。
4. 门面 + 审批服务。
5. 计划审阅。
6. 遥测与 wire 事件。
7. 子代理/广播。

> 先做 4/10/13 就能跑通"auto/yolo/manual + 只读白名单"；危险命令和 git 规则可以后补，但**顺序不要改**。

---

## 11. 证据索引

- `agent/permissionMode/{permissionMode,permissionModeService,permissionModeOps}.ts`
- `agent/permissionMode/injection/permissionModeInjection.ts`
- `agent/permissionPolicy/{permissionPolicy,permissionPolicyService,types}.ts`
- `agent/permissionPolicy/policies/*.ts`
- `agent/permissionRules/{permissionRules,permissionRulesService,permissionRulesOps,configSection,matchesRule,projectPermissionRules,projectPermissionRulesService}.ts`
- `agent/permissionGate/{permissionGate,permissionGateService}.ts`
- `agent/toolApproval/{toolApproval,toolApprovalService}.ts`
- `features/plan/exitPlanModeReview.ts`
