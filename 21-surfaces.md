# 21 · CLI 入口与附加服务：TUI、Headless、ACP、KAP 与远程控制

> 目标：解释 CLI 如何在同一核心引擎上提供终端、进程间和可选网络入口，以及各入口的协议、鉴权与能力差异。重实现时先做 headless + TUI；ACP、KAP 和远程控制按产品需要增加。
>
> 范围：本文不把 Kimi Code Desktop 或 VS Code 扩展当作 CLI 的组成部分。ACP 是 `kimi acp` 提供的 IDE 接入协议，不等同于 VS Code 扩展。

---

## 1. 共同起点

本文描述的 CLI 入口都：

1. 调 `bootstrap()` 创建 App `Scope`；
2. 通过 DI 拿 `IConfigService`、`ISessionManager`、`IAgentLifecycleService` 等服务；
3. 用 `klient`（契约门面）或直接 accessor 调引擎；
4. 只把"传输事件"翻译成引擎调用，不实现业务逻辑。

App scope 的存储目录权限：`0700`，文件 `0600`。

---

## 2. TUI

入口：`apps/kimi-code/src/main.ts` → `cli/main.ts` → `runShell` / `runPrompt`。

- TUI 通过 `KimiHarness`（`@moonshot-ai/kimi-code-sdk`）在进程内驱动 v2 引擎；
- 终端渲染基于 `pi-tui`（差分渲染）；
- 负责：输入框、会话流、审批面板、状态栏、slash 命令、文件补全、图片/视频粘贴；
- 权限模式、Plan、模型、工具都通过 SDK 调引擎，不直接改状态。

Headless `kimi -p`：

- 单次 prompt；
- `--output-format text|stream-json`；
- 退出码有语义（如 goal blocked=3、paused=6）。

---

## 3. Web / KAP server

`kimi web` 在进程内启动 `@moonshot-ai/kap-server`。

### 3.1 网络

- 默认 `127.0.0.1:58627`；
- 端口占用则 +1 重试，最多 100 次；
- 服务 `dist-web` / SEA 解包后的 Web 资源；
- `--json` 使用临时内存 bearer token，输出：
  ```json
  { "origin": "...", "ws_url": "...", "token": "...", "kimi_agent_protocol_version": "..." }
  ```

### 3.2 API

- 统一信封 `{ code, msg, data, request_id }`；
- 路由前缀 `/api/v1`；
- 主要路由：
  ```
  auth/oauth, config, client-configs, model-catalog,
  sessions (+ context, agents, prompts/steer, interactions,
            file-history, media, export, rating, skills),
  workspaces/fs, hosts, environment, search, tools,
  mcp, plugins, capabilities, remote-control, shutdown,
  healthz, OpenAPI/AsyncAPI
  ```

### 3.3 鉴权

- bearer token 来自 `$KIMI_CODE_HOME/server.token`（32 字节随机 base64url，`0600`）；
- 常量时间比较；按 mtime/inode 缓存；
- 权限过宽拒绝；
- 豁免：OPTIONS、`GET /api/v1/healthz`、非 `/api/` 非 meta 路径；
- 401 码 `40101`；非 loopback 失败有限流；
- `--dangerous-bypass-auth` 关闭所有 REST/WS 鉴权（危险）。

### 3.4 Host / Origin

- Host 允许：`localhost`、`*.localhost`、任意 IP、绑定 host、`KIMI_CODE_ALLOWED_HOSTS`（逗号分隔，`.` 前缀表示后缀）；
- `KIMI_CODE_DISABLE_HOST_CHECK=1` 关闭；
- 403 码 `40301`；
- Origin：Origin host == Host（忽略端口）或都为 loopback，或命中 `KIMI_CODE_CORS_ORIGINS`；
- 回显 origin，允许 `Authorization`、`X-Kimi-Client-*`；
- WS upgrade 也检查 host/origin。

### 3.5 WebSocket

- 路径 `/api/v1/ws`；
- bearer 从 `Authorization` 或 `Sec-WebSocket-Protocol`；
- upgrade 前 401/403；
- 支持 JSON 批次、背压、心跳；
- debug WS 仅 loopback。

### 3.6 绑定

- 非 loopback 且无 TLS → 拒绝，除非 `--insecure-no-tls`；
- shutdown 仅 loopback 或 `--allow-remote-shutdown`；
- debug endpoints 仅 loopback。

---

## 4. ACP server

`kimi acp` 通过 stdin/stdout 跑 NDJSON ACP：

- 创建 v2 core + 内存 `klient`；
- 方法：
  ```
  initialize, authenticate, logout,
  session/new|load|resume|list|close|delete|fork,
  session/set_mode, session/set_config_option,
  session/prompt, session/cancel,
  session/set_model (custom)
  ```
- 能力：
  ```
  loadSession: true
  prompt: { image: true, audio: false, embeddedContext: true }
  session: { list, resume, close, delete, fork, additionalDirectories }
  mcp: { http, sse }
  auth: { logout }
  ```
- 模式映射：
  ```
  default → manual
  plan    → plan + manual
  auto    → auto
  yolo    → yolo
  ```
- 模型/thinking/mode 作为 config option；
- 工具调用、计划、usage、配置更新映射到 `session/update`；
- 权限与提问通过 `AcpInteractionBridge` / elicitation。

---

## 5. 非目标：Desktop 与 VS Code 扩展

上游 monorepo 还包含两个独立产品形态：

- Desktop 由 Electron main 启动内嵌 KAP server，renderer 走 REST/WS；
- VS Code 扩展在扩展宿主内运行 `KimiRuntime` / `createKimiHarness`，webview 使用自己的 RPC bridge。

二者可以复用核心引擎，但运行方式、配置和产品能力不能直接套用到 CLI。它们的实现与重实现不在本文范围内。

---

## 6. 远程控制

`kimi rc` / `packages/remote-control`：

- 把本机注册到 relay（默认 `https://code-rc.kimi.com`，全球 `https://code-rc.kimi.ai`）；
- 需要 Kimi 登录 refresh token + 本地 bearer token；
- 单实例锁 `$KIMI_CODE_HOME/server/rc.json`；
- 管理 WS `/v1/remote/create`：发 `register`，等 `register_ack`；
- HTTP 隧道 WS `/v1/remote/http?device_id=...&capabilities=request_chunk,response_chunk,abort`；
- 每流 WS `/v1/remote/stream/<id>`；
- 转发到本地 origin，带 `Authorization: Bearer <server.token>`；
- 过滤 hop-by-hop/auth 头；限制字节；gzip 重写；ETag；
- URL：`https://code-rc.kimi.com/devices/<deviceId>/?rc=1&from=<urlSource>`；
- 重连 ≤30s，ping 30s，静默 300s。

---

## 7. CLI 入口能力矩阵

| 能力 | TUI | Headless | ACP | 可选 Web/KAP |
| --- | --- | --- | --- | --- |
| 会话列表/恢复 | ✓ | 命令参数选择会话 | ✓ | ✓ |
| 人工审批 | ✓ | 非交互，默认拒绝 ask | ✓ | ✓ |
| 图片/视频输入 | ✓ | 取决于输入与模型 | 图片 | ✓ |
| Plan 审阅 | ✓ | 非交互时保持 Plan | ✓ | ✓ |
| 多会话 | ✓ | 每次进程运行一次任务 | ✓ | ✓ |
| 终端渲染 | ✓ | 文本 / stream-json | — | — |
| 浏览器访问 | — | — | — | ✓ |
| 远程控制 | — | — | — | 通过 `kimi rc` 可选启用 |

---

## 8. 不变量

1. 表面不能绕过权限/工具策略；所有决策在引擎层。
2. 鉴权必须在 upgrade/路由之前。
3. Host/Origin 检查必须覆盖 WS。
4. 非 loopback 无 TLS 必须拒绝（除非显式危险开关）。
5. token 必须 `0600`，不能写日志。
6. ACP 的模式映射必须一一对应，不能把 `auto` 映射成 `manual`。
7. 各入口的本地设置不能暗中改变引擎配置语义；非目标产品的设置不属于本矩阵。
8. 远程控制只是转发，不能改变鉴权模型。

---

## 9. 重实现最小版

1. 先做 headless + 引擎 API。
2. TUI 接引擎（先纯文本）。
3. 用真实 CLI 验收会话、工具、审批和恢复。
4. 需要 IDE 集成时再加 ACP：`initialize` + `session/new` + `prompt` + 审批。
5. 需要浏览器入口时再加 KAP：会话 CRUD + prompt + 审批 + WS，并加 token/host/origin。
6. 远程控制最后做；Desktop/VS Code 不属于本计划。

测试重点：

- 无 token 请求 401；
- Host 不在白名单 403；
- Origin 不匹配拒绝；
- WS 同样鉴权；
- ACP 模式映射；
- 审批事件在 TUI/Web 都能渲染；
- 非 loopback 无 TLS 拒绝。

---

## 10. 证据索引

- `apps/kimi-code/src/{main.ts,cli/**,tui/**}`
- `packages/kap-server/src/{start.ts,routes/**,middleware/**,transport/ws/**,security/**}`
- `packages/acp-server/src/{start.ts,server.ts,modes.ts,config-options.ts}`
- 非目标背景：`apps/desktop/src/main/server/*`、`apps/vscode/src/runtime/*`
- `packages/remote-control/src/*`
- `packages/node-sdk/src/*`
