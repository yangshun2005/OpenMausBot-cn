# Issue #1641 — 让 main 的 CI 恢复绿色

## 目的

将 #1637 合并到 `main`（`6d72b3ac`）后的 push CI
https://github.com/milind-soni/OpenMausBot/actions/runs/35702797939
失败了。此前的 `main`（`e8729497`）是绿色的。要在不改变产品置顶和更新顺序的前提下，让失败的检查通过。

## 失败项

1. Android `ThreadNavigationTest.closedThreadsRemainAccessibleWhenCurrentUnreadRunningWaitingOrManaging`。`threadGroups()` 是按置顶 + 更新顺序排列的，但检查却期望旧的注意顺序 `waiting, busy, queued, unread, current`。550 件中只有这 1 件失败。
2. `server/independent-threads-api.test.ts` 中「peer 的批准之后 coordinated work 保持 queued」这一检查。在 Windows shard 1/4 上持续 `running` 达 10 秒。在 PR head 上，macOS 和 Windows 都以相同的断言失败。合并后的 macOS shard 先读到 `queued` 而通过。
3. Windows shard 3/4 的 `server/comms.test.ts` 出现 `fetch failed` / `ECONNRESET`。该文件在 #1637 和本次均未改动。除非能复现，否则另作处理。

iOS UI 在 PR head 上失败，合并后的 run 中通过。本次不处理。

## 验收标准

- Android 的列表检查在相同时间戳下按保存顺序（可见线程为 `current, unread, busy, waiting, queued`）。注意顺序仍由 `orderedThreads` 保持为 `waiting, busy, queued, unread, current`。
- `TaskRules.tasks` 的检查也采用相同契约。打开的带内按保存顺序或更新顺序排列，不按注意顺序排列。置顶在带首。未读的归档保留在打开的带内，但时间相同时保持保存位置。仅 routine 执行从选择器中隐藏。
- 即使并发线程数为 3，只要 peer 的批准卡片处于打开状态，新的 coordinated work 就跨 tick 保持 `queued`，其他线程不启动。批准后只送达 1 次。维持兄弟执行中时的空槽启动，以及 resume 不被卡片阻塞。批准 socket 在无人值守时不作答，评审正文不进入有人打开的线程。
- 不改变置顶、更新顺序、`updatedAt` 的赋值方式。

## 非范围

- 将 handoff 的 busy 判定恢复为 bot 整体的 busy（#1589 会启动空槽）。
- iOS 的开关检查。合并后的 CI 中已通过。
- `comms.test.ts` 的 `ECONNRESET`。本次差分中没有复现步骤。
- Vercel 的部署授权。