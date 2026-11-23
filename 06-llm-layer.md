# 06 · LLM 适配层

> 目标：解释 Kimi Code 如何把统一的 agent 请求翻译成 OpenAI / OpenAI Responses / Anthropic / Google GenAI 四种协议，并把流式响应还原成统一消息。重实现时这是"接任何模型"的关键。

---

## 1. 两层

| 层 | 位置 | 职责 |
| --- | --- | --- |
| wire core | `packages/core/src/llm` | 协议无关的请求/响应、编码管线、重试/恢复 |
| agent adapter | `packages/agent-core-v2/src/llm-adapter` | 模型目录、provider 定义、路由、能力合并 |

`core` 不知道 "Kimi" 或 "OpenAI"；它只知道 `ProtocolName` 和 `Protocol` 接口。

---

## 2. 协议抽象

```ts
type ProtocolName = "openai" | "openai_responses" | "anthropic" | "google-genai";

interface LlmModule {
  model: LlmModelSpec;
  profile: LlmRequestProfile;
  content: LlmRequestContent;
  options?: Record<string, unknown>;
}

interface DecodeSink {
  onDelta(part: StreamedMessagePart): void;
  onFinish(reason: FinishReason): void;
  onMessageId(id: string): void;
  onUsage(usage: UsagePatch): void;
  onError(error: unknown): void;
  onToolCallEnd(index: number): void;
}

interface Protocol {
  media: MediaPolicy;
  toolCallId?: ToolCallIdPolicy;
  capability?: ProtocolCapability;
  encode(input): EncodedRequest;
  client(endpoint): Client;
  send(client, request, signal): Promise<Response>;
  sendStream(client, request, signal): AsyncIterable<Delta>;
  decoder(): Decoder;
  replay?: ReplayPolicy;
  requestId?(response): string | undefined;
  decodeError?(error): LlmError | undefined;
  diagnoseError?(error): Diagnostics | undefined;
}
```

`defineProtocol()` 会自动注入 `encode = runPipeline(...)`，`with()` 可以局部覆盖。

---

## 3. 编码管线

`protocol/pipeline.ts` 的顺序是固定的：

```
legalize → convert → merge-turns → annotate → finalize → flatten
```

| 阶段 | 作用 |
| --- | --- |
| legalize | 修正非法消息序列（孤儿 tool result、重复 ID、空结果） |
| convert | 统一消息/工具/参数：system、messages、tools（过滤 `deferred===true`）、cacheKey、thinking、responseFormat、maxOutputTokens、sampling |
| merge-turns | 按协议要求合并相邻 turn（如 Anthropic 的 user/assistant 交替） |
| annotate | 加 provider 特有标记（cache control、thinking signature、reasoning item） |
| finalize | 生成最终 JSON body |
| flatten | 展平为 provider 需要的形状 |

工具结果在 lower 之前会经过 `renderToolResultForModel`（统一错误/空输出/耗时前缀）。

---

## 4. 统一消息模型

`core/src/llm/message.ts`：

```ts
type FinishReason =
  | "completed" | "tool_calls" | "truncated" | "filtered" | "paused" | "other";

type ContentPart =
  | { type: "text"; text: string }
  | { type: "think"; think: string; encrypted?: string; detailsIndex?: number; hidden?: boolean; reasoningKey?: string }
  | { type: "image_url"; image_url: { url: string } }
  | { type: "audio_url"; ... }
  | { type: "video_url"; ... };

interface ToolCall {
  type: "function";
  id: string;
  name: string;
  arguments: string | null;
  extras?: unknown;
  rawId?: string;
  _streamIndex?: number;
}

interface Message {
  role: "system" | "user" | "assistant" | "tool";
  content: ContentPart[];
  toolCalls?: ToolCall[];
  toolCallId?: string;
  name?: string;
  // attributes: source, origin, partial, llmTiming, note, durationMs,
  // promptId, isError, stopTurn, detached, model, usage, headers, finish, messageId
}
```

流式合并用 `createMessageAccumulator()`：文本/think 增量拼接，tool call 按 `_streamIndex` 追加。

---

## 5. 工具调用 ID 的处理

不同 provider 的 ID 规则不同，必须归一化：

| Provider | 来源 | 处理 |
| --- | --- | --- |
| OpenAI | `tool_calls[].id`，可能缺失 | 缺失则 `randomUUID()`；sanitize，最大 64 |
| OpenAI Responses | `call_id` | 保留 `rawId` |
| Anthropic | `tool_use.id` | 原样 |
| Google | `functionCall` 无稳定 id | 合成 `name_rawId_<8hex>`，保留 `rawId` |

核心规则：

- `sanitizeToolCallId`：去非法字符、限长。
- `buildToolCallIdMap`：重复 ID 加 `_2`、`_3`。
- 流式 normalizer：按 `rawId + _streamIndex` 跟踪，冲突加 `__n`。
- `legalizeToolCallIds`：重写历史时保持一致。
- `verifyToolExchanges`：拒绝重复调用、孤儿/重复结果、空结果、未回答调用。
- `legalizeStructure`：清洗后把结果配到调用旁边。

**这是最容易出 bug 的地方**：历史里的 tool call ID 必须与结果严格配对，否则 provider 直接 400。

---

## 6. 流式与结算

事件：

```ts
type LlmDeltaEvent =
  | { type: "response_header"; ... }
  | { type: "request_id"; id: string }
  | { type: "timing"; ... }
  | { type: "part"; part: StreamedMessagePart; block?: number }
  | { type: "usage"; usage: UsagePatch }
  | { type: "finish"; reason: FinishReason }
  | { type: "tool_call_end"; index: number }
  | { type: "message_id"; id: string };

type LlmOutcome =
  | { type: "done"; message; usage; headers; finish; messageId; requestId; timing }
  | { type: "failed"; error; rawError?; diagnostics?; partial? }
  | { type: "aborted"; message: Message | null };
```

`settleLlmRequest`：

1. 累积 parts；
2. 归一化 tool call ID；
3. abort/failure 映射；
4. salvage 可用输出（部分文本 + 已完成 tool call）；
5. 构造 `AssistantMessage`，`source` 为 `llm` 或 `salvaged`；
6. 空响应/只有 think/重复响应 → `emptyResponseError`（可重试，`filtered` 除外）；
7. 流结束但没有 finish 事件 → provider error（可重试）。

重实现时务必实现"部分输出 salvage"，否则网络抖动会丢掉已生成的内容。

---

## 7. Thinking / reasoning

统一模型：

```ts
interface ModelThinkingMetadata {
  supportEfforts?: string[];
  defaultEffort?: string;
  offEffort?: string;
  alwaysThinking?: boolean;
  adaptiveThinking?: boolean;
}

type ThinkingEffort = "off" | "on" | string;
```

`resolveThinkingEffort`：

- `on` → 静默使用默认；
- `off` → 若声明 `offEffort` 用之；若 `alwaysThinking` 或不支持则报错；
- 其它值必须在 `supportEfforts` 内，否则报错；
- 模型 `capability.thinking === false` → 报错。

各协议：

| Provider | 表达 |
| --- | --- |
| OpenAI | `reasoning_effort`；历史有 think 但当前 off 时用 `medium` |
| Anthropic | `thinking: { type: "enabled", budget_tokens }`（low=1024, medium=4096, high=32000）；自适应模式 `{type:"adaptive"}` + `output_config.effort`；`context_management` 清 thinking |
| Google | `thought: true` part + `thoughtSignature`，文本带 `thought_signature_b64` |
| Responses | reasoning item `summary_text` + `encrypted_content` |

---

## 8. Usage

```ts
interface TokenUsage {
  inputOther: number;
  output: number;
  inputCacheRead?: number;
  inputCacheCreation: number;
  raw?: unknown;
}
```

- `inputTotal` 只在 `inputCacheRead` 有定义时返回；否则 undefined。
- `grandTotal = inputTotal + output`。
- `mergeUsagePatch` 合并流式部分更新。

各 provider 的映射：

| Provider | input | cache |
| --- | --- | --- |
| OpenAI | `prompt_tokens - cached` | read = cached，creation = 0 |
| Anthropic | `input_tokens` | read = cache_read_input_tokens，creation = cache_creation_input_tokens |
| Google | `promptTokenCount - cachedContentTokenCount` | read = cached |
| Responses | `input_tokens - cached_tokens` | read = cached |

只有请求消息与上下文匹配时才把 provider usage 记为"测量值"（`requestObservation`）。

---

## 9. 重试与恢复

`runLlmRequest` → `runAttempts`：

1. 应用 recovery ledger；
2. 运行 resolvers（结构投影、媒体降级、动态工具选择）；
3. `onAttempt` + `llm.requested`；
4. `settleLlmRequest`；
5. done/aborted 返回；
6. 有 recovery 建议 → `llm.retry reason='recovery'`，**attempt 重置为 1**；
7. 否则 `shouldRetry` → `llm.retry reason='backoff'`，延迟 `retryAfterMs ?? backoff`，可中断 sleep；
8. 否则失败。

`retry.ts`：默认 10 次；无限重试仅对非确定性错误；`syntax|abort|context_overflow` 不无限。

---

## 10. Provider / endpoint 配置

```ts
interface LlmEndpoint {
  baseUrl?: string;
  headers?: Record<string, string>;
  credentials?: CredentialProvider;
  defaultBaseUrl?: string;
  clientFactory?: ...;
  fetch?: ...;
  files?: ...;
}
```

解析：

```
baseUrl = explicit ?? defaultBaseUrl
headers = endpoint.headers + credential.headers
```

环境变量：

| Provider | Key | Base URL |
| --- | --- | --- |
| OpenAI | `OPENAI_API_KEY` | `OPENAI_BASE_URL` |
| Anthropic | `ANTHROPIC_API_KEY` | `ANTHROPIC_BASE_URL` |
| Gemini | `GOOGLE_API_KEY` | `GOOGLE_GEMINI_BASE_URL` |
| Vertex | `VERTEXAI_API_KEY` | `GOOGLE_VERTEX_BASE_URL` |
| Bedrock | `AWS_BEARER_TOKEN_BEDROCK` | `AWS_BEDROCK_BASE_URL` |
| Kimi | `KIMI_API_KEY` | `KIMI_BASE_URL` |

模型配置（`models.<alias>`）关键字段：

```
providerId, baseUrl, apiKey, oauth, protocol, name, aliases, model,
maxContextSize, maxInputSize, maxOutputSize, capabilities,
displayName, reasoningKey, adaptiveThinking, betaApi,
supportEfforts, defaultEffort, offEffort, upgrade, overrides
```

凭证选择顺序：静态 key → env key → OAuth token → Bedrock token → 无。

---

## 11. 能力（capabilities）

```ts
interface ModelCapability {
  image_in?: boolean;
  video_in?: boolean;
  audio_in?: boolean;
  thinking?: boolean;
  tool_use?: boolean;
  max_context_tokens?: number;
  max_input_tokens?: number;
  dynamically_loaded_tools?: boolean;
}
```

合并规则：

- `image_in/video_in/audio_in`：声明 OR 协议探测；
- `thinking`：包含 `always_thinking`；
- `tool_use`：声明或探测非 false；
- `dynamically_loaded_tools`：OR。

这决定：
- 是否暴露 `ReadMediaFile`；
- 是否走 `select_tools` 动态工具；
- 是否允许 effort 参数。

---

## 12. 不变量

1. 历史中的 tool call / result 必须严格配对。
2. tool call ID 在会话内稳定；重写历史要一致。
3. 空响应不能静默成功；要么 salvage，要么失败重试。
4. `filtered` 不重试。
5. `maxOutputTokens` 不能超过窗口剩余；未知窗口时按请求值。
6. thinking 关闭时不能把 think part 原样发给不支持它的模型。
7. usage 只用于"测量"，不能替代本地估算（见 07）。
8. 任何 provider 特有字段放在 `extras`/`rawId`，不要污染统一模型。

---

## 13. 重实现最小版

1. 定义统一 Message / ContentPart / ToolCall / Usage / FinishReason。
2. 先实现 OpenAI chat 的 encode + 非流式 send + decode。
3. 加流式与 accumulator。
4. 加 tool call ID 归一化与结构校验。
5. 加 Anthropic / Google。
6. 加 thinking、usage、重试。
7. 最后加 recovery resolvers（媒体降级、结构修复、动态工具）。

测试重点：

- 四种协议对同一统一消息的编码快照；
- 流式文本/工具参数的增量拼接；
- 重复 tool call ID 的去重；
- 孤儿 tool result 被修掉；
- 空响应触发重试；
- thinking off 时历史 think 的处理；
- usage 映射（含 cache）。

---

## 14. 证据索引

- `packages/core/src/llm/{message,request,response,stream,thinking,usage,generate,endpoint,requester,context-usage}.ts`
- `packages/core/src/llm/protocol/{protocol,pipeline,rules/*}.ts`
- `packages/core/src/llm/builtin/protocol/{openai,openai-responses,anthropic,google-genai}/*`
- `packages/core/src/llm/policy/{policy,retry,settle}.ts`
- `packages/agent-core-v2/src/llm-adapter/{model,contract,protocolAdapterRegistry,requestObservation}/**`
