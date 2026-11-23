# 07 · 上下文、投影与压缩

> 目标：解释"历史消息如何变成发给模型的消息"以及"上下文快满时如何压缩"。重实现时，这一层决定长会话是否稳定、tool 配对是否合法、压缩后是否还能继续工作。

---

## 1. 上下文消息

`agent/contextMemory/types.ts`：

```ts
type PromptOrigin =
  | { kind: "user"; clientMetadata?; skillActivations?; attachments? }
  | { kind: "skill_activation"; trigger: "user-slash" | "model-tool" | "nested-skill"; ... }
  | { kind: "plugin_command"; pluginId; commandName; ... }
  | { kind: "injection"; variant; ownerPromptId?; disclosure? }
  | { kind: "shell_command" }
  | { kind: "compaction_summary" }
  | { kind: "system_trigger"; name }
  | { kind: "task" }
  | { kind: "cron_job" | "cron_missed" }
  | { kind: "hook_result" }
  | { kind: "retry" };

interface ContextMessage extends Message {
  id?: string;              // msg_<ulid>
  providerMessageId?: string;
  origin?: PromptOrigin;
  isError?: boolean;
  partial?: boolean;
  toolCallDisplays?: unknown;
  note?: string;
  usage?: TokenUsage;
  llmTiming?: unknown;
  durationMs?: number;
}
```

origin 不是装饰：它决定"这条消息在压缩时是否保留""是否算真实用户输入""是否能被重放"。

---

## 2. 系统提醒（reminders）

`features/reminder/reminderService.ts`：

- 在 `onWillBeginStep` 注入注册的 provider；
- 按 `REMINDER_VARIANT_PRIORITY` 排序（`date_change = -1` 最先）；
- 每个 turn 第一次 step 注入一次，压缩 splice 后再注入一次；
- provider 通过 `loop.remind(...)` 追加；
- 字符串内容包成 `<system-reminder>...</system-reminder>`；
- origin `{ kind: "injection", variant, ownerPromptId?, disclosure? }`。

提醒只在非 `promptOnly` turn 注入；turn 被 abort 时，待注入提醒会 flush。

---

## 3. 投影：历史 → 请求

`agent/loop/projection.ts`：

```ts
project(history, onAnomaly): Message[]
projectStrict(history): Message[]  // project + dedupe + merge + drop leading non-user
```

`pairBlocks` 是核心：

1. 跳过 `partial` 消息；
2. assistant 的 tool calls 登记为 open calls；
3. tool result 必须匹配 open call，否则作为孤儿丢弃；
4. 被其他消息隔开的结果会被重排到 call 之后；
5. 缺失/中断的结果合成 `TOOL_INTERRUPTED_TEXT`。

`flattenBlocks`：先发 call，再按 call 顺序发 results。

`cleanContent`：

- 丢弃空白 text，但保留带 `meta.thought_signature_b64` 的 part；
- tool 消息变空 → `REQUEST_INVALID`。

strict 额外规则：

- 重复 tool call ID 丢弃（保留第一个）；
- 重复 result 丢弃（除非替换合成的 interrupted 结果）；
- 连续 assistant 合并；
- 开头的非 user 消息丢弃。

这些规则保证任何历史都能变成 provider 接受的序列。

---

## 4. 工具结果渲染

`contextMemory/toolResultRender.ts`：

- 错误：前缀 `<system>ERROR: Tool execution failed.</system>`；
- 空输出：`<system>Tool output is empty.</system>`；
- 耗时：`Agent|AgentSwarm|Bash|FetchURL|Glob|Grep|WebSearch|MCP` 前面加 `Wall time: N seconds`；
- `note` 追加；
- 历史里的 `note/durationMs/isError` 可以存在 attributes，投影时再拼回去。

---

## 5. 媒体策略

模型不支持图片/视频时：

- 移除 `ReadMediaFile` 工具调用；
- 把媒体结果替换成用户提示（"当前模型不支持…"）；
- resolver 也可移除该工具；
- 恢复时用 `createMediaDegradeRecovery`。

core 的 legalization：

- 不支持的媒体 → 原因特定的占位符；
- `relocate` 模式的工具媒体 → 移到后续 user 消息。

降级恢复（`media-degrade.ts`）：

- 保留最新 2 个媒体；
- `request_too_large` → 把更旧的媒体全部降级；
- 已剥离/其它媒体错误 → 去掉所有目标媒体。

---

## 6. 压缩：策略参数

`agent/fullCompaction/strategy.ts`：

| 参数 | 默认 |
| --- | --- |
| `triggerRatio` | 0.85 |
| `blockRatio` | 0.85 |
| `reservedContextSize` | 50,000 |
| `maxCompactionPerTurn` | ∞ |
| `maxOverflowCompactionAttempts` | 3 |
| `maxRecentMessages` | 4 |
| `maxRecentUserMessages` | ∞ |
| `maxRecentSizeRatio` | 0.2 |
| `minOverflowReductionRatio` | 0.05 |

有效窗口：

```
maxSize = maxInputSize ?? maxContextSize
```

触发：

```
used >= maxSize * triggerRatio
OR (reserved > 0 && reserved < maxSize && used + reserved >= maxSize)
```

阻止（必须压缩后才能继续）：同上，用 `blockRatio`。

`checkAfterStep` 只在 `triggerRatio !== blockRatio` 时为 true；默认相等，所以只在 step 前检查。

---

## 7. 切割点合法性

`canSplitAfter` 不能在以下位置切：

- `user` 之后；
- 带 tool calls 的 assistant 之后；
- `tool` 消息之前；
- 打开的 tool exchange 内部。

这是压缩最容易出错的地方：切在 tool call/result 中间会产生非法请求。

---

## 8. 压缩计数

自动压缩（`countAuto`）：

- 从尾部往前累积消息；
- 统计 user 数和估算大小；
- 记住最后一个合法切割点；
- 当 recentMessages ≥ 4 或 recentUsers ≥ limit 或 recentSize ≥ maxSize*0.2 时停止（前提是有切割点）；
- 再用 `fitCompactCountToWindow` 从尾部收缩，直到压缩前缀 ≤ maxSize 且切割合法。

手动压缩：从尾部找最后一个合法切割点，再 fit。

溢出恢复：从尾部累积，找到"减少量 ≥ max(1, ceil(maxSize*0.05))"的最后一个合法切割点；否则用最佳切割或全长。

---

## 9. 压缩服务算法

`fullCompactionService.ts`：

1. `begin`：拒绝重复；手动压缩要求 quiescence；检查非空历史；派发 `FullCompactionBegin`；启动 worker。
2. `beforeStep` / `checkAutoCompaction`：如果自上次压缩没有进展则跳过；超限抛 `CONTEXT_OVERFLOW`（除非非抛出模式）。
3. `block`：标记 `blockedByTurn`，把 turn abort 传播给压缩，派发 `CompactionBlocked`，等待压缩 promise。
4. `compactionRound`：
   - flush reminders；
   - 快照 `originalHistory`，记录 `tokensBefore`；
   - `onWillCompact`；
   - 解析模型；
   - 压缩最大输出 = `maxOutputSize ?? min(maxContextSize, 128K)`；
   - 指令 = `renderCompactionInstruction`；
   - 尝试次数 = `compactionMaxAttempts ?? 5`；
   - 构建逐消息 token 画像（锚定到 assistant usage）；
   - 预收缩到窗口预算：
     ```
     outputReserve = min(maxOutput, max/8)
     inputLimit = max - outputReserve
     messageBudget = floor(inputLimit * 0.85) - overhead
     取最新消息，丢掉开头 tool result
     ```
5. 循环：
   - 追加指令 user 消息；
   - 估算请求 token；
   - `usedContextTokens = estimate + ceil(effectiveMax * 0.02)`；
   - 用 operation requester（`maxOutputSize`，首次探测 prompt cache，source `operation/full_compaction`）；
   - 只传 `{messages, usedTokens}`，空 profile。
6. 失败处理：
   - 上下文溢出 → 观测最大窗口 = `floor(estimated*0.85)`（若更低），按 `[0.7, 0.5, 0.35]` 收缩历史，重试；
   - 截断/空响应（非 filtered）→ 丢最旧消息 + 开头 tool result，重试；
   - 可重试错误 → 退避后重试；
   - 超过尝试/收缩上限 → 抛错；
   - 不可重试 → 抛错。
7. `collectSummary`：拒绝截断；拼接文本；空 → `APIEmptyResponseError`。
8. 安全检查：历史在快照后变化 → 取消（长度/前缀身份一致，新增只能是真实用户输入或 injection）。
9. `postProcessSummary` 追加当前 TODO；`captureWireLines` + `renderRecoveryFooter` 追加 `## Context Recovery` 指针。
10. 构造新上下文（`buildContextCompactionShape`），`replaceHistory(...,"compaction")` 必须被接受；派发 `ContextApplyCompaction` 与 `tokenCounting.rebase`；发 `CompactionCompleted`。

---

## 10. 压缩后保留什么

`compactionHandoff.ts`：

- 正常模式（非 legacyTail）：
  - 收集"可压缩的真实用户消息"（排除压缩摘要）；
  - 最多保留 20,000 token，其中头部预算 2,000、其余给尾部；
  - 边界消息可截断，并写"省略了多少 token"的说明；
  - 追加摘要消息、继续提示、`compactedCount` 之后的 injection；
  - 非用户/非真实用户消息全部丢弃。
- legacy 模式：一条摘要 + `history.slice(compactedCount)`。

摘要 prompt（`compaction-instruction.md`）要求：

- 最新请求；
- 绑定约束；
- 精确命令/路径/结果；
- 未知项；
- 前向计划；
- 只输出文本。

继续提示：

```
<system-reminder>Context compaction is complete — continue the work that was in progress when it began.</system-reminder>
```

---

## 11. Token 计数

估算器：

```
ASCII 字符 / 4 + 非 ASCII 字符
```

工具：name + description + JSON(parameters)。
消息：role + parts + assistant tool names/args JSON；用 `WeakMap<Message, number>` 缓存。
媒体：`MEDIA_TOKEN_ESTIMATE = 2000`。

锚定模型（`tokenCountingAgentModel`）：

- 状态 `{ anchors: {length, tokens, measured}[], tokens }`；
- provider 报 usage 时，若消息与上下文匹配，追加 measured anchor；
- `get(start,end)`：取 `length <= context.length` 的最新 anchor；前缀用 measured，后缀用估算；
- `statusSize(strategy)`：
  - `measured` → 最新 measured；
  - `estimated` → 全量估算；
  - `measured+estimated`（默认）→ max(get().size, latestMeasured)。

有效窗口：`maxInputSize ?? maxContextSize`。
观测窗口：`min(configured, observed)`（溢出时下调）。

请求大小 = system prompt + 非 deferred 工具 + 消息。

---

## 12. 不变量

1. 投影后不能有孤儿 tool result 或未回答的 tool call。
2. tool 消息顺序必须与 call 顺序一致。
3. 压缩切割点必须合法；否则请求 400。
4. 压缩摘要不能包含截断输出；空摘要视为失败。
5. 压缩期间历史变化 → 取消，不能覆盖用户新输入。
6. 压缩后必须保留继续工作所需的最小用户意图与 TODO。
7. 默认 `measured+estimated` 策略取两者最大值，因此状态栏采用的 token 数不能低于最近一次有效 measured；显式 `estimated` 策略本身不保证这一点。
8. 媒体降级只影响发送，不删除历史原文。

---

## 13. 重实现最小版

1. 定义 ContextMessage + origin。
2. 实现投影：配对、重排、合成中断结果。
3. 实现 reminder 注入。
4. 实现 token 估算（先不锚定）。
5. 实现手动压缩：找切割点 → 摘要请求 → 替换历史。
6. 加自动触发与阻止。
7. 加溢出收缩与锚定测量。
8. 最后做 handoff（保留用户意图）与恢复指针。

测试重点：

- 孤儿 tool result 被丢弃/修复；
- 连续 assistant 合并；
- 压缩切割不落在 tool 中间；
- 摘要为空 → 失败而不是写空；
- 压缩后仍能继续调用工具；
- 溢出时窗口下调并成功重试；
- token 估算与 provider usage 的偏差被锚定修正。

---

## 14. 证据索引

- `packages/agent-core-v2/src/agent/contextMemory/{types,contextEvents,contextTranscript,toolResultRender,compactionHandoff,messageId}.ts`
- `packages/agent-core-v2/src/agent/loop/{projection,projector,mediaPolicy,projectionPolicy}.ts`
- `packages/agent-core-v2/src/features/reminder/*`
- `packages/agent-core-v2/src/agent/fullCompaction/*`
- `packages/agent-core-v2/src/session/tokenCounting/*`
- `packages/core/src/llm/{context-usage,usage}.ts`
- `packages/core/src/llm/protocol/rules/{media,structure}.ts`
- `packages/agent-core-v2/src/llm-adapter/contract/tokens.ts`
