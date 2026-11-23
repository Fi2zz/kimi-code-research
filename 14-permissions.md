# 14 · 权限系统：模式、策略链与规则

> 这是整套调研的核心。Kimi Code 的权限不是"if 一堆布尔"，而是**一个有序策略链 + 规则引擎 + 路径/命令守卫 + 人工审批**。重实现时，顺序就是语义，不能随意调整。

---

## 1. 三种模式

`agent/permissionMode/`：

```ts
type PermissionMode = "manual" | "yolo" | "auto";
```

| 模式 | CLI 名称 | 行为 |
| --- | --- | --- |
| `manual` | Always Ask | 只读/白名单自动；写/执行逐个问；内置守卫生效 |
| `yolo` | Ask When Needed | 常规工具自动；危险命令/敏感文件/git 控制路径仍问；Plan 退出仍确认 |
| `auto` | Never Ask | 全自动；额外拒绝 `AskUserQuestion`；Plan 审阅仍保留 |

配置：`default_permission_mode`；CLI `-y/--yolo`、`--auto`；TUI `/yolo`、`/auto`。

模式是**每 agent** 的状态；主代理切换会广播给子代理（除非 profile 固定模式，如 Tower worker）。

---

## 2. 策略链：13 条，first-match-wins

`agent/permissionPolicy/permissionPolicyService.ts` 的构造顺序：

```
 1 auto-mode-ask-user-question-deny
 2 user-configured-deny
 3 dangerous-command-ask
 4 auto-mode-approve
 5 session-approval-history
 6 user-configured-ask
 7 user-configured-allow
 8 sensitive-file-access-ask
 9 git-control-path-access-ask
10 yolo-mode-approve
11 default-tool-approve
12 git-cwd-write-approve
13 fallback-ask
```

`evaluate()` 逐条调用，第一条返回非 undefined 的结果即终止。

### 为什么顺序是这样

- **deny 永远最先**（除 auto 的 AskUserQuestion 特例）：用户明确禁止不能被模式绕过。
- **危险命令在 auto/yolo 放行之前**：所以 yolo 仍会问；auto 在第 4 条就放行了，所以不问。
- **auto 放行在第 4 条**：早于规则 ask/allow，晚于 deny 和危险命令（但危险命令在 auto 下自身返回 undefined）。
- **session 缓存在 ask 规则之前**：用户"本会话允许"优先于"每次都问"的规则。
- **用户 allow 在敏感文件之前**：但敏感文件是硬阻断（见 16 篇），allow 也读不到。
- **yolo 放行在默认白名单之前**：yolo 下所有常规工具都过。
- **git-cwd-write 在 fallback 之前**：manual 下 git 工作区内的编辑自动放行。
- **fallback 最后**：其余一律问。

### 决策类型

```ts
type PermissionDecision = "approve" | "deny" | "ask";

type PermissionPolicyResult =
  | { kind: "approve"; reason?; executionMetadata? }
  | { kind: "deny"; reason?; message? }
  | { kind: "ask"; reason?; resolveApproval?; resolveError?; resolveNonInteractive? };
```

`ask` 可以带：
- `resolveApproval(response)`：把用户的批准/拒绝转换成结果；
- `resolveNonInteractive()`：非交互模式的默认（通常是 deny）；
- `resolveError(error)`：审批过程出错时的处理。

---

## 3. 每条策略的具体逻辑

### 3.1 auto-mode-ask-user-question-deny

```
if mode !== "auto" → undefined
if tool !== "AskUserQuestion" → undefined
→ deny "decide and continue"
```

目的：auto 模式是无人值守，模型不能停下来问人。

### 3.2 user-configured-deny

遍历用户规则中 `decision = deny`，用 `matchPermissionRule` 匹配；命中 → deny，消息带 reason。

### 3.3 dangerous-command-ask

```
if 守卫关闭 → undefined
if mode === "auto" → undefined
if tool !== "Bash" → undefined
解析命令 → dangerous? ask : (yolo? undefined : ask)
```

fail-closed：无法解析 → manual 下 ask；yolo 下放行。
非交互模式**不注册**这条策略（`bootstrap.args.nonInteractive` 为 true 时跳过）。

详见 15 篇。

### 3.4 auto-mode-approve

```
mode === "auto" → approve
```

### 3.5 session-approval-history

遍历 `sessionApprovalRulePatterns`，用 `allow` 规则匹配；命中 → approve。

"Approve for this session" 写入的规则就是该次调用的 `execution.approvalRule`（例如 `Bash(<该命令>)`）。

### 3.6 / 3.7 user-configured-ask / allow

同上，按 decision 遍历规则。`ask` 在 `allow` 之前，所以"ask 规则压过 allow 规则"。

### 3.8 sensitive-file-access-ask

```
if 守卫关闭 → undefined
找到第一个敏感文件访问 → ask
```

敏感文件列表见 16 篇。

### 3.9 git-control-path-access-ask

Write/Edit 目标是 `.git/` 等控制路径 → ask（在 git-cwd 自动放行之前）。

### 3.10 yolo-mode-approve

```
mode === "yolo" → approve
```

### 3.11 default-tool-approve

固定白名单：

```ts
Read, Grep, Glob, ReadMediaFile, SetTodoList, TodoList, TaskList,
TaskOutput, WaitFor, CronList, WebSearch, FetchURL, Agent, AgentSwarm,
AskUserQuestion, NotifyUser, Skill, ExitPlanMode, CreateGoal, GetGoal,
SetGoalBudget, UpdateGoal, select_tools
```

注意：
- `Write`/`Edit`/`Bash` **不在**；
- `TaskStop`/`CronCreate`/`CronDelete` **不在**；
- MCP 工具**不在**。

### 3.12 git-cwd-write-approve

```
if tool not Write/Edit → undefined
if 环境没有 fs / pathClass !== posix → undefined
if 所有写访问都在 workspace（含 additionalDirs）内
  且 cwd 在 git worktree 内
→ approve
```

这是最容易忽略的行为：**manual 模式下，git 仓库内的 `Write/Edit` 不弹审批**。理由是有 file history 可撤销 + git diff 可见。但：
- `Bash` 仍然要问；
- 工作区外仍然要问；
- `.git` 控制路径仍然要问（第 9 条更高）；
- 敏感文件硬阻断（不依赖这条）。

### 3.13 fallback-ask

一律 ask。

---

## 4. 规则模型

`agent/permissionRules/`：

```ts
interface PermissionRule {
  decision: "allow" | "deny" | "ask";
  scope: "turn-override" | "session-runtime" | "project" | "user";
  pattern: string;
  reason?: string;
}
```

配置：

```toml
[[permission.rules]]
decision = "deny"
pattern  = "Bash(rm -rf*)"
reason   = "no destructive rm"

[[permission.rules]]
decision = "allow"
pattern  = "Read"

[[permission.rules]]
decision = "ask"
pattern  = "Bash"
```

兼容旧格式：

```toml
[permission]
deny  = [{ tool = "Bash", match = "rm -rf*" }]
allow = ["Read"]
ask   = ["Bash"]
```

会被转换成统一 `rules`。

### 模式语法

```
ToolName
ToolName(arg-pattern)
```

- `ToolName` 用 `picomatch` 匹配；
- `*` 只对工具名有效；
- 非 MCP 工具名不支持通配（`findInactiveToolPatterns` 会诊断）；
- MCP 名字支持 `mcp__server__*` 这类通配；
- `arg-pattern` 由工具自己的 `matchesRule` 解释：
  - `Bash`：命令字符串（glob）；
  - `Read/Write/Edit`：路径（glob）；
  - 其它工具：可能不支持参数模式。

### 作用域

| scope | 持久化 | 说明 |
| --- | --- | --- |
| `turn-override` | 否 | 单 turn |
| `session-runtime` | 否 | 本会话（"Approve for session"） |
| `project` | 是 | `.kimi-code/local.toml`，仅本地+已信任 |
| `user` | 是 | 用户 `config.toml` |

用户配置里旧的 `turn-override`/`session-runtime` 会被静默忽略。
项目配置只接受 `project`/`user`，省略 scope 默认 `project`。
项目规则只在**本地环境且工作区已信任**时读取；远程不支持。
项目规则变更需新会话，不热更新。

### 匹配

```ts
function matchPermissionRule({rule, toolName, execution}) {
  parsed = parsePattern(rule.pattern);
  if (parsed.toolName !== "*" && !picomatch.isMatch(toolName, parsed.toolName)) return undefined;
  if (parsed.argPattern === undefined) return { strategy: "tool_name_only" };
  return execution.matchesRule?.(parsed.argPattern) === true
    ? { strategy: "matches_rule" }
    : undefined;
}
```

无法解析的模式 → 不匹配（不会导致崩溃）。

---

## 5. 审批流程

`agent/toolApproval/toolApprovalService.ts`：

1. 策略返回 `ask`；
2. 若 `nonInteractive` → `resolveNonInteractive?.()` 或默认 deny（"没人能批准"）；
3. 生成 `approval_<uuid>`；
4. 派发 `permission.approval.requested` + `interaction.request`；
5. 等待 `abortable(interactions.request(...), signal)`；
6. 响应 `approved/rejected/cancelled`，可带 `scope: "session"`、`feedback`、`selectedLabel`；
7. `scope === "session"` → 记录 `execution.approvalRule` 到 session 规则；
8. 派发 `permission.approval.resolved` + `permission.record_approval_result`；
9. 遥测 `permission_approval_result`；
10. 若策略有 `resolveApproval` → 用其结果（例如 ExitPlanMode 的"选择方案"）；
11. 批准 → 继续；拒绝/取消 → veto，返回拒绝消息。

子代理拒绝时附加："Try a different approach — don't retry the same call, don't attempt to bypass the restriction."

---

## 6. 计划审阅：绕过模式

`features/plan/exitPlanModeReview.ts`：

- `ExitPlanMode` 的 `display.kind === "plan_review"`；
- 直接调用 `requestToolApproval(..., {kind:"ask", ...}, "exit-plan-mode-review-ask")`；
- 因此 **auto/yolo 也会弹计划审阅**；
- 非交互 → deny，Plan 模式保持。

这是"模式放行"之外的一个**特性级强制审批**，重实现时不要把它混进策略链。

---

## 7. 非交互模式

`kimi -p`：

- `dangerous-command-ask` 不注册；
- 所有 `ask` 走 `resolveNonInteractive` 或 deny；
- `ExitPlanMode` 无法审阅 → 保持 Plan；
- 因此 `-p` 下能跑什么，取决于模式和规则；`auto` 最宽松。

---

## 8. 子代理与 Tower

- 子代理创建时 `setMode(caller.mode)`；
- 主代理切模式广播到所有未固定模式的 agent；
- 规则（尤其 session allow）对子代理生效；
- Tower worker/reviewer 的 profile 固定模式（`hasPinnedPermissionMode`），不会被广播覆盖；
- Tower 只有协调者支持 Ask When Needed；worker/reviewer 强制 Never Ask。

---

## 9. 可观测性

- 每条决策遥测 `permission_policy_decision`：
  ```
  turn_id, tool_call_id, policy_name, tool_name,
  permission_mode, decision, ...reason
  ```
- 审批结果遥测 `permission_approval_result`：
  ```
  result (approved/rejected/approved_for_session/cancelled/error),
  approval_surface, duration_ms, session_cache_written, has_feedback
  ```
- wire 事件：`permission.set_mode`、`permission.record_approval_result`、`interaction.request/resolved`。

这让"为什么这次被放行/拒绝"可复盘。

---

## 10. 典型场景表

| 场景 | manual | yolo | auto |
| --- | --- | --- | --- |
| `Read` | 放行 | 放行 | 放行 |
| `Write` 在 git 工作区 | **放行** | 放行 | 放行 |
| `Write` 在工作区外 | 问 | 问 | 放行 |
| `Bash ls` | 问 | 放行 | 放行 |
| `Bash rm -rf /` | 问 | 问 | 放行（危险分析在 auto 下跳过） |
| 读 `.env` | 硬阻断 | 硬阻断 | 硬阻断 |
| 写 `.git/config` | 问 | 问 | 放行 |
| `AskUserQuestion` | 放行 | 放行 | **拒绝** |
| `ExitPlanMode` | 计划审阅 | 计划审阅 | 计划审阅 |
| MCP 工具 | 问 | 放行 | 放行 |
| `TaskStop` | 问 | 放行 | 放行 |
| 用户 `deny Bash` | 拒绝 | 拒绝 | 拒绝 |

---

## 11. 不变量

1. 策略链顺序是语义；不要按"看起来更安全"重排。
2. deny 必须早于任何 approve。
3. 用户规则不能覆盖敏感文件硬阻断。
4. "Approve for session" 只缓存本次调用模式，不是整类工具。
5. `ask` 必须有可执行的 `resolveNonInteractive`，否则默认 deny。
6. 审批请求必须可取消，且取消不能导致工具执行。
7. 子代理不能比父代理更宽松（除非固定模式明确允许）。
8. 项目规则只在信任的本地工作区生效。
9. 规则模式解析失败不能崩溃，只能不匹配。
10. 任何策略决策都要能解释（policy name + reason）。

---

## 12. 重实现清单

1. 定义 `PermissionMode`、`PermissionRule`、`PermissionPolicyResult`。
2. 实现规则解析/匹配（tool name + arg pattern）。
3. 实现 13 条策略，严格按顺序。
4. 实现审批服务（事件 + 交互 + session 缓存）。
5. 实现路径守卫 + 敏感文件硬阻断。
6. 实现危险命令分析（见 15）。
7. 实现 git cwd 自动放行与 git 控制路径 ask。
8. 实现 Plan 审阅强制审批。
9. 实现非交互 deny 路径。
10. 加遥测与 wire 事件。

测试重点：

- 每条策略的单元测试（顺序、短路）；
- deny > ask > allow；
- yolo 下危险命令仍 ask；
- auto 下 AskUserQuestion deny；
- manual 下 git 内 Write 放行、工作区外 ask；
- `.git/config` ask；
- 敏感文件硬阻断；
- session 缓存命中；
- 非交互 deny；
- 子代理继承模式。

---

## 13. 证据索引

- `packages/agent-core-v2/src/agent/permissionMode/*`
- `packages/agent-core-v2/src/agent/permissionPolicy/{permissionPolicyService,policies/*}.ts`
- `packages/agent-core-v2/src/agent/permissionRules/*`
- `packages/agent-core-v2/src/agent/permissionGate/*`
- `packages/agent-core-v2/src/agent/toolApproval/*`
- `packages/agent-core-v2/src/features/plan/exitPlanModeReview.ts`
- `packages/agent-core-v2/src/session/agentLifecycle/agentLifecycleService.ts`（广播）
- `docs/en/configuration/config-files.md#permission`
