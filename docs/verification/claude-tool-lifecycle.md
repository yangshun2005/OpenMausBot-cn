# Claude 协调与回合级工具

“OpenMausBot: the turn ended” 是审批代理的拒绝，不是网络中断的证据。浏览器能力也会在所属回合结束时过期；越过该边界后的“未授权”不能通过给 worker 永久浏览器凭证来修复。

Claude 驱动保持原生后台任务禁用。原生子代理仍可在当前回合内工作；跨 bot 的异步工作是 OpenMausBot 作业运行时上的 `coordinate_bots` 任务（docs/runtime-contract.md），它负责每个接收者的回合、审批，以及恢复请求者的报告。长时间运行的原生 Bash 命令不再能自动后台化并越过所属回合。这不会改变审批模式，也不会禁用沙箱保护。

Claude 可以发出 `origin.kind = "task-notification"` 的合成结果。该结果不得结算已提交的用户回合，也不得消耗其权限。回归测试夹具发出此结果，然后请求 WebFetch 权限，只有在测试释放文件门控后才完成父级。

小型 `agents` 和 `ogb` MCP 服务器使用 `alwaysLoad: true`，因此首个提示就包含协调和审批工具。其他 MCP 服务器保留正常的延迟加载。这避免了延迟查找依赖；它不会神奇地重连崩溃的服务器，也不会覆盖用户拒绝的工具策略。

来源：

- [Claude MCP server loading](https://code.claude.com/docs/en/mcp#exempt-a-server-from-deferral)
- [Claude background-task setting](https://code.claude.com/docs/en/env-vars)
- [SDK result and background-task messages](https://code.claude.com/docs/en/agent-sdk/typescript)

回归检查：

```sh
pnpm exec vitest run server/drivers/claude.test.ts server/drivers/agents-proxy.test.ts server/harness-mcp-proxy.test.ts
```

回合边界本身——所属回合已被替换的能力即使请求体迟到也会被拒绝并返回 401——由 `server/index.test.ts` 覆盖（“binds profile proposals to the capability's bot and thread and rechecks late bodies”）；每个内部能力，包括浏览器的，都经过同一道门控。

线程从一个回合到下一个回合保留其集成凭证，因此其引擎保持热状态，而不必每个回合重新启动。同一 bot 在同一线程上，拥有相同的授权和审批级别，持有相同的 bearer，且 harness 仅在该线程的某个回合运行期间认可它。停止、停滞、删除 bot 或线程、更改授权或审批级别，或提供方重载，都会给下一个回合新的 bearer。计算机每个回合获得新的 bearer：其工具桥接到一台可能在回合之间停止的机器（空闲的 Local VM、Auto 启动的 VPS），而热引擎不会重启机器已消失的桥接。每回合限制（创建的 bot、打开的线程、房间发帖，以及不在发起委托的回合内检查该委托）由 harness 按回合计数，而非由代理计数，因为热引擎会跨回合保留代理。

```sh
pnpm exec vitest run server/session-credentials.e2e.test.ts server/post-to-room.test.ts server/vps-routing.test.ts
```

被引导进运行中 Claude 回合的消息，仅在尚未开始的模型调用之前被并入。落在该回合最后一次调用期间的内容会被排队，并作为 CLI 的下一个原生回合运行，在同一进程、同一工具上。驱动会保持该回合打开，直到排队的消息得到应答：一次 `turn.completed`，bot 全程忙碌，且该回合的内部工具通行证在此之前一直有效——绝不在运行中的延续下被撤销，也绝不重新签发。从 Claude Code 2.1.282 起，驱动以 `--replay-user-messages` 运行 CLI：CLI 在模型调用接收每条 stdin 消息时，回显该消息及其发送时的 uuid。在工具调用期间写入的引导会在该工具结果之后立即回显；在该回合最后一次模型调用期间写入的引导，只会在该回合的 `result` 之后、作为其自身回合开始时回显。当 CLI 报告 `queued_turn_count` 大于零时，或当某条引导尚未被回显时，`result` 会被保持（2.1.282 对等待 stdin 的消息报告 0，因此 0 不决定任何事）。在较旧的 CLI 上，没有任何信息表明引导是否被并入，因此自上次保持以来的每条引导都会保持结果，而被并入的那条会付出下面的宽限期代价。若 2 秒内没有 `init` 跟随（消息毕竟被并入了），或若延续的 `init` 之后有 30 秒静默，则被保持的结果作为该回合的结果成立。队列引导按钮走同一个 `steer()`，因此落在最后一次模型调用的引导会以相同方式将该回合延长到其延续。（队友的问题不再落入运行中的回合：它是作业运行时上的 `coordinate_bots` 咨询，在其自身的工作线程中应答。）CLI 的 `total_cost_usd` 是累计总额，因此该回合的成本是最新数字；每回合 token 用量累加。

```sh
pnpm exec vitest run server/drivers/claude.test.ts -t "could not fold"
pnpm exec vitest run server/drivers/claude.test.ts -t "queued_turn_count|tool result did not take in|echoed as taken in|does not echo|held window|stays silent|steered continuation|whole grace"
pnpm exec vitest run server/steer-e2e.test.ts -t "internal tool pass"
```

对于端到端验证，启动 [README.md](README.md) 中描述的隔离夹具，然后遵循 [Chat turns](chat-turns.md)。绝不要使用客户正在运行的应用来创建测试 bot、批准请求或轮换工具。这些夹具检查不能证明客户的真实提供方账户或网络是健康的。如果故障仍然存在，请索取他们的应用版本、Claude CLI 版本，以及一份脱敏的诊断导出；不要索取凭证。

## 重建的对话

编辑消息或重建上下文必须启动新的 Claude 会话，即使其旧进程仍处于空闲。普通后续会继续复用或恢复当前会话。重置后的瞬时重试恢复替代会话，而非被放弃的那个。

```sh
pnpm exec vitest run server/drivers/claude.test.ts server/turn-context.test.ts server/resume-recovery.test.ts
pnpm exec vitest run server/delta-context.e2e.test.ts -t "resets Claude's native context"
```

第二条命令通过共享验证启动器启动一个一次性服务器和假 CLI。它创建一个对话，用 `control-omb edit` 编辑最后一条用户消息，并检查原生启动、重置日志、活动分支重放及后续恢复。被放弃的请求/回复标记不得进入替代提示。它不演练真实的 Claude 账户，也不声称已实现自动上下文压缩。