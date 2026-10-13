# 规范：无中断引导 + 双击 Enter 手势（codex）

状态：已冻结。标准 (d) 于 2026-09-16 经用户裁定修订：房间和群组与 1:1 线程享受相同的引导处理，而非仅排队。

## 协议结论（codex-cli 0.154.0）

原生回合中引导确实存在；无需杀死或重启子进程。

- 真实二进制文件：`/opt/homebrew/bin/codex` 是一个 opencodex 垫片，它 `exec` 执行 `/opt/homebrew/bin/codex.opencodex-real` -> `/opt/homebrew/Caskroom/codex/0.154.0/bin/codex`（原生 Rust 构建，Homebrew cask；不是 npm 包）。
- 对 `codex.opencodex-real app-server` 的实时 stdio 探测（记录：本机 /private/tmp/omb-steer-probe2.log）：服务器标识为 `omb-probe/0.154.0`。`turn/steer`、`turn/interrupt` 和 `thread/queue/add` 均作为 JSON-RPC 方法存在（空参数 -> 错误 `-32600` "Invalid request: missing field `threadId`"；未知方法会返回 `-32601`）。使用格式正确的 threadId 时，`turn/steer` 可正常派发，无参数错误。
- 二进制字符串（对 Mach-O 执行 `strings`）显示了周边协议：方法 `turn/start`、`turn/steer`、`turn/interrupt`、`thread/queue/add`、`thread/start`、`thread/resume`、`thread/read`；错误变体 `ActiveTurnNotSteerable { turnKind }`、`NotIdle`、`PendingTriggerTurn`、`NoActiveTurn`、`ExpectedTurnMismatch`；通知 `ThreadQueueChanged`、`TurnAborted`、`TurnStarted`、`TurnComplete`、`ItemStarted`、`ItemCompleted`；每个线程有一个持久化的 `queued_items` sqlite 表。
- 上游 app-server 协议文档（openai/codex codex-rs app-server-protocol，v2 JSON schema ~0.153.x）将 `activeTurnNotSteerable` 描述为："在当前活动回合不可引导时提交了 turn/start 或 turn/steer，例如 /review 或手动……"——即两个方法都会介入活动回合。
- 语义：引导会重定向正在运行的回合。Codex 在安全边界处中止进行中的模型流（通过协议进行回合级中止），并将新输入合并后继续同一回合/会话。app-server 子进程和原生线程保持存活；没有任何 SIGTERM 或重启。OpenMausBot 的 codex 驱动已经每个回合运行一个 app-server，因此 `turn/steer` 会搭载拥有正在运行回合的实时请求实例。

### Adapter.steer 契约（三态）

`steer(threadId, text)` 只会解析为以下之一：

- `"steered"` — 引擎已将输入接受到实时回合中。
- `"refused"` — 可证明未送达（无实时回合、显式 RPC 拒绝、stdin 写入失败）。调用方可将这些内容排队到后续回合，而无需担心重复执行。
- `"indeterminate"` — 已送达，但结果未知：请求发送后 RPC 超时、传输中断，或回合在应答在途时已结束。调用方绝不能重新排队这些内容；引擎可能已在运行它们，重放会导致执行两次。调用方只记录一次（记录 + 队列已结算）。

Phase 2 需确定的开放项（非阻塞）：除 threadId 外的确切 `turn/steer` 参数字段（输入项、审批/effort 透传、turn-id 守卫）、其响应/事件序列（`turn/aborted` -> 恢复的项？），以及当 `NoActiveTurn` 与完成竞争时的回退（回退到队列）。

## 验收标准（每项用户需求一条）

(a) 在繁忙且支持引导的 1:1 线程上，Enter 仍然排队且不增加延迟，短窗口内第二次 Enter 会将刚排队的消息引导到正在运行的回合中。普通发送保持无延迟；该手势可通过现有引导区域字符串被发现。
(b) 引导绝不中断或杀死正在运行的机器人：引导或队列的任何路径都不得在回合中途调用 killCliTree（或以其他方式对子进程发送 SIGTERM）。turn/steer 内部的回合级协议中止在范围内且符合预期；进程级杀死不在范围内。
(c) SIGTERM 运行时错误（"codex exited null (signal SIGTERM) before turn/completed; no stderr after the last app-server output"）从引导、队列和显式 Stop 均不可达。Stop 首先使用优雅的协议中断（`turn/interrupt`），仅在升级时杀死，且关闭处理器不得将有意停止的原始 SIGTERM 作为 runtime.error 暴露（修复 stopRequested 竞态）。
(d) 房间和群组的行为与 1:1 线程一致。Enter 在房间中仍然排队（房间发送本身绝不实时引导）；窗口内第二次 Enter 会将刚排队的头部消息引导到正在运行的房间回合中；房间的 Steer 芯片引导而不中断。正在运行的驱动无法引导的房间，获得与不支持引导的 1:1 驱动相同的回退处理和诚实的打断文案。

## 行为矩阵（待测试行）

| 操作 | codex（支持） | claude（已支持引导） |
| --- | --- | --- |
| Enter（繁忙 1:1） | 排队，不增加延迟 | 排队，不变 |
| 双击 Enter（繁忙 1:1） | 第二次 Enter 通过 turn/steer 引导排队消息；子进程存活；无 SIGTERM 错误 | 通过现有 adapter.steer 执行相同手势 |
| Steer 芯片 | 非中断引导（turn/steer） | 非中断引导（现有） |
| Stop | 优雅 turn/interrupt；干净结束，无 SIGTERM runtime.error；仅在升级时杀死 | 不变 |
| Enter（繁忙房间） | 排队；不尝试实时引导，不增加延迟 | 排队；不变 |
| 双击 Enter（繁忙房间） | 第二次 Enter 通过 turn/steer 将排队的头部引导到正在运行的发言者回合中；子进程存活；无 SIGTERM 错误 | 通过发言者现有 adapter.steer 执行相同手势 |
| Steer 芯片（房间） | 当正在运行的发言者支持时非中断引导；不支持时诚实的打断回退 | 非中断引导（现有） |
| Stop（房间） | 优雅中断；不变 | 不变 |

## 测试计划（每条标准一个测试）

(a) Composer/驱动测试：Enter 排队；窗口内第二次 Enter 将排队消息路由到 adapter.steer，队列清空（不启动新回合，不派发中断）。
(b) 驱动测试：引导通过同一子进程上的 turn/steer 到达正在运行的回合；未调用 killCliTree/terminate；子进程之后仍发出回合事件。
(c) 驱动测试：显式 Stop 发送 turn/interrupt，关闭处理器报告干净结束（无提及 SIGTERM 的 runtime.error）；仅在中断失败/超时后升级杀死。队列排空测试断言无 SIGTERM 路径。
(d) UI 测试：双击 Enter 窗口在正在运行的发言者支持引导的繁忙房间上打开，房间芯片路由到群组引导端点，而非 interruptGroup。E2E 测试：排队的房间头部被引导到正在运行的 codex 发言者回合中（同一子进程，无中断，引导消息已记录，尾部保持排队）；不支持引导的房间通过端点保持其队列，并在 Stop 后排空。

## 边界

Phase 1 仅添加本规范：无驱动、服务器或 UI 更改。