# 10 · 动态工具加载（select_tools）

> 目标：解释 Kimi Code 如何在不把上百个工具 schema 塞进每次请求的情况下，让模型按需加载工具。这是长会话与大量 MCP 场景的关键优化。

---

## 1. 问题

一个会话可能有：

- 30 个内置工具；
- 50+ 个 MCP 工具；
- 插件/skill 贡献的工具。

如果全部内联进每次请求：

- 系统提示膨胀，成本高；
- 模型注意力被无关工具分散；
- 小上下文模型直接爆。

Kimi 的解法：**工具披露（disclosure）分两级**：

- `inline`：每次请求都带完整 schema；
- `deferred`：只在"可加载公告"里出现名字，模型调用 `select_tools` 后才注入完整 schema。

---

## 2. 数据结构

`agent/toolSelect/`：

```ts
type ToolDisclosure = "inline" | "deferred";

interface ToolInfo {
  name: string;
  description: string;
  parameters?: Record<string, unknown>;
  source: "builtin" | "user" | "mcp";
  disclosure: ToolDisclosure;
  info?: Record<string, unknown>;
}
```

工具注册时可指定 `disclosure`；MCP 由配置 `deferred` 决定（全局 `[mcp].defaultDeferred` 或每服务器）。

---

## 3. 公告机制

系统上下文里会出现：

```
<tools_added>
  server_a__tool_1
  server_a__tool_2
</tools_added>

<tools_removed>
  server_b__old
</tools_removed>
```

规则：

- 公告是 **delta**：新增/移除；
- 首次是 snapshot；
- 模型必须"按顺序折叠"公告才能得到当前可加载列表；
- 只公告 `deferred` 工具；`inline` 工具直接可用，不需要 select。

公告以 system 消息注入，origin 带 `disclosure` 元数据（便于回放时重建）。

---

## 4. select_tools 工具

```ts
{ names: string[] }
```

`SelectToolsTool.resolveExecution`：

1. 调 `toolSelect.load(names)`，得到：
   - `toLoad`：真正要加载的；
   - `alreadyAvailable`：已内联；
   - `alreadyCallable`：已加载过；
   - `staleLoaded`：曾加载但已注销；
   - `policyDenied`：被工具策略禁用；
   - `unknown`：未知名字；
   - `escalated`：多次失败的名字；
   - `consecutiveFailures`；
   - `suggestions`；
   - `loadable`；
   - `schemas`。
2. 对每个成功加载的工具，构造一条 **system 消息**：

```ts
{ role: "system", content: [], tools: [toolSchema],
  origin: { kind: "injection", variant: "dynamic_tool_schema" } }
```

3. `ctx.remind?.(message)` 注入；
4. 返回文本结果（Loaded / Already available / 错误提示）。

**注意**：工具 schema 是以 system 消息注入的，不是修改请求的 tools 数组。这样历史可回放，且不依赖 provider 的动态 tools 能力。

---

## 5. 加载状态

`toolSelectOps.ts` 维护：

- 已加载集合；
- 每个名字的失败次数；
- 连续失败次数；
- 当前可加载列表（由公告折叠）。

状态是 Agent scope 的 replayable state，因此恢复后仍然知道"哪些工具已加载"。

---

## 6. 失败提示（防呆）

`select_tools` 的返回值会明确区分：

| 情况 | 提示 |
| --- | --- |
| 已内联 | "already available — call it directly" |
| 已加载过 | "already available" |
| 曾加载但注销 | "no longer registered; cannot restore" |
| 策略禁用 | "disabled by tool policy; cannot enable" |
| 未知 | "Unknown tool … Did you mean: X?" 或列出可加载项 |
| 多次失败 | "has already failed selection N times; stop retrying" |
| 连续失败 ≥3 | "stop guessing names" |

这比简单返回 `not found` 更能阻止模型循环。

---

## 7. 模型能力

`ModelCapability.dynamically_loaded_tools`：

- 为 true → 启用 select_tools 机制；
- 为 false → 所有工具强制 inline（或不提供 select_tools）。

本地配置里 K3/K2.8 声明了该能力；DeepSeek 模型没有。

---

## 8. 不支持动态工具的模型

`dynamicTools.ts` 的 `shapeUnsupportedDynamicToolContext`：

- 隐藏 `select_tools` 的 assistant 调用与 tool 结果；
- 把 `<tools_added>` 公告改写成普通 `system-reminder` 文本；
- 去掉 `dynamic_tool_schema` system 消息里的 `tools` 字段（保留文本）；
- 目的：让同一段历史可以在"支持/不支持动态工具"的模型之间切换而不 400。

---

## 9. 与工具策略的关系

- 动态加载**不能绕过** `toolPolicy`：`policyDenied` 的名字不会被加载。
- `select_tools` 本身是一个 inline 工具（永远可见）。
- profile 的 `tools` 列表里如果有 `mcp__*`，表示允许该 profile 使用 MCP；具体工具仍可 deferred。

---

## 10. 不变量

1. 只有公告过的名字可加载。
2. 已加载的工具不能重复注入 schema。
3. 注销的工具不能通过 select 恢复。
4. 失败提示必须可操作（建议/停止），不能只报错。
5. 动态 schema 必须进历史（system 消息），否则下一步模型看不到。
6. 公告折叠顺序必须稳定；回放时不能丢。
7. 不支持动态工具的模型必须能安全降级。

---

## 11. 重实现最小版

1. 给工具加 `disclosure` 字段。
2. 请求编码时过滤 `deferred` 工具。
3. 维护"可加载集合"，生成 `<tools_added>/<tools_removed>`。
4. 实现 `select_tools`：校验名字 → 注入 schema 消息 → 返回结果。
5. 加失败计数与建议。
6. 加历史整形（降级）。
7. 最后做 MCP 的 deferred 配置。

测试重点：

- deferred 工具不出现在初始 tools；
- select 后下一步可调用；
- 未知名字给出建议；
- 已加载不重复注入；
- 策略禁用不能加载；
- 恢复后已加载状态仍在；
- 不支持动态工具的模型历史可安全投影。

---

## 12. 证据索引

- `packages/agent-core-v2/src/agent/toolSelect/{toolSelect,toolSelectService,toolSelectOps,toolSelectAnnouncements,toolSelectAnnouncementsService,dynamicTools,flag}.ts`
- `packages/agent-core-v2/src/agent/tools/select-tools/*`
- `packages/agent-core-v2/src/tool/toolContract.ts`（`ToolDisclosure`）
- `packages/agent-core-v2/src/agent/loop/llmRequestOps.ts`（请求侧过滤）
- 真实证据：会话 `tool_select.initialized` 88 个可加载工具
