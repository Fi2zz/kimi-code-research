# 15 · 危险命令分析（纯 TS tree-sitter-bash）

> 目标：解释 Kimi Code 如何判断一条 shell 命令"危险到需要人工确认"，包括解析器、包装器递归、判定规则、失败策略与已知盲区。这是它区别于"正则黑名单"的核心技术点。

---

## 1. 为什么不用正则

需要识别的不是字符串，而是**执行语义**：

```bash
sudo env FOO=1 bash -c 'rm -rf /'
command rm -rf /
eval "$(echo cm0gLXJmIC8= | base64 -d)"
busybox rm -rf /
```

正则很容易漏掉包装、引号、变量拼接。Kimi 的做法是：**解析成语法树 → 找到真正要执行的命令 → 递归拆包装**。

---

## 2. 解析器：`@moonshot-ai/tree-sitter-bash`

`packages/tree-sitter-bash/`：

- 纯 TypeScript，**无原生/WASM 依赖**；
- 产出的命名节点类型与官方 `tree-sitter-bash` 0.25.0 一一对应；
- 用官方语料做差分测试，逐字节比较语法树；
- offset 是 UTF-16 code unit；`node.text` 就是 `source.slice(start,end)`；
- 解析**永不抛异常**：
  - 预算耗尽 → `{ ok: false, reason: "aborted" }`；
  - 语法错误 → 降级树 + `hasError: true`。

API：

```ts
parse(source, { timeoutMs?, maxNodes? }): ParseResult

type ParseResult =
  | { ok: true; rootNode: SyntaxNode; hasError: boolean }
  | { ok: false; reason: "aborted" };

interface SyntaxNode {
  type: string;
  text: string;
  startIndex: number;
  endIndex: number;
  isNamed: boolean;
  parent?: SyntaxNode;
  children: SyntaxNode[];        // 含匿名标点/关键字
  namedChildren: SyntaxNode[];
}
```

默认预算：50ms / 50,000 节点。危险分析用 500ms / 10,000 节点。

深度上限：

```
MAX_SUBSTITUTION_DEPTH = 150   // $(...) / `...` / <(...)
MAX_PARSE_DEPTH = 500          // subshell/compound/${...}/expression
MAX_SCAN_DEPTH = 1024          // lexer
```

超过时局部降级（ERROR 节点 / `hasError`），不爆栈。

---

## 3. 分析入口

`agent/permissionPolicy/policies/dangerous-command-ask.ts`：

```ts
evaluate(context) {
  if (!guardEnabled) return undefined;
  if (mode === "auto") return undefined;
  if (toolCall.name !== "Bash") return undefined;

  const command = bashCommandText(context.args);
  const verdict = command === undefined
    ? { kind: "unanalyzable" }
    : analyzeSource(command, 0, s => bashParser.parse(s, PARSE_OPTIONS));

  if (!verdict) return undefined;
  if (verdict.kind === "dangerous") return { kind: "ask", reason: { dangerous_command: verdict.command } };
  if (mode === "yolo") return undefined;
  return { kind: "ask", reason: { unanalyzable_command: true } };
}
```

关键语义：

| 情况 | manual | yolo | auto |
| --- | --- | --- | --- |
| 明确危险 | ask | ask | 放行（策略不运行） |
| 无法分析 | ask | **放行** | 放行 |
| 安全 | 放行 | 放行 | 放行 |

"无法分析"在 yolo 下放行，是刻意的取舍：避免每条形如 `./script.sh` 的命令都弹窗。

---

## 4. 词法提取：literalText

只把"字面量"当作可分析参数：

| 节点 | 规则 |
| --- | --- |
| `word` | 若包含 `$`、`` ` ``、`*`、`?`、`[`、`]`、`~` → 不可分析；否则反转义后返回 |
| `number` | 直接返回 |
| `raw_string` | 去引号；含不安全字符 → 不可分析 |
| `string` | 拼接 `string_content`；含命名子节点（变量等）→ 不可分析 |

`UNSAFE_OPERAND = /[$`*?[\]~]/`。

只要有一个参数"丢失"（`dropped = true`），后续判定就不能当作安全，只能返回 `unanalyzable` 或保守危险。

---

## 5. 递归拆包装

`analyzeInvocation(name, args, dropped, depth, parse)`：

### 5.1 提权包装

```
sudo, doas
```

跳过值选项：

```
-u/--user, -g/--group, -h/--host, -p/--prompt, -C/--close-from,
-T/--command-timeout, -U/--other-user, -r/--role, -t/--type
```

然后对内层命令继续分析。

### 5.2 启动包装

```
env, command, exec, nohup, builtin, nice
```

- 值选项：`-u/--unset, -C/--chdir, -S/--split-string, -a, -n, --adjustment`；
- `env` 额外跳过 `NAME=value`；
- `command` 遇 `-v`/`-V` 视为只读，直接放行。

### 5.3 嵌套 shell

```
sh, bash, dash, zsh, ksh, ash
```

- 找 `-c` 后的 payload；
- 只看第一个非选项参数；
- 递归解析 payload；
- 深度 ≥4 → `unanalyzable`。

### 5.4 eval

- 无参数 → 无危险；
- 有参数但 `dropped` 或深度 ≥4 → `unanalyzable`；
- 否则把参数拼成源码递归解析。

### 5.5 busybox

- 第一个非选项参数是 applet；
- 以 applet 名继续分析其余参数。

---

## 6. 危险判定规则

### 6.1 固定危险命令

```
shutdown, halt, poweroff, reboot,
bcdedit, diskpart, format,
restart-computer, stop-computer,
mkfs, wipefs
```

`mkfs.*` 也命中。

### 6.2 init / telinit

参数含 `0` 或 `6` → 危险。

### 6.3 systemctl

跳过值选项（`-H/--host`, `-M/--machine`）后，子命令为：

```
poweroff, reboot, halt, kexec
```

→ 危险。

### 6.4 dd

任一 `of=/dev/...` 且不在安全设备列表：

```
/dev/null /dev/zero /dev/full /dev/random /dev/urandom
/dev/stdin /dev/stdout /dev/stderr
```

→ 危险。

### 6.5 rm

解析选项：

- `--recursive` / `-r` / `-R` → recursive；
- `--force` / `-f` → force；
- `--` 之后全是操作数。

若 recursive && force：

- 无 `dropped`、操作数非空、且每个操作数都在 `/tmp` 或 `/temp` 下（且不含 `..`）→ **安全**；
- 否则 → 危险 `rm -rf`。

这是唯一的"危险命令白名单"：`rm -rf /tmp/build` 不弹窗。

### 6.6 其它

未识别的命令 → 安全（返回 undefined）；但若 `dropped` → `unanalyzable`。

---

## 7. 命令名归一化

```
normalizeCommandName(raw):
  1. 去掉最后一个 / 或 \ 之前的部分（路径）
  2. 转小写
  3. 去掉 .exe 后缀
```

因此 `/usr/bin/RM` → `rm`，`RM.EXE` → `rm`。

---

## 8. 失败策略

| 情况 | 结果 |
| --- | --- |
| 解析失败/超预算 | `unanalyzable` |
| 有 `hasError` | `unanalyzable` |
| 参数含变量/通配 | 该参数丢失；若影响判定 → `unanalyzable` |
| 嵌套深度 ≥4 | `unanalyzable` |

`unanalyzable` 在 manual 下 → ask；yolo 下 → 放行；auto 下策略不运行。

---

## 9. 测试向量（建议直接做成用例）

**应判定危险：**

```bash
rm -rf /
rm -rf ~
rm -fr /home
sudo rm -rf /
sudo -u root rm -rf /
env FOO=1 rm -rf /
nohup rm -rf /
bash -c 'rm -rf /'
sh -c "sudo rm -rf /"
eval 'rm -rf /'
busybox rm -rf /
dd if=/dev/zero of=/dev/sda
systemctl reboot
systemctl poweroff
shutdown -h now
reboot
mkfs.ext4 /dev/sda1
wipefs -a /dev/sda
init 0
```

**应判定安全/放行：**

```bash
rm -rf /tmp/build
rm -rf /temp/cache
rm file.txt
rm -r dir
ls -la
cat /etc/hosts
dd if=/dev/zero of=/dev/null
command -v git
echo "rm -rf /"
```

**应判定不可分析（manual 下 ask，yolo 下放行）：**

```bash
bash -c "$CMD"
eval "$(cat script.sh)"
./deploy.sh
```

注意：`rm -rf $DIR` **不是**不可分析。`$DIR` 的字面量提取失败会让 `dropped=true`，但 `rm` 分支在 recursive+force 时仍直接判 `dangerous`（保守设计）。

---

## 10. 已知盲区

这是"启发式"而不是"沙箱"：

1. **只分析 Bash 工具**，不分析其它工具内部执行的命令。
2. **只识别固定集合**：`chmod -R 000 /`、`> /dev/sda`、`find / -delete`、`git clean -fdx` 等不在列表。
3. **脚本内容不可见**：`./script.sh` 只能判为不可分析。
4. **变量/命令替换**：保守地判为不可分析，yolo 下放行。
5. **编码/混淆**：`base64 -d | sh` 不会被展开。
6. **别名/函数**：不解析 shell 启动文件。
7. **非交互模式**：这条策略根本不注册。
8. **auto 模式**：完全不运行。
9. **平台差异**：主要覆盖 POSIX；Windows 的 `format`/`diskpart` 只有名字匹配。

结论：它降低误操作概率，但**不能替代沙箱/备份/代码审查**。

---

## 11. 重实现最小版

1. 先做正则黑名单（`rm -rf`、`shutdown`、`mkfs`、`dd of=/dev/`）——能覆盖 80%。
2. 引入 bash 解析器（可用官方 tree-sitter-bash WASM，或照 Kimi 做纯 TS）。
3. 实现命令收集 + 字面量提取。
4. 实现 `sudo/env/sh -c/eval` 递归。
5. 实现 `rm/dd/systemctl/init/mkfs` 规则。
6. 做 fail-closed 策略（manual ask / yolo 放行）。
7. 用第 9 节向量做测试。

如果你不做纯 TS 解析器，至少要做到：

- 解析失败 → 不可分析；
- 包装器递归；
- `rm -rf` 的 `/tmp` 例外；
- 不要把"含变量"当安全。

---

## 12. 证据索引

- `packages/agent-core-v2/src/agent/permissionPolicy/policies/dangerous-command-ask.ts`
- `packages/agent-core-v2/src/app/bashParser/*`
- `packages/tree-sitter-bash/{README.md,src/*,test/*}`
- `packages/agent-core-v2/src/agent/permissionPolicy/permissionPolicyService.ts`（注册顺序）
