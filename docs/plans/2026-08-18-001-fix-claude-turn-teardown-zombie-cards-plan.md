---
artifact_contract: ce-unified-plan/v1
artifact_readiness: implementation-ready
execution: code
product_contract_source: ce-plan-bootstrap
---

# fix: 在 Claude 轮次拆除时终止子进程并丢弃迟到的请求

**来源 Issue：** [milind-soni/OpenMausBot#211](https://github.com/milind-soni/OpenMausBot/issues/211) — "轮次拆除时丢弃权限代理，却未确保子 CLI/MCP 退出——迟到的审批请求变成无用的'僵尸'卡片"

---

## 概述

当 Claude 驱动的轮次结束时，`server/drivers/claude.ts` 的 `settle()` 会拆除权限代理并遗忘该轮次——但从不终止已生成的 `claude -p --resume` 子进程。如果该子进程（或正在执行后台工作的 MCP 孙进程）不自行退出，它稍后可能在其仍然打开的代理连接上发出新的权限请求。由于 `net.Server.close()` 不会触及已打开的套接字，该请求仍会被处理并表现为 `request.opened` 卡片——但该轮次的 `active` 条目（以及 UI 应答所需的代理引用）已经不存在，因此该卡片永远无法被解决。此修复让 `settle()` 无条件终止子进程的进程树，并让代理的 `close()` 真正停止响应任何连接上的请求，无论连接是否已关闭。

## 问题背景

`server/drivers/claude.ts` 中的 `sendTurn()` 每轮生成 `claude` CLI，并将 `createPermissionBroker()` 实例连接到每轮的 unix 套接字（Windows 上为命名管道）。CLI 自身生成的 MCP `ogb` 进程（`server/permission-proxy.ts`）通过该套接字将 `approve`/`ask_user` 工具调用转发给代理，harness 将其渲染为 `request.opened` 卡片。

两个独立的缺口叠加造成了所报告的症状：

1. **拆除时从不终止子进程。** `settle()`（由 `result` stream-json 帧、spawn `error`，或 `result` 之前的退出 `close` 触发）从 `active` 中删除该轮次并关闭代理，但没有任何地方调用 `killCliTree(child)`——该工具仅连接到 `interruptTurn`（用户显式停止）。`-p` 一次性 CLI 进程预期在打印 `result` 后立即退出，但如果它（或执行 `run_in_background` 工作的孙进程）不退出，它就会继续运行，拆除侧没有任何强制措施。

2. **已关闭的代理仍会应答仍然打开的连接。** `createPermissionBroker.close()` 调用 `server.close()`，根据 Node 的 `net` 文档，这会"停止服务器接受新连接"，但**不会**触及已连接的套接字。在 `createNetServer((conn) => {...})` 中注册的 `conn.on("data", ...)` 处理器仍完全连接到任何活跃连接——残留子进程的 MCP 代理仍可发送 `{t:"ask",...}` 消息，处理器会欣然将其加入（仍通过闭包引用的）`pending` Map，并通过 `opts.onAsk(ask)` 转发，发出全新的 `request.opened` 事件。但 `active.delete(threadId)` 已在 `settle()` 内执行，因此 `respondToRequest` 找不到该线程的代理并返回 `"unavailable"`——正是 Issue 所描述的那种无法应答的"自动模式无法应答此请求"卡片。

**本代码库中的先例：** `server/drivers/antigravity.ts` 已经独立于 `active` 跟踪活跃子进程（一个 `children: Set<ChildProcess>`），并通过 `reapChildren(escalate)` 回收它们——先 `killCliTree`，然后在 POSIX 上设置 `escalate` 时经过 2 秒宽限期后可选地发送 SIGKILL。该机制仅从 `stopAll()`/`dispose()`（整个驱动关闭）调用，而非每轮调用，因此它本身无法弥补此缺口——但其升级形态是本修复的正确参考。

## 需求

- **R1**：在 Claude 轮次的每个终止路径上（成功的 `result`、spawn `error`，或 `result` 之前的退出），已生成子进程的整个进程树必须作为拆除的一部分被强制终止——不能任其自行退出，也不能推迟到 `interruptTurn`/`stopAll`/`dispose`。
- **R2**：在该轮次的代理已关闭后到达的权限/问题请求，绝不能成为可操作的 `request.opened` UI 事件，也不能让调用方的 MCP 工具调用永远挂起。它必须通过直接写入仍然打开的连接的系统来源拒绝/应答来解决——与 `close()` 已用于处理进行中 `pending` 请求的形态相同——绝不出现在卡片上，也绝不静默丢弃（见 KTD2：`server/permission-proxy.ts` 没有独立的每请求超时，因此静默丢弃会让该 `tools/call` promise 无限期挂起）。
- **R3**：两个修复都不得改变已正常工作场景的行为：子进程在 `result` 后迅速干净退出、没有迟到请求的轮次，必须产生与今天完全相同的事件序列和 `turn.completed` 载荷。
- **R4**：`killCliTree` 的现有契约（POSIX 上向进程组发送 SIGTERM，Windows 上强制 `taskkill /T /F`）不得退化——本修复复用它，而非分叉它。

## 范围边界

**范围内：** `server/drivers/claude.ts`（`settle()` 和 `createPermissionBroker()`）、其测试文件 `server/drivers/claude.test.ts`，以及共享的假 CLI `server/testing/fake-claude-cli.ts`（新增一种模式以复现 `result` 之后残留的子进程）。

**范围外：**
- Issue 的"第 3 点"（启动时扫描，回收拆除本身从未运行过的崩溃所遗留的孤儿进程）。不同的故障模式——崩溃恢复，而非正常路径泄漏——需要在启动时跨整个应用扫描进程标记。推迟到后续工作。
- `server/drivers/codex.ts` 和 `server/drivers/acp/core.ts` 共享结构上相同的仅在 `interruptTurn` 上调用 `const stop = () => killCliTree(child);` 的模式，很可能有相同的潜在缺陷，但所报告的 Issue 专门针对 `claude -p --resume`。相同形态的修复，不同的表面——推迟到后续工作，不并入本 PR。
- 将 `antigravity.ts` 的 `reapChildren` 重新接线为也每轮运行（它已有自己的驱动特定 `children` 跟踪；改动它与本 Issue 无关）。

### 推迟到后续工作
- 启动时孤儿进程扫描（Issue 的第 3 点）。
- 将相同的每轮 settle 时终止 + 丢弃迟到请求的修复移植到 `codex.ts` 和 `acp/core.ts`。

## 关键技术决策

**KTD1：直接在 `settle()` 内终止子进程，原样复用 `killCliTree`——在此调用点不添加 SIGTERM 后再 SIGKILL 的升级。**
理由：`killCliTree` 已按平台执行全强度终止（POSIX：向进程组发送 `SIGTERM`，与 `antigravity.ts` 非升级路径发送的信号相同；Windows：`taskkill /T /F`，已是强制性的——升级在那里是文档化的空操作）。`server/kill-tree.test.ts` 证明，*当该孙进程留在已生成子进程的进程组内*时，这能可靠地回收孙进程——对于 CLI 自身未使用 `detached: true` 生成的普通 MCP 服务器而言正是如此。`antigravity.ts` 的宽限期后 SIGKILL 升级是为其 `dispose()`/`stopAll()` 路径而存在的，该路径一次回收*一组*可能很多的子进程，可以承担共享的宽限窗口；在此为单个子进程改造每轮 `setTimeout` 会增加一个需要管理（并在测试中考虑）的定时器，而 `kill-tree.test.ts` 并未显示有此必要。如果日后现实中的 SIGTERM 停滞确实成为问题，`antigravity.ts` 的升级可供复制。
**已知限制，明确超出范围：** 如果孙进程刻意重新脱离（调用自己的 `spawn(..., { detached: true })`/`setsid`，逃逸到新进程组——经典的守护进程化模式），对原进程组执行 `process.kill(-pid, "SIGTERM")` 将无法触及它。`kill-tree.test.ts` 现有的孙进程*不会*自我脱离，因此它不能证明覆盖了此情况。本 harness 生成的已知 MCP 服务器（`permission-proxy.ts`、`computer-proxy`、`dweb-proxy`、`agents-proxy`）目前都不这样做，因此这是一个文档化的缺口，而非实际回归——但这意味着如果"后台工作"日后重新脱离，KTD1 无法完全关闭问题背景中的该场景。若成为实际问题则后续处理：添加一个带自我脱离孙进程的 kill-tree 测试，若失败则采用更强的回收策略。
考虑并否决的替代方案：现在就添加相同的宽限期+SIGKILL 升级——被否决，因为在此代码路径中没有证据表明普通 SIGTERM 会留下进程（不像 `antigravity.ts` 已有多个子进程清理来证明其合理性），属于无正当理由的复杂性，且升级对上述自我脱离情况也无济于事（向错误的进程组发送 SIGKILL 仍是空操作）。

**KTD2：在连接的 `data` 处理器内用 `closed` 布尔值守卫迟到请求；对于迟到请求，始终向连接写入系统来源的拒绝/应答——绝不静默丢弃。**
理由：这是满足 R2 的最简修复，且不改变 `close()` 对进行中 `pending` 请求的现有契约。`data` 处理器已在拥有 `pending`、`timeoutMs` 以及现在在 `close()` 顶部设置的 `closed` 标志的闭包内运行；在创建新 pending 条目之前检查它是一行、易于测试的守卫。闭包处理优先于活跃轮次重复 ID 守卫。**响应必须是显式的系统来源拒绝/应答，而非静默丢弃：** 权限请求收到 `deny`，消息为 `OpenMausBot: the turn ended`；问题收到 `answer`，消息为 `OpenMausBot: the turn is ending — wrap up.`。`server/permission-proxy.ts` 的 `waiting` map（CLI 的 `tools/call` 所等待的子进程侧 promise）仅由传入的 `{t:"answer",...}` 消息或连接自身的 `error`/`close` 触发 `dead()` 来解析——`permission-proxy.ts` 中没有任何东西会自行对单个请求超时。代理侧的静默丢弃会让该特定 MCP 工具调用一直挂起，直到其他东西关闭连接，而这正是 R2 要防止的挂起。
考虑并否决的替代方案：(a) 在 `close()` 中跟踪并销毁活跃连接——被否决，因为更多状态却不增加正确性，目标是"绝不创建看似可应答的死卡片"，而非"切断管道"，且按 KTD1 终止的子进程在常见情况下无论如何都会切断它；`closed` 标志加显式回复覆盖了子进程尚未被终止的情况。(b) 复用 `active.has(threadId)` 而非专用的 `closed` 布尔值，因为 `respondToRequest` 已将缺失的 `active` 条目视为"无活跃轮次"——被否决，因为存在顺序隐患：`createPermissionBroker` 在 `sendTurn` 中 `active.set(threadId, ...)` 运行*之前*构造（代理需要存在以构建传给 spawn 调用的 MCP 配置），因此 `active.has(threadId)` 在那个短暂的合法启动窗口内会错误地读到 `false`，拒绝在轮次完全注册之前到达的请求。默认值为 `false` 的专用标志没有这样的窗口。

**KTD3：为测试修复 A 新增假 CLI 模式 `result-then-hang`；修复 B 无需新模式。**
理由：修复 A 的契约关乎操作系统进程，而不仅仅是发出的事件——现有的 `hang` 模式从不到达 `result`，因此无法证明"settle 已发生且进程已消失"。一个打印 `result` 后调用 `hang` 已使用的相同永久空闲 `setInterval` 的新模式，是最小且对称的补充。修复 B 的测试不需要新的 CLI 模式：它像现有"brokers a permission ask"测试一样直接通过套接字驱动代理，只是在 `turn.completed`/close 之后发送第二个 `{t:"ask",...}`——不涉及 CLI 行为。

## 实现单元

### U1. 在每个 settle() 路径上终止子进程的进程树

**目标：** 消除泄漏/残留的进程（R1）。

**需求：** R1、R3、R4

**依赖：** 无

**文件：**
- `server/drivers/claude.ts`（修改 `settle()`，约第 472-489 行）
- `server/testing/fake-claude-cli.ts`（新增 `result-then-hang` 模式）
- `server/drivers/claude.test.ts`（新测试；测试文件路径已存在）

**方法：**
- 在 `settle()` 中，先调用 `broker?.close()` 使闭包成为终态并解析所有当前请求，然后在临时目录清理、活跃轮次删除和 `turn.completed` 之前无条件调用 `killCliTree(child)`。`child` 已在作用域内（在 `sendTurn` 中 `spawnCli` 调用处更早定义）；不要引入新绑定或依赖稍后声明的 `stop` const。
- 当进程已退出时（`child.exitCode !== null || child.signalCode !== null`），`killCliTree` 是空操作，因此对于 CLI 在 `result` 被解析且 `settle()` 运行时已退出的常见情况是安全的——R3 的无回归要求由构造保证成立。
- 向 `server/testing/fake-claude-cli.ts` 添加 `result-then-hang`：发出与默认 `happy` 路径相同的 `system`/`assistant`/`user`/`result` 序列，然后不执行 `process.exit(0)`，而是调用 `hang` 模式使用的相同 `setInterval(() => {}, 1_000)` 以保持存活。
- **测试需要假 CLI 的真实操作系统 pid 来验证它确实已死，而目前没有任何东西暴露它。** `ProviderInstance.adapter` 没有 pid 访问器，`FAKE_CLAUDE_DUMP` 的载荷（`{argv, env, prompt, mcpConfig}`）也不携带。向 `fake-claude-cli.ts` 通过 `writeFileSync(process.env.FAKE_CLAUDE_DUMP, ...)` 写入的对象添加 `pid: process.pid`——复用现有的 dump 机制而非新增——这样测试就能像现有测试从该文件读取 `argv`/`env` 一样读回它。
- 在 `server/drivers/claude.test.ts` 中新增测试：将 `FAKE_CLAUDE_DUMP` 设为临时路径，以 `result-then-hang` 模式运行一个轮次，等待 `turn.completed`，从 dump 文件读回 pid，然后在有界超时内断言底层进程确实已消失（轮询 `process.kill(pid, 0)` 抛出异常，或等效的存活检查——参见 `server/kill-tree.test.ts` 的 `alive()` 辅助函数作为既定模式）。这是今天会失败（子进程仍存活）而修复后会通过的测试。

**遵循的模式：** `server/kill-tree.test.ts` 的 `alive(pid)` 辅助函数（POSIX signal-0 探测），用于断言进程死亡而无需猜测平台特定的超时。

**测试场景：**
- 正常路径：`result-then-hang` 模式——`turn.completed` 仍以正确的 `ok`/`stopReason`/`usage` 载荷触发（与今天相同），且底层进程在不久后可验证地死亡。
- 回归/空操作检查：默认 `happy` 模式——轮次与今天完全一样完成（相同事件序列、相同 `turn.completed` 字段），同时现在也调用了 `killCliTree`（由于进程已退出，应为无害的空操作）。
- 边界情况：`exit-early` 模式（result 之前崩溃）——`killCliTree` 也从 `close` 处理器的 `settle(false, "exit_before_result")` 路径运行；确认对已退出/崩溃的子进程调用 `killCliTree` 不抛出错误。

**验证：** 在本地运行 `pnpm vitest run server/drivers/claude.test.ts`（或仓库文档化的等效命令）；文件中所有现有测试继续通过，新的 `result-then-hang` 测试证明进程被终止。

### U2. 丢弃代理关闭后到达的权限/问题请求

**目标：** 消除无法应答的"僵尸"卡片（R2）。

**需求：** R2、R3

**依赖：** U1（不是硬性代码依赖，但 U1 消除了此竞态的常见触发因素；U2 是请求发出与实际送达终止信号之间剩余窗口的正确性兜底）

**文件：**
- `server/drivers/claude.ts`（修改 `createPermissionBroker`，约第 196-278 行）
- `server/drivers/claude.test.ts`（新测试）

**方法：**
- 在 `createPermissionBroker` 的闭包内添加 `let closed = false;`。
- 在 `conn.on("data", ...)` 处理器的逐行处理顶部，解析 `msg` 之后、重复 ID 处理或创建 `Ask`/`pending` 条目之前：如果 `closed` 为 true，直接向 `conn` 写入系统来源回复——权限请求为 `{t:"answer", id: msg.id, behavior: "deny", message: "OpenMausBot: the turn ended"}`，问题为 `{t:"answer", id: msg.id, behavior:"answer", message: "OpenMausBot: the turn is ending