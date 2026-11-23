# 05 · Agent 循环：turn / step 状态机

> 目标：把"一次用户输入如何变成一次或多次模型调用 + 工具执行"讲到可以实现。核心是一个单线程 actor + 显式阶段机，不是递归函数。

---

## 1. 两层结构

| 层 | 包 | 职责 |
| --- | --- | --- |
| 通用 loop | `packages/core/src/loop` | turn/step、消息、工具批次、中断、通知 |
| Agent 包装 | `packages/agent-core-v2/src/agent/loop` | 投影、提醒、压缩、目标、权限/工具钩子 |

`core` 的 loop 不认识 session、权限、wire；它只认 `requester`、`executor`、`hook`。Agent 层通过钩子把策略注入。

---

## 2. 状态类型

```ts
type LoopPhase = "idle" | "running" | "closed";

interface LoopState {
  status: "idle" | "running" | "interrupted";
  held: boolean;
  resetting: boolean;
  inbox: Inbox;
  profile: LoopProfile;
  messages: Message[];
  nextTurnId: number;
  turn?: LiveTurn;
  calls: Map<string, ToolCall>;
  completionPending: boolean;
}

type TurnPhase =
  | "starting" | "appending" | "stepStarting" | "requesting"
  | "batching" | "acting" | "ending" | "finished";

interface LiveTurn {
  id: number;
  trigger: "normal" | "notify" | "continue";
  step: number;
  bypassMaxSteps?: boolean;
  phase: TurnPhase;
}

interface StepState {
  scope: AbortScope;
  steerRequested: boolean;
  toolCallStreaming: boolean;
  llm?: AssistantMessage;
  partialMessage?: AssistantMessage;
  calls: LiveTool[];
  settled: boolean;
  outcome?: StepOutcome;
}
```

`Inbox`：

```ts
interface Inbox {
  normal: UserMessage[];  // 普通用户输入
  notify: UserMessage[];  // 后台通知/系统触发
  remind: Message[];      // 系统提醒
}
```

---

## 3. Actor 模型

`LoopActor` 是一个单线程 mailbox：

- `post(message)` 入队；
- `kick()` 排空事件，然后一次处理一个 `ActorMessage`；
- mailbox 空时调度下一 turn。

好处：不需要锁；所有状态变更串行；工具并发在 actor 之外（见 §5）。

---

## 4. Turn/Step 精确顺序

```
submit/notify/sendNow/remind
  → turn.start
  → startTurn
  → turn.starting hook
  → announceTurn / turn.append
  → message.appending hook（每个 QueuedItem）
  → message.appended
  → step.start
  → startStep（turn.step++）
  → step.starting hook（可改消息/阻止/进入 requesting）
  → afterStepStarting（drain reminders、tools.settled）
  → request.start
  → runLoopRequest
  → llm.completed
  ├─ 有 tool calls → batching hook → acting
  └─ 无 tool calls → ending
  → acting:
      startActing（每个 call 建 LiveTool）
      executeTools（全部同时启动）
      respondToTool（每个 settle）
      allSettled → acting.complete
      completeActing（按 call 顺序追加 tool 消息）
  → step.ending
  → step.stopping / step.continuing hook
  → finalizeStep
      continue → afterStepFinished → 下一个 step.start
      done     → turn.finish
  → finishTurn
  → turn.ended
  → schedule 下一 turn
```

关键点：

- **tool 消息按调用顺序追加，不按完成顺序**。这是保证可重放/可比较的关键。
- `promptOnly` turn 不请求 LLM（用于只注入提醒）。
- 一个 step 只有在"没有 tool calls"或"显式 stop"时才结束。

---

## 5. 工具批次与并发

一个 assistant message 里的所有 tool call **同时启动**（`executeTools` 不 await 彼此）。并发受 `ToolScheduler` 控制：

```ts
class ToolScheduler {
  // 若 ToolAccesses.conflict(active.accesses, candidate.accesses) 则阻塞
  // 否则启动；active 完成后启动队列
}
```

访问冲突由工具声明的 `accesses` 决定（读/写/搜索哪些路径）。两个都写同一文件 → 串行；一个读一个写 → 也冲突。这是"并行安全"的基础。

lease 按 `turnId:step` 批次获取，detach/release 时释放。

---

## 6. 停止条件

| 条件 | 结果 |
| --- | --- |
| 无 tool calls | step done |
| 工具返回 `stopTurn` | step done，原因 `stop_turn` |
| `maxSteps` | step 被 hook 阻止，原因 `max_steps` |
| `promptOnly` | 不请求 LLM |
| finishReason `filtered` | turn failed `PROVIDER_FILTERED` |
| 上下文溢出 | 触发压缩并重试 |
| abort / steer / close | 中断 |
| hook 在任一阶段阻止 | 按 hook 结果 |

`maxSteps` 由内置 loop hook 在 `step.continuing` 检查：

```ts
if (maxSteps !== undefined && maxSteps !== 0 && event.step >= maxSteps) block("max_steps");
```

配置：`[loop_control].max_steps_per_turn`，未设置/0 = 无限。默认代码里是 200（用户配置可改）。

---

## 7. 重试

loop 本身没有重试计数器；重试在 LLM requester：

- 默认最多 10 次；
- 退避 `500ms * 2^n`，上限 32s，±25% jitter；
- `retryAfterMs` 优先；
- 仅对非确定性错误无限重试（`KIMI_CODE_INFINITE_RETRY`）；
- 确定性错误（syntax/abort/context_overflow）不无限重试。

Agent 层的 loop error handler 可以在 `step.stopping` 调 `context.retry()`，把 step 标记为 `continue reason='retry'`。

---

## 8. 中断与 steer

- `steer(promptIds?)`：把 inbox.normal 中匹配的条目移到 notify；只有在 `stepStarting|requesting` 且没有 tool-call 流时才 abort 当前请求。
- `sendNow(message|ids)`：入 notify + abort 请求。
- `Ctrl-S` 对应"立即注入"。
- abort：把未处理项放回 mailbox，abort turn scope，调用 `handler.stop`，等 `abortGraceMs`（默认 2500ms），然后 `abort.expired`。
- `continueAfterAbort` 允许下一 turn。

---

## 9. 队列

`take()` 优先 `normal[0]`，否则 `notify[0]`；`promptOnly` = 头条是 promptOnly 或剩余 notify 里有非 promptOnly。

- 后台任务完成 → 入 notify；
- 用户输入 → 入 normal；
- 提醒 → 入 remind，只在非 promptOnly turn 注入。

恢复时 `restorePending` 重新入队；被阻止的 turn 只恢复 notify，normal/remind 直接进上下文。

---

## 10. 后台任务通知

detach 的工具：

1. `handler.detach(ack)` → `tool.ended { detached: true }`；
2. slot 保留在 `tools`；
3. 真正完成时 `completeTool` 生成完成消息；
4. `drainDetachedCompletions` 记录 `tool.async_ended`，入 notify，设置 `completionHandoff`；
5. 当前 step 以 `done reason=notify` 结束，下一步包含完成消息；若 idle，则 `schedule()` 启动一个 notify turn。

完成消息格式：

```
[async tool completed|aborted|failed] <name> (tool_call_id=...)
```

带 attributes：`source: "async-tool"`, `toolCallId`。

重启后，`restoreTaskCompletions` 会重新投递未送达的终态任务，按 `(taskId,status,notificationId)` 去重。

---

## 11. Goal 与 loop 的交互

`features/goal/goalService.ts`：

- `onWillBeginStep` → 目标 turn 计数 +1；
- `onDidFinishStep` → 检查预算：
  - 预算用尽且还有一次宽限 → `reminder.notify`；
  - 否则 `ctx.stopTurn = true`；
- 如果本步看到了目标结果工具，且还有步数预算 → `loop.notify()`，让当前 step 以 `notify` 结束，下一 step 处理目标；
- `TurnEnded` 时，如果目标仍 active 且 turn 不是 blocked/cancelled/failed（max_steps 不算失败）→ 提交 `GOAL_CONTINUATION_PROMPT`，origin `system_trigger: goal_continuation`。

这解释了"目标模式为什么能跨 turn 继续"，而不需要外部调度器。

---

## 12. Agent 层投影

`turnProjection.ts` 把 core 的 step 结果映射成 Agent 语义：

- `stepStarting`：重置 retry/trace，记录 gatedSteps，跑 `onWillBeginStep`；
- `stepStopping`：跑 `onDidFinishStep`，处理 `stopTurn`，把 `filtered` 转成失败，分发给 error handlers；
- `finishTurn`：
  - aborted → `cancelled`
  - filtered → `failed PROVIDER_FILTERED`
  - max_steps → `failed createMaxStepsExceededError`
  - blocked/failed → `failed`
  - 其它 → `completed`，`truncated = lastStopReason === "truncated"`

---

## 13. 不变量

1. 一个 agent 同时只有一个 active turn；其余入队。
2. tool 消息顺序 = call 顺序。
3. 每个 tool call 必须有且只有一个 tool 结果（缺失时合成"中断"结果）。
4. step 的 outcome 只由 LLM 结果 + 工具结果 + hook 决定，不由 UI 决定。
5. abort 不能丢消息；未处理项回到 inbox。
6. 后台完成必须走 notify，不能直接改历史。
7. 任何 hook 都不能让 loop 进入非法阶段；被阻止要给出明确 outcome。
8. `maxSteps` 是"步"不是"turn"；`bypassMaxSteps` 只对 turn 第一步有效。

---

## 14. 重实现最小版

1. 实现 `Inbox` + actor mailbox。
2. 实现 turn/step 状态与阶段转移。
3. 实现 `requester.generate`（先同步非流式）。
4. 实现 tool batch（先串行）。
5. 加 `maxSteps`、abort、队列。
6. 再加 scheduler 并发、steer、后台通知。
7. 最后接压缩/目标/提醒。

测试重点：

- 一次 turn 两步（工具后再回答）顺序正确；
- 工具慢/快完成，消息顺序仍按 call；
- abort 后继续下一 turn，历史不丢；
- 后台任务完成后从 notify 进入下一步；
- `maxSteps` 命中给出明确错误；
- 两个写同一文件的工具串行。

---

## 15. 证据索引

- `packages/core/src/loop/{create-loop,loop-actor,state,contract/*,tool/*,request,outcome}.ts`
- `packages/core/src/hook/loop/{builtin,registry,run}.ts`
- `packages/agent-core-v2/src/agent/loop/{loopService,turnProjection,projection,mediaPolicy}.ts`
- `packages/agent-core-v2/src/agent/toolExecutor/{toolScheduler,toolExecutorService}.ts`
- `packages/agent-core-v2/src/features/goal/goalService.ts`
- `packages/agent-core-v2/src/agent/task/{taskService,taskOutput,completionRestore}.ts`
