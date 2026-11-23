# 18 · 子代理（Subagents）

> 目标：解释 `Agent` 工具如何创建、隔离、调度、恢复、超时和回收一个子代理。重实现时，关键是"独立上下文 + 受控工具表 + 继承权限 + 结果只回传最终消息"。

---

## 1. 工具表面

```ts
{
  prompt: string;                 // 必填，完整任务
  description: string;            // 必填，3-5 词
  subagent_type?: string;         // 默认 coder
  resume?: string;                // agent id；与 subagent_type/model 互斥
  run_in_background?: boolean;
  model?: string;                 // secondary_model 别名
  fork?: boolean;                 // 实验，默认 false
}
```

输出：

```ts
{ result: string; usage: { input; output; cache_read?; cache_write? } }
```

工具注册：

```ts
{
  name: "Agent",
  domain: "subagent",
  requiredEnvironmentCapabilities: ["process"],
}
```

没有 `process` 能力的环境（某些远程）不提供该工具。

---

## 2. 内置 profile

`session/agentLifecycle/profile/profiles.ts`：

| profile | 工具 | 说明 |
| --- | --- | --- |
| `agent`（默认主代理） | 全套 + `Agent`/`AgentSwarm` | 主代理 |
| `coder` | 读写/执行/搜索/任务/技能/MCP | 唯一能改文件的子代理 |
| `explore` | 只读 + Bash + Web | 代码库探索；提示层只读 |
| `plan` | 只读（无 Bash） | 实现规划 |

`coder` 工具集：

```
NotifyUser, Bash, Cron*, Edit, Glob, Grep, Read, ReadMediaFile,
Skill, TaskList, TaskOutput, TaskStop, TodoList, WaitFor,
WebSearch, FetchURL, Write, mcp__*
```

`explore` 没有 `Write/Edit`；`plan` 连 `Bash` 都没有。

---

## 3. 允许哪些子代理

`subagentService.planSpawn`：

- 调用者必须存在；
- 允许列表 = `caller.subagents ?? catalog.getDefault().subagents`；
- `["*"]` → 无限制；
- 非根调用者会被 `withoutDelegatingTargets` 过滤：去掉"自己还能派发"的 profile；
- 默认 allowlist = `["coder","explore","plan"]`，且这三个都不能再派发。

因此默认**递归深度为 1**；要更深必须显式声明 `subagents`。

失败：

- `AGENT_TYPE_NOT_ALLOWED`
- `PROFILE_UNKNOWN`
- `MODEL_NOT_CONFIGURED`

---

## 4. 创建流程

```
resolveExecution
  → launch
    ├─ resume? → resolveResumeTarget → run
    └─ planSpawn → spawn → run
```

`spawn`：

1. 非 fork：`environment.acquireWhenReady(["process"])`；
2. `agentLifecycle.create({ binding:{profile,model,thinking}, labels, environmentId: caller.environmentId, environmentCwd: caller.cwd })`；
   - **继承父环境绑定**；
3. fork：`agentLifecycle.fork` + `FORK_CONTEXT_NOTICE`；
4. `permissionMode.setMode(caller.mode)`；
5. 继承 user tools；
6. 应用 profile `promptPrefix`（如 explore 的 git 上下文）；
7. `run(ctx, {kind:"prompt", prompt}, {signal})`；
8. `mirrorAgentRun` 把子代理事件投影到父会话（UI 可见）。

---

## 5. 上下文隔离

- 新 agent = 全新 scope/history；
- 只传 prompt（+ profile promptPrefix）；
- fork 复制的是**已完成历史的快照**，并明确标注"参考材料，不是连续性"；
- 子代理的中间推理/tool 调用不回流；
- 父代理只看到最终消息（+ usage）。

子代理 turn 注入 origin `{kind:"system_trigger", name:"subagent"}`。

---

## 6. 工具表

profile 的 `tools` / `disallowedTools` 决定可见工具；`resolveActiveToolNames` 组合四层策略。

动态公告：

- 只有当某个子代理模型支持图片/视频时，才公告 `ReadMediaFile`；
- 只有当池模型支持动态工具时，才公告 `select_tools`。

---

## 7. 权限继承

- 模式：`setMode(caller.mode)`；
- 主代理切换模式 → 广播到所有未固定模式的 agent；
- 规则（尤其 session allow）对子代理生效；
- Tower 的 `tower-worker` profile 固定模式，不会被覆盖。

---

## 8. 恢复（resume）

- 按 **agent id**，不是 task id；
- 校验：
  - meta 存在；
  - 是子代理；
  - `parentAgentId === caller`；
  - 不是 running；
- 冷恢复：`createAgentAwaitingClose` 轮询 50ms，直到 `AGENT_ALREADY_EXISTS` 清除或超时（默认 15s，`KIMI_CODE_SUBAGENT_SCOPE_EVICT_TIMEOUT_MS`）；
- 历史在重启后仍在；**进行中的 tool 结果不在**；
- `resume` 不能同时传 `subagent_type` / `model`。

---

## 9. 超时与部分输出

- `[subagent].timeout_ms` 默认 7,200,000（2h）；`KIMI_SUBAGENT_TIMEOUT_MS`；0 = 无超时；
- 超时：abort → `kill_grace_period_ms`（默认 5s）→ 写 partial → 状态 `timed_out`；
- 部分输出：保留最近 32 KiB code points，前缀 `[earlier text truncated]`；
- 返回时父代理看到 `[partial output] ...` 或 `[partial output] none ...`。

---

## 10. 后台与通知

- 后台仅当 `TaskList/TaskOutput/TaskStop` 都可用；
- `detach()` 返回 `task_id`、`agent_id`、`automatic_notification: true`、`resume_hint`；
- 完成：
  - `TaskTerminated` → 状态 + notice；
  - loop 用 `completionMessage` 生成
    `<notification id="task:<id>:<status>" category="task" type="task.<status>" source_kind="background_task" ...>`；
  - 入 notify，下一步处理；
- 重启后 `restoreTaskCompletions` 重投未送达终态，按 `(taskId,status,notificationId)` 去重；
- 子代理结束 turn 时若还有后台任务，通知无人接收；会提醒它 `WaitFor`。

---

## 11. 模型池

`[secondary_model]`：

```toml
[secondary_model]
default_model = "kimi-code/k3"
default_effort = "high"
[secondary_model.models]
fast = "kimi-code/kimi-for-coding-highspeed"
deep = "kimi-code/k3"
```

规则：

- 无池 → 继承调用者模型；显式 `model` 被拒；
- 有池 → `model` 必须是池里的 alias；
- `modelSource = "inherited" | "secondary_pool"`；
- thinking = 显式 / 配置 / 模型默认；
- 校验 defaultModel、alias、effort。

---

## 12. 并发

- 没有全局子代理上限；
- 同一 agent 不能并发运行（`AGENT_ALREADY_RUNNING`）；
- 并行来自：多个 tool call、后台、Swarm。

---

## 13. 结果分类

停止原因：

```
completed | repeat_breaker | max_tokens | max_steps | filtered
| provider_error | no_final_message | cancelled | stopped
| timed_out | error
```

- 等 `loop.settled()` + 所有子任务静默；
- 摘要 = 最后一条 assistant 文本；
- 空 → `AGENT_NO_FINAL_MESSAGE`；
- 截断 → `AGENT_MAX_TOKENS_EXCEEDED`；
- max-steps / rate-limit 错误码会重映射。

---

## 14. 不变量

1. 子代理有独立上下文，不能读父历史。
2. 默认不能无限递归派发。
3. 子代理不能比父代理权限更宽松。
4. resume 必须校验 parent 与 running 状态。
5. 超时必须先 abort 再杀，保留 partial。
6. 后台完成必须通过 notify，不能直接改父历史。
7. 子代理最终消息是唯一回传内容（+ usage）。
8. 环境绑定继承，不能偷偷切换。

---

## 15. 重实现最小版

1. 定义 profile（工具表 + 系统提示）。
2. 实现 `Agent` 工具：创建新 agent scope → 跑 loop → 取最后消息。
3. 加 allowlist 与递归限制。
4. 加权限模式继承。
5. 加超时与 partial。
6. 加后台 + 通知。
7. 加 resume。
8. 最后加模型池与 fork。

测试重点：

- 子代理看不到父历史；
- `explore` 不能 Write；
- 默认不能派发子代理的子代理；
- 超时返回 partial；
- resume 续上历史；
- 后台完成通知进入父 turn；
- 权限模式继承；
- 同一 agent 并发被拒。

---

## 16. 证据索引

- `packages/agent-core-v2/src/agent/tools/agent/*`
- `packages/agent-core-v2/src/session/subagent/*`
- `packages/agent-core-v2/src/session/agentLifecycle/profile/{profiles,explore-overlay}.ts`
- `packages/agent-core-v2/src/agent/task/*`
- `packages/agent-core-v2/src/app/agentProfileCatalog/profile-shared.ts`
- `packages/agent-core-v2/src/agent/tools/agent/agent.md`（工具描述原文）
