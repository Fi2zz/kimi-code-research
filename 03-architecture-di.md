# 03 · 架构：DI × Scope

> 目标：解释 Kimi Code 如何用依赖注入 + 作用域把"一个进程内的多个会话、多个 agent、多个工具/特性"组织起来，以及为什么这样组织。重实现时这是最容易被低估的一层。

---

## 1. 为什么不是普通的模块单例

一个 coding agent 进程里同时存在：

- 多个会话（session），每个会话有自己的 cwd、环境、权限规则；
- 每个会话里一个 main agent + N 个 subagent；
- 每个 agent 有自己的工具表、历史、权限模式；
- 特性（todo/goal/cron/mcp/...）按需启停，且要能热插拔。

如果全用模块级单例，会出现"哪个 session 的规则生效"这类问题。Kimi 的解法是：

```
ServiceIdentifier（token） + Scope（生命周期容器） + Collection（多提供者）
```

同一个 token 在不同 scope 里可以解析到不同实例；特性通过 collection 贡献服务/工具；scope 关闭时统一释放资源。

---

## 2. 三个生命周期作用域

`packages/agent-core-v2/src/app/scopes.ts`：

```ts
enum LifecycleScope { App = "app", Session = "session", Agent = "agent" }
const SCOPE_TOPOLOGY = [App, Session, Agent];
```

| Scope | 生命周期 | 典型内容 |
| --- | --- | --- |
| App | 进程 | 配置、存储、遥测、模型目录、插件目录、MCP 配置 |
| Session | 一次会话 | 会话上下文、wire、状态、工作区、session 级 MCP、权限规则 |
| Agent | 一个 agent | 历史、工具表、权限模式、循环、审批、task |

规则：

- 子 scope 的 kind 索引必须大于父 scope（`setScopeTopology` 强制）。
- 子容器共享父的 `CascadeTree` 和 `CollectionStore`，所以依赖级联和 collection 可见性是全拓扑的。
- 只有 App 是 strict 容器；未知服务直接报错。Session/Agent 允许按需贡献。

---

## 3. InstantiationService：容器

`_base/di/instantiationService.ts`：

```ts
class InstantiationService implements IInstantiationService {
  constructor(
    services: ServiceCollection = new ServiceCollection(),
    strict = false,
    parent?: InstantiationService,
    enableTracing = false,
  );

  invokeFunction(fn, ...args);
  createInstance(descriptor | ctor, ...args);
  createChild(services, store?): InstantiationService;
  provide(id, instanceOrDescriptor, options?): ProvideHandle;
  unprovide(id);
  dispose(); disposeAsync();
}
```

关键内部：

- `_services: ServiceCollection`：token → `{ value, uid, config?, recipe? }`。
- `_ledger: Ledger`：本容器创建的所有 disposer。
- `_tree: CascadeTree` / `cascade: CascadeEngine`：依赖图与重建。
- `_collectionStore`：collection 记录。
- `_instanceEntries` / `_provideEntries`：构造追踪。

构造服务时：

1. 读取构造函数上的 `DI_DEPENDENCIES` 元数据；
2. 按参数顺序解析每个 token；
3. 用 `Reflect.construct` 构造；
4. 若服务是 `Service` 子类，包在构造帧里，允许它在构造期 `provide/effect/on`；
5. 注册进 ledger（`service:<token>`）。

循环依赖抛 `CyclicDependencyError`；strict 容器未知服务抛错。

---

## 4. ServiceIdentifier 与装饰器

`_base/di/instantiation.ts`：

```ts
interface ServiceIdentifier<T> {
  (target: any, key: string | symbol | undefined, index: number): void;
  readonly type: T;
  toString(): string;
}

const IConfigService = createDecorator<IConfigService>("configService");
```

- `createDecorator(name)` 返回按 name 记忆化的装饰器；`toString()` 返回 name。
- 装饰器把 `{ id, index }` 写进构造函数的 `$di$dependencies`。
- 同一个参数可以挂多个装饰器（累积）。
- `ref<T>(id)` 注入 `LiveRef<T> { current?, onDidChange }`，用于"服务可能后出现"的场景。
- `collection<T>(name)` 注入 `CollectionView<T>`，用于多提供者。

使用形态：

```ts
class AgentPermissionGate {
  constructor(
    @IAgentPermissionModeService private readonly mode: IAgentPermissionModeService,
    @IAgentToolExecutorService executor: IAgentToolExecutorService,
  ) {}
}
```

---

## 5. 作用域注册与激活

`_base/di/scope.ts`：

```ts
registerScopedService(
  LifecycleScope.Agent,
  IAgentPermissionGate,
  AgentPermissionGate,
  ScopeActivation.OnScopeCreated,
  "permissionGate", // domain，仅诊断用
);

enum ScopeActivation {
  OnScopeCreated, // eager：scope 建好后立即构造
  OnDemand,       // 第一次 get 时构造
}
```

注册表项：

```ts
interface ScopedEntry {
  scope: ScopeKind;
  id: ServiceIdentifier<unknown>;
  descriptor: SyncDescriptor<unknown>;
  domain: string;
  activation: ScopeActivation;
}
```

创建 scope：

`createScopedChildHandle(parent, kind, id, options)`：

1. `parent.createChild(collection)`；
2. 设置 `debugLabel = id`；
3. **暂停 cascade 激活**；
4. `watchScopeUnits(container, kind)`：把特性贡献的服务 recipe 物化进来；
5. 用户 `configureContainer`；
6. `provideScopeServices`：注册该 scope 的所有 scoped services；
7. **恢复激活**（eager 服务此时构造）；
8. 失败则销毁子容器。

`ScopeSeed` 是 `[token, value][]`，在 scoped services 之前放入容器；`provideScopeServices` 会跳过已 seed 的 token。这是把"每个 session 的上下文对象"注入子容器的标准方式。

---

## 6. Collection：多提供者

`_base/di/collection.ts`：

```ts
const AgentToolContribution = collection<AgentToolContribution>("agent-tool");

interface CollectionView<T> {
  items: readonly T[];
  records: CollectionRecord<T>[]; // { value, providerName, scopePath }
  onDidChange: Event<{ added; removed }>;
}
```

- `collection(name)` 返回一个记忆化的 token，可作构造参数装饰器。
- `definition(name)` 是单例 collection，第二次提供会校验失败。
- 可见性：祖先/后代相关容器可见（`_isRelated`）。因此一个 agent 能看到 session 贡献的工具，也能看到自己贡献的。
- `ScopeUnits(kind)` 是一个特殊 collection（`scope-units:<kind>`）：`Feature.contributeService()` 把 recipe 放进去，`watchScopeUnits` 在 scope 创建时物化。

这解决"特性在编译期注册、在运行期按 scope 生效"的问题，不需要全局可变注册表。

---

## 7. Cascade：依赖重建

`_base/di/cascadeEngine.ts`：

```ts
type UnitState = "Pending" | "Activating" | "Active" | "Unloading" | "Failed";
type UnitActivation = "eager" | "ondemand";
```

- `provide(descriptor)` → unit 变 `Pending`；`provide(instance)` → 立即 `Active`。
- `suspendActivation()` / `resumeActivation()`：批量注册时延迟 eager 构造，恢复后按 `cascadeDepth` 做不动点激活。
- 事务：计算 `graph.affectedSet([...])` → 标记 in-flight → `onWillCascade` 可中止 → 退役受影响的已物化 unit → 从 provider 重建 → 记录 `CascadeHistoryEntry`。
- `_retireUnit` 会 unmaterialize 服务并 dispose 其 ledger entry，下次访问时重建。

这对"配置热更新后，依赖配置的服务自动重建"很关键。重实现时如果不想做完整级联，至少要做到：**配置变更 → 显式重建受影响的少数服务**，而不是全局重启。

---

## 8. Ledger：释放

`_base/lifecycle/ledger.ts`：

```ts
class Ledger {
  state: "active" | "disposing" | "disposed";
  register(disposer, label);
  registerFinalizer(fn);
  effect(body);
  teardown(reason: "scope-close" | "cascade" | "unload");
}
```

- 每个服务、子容器、监听器都进 ledger。
- 释放按注册的逆序执行（LIFO）。
- `isActive` 阻止"已释放后还注册"的泄漏。
- Scope dispose 幂等；先释放子容器，再 teardown 本 ledger。

重实现建议：**所有 `setInterval`、文件监听、子进程、事件订阅都必须经过一个 scope 级 disposer**，否则长会话必漏。

---

## 9. 服务地图（按 scope）

| Scope | 代表服务 |
| --- | --- |
| App | `IConfigService`、`IStorageService`、`IBlobStore`、`ITelemetryService`、`IModelCatalog`、`IPluginService`、`ISessionManager`、`IMcpRegistryService` |
| Session | `ISessionContext`、`ISessionMetadata`、`ISessionStateService`、`ISessionEventBus`、`IWorkspaceContext`、`ISessionLockService`、`ISessionMcpHandle` |
| Agent | `IAgentLoopService`、`IAgentToolRegistryService`、`IAgentPermissionModeService`、`IAgentPermissionRulesService`、`IAgentPermissionGate`、`IAgentToolExecutorService`、`IAgentToolApprovalService`、`IAgentStateService`、`IEventDispatcher`、`IWireService`、`IAgentTaskService` |

注意：`IEventDispatcher` 是 **Agent scope**，所以每个 agent 有独立的 wire、状态、重放。

---

## 10. 不变量

1. 一个 token 在一个容器里只有一个实例；不同 scope 可以有不同实例。
2. eager 服务在 scope 创建完成前必须可构造；不要依赖"稍后才出现"的服务（用 `ref`）。
3. collection 的提供者必须在其 scope 存活期间有效；scope 关闭后记录消失。
4. 服务构造期不要做 I/O；把异步初始化放到显式 `initialize()` 或 scope 创建后的钩子。
5. 任何跨 scope 的引用都要考虑"父先死"；用事件或 `ref`，不要硬持有。
6. 子 scope 不能注册父 scope 的服务（拓扑顺序）。
7. 释放必须幂等；重复 dispose 不能抛错。

---

## 11. 重实现最小版

如果不想一上来做完整 cascade：

1. 实现 `ServiceCollection` + `createDecorator` + 构造注入；
2. 实现 App/Session/Agent 三层容器（父子关系 + ledger）；
3. 实现 `registerScopedService` + `OnScopeCreated`/`OnDemand`；
4. 先不做 cascade 重建，配置变更时手动重建；
5. collection 先做"只读聚合 + 变更事件"，不做跨容器可见性推导；
6. 所有 disposer 走 ledger。

测试重点：

- 同一 token 在父子容器解析到不同实例；
- scope dispose 后，子容器服务全部释放，父容器不受影响；
- eager 服务在 scope 创建后立即可用；
- OnDemand 服务在第一次 get 前未构造；
- 循环依赖报错而不是死循环。

---

## 12. 证据索引

- `packages/agent-core-v2/src/_base/di/{instantiation,instantiationService,serviceCollection,scope,collection,cascadeEngine,service,lifecycle/ledger,lifecycle/disposer}.ts`
- `packages/agent-core-v2/src/app/scopes.ts`
- `packages/agent-core-v2/src/_base/di/fiber.ts`（`ScopeUnits`）
- `packages/agent-core-v2/src/features/feature.ts`（`contributeService` / `contributeTool`）
