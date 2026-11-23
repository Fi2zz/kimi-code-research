# 11 · MCP（Model Context Protocol）

> 目标：解释 Kimi Code 如何发现、连接、命名、鉴权、缓存和调用 MCP 服务器，以及失败时怎么办。重实现时，MCP 是"工具数量爆炸"的主要来源，必须先把生命周期做对。

---

## 1. 配置文件与优先级

`app/mcpConfig/configLoader.ts` 按顺序读取：

1. `$KIMI_CODE_HOME/mcp.json`（用户）
2. `<git 工作树根>/.mcp.json`（项目根）
3. `<cwd>/.kimi-code/mcp.json`（项目）

合并规则：同名服务器**后者覆盖前者**；记录 `origins[name]` 指向文件。

- `.mcp.json` 的 stdio `cwd` 默认是项目根的父目录；相对 cwd 相对于它解析。
- 项目文件受**工作区信任**门控：未信任时只加载用户文件。
- 管理写入（`/mcp-config`）写回用户 `mcp.json`。

全局 `[mcp]` 配置：

```toml
[mcp]
startup_timeout_ms = 30000
tool_timeout_ms = 30000
default_deferred = false
reconnect_initial_delay_ms = 5000
reconnect_max_delay_ms = 300000
```

优先级：每服务器字段 > 环境变量 > `config.toml` > 内置默认。

环境变量：

```
KIMI_MCP_STARTUP_TIMEOUT_MS
KIMI_MCP_TOOL_TIMEOUT_MS
KIMI_MCP_RECONNECT_INITIAL_DELAY_MS
KIMI_MCP_RECONNECT_MAX_DELAY_MS
```

---

## 2. 服务器 schema

`mcpCore/config-schema.ts`：

```ts
// 公共
{
  enabled?: boolean;
  deferred?: boolean;
  startupTimeoutMs?: number;
  toolTimeoutMs?: number;
  enabledTools?: string[];   // 原始工具名
  disabledTools?: string[];  // 原始工具名
}

// stdio
{
  transport: "stdio";
  command: string;
  args?: string[];
  env?: Record<string, string>;
  cwd?: string;
  executor?: "local" | "kaos";
  environment_id?: string;
}

// http
{
  transport: "http";
  url: string;
  headers?: Record<string, string>;
  auth?: "oauth";
  bearerTokenEnvVar?: string;
}

// sse
{
  transport: "sse";
  url: string;
  headers?: Record<string, string>;
  auth?: "oauth";
  bearerTokenEnvVar?: string;
}
```

预处理：

- 无 `transport` + `command` → stdio；
- 无 `transport` + `url` → **http**；
- 显式 `transport` 优先。

无效 schema → `CONFIG_INVALID`，加载失败。

---

## 3. 传输

| 传输 | SDK | 说明 |
| --- | --- | --- |
| stdio | `Client` + 自定义 `EnvironmentStdioTransport` | 通过环境解析器 spawn；本地合并父环境，远程只用 `env` 覆盖；stderr 保留 4 KiB 尾部 |
| http | `StreamableHTTPClientTransport` | headers + 可选 `Authorization: Bearer <env>`；OAuth fetch/authProvider |
| sse | `SSEClientTransport` | 同上 |

stdio 关闭：SIGTERM → 5s → SIGKILL。
启动永久错误（ENOENT/EACCES/ENOEXEC/ENOTDIR/EISDIR）→ 归类 `config`，不重试。
`executor` 非 `local` → `NOT_IMPLEMENTED`。

---

## 4. 工具命名

`mcpCore/tool-naming.ts`：

```
mcp__<server>__<tool>
```

- server/tool 中非 `[a-zA-Z0-9_-]` → `_`，连续 `_` 合并；
- 总长 ≤64；超长 → `head + "_" + fnv1a32-hex`。

例：

```
plugin-kimi-datasource:data + call_data_source_tool
→ mcp__plugin-kimi-datasource_data__call_data_source_tool
```

冲突规则（`agent/mcp/mcpService.ts`）：

- 同服务器重复工具 → 保留第一个，丢弃后者（`MCP_TOOL_NAME_COLLISION`）；
- 不同服务器同名 → 丢弃**新**工具；
- 同服务器重连 → 替换旧注册。

---

## 5. 发现与缓存

- 连接 = `connect()` + `listTools()`；
- 每个工具 input schema 必须是 JSON object，否则 `MCP_STARTUP_FAILED`；
- 缓存 `tools`、`rawTools`、`enabledNames`、`connectedAt`；
- `enabledNames` = `enabledTools` 允许列表 → 再减去 `disabledTools`，匹配原始工具名；
- durable discovery 记录：对 `{tools, enabledNames, collisions}` 做 sha256，key `${server}\n${hash}`，存 replayable state `mcp.discovery`；
- 工具→服务器索引：`mcp.mcpToolsByServer`；
- 失败开放集合：`mcp.failures.open`。

Discovery 写入在会话恢复前缓冲，恢复后 flush。

---

## 6. 生命周期与状态

公开状态：

```
pending | connected | failed | disabled | needs-auth | removed
```

内部：

```
connecting | connected | probing | retryWaiting | failed | needsAuth | disabled | removed
```

- `probing` 对外映射为 `connected`。
- 默认：启动 30s；重连 5s → 300s 指数退避 + jitter（0.5–1.0），最大失败连击 31。
- 启动失败分类：
  - `config | protocol | auth-static` → `failed`（不重试）；
  - `auth-oauth` 且有 OAuth 服务 → `needs-auth`；
  - 其它 → `retryWaiting`（若启用重连）。
- 已连接后断开：
  - stdio 硬关闭；
  - HTTP 重连耗尽 → 弱探测（`ping`，5s）；成功 → connected，失败 → retry/failed。
- 单一定时器按最早 `nextRetryAt` 调度。
- `remove/markRemoved`：条目仍列出，但工具调用返回删除错误；新会话不再注册。
- `needs-auth`：注销真实工具，注册合成工具 `mcp__<server>__authenticate`（阻塞 15 分钟等授权），通过 tool update 发出授权 URL；成功后重连并用真实工具替换。

初始加载：`connectAll()` + `waitForInitialLoad`；agent 在加载完成前阻止 step/tool 执行。

---

## 7. OAuth

触发：

- 远程配置、无 `bearerTokenEnvVar`、`auth: "oauth"` 或无静态 headers；
- `/mcp-config login <name>` 或合成 authenticate 工具；
- 启动 401 分类为 `needs-auth`。

流程：

1. SDK `discoverOAuthServerInfo`（带缓存）。
2. 本地回调服务器绑定 `127.0.0.1:0`，路径 `/callback`。
3. PKCE + 16 字节随机 `state`；不匹配 → `MCP_OAUTH_FAILED`（CSRF）。
4. 交换 code 必须返回 `AUTHORIZED`。
5. 进行中的流程共享；可按 `waitId` 重连；可取消。
6. scope：若服务器声明 `offline_access` 且基础 scope 没有，则追加。

凭据存储：

```
$KIMI_CODE_HOME/credentials/mcp/<sanitize(name)-sha256(name\0canonicalUrl)[0:24]>-tokens.json
                                                                    -client.json
                                                                    -discovery.json
                                                                    -meta.json
```

- canonical URL 去掉 hash；
- token 带 `obtained_at`；
- `OAuthTokenTransaction` 提供原子/compare-and-clear；
- meta 在提交后写，清除时删。

客户端元数据：`token_endpoint_auth_method: "none"`、`grant_types: ["authorization_code","refresh_token"]`、`response_types: ["code"]`、默认 `client_name: "kimi-code (<server>)"`。

刷新：在 `expires_at - min(120s, lifetime/2)` 调度；失败发 `refresh-failed`，若已连接则重连。

失效：服务器删除/URL 变化/插件删除 → 清 tokens/client/discovery/verifier。

---

## 8. 权限交互

- MCP 工具注册为 `source: "mcp"`，`disclosure` 由 `deferred` 决定。
- `approvalRule` = 完整工具名。
- MCP 工具**不在** `DEFAULT_APPROVE_TOOLS`，所以默认落到 `fallback-ask`。
- 权限规则只对 MCP 名字支持 `*`/`**`（picomatch）；非 MCP 通配永远不匹配。
- profile 里用 `'mcp__*'` 表示"允许该 profile 使用 MCP"。
- "本会话允许"存的是完整工具名规则。

---

## 9. 结果整形

`agent/mcp/output.ts`：

- 单个二进制 part 上限 10 MiB；超限保存后替换为提示，`truncated: true`。
- 支持 text/image/audio/resource/resource_link；不支持 → `[MCP content dropped: …]`。
- `image_url` 受模型能力门控。
- resource link 不自动抓取。
- 附件持久化到 session media store（`f_<sha256>` + `kimi-file://` daemon URL + `media/<fileId><ext>`）。
- `structuredContent` 以 `<mcp-result-extras>{...}</mcp-result-extras>` 追加（若文本里没有相同 JSON）；`_meta` 过滤保留键。
- 仅媒体结果包 `<mcp_tool_result name="…">`。
- 单文本折叠为字符串；保留 `isError`。

---

## 10. 调用失败

`agent/mcp/tools/mcp.ts`：

- 401 → `markNeedsAuth`，返回 `MCP_OAUTH_FAILED` 指引；
- 其它传输失败 → ping：
  - 存活 → 立即重试一次；
  - 否则状态更新、重连、再重试一次；
  - 重连失败 → `MCP_STARTUP_FAILED`；
- 协议错误/畸形结果/abort → 不重试。

---

## 11. 不变量

1. 工具名必须全局唯一；冲突按"先到先得"。
2. 连接失败不能阻塞整个会话；工具应显示为不可用。
3. `needs-auth` 必须有可操作的授权入口。
4. 凭据文件权限 0600；不得写入日志。
5. discovery 缓存必须有 hash；重放时不能重复注册。
6. 删除服务器后，旧工具调用必须明确报错，不能静默。
7. OAuth state 必须校验；回调只接受 GET。
8. 远程 stdio 的环境变量只来自配置，不继承本机。

---

## 12. 重实现最小版

1. 读三层 `mcp.json` + schema 校验。
2. 实现 stdio 连接 + `listTools`。
3. 命名 `mcp__server__tool` + 冲突处理。
4. 注册到工具表（先 inline）。
5. 实现状态机 + 重连。
6. 加 http/sse。
7. 加 OAuth。
8. 加 deferred/discovery 缓存。
9. 最后做结果整形与附件。

测试重点：

- 三层覆盖顺序；
- 同名工具冲突；
- 启动失败分类；
- 断线重连后工具可用；
- 401 触发授权入口；
- OAuth state 不匹配被拒；
- 大媒体落盘；
- 删除服务器后调用报错。

---

## 13. 证据索引

- `packages/agent-core-v2/src/mcpCore/{config-schema,tool-naming,connection-manager,client-*}.ts`
- `packages/agent-core-v2/src/mcpCore/oauth/*`
- `packages/agent-core-v2/src/agent/mcp/*`
- `packages/agent-core-v2/src/app/mcpConfig/*`、`app/mcpManagement/*`、`app/mcpRegistry/*`
- `packages/agent-core-v2/src/workspace/workspaceMcp*/*`
- `docs/en/customization/mcp.md`
