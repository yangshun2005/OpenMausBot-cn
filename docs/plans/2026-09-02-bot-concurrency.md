# Bot 并发：成员扇出、轮次串行、唤醒排队

状态：决策记录（2026 年 9 月 2 日）。界定 `2026-08-22-channels-plan.md` 中"并行运行时"部分，对照 Grok Bot 的实际行为。

更新（9 月 9 日）：下文的直接对话决策已被[独立线程与项目](2026-09-09-independent-threads.md)取代。早先的审计仍是有用的历史记录；多 Bot 频道轮次仍采用此处描述的保守的每 Bot 通道。

## Grok Bot 的行为（已对照 docs.x.ai/grok-bot 与实测报告核实）

- **成员扇出。** 一个 Bot 可以同时待在多个群聊中；xAI 自己的指南就在各项目频道中复用同一个 Coder/Researcher/Writer。Bot 是人格，不是副本。
- **执行按 Bot 串行。** 一个 Bot 只有一个对话和一个*当前轮次*。用户的私信"优先于后台工作，并可重定向当前轮次"；"立即停止"会结束它。Bot 之间的消息是异步*唤醒*（"接收方 Bot 被唤醒、处理请求，并可稍后回复"）。在群组中，"主持人依次唤醒每个成员"。没有队列 UI，也没有第二条通道：给忙碌的 Bot 发消息会重定向它。
- **并行来自更多 Bot。** 文档中所有关于并行的说法都是复数（"多个 Bot 可以同时工作……各自拥有自己的屏幕"），从不是一个 Bot 同时做两件事。每个屏幕同一时间只跑一个计算机使用任务；每个用户共享一台计算机（不是安全边界）。
- **例程是所属 Bot 通道上的唤醒**（它们在该 Bot 的对话上下文中运行——xAI 员工建议使用专用的例程 Bot，以免调度排在一个长对话轮次之后）。
- 记忆按 Bot 隔离，文件按用户隔离；未公布任何并发写入语义——产品把冲突推给"每个阶段单一所有者"。

## OpenMausBot 的决策

1. **默认保持每 Bot 串行；成员扇出；忙碌的 Bot 稍后被*唤醒*，绝不跳过。** 普通房间轮次加入与目标相同的有限等待（`waitForGroupGoalBot` → 共享成员等待）：在别处忙碌的响应者空闲时轮到自己，并显示"正在完成另一个对话"的提示；超过等待上限后本轮继续推进，并显示说明这一点的标签。房间上的 Stop 会释放等待。这为聊天室消除了"成员忙碌 → 跳过/失败"的脆弱性，正如 #702 为目标所做的那样。
2. **用户的私信优先于后台工作**（Grok Bot 的规则）：引导队列已将其保留到下次稳定；Stop 仍是重定向手段。
3. **单个 Bot 的并行轮次是显式选择加入，而非默认** —— channels 计划的轮次租约设计（`maxConcurrentTurnsPerBot > 1`，对 Bot 的计算机及任何共享工作文件夹加独占租约，记忆单写者）留作后续（该计划的 PR 3/4），以驱动符合性为前提，且默认关闭。UI 绝不能为串行的驱动或资源承诺并行工作。

## 任何选择加入并行之前的前置条件（来自代码库审计）

- `unattendedBots` 按 Bot 为键，基于一次一个轮次的假设。在两个活动轮次下，一个有人值守的聊天轮次会清除并发运行的 webhook 轮次的无人值守标记，并使其获得自动批准。先按线程重新设键——这是权限修复，不是 UX 修复。
- `MEMORY.md` 写入走原子 temp→rename 辅助函数（#703）；在一个 Bot 的两个轮次都能"记住"之前，需要每 Bot 单写者路径。
- 409 文本"already working"在三条重试路径（委托唤醒、连接器恢复、密钥恢复）中被当作控制流做字符串匹配。在改动任何门控之前，先引入类型化错误码。
- Stop 路由（`activeGroupTurnForBot`）是首个匹配扫描；它必须按 (bot, thread) 解析。
- 驱动无需改动：Claude、Codex 以及所有 ACP 驱动已按线程隔离轮次。

开放的产品问题：停在人工审批卡（`waiting-on-you`）上的轮次算作忙碌。在每 Bot 通道计数下，它会占用一个槽位；UI 必须说明这一点，否则"2 个中 1 个忙碌"会被读成 bug。

来源：docs.x.ai/grok-bot（bots、chat-and-collaboration、computer-and-apps、faq、overview、skills-routines-and-automations、teams-and-enterprises）、x.ai/bot/guides/how-i-run-multiple-teams-of-grok-bots、brianlovin.com（第一印象）、flaviocopes.com/grok-bot（Matt Palmer）、cursor.com/docs/grok-bot。对比：Buzz（每个频道一个在途提示，agents × channels 会话）、OpenClaw（每会话通道，跨会话 `maxConcurrent`，steer/followup/collect/interrupt）、Hermes kanban（`max_in_progress_per_profile`）、Claude Code agent teams（一个队友 = 一个进程，一个任务）。