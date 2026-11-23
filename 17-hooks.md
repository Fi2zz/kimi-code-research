# 17 · 生命周期钩子（Hooks）

> 目标：解释 Kimi Code 如何让用户在 agent 生命周期的关键点运行本地脚本，以及"拦截/放行/失败"的精确语义。重实现时，重点是**信任边界**与**fail-open**。

---

## 1. 定位

Hooks 是"当 X 发生时，运行我的脚本"。它：

- 可以拦截工具调用（安全网）；
- 可以发通知、追加上下文、做审计；
- **不是**沙箱，也不是唯一安全屏障；
- 永远在**运行 Kimi 的机器**上执行（即使会话绑定远程环境）。

---

## 2. 事件清单

`features/externalHooks/internal/types.ts`：

```ts
const HOOK_EVENT_TYPES = [
  "PreToolUse",
  "PostToolUse",
  "PostToolUseFailure",
  "PermissionRequest",
  "PermissionResult",
  "UserPromptSubmit",
  "UserPromptQueued",
  "TurnStarted",
  "Stop",
  "StopFailure",
  "Interrupt",
  "SessionStart",
  "SessionEnd",
  "SessionHeartbeat",
  "SubagentStart",
  "SubagentStop",
  "TaskStarted",
  "PreCompact",
  "PostCompact",
  "Notification",
] as const;
```

| 事件 | 时机 |
| --- | --- |
| `PreToolUse` | 工具执行前，可 block |
| `PostToolUse` | 成功后 |
| `PostToolUseFailure` | 失败后 |
| `PermissionRequest` | 审批请求发出时 |
| `PermissionResult` | 审批结果 |
| `UserPromptSubmit` | 用户提交 |
| `UserPromptQueued` | 消息入队 |
| `TurnStarted` | turn 开始 |
| `Stop` | turn 正常结束 |
| `StopFailure` | turn 失败结束 |
| `Interrupt` | 中断 |
| `SessionStart` / `SessionEnd` | 会话生命周期 |
| `SessionHeartbeat` | 心跳 |
| `SubagentStart` / `SubagentStop` | 子代理 |
| `TaskStarted` | 后台任务开始 |
| `PreCompact` / `PostCompact` | 压缩前后 |
| `Notification` | 通知（如后台任务完成） |

---

## 3. 配置

只允许在**用户级 `config.toml`** 与**本地插件**中定义；没有项目级 hook 文件。

```toml
[[hooks]]
event = "PreToolUse"
matcher = "Bash"                 # 正则，可选
command = "my-guard.sh"
timeout = 30                     # 秒，1-600，默认 30
cwd = "/path"                    # 可选
env = { FOO = "bar" }            # 可选
```

Schema 严格：只允许这 5 个字段，多余字段会导致配置加载失败。

插件贡献的 hooks：`cwd = 插件根`，注入 `KIMI_CODE_HOME`、`KIMI_PLUGIN_ROOT`。

---

## 4. 输入

脚本通过 **stdin** 收到 JSON：

```json
{
  "hook_event_name": "PreToolUse",
  "session_id": "session_abc",
  "session_title": "Fix the login page",
  "client_type": "kimi_code_cli",
  "cwd": "/path/to/project"
}
```

不同事件追加字段，例如 `PreToolUse` 带工具名/参数，`Notification` 带任务状态。字段名统一 snake_case。

---

## 5. 返回语义

| 退出码 | 含义 | CLI 行为 |
| --- | --- | --- |
| `0` | 正常 | 继续；stdout 可作为上下文/提示 |
| `2` | 有意阻止 | 停止当前操作；stderr 作为原因 |
| 其它非 0 | 脚本错误 | **默认放行（fail-open）** |
| 超时 | — | 放行 |
| 启动失败 | — | 放行 |

结构化输出（stdout JSON）：

```json
{
  "message": "...",
  "hookSpecificOutput": {
    "permissionDecision": "allow" | "deny" | "ask",
    "permissionDecisionReason": "..."
  }
}
```

`action: "block"` + `reason/message` 也会被识别。

---

## 6. 执行细节

`internal/runHook.ts`：

- `shell: true` 启动；
- stdin 写入 JSON 后关闭；
- stdout/stderr 全量收集；
- 超时：非 Windows 独立进程组，先温和信号，再强杀；
- 退出码 2 → block；
- 退出码 0 + 结构化 block → block；
- 其它 → allow（即使 errored）。

多规则匹配：

- 同一事件的所有匹配 hook **并行**执行；
- 相同 `command` 去重，只跑一次；
- 任一 block → 阻止（具体合并策略以实现为准）。

---

## 7. 与权限的关系

- `PreToolUse` 在权限策略**之前**运行；
- 返回 block → 工具不执行，原因回给模型；
- 返回 allow **不能**绕过权限策略（它只是"不阻止"）；
- `PermissionRequest/Result` 是观察点，不改变决策。

因此：

```
hook block > permission deny > permission ask > permission approve
```

准确地说：hook 可以额外阻止；但不能额外放行一个会被权限拒绝的调用。

---

## 8. 信任边界

- **无项目级 hook**：checkout 下来的仓库不能注入 hook。
- **插件 hook**：安装插件时用户已确认信任。
- **fail-open**：脚本崩溃不会卡住 agent；但也意味着 hook 不能当唯一安全屏障。
- **本地执行**：远程会话中，hook 看到的是本地文件/进程，不是目标环境。
- **环境变量**：插件 hook 只注入声明的最小集合；用户 hook 继承当前进程环境。
- **超时**：默认 30s，最大 600s。

---

## 9. 典型用法

**拦截危险命令：**

```toml
[[hooks]]
event = "PreToolUse"
matcher = "Bash"
command = "grep -qE 'rm -rf /' && exit 2 || exit 0"
```

**桌面通知：**

```toml
[[hooks]]
event = "Notification"
matcher = "task\\.completed"
command = "terminal-notifier -title Kimi -message 'Task done'"
```

**每次提交追加 git 分支上下文：**

```toml
[[hooks]]
event = "UserPromptSubmit"
command = "printf 'Current branch: '; git branch --show-current"
```

---

## 10. 不变量

1. hook 不能把"会被拒绝的调用"变成允许。
2. hook 失败/超时不能阻塞 agent（fail-open）。
3. hook 命令只在本地执行，不在目标环境。
4. 项目文件不能定义 hook。
5. 相同 command 在同一事件只跑一次。
6. 退出码 2 的 stderr 必须作为可见原因。
7. hook 输出不能直接写入 wire（除非通过正常事件）。
8. 超时必须真正杀死进程组，不能留孤儿。

---

## 11. 重实现最小版

1. 定义事件枚举与配置 schema。
2. 实现 `runHook`：spawn + stdin JSON + 超时 + 退出码。
3. 在 `PreToolUse/PostToolUse` 接执行器。
4. 加 `Stop/SessionStart/Notification`。
5. 加结构化输出与并行/去重。
6. 加插件 hook 注入。
7. 最后做 `Permission*`/`Compact*`。

测试重点：

- 退出码 2 阻止；
- 非 0 放行；
- 超时放行且进程被杀；
- stdin JSON 字段完整；
- 多 hook 并行；
- 相同 command 去重；
- 项目配置里的 hooks 被忽略/报错。

---

## 12. 证据索引

- `packages/agent-core-v2/src/features/externalHooks/{configSection,externalHooksFeature}.ts`
- `packages/agent-core-v2/src/features/externalHooks/internal/{types,runHook,matchHooks,userPrompt}.ts`
- `packages/agent-core-v2/src/features/externalHooks/{agent,app,session}/*`
- `docs/en/customization/hooks.md`
