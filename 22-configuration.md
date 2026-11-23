# 22 · 配置系统

> 目标：解释 Kimi Code 的配置分层、schema 注册、环境变量绑定、TOML 读写、诊断与热更新。重实现时，配置是最容易"看起来简单、实际处处是坑"的一层。

---

## 1. 两套栈（不要混）

| 栈 | 位置 | 用途 |
| --- | --- | --- |
| v2 `ConfigService` | `agent-core-v2/src/app/config/` | KAP/ACP/桌面/SDK；按 domain 注册、可变、热更新 |
| v1 `KimiConfig` | `node-sdk/src/config/` | `KimiHarness.getConfig()`、TUI/VS Code 的 provider 写入；整文件解析 |

重实现时只需要一套；建议采用 v2 的"domain + zod + 分层覆盖"。

---

## 2. 有效值优先级（低 → 高）

1. 注册的 `defaultValue`
2. 用户 `config.toml`
3. 该 section 的环境变量绑定
4. 有效 overrides（只有两个）：
   - `kimiModelEnvOverlay`：`KIMI_MODEL_*` 合成 provider/model
   - `servicesCredentialEnvOverlay`：web search/fetch 凭据
5. 内存/session override（`ConfigTarget.Memory`，不持久化）

注意：

- **CLI flag 不是配置层**。`--model`、`--plan`、`--yolo`、`--auto`、`--skills-dir`、`--agent`、`--add-dir` 在会话创建时作为 session state，不写 `config.toml`。
- 持久化写入会**剥掉环境变量绑定的字段**，避免把 env 值写进文件。
- 项目 `.kimi-code/local.toml` **不是** ConfigService 的一层；它由 `FileProjectLocalConfigService` 单独读取（只含 `workspace.additional_dir` 和原始 `[permission].rules`）。

---

## 3. Section 注册

```ts
registerConfigSection("permission", PermissionConfigSchema, {
  fromToml, toToml, env: permissionEnvBindings, stripEnv: stripPermissionEnv,
});
```

- `ConfigRegistry` 用 zod `{ parse }`；
- 默认合并：递归 `deepMerge`；
- 默认 scope：`ConfigScope.Core`；
- `Session`/`Project` scope 存在但当前没有 section 使用。

---

## 4. 关键 section 与字段

| TOML | 关键字段 | 备注 |
| --- | --- | --- |
| `default_model` | string | v2 未注册 schema，原样传递；`KIMI_MODEL_NAME` 可覆盖 |
| `default_permission_mode` | `manual|auto|yolo` | 默认 `manual` |
| `default_plan_mode` | bool | |
| `loop_control` | `max_steps_per_turn`、`max_attempts_per_step`、`max_ralph_iterations`、`reserved_context_size`、`compaction_trigger_ratio`(0.5–0.99)、`compaction_max_attempts` | 环境变量 `KIMI_LOOP_*` |
| `background` / `task` | `bash_auto_background_on_timeout`、`bash_task_timeout_s`、`kill_grace_period_ms`、`print_*` | `task` 优先于 `background` |
| `tools` | `enabled`、`disabled` | 工具可见性 |
| `permission` | `rules[]`、`dangerous_command_guard` | 见 14 篇 |
| `hooks` | `[[hooks]]` 数组 | 严格 schema |
| `environments` | `default` + 每 id 的 ssh/docker/command | 见 20 篇 |
| `subagent` | `timeout_ms`（默认 2h） | `KIMI_SUBAGENT_TIMEOUT_MS` |
| `secondary_model` | `default_model`、`models`、`default_effort` | 子代理模型池 |
| `swarm` | `timeout_ms` | |
| `read` | `default_max_chars`、`max_chars` | |
| `experimental` | `record<string, boolean>` | 实验 flag |
| `workspace` | `require_trust`（默认 true） | |
| `providers` | id → baseUrl/type/apiKey/oauth/... | Kimi/自定义 |
| `models` | alias → provider/model/context/capabilities/... | |
| `services` | `moonshot_search`、`moonshot_fetch` | |
| `thinking` | `enabled`、`effort`、`forced_effort`、`keep` | |
| `token_counting` | `strategy`（`measured+estimated|measured|estimated`） | |
| `mcp` | 超时、deferred、重连 | |
| `identity` | `name`、`slug` | |
| `hosts` | SSH 主机 | |
| `extra_skill_dirs` | string[] | |
| `merge_all_available_skills` | bool | |
| `builtin_product_skills` | bool | |
| `extra_agent_dirs` | string[] | |

---

## 5. 环境变量绑定

```ts
type EnvBinding = string | {
  env: string;
  deprecatedEnv?: string;
  parse?: (raw: string) => unknown | undefined;
  default?: unknown;
};
```

规则：

- env 在文件之后应用，然后重新 zod 校验；
- `parse` 返回 `undefined` → 忽略；
- 非法 env → 警告，保留文件值；
- 弃用 env → 警告 "use Y"；
- 保存时 `stripEnv` 删除 env 派生字段。

示例：

```
KIMI_CODE_DANGEROUS_COMMAND_GUARD
KIMI_LOOP_MAX_STEPS_PER_TURN
KIMI_SUBAGENT_TIMEOUT_MS
KIMI_CODE_AGENT_SWARM_MAX_CONCURRENCY
KIMI_MCP_STARTUP_TIMEOUT_MS
KIMI_TOKEN_COUNTING_STRATEGY
```

---

## 6. TOML 读写

默认 `fromToml`：顶层 key 的 snake→camel，嵌套不转。
默认 `toToml`：克隆原始子树，camel→snake，保留未知 key/顺序/注释。

自定义 transform：

| section | 特殊处理 |
| --- | --- |
| `permission` | 旧 `allow/deny/ask` 数组 → 统一 `rules`；写回时删除旧字段 |
| `providers` / `services` | 嵌套 `oauth/env/customHeaders`；undefined 删除 |
| `models` | 嵌套 `overrides/upgrade`；capabilities 数组保留 |
| `hooks` | 数组表 |
| `experimental` | 标识映射 |
| `hosts` | `identity_file`/`remote_bin` |

写入：

1. 重新读磁盘状态；
2. 只 rebase 目标 domain；
3. 保留未知键；
4. 剥 env 字段；
5. `planConfigWriteback` 做行级编辑，保留注释/EOL；
6. 重新解析并验证期望值；
7. 失败 → 全量重序列化；
8. 原子写：tmp + fsync + rename。

---

## 7. 校验、诊断、热更新

- 每个 section zod 校验；无效 section 忽略并保留上一值；
- 诊断：`{ domain?, severity: "warning"|"error", message }`，去重，发 `onDidChangeDiagnostics`；
- 文件不可读/解析失败：保留上一值；写入被阻止（`config.persist_blocked`）；
- 热更新：只 watch `~/.kimi-code/config.toml`，150ms debounce；
- `reload()` 串行化；内容未变则 no-op；
- 变更事件 `ConfigChanged { changedFields, config }`，KAP 10ms 合并后广播。

---

## 8. 项目本地配置

`FileProjectLocalConfigService`：

- 根 = 最近 `.git` 祖先，否则初始 workDir；
- 路径 = `<root>/.kimi-code/local.toml`；
- schema：`workspace.additional_dir: string[]`；
- 值相对项目根解析，`~` 展开，必须存在且是目录；
- `[permission].rules` 原样读取；
- 追加规则拒绝符号链接，temp+rename 原子写；
- 带 debounce 的文件监听；
- **只在本地环境且工作区已信任时加载**。

注意 key 冲突：`workspace` 在用户配置里是信任设置，在 local.toml 里是 `additional_dir`。

---

## 9. 配置相关的坑

1. `default_model` 在 v2 没有 schema，拼错不会被校验。
2. `background` 与 `task` 两个 section 都注册同一 schema，`task` 逐 key 覆盖。
3. CLI flag 不写配置；别指望从 `config.toml` 读到 `--model`。
4. env 值不会写回文件，但会覆盖文件值。
5. 项目 local.toml 不是 ConfigService 层，热更新/优先级都不同。
6. `models` 条目的 `protocol`/`name`/`max_context_size` 缺失会导致该模型不可用。
7. `providers` 的 `type` 决定协议；写错会路由到错误 requester。
8. 写入失败会阻止后续写，直到文件修好。
9. 文件权限：v2 写 `0600`；v1 某些路径按 umask。
10. `require_trust=false` 会让项目配置无确认加载。

---

## 10. 不变量

1. 任何配置值都必须经过 schema 校验（除非显式 passthrough）。
2. env 覆盖后必须重新校验。
3. 写入必须保留未知键与注释，且原子。
4. 解析失败不能清空用户配置。
5. 项目配置只在信任的本地工作区生效。
6. 内存 override 不能持久化。
7. 诊断必须可见，不能静默忽略。
8. 热更新不能重建整个进程。

---

## 11. 重实现最小版

1. 定义 section 注册 + zod。
2. 读 `config.toml`，顶层 snake→camel。
3. 实现 env 绑定 + 重新校验。
4. 实现默认值合并。
5. 实现原子写 + 保留未知键。
6. 加诊断。
7. 加 watch/reload。
8. 最后加项目 local.toml 与 session override。

测试重点：

- 默认 < 文件 < env < memory；
- 非法 section 保留上一值；
- 写入保留注释/未知键；
- env 不写回；
- 原子写（中断不损坏）；
- 项目规则只在信任时加载；
- 热更新不重启。

---

## 12. 证据索引

- `packages/agent-core-v2/src/app/config/{config,configService,toml,tomlWriteback,deprecations,sectionDiff,configEvents,configSectionContributions}.ts`
- 各 section：`agent/loop/configSection.ts`、`agent/permissionRules/configSection.ts`、`agent/task/configSection.ts`、`agent/toolPolicy/configSection.ts`、`features/externalHooks/configSection.ts`、`session/subagent/configSection.ts`、`session/tokenCounting/configSection.ts`、`app/kosongConfig/configSection.ts`、`app/auth/configSection.ts`、`workspace/workspaceTrust/configSection.ts`
- `packages/agent-core-v2/src/persistence/backends/node-fs/projectLocalConfigService.ts`
- `packages/node-sdk/src/config/*`
- `docs/en/configuration/*`
