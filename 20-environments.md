# 20 · 环境抽象与远程执行

> 目标：解释工具如何"不关心本机还是远程"，以及 ssh/docker/自定义 launcher 的协议、能力模型、路径类、生命周期与限制。重实现时，这是把 agent 从"本机脚本"升级为"可远程开发"的关键抽象。

---

## 1. Environment 模型

`environment/environment.ts`：

```ts
type EnvironmentCapability = "fs" | "process";
type EnvironmentStatus = "pending" | "connecting" | "ready" | "disconnected" | "disposed";

interface Environment {
  identity: { environmentId: string; generation: string };
  capabilities: Set<EnvironmentCapability>;
  host?: HostInfo;                 // osKind/osArch/shellName/shellPath/pathClass/homeDir...
  path: EnvironmentPath;           // separator/delimiter/isAbsolute/join/relative/resolve/basename/dirname
  workspace: { mapRoots(roots): roots };
  shellPathBridge?: ShellPathBridge;
  fs?: IHostFileSystem;
  process?: IHostProcessService;
  status: EnvironmentStatus;
  onDidChangeStatus: Event<...>;
  whenReady(): Promise<void>;
  connectError?: Error;
  connect?(): Promise<void>;
  dispose(): void;
}

interface EnvironmentLease {
  environment: Environment;
  track(resource, sessionId?): resource;
  dispose(): void;
}
```

本地环境 id 固定 `local`，始终注册。

---

## 2. 能力（capabilities）

| 能力 | 提供 |
| --- | --- |
| `fs` | 文件读写、目录遍历、路径规范化 |
| `process` | spawn、stdin/stdout/stderr、signal、流控 |

工具声明所需能力：

| 工具 | 能力 |
| --- | --- |
| `Read/Write/Edit` | `fs` |
| `Grep/Glob` | `fs` + `process`（ripgrep） |
| `Bash` | `process` |
| `Agent` | `process` |

如果环境缺少能力但实现了 `connect()`，工具仍会列出，调用时失败并给出 `capability_unavailable`。

---

## 3. 声明式远程环境

`[environments]`：

```toml
[environments]
default = "devbox"

[environments.devbox]
type = "ssh"
host = "user@host"
remote_bin = "~/.kimi-code/bin/kimi"
default_cwd = "/home/user/project"

[environments.container]
type = "docker"
container = "dev"
context = "default"
remote_bin = "/usr/local/bin/kimi"
default_cwd = "/workspace"

[environments.custom]
command = "my-launcher"
args = ["--flag"]
env = { FOO = "bar" }
remote_bin = "kimi"
default_cwd = "/srv/app"
```

规则：

- id ≤64 字符，去首尾空白；
- 保留字 `local`、`default`；
- `type` 与 `command` 互斥；
- `default` 必须存在，且必须设置 `defaultCwd`；
- 只从用户配置读取；
- 变更时 `waitUntil` 重载；单个 id 失败只影响该 id。

---

## 4. 启动器

`remote/client/launchers.ts`：

| 类型 | 命令 |
| --- | --- |
| 默认 | `<remoteBin> exec-server --listen stdio` |
| ssh | `ssh -T <host> ... <remoteBin> exec-server --listen stdio`（BatchMode=yes, ConnectTimeout=10, ServerAliveInterval=15） |
| docker | `docker [--context c] exec -i <container> sh -c 'exec <bin> exec-server --listen stdio'` |
| command | `<command> <args...>`；若 args 未包含 `exec-server`，追加 `<remoteBin> exec-server` |

默认 `remoteBin = ~/.kimi-code/bin/kimi`。

---

## 5. 远程协议

`remote/protocol/`：

- NDJSON 帧；
- JSON-RPC 形状，但**没有** `jsonrpc` 字段；
- 方法：

```
initialize, ping,
fs/readFile, fs/writeFile, fs/createDirectory, fs/getMetadata,
fs/canonicalize, fs/readDirectory, fs/walk, fs/remove, fs/rename,
process/start, process/output, process/exited, process/closed,
process/write, process/signal, process/flow
```

`initialize` 返回：

```ts
{
  osKind, osArch, osVersion,
  shellName, shellPath, pathClass,
  homeDir, cwd, tempDir,
  windowsProcessLifecycle?
}
```

最低执行器版本：`2.5.0`。

限制：

- 单次读 1 MiB；
- 整文件 32 MiB；
- 写块 1 MiB；
- 目录 50,000 项；
- walk 8 MiB / 55s；
- 在途调用 256；
- 待处理 64 MiB；
- 单请求 60s 超时（`process/write` 例外，等 stdin 排空）；
- 二进制用 base64。

---

## 6. 路径类与工作区视图

- `pathClass: "posix" | "win32"` 来自目标；
- `EnvironmentWorkspaceView`：
  - 相对路径基于 `workDir` 解析；
  - `shellPathBridge` 处理 Windows Git Bash 的 `/c/...` ↔ `C:\...`；
  - `assertAllowed` 防止逃出工作区。
- 远程 sandbox 不是安全边界，只是"在另一台机器执行"。

---

## 7. 工具如何路由

每个工具操作：

1. `acquireOrWhenReady(env, capabilities)`；
2. 使用 `lease.environment.fs/process`；
3. 记录 generation；
4. 若 generation 变化 → 失败/重试（不会静默切回本地）；
5. `lease.track(proc, sessionId)` 让资源随租约释放。

示例：

- `Read`：`fs.readFile`；
- `Bash`：`process.spawn(shellPath, ["-c", command])`；
- `Grep`：`process.spawn(rg, ...)`；
- `Agent`：需要 `process`。

---

## 8. 绑定生命周期

`EnvironmentBindingService`：

- durable 事件 `environment.set_binding`；
- `current = state ?? seed`；
- `bind()`：先 acquire/release 租约验证，再提交、派发事件、应用 session workDir、发环境提醒；
- 恢复时若状态缺失 → 重新派发；
- AGENTS.md 探测走绑定环境的 `fs`；
- `AgentEnvironmentService` 提供 `inspect/isAvailable/acquire/acquireWhenReady/workspaceRoots`；
- 只对 main agent 发布 `EnvironmentStatusChanged`。

注册表：

- `acquireWhenReady` → 触发 `connect()` / `whenReady()` → `acquire`；
- `acquire` 拒绝未就绪 / 不存在 / 缺能力；
- 租约引用计数；
- `remove` 排空租约（5s 上限）并释放资源。

---

## 9. 失败模式

| 情况 | 错误 |
| --- | --- |
| 未绑定/未就绪 | `environment.unavailable` |
| 未知 id | `environment.not_found` |
| 缺能力 | `capability_unavailable` |
| 连接失败 | 设置 `connectError` + `disconnected`；无自动重试循环、无本地回退 |
| 声明变更 | 丢弃当前会话进程 |
| 在途调用时断连 | 该调用失败，不重试；下次调用重连 |
| 目录 >50k | 失败 |

---

## 10. 限制（官方明确）

- `WebSearch` / `FetchURL` 永远在本机；
- hooks 永远在本机；
- 未声明 `environment_id` 的 stdio MCP 在本机；
- 项目级 agent 文件不从目标加载；
- AGENTS.md / Skills 不热更新（只在 create/resume/`/reload`）；
- prompt 中的路径在目标端解析；
- Tower 在远程工作区不可用；
- `/feedback` 扫描器不可用；
- 同一根路径跨主机共享一个 workspace 条目；
- 不静默回退本地。

---

## 11. 远程控制（不是环境）

`packages/remote-control` 是另一件事：

- 把本机注册到 relay（默认 `https://code-rc.kimi.com`）；
- 通过 WebSocket 转发 HTTP/WS 到本地 KAP server；
- 需要 Kimi 登录 + 本地 bearer token；
- 单实例锁 `~/.kimi-code/server/rc.json`；
- 管理 WS `/v1/remote/create`，HTTP 隧道 `/v1/remote/http`，流 `/v1/remote/stream/<id>`；
- 转发时带 `Authorization: Bearer <server.token>`；
- 限制：请求 104 MiB、响应保留 208 MiB、块 5 MiB、请求 30s、重连 30s、relay 静默 300s。

它**不是**远程执行环境，只是"从外部访问本机 server"。

---

## 12. 不变量

1. 工具不能直接读本机 fs/process；必须走环境租约。
2. 能力缺失时不能假装可用。
3. 断连不能静默回退本地。
4. generation 变化必须让在途操作失败，而不是用旧句柄。
5. 路径解析必须在目标端进行。
6. 远程环境不是沙箱。
7. Web/hooks/MCP 的本地性必须明确。
8. 绑定是 durable 的，恢复后一致。

---

## 13. 重实现最小版

1. 定义 `Environment` + `Lease` + capability。
2. 本地环境实现 `fs/process`。
3. 工具改成 `acquire` + `lease.environment`。
4. 加 `environment.set_binding` 与 workspace view。
5. 加 ssh launcher + NDJSON 协议（先 fs，再 process）。
6. 加 docker/command。
7. 加断连/重连/错误。
8. 最后加远程控制隧道（独立模块）。

测试重点：

- 本地/远程同一工具行为一致；
- 缺 `process` 时 Bash 不可用；
- 相对路径基于 workDir；
- Windows 路径桥；
- 断连后调用失败且不回退；
- 声明变更后旧会话失效；
- 远程不加载项目 agent/skill。

---

## 14. 证据索引

- `packages/agent-core-v2/src/environment/{environment,environmentWorkspaceView,environmentDefaults,environmentRegistry}.ts`
- `packages/agent-core-v2/src/agent/environmentBinding/*`
- `packages/agent-core-v2/src/remote/{declaration,client,protocol}/*`
- `packages/agent-core-v2/src/os/backends/{node-local,remote}/*`
- `packages/remote-control/src/*`
- `docs/en/guides/remote-environment.md`
