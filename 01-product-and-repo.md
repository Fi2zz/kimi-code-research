# 01 · 产品形态与仓库地图

> 研究对象：`MoonshotAI/kimi-code` v2.7.2（commit `e12cad1`，2026-11-22），MIT。
> 本地产物：`~/.kimi-code/bin/kimi`，190,995,520 bytes，Mach-O arm64。
> 本文只描述"它是什么、由哪些部分组成"，不涉及实现细节；实现细节见后续各篇。

---

## 1. 一句话定义

Kimi Code CLI 是一个**终端优先的编码智能体运行时**：它把"模型 + 工具 + 权限 + 会话"打包成一个单文件可执行程序，同时通过同一套引擎对外提供 TUI、浏览器、桌面、IDE（ACP）和 SDK 五种入口。

它不是一个"聊天壳"，而是一个有明确状态机的 agent runtime：

```
用户输入 → turn → step → LLM 请求 → tool batch → 工具执行 → 结果回填 → 下一步
```

所有中间状态都进入事件日志，可恢复、可分叉、可导出。

---

## 2. 版本与许可

| 项 | 值 |
| --- | --- |
| 版本 | 2.7.2（`apps/kimi-code/package.json`） |
| 许可 | MIT（仓库根 `LICENSE`） |
| 公开仓库 | `github.com/MoonshotAI/kimi-code` |
| Stars / Forks | 8.3k / 1.4k（调研时） |
| Commits | 1,967 |
| 语言 | TypeScript（含 Vue） |
| 规模 | `agent-core-v2/src` 1,006 个 TS 文件 / 143,393 行；全仓 TS/TSX/Vue 约 122 万行 |

> 注意：仓库开源，但**模型服务是闭源的**（`managed:kimi-code` provider）。本地二进制与仓库同版本，因此"源码即行为"这一假设基本成立；闭源部分只有远端 API。

---

## 3. 六种表面（同一引擎，不同外壳）

| 表面 | 入口 | 传输 | 说明 |
| --- | --- | --- | --- |
| TUI | `kimi` | 进程内 | 主界面：输入框、会话流、审批面板、状态栏 |
| Headless | `kimi -p "..."` | 进程内 | 单次提示，`text` / `stream-json` 输出，退出码有语义 |
| Web | `kimi web` | 本地 HTTP/WS | 启动 KAP server，浏览器打开 |
| Desktop | Electron | 内嵌 KAP server | 复用同一 server，渲染层是 Vue |
| IDE | `kimi acp` | stdio NDJSON | Agent Client Protocol（Zed、JetBrains） |
| SDK | `@moonshot-ai/kimi-code-sdk` | 进程内 / IPC | 供其他程序嵌入 |
| Remote Control | `kimi rc` | WebSocket 隧道 | 把本机注册到 relay，外部访问本地 server |

关键点：**TUI 不是"另一个 agent"**。它通过 `KimiHarness` 调 v2 引擎；Web/桌面通过 KAP REST/WS 调同一引擎；ACP 通过 NDJSON 调同一引擎。因此权限、工具、会话行为在所有表面一致。

---

## 4. 单二进制里有什么

`strings` / 构建脚本可确认：

| 组成 | 证据 |
| --- | --- |
| Node.js 24.15.0 SEA | `NODE_SEA_BLOB`、`NODE_SEA_FUSE_fce680ab...`、`node_sea.cc` |
| 783 个 Web 资源 | `web/darwin-arm64/dist-web/assets/*`（Vue、KaTeX、Shiki 等） |
| 原生依赖 | `@mariozechner/clipboard`、`@moonshot-ai/pi-tui` 的 `.node` |
| 运行时 worker | minidb 文本索引 worker、KAP search worker |
| 应用 JS | 单个 CJS bundle（`main.cjs`）注入 SEA |

**不在二进制里**：

- `rg`（ripgrep）：优先 PATH，其次 `~/.kimi-code/bin/rg`，否则按区域 CDN 下载并校验 SHA-256。
- `fd`：只查 PATH / `~/.kimi-code/bin/fd`，没有下载器。

这解释了为什么安装脚本"不需要 Node.js"，但第一次用 `Grep` 可能需要联网。

---

## 5. 仓库地图

```
kimi-code/
├── apps/
│   ├── kimi-code/        # CLI 主程序：TUI、web 子命令、session-import、native 构建
│   ├── desktop/          # Electron 应用（内嵌 KAP server）
│   ├── vscode/           # VS Code 扩展（进程内 harness + webview）
│   └── vis/              # 会话可视化（Vite）
├── packages/
│   ├── core/             # 无 DI 内核：loop、LLM、tool executor、store
│   ├── agent-core-v2/    # DI×Scope 引擎：权限、工具、会话、子代理、特性
│   ├── kap-server/       # REST/WS 服务（Web/桌面后端）
│   ├── acp-server/       # ACP host（stdio）
│   ├── klient/           # 客户端 SDK：契约门面，可走 IPC/内存
│   ├── node-sdk/         # KimiHarness（TUI/VS Code 用）
│   ├── app-*             # Web/桌面共享 UI 逻辑（core/client/ui/components…）
│   ├── pi-tui/           # 终端 UI 库（差分渲染）
│   ├── minidb/           # 纯 Node 嵌入式 KV（WAL + snapshot）
│   ├── tree-sitter-bash/ # 纯 TS bash parser（危险命令分析）
│   ├── session-import/   # 导入 Claude Code / Codex 会话
│   ├── remote-control/   # 远程控制隧道客户端
│   ├── telemetry/        # 遥测
│   └── oauth/            # Kimi OAuth / 区域
├── docs/                 # VitePress 文档（中英）
├── skills/use-kimi-code/ # 自带 skill
├── tests/                # 场景/回放/冷恢复测试
└── scripts/              # 构建、检查、发布
```

### 分层依赖（简化）

```
apps/*  ──►  klient / node-sdk  ──►  agent-core-v2  ──►  core
                                      │
                                      ├─ kap-server
                                      └─ acp-server
```

`core` 不知道 DI、会话、权限；`agent-core-v2` 不知道 TUI；`apps` 只做传输与渲染。这个边界是重实现时最值得保留的。

---

## 6. 运行时数据目录

默认 `~/.kimi-code`（可用 `KIMI_CODE_HOME` 覆盖），目录 `0700`，文件 `0600`：

```
~/.kimi-code/
├── config.toml               # 用户配置
├── mcp.json                  # MCP 服务器
├── projects.json             # 工作区目录
├── device_id                 # 遥测设备 id
├── region                    # mainland-cn / global
├── credentials/              # OAuth 凭据（0600）
├── oauth/                    # 旧版 token 路径
├── sessions/<workspaceId>/<sessionId>/
│   ├── state.json            # 会话元数据
│   └── agents/<agentId>/
│       ├── wire.jsonl        # 事件日志（真相）
│       ├── blobs/            # 媒体
│       └── file-history/     # 文件备份
├── cache/query-store/        # minidb 会话索引（可重建）
├── plugins/                  # installed.json、managed/、cache/
├── server/server.token       # KAP bearer token
├── logs/kimi-code.log
└── search-index/
```

**核心区分**：`wire.jsonl` 是真相；`state.json` 是元数据；`cache/query-store` 是可重建索引。重实现时不要反过来。

---

## 7. 关键数字（用于校准复刻）

| 指标 | 值 | 来源 |
| --- | --- | --- |
| 内置工具（不含 MCP） | ~30 | `agent/tools` + `features/*/tools` |
| 本地真实会话 active 工具 | 87 | 会话 `llm.tools_snapshot` |
| 可延迟加载工具 | 88 | `tool_select.initialized` |
| Web 资源 | 783 | 二进制 strings |
| 会话协议版本 | 1.5 | `wire/record.ts` |
| 默认上下文压缩阈值 | 0.85 | `loop_control.compaction_trigger_ratio` |
| 工具结果内联上限 | 50,000 字符 | `toolContract.ts` |
| 工具结果落盘保留上限 | 10,000,000 字符 | 同上 |
| 危险命令解析预算 | 500 ms / 10,000 节点 | `dangerous-command-ask.ts` |
| 子代理默认超时 | 2 小时 | `[subagent].timeout_ms` |
| Swarm 上限 | 128 | `MAX_AGENT_SWARM_SUBAGENTS` |

---

## 8. 与原版 Claude Code 的差异（定位参考）

不是"换皮"，而是同一类产品的不同取舍：

| 维度 | Kimi Code | 典型 Claude Code 风格 |
| --- | --- | --- |
| 分发 | 单二进制 SEA + Web UI 内嵌 | npm 包 + 终端 |
| TUI | 自研 pi-tui，差分渲染 | Ink/React 或原生 |
| 会话 | 事件溯源 + 分支树 | JSONL 转录 |
| 工具加载 | 动态披露（`select_tools`） | 全量 schema |
| 多智能体 | 子代理 + Swarm + Tower（worktree 任务） | 子代理 |
| 权限 | 13 条策略链 + 规则引擎 | allowlist / 模式 |
| 危险命令 | 纯 TS bash 语法树 + 包装递归 | 正则/前缀匹配 |
| 远程 | 环境抽象（ssh/docker/command） | 本地为主 |
| 扩展 | MCP + Skills + 插件市场 + Hooks | MCP + hooks |

---

## 9. 重实现时需要先决定的 5 件事

1. **是否要单二进制**：要，则 Node 24.15 + SEA + 资源 manifest；不要，则先做 npm CLI，后补 SEA。
2. **是否要事件溯源**：要，则先定义 Event2 / state fold / wire 格式；不要，则退化为 JSONL 转录，后续很难加分叉/回放。
3. **是否要多表面**：要，则引擎必须与传输解耦（core 不依赖 TUI/server）。
4. **是否要动态工具**：要，则工具注册表必须支持 `inline` / `deferred` 两种披露。
5. **是否要 OS 沙箱**：原版没有；如果你要，必须在环境层加，而不是在权限层假装有。

---

## 10. 证据索引

- 产品说明：仓库 `README.md`
- CLI 表面：`apps/kimi-code/src/cli/*`、`src/tui/*`
- 包说明：各 `packages/*/package.json`
- 二进制：本地 `strings` / `otool` / `scripts/native/*`
- 会话布局：`packages/agent-core-v2/src/wire/*`、`session/*`、`persistence/*`
- 真实行为：`~/.kimi-code/sessions/.../agents/main/wire.jsonl`
