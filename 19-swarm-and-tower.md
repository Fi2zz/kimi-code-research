# 19 · Swarm 与 Tower：批量与仓库级多智能体

> 目标：解释两种"多智能体"形态的差异、输入、调度、合并门禁与失败模式。  
> - **Swarm**：一次调用并行跑 N 个同构子任务。  
> - **Tower**：把一个开发目标拆成 mission，每个 mission 独立 git worktree，worker 改、reviewer 审、tower 合并。

---

## 第一部分：Swarm

### 1. 定位

`AgentSwarm` 不是"多次 Agent 调用"的语法糖。它的约束更强：

- 一次调用只能有一个 `AgentSwarm`，且不能与其它工具同批；
- 所有成员共享一个 prompt 模板；
- 并发有爬坡与限流；
- 结果是聚合报告，不是 N 条独立消息。

### 2. 工具参数

```ts
{
  description: string;
  subagent_type?: string;           // 默认 coder
  prompt_template?: string;         // 必须含 {{item}}
  items?: string[];                 // 2..128
  resume_agent_ids?: Record<string, string>;
  model?: string;
  fork?: boolean;
}
```

校验：

- `resume 数量 > 0` 或 `items >= 2`；
- 总数 ≤128；
- 有 `items` 必须有 `prompt_template`，且含 `{{item}}`；
- 渲染后的 prompt 不能重复；
- resume 项先执行（索引 1..n），items 随后；
- fork + resume 拒绝；
- `model` 无 items 拒绝；
- `subagent_type` + `resume_agent_ids` 允许（只影响新 spawn）。

### 3. 模式门控

- 只在 **swarm mode** 开启时可用；
- 只有用户能开启（`/swarm`）；`enter("tool")` 抛 `VALIDATION_FAILED`；
- 状态持久化：`swarm_mode.enter/exit`；
- 触发来源 `manual | task | tool`；
- `task`/`tool` 触发的模式在 `TurnEnded` 自动退出；
- 关闭时 `resolveExecution` 直接返回 `SWARM_MODE_OFF_RESULT`，不执行。

### 4. 并发爬坡

`session/agentRunBatch.ts`：

- 初始并发 5；
- 之后每 700ms 启动一个；
- 硬上限 `KIMI_CODE_AGENT_SWARM_MAX_CONCURRENCY`（非正整数 → `VALIDATION_FAILED`）；
- 自适应限流：
  - 容量 = 当前成功数；
  - 429 时每 2s 减 1；
  - 180s 无 429 恢复 +1；
  - 重试延迟指数：base 3s，factor 2；
- 被限流成员标记 `SubagentSuspended`，由 resume/retry 在同一 agent id 上重试；
- 若它是唯一未完成任务 → 直接失败，而不是无限重排；
- 每任务独立超时（默认 2h）。

### 5. 单调用约束

swarm 模式激活时，工具批次 hook：

- 多个 `AgentSwarm` → 阻止；
- `AgentSwarm` 与其它工具混批 → 阻止。

原因：聚合报告与并发控制需要独占一个 step。

### 6. 聚合

```
<agent_swarm_result>
  <summary>completed: N, failed: M, aborted: K</summary>
  [<resume_hint>…]
  <subagent mode="resume"? agent_id item state outcome="completed|failed|aborted" stop_reason?>
    body
  </subagent>
  ...
</agent_swarm_result>
```

成员也会投影为 `TaskStarted/TaskTerminated`，带 `swarmIndex`。

### 7. Swarm 不变量

1. 只有用户能开模式。
2. 一次只允许一个 Swarm，且独占批次。
3. 模板必须含 `{{item}}`。
4. 总数 ≤128；无 resume 时 ≥2。
5. 渲染后 prompt 唯一。
6. 429 不能导致无限重试；唯一未完成任务要失败。
7. 超时任务标记失败，不阻塞聚合。

---

## 第二部分：Tower

### 8. 定位

Tower 是**实验性**的仓库级协作模式：

- main agent 变成 "tower"（协调者）；
- 把目标拆成 mission；
- 每个 build/artifact mission 有自己的 branch + worktree；
- worker 在 worktree 里开发/提交；
- reviewer 只读审查分支；
- tower 按依赖顺序合并通过门禁的 mission。

它与 Swarm 的区别：Swarm 是"同构并行"，Tower 是"异构任务 + 独立分支 + 审查门禁 + 合并"。

### 9. 门控

- flag `tower`；`KIMI_CODE_EXPERIMENTAL_TOWER=1` 或 `[experimental] tower = true`；
- 只能从 main agent、只能用户 `/tower on` 进入；
- 进入校验：
  - 非 main → 拒；
  - 实验关闭 → 拒；
  - 未装配 → 拒；
  - 已有活跃 owner → 忙/有待处理交互则拒；idle owner 则 `exit("takeover")`；
- 进入后把 `TowerInit...TowerStatus` 加入 main 的工具表；
- `isActive = main ∧ flag ∧ assembled ∧ durable state`；
- flag 关闭时所有 tower 工具被 veto。

### 10. 工具

| 工具 | 作用 |
| --- | --- |
| `TowerInit {base?}` | 初始化 `.tower/`，确定基础分支 |
| `TowerPlan {missions[]}` | 创建 mission（title/scope/tasks/context/deps/kind） |
| `TowerSpawn {name, kind worker\|reviewer, model?, mission_id?, review_target?, instructions?}` | 启动 worker/reviewer |
| `TowerComplete {report, key_artifacts?}` | worker 完成报告 |
| `TowerMerge {branch}` | 合并通过门禁的分支 |
| `TowerRebase {mission}` | 重放到基础分支 |
| `TowerTeardown {force?, exclude?, dry_run?}` | 清理 worktree |
| `TowerSend {to,subject,body,...}` | 给 agent 发消息 |
| `TowerInbox {limit?,include_read?}` | 读消息 |
| `TowerFinding {type,title,severity?,summary,...}` | 记录发现 |
| `TowerReview {target,status,merge,findings,checks?,decision,...}` | reviewer 提交审查 |
| `TowerMission {id,status?,note?,blocker?,...}` | 更新 mission |
| `TowerStatus {limit?}` | 状态 |

约束：

- `TowerInit/Plan/Spawn/Merge/Rebase/Teardown` 需要 `callerAgentId === "main"` 且 `tower.isActive`；
- 其余工具任何角色可调（但有各自校验）。

### 11. 角色与权限

- `TowerAgentKind = "worker" | "reviewer"`；
- 两者都 spawn 到 `tower-worker` profile；
- profile 工具表固定，权限模式**固定**（不会被主代理广播覆盖）；
- spawn 时模式设为 `auto`（Never Ask）；
- worker 只能写自己的 worktree：文件访问越界 → 硬 veto；
- reviewer 在主 checkout 只读。

> 文档明确警告：只有 tower 协调者支持 Ask When Needed；worker/reviewer 全自动。

### 12. 持久化 `.tower/`

```
.tower/
├── state.json
├── state.json.lock          # pid=/since=/token=
├── activity.log
├── MISSIONS.md
├── comms/
│   ├── inbox
│   ├── findings
│   ├── reviews
│   ├── missions
│   └── log
└── worktrees/
```

- `state.json` 原子写（tmp + rename）；
- 锁：10s 超时，10ms 轮询；通过 git ref + `kill(pid,0)` 清理陈旧持有者；
- 乐观并发：`runPrepared` / `TowerPreparationConflict`，状态/inbox/reviews 变化时重试；
- 缺 `state.json` 但 `.tower/` 存在 → 明确"state lost"；只有 `TowerSend/TowerComplete` 能恢复最小状态（`recoveredAt`）；
- `.tower/` 自动加入 git exclude。

### 13. Mission 生命周期

状态：

```
planned | active | completed | blocked | paused | merged | abandoned
```

类型：

```
build | survey | artifact
```

- `TowerPlan` 分配 `id=M<n>`、`branch=feat/<slug(title,40)>`、`worktree=wt-<n>`、status `planned`；
- title 必须是可打印 ASCII；
- 未知依赖/重复或已存在分支/整仓 scope → 拒绝；
- scope 重叠 → 自动加依赖或失败；
- `updateMission`：
  - 非 tower 调用者只能是该 mission 的 worker；
  - `merged` 不能通过 patch 设置；
  - `merged/abandoned` 终态，不可重开；
  - `owner/scope/abandon` 只能 tower；
  - `blocker` 强制 `blocked`；
  - `task_drop` 必须给原因并记日志；
  - `completed` 需要 `assertCompletable` + inbox 已读；
  - artifact `completed` 记录 `completionTip/keyArtifacts`。

### 14. Worktree / branch

- `git worktree add <path> -b <branch> <base>`；
- 若基础 checkout 脏：把脏改动提交为 `spawnBase`（分支第一个提交），**不动原 checkout**；
- branch lease 串行化：第二个操作直接失败；
- `TowerSpawn` 也持有 branch lease。

### 15. Worker 简报

`buildPrompt` 给 worker：

- worktree 绝对路径、branch/base、spawnBase 说明；
- mission scope、任务、上下文；
- 之前的 review/acceptance 摘要；
- 沟通与完成协议（build/survey/artifact 三种文案）。

给 reviewer：

- base/target diff 范围；
- mission 意图、作者请求；
- 最近 5 条 review 摘要（≤8KB）；
- 检查清单；
- 必须调用 `TowerReview`。

roster 记录 `worktree/branch/missionId/reviewTarget/spawnedAt`；任务非正常终止标记死亡；`SubagentStarted` 清除。

限流：最多 256 running；provider 429 后暂停 60s。

### 16. 合并门禁（顺序很重要）

`missionGate.ts`：

1. 状态不是 completed → 拒
2. 有 open tasks → 拒
3. 有 blocker → 拒
4. worker busy → 拒
5. worker 未知 → 拒
6. 分支不存在 → 拒
7. 依赖未合并 → 拒
8. survey 有 diff → `read-only-survey`；否则直接关闭（不 merge）
9. base 不是 ancestor → `stale-base`
10. 没有 review → 拒
11. artifact 验收必须 reviewer=tower + mission stamped
12. review `hold` → 拒
13. review status 不是 `clean` → 拒
14. verdict 不是 `merge` → 拒
15. tip 移动 → `stale-tip`
16. artifact `completionTip` 不匹配 → 拒
17. 共享分支上的未 stamp review → 拒
18. 越界文件（picomatch over changedFiles）→ 拒
19. 主 checkout 不在 base → `base-mismatch`
20. base 在合并涉及文件上脏 → `base-dirty`

Review 绑定：`exact-tip | clean-rebase-waived | stale-tip`。
无冲突重建只对 **build** waived 重审，artifact 永不 waived。

### 17. 合并

`merge`：

- 按 branch 找 mission；已关闭 → 明确拒绝；
- 评估门禁；
- survey → noop 关闭，不动 git；
- `mergeNoFf`（拒绝已有 `MERGE_HEAD`；失败时 abort 并验证恢复）；
- 验证 HEAD 包含评估的 tip；
- 若 branch 在合并期间前进 → 返回 `PARTIAL MERGE`，mission **不**标记 merged；
- 否则标记 merged，并报告重叠未合并分支（`conflictsWith`）。

### 18. Rebase / Review / Teardown

`rebaseMission`：

- 只能 tower；
- 非 survey、未关闭；
- worker idle 可观察；
- branch+worktree 已登记；
- worktree 干净、无 rebase in progress；
- worktree 在 mission branch 且 HEAD = branch tip；
- `git rebase <baseTip>`；冲突保留 rebase，mission 阻塞。

`submitReview`：

- 调用者必须是该 target 的 reviewer（或 tower）；
- status/merge 校验；
- artifact 验收要求 `completionTip`、`acceptance_tip`、`reviewedCommit` 三者匹配；
- 每 reviewer 轮次上限 `MAX_REVIEW_ROUNDS`（不收敛拒绝）。

`TowerTeardown`：

- 若被其它活跃会话持有 → 拒；
- 每个 worktree：
  - 已删除/排除/有活跃 roster agent/脏且无 force → 保留；
  - 否则 `git worktree remove --force`；
- 分支与 `.tower/comms` 保留；模式仍 active；
- `dry_run` 只报告。

### 19. Tower 期间的额外 veto

- `TodoList`、`AgentSwarm`、`CreateGoal`、goal resume（`UpdateGoal status=active`）被禁；
- `Agent(resume=...)` 对 tower agent：前台拒；branch lease 期间拒；
- Bash guard 只解析含 `rm` 或 `git clean|reset` 的命令（500ms/10k 节点），保护 `.tower/comms` 与 tower 根。

### 20. 常量

- 文件 IO 并发 8；
- git 并发 4；
- 最大 running 256；
- 429 暂停 60s；
- state 锁 10s；
- review 摘要 8KB；
- title slug 40 字符。

### 21. Tower 不变量

1. 只有用户能进入 Tower。
2. 只有 main 能执行编排类工具。
3. mission 分支/worktree 一一对应。
4. worker 不能写自己 worktree 之外。
5. reviewer 只读。
6. 合并必须过全部门禁；artifact 必须验收。
7. 终态不可重开。
8. 合并期间分支前进 → 不标记 merged。
9. 冲突 rebase 必须保留现场，不能静默丢改动。
10. teardown 不能删除活跃 agent 的 worktree。

### 22. 重实现优先级

1. 先做 Swarm（复用现有子代理 + 批次调度）。
2. Tower 单独排期：它需要 git worktree、mission 状态机、锁、review 门禁、合并事务，工作量远大于 Swarm。
3. 如果产品不需要"仓库级并行开发"，可以永远不做 Tower。

### 23. 证据索引

- `packages/agent-core-v2/src/features/swarm/*`
- `packages/agent-core-v2/src/session/agentRunBatch.ts`
- `packages/agent-core-v2/src/features/tower/{tower.ts,towerService.ts,towerFeature.ts,tools/*,protocol/*,workerProfile.ts,bashGuard.ts,towerRateLimitService.ts}`
- `packages/agent-core-v2/src/session/swarmService.ts`
- `docs/en/guides/tower.md`
