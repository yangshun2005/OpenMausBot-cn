# Issue #1641 — 设计

## 原因

### Android

`Bot.threadGroups()` 按 `listedThreads` 排序（置顶优先，其次 `updatedAt ?: createdAt`，同值时按保存顺序）。注意顺序保留在 `orderedThreads` 中。只有失败的检查才期望列表中出现旧的注意顺序。目标线程全部为 `createdAt = 1` 且无 `updatedAt`，因此可见行的列表顺序为保存顺序 `current, unread, busy, waiting, queued`。`quiet` 是已关闭的 idle，因此折叠。`run` 是 routine 执行，不出现在列表中。

### handoff

`roomHandoffs` 的 fresh work，即使对方另一线程处于 `waiting-on-you`，只要有空闲槽位，下一个 tick 就会变为 `running`。tick 间隔为 250ms，因此从 enqueue 到下一个 tick 为 0〜250ms。检查若在 tick 前读取 `queued` 则通过，负载高的 runner 会看到 10 秒 `running` 而失败。`e8729497` 也一样，#1637 并未改变 busy。

在审批卡片期间等待 fresh work 是 #1128 的邮箱契约。#1278 只将 resume 改为线程粒度，并写明 fresh work 应在卡片期间等待。#1589 允许空闲槽位，是在兄弟线程实际运行时，而审批卡片的检查仍为 `queued`。Hook 不再区分卡片与执行中，这是回归，而竞态掩盖了它。

#1626 将 `expect(status).toBe("queued")` 改为 `expect.poll(...).toBe("queued")`。状态从一开始就是 `queued`，tick 之后不再是 `queued`。若 poll 的首次观测在 tick 之后，就会持续看到 10 秒 `running` 而失败。快速观测在 tick 前的 `queued` 则通过。合并 CI 的 macOS 成功与 Windows 失败、本地加入延迟后 `e8729497` 也失败，都可由该竞态解释。变为 `running` 后再审批，结果只会向另一线程送达一次，已打开的线程不会收到 `MAILBOX_REVIEW`。

### comms

`TypeError: fetch failed` / `read ECONNRESET`。#1637 的 diff 中没有 `server/comms.test.ts`。本次也不改。

## 变更

- Android 的 core 检查同时断言列表顺序与 `orderedThreads` 的注意顺序。`threadGroups` 的 KDoc 不再写「注意会移动行」。
- `TaskRules.tasks` 也采用相同的列表顺序，分为打开带、关闭带、归档带后，再在各带内排序。由于 core 失败导致 app 的单元测试未能执行，因此同步调整 `TaskRulesTest` 的注意顺序期望。KDoc 也一并调整。
- fresh work 的 busy，除目标线程、槽位、群组轮次外，也在其他线程处于 `waiting-on-you` 时等待。resume 按 #1278 不变，不在卡片处等待。兄弟线程处于 `working` 时的空闲槽位启动（#1589）维持不变。
- handoff 检查保持同时线程数 3，验证审批卡片打开期间 ledger 跨 tick 保持 `queued`，且其他线程不进入 `busy`。审批后送达一次的行为保留。
- `orderedThreads` 未被画面调用，因此将其说明改为「未使用的注意顺序辅助函数」。

## 影响范围

| 对象 | 位置 | 类型 | 方针 |
| --- | --- | --- | --- |
| `threadGroups` 的说明 | `android/core/.../ThreadNavigation.kt` | 仅注释 | 与实现对齐。排序代码不变 |
| 列表检查 | `ThreadNavigationTest.kt` 的对应函数 | 测试 | 列表顺序 + `orderedThreads` |
| 面板排序 | `TaskRules.kt` 的 KDoc 与 `TaskRulesTest.kt` | 注释与测试 | 实现仍为 `listedThreads`。期望改为带内的保存顺序 / 更新顺序 |
| handoff 的 busy | `server/index.ts` 的 `recipientAwaitingPerson` | 功能 | 仅 fresh work，在其他线程处于 `waiting-on-you` 时等待。resume 与兄弟线程 `working` 时的空闲槽位不变 |
| handoff 检查 | `server/independent-threads-api.test.ts` 的对应 `it` | 测试 | 即使槽位为 3，审批中仍为 `queued` |

`listedThreads` / `orderedThreads` / `roomHandoffs` 的 busy 实现不变。调用方行为不变。

## 测试

- 将修改后的 vitest 作为隔离 fixture 运行。本地为带 `node:sqlite` 的 Node 22.19.0。CI 与 `package.json` 的 `engines.node` 为 24。该检查的期望不依赖 Node 版本。
- 变异：将 busy 改回「direct 且 bot 为 busy 则等待」时，`running` 的 poll 会失败。确认后还原。
- Android 的 Gradle 需要 JVM 17 以上，本机仅有 JDK 16。该检查在 CI 的 Kotlin job 中确认。断言为完全匹配，因此 `threadGroups` 若回到注意顺序，列表期望会失败；`orderedThreads` 若不再按注意顺序，第二个期望会失败。

## 非目标

iOS UI、`comms.test.ts`、置顶与 `updatedAt` 的产品行为。