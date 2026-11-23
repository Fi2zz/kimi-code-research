# 26 · 二进制与源码一致性核验

> 日期：2026-11-23  
> 对象：`~/.kimi-code/bin/kimi`（v2.7.2）与 `MoonshotAI/kimi-code` 仓库。  
> 结论先说：**二进制对应 tag `@moonshot-ai/kimi-code@2.7.2`，不是当前 main。** 核心权限/工具代码在 tag 与 main 之间没有变化，因此本套文档对二进制仍然成立。

---

## 1. 结论摘要

| 检查项 | 结果 |
| --- | --- |
| 版本 | `2.7.2`（`kimi --version`） |
| 构建目标 | `darwin-arm64`（内嵌 build info） |
| 构建 commit | **空字符串**（未注入） |
| 源码路径匹配 tag 2.7.2 | **1517 / 1517** |
| 源码路径匹配 main | 1510 / 1517（7 个已删除文件） |
| Web 资源哈希校验 | **782 / 782 通过** |
| 原生依赖哈希校验 | **6 / 6 通过** |
| 运行时 worker 哈希校验 | **2 / 2 通过** |
| 权限策略文件 tag vs main | **完全相同** |
| 危险命令分析文件 tag vs main | **完全相同** |
| KAP 运行时版本 | `server_version: 2.7.2` |
| KAP 协议版本 | `1.0.0`（注意：不是 wire 的 1.5） |
| Wire 协议版本 | `1.5`（bundle 内常量） |

**判定**：二进制由 tag 2.7.2 构建；main 比 tag 多 625 个文件变更，但**权限链、危险命令、路径守卫、计划审阅**这些核心语义未变。文档可以放心引用。

---

## 2. 方法

不是"跑一下 strings 就下结论"，而是四层证据：

1. **Mach-O 段定位**：找到 `NODE_SEA` 段与 `__NODE_SEA_BLOB` section。
2. **SEA 资产表解析**：按 `[u64 name_len][name][u64 data_len][data]` 扫描，抽出全部内嵌资产。
3. **哈希校验**：用内嵌 `web/.../manifest.json` 与 `native/.../manifest.json` 逐文件比对 SHA-256。
4. **源码路径比对**：bundle 里保留了 `packages/.../src/...ts` 注释路径，与 tag/main 的文件树做集合比较。
5. **运行时核验**：临时 `KIMI_CODE_HOME` 启动 `kimi web`，读取 `/api/v1/meta`。

---

## 3. 二进制元数据

```bash
$ ~/.kimi-code/bin/kimi --version
2.7.2

$ shasum -a 256 ~/.kimi-code/bin/kimi
f97206f936eda7d446fcb4fb67d8029da0efe7f8b5f2bee404ed6577eaac604c

$ stat -f "%z bytes" ~/.kimi-code/bin/kimi
190995520 bytes
```

Mach-O：

```
segname NODE_SEA
  fileoff 91160576
  filesize 70715021
Section
  sectname __NODE_SEA_BLOB
  segname NODE_SEA
  size 0x000000000437068d   (= 70,715,021)
```

内嵌 build info（从 `main.cjs` 提取）：

```js
KIMI_BUILD_INFO = {
  version: "2.7.2",
  channel: "",
  commit: "",
  buildTarget: "darwin-arm64"
}
```

注意 `commit` 为空：无法从二进制直接反查 commit，只能用源码路径与 tag 比对。

---

## 4. SEA 资产表格式（实测）

blob 前 9 字节：

```
20 da 43 01 1d 00 00 00 01
```

随后是第一条记录：

```
[u64 name_len = 0x58 (88)]
"/Users/runner/work/kimi-code/kimi-code/apps/kimi-code/dist-native/intermediates/main.cjs"
[u64 data_len = 0x0185f7b6 (25,556,918)]
"#!/usr/bin/env node\n// \0rolldown/runtime.js\n..."
```

后续资产按同样模式排列：

```
[u64 name_len][name][u64 data_len][data]
```

这是**扫描式解析**，不依赖完整容器头；只要名字前 8 字节是合理长度、名字可打印、后 8 字节是合理数据长度，就能定位。

> 说明：9 字节头部的各字段语义没有在本次核验中完全反推；不影响资产抽取。

---

## 5. 抽取结果

共 **793** 个资产，解包后 **58,392,540** 字节：

| 类别 | 数量 | 说明 |
| --- | --- | --- |
| Web | 783 | 782 个文件 + `web/darwin-arm64/manifest.json` |
| Native | 9 | `native/.../manifest.json` + 3 个包共 6 个文件 + 2 个 runtime worker |
| 应用主 bundle | 1 | `main.cjs` |
| 合计 | 793 | |

`main.cjs`：

```
size    25,556,918 bytes
lines   553,484
sha256  905fc3a4a19a121886575b7c5efa8d4d5f0ada38b00dab2a5b51964b4c01f77a
```

`main.cjs` 未压缩（含注释与源码路径），因此可以直接做语义比对。

---

## 6. 哈希校验

### 6.1 Web

`web/darwin-arm64/manifest.json` 列出 **782** 个文件，每个带 `sha256`。

结果：

```
listed 782  missing 0  hash-mismatch 0  bytes 30,609,131
```

说明 blob 内的 Web 资源与 manifest **逐字节一致**。

### 6.2 Native

`native/darwin-arm64/manifest.json`：

```json
{
  "version": 2,
  "target": "darwin-arm64",
  "packages": [
    { "name": "@mariozechner/clipboard", "files": [...] },
    { "name": "@mariozechner/clipboard-darwin-arm64", "files": [...] },
    { "name": "@moonshot-ai/pi-tui", "files": [...] }
  ],
  "runtimeFiles": [
    { "key": "minidb-text-build-worker", ... },
    { "key": "kap-search-worker", ... }
  ]
}
```

结果：

```
native package files: ok 6, missing 0, bad 0
runtime: minidb-text-build-worker true (38,461 bytes)
runtime: kap-search-worker true (470,703 bytes)
```

### 6.3 这说明什么

- blob 内部**自洽**：manifest 与实际资产一致。
- 但这是"自洽校验"，不是外部签名；不能证明二进制未被篡改（需要 codesign/notarization 校验）。
- 至少排除了"manifest 与资产不匹配"这类构建错误。

---

## 7. 源码路径比对

`main.cjs` 中保留了 rollup/rolldown 的模块注释路径，例如：

```
// ../../packages/agent-core-v2/src/_base/di/instantiationService.ts
// ../../packages/kap-server/src/routes/meta.ts
```

提取所有形如 `(apps|packages)/.../*.(ts|tsx|vue|mjs|js)` 的路径：

```
bundle source-like paths: 1517
```

与两棵树比较：

| 树 | 存在 | 缺失 |
| --- | --- | --- |
| tag 2.7.2 | **1517 / 1517** | **0** |
| main (`e12cad1`) | 1510 / 1517 | 7 |

main 缺失的 7 个路径：

```
packages/agent-core-v2/src/_base/utils/fs.ts
packages/agent-core-v2/src/tool/executor/hook-composer.ts
packages/core/src/loop/builtin/hooks.ts
packages/core/src/loop/hook/failure.ts
packages/core/src/loop/hook/registry.ts
packages/core/src/loop/hook/run.ts
packages/core/src/tool/hook-composer.ts
```

这些是 tag 之后被移动/删除的文件。**tag 2.7.2 全命中**，这是"二进制 == tag"的最强证据。

---

## 8. tag 与 main 的差异

```bash
$ git diff --stat v2.7.2 HEAD | tail -1
625 files changed, 80518 insertions(+), 8148 deletions(-)
```

tag：

```
@moonshot-ai/kimi-code@2.7.2
  tag object 1781dbd74bc7fabf23ac4721a570397f278817f1
  commit     56145f849caa305d42acd09b67fb65d11146f887
```

但核心文件无变化：

| 文件 | tag vs main |
| --- | --- |
| `agent/permissionPolicy/permissionPolicyService.ts` | same |
| `agent/permissionPolicy/policies/dangerous-command-ask.ts` | same |
| `agent/permissionPolicy/policies/default-tool-approve.ts` | same |
| `tool/path-access.ts` | same |
| `features/plan/exitPlanModeReview.ts` | same |
| `packages/tree-sitter-bash/src/parse.ts` | same |

权限/工具目录的 diff 只有 2 个文件（`tool/executor/hook-composer.ts` 被删除、`index.ts` 少一行导出）：

```
packages/agent-core-v2/src/tool/executor/hook-composer.ts | 1 -
packages/agent-core-v2/src/tool/executor/index.ts         | 1 -
```

结论：

- 本文档基于 main，但**权限、危险命令、路径守卫、计划审阅**的语义与二进制一致。
- 其它子系统（TUI、会话、MCP 等）可能存在 2.7.2 之后的增量；引用时以 tag 为准更稳。

---

## 9. 运行时核验（KAP）

用临时 home 启动，避免污染用户配置：

```bash
H=/tmp/kimi-verify
KIMI_CODE_HOME=$H ~/.kimi-code/bin/kimi web --no-open --json --port 59999
```

输出：

```json
{"origin":"http://127.0.0.1:59999","ws_url":"ws://127.0.0.1:59999/api/v1/ws","token":"...","kimi_agent_protocol_version":"1.0.0"}
```

`GET /api/v1/meta`：

```json
{
  "kimi_agent_protocol_version": "1.0.0",
  "server_version": "2.7.2",
  "product": "kimi-code-cli",
  "capabilities": {
    "websocket": true, "file_upload": true, "fs_query": true,
    "mcp": true, "tasks": true, "interactions_list": true, "session_wait": true
  },
  "dangerous_bypass_auth": false,
  "experimental_flags": {
    "notify_user": false, "wait_for": true, "prompt_prediction": false,
    "subagent_fork": false, "tower": false, "tool-select": false
  }
}
```

`GET /api/v1/healthz` → `{ok:true}`。
`GET /api/v1/tools` → `[]`（没有活动会话，符合预期）。

### 重要发现：两个协议版本

| 协议 | 版本 | 用途 |
| --- | --- | --- |
| KAP（REST/WS） | **1.0.0** | Web/桌面与 server 通信 |
| Wire（`wire.jsonl`） | **1.5** | 会话事件日志 |

写文档/实现时不要混用。`kimi_agent_protocol_version` 是 KAP，`protocol_version`（wire metadata）是 1.5。

---

## 10. 关键语义在 bundle 中的存在性

`main.cjs` 中可直接搜到：

| 标记 | 次数/存在 |
| --- | --- |
| `git-cwd-write-approve` | 2 |
| `auto-mode-ask-user-question-deny` | 2 |
| `dangerous_command` | 3 |
| `unanalyzable_command` | 1 |
| `PATH_SENSITIVE` | 1 |
| `Try a different approach` | 2 |
| `Context compaction is complete` | 1 |
| `select_tools` | 22 |
| `mcp__` | 13 |
| `tools_added` | 7 |
| `max_steps_per_turn` | 4 |
| `WIRE_PROTOCOL_VERSION` | 4 |
| `KIMI_CODE_DANGEROUS_COMMAND_GUARD` | 1 |
| `"1.5"` | 存在 |
| `2.7.2` | 2 |

工具名字面量：

```
Read 26, Write 24, Edit 21, Grep 16, Glob 18, Bash 36,
Agent 22, AgentSwarm 20, AskUserQuestion 11, Skill 12,
TaskList 11, TaskOutput 12, TaskStop 12, WaitFor 16,
CronCreate 6, CronDelete 6
```

这证明：bundle 不是"简化版"，权限、工具、动态加载、压缩、wire 都在。

---

## 11. 构建路径泄漏

资产名中包含：

```
/Users/runner/work/kimi-code/kimi-code/apps/kimi-code/dist-native/intermediates/main.cjs
```

含义：

- 由 GitHub Actions（`runner` 用户）构建；
- 工作目录是 `kimi-code/kimi-code`；
- `main.cjs` 是构建中间产物，被直接嵌入。

这本身不是漏洞，但暴露了 CI 路径；重实现时建议用相对路径或虚拟路径，避免把构建机信息带进产物。

---

## 12. 复现脚本

```bash
# 1. 定位 SEA 段
otool -l ~/.kimi-code/bin/kimi | grep -A8 NODE_SEA

# 2. 抽出 blob（offset/size 来自上一步）
dd if=~/.kimi-code/bin/kimi of=/tmp/kimi.blob bs=1 skip=91160576 count=70715021

# 3. 扫描资产（Python）
python3 - <<'PY'
import struct, re, os, hashlib, json
data=open('/tmp/kimi.blob','rb').read()
def u64(i): return struct.unpack_from('<Q',data,i)[0]
found={}
for pat in [b'web/darwin-arm64/', b'native/darwin-arm64/', b'/Users/runner/work/kimi-code/']:
    for m in re.finditer(re.escape(pat), data):
        i=m.start()
        if i<8: continue
        nlen=u64(i-8)
        if nlen<8 or nlen>512: continue
        name=data[i:i+nlen]
        if len(name)!=nlen or any(b<32 or b>126 for b in name): continue
        j=i+nlen
        dlen=u64(j)
        if j+8+dlen>len(data): continue
        found[name.decode()]=(j+8,dlen)
print('assets',len(found))
PY

# 4. 校验 web manifest
#    （按 §6 逻辑逐文件 sha256 比对）

# 5. 比对源码路径
#    （提取 main.cjs 中的 packages/.../src/... 与 tag 文件树求交）
```

```bash
# 6. 取 tag
git fetch --depth 1 origin 'refs/tags/@moonshot-ai/kimi-code@2.7.2:refs/tags/v2.7.2'
git archive v2.7.2 | tar -x -C /tmp/kimi-2.7.2

# 7. 运行时
KIMI_CODE_HOME=/tmp/kimi-verify ~/.kimi-code/bin/kimi web --no-open --json --port 59999
curl -H "Authorization: Bearer <token>" http://127.0.0.1:59999/api/v1/meta
```

---

## 13. 本报告没有证明的事

1. **没有做可重复构建**：没有在相同 Node/依赖下重新构建并比对 `main.cjs` 哈希。
2. **没有校验代码签名/公证**：只读了段与资产，没有 `codesign`/`spctl` 验证。
3. **没有反编译 V8 代码缓存**：`useCodeCache` 只在本机 target 启用；本次未解析 code cache。
4. **没有验证所有系统调用行为**：运行时只验了 KAP 元数据，没有逐工具执行。
5. **manifest 哈希是 blob 内自洽**，不是外部信任根。

这些不影响"文档语义与二进制一致"的结论，但影响"二进制绝对可信"的结论。

---

## 14. 对复刻的意义

1. 以 **tag 2.7.2** 为行为基线，而不是 main。
2. 文档中引用的文件路径如果来自 main，先确认它在 tag 存在；本次核心权限/工具文件都在。
3. 复刻时把 **KAP 1.0.0** 与 **wire 1.5** 分开实现。
4. 资源打包要复刻 manifest + SHA-256 校验；这是可验证的完整性层。
5. 不要把构建机绝对路径写进资产 key。

---

## 15. 证据文件

本次抽取的资产与索引在：

```
$TMPDIR/kimi-sea/
├── assets/                 # 793 个解包资产
└── index.json              # 首轮解析索引（仅 main.cjs）
```

（临时目录，重启后可能被清理；复现脚本见 §12。）
