# 15A · 危险命令分析：逐函数实现笔记

> 配套 `15-dangerous-command-analysis.md`。本文分三层：**解析器（tree-sitter-bash）→ 分析器（policy）→ 判定规则**，逐函数给出语义、边界与测试向量。  
> 关键结论：这是一套**保守的启发式**，不是沙箱；"无法分析"在 manual 下问、yolo 下放行。

---

## 0. 处理管线

```
Bash tool args.command
  → bashCommandText(args)
  → analyzeSource(command, depth=0, parse)
      → parser.parse(source, {timeoutMs:500, maxNodes:10000})
      → collectCommands(root)
      → for each command: analyzeCommand(...)
          → literalText(...)
          → normalizeCommandName(...)
          → analyzeInvocation(name, args, dropped, depth, parse)
              ├─ 提权/启动包装递归
              ├─ 嵌套 shell / eval 递归
              └─ 固定规则判定
  → { kind:"dangerous", command } | { kind:"unanalyzable" } | undefined
```

---

## 1. 解析器包结构

`packages/tree-sitter-bash/src/`：

| 文件 | 行数 | 职责 |
| --- | --- | --- |
| `budget.ts` | 46 | 时间/节点预算，超限抛 `Aborted` |
| `lexer.ts` | 671 | 词法：word/op/io_number/newline/comment/eof；引号/替换/heredoc 扫描 |
| `grammar.ts` | 110 | 保留字、运算符、重定向运算符、表达式优先级 |
| `node.ts` | 78 | `SyntaxNode`、`SyntaxNodeBuilder`、`descendantsOfType` |
| `parser.ts` | 3,096 | 递归下降解析器，产出 `Frame` 树 |
| `parse.ts` | 26 | 公共入口 `parse()` |
| `index.ts` | 4 | 导出 |

---

## 2. `budget.ts`

```ts
const DEFAULT_TIMEOUT_MS = 50;
const DEFAULT_MAX_NODES = 50_000;

class Aborted extends Error {}

class ParseBudget {
  deadline = Date.now() + (options.timeoutMs ?? 50)
  maxNodes = options.maxNodes ?? 50_000
  nodeCount = 0

  tick(): void {
    nodeCount++
    if (nodeCount > maxNodes) throw new Aborted("node budget exceeded")
    if (Date.now() >= deadline) throw new Aborted("timeout")
  }

  progress(): void {
    if (Date.now() >= deadline) throw new Aborted("timeout")
  }
}
```

- `tick()` 每创建一个节点调用一次；
- `progress()` 用于长扫描循环（每 2048 字符一次），只检查时间；
- 超限抛 `Aborted`，`parse()` 捕获后返回 `{ ok:false, reason:"aborted" }`。

危险分析覆盖默认值：`{ timeoutMs: 500, maxNodes: 10_000 }`。

---

## 3. `node.ts`

```ts
interface SyntaxNode {
  type, text, startIndex, endIndex, isNamed,
  parent, children, namedChildren
}

class SyntaxNodeBuilder {
  addChild(child) {
    // child 不能已有 parent
    // child 范围必须落在 parent 内
    // child.startIndex 不能小于上一个 sibling 的 endIndex（不允许重叠）
  }
}

function descendantsOfType(root, ...types): SyntaxNode[] {
  // 迭代式 DFS，按 namedChildren 前序
}
```

不变量：

- 子节点范围必须被父节点包含；
- 兄弟节点不能重叠；
- `text` 始终是 `source.slice(start,end)`（UTF-16 code unit）。

---

## 4. `lexer.ts`

### 4.1 Token

```ts
type TokenType = "word" | "op" | "io_number" | "newline" | "comment" | "eof";

interface Token {
  type; start; end;
  heredocBodies: HeredocBody[];
}
```

### 4.2 扫描优先级（`Lexer.scanToken`）

```
budget.progress()
skipBlanks()
if pos >= end        → eof
if ch === "\n"        → newline（同时读取 pending heredoc 体）
if ch === "#"         → comment 到行尾
if ch === "<" || ">"  → 若下一个是 "(" → scanWord（进程替换）；否则 scanOp(REDIRECT_OPERATORS)
if ch === "&|;()"     → 若 "((" → scanWord（算术）；否则 scanOp(CONTROL_OPERATORS)
if ch === "{}[]"      → 单字符 word
if digit              → 读数字；若后面是 < 或 > → io_number；否则 scanWord
else                  → scanWord
```

`REDIRECT_OPERATORS`：

```
<<- <<< << <&- <& <> < >&- >& >> >| >
```

`CONTROL_OPERATORS`：

```
&>> &> && & |& || | ;;& ;; ;& ; ( )
```

### 4.3 关键辅助函数

| 函数 | 语义 |
| --- | --- |
| `skipDoubleQuoted(source,budget,i,end,depth)` | 跳过 `"..."`；处理 `\`、`` ` ``、`$(...)`、`${...}`；深度 ≥1024 返回 end |
| `skipSingleQuoted` | 找下一个 `'`；没有返回 end |
| `skipBacktick` | 跳过 `` `...` ``；处理 `\` |
| `scanBalanced(source,budget,i,end,open,close,depth)` | 跳过平衡括号；返回 `{end, balanced}` |
| `scanBalancedStatements` | 专门处理 `$( ... )` 内的语句列表 |
| `skipDollar` | 处理 `$(`、`${`、`$[`、`$'...'`、`$NAME`、`$@` 等 |
| `scanHeredocDelimiter` | 解析 `<<EOF`、`<<-EOF`、`<<"EOF"` |
| `skipHeredocBodies` | 跳过 heredoc 正文直到 delimiter |

### 4.4 `Lexer`

```ts
class Lexer {
  pos
  private lookahead: Token[]
  private pendingHeredocs: HeredocSpec[]

  queueHeredoc(spec)
  peek() / peekAt(n)
  next()
  reposition(pos)
  text(token)
  private scanToken()
  private scanBoundary(type, start, end)  // 读取所有 pending heredoc bodies
  ...
}
```

`peekAt` 会按需扫描并缓存；`reposition` 清空 lookahead（用于重新解析区间）。

---

## 5. `parser.ts`

### 5.1 Frame

```ts
interface Frame {
  type: string;
  start: number;
  end: number;
  isNamed: boolean;
  parent: Frame | null;
  children: Frame[];
}
```

`Parser.frame()` 每次调用 `budget.tick()`；`anon()` 创建未命名节点。

### 5.2 Parser 字段

```ts
hasError: boolean
lexer: Lexer
heredocQueue: PendingHeredoc[]
scopeDepth: number
literalDepth: number
exprDepth: number
noHeredoc: boolean
caseItemDepth: number
lastTerminatorEnd: number
depth: number   // 嵌套 Parser 深度
```

### 5.3 主流程

```
parseProgram()
  lexer = new Lexer(source, budget, 0, len)
  children = parseStatementList()
  return frame("program", 0, len, children)
```

`parseStatementList(options)`：

- 循环 peek：
  - `eof` → 完成/失败 heredoc，break；
  - `newline` → 消费，完成 heredoc，重置 `needTerminator`；
  - `comment` → 记录；
  - 命中 `stopWords` → break（若需要终止符则 `hasError=true`）；
  - 命中 `stopOps` → break；
  - `)` + `stopAtParen` → break；
  - `;`/`&`/`;;` → 作为匿名节点；
  - 非法 `)`/`&&`/`||`/`|`/`|&`/`;&`/`;;&` → 记录 `ERROR`，继续；
  - 若 `needTerminator` 为真却来了新语句 → `hasError=true`；
  - 否则 `parseList()`。

`parseList()`：`parsePipeline()`，然后循环 `&&`/`||` 组合成 `list` 节点。

`parsePipeline()`：`parsePipelineTail(parseStatementNotPipeline())`，处理 `|`/`|&`。

`parseStatementCore()`：分发到：

- `parseCompoundStatement()`：`if/while/for/case/{...}/(...)`
- `parseFunctionDefinition()`
- `parseDeclarationCommand()`：`declare/typeset/export/readonly/local`
- `parseUnsetCommand()`
- `parseCommand()`

`parseCommand()`：

- 处理 `command_name`、参数、重定向、变量赋值、数组、subshell、算术等；
- `parseCommandArgument()` → `parseWordArgument()` → `parseLiteral()` / `parseString()` / `parseDollar()`；
- `parseRedirect()` → `parseFileRedirect()` / `parseHerestringRedirect()` / `parseHeredocRedirect()`。

`parseScopedStatements(start,end)`：新建 `Parser`，深度 +1；≥`MAX_SUBSTITUTION_DEPTH`（150）时返回 `ERROR`。

### 5.4 常量

```ts
MAX_PARSE_DEPTH = 500
MAX_SUBSTITUTION_DEPTH = 150
```

`MAX_SCAN_DEPTH = 1024` 在 lexer。

### 5.5 `materialize(root, source)`

```
1. 迭代遍历 Frame，收集 order（父在前）
2. 为每个 Frame 创建 SyntaxNodeBuilder
3. 再遍历一次，按 children 顺序 addChild
4. 返回 root builder
```

`parse()` 的异常处理：

```
try { root = parser.parseProgram(); return {ok:true, rootNode: materialize(root), hasError: parser.hasError} }
catch (Aborted) { return {ok:false, reason:"aborted"} }
catch (any) {
  root = program[0,len] + ERROR[0,len]
  return {ok:true, rootNode: root, hasError:true}
}
```

即：**除了预算终止，其它异常都降级为 `hasError` 的树，不抛给调用方**。

---

## 6. 分析器逐函数

### 6.1 `bashCommandText(args)`

```ts
if (typeof args !== "object" || args === null) return undefined
const command = (args as {command?: unknown}).command
return typeof command === "string" ? command : undefined
```

### 6.2 `analyzeSource(source, depth, parse)`

```ts
const parsed = parse(source)
if (!parsed.ok || parsed.hasError) return { kind: "unanalyzable" }

const commands = []
collectCommands(parsed.root, commands)
for (const command of commands) {
  const verdict = analyzeCommand(command, depth, parse)
  if (verdict !== undefined) return verdict
}
return undefined
```

注意：**任一命令危险/不可分析就立即返回**；安全命令继续。

### 6.3 `collectCommands(node, out)`

```ts
if (node.type === "command") out.push(node)
for (const child of node.children) collectCommands(child, out)
```

递归所有子节点（包括匿名），收集 `command` 节点。

### 6.4 `analyzeCommand(command, depth, parse)`

```ts
const nameIndex = command.children.findIndex(c => c.type === "command_name")
const nameNode = nameIndex >= 0 ? command.children[nameIndex] : undefined
const nameWord = nameNode?.children.find(c => c.isNamed)
const rawName = nameWord === undefined ? undefined : literalText(nameWord)
if (rawName === undefined || rawName.length === 0) return { kind: "unanalyzable" }

const args = []
let dropped = false
for (const child of command.children.slice(nameIndex + 1)) {
  if (SKIPPED_COMMAND_CHILDREN.has(child.type)) continue
  const value = literalText(child)
  if (value === undefined) dropped = true
  else if (value.length > 0) args.push(value)
}

return analyzeInvocation(normalizeCommandName(rawName), args, dropped, depth, parse)
```

`SKIPPED_COMMAND_CHILDREN`：

```
variable_assignment, file_redirect, heredoc_redirect
```

注意：重定向被跳过（不把 `>/dev/sda` 当参数），所以 `cmd > /dev/sda` 不会被 `dd`/`rm` 规则识别；这是一个已知盲区。

### 6.5 `literalText(node)`

| 节点类型 | 规则 |
| --- | --- |
| `word` | 原文含 `UNSAFE_OPERAND` → undefined；反转义 `\\(.)`；反转义后仍含不安全字符 → undefined；否则返回 |
| `number` | 返回 `node.text` |
| `raw_string` | 长度 <2 → undefined；去首尾引号；含不安全字符 → undefined |
| `string` | 拼接 `string_content`；遇到其它 named child（变量/替换）→ undefined；去转义；含不安全字符 → undefined |
| 其它 | undefined |

`UNSAFE_OPERAND = /[$`*?[\]~]/`。

因此以下都会 `dropped`：

```
$VAR  ${VAR}  $(cmd)  `cmd`  *.ts  ?  [a]  ~  ~/x
```

### 6.6 `analyzeInvocation(name, args, dropped, depth, parse)`

#### (1) 提权包装

```
PRIVILEGE_WRAPPERS = { sudo, doas }
PRIVILEGE_VALUE_OPTIONS = {
  -u,--user, -g,--group, -h,--host, -p,--prompt,
  -C,--close-from, -T,--command-timeout,
  -U,--other-user, -r,--role, -t,--type
}
```

```
rest = dropLeadingOptions(args, PRIVILEGE_VALUE_OPTIONS)
inner = rest[0]
if (inner === undefined) return dropped ? unanalyzable : undefined
return analyzeInvocation(normalize(inner), rest.slice(1), dropped, depth, parse)
```

#### (2) 启动包装

```
LAUNCH_WRAPPERS = { env, command, exec, nohup, builtin, nice }
WRAPPER_VALUE_OPTIONS = {
  -u,--unset, -C,--chdir, -S,--split-string,
  -a, -n, --adjustment
}
```

`command` 特例：

```
for (arg of args) {
  if (arg === "--") break
  if (arg === "-") continue
  if (!arg.startsWith("-")) break
  if (/[vV]/.test(arg)) return undefined   // command -v / -V 视为只读
}
```

然后：

```
rest = dropLaunchWrapperOperands(name, args)
inner = rest[0]
if (!inner) return dropped ? unanalyzable : undefined
return analyzeInvocation(normalize(inner), rest.slice(1), dropped, depth, parse)
```

`dropLaunchWrapperOperands`：

```
rest = dropLeadingOptions(args, WRAPPER_VALUE_OPTIONS)
if (name === "env") {
  i = rest[0] === "-" ? 1 : 0
  while (i < rest.length && /^[A-Za-z_][A-Za-z0-9_]*=/.test(rest[i])) i++
  rest = rest.slice(i)
}
return rest
```

#### (3) 嵌套 shell

```
NESTED_SHELLS = { sh, bash, dash, zsh, ksh, ash }
```

```
payloadIndex = -1
for (i=0; i<args.length; i++) {
  arg = args[i]
  if (arg === "--") break
  if (/^-[a-zA-Z]+$/.test(arg)) { if (arg.includes("c")) payloadIndex = i + 1 }
  else break
}
if (payloadIndex < 0) return dropped ? unanalyzable : undefined
payload = args[payloadIndex]
if (payload === undefined || depth >= 4) return unanalyzable
return analyzeSource(payload, depth + 1, parse)
```

#### (4) `eval`

```
if (args.length === 0) return dropped ? unanalyzable : undefined
if (dropped || depth >= 4) return unanalyzable
return analyzeSource(args.join(" "), depth + 1, parse)
```

#### (5) `busybox`

```
applet = args[0]
if (applet === undefined || applet.startsWith("-")) return dropped ? unanalyzable : undefined
return analyzeInvocation(normalize(applet), args.slice(1), dropped, depth, parse)
```

#### (6) 固定危险命令

```
SIMPLE_DANGEROUS_COMMANDS = {
  shutdown, halt, poweroff, reboot,
  bcdedit, diskpart, format,
  restart-computer, stop-computer,
  mkfs, wipefs
}
if (SIMPLE_DANGEROUS_COMMANDS.has(name) || name.startsWith("mkfs.")) return { kind:"dangerous", command:name }
```

#### (7) `init` / `telinit`

```
if (name === "init" || name === "telinit") {
  if (args.some(a => a === "0" || a === "6")) return dangerous
  return dropped ? unanalyzable : undefined
}
```

#### (8) `systemctl`

```
SYSTEMCTL_DANGEROUS_SUBCOMMANDS = { poweroff, reboot, halt, kexec }
SYSTEMCTL_VALUE_OPTIONS = { -H,--host, -M,--machine }

sub = dropLeadingOptions(args, SYSTEMCTL_VALUE_OPTIONS)[0]
if (sub && dangerous.has(sub)) return { kind:"dangerous", command:`systemctl ${sub}` }
return dropped ? unanalyzable : undefined
```

#### (9) `dd`

```
DD_SAFE_DEVICE_TARGETS = {
  /dev/null, /dev/zero, /dev/full, /dev/random,
  /dev/urandom, /dev/stdin, /dev/stdout, /dev/stderr
}
for (arg of args) {
  if (!arg.startsWith("of=")) continue
  target = arg.slice(3)
  if (target.startsWith("/dev/") && !safe.has(target)) return dangerous "dd"
}
return dropped ? unanalyzable : undefined
```

#### (10) `rm`

```
recursive=false; force=false; operands=[]; optionsEnded=false
for (arg of args) {
  if (!optionsEnded && arg === "--") { optionsEnded = true; continue }
  if (optionsEnded) { operands.push(arg); continue }
  if (arg === "--recursive") recursive = true
  else if (arg === "--force") force = true
  else if (/^-[a-zA-Z]+$/.test(arg)) {
    if (/[rR]/.test(arg)) recursive = true
    if (arg.includes("f")) force = true
  } else operands.push(arg)
}

if (recursive && force) {
  if (!dropped && operands.length > 0 && operands.every(isSafeTempRmOperand)) return undefined
  return { kind:"dangerous", command:"rm -rf" }
}
return dropped ? unanalyzable : undefined
```

`isSafeTempRmOperand`：

```
for (segment of operand.split("/")) if (segment === "..") return false
return operand === "/tmp" || operand.startsWith("/tmp/")
    || operand === "/temp" || operand.startsWith("/temp/")
```

**重要修正**：`rm -rf $DIR` 会因为 `dropped=true` 且 `!dropped` 为假，落到 `dangerous`，而不是 `unanalyzable`。这是保守设计：只要递归+强制，即使参数不可分析也判危险。

#### (11) 其它

```
return undefined
```

---

## 7. 辅助函数

### 7.1 `normalizeCommandName(raw)`

```
name = raw
sep = max(lastIndexOf("/"), lastIndexOf("\\"))
if (sep >= 0) name = name.slice(sep + 1)
name = name.toLowerCase()
if (name.endsWith(".exe")) name = name.slice(0, -4)
return name
```

### 7.2 `dropLeadingOptions(args, valueOptions)`

```
for (i=0; i<args.length; i++) {
  arg = args[i]
  if (arg === "--") return args.slice(i+1)
  if (arg === "-" || !arg.startsWith("-")) return args.slice(i)
  if (!arg.includes("=") && valueOptions.has(arg)) i++
}
return []
```

### 7.3 `dropLaunchWrapperOperands(name, args)`

见 §6.6(2)。

---

## 8. AST 形状示例（分析器关心的部分）

### 8.1 `sudo rm -rf /`

```
command
├─ command_name
│  └─ word "sudo"
├─ word "rm"
├─ word "-rf"
└─ word "/"
```

`analyzeCommand`：rawName=sudo；args=["rm","-rf","/"]；dropped=false。
`analyzeInvocation(sudo,...)` → 提权包装 → `dropLeadingOptions` 无选项 → rest=["rm","-rf","/"] → 递归 `rm` → dangerous。

### 8.2 `bash -c 'rm -rf /'`

```
command
├─ command_name → word "bash"
├─ word "-c"
└─ raw_string "'rm -rf /'"
```

args=["-c","rm -rf /"]；NESTED_SHELLS 找到 `-c`，payload="rm -rf /"；深度 1 递归；dangerous。

### 8.3 `rm -rf /tmp/x`

args=["-rf","/tmp/x"]；recursive+force；operand 安全 → undefined。

### 8.4 `rm -rf $DIR`

`word "$DIR"` 含 `$` → literalText undefined → dropped=true，operands=[]；
`recursive && force` → 不是 safe temp → `dangerous rm -rf`。

### 8.5 `eval "$(cat script.sh)"`

`string` 内含 `$(...)` named child → literalText undefined → args 空/dropped；
eval 分支：args.length===0 → `dropped ? unanalyzable : undefined` → unanalyzable。

### 8.6 `command -v git`

command 分支扫描 args，遇 `-v` → 返回 undefined（安全）。

---

## 9. 测试矩阵（建议直接落成单测）

| 命令 | 期望 |
| --- | --- |
| `rm -rf /` | dangerous `rm -rf` |
| `rm -rf /tmp/build` | safe |
| `rm -rf /tmp/../etc` | dangerous |
| `rm -rf $DIR` | dangerous `rm -rf` |
| `rm -rf` | dangerous `rm -rf` |
| `rm -fr ~` | dangerous `rm -rf` |
| `sudo rm -rf /` | dangerous |
| `sudo -u root rm -rf /` | dangerous |
| `doas rm -rf /` | dangerous |
| `env FOO=1 rm -rf /` | dangerous |
| `env -u FOO rm -rf /` | dangerous |
| `command rm -rf /` | dangerous |
| `command -v git` | safe |
| `nohup rm -rf /` | dangerous |
| `nice -n 10 rm -rf /` | dangerous |
| `sh -c 'rm -rf /'` | dangerous |
| `bash -lc 'rm -rf /'` | dangerous |
| `bash -c "$CMD"` | unanalyzable |
| `eval 'rm -rf /'` | dangerous |
| `eval "$X"` | unanalyzable |
| `busybox rm -rf /` | dangerous |
| `dd if=/dev/zero of=/dev/sda` | dangerous `dd` |
| `dd if=/dev/zero of=/dev/null` | safe |
| `systemctl reboot` | dangerous `systemctl reboot` |
| `systemctl --host x reboot` | dangerous |
| `shutdown -h now` | dangerous |
| `mkfs.ext4 /dev/sda1` | dangerous |
| `wipefs -a /dev/sda` | dangerous |
| `init 0` | dangerous |
| `init 3` | safe |
| `cat /etc/hosts` | safe |
| `echo "rm -rf /"` | safe（字符串在 echo 参数里，不执行） |

### 误报/漏报测试

| 命令 | 实际行为 | 评价 |
| --- | --- | --- |
| `chmod -R 000 /` | safe | 漏报 |
| `find / -delete` | safe | 漏报 |
| `git clean -fdx` | safe | 漏报 |
| `> /dev/sda` | safe（重定向被跳过） | 漏报 |
| `./script.sh` | safe（未识别命令） | 漏报 |
| `base64 -d x \| sh` | pipeline 内 `sh` 无 `-c` → safe | 漏报 |
| `rm -rf ./build` | dangerous（非 /tmp） | 可能误报 |
| `rm -rf /tmp` | safe | 符合设计 |
| `docker run --rm -v /:/host alpine rm -rf /host` | safe（未识别 docker） | 漏报 |

这些正是"不能当沙箱"的原因。

---

## 10. 实现顺序

1. `ParseBudget` + `SyntaxNode`。
2. Lexer：先 word/op/newline/comment；再加引号/替换/heredoc。
3. Parser：statement list → list → pipeline → command → args/redirects。
4. `materialize`。
5. 分析器：`analyzeSource`/`collectCommands`/`analyzeCommand`/`literalText`。
6. 包装递归：sudo → env/command → sh -c → eval → busybox。
7. 固定规则：rm/dd/systemctl/init/mkfs。
8. 失败策略与测试向量。

如果不想写完整 parser：

- 用官方 tree-sitter-bash WASM；
- 或先做"分词 + 递归下降"只覆盖 `command`/`word`/引号；
- 但**必须**实现包装递归和 fail-closed，否则等于正则。

---

## 11. 证据索引

- `packages/agent-core-v2/src/agent/permissionPolicy/policies/dangerous-command-ask.ts`
- `packages/agent-core-v2/src/app/bashParser/*`
- `packages/tree-sitter-bash/src/{budget,lexer,grammar,node,parser,parse,index}.ts`
- `packages/tree-sitter-bash/test/*`
- `packages/tree-sitter-bash/README.md`（差分测试与已知差异）
