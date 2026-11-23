# 24 · 安全模型与威胁分析

> 目标：把 Kimi Code 的安全控制、真实边界、攻击面与建议讲透。重实现时，最重要的不是"再加一个正则"，而是**明确哪些是策略、哪些是硬阻断、哪些根本没有**。

---

## 1. 资产

| 资产 | 位置 |
| --- | --- |
| 源代码/工作区 | 用户目录 |
| 凭据 | `~/.kimi-code/credentials/`、OAuth token、MCP token |
| 会话历史 | `wire.jsonl`（可能含代码片段、密钥误贴） |
| 服务器 token | `server/server.token` |
| 用户机器 | 任意命令执行能力 |
| 远端环境 | ssh/docker 目标 |

---

## 2. 信任边界

```
用户
 │ 信任
 ▼
Kimi 进程 ──► 模型 API（不可信内容）
 │
 ├─► 工具（内置/MCP/插件）
 ├─► 本地文件系统
 ├─► 本地/远程 shell
 └─► hooks（本地脚本）
```

**模型输出、MCP 结果、网页内容、仓库文件都是不可信输入**。它们可以诱导模型调用工具。

---

## 3. 现有控制

| 控制 | 类型 | 覆盖 |
| --- | --- | --- |
| 三级权限模式 | 策略 | 工具调用 |
| 13 条策略链 | 策略 | 工具调用 |
| 用户/项目规则 | 策略 | 工具调用 |
| 审批 + 会话缓存 | 人工 | 工具调用 |
| 危险命令分析 | 启发式 | `Bash` |
| 敏感文件硬阻断 | **强制** | Read/Write/Edit |
| 工作区路径守卫 | 策略 | 文件工具 |
| `.git` 控制路径 | 策略 | Write/Edit |
| 工作区信任 | 策略 | 项目配置/MCP/skill/agent |
| Hooks block | 策略 | 工具调用 |
| KAP bearer + host/origin | 强制 | Web/WS |
| 环境租约/能力 | 架构 | 工具执行 |
| 无 OS 沙箱 | **缺失** | — |

---

## 4. 真实边界（按风险排序）

### 4.1 没有 OS 级沙箱

源码中搜不到 seatbelt / bubblewrap / seccomp / landlock / apparmor。
所有限制都是**应用层策略**。一旦 `Bash` 被批准，模型可以：

- 读写当前用户可访问的任何文件；
- 发起网络请求；
- 修改 shell 配置、植入持久化；
- 读取环境变量中的密钥。

远程环境也不是沙箱，只是"在另一台机器执行"。

**结论**：这是"助手"模型，不是"不可信代码执行"模型。重实现若要跑不可信任务，必须加沙箱（容器/VM/seatbelt），不能靠权限链。

### 4.2 Bash 绕过文件守卫

- `Read/Write/Edit` 对 `.env`、私钥硬阻断；
- `Grep/Glob` 过滤结果；
- **`Bash` 不受同一守卫约束**：`cat .env`、`cp ~/.ssh/id_rsa /tmp` 可以。
- 危险命令分析只覆盖一小撮破坏性命令，不覆盖数据外泄。

**攻击场景**：恶意仓库的 `AGENTS.md` 指示模型"运行 `./setup.sh` 并读取 `.env` 验证配置"。

**缓解**：沙箱、网络出口控制、把敏感文件放到 agent 用户不可读的位置、禁止 Bash 读 home。

### 4.3 manual 模式在 git 工作区自动批准编辑

`git-cwd-write-approve` 让 `Write/Edit` 在 git worktree 内不弹窗。理由是可撤销 + 可 diff。但这意味着：

- 用户以为"Always Ask 每个编辑都问"，实际不是；
- 恶意 prompt 可以让模型静默改代码（仍有 file history）。

**缓解**：重实现时把这条做成可配置，默认关闭；或只在"已信任 + 干净工作区"时启用。

### 4.4 yolo / auto 的宽松

- `yolo`：普通工具全放行，只拦危险命令/敏感文件/git 控制路径；
- `auto`：连这些都不拦（危险分析根本不跑），且拒绝 `AskUserQuestion`；
- `kimi -p` + yolo = 完全无人值守。

**攻击场景**：CI 里跑 `kimi --auto -p "fix the build"`，模型可以执行任意命令、读写任意文件。

**缓解**：CI 用容器 + 只读挂载 + 网络白名单；不要把 `auto` 当默认。

### 4.5 项目 allow 规则可压制内置守卫

`.kimi-code/local.toml` 的 project `allow` 可以压制 sensitive-file / git-control 的 `ask`（但硬阻断仍在）。工作区信任后自动加载。

**攻击场景**：恶意仓库带 `.kimi-code/local.toml`，把 `Bash` allow 掉，减少弹窗。

**缓解**：信任前审查项目配置；重实现时对项目 allow 做更严格的 UI 提示。

### 4.6 hooks fail-open + 本地执行

- hook 崩溃/超时 → 放行；
- hook 不能把"会被拒绝的调用"变成允许；
- 但 hook 可以**额外阻止**，也可以读取 stdin 里的工具参数（可能含密钥）。
- 没有项目级 hook（好）。

**风险**：用户 hook 脚本本身可能不安全；插件 hook 在安装时被信任。

### 4.7 插件/MCP/网页的提示注入

- MCP 工具返回的文本、网页内容、仓库文件都可能包含指令；
- 插件可注入最多 32KB×N（总 64KB）系统提示；
- 系统提示里明确说插件指令"不是特权通道"，但模型仍可能被诱导。

**缓解**：工具结果标注不可信；对高危工具（Bash/Write）保持审批；不要让 `auto` 跑不可信内容。

### 4.8 凭据与日志

- 凭据文件 `0600`，目录 `0700`；
- 但 `wire.jsonl` 会保存对话，可能含用户粘贴的密钥；
- 日志可能含路径/命令；
- 遥测有脱敏（URL/email/token/路径），但旧版遥测路径没有字符串脱敏。

**缓解**：导出/分享前审查 wire；不要把 `.kimi-code` 提交到仓库；日志级别默认 info。

### 4.9 KAP 暴露面

- 默认 loopback；
- 非 loopback 无 TLS 拒绝（除非 `--insecure-no-tls`）；
- bearer token + host/origin 检查；
- `--dangerous-bypass-auth` 会关闭所有鉴权；
- debug endpoints 仅 loopback。

**风险**：用户为了远程访问开 `--insecure-no-tls` + `0.0.0.0`，等于把 agent 暴露给局域网。

### 4.10 远程环境信任

- ssh/docker 目标上的文件/命令被视为可信执行环境；
- hooks/Web/MCP 仍在本机；
- 项目级 agent/skill 不从目标加载（降低注入面）；
- 但目标上的仓库内容仍可提示注入。

---

## 5. 攻击树（简化）

```
诱导模型
├─ 读敏感文件
│  ├─ Read → 硬阻断
│  ├─ Grep/Glob → 结果过滤
│  └─ Bash cat → 可绕过 ★
├─ 外泄数据
│  ├─ FetchURL/WebSearch → 若敏感值被放进 URL/查询词，可能随请求外泄
│  └─ Bash curl → 可绕过 ★
├─ 执行恶意代码
│  ├─ Bash ./script.sh → 未识别命令，判为 safe；manual 仍会落到 fallback ask，yolo 直接放行 ★
│  └─ npm install → 不受限 ★
├─ 持久化
│  ├─ 改 ~/.zshrc → Bash 可
│  └─ 改 .git/hooks → 需审批（manual/yolo）
└─ 破坏
   ├─ rm -rf / → 危险分析拦截（manual/yolo）
   └─ 其它破坏命令 → 可能漏 ★
```

★ = 现有控制的真实缺口。

---

## 6. 重实现建议

### 6.1 必须做

1. **明确产品定位**：助手 vs 不可信代码执行器。后者必须沙箱。
2. **保留敏感文件硬阻断**，并扩展到 Bash（至少检测重定向/读取常见密钥路径）。
3. **权限链顺序固定**，deny 永远最先。
4. **所有工具声明 accesses**，并发/审计都依赖它。
5. **凭据 0600、目录 0700、原子写**。
6. **KAP 默认 loopback + token + host/origin**。
7. **hooks fail-open 但不可提权**。
8. **审计日志**：记录每次工具调用、审批、策略命中。

### 6.2 建议做

1. 沙箱（容器/seatbelt/bwrap）作为环境的一种能力，而不是权限模式。
2. 网络出口策略（域名白名单/代理）。
3. 对 `Bash` 做"读敏感路径"检测（`cat .env`、`grep -r AWS_ ~/.aws`）。
4. 文件操作 realpath 后再判定，防 symlink。
5. `git-cwd-write-approve` 默认关闭或可配置。
6. 项目 `allow` 规则需要显式确认。
7. 导出/分享前扫描 wire 中的密钥模式。
8. MCP/网页内容打不可信标记，并在系统提示中强调。

### 6.3 不要做

1. 不要用正则黑名单冒充沙箱。
2. 不要让 `auto` 成为默认。
3. 不要把"ask"当成"禁止"。
4. 不要在日志/遥测里记录 token。
5. 不要信任仓库里的 hook/配置（项目 hook 已禁止，保持）。

---

## 7. 安全测试矩阵

| 场景 | 期望 |
| --- | --- |
| Read `.env` | 硬阻断 |
| Read `.env.example` | 允许 |
| Grep `AWS_SECRET` | 不返回 `.env` |
| Bash `cat .env` | **当前可**；重实现应拦截或沙箱 |
| `rm -rf /`（manual） | ask |
| `rm -rf /tmp/x` | 允许 |
| `sudo sh -c 'rm -rf /'` | ask |
| `bash -c "$CMD"` | unanalyzable → ask |
| `./script.sh` | 分析器判 safe；manual 由 fallback 策略 ask，yolo 放行 |
| 写 `.git/config` | ask |
| 写 `src/a.ts`（git 内） | manual 自动放行（需评审） |
| 写工作区外相对路径 | 拒绝 |
| 写工作区外绝对路径 | ask |
| 项目 allow 压制敏感 ask | 不应影响硬阻断 |
| hook 退出 2 | block |
| hook 崩溃 | 放行 |
| 无 token 访问 KAP | 401 |
| 错误 Host | 403 |
| 非 loopback 无 TLS | 拒绝 |
| 插件路径 `../` | 拒绝 |
| MCP 结果含指令 | 不执行（模型层仍可能被诱导） |

---

## 8. 证据索引

- `agent/permissionPolicy/policies/*`
- `tool/path-access.ts`
- `agent/tools/os/{bash,read,write,grep,glob}/*`
- `features/externalHooks/*`
- `workspace/workspaceTrust/*`
- `kap-server/src/{middleware,security,transport}/**`
- `app/plugin/manifest.ts`
- `packages/telemetry/src/privacy.ts`
- `docs/en/configuration/config-files.md`（安全警告段落）
