# SIGTERM 审计 — codex 子进程终止路径

范围：`feat/steer-vs-queue` 分支上所有可能终止或杀死 codex app-server 子进程的路径，对照 [steer-vs-queue.md](./steer-vs-queue.md) 的 SPEC 标准 (c)：原始运行时错误 `codex exited null (signal SIGTERM) before turn/completed` 必须无法从 steer、queue 和显式 Stop 触发。

## 终止点

所有进程终止都经过 `killCliTree`。只有三个导入点能触及 codex 子进程：

| # | 位置 | 触发条件 | 轮次中？ | 类别 | 证据 | 残余风险 |
|---|------|---------|-----------|-------|----------|---------------|
| 1 | Catalog 探测回收 | 模型目录查询完成 | 否 — 独立探测进程，绝非轮次子进程 | 终止（回收已应答的探测） | server/drivers/codex-catalog.ts:87 | 对 (c) 无风险 |
| 2 | 认证/登出探测回收 | 身份探测应答或超时（1s） | 否 — 独立探测进程 | 终止（回收） | server/drivers/codex-identity.ts:44,52 | 对 (c) 无风险 |
| 3 | 轮次子进程漏斗 `terminate()` | 轮次 app-server 的唯一终止路径 | 是（Stop 语义） | 终止，但见下方调用方 | server/drivers/codex.ts:783 | 见各调用方行 |

### 轮次子进程漏斗的调用方（`terminate`，codex.ts:783）

| 调用方 | 触发条件 | 轮次中？ | 优雅终止还是杀死 | 证据 | 残余风险 |
|--------|---------|-----------|------------------|----------|---------------|
| `stop()` 升级 | 任何 interruptTurn：Stop 按钮、群组/房间取消、停滞看门狗、房间截止时间、例程取消、发言者取消、关闭 | 是 | 先优雅：`turn/interrupt` 加宽限窗口（默认 750ms；测试中用 `FAKE_CODEX_INTERRUPT_GRACE_MS`），然后杀死 | codex.ts:795-816（升级在 :813） | 卡死的服务器在宽限窗口后仍会因 SIGTERM 而死 — 但 close 处理器会将其静默结算为 `interrupted` |
| `settle()` → `stop()` 轮次后清理 | 轮次正常完成 | 否 — 在 `turn.completed` 之后 | 杀死（每轮 app-server 从不自行退出） | codex.ts:818-833 | 若杀死失败，会触发另一个诚实的错误（"did not shut down"），绝不会是 SIGTERM 那个 |
| 重试重启，确认前退出 | 子进程在轮次被确认前短暂退出 | 是，但尚未流式输出（`codexTurnId === null`） | 杀死已死/卡死的尝试；先设置 `abandoned = true` | codex.ts:1290-1296 | 设计上静默：close 处理器对已放弃的尝试立即返回 |
| 重试重启，握手失败 | initialize/kickoff 上的瞬时 5xx/过载，无流式输出 | 同样的守卫（`sawStreamDelta === false`） | 杀死失败的尝试，先标记 abandoned | codex.ts:1469-1487 | 同样的静默结算逻辑 |

## 所有 `interruptTurn` 调用方（全部汇入 `stop()`，优雅优先）

server/index.ts：显式 Stop 端点 :1023；停滞看门狗 :3359；直接轮次分发中止 :6134；群组停止 :6404 和 :6439；例程取消门面 :6525（routines.ts:1111,1306,1361）；房间轮次截止时间 :7804；与适配器认证竞争的停止 :7869；发言者取消批次 :9812；群组取消批次 :13402、:14032、:14129、:14986。还有引擎切换/实例移除：persistProviderInstance → `registry.dispose` :9838；provider 重载和服务器关闭 → `registry.disposeAll` :9863 和 :17028。全部到达 codex `dispose`（codex.ts:1594-1598 → 对每个活跃轮次调用 `stop()`）或适配器的 `interruptTurn`（codex.ts:1571-1573 → `stop()`）。

线程删除（`deleteBotWithLifecycle`，index.ts:6647）在例程、群组轮次、Local VM 轮次或 Boat 配置的忙碌轮次活跃时拒绝执行（409 "stop this bot's work…"）；普通忙碌轮次在其实例被 dispose 时通过同样的优雅优先 stop 拆除。没有删除路径直接调用 `killCliTree`。

## Close 处理器归因（codex.ts:1239-1256）

1. `abandoned` → 静默返回（:1240）— 已退役的重试尝试无法上报。
2. `state.settled` → `void stop()` 并返回（:1241-1244）— 轮次后清理；不发出错误。
3. `stopRequested` → `settle(false, "interrupted")`（:1250-1252）— 有意 Stop 的杀死静默结算；SIGTERM 运行时错误无法从 Stop 触发。竞态已关闭，因为 `stop()` 在入口处（:796）同步设置 `stopRequested = true`，早于其第一个 await，所以 close 事件绝不可能在标志与 interrupt 之间到达。
4. 否则 → 诚实的崩溃归因（:1254 起），包括外部杀死子进程的信号文本。仅能从 steer/queue/Stop 之外触发（OOM、用户 `kill -9`），这是正确的。

## Steer 和 queue 路径不含终止调用（静态证明）

- `steerActiveTurn`（codex.ts:840-858）发出一个 `turn/steer` 请求，任何守卫或拒绝都返回 false — 无 `terminate`/`killCliTree` 调用。
- queue/steer 端点（index.ts:14707-14751）只调用 `adapter.steer()`；拒绝时恢复队列（`restoreHeldSteeredQueue`），并可能将其作为新轮次排空。无 interrupt 或 kill。
- 队列排空（index.ts:5055-5075）订阅 `turn.completed` 并调用 `startTurn`；从不触及前一个子进程。
- `rg -n "kill|interrupt|stop\(" server/steer-queue.ts` 无匹配（只有注释/警告）。
- 房间 queue/steer 端点（index.ts:13355-13437）遵循同一契约：原子地提取通道队列（holdChannelQueue），对拥有运行中房间轮次的引擎调用 adapter.steer()，并恢复或结算持有的队列（restoreHeldChannelQueue / settleHeldChannelQueueHead），无任何 interrupt 或 kill 调用。运行中引擎无法 steer 的房间保留其队列，与无能力的 1:1 引擎完全一致。

## 标准 (c) 测试证明

- Steer 从不杀死：codex.test.ts:1411（子进程存活 + 无 runtime.error）和 :1438（拒绝保持子进程存活）。
- Queue 从不杀死：codex.test.ts:1454（拒绝 steer → 队列 → 排空轮次；对 `killCliTree` 的间谍既看不到轮次子进程也看不到其排空后继，两个子进程在轮次中均验证存活）。
- Stop 是优雅的：codex.test.ts:1491（观察到 `turn/interrupt`，无信号错误）。
- 升级静默结算：codex.test.ts:1507（卡死的服务器在宽限窗口后被杀死，`stopReason: "interrupted"`，无 runtime.error）。
- 端到端：steer-e2e.test.ts:294/:335/:368（实时 steer、无 SIGTERM 错误的 Stop、无能力引擎上保留队列）。