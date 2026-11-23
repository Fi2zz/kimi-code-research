# 13 · 插件（Plugins）

> 目标：解释 Kimi Code 的插件单元：清单格式、安装/更新/删除、信任门、能贡献什么、限制与失败模式。插件是"把 skill + agent + MCP + 提示 + 命令"打包安装的机制。

---

## 1. 清单位置

```
<root>/kimi.plugin.json          # 首选
<root>/.kimi-plugin/plugin.json  # 回退
```

两者都有 → 根目录优先，目录清单记为 `shadowedManifestPath`。
没有清单 → 诊断错误。

---

## 2. 清单 schema

```jsonc
{
  "name": "my-plugin",            // 必填，正则约束，小写后作为 id
  "version": "1.0.0",
  "description": "...",
  "keywords": ["..."],
  "author": { "name": "...", "email": "..." },
  "homepage": "...",
  "license": "MIT",

  "skills": ["./skills"],          // 字符串或数组；必须在插件根内
  "rootSkillFallback": true,       // 省略 skills 且根有 SKILL.md 时
  "agents": ["./agents"],
  "sessionStart": { "skill": "welcome" },

  "mcpServers": {
    "data": { "transport": "stdio", "command": "node", "args": ["server.js"] }
  },

  "hooks": [ { "event": "SessionStart", "command": "..." } ],

  "commands": ["./commands"],

  "interface": {
    "displayName": "...",
    "shortDescription": "...",
    "longDescription": "...",
    "developerName": "...",
    "websiteURL": "...",
    "defaultLocale": "en-US",
    "locales": "./locales",
    "mcpOverrides": {},
    "skillOverrides": {}
  },

  "skillInstructions": "...",
  "systemPrompt": "...",           // ≤32 KiB
  "systemPromptPath": "./PROMPT.md" // ≤32 KiB
}
```

`name` 正则：

```
^[A-Za-z0-9_-]+(?:\.[A-Za-z0-9_-]+)*(?:@[A-Za-z0-9_-]+)?$
```

路径约束：

- `skills/agents/commands/systemPromptPath` 必须在插件根内（realpath 后）；
- `mcpServers` 的 `command` 是 PATH 命令或 `./` 相对路径；含 `/` 但不是 `./` → 拒绝；
- stdio `cwd` 必须是 `./` 且根内。

不支持字段：`tools`、`apps`、`inject`、`configFile`、`bootstrap` → 信息诊断。

---

## 3. 持久化

```
$KIMI_CODE_HOME/plugins/
├── installed.json          # {version:1, plugins:[...]}
├── availability.json       # 市场可用性缓存（+ .lock）
├── managed/<id>/           # 托管安装副本
└── cache/archives/<sha256>.zip  # ≤500 MiB LRU
```

`installed.json` 记录：

```ts
{
  id, root, source, origin, catalogId?, marketplaceSource?,
  enabled, installedAt, updatedAt?, originalSource?,
  capabilities?, github?
}
```

原子写（tmp + rename）。解析/IO 失败 → `PLUGIN_LOAD_FAILED`，要求修复并 `/plugins reload`。

安装是**按用户**的，对所有项目生效；暂不支持项目级安装。

---

## 4. 安装 / 更新 / 删除

### 来源

- GitHub URL：`/`、`/tree/<ref>`、`/releases/tag/<tag>`、`/commit/<sha>` → codeload zip；
- 任意 `http(s)://` zip；
- 本地绝对路径；
- 相对路径拒绝。

GitHub 版本解析：

- 最新 release tag：`github.com/<o>/<r>/releases/latest` 重定向；
- 回退 `codeload.../zip/HEAD`；
- commit SHA：commits `.atom`。

### 安全

- 市场安装要求 `expectedSha256` + zip URL；
- 下载后校验 sha256，不匹配拒绝；
- zip 解压拒绝路径穿越，恢复 unix mode，识别单一嵌套根；
- 下载超时 5 分钟。

### 流程

1. 解析来源 → 下载 → 校验；
2. 解压到暂存；
3. 解析清单；id 不匹配 → 失败；
4. 复制到 `managed/<id>`（旧根先改名备份）；
5. 持久化 `installed.json`；
6. 可选 reload，并断言插件存在、健康、所需 skill 存在；
7. 失败 → 恢复旧记录/旧副本，否则聚合回滚错误。

更新 = 重装同一 `managed` 根；旧副本成功后删除。
GitHub 更新检查：显式 pin/ref/SHA 比较；tag/SHA pin 不更新；branch/HEAD 比较最新 SHA。

删除只删安装记录；`managed` 副本与源保留。删除远程 MCP 插件的 OAuth 凭据会失效。

---

## 5. 信任门

**核心安装 API 没有信任提示**；门在 UI 层：

- TUI `/plugins install`、非官方市场安装 → 弹 `PluginInstallTrustConfirmComponent`：
  "第三方插件，Kimi 未审查……可能捆绑 MCP 服务器、skills 或运行代码的文件"，确认按钮 "Trust and install"。
- "官方"只是 URL 白名单：
  - `code.kimi.com|code.kimi.ai` 的 `/kimi-code/plugins/official/`；
  - `cdn.kimi.com|cdn.kimi.ai` 的 `/kimi-computer-use*/`。
- 删除总是需要确认。
- 市场 UI 的 `previewPlugin` 会下载 **sha256 固定**的 zip、解析清单、返回能力，不安装。

工作区信任**不覆盖**插件；插件代码不依赖 `IWorkspaceTrustStore`。

安全模型：

- 不支持命令工具/旧版 tool env 执行；
- 所有清单路径 realpath 后必须在插件根内；
- 坏清单只产生诊断。

---

## 6. 贡献

只有 `available && enabled && state === "ok" && manifest` 才贡献：

| 能力 | 形态 |
| --- | --- |
| Skills | `SkillRoot { source:"extra", plugin:{id,instructions}, scanMode }` |
| Agents | `{ path, source:"plugin" }`，优先级低于 user/extra/project/`--agent-file` |
| MCP servers | 运行时名 `plugin-<id>:<server>`；stdio 强制 `KIMI_CODE_HOME`、`KIMI_PLUGIN_ROOT`；默认 cwd 插件根 |
| Hooks | 来自启用插件；cwd = 插件根；env 注入 `KIMI_CODE_HOME`、`KIMI_PLUGIN_ROOT` |
| Commands | Markdown 命令；名称 = 相对路径去 `.md` |
| sessionStart | `{pluginId, skillName}`，会话开始注入一次 |
| systemPrompt | 聚合，标记 `<!-- From: plugin <id> -->` |
| skillInstructions | 附加到该插件的每个 skill 渲染 |
| interface | 仅展示/本地化 |

Electron 下 `node` 命令会被改写（`ELECTRON_RUN_AS_NODE=1` 或原生二进制 `__plugin_run_node` 子命令）。

---

## 7. 限制

- 每个 `systemPrompt` 字段/文件 ≤32 KiB；超限忽略 + 诊断；
- 所有插件 system prompt 聚合 ≤64 KiB/次构建；超预算插件被跳过并警告；
- zip 下载 5 分钟；
- archive 缓存 500 MiB；
- 市场目录请求 10 秒；重定向最多 10 次；
- 安装大体原子（记录 + 副本可回滚），但没有插件文件大小上限。

---

## 8. 市场

- 默认 `${baseUrl}/plugins/marketplace`，base 来自 provider/区域或 `KIMI_CODE_BASE_URL`；
- 环境覆盖 `KIMI_CODE_PLUGIN_MARKETPLACE_URL`；
- 托管目录要求 version/distribution/platforms/http(s) zip + sha256；
- 托管目录请求带 Kimi OAuth bearer；artifact 401/403 同源重试一次；跨域重定向剥敏感头；
- 条目字段：`id, displayName, source|url|downloadUrl, classification, tier, version, description, sha256sum, filename, clients, platforms, distribution{...}, locale`；
- 版本/SHA 漂移 → `PLUGIN_CATALOG_CHANGED`；zip 安装要求展示摘要。

---

## 9. 不变量

1. 没有清单不能安装。
2. 清单路径不能逃出插件根。
3. 安装/更新失败必须回滚记录与副本。
4. 未启用插件不能贡献任何能力。
5. 删除插件不能删除用户源码/托管副本（只删记录）。
6. 插件 system prompt 不是特权指令通道，不能覆盖系统指令。
7. 市场安装必须校验 sha256。
8. 插件的 MCP stdio 不能继承本机全部环境（只注入声明的）。
9. 插件升级后，已存在会话不自动换提示/工具；需 `/new` 或 `/reload`。

---

## 10. 重实现最小版

1. 解析 `kimi.plugin.json`，校验名字/路径。
2. 支持 skills + agents + MCP 三类贡献。
3. 实现 `installed.json` 与 managed 目录。
4. 从本地路径/zip 安装，校验路径与 id。
5. 加启用/禁用与 reload。
6. 加 GitHub/市场下载 + sha256。
7. 加 UI 信任确认（至少文档/CLI 提示）。
8. 最后加 hooks/commands/systemPrompt。

测试重点：

- 清单缺 name → 失败；
- 路径 `../` → 拒绝；
- 安装后 skill 可见；
- 禁用后贡献消失；
- 更新失败回滚；
- sha256 不匹配拒绝；
- 插件 MCP 工具名带 `plugin-<id>:` 前缀。

---

## 11. 证据索引

- `packages/agent-core-v2/src/app/plugin/{manifest,manager,store,source,archive,packageCache,commands,marketplace,github-resolver,availability}.ts`
- `packages/agent-core-v2/src/agent/plugin/*`、`agent/pluginCommand/*`
- `apps/kimi-code/src/tui/components/dialogs/plugins-selector.ts`
- `apps/kimi-code/src/tui/utils/plugin-source-label.ts`
- `docs/en/customization/plugins.md`
