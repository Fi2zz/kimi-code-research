# 12 · Skills

> 目标：解释 Kimi Code 的 Skill 发现、解析、优先级、注入与调用。Skill 是"轻量能力包"，与 MCP（外部进程工具）和插件（安装单元）不同。

---

## 1. 两种形态

- 目录：`<name>/SKILL.md`（可带子文件、子 skill）；
- 单文件：`<name>.md`。

目录形式必须是：

```markdown
---
name: code-review
description: Review a diff for correctness
type: prompt
whenToUse: ...
disableModelInvocation: false
---

# Instructions
...
```

缺少开头 `---` → `SKILL_PARSE_FAILED`；目录形式必须有非空 `name` 和 `description`。
单文件形式可省略名字/描述：名字取文件名，描述取正文第一行（截断 239 字符 + `…`）。

---

## 2. Frontmatter 字段

| 字段 | 说明 |
| --- | --- |
| `name` | 目录形式必填；单文件可省 |
| `description` | 目录形式必填；用于模型选择 |
| `type` | `prompt`（默认）/`inline`/`flow`/`reference`；未知类型跳过 |
| `whenToUse` | 列表里显示"何时使用" |
| `disableModelInvocation` | true 时模型不能调用 |
| `isSubSkill` | 子 skill 标记 |
| `safe` | 声明但当前未消费 |
| `arguments` | 命名参数（`$name`） |
| 其它 | 透传保留 |

别名：

- `when-to-use` / `when_to_use` → `whenToUse`；
- `disable-model-invocation` / `disable_model_invocation` → `disableModelInvocation`。

**没有** `allowed-tools`：这是 Claude Code 的字段，Kimi 不支持。

---

## 3. 发现根与优先级

`catalog/skillRoots.ts`：

| 范围 | 路径 |
| --- | --- |
| User | `$KIMI_CODE_HOME/skills`，然后 `~/.agents/skills`（只取第一个存在的） |
| Project | `<projectRoot>/.kimi-code/skills`，然后 `<projectRoot>/.agents/skills` |
| Extra | `extra_skill_dirs`（`~`、绝对、相对项目根） |
| Builtin | 内置常量，可被 `builtinProductSkills` / 实验 flag 过滤 |
| Plugin | 插件的 skills 目录，来源标记 `extra` |

`projectRoot` = 最近的 `.git` 祖先。

显式 `--skill-dir`：非空时，默认的 user/workspace 来源**不贡献**任何 skill。

优先级（数字越大越优先）：

```
builtin 0 < plugin 5 < extra 10 < user 20 < workspace 30
```

文档表述：**Project > User > Extra > Built-in**。

同名不同路径的变体会保留（`registerVariants`）。

---

## 4. 扫描规则

- 最大深度 8；
- 跳过 `node_modules` 和点开头目录；
- 目录 `SKILL.md` 每层都发现；单文件 `*.md` 只在根层；
- 同名时目录优先于单文件；
- 子 skill：父 `SKILL.md` 有 `has-sub-skill: true`（或嵌套 metadata）才递归；子名字限定为 `parent.child`，`metadata.isSubSkill = true`；
- 未启用的父目录不下钻；
- 解析错误记录并跳过，不影响其它 skill。

---

## 5. 列表注入

`InMemorySkillCatalog.getModelSkillListing()`：

```
DISREGARD any earlier skill listings. Current available skills:

### Project
- code-review: Review a diff for correctness
  When to use: ...
  Workspace: ...

### User
...

### Extra
...

### Built-in
...

### Skill roots
- ...
```

- 只列出可调用（`disableModelInvocation !== true`、inline 类型）且非 sub-skill 的；
- 描述截断到 250 graphemes + `...`；
- 插件 skill 名为 `<plugin-id>:<name>`，归到 Extra；
- 每次 prompt 构建注入一次；listing hash 变化会发提醒重新列。

系统提示中的位置：`${skills_section}`（`system.md`），并附"Project overrides User overrides Extra overrides Built-in"。

---

## 6. 调用

`Skill` 工具：

```ts
{ skill: string; path?: string; args?: string }
```

解析顺序：

1. 有父 workspace → 先同 workspace 名字，再全局；
2. 不存在 → 错误；
3. `disableModelInvocation: true` → 错误；
4. `flow`/`reference` → "cannot be invoked by the model"；
5. `type` 非 inline/prompt → 错误。

触发来源：

- 模型调用：`model-tool`；
- 嵌套：`nested-skill`；
- 用户 slash：`user-slash`。

深度：`MAX_SKILL_QUERY_DEPTH = 3`；`>= 3` → `SKILL_NESTED_TOO_DEEP`。

注入：

- 记录 `SkillActivated` 事件；
- `loop.remind()` 一条 `user` 消息；
- 内容包 `<skill-loaded name=… trigger=… source=… dir=… args=…>`；
- 工具返回值只是 `Skill "<name>" loaded inline. Follow its instructions.`；
- 内容在进入上下文前会经过工具结果截断。

---

## 7. 参数插值

`catalog/registry.ts`：

- 按空白分词，支持单/双引号分组（引号去掉）；
- 命名参数：`arguments: [a,b]` → `$a`、`$b`（后面不能跟 `[` 或单词字符）；
- 位置参数：`$ARGUMENTS[n]`、`$n`（0-based）；
- `$ARGUMENTS` = 原始参数字符串；
- 值做 XML 转义；
- `${KIMI_SKILL_DIR}` → skill 目录；
- `${KIMI_SESSION_ID}` → 会话 id；
- 如果没有任何占位符被替换且有参数 → 末尾追加 `\n\nARGUMENTS: <raw>`。

---

## 8. Slash 命令

TUI：

- 内置或 sub-skill → `/<name>`；
- 其它 → `/skill:<name>`；
- 只列可用户激活（`prompt|inline|flow`）且 scope 含 `tui` 的；
- 同名变体记录 path。

Web：

- 同样的 `skill:` 前缀；
- `matchSlashItem` 接受裸名 → `skill:` 回退；
- 参数从剩余文本解析。

执行 `IAgentSkillService.activate()`：

1. 按 name/path 查找；
2. 拒绝非用户可激活；
3. 渲染 prompt；
4. 提交 `user` 内容，origin `{trigger:"user-slash", activationId, skillName, skillType, skillPath, skillSource, skillArgs}`；
5. 记录 `SkillActivated`；
6. 遥测 `skill_invoked` / `flow_invoked`；
7. 若 submit 没有产生 turn → `TURN_AGENT_BUSY`。

`promptWithSkills()` 支持一次多个 skill，返回 `{turn_id,prompt_id,created_at,state,steered}`。

---

## 9. 插件 skill

- 通过插件根发现，注册为 `<plugin-id>:<name>`；
- `getPluginSkill(pluginId, name)` 解析裸名；
- 渲染时若存在 `skillInstructions`，前置 `<plugin-instructions plugin="…">…</plugin-instructions>`；
- `sessionStart.skill` 在会话开始时注入一次 `<plugin_session_start plugin="…" skill="…">…</plugin_session_start>`，只对 main agent；缺失只警告一次；
- 快照通过 durable `plugin.session_start` 事件持久化。

---

## 10. 安全边界

- Skill 本身是**文本指令**，不是可执行代码；但会进入模型上下文，属于提示注入面。
- 插件 skill 的 `plugin-instructions` 明确声明"不是特权指令通道"，不能覆盖系统指令。
- `disableModelInvocation` 只阻止模型自动调用，不阻止用户 slash。
- 项目级 skill 受工作区信任影响（项目文件在未信任时不加载）。
- 同名覆盖可能让低权限来源被高权限来源替换；要意识到这一点。

---

## 11. 不变量

1. 同名 skill 只保留最高优先级（变体除外）。
2. 模型列表只包含可调用项。
3. 子 skill 不能越过父启用状态。
4. 嵌套深度 ≤3。
5. 解析失败不能中断整个目录扫描。
6. 调用必须记录来源（用户/模型/嵌套），便于审计。
7. 参数插值必须转义，不能把用户输入当 Markdown 结构。

---

## 12. 重实现最小版

1. 扫描 user/project 两个根，解析 `SKILL.md`。
2. 生成 listing 注入系统提示。
3. 实现 `Skill` 工具：按名查找 → 注入内容。
4. 加优先级与同名覆盖。
5. 加参数插值。
6. 加 slash 命令。
7. 最后加子 skill、插件 skill、嵌套深度。

测试重点：

- 目录/单文件两种解析；
- 同名覆盖顺序；
- 缺 name/description 报错；
- 不可调用类型被拒；
- 参数插值（命名/位置/引号/转义）；
- 嵌套超深报错；
- listing 截断。

---

## 13. 证据索引

- `packages/agent-core-v2/src/features/skill/catalog/{types,parser,registry,skillRoots,fileSkillDiscovery,skillSource,builtin/*}.ts`
- `packages/agent-core-v2/src/features/skill/workspace/*`
- `packages/agent-core-v2/src/features/skill/{skillService,prompt,skillOps,skillFeature}.ts`
- `packages/agent-core-v2/src/features/skill/tools/*`
- `packages/agent-core-v2/src/app/plugin/manager.ts`（插件 skill 根）
- `docs/en/customization/skills.md`
