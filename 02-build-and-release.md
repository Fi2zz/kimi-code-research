# 02 · 构建、打包与发布

> 目标：解释 Kimi Code 如何从 TypeScript 源码得到一个可签名的单文件可执行程序，以及发布/更新链路。重实现时可以照此复刻，或明确跳过哪一步。

---

## 1. 两条分发路线

| 路线 | 产物 | 适用 |
| --- | --- | --- |
| npm 包 | `@moonshot-ai/kimi-code`，`bin.kimi = dist/main.mjs` | 已有 Node 的用户、CI |
| 原生单文件 | `kimi` / `kimi.exe`，Node SEA | 普通用户，安装脚本默认 |

`package.json` 关键字段：

```json
{
  "bin": { "kimi": "dist/main.mjs" },
  "files": ["dist", "dist-web", "native", "scripts/postinstall.mjs", "scripts/postinstall", "README.md"],
  "engines": { "node": ">=22.19.0" }
}
```

注意：**npm 包要求 Node ≥22.19，原生构建要求 Node ≥24.15**。这是两个不同的下限。

---

## 2. 普通 JS 构建

```
prebuild:  build-vis-asset  →  sync-web-dist
build:     tsdown (ESM)     →  tsdown workers
           → copy-native-assets
           → check-web-assets
```

- `tsdown.config.ts`：单入口 `src/main.ts`，ESM，输出 `dist/main.mjs`，不拆包。
- `tsdown.worker.config.ts`：构建 worker（minidb 文本索引、KAP 搜索）。
- `copy-native-assets.mjs`：把 `packages/pi-tui/native/<platform>/prebuilds/**` 复制到 `apps/kimi-code/native/`。
- `check-web-assets.mjs`：要求 `dist-web/index.html` 存在，否则构建失败。

`postinstall.mjs` 只对全局安装生效：识别 pnpm/yarn/npm，迁移旧的 `kimi` shim，**不下载二进制**。

---

## 3. 原生 SEA 构建（五步）

入口：`pnpm build:native:sea`（local）或 `build:native:release`（release）。

### 3.1 版本与目录

```
apps/kimi-code/
├── dist-native/
│   ├── intermediates/
│   │   ├── main.cjs
│   │   ├── kimi.blob
│   │   └── built-in-catalog.json
│   └── bin/<target>/kimi
└── scripts/native/
    ├── build.mjs
    ├── 01-bundle.mjs
    ├── 02-sea-blob.mjs
    ├── 03-inject.mjs
    ├── 04-sign.mjs
    ├── 05-verify.mjs
    ├── assets.mjs / manifest.mjs / native-deps.mjs / web-assets.mjs
    ├── sea-options.mjs / package.mjs / produce-manifest.mjs
    └── entitlements.plist
```

目标标识：`<platform>-<arch>`，例如 `darwin-arm64`、`linux-x64`、`win32-x64`。

### 3.2 Step 01 — bundle

```bash
build-vis-asset.mjs
tsdown --config tsdown.native.config.ts   # → main.cjs（CJS）
tsdown --config tsdown.worker.config.ts
node check-bundle.mjs
```

`check-bundle.mjs` 会拒绝 bundle 中残留的外部/相对 import（白名单除外），确保运行时没有意外依赖。

### 3.3 Step 02 — SEA blob

```bash
node --experimental-sea-config sea-config.json
```

`sea-config.json` 关键字段：

```json
{
  "main": "main.cjs",
  "output": "kimi.blob",
  "assets": { "<key>": "<absolute path>" },
  "disableExperimentalSEAWarning": true,
  "useCodeCache": true,          // 仅当 target == host
  "useSnapshot": false,
  "execArgv": ["--max-semi-space-size=16"]
}
```

**Asset key 约定**（重实现时照抄，否则运行时解包会找不到）：

| 类别 | key | 内容 |
| --- | --- | --- |
| native manifest | `native/<target>/manifest.json` | 原生依赖清单（版本 2） |
| native 文件 | `native/<target>/node_modules/<pkg>/<rel>` | 包文件 |
| worker | `native/<target>/runtime/minidb-text-build-worker` | 可执行 worker |
| worker | `native/<target>/runtime/kap-search-worker` | 可执行 worker |
| web manifest | `web/<target>/manifest.json` | Web 资源清单（版本 1） |
| web 文件 | `web/<target>/dist-web/<rel>` | Vue 构建产物 |

清单包含每个文件的 SHA-256、目标、版本；运行时逐项校验后原子写入缓存。

### 3.4 Step 03 — inject

```bash
cp $(which node) dist-native/bin/<target>/kimi
codesign --remove-signature ...        # macOS
postject kimi NODE_SEA_BLOB kimi.blob \
  --sentinel-fuse NODE_SEA_FUSE_fce680ab2cc467b6e072b8b5df1996b2 \
  --macho-segment-name NODE_SEA         # macOS 额外参数
```

Windows 用 `signtool remove`，注入后重新签名。

### 3.5 Step 04 — sign

macOS release：

```bash
codesign --sign "$APPLE_SIGNING_IDENTITY" \
  --options runtime \
  --entitlements entitlements.plist \
  --timestamp \
  --keychain "$APPLE_KEYCHAIN_PATH" \
  --force kimi
```

本地/ad-hoc：`codesign --sign -`。
Windows 可选 Azure Trusted Signing（`KIMI_AZURE_TRUSTED_SIGNING=true`）。
总是生成 `<exe>.sha256`。

Entitlements 包含：

```
com.apple.security.cs.allow-jit
com.apple.security.cs.allow-unsigned-executable-memory
com.apple.security.cs.disable-library-validation
com.apple.security.cs.allow-dyld-environment-variables
```

### 3.6 Step 05 — verify

- `codesign -dv --verbose=2`
- 可选 `spctl -a -vvv -t install`
- Windows `Get-AuthenticodeSignature` 必须为 `Valid`

然后：notarize（macOS）、native smoke（exec-server 握手、fs metadata、askpass）、zip（`kimi-code-<target>.zip`，mode `0100755`）。

### 3.7 产物清单

`produce-manifest.mjs` 校验 zip 校验和，生成：

```
manifest.json {
  version, tag,
  platforms: {
    "<target>": {
      filename, checksum,
      compressed: { filename, checksum }
    }
  }
}
```

以及 `.zst`、`.tar.gz`、各自 `.sha256`。更新器只读这个 manifest。

---

## 4. 原生依赖：哪些进包，哪些不进

| 依赖 | 是否内嵌 | 说明 |
| --- | --- | --- |
| `@mariozechner/clipboard` | 是 | JS + 各平台原生子包 |
| `@moonshot-ai/pi-tui` 的 `.node` | 是 | 终端渲染原生模块 |
| `rg`（ripgrep 15.0.0） | **否** | PATH → `~/.kimi-code/bin/rg` → CDN 下载 + SHA-256 |
| `fd` | **否** | 只查 PATH / `~/.kimi-code/bin/fd`，无下载器 |

`rg` 下载地址按区域：

```
${cdnBase}/rg/ripgrep-15.0.0-<target>.{tar.gz|zip}
```

下载后校验、解压、`chmod 755`、原子 rename。

运行时缓存根：

```
KIMI_CODE_CACHE_DIR
  ?? macOS: ~/Library/Caches/kimi-code
  ?? Windows: %LOCALAPPDATA%/kimi-code
  ?? Linux: $XDG_CACHE_HOME/kimi-code or ~/.cache/kimi-code
native/<version>/<target>/<sha256(manifest)>/
web/<version>/<target>/<sha256(manifest)>/dist-web/
```

---

## 5. 版本与更新

构建期注入常量：

```ts
__KIMI_CODE_VERSION__
__KIMI_CODE_CHANNEL__      // stable / canary
__KIMI_CODE_COMMIT__
__KIMI_CODE_BUILD_TARGET__
```

每次 CLI 启动都会做"强制 + 被动"更新预检：

1. 请求 CDN：`/latest`、`/latest.json`、`/channels/canary.json`。
2. 按 rollout 批次决定是否提示。
3. 缓存结果到 update-state JSON。
4. npm/pnpm/yarn/bun/native 可自动安装；Homebrew 只提示手动。
5. 原生更新：下载新二进制 → 校验 manifest（semver + 64-hex）→ 暂存 → 重启替换。

原生更新 manifest：

```
${cdnBase}/binaries/<version>/manifest.json
```

平台 key = `<platform>-<arch>`。

---

## 6. 重实现的最小可行路径

如果你不打算第一天就做单二进制：

1. **阶段 A**：pnpm + tsdown，产出 `dist/main.mjs`，`bin` 指向它。  
   验证：`node dist/main.mjs --version`。
2. **阶段 B**：加 `postject` + `sea-config.json`，只嵌入应用 JS，不嵌 Web/原生依赖。  
   验证：`./kimi --version` 在无 Node 环境下可跑。
3. **阶段 C**：加资源 manifest + 运行时解包（Web/worker/原生）。  
   验证：断网、清缓存后仍能启动 TUI/Web。
4. **阶段 D**：签名/公证/更新器。  
   验证：篡改一个资源字节，启动必须拒绝（SHA-256 校验）。

---

## 7. 不变量与坑

- **Asset key 必须与运行时读取逻辑完全一致**；改 key 等于改协议。
- **`useCodeCache` 只能在本机目标启用**；交叉编译时禁用，否则 V8 缓存不兼容。
- **签名顺序**：先 remove 旧签名 → 注入 → 重新签名；顺序错了 macOS 会拒绝启动。
- **`rg` 不是内置的**：首次离线使用 `Grep` 会失败并提示下载。
- **npm 包与原生包的 Node 下限不同**：文档/CI 要分开写。
- **Web 资源版本号**（1）与 native 资源版本号（2）独立演进，不要合并。

---

## 8. 证据索引

- `apps/kimi-code/package.json`
- `apps/kimi-code/scripts/native/{build,01-bundle,02-sea-blob,03-inject,04-sign,05-verify,assets,manifest,native-deps,web-assets,sea-options,package,produce-manifest}.mjs`
- `apps/kimi-code/scripts/native/entitlements.plist`
- `apps/kimi-code/src/native/{native-assets,web-assets}.ts`
- `packages/agent-core-v2/src/os/backends/node-local/tools/rgLocator.ts`
- `packages/agent-core-v2/src/workspaceFs/internal/fdSuggest.ts`
- 本地二进制：`strings` 可见 783 个 `web/...` key 与 `NODE_SEA_BLOB`
