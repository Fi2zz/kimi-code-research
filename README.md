# Kimi Code CLI 逆向调研文档集

研究对象：`MoonshotAI/kimi-code` v2.7.2（commit `e12cad1`，MIT）与本机 `~/.kimi-code/bin/kimi`。
方法：源码阅读 + 本地二进制静态检查（含 SEA 资产抽取/哈希校验）+ 真实会话 `wire.jsonl` + 官方文档。共 29 篇，约 11,000 行。

> 这些文档是**调研**，不是实现代码。每篇都给出接口、算法、格式、不变量、失败模式与证据路径，目标是理解并选择性重建 Kimi Code CLI 与核心 agent 运行时。

## 范围

- **主范围**：CLI 的 TUI、headless、配置、会话、Agent 循环、LLM、上下文、工具、权限、安全与扩展机制。
- **CLI 附加入口**：`kimi acp`、`kimi web` / KAP、`kimi rc`。它们不是首版重实现的必要条件。
- **非目标**：Kimi Code Desktop、VS Code 扩展及其 Electron/webview/UI 细节。上游仓库中的相关路径只用于说明代码归属，不用于推导 CLI 行为。
- **共享引擎的边界**：不同入口可以复用权限、工具与会话核心逻辑，但不因此获得相同的 UI、配置或产品能力。

---

## 阅读顺序

| # | 文档 | 一句话 |
| --- | --- | --- |
| 01 | [产品与仓库地图](01-product-and-repo.md) | CLI 入口、可选接入、单二进制构成、数据目录 |
| 02 | [构建与发布](02-build-and-release.md) | Node SEA 五步、资源打包、签名、更新 |
| 03 | [DI 架构](03-architecture-di.md) | DI × Scope、装饰器、集合、级联、释放 |
| 04 | [事件与会话](04-events-and-sessions.md) | Event2、状态折叠、wire.jsonl、重放、分叉 |
| 05 | [Agent 循环](05-agent-loop.md) | turn/step、工具批次、重试、steer、通知 |
| 06 | [LLM 适配层](06-llm-layer.md) | 四协议、消息模型、tool ID、流式、thinking |
| 07 | [上下文与压缩](07-context-and-compaction.md) | 投影/配对、提醒、媒体、压缩、token |
| 08 | [工具框架](08-tool-framework.md) | 契约、注册、执行、截断、并发、去重 |
| 09 | [内置工具规格](09-builtin-tools.md) | 约 30 个工具的参数/默认值/坑 |
| 10 | [动态工具](10-dynamic-tools.md) | `select_tools`、deferred、公告、降级 |
| 11 | [MCP](11-mcp.md) | 三层配置、传输、OAuth、命名、重连 |
| 12 | [Skills](12-skills.md) | 发现、frontmatter、优先级、参数 |
| 13 | [Plugins](13-plugins.md) | 清单、安装、信任、贡献、市场 |
| 14 | [权限系统](14-permissions.md) | **13 条策略链、三模式、规则、审批** |
| 14A | [权限逐函数实现](14A-permissions-function-level.md) | 每个类/方法/事件/测试点 |
| 15 | [危险命令分析](15-dangerous-command-analysis.md) | **纯 TS bash 解析、包装递归、盲区** |
| 15A | [危险命令逐函数实现](15A-dangerous-command-function-level.md) | lexer/parser/analyzer 全链路 |
| 16 | [路径与工作区守卫](16-path-and-workspace-guards.md) | 敏感文件硬阻断、git、信任 |
| 17 | [Hooks](17-hooks.md) | 20 事件、退出码、fail-open |
| 18 | [子代理](18-subagents.md) | Agent 工具、隔离、resume、超时 |
| 19 | [Swarm 与 Tower](19-swarm-and-tower.md) | 批量并行、worktree mission、合并门禁 |
| 20 | [环境与远程执行](20-environments.md) | ssh/docker、能力模型、协议、限制 |
| 21 | [CLI 入口与附加服务](21-surfaces.md) | TUI、headless、ACP、可选 KAP/远程控制 |
| 22 | [配置系统](22-configuration.md) | 分层、env 绑定、TOML、热更新 |
| 23 | [数据与格式](23-data-and-formats.md) | 目录、wire、state.json、索引、导出 |
| 24 | [安全模型](24-security-model.md) | 资产、边界、攻击树、缺口 |
| 25 | [重新实现蓝图](25-reimplementation-plan.md) | 里程碑、接口、任务、测试矩阵 |
| 26 | [二进制一致性核验](26-binary-vs-source-verification.md) | SEA 抽取、哈希、tag 2.7.2 比对 |
| 27 | [详细任务计划](27-reimplementation-task-plan.md) | 20 个任务、验收命令、测试矩阵 |

---

## 十个最重要的结论

1. **它是单二进制 Node 24 SEA**：190MB，内嵌 783 个 Vue Web 资源；`rg` 是运行时下载，`fd` 不内嵌。
2. **会话是事件溯源**：`wire.jsonl` 是真相，`state.json` 是元数据，索引可重建。
3. **权限是 13 条 first-match-wins 策略链**，顺序即语义；deny 永远最先。
4. **manual 模式在 git 工作区内自动放行 Write/Edit**（有 file history 可撤销），但 Bash 仍要问。
5. **敏感文件是硬阻断**（`.env`/私钥），连 `allow` 规则也绕不过——但 `Bash cat .env` 可以绕。
6. **危险命令分析用纯 TS 重写的 tree-sitter-bash**，递归拆 `sudo/env/sh -c/eval/busybox`，失败即问（yolo 下放行）。
7. **没有 OS 级沙箱**：所有限制都是应用层策略；远程环境也不是沙箱。
8. **工具分 inline/deferred**：`select_tools` 按需注入 schema，长会话/大量 MCP 的关键优化。
9. **子代理默认不能递归派发**；Tower worker/reviewer 强制 Never Ask。
10. **CLI 各入口复用同一个 v2 引擎**：TUI、headless 与 ACP 共享权限、工具和会话核心逻辑；这不等于 Desktop、VS Code 等其他产品形态具有相同的完整行为。

---

## 证据强度

| 级别 | 来源 |
| --- | --- |
| 强 | 源码（文件+行号）、真实 `wire.jsonl`、二进制 strings |
| 中 | 官方文档、测试用例 |
| 弱/推断 | 文档中标注"推断/未验证"的部分 |

版本一致性：本地二进制 `2.7.2` 与仓库 `apps/kimi-code/package.json` 相同，因此源码行为可代表本地。进一步核验（见 26 篇）表明二进制对应 **tag `@moonshot-ai/kimi-code@2.7.2`**，且权限/危险命令等核心文件在 tag 与 main 之间完全相同。

协议版本注意：KAP（REST/WS）是 **1.0.0**，wire.jsonl 是 **1.5**，两者不要混用。

---

## 已知未覆盖与非目标

- 具体的模型服务端行为（闭源 API）。
- Tower 的极端并发/故障恢复。
- 遥测服务端。
- 部分历史迁移代码。
- Desktop、VS Code 扩展及其 UI 是明确的非目标，不属于"尚未覆盖的 CLI 行为"。

这些缺口不妨碍重建一个 CLI 核心系统，但意味着不能据此声称完整复刻整个 Kimi Code 产品族。
