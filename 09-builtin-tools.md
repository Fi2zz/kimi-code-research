# 09 · 内置工具规格

> 目标：逐个说明 Kimi Code 内置工具的输入、默认值、行为、权限与坑。重实现时可以把它当验收清单。  
> 说明：参数名以源码/文档为准；"自动"= 默认审批策略直接放行；"需审批"= 落到 fallback ask（manual）或 yolo 放行。

---

## 0. 总表

| 类别 | 工具 | 默认审批 |
| --- | --- | --- |
| 文件 | `Read` `Write` `Edit` `Grep` `Glob` `ReadMediaFile` | 读/搜自动；写/改需审批（git 工作区内例外） |
| Shell | `Bash` | 需审批 |
| Web | `WebSearch` `FetchURL` | 自动 |
| 计划 | `ExitPlanMode` | 自动放行 + 强制人工审阅 |
| 状态 | `TodoList` | 自动 |
| 协作 | `Agent` `AgentSwarm` `AskUserQuestion` `NotifyUser` `Skill` `select_tools` | 自动 |
| 后台 | `TaskList` `TaskOutput` `TaskStop` `WaitFor` | 除 `TaskStop` 外自动 |
| 定时 | `CronCreate` `CronList` `CronDelete` | 创建/删除需审批 |
| 目标 | `CreateGoal` `GetGoal` `SetGoalBudget` `UpdateGoal` | 自动 |
| Tower | 13 个 `Tower*` | 实验特性 |
| MCP | `mcp__<server>__<tool>` | 同策略链（默认 ask） |

---

## 1. Read

**用途**：读取文本文件，支持分页、列偏移、尾部读取。

**参数**：

```ts
{
  path: string;
  line_offset?: number;   // 1-based；负数从尾部数
  column_offset?: number; // 0-based，仅正向读
  n_lines?: number;       // 请求行数；省略则读到末尾
  max_chars?: number;     // 默认/上限 100,000
}
```

**行为**：

- 默认 100,000 字符；单次还受约 1/4 token 上限约束（默认 25,000 token）。
- 优先返回完整行；返回状态块 + `Next Read` 参数。
- 超长单行会分片返回，状态给出列范围。
- 尾部读取先返回最新的完整行；没有完整行时给正向 `Next Read`。
- 检测 UTF-16 LE/BE（≤10 MiB）；解码失败用 `�` 替换并警告。
- 文件在读取期间变化 → 报告，建议重读。
- 敏感文件硬阻断（`.env`、私钥等）。

**不变量**：`column_offset` 不能与负 `line_offset` 同用；行号不计入列偏移。

---

## 2. Write

**参数**：

```ts
{
  path: string;
  content: string;
  mode?: "overwrite" | "append"; // 默认 overwrite
}
```

**行为**：

- 自动创建父目录。
- `append` 不自动加换行。
- 写已存在文件（两种模式）**要求本会话先 Read 过该文件**。
- 文件自上次 Read 后变化 → 拒绝。
- 新建文件豁免"先读"。
- 敏感文件硬阻断。

**不变量**：写入前必须能证明"模型看过当前内容"，防止覆盖用户未提交的修改。

---

## 3. Edit

**参数**：

```ts
{
  path: string;
  old_string: string;
  new_string: string;
  replace_all?: boolean; // 默认 false
}
```

**行为**：

- 默认只替换唯一匹配；多处匹配 → 报错并建议 `replace_all: true`。
- `old_string === new_string` → 报错。
- 文件必须本会话 Read 过；变化 → 拒绝。
- 敏感文件硬阻断。

**不变量**：编辑是"精确字符串替换"，不是行号替换；避免并发写导致错位。

---

## 4. Grep

**参数**：

```ts
{
  pattern: string;
  path?: string;
  type?: string;          // ts, py, ...
  glob?: string;
  output_mode?: "files_with_matches" | "content" | "count_matches";
  "-A"?: number; "-B"?: number; "-C"?: number;
  "-i"?: boolean;
  "-n"?: boolean;         // 默认 true
  multiline?: boolean;
  offset?: number;
  head_limit?: number;    // 默认 250；0 = 不限
  include_ignored?: boolean;
}
```

**行为**：

- 调 ripgrep；`rg` 不在 PATH 时尝试 `~/.kimi-code/bin/rg` 或下载。
- 默认 `files_with_matches`。
- 支持上下文行、大小写、行号、多行。
- 分页 `offset + head_limit`。
- 自动过滤敏感文件（即使 `include_ignored`）。
- 超时/不可读目录/输出上限会警告，结果可能不完整。

**不变量**：搜索不修改文件；`include_ignored` 只影响 `.gitignore`，不影响敏感文件过滤。

---

## 5. Glob

**参数**：

```ts
{
  pattern: string;
  path?: string;          // 默认 cwd
  offset?: number;
  head_limit?: number;    // 默认 100；0 = 不限
  include_ignored?: boolean;
}
```

**行为**：

- 按 mtime 降序返回。
- 默认尊重 `.gitignore` / `.ignore` / `.rgignore`。
- 支持 brace（`*.{ts,tsx}`）。
- 分页；大页落盘。
- 每次调用重新扫描，分页间文件变化会移动结果。
- 敏感文件过滤。

---

## 6. ReadMediaFile

**参数**：

```ts
{
  path: string;
  region?: { x; y; width; height };
  full_resolution?: boolean;
}
```

**行为**：

- 读图片/视频，作为多模态内容发给模型。
- 文件上限 100 MB。
- 默认按模型限制压缩；压缩不达标 → 报错并建议生成小图，不发原图。
- 依赖模型 `image_in` / `video_in` 能力。

**不变量**：模型不支持时不应暴露该工具（或调用时拒绝）。

---

## 7. Bash

**参数**：

```ts
{
  command: string;
  cwd?: string;
  timeout?: number;              // 秒；前台默认 60，最大 300
  run_in_background?: boolean;
  description?: string;          // 后台必填
  disable_timeout?: boolean;     // 仅后台
}
```

**行为**：

- 前台阻塞当前 turn，流式输出到工具卡。
- 前台超时默认**不杀**，转后台（`bash_auto_background_on_timeout`）；后台默认 600s（`bash_task_timeout_s`，0 = 无超时）。
- 后台立即返回 task id；完成自动通知。
- stdin 永远关闭（交互命令立即 EOF）。
- 终止：SIGTERM → 5s 宽限 → SIGKILL。
- 子进程环境注入 `KIMI_CODE_SESSION_ID`、`KIMI_CODE_WORKSPACE_ID`。
- Windows 走 Git Bash。

**权限**：`approvalRule = Bash(<command>)`；危险命令分析见 15 篇。

---

## 8. WebSearch

**参数**：`{ query: string }`

**行为**：

- 需要 host 注入搜索实现；未注入则不出现。
- 始终在**本机**执行，即使会话绑定远程环境。

---

## 9. FetchURL

**参数**：`{ url: string }`

**行为**：

- HTML → 提取正文；纯文本/Markdown 直通。
- 需要 host 注入实现。
- 始终在本机执行。

---

## 10. ExitPlanMode

**参数**：

```ts
{
  options?: Array<{ label: string; description: string }>; // 1-3 个
}
```

**行为**：

- 只有 Plan 模式 active 时可调用。
- 读取计划文件 → 展示 `plan_review` → **无论权限模式都要求人工批准**。
- 可选方案最多 3 个；label ≤80 字符；不能使用保留词（Approve/Reject/Reject and Exit/Revise）。
- 批准后退出 Plan 模式，返回"Approved Plan"。
- 拒绝/修订：Plan 模式保持；`Reject and Exit` 才退出。

**权限**：`default-tool-approve` 放行，但 `ExitPlanModeReview` 直接请求审批，绕过模式。

---

## 11. TodoList

**参数**：

```ts
{ todos?: Array<{ title: string; status: "pending" | "in_progress" | "done" }> }
```

**行为**：

- 省略 `todos` → 查询当前列表。
- 传空数组 → 清空。
- 会话内状态。
- 压缩摘要会带上当前 TODO。

---

## 12. Agent（子代理）

**参数**：

```ts
{
  prompt: string;
  description: string;            // 3-5 词
  subagent_type?: string;         // 默认 coder
  resume?: string;                // agent id，与 subagent_type/model 互斥
  run_in_background?: boolean;
  model?: string;                 // secondary_model 别名
  fork?: boolean;                 // 实验，默认 false
}
```

**行为**：

- 新 agent 独立上下文；只回传最终消息。
- 子代理继承父的环境绑定与权限模式。
- 前台等待；后台返回 task id，完成自动通知。
- 可 resume 已有 agent（按 agent id，不是 task id）。
- 默认超时 2h（`[subagent].timeout_ms`，0 = 无）。
- 停止/超时/失败时，父代理仍收到部分输出（标记不完整）。
- 同一 agent 不能并发运行。

详见 18 篇。

---

## 13. AgentSwarm

**参数**：

```ts
{
  description: string;
  subagent_type?: string;         // 默认 coder
  prompt_template?: string;       // 必须包含 {{item}}
  items?: string[];               // 2..128
  resume_agent_ids?: Record<string, string>;
  model?: string;
  fork?: boolean;
}
```

**行为**：

- 只在 swarm 模式开启时可用（用户 `/swarm`）。
- 每个 item 替换模板，启动一个子代理。
- 最多 128 个；至少 2 个（除非 resume）。
- 同一响应中只能有这一个工具调用（swarm 模式 hook 强制）。
- 并发爬坡：先 5 个，然后每 700ms 一个；`KIMI_CODE_AGENT_SWARM_MAX_CONCURRENCY` 可设上限。
- 429 自适应降速；被限流的成员可 resume。
- 聚合报告 `<agent_swarm_result>`。
- 超时默认 2h。

详见 19 篇。

---

## 14. AskUserQuestion

**参数**：

```ts
{
  questions: Array<{
    question: string;              // 必须以 ? 结尾
    header: string;                // ≤12 字符
    options: Array<{ label; description }>; // 2-4
    multi_select?: boolean;
  }>;                              // 1-4 个
  background?: boolean;
}
```

**行为**：

- 自动追加 "Other"。
- 后台提问：立即返回 task id，问题在 turn 结束后仍可回答。
- host 不支持交互 → 返回失败，让模型改用文本提问。
- `auto` 模式下该工具被策略拒绝。

---

## 15. NotifyUser

**参数**：`{ title?: string; message: string }`

**行为**：

- 实验特性；需 `notify_user` flag。
- 只在成功显示后出现；失败/中断不展示。
- TUI 在输入框上方 Updates 面板；Web/桌面在对话中显示进度行。
- 不结束 turn。

---

## 16. Skill

**参数**：

```ts
{ skill: string; path?: string; args?: string }
```

**行为**：

- 只能调用 `type = inline/prompt` 的 Skill；`flow`/`reference` 不可由模型调用。
- `disableModelInvocation: true` → 拒绝。
- 嵌套深度上限 3。
- 内容以 `<skill-loaded>` 注入历史；返回值只是 "loaded" 提示。
- `path` 选择同名工作区变体。

详见 12 篇。

---

## 17. select_tools

**参数**：`{ names: string[] }`

**行为**：

- 只加载 `<tools_added>` 公告里的精确名字。
- 加载后完整 schema 以 system 消息注入，下一步可调用。
- 未知/已加载/被策略禁用/过期分别给出不同提示。
- 连续失败 3 次提示停止猜测。

详见 10 篇。

---

## 18. 后台任务工具

### TaskList

```ts
{ active_only?: boolean /* 默认 true */; limit?: number /* 20，1-100 */ }
```

### TaskOutput

```ts
{ task_id: string }
```

- 非阻塞快照。
- 内联最多最近 32 KB；完整日志落盘，返回 `output_path`。

### TaskStop

```ts
{ task_id: string; reason?: string }
```

- 对终态任务安全。
- 停止子代理会返回其部分输出与 agent id，便于 resume。
- 需审批。

### WaitFor

```ts
{ timeout: number /* 秒，最大 90 */; task_id?: string }
```

- 等任一后台任务完成或超时；超时不是错误。
- 新消息/steer 会提前结束等待。

---

## 19. 定时任务

### CronCreate

```ts
{ cron: string /* 5 段 */; prompt: string /* ≤8KB */; recurring?: boolean /* 默认 true */ }
```

- 返回 8 位 hex id、`humanSchedule`、`nextFireAt`。
- 抖动：循环任务前移 `min(10% 周期, 15min)`；整点/半点一次性任务前移 ≤90s。
- 睡醒后错过的多次只触发一次，带 `coalescedCount`。
- 存活 >7 天的循环任务最后触发一次（`stale=true`）后删除。
- 需审批。

### CronList

无参数，返回 `id/cron/humanSchedule/nextFireAt/recurring/ageDays/stale`。

### CronDelete

`{ id: string }`；不可逆；需审批；Plan 模式禁止。

---

## 20. 目标工具

| 工具 | 参数 | 说明 |
| --- | --- | --- |
| `CreateGoal` | `objective`, `token_budget?` | 创建目标 |
| `GetGoal` | 无 | 查询状态 |
| `SetGoalBudget` | `token_budget` | 设置/调整预算 |
| `UpdateGoal` | `status`, `note?` | active/paused/cancelled/complete/blocked |

目标跨 turn；预算只在实际工作时计时；完成后清理并总结。

---

## 21. Tower 工具（实验）

`TowerInit` `TowerPlan` `TowerSpawn` `TowerComplete` `TowerMerge` `TowerRebase` `TowerTeardown` `TowerSend` `TowerInbox` `TowerFinding` `TowerReview` `TowerMission` `TowerStatus`。

核心语义：

- 只有 main agent、且 Tower active 时可调用（部分工具任何角色可调）。
- mission 有 `build/survey/artifact` 三类。
- worker 在独立 git worktree 改代码；reviewer 只读审查。
- merge 有严格门禁（状态、依赖、review、tip、scope、base 干净）。
- 细节见 19 篇。

---

## 22. 重实现优先级

先做 8 个就能跑通编码闭环：

```
Read, Write, Edit, Grep, Glob, Bash, TodoList, Agent
```

然后按需加：

```
WebSearch, FetchURL, Task*, AskUserQuestion, Skill,
ExitPlanMode, Goal*, Cron*, select_tools
```

Tower 最后做；它依赖 worktree、mission 状态机、review 门禁，复杂度远高于普通子代理。

---

## 23. 证据索引

- `packages/agent-core-v2/src/agent/tools/**`
- `packages/agent-core-v2/src/features/{todo,goal,cron,plan,skill,swarm,tower}/**`
- `packages/agent-core-v2/src/agent/permissionPolicy/policies/default-tool-approve.ts`
- `docs/en/reference/tools.md`（官方逐工具说明）
