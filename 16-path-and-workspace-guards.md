# 16 · 路径、敏感文件与工作区信任

> 目标：解释文件工具的路径解析、工作区边界、敏感文件硬阻断、Git 控制路径与工作区信任。重实现时，这一层决定"模型能不能碰到不该碰的文件"。

---

## 1. 工作区访问策略

`tool/path-access.ts`：

```ts
interface WorkspaceAccessPolicy {
  guardMode: "absolute-outside-allowed" | "disabled";
  checkSensitive: boolean;
}

const DEFAULT_WORKSPACE_ACCESS_POLICY = {
  guardMode: "absolute-outside-allowed",
  checkSensitive: true,
};
```

含义：

- **工作区内**：允许；
- **工作区外**：
  - 相对路径 → 拒绝（`PATH_OUTSIDE_WORKSPACE`）；
  - 绝对路径 → 允许解析，但会走到权限策略的 `ask`；
- `checkSensitive: true` → 敏感文件**直接抛错**，不是 ask。

这不是"禁止访问工作区外"，而是"必须给绝对路径并接受审批"。

---

## 2. 路径规范化

```ts
normalizeUserPath(path, pathClass)      // win32 处理盘符
canonicalizePath(path, cwd, pathClass)  // 展开 ~、解析相对、normalize
isWithinDirectory(candidate, base, pathClass)
isWithinWorkspace(candidate, {workspaceDir, additionalDirs}, pathClass)
```

规则：

- 空路径 → `PATH_INVALID`；
- Windows 盘符相对路径（`C:foo`）→ `PATH_INVALID`；
- 相对路径 + 非绝对 cwd → `PATH_INVALID`；
- `~` 展开为 home；
- Windows 大小写不敏感比较；
- 工作区 = 主工作目录 + `--add-dir` + skill roots。

---

## 3. 敏感文件

```ts
SENSITIVE_BASENAMES = [".env", "id_rsa", "id_ed25519", "id_ecdsa", "credentials"]

SENSITIVE_PATH_SUFFIXES = [".aws/credentials", ".gcp/credentials"]

ENV_PREFIX = ".env."
ENV_EXEMPTIONS = [".env.example", ".env.sample", ".env.template"]

SENSITIVE_BASENAME_PREFIXES = ["id_rsa", "id_ed25519", "id_ecdsa", "credentials"]

PUBLIC_KEY_BASENAMES = ["id_rsa.pub", "id_ed25519.pub", "id_ecdsa.pub"]

SENSITIVE_DOT_VARIANT_SUFFIXES = [
  ".bak", ".backup", ".copy", ".disabled", ".key", ".old",
  ".orig", ".pem", ".save", ".tmp",
]
```

判定：

1. 豁免 `.env.example` 等、公钥；
2. 精确名命中；
3. `.env.*` 命中；
4. `id_rsa-*` / `id_rsa_*` / `id_rsa.<variant>` 命中；
5. 路径包含 `.aws/credentials` 或 `.gcp/credentials` 命中；
6. 大小写不敏感。

### 硬阻断

`resolvePathAccess`：

```ts
if (policy.checkSensitive && isSensitiveFile(canonical)) {
  throw new PathSecurityError("PATH_SENSITIVE", ...);
}
```

发生在**权限策略之前**，所以：

- 用户 `allow Read(.env)` 也读不到；
- `auto` 模式也读不到；
- 只有 `checkSensitive: false` 的工具能绕过。

哪些工具 `checkSensitive: false`？

- `Grep`
- `Glob`

它们在搜索时**过滤结果**，不返回敏感文件内容。`Bash` 不受这个守卫约束（见 §8）。

---

## 4. 读 / 写 / 搜索的差异

| 工具 | 操作 | 敏感检查 | 工作区外 |
| --- | --- | --- | --- |
| `Read` | read | 硬阻断 | 绝对路径 + ask |
| `Write` | write | 硬阻断 | 绝对路径 + ask |
| `Edit` | readwrite | 硬阻断 | 绝对路径 + ask |
| `Grep` | search | 结果过滤 | 允许（绝对路径） |
| `Glob` | search | 结果过滤 | 允许（绝对路径） |
| `Bash` | 任意 | **无** | 无 |

---

## 5. Git 控制路径

`git-control-path-access-ask`：

- Write/Edit 目标是 `.git/` 下（config、hooks 等）→ ask；
- 排在 `git-cwd-write-approve` 之前，所以 git 工作区自动放行**不覆盖** `.git`。

`git-cwd-write-approve`：

- 仅 `Write`/`Edit`；
- 仅 POSIX 路径类；
- 仅工作区内（含 additionalDirs）；
- cwd 必须在 git worktree 内；
- → approve。

这解释了两个看似矛盾的行为：

- 在 git 仓库里改代码不弹窗；
- 改 `.git/config` 会弹窗。

---

## 6. 工作区信任

`workspace.require_trust`（默认 true）：

- 第一次打开未信任工作区 → 弹信任确认；
- 信任后，项目级配置（`.kimi-code/local.toml`、项目 MCP、项目 skill/agent）才加载；
- `KIMI_CODE_TRUST_WORKSPACE=1` 可作为单进程覆盖；
- `require_trust = false` → 所有工作区视为信任（危险，文档明确警告）。

信任的边界：

- **覆盖**：项目 MCP、项目权限规则、项目 skill/agent、项目 hooks（如果有）。
- **不覆盖**：插件（插件是用户级安装，不看工作区信任）。

---

## 7. 远程环境

- 路径解析在目标环境执行（`EnvironmentWorkspaceView`）；
- `pathClass` 来自目标（posix/win32）；
- `shellPathBridge` 处理 Windows Git Bash 路径；
- 工作区外/敏感文件规则同样适用；
- 项目权限规则**远端不支持**；
- 项目级 agent/skill 不从目标加载（仅本地项目文件）；
- hooks 永远在本地执行。

---

## 8. 已知边界与风险

1. **Bash 是后门**：`cat .env`、`cp ~/.ssh/id_rsa /tmp` 不受文件守卫约束。
2. **Grep/Glob 只过滤结果**：如果搜索模式本身泄漏路径/内容，仍可能暴露。
3. **符号链接**：`canonicalizePath` 做的是字面规范化，是否解析 symlink 取决于底层 fs；重实现时建议 realpath 后再判定。
4. **大小写**：macOS 默认大小写不敏感，但代码只在 win32 做大小写归一；`.ENV` 在 macOS 可能绕过（`isSensitiveFile` 自身做了 lowercase，所以文件名层面没问题；路径后缀也 lowercase）。
5. **工作区外绝对路径**：允许，只是要审批；不要把"ask"当成"禁止"。
6. **信任模式**：`require_trust=false` 会让项目配置无确认加载。
7. **`additionalDirs`**：显式加入的目录享受工作区同等待遇；不要误以为"只有 cwd"。
8. **`/tmp` 例外只用于危险命令**，不影响路径守卫。

---

## 9. 不变量

1. 敏感文件在任何权限模式下都硬阻断（文件工具）。
2. 工作区外相对路径必须拒绝。
3. 工作区外绝对路径必须进入审批。
4. `.git` 控制路径必须审批，优先于 git 工作区自动放行。
5. 项目配置只在信任的本地工作区加载。
6. 路径比较必须考虑平台大小写规则。
7. `--add-dir` 与 skill roots 必须计入工作区。
8. 远程环境的路径解析必须在目标端进行。

---

## 10. 重实现最小版

1. 实现 `canonicalizePath` + `isWithinWorkspace`。
2. 实现敏感文件列表与硬阻断。
3. Read/Write/Edit 走 `resolvePathAccess`。
4. Grep/Glob 关闭敏感检查但过滤结果。
5. 加 `.git` 控制路径 ask。
6. 加 git 工作区写放行。
7. 加工作区信任门。
8. 加 `--add-dir`/skill roots。

测试重点：

- `.env` 读写被硬阻断（即使 allow）；
- `.env.example` 放行；
- `id_rsa.pub` 放行；
- `../x` 拒绝；
- `/abs/x` 进入 ask；
- `.git/config` ask；
- git 内 `src/a.ts` Write 放行；
- Grep 不返回 `.env`；
- 远程路径解析正确。

---

## 11. 证据索引

- `packages/agent-core-v2/src/tool/path-access.ts`
- `packages/agent-core-v2/src/tool/executor/*`
- `packages/agent-core-v2/src/agent/permissionPolicy/policies/{sensitive-file-access-ask,git-control-path-access-ask,git-cwd-write-approve,path-utils}.ts`
- `packages/agent-core-v2/src/agent/tools/os/{read,write,grep,glob}/*`
- `packages/agent-core-v2/src/workspace/workspaceTrust/*`
- `docs/en/configuration/config-files.md#workspace-trust`
