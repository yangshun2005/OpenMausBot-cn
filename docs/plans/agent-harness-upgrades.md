# OpenMausBot harness 实现计划

- 状态：提案中
- 最近仓库审查：2026-08-17
- 跟踪产物：添加本文档的 Pull Request

## 摘要

对 `server/` 和 `src/` 的十七项改动，分为三轮，外加一份推迟清单。不引入任何外部 agent harness 作为依赖。

本计划取代了早先的草稿。那份草稿通过阅读其他 harness（pi、agent-orchestrator、deepseek-harness）并对比 OpenMausBot 缺什么来构建，结果画出了一张"什么是可能的"的好地图，却是一张"实际缺什么"的差地图。以下条目来自阅读本仓库。其设计仍借鉴了那些项目中确实优秀的部分，已验证的事实保留在[上游参考](#上游参考)中。

**重新定位改变了排序。** OpenMausBot 本身就是一个 harness。agent CLI 自己的会话文件是一种*优化*——一个按引擎划分的缓存，恰好保存了某台机器上某个引擎的上下文。规范、持久、跨引擎的记录是 `server/store.ts`。因此，凡是该记录必须变成模型上下文的路径都是承重的，而今天每一条都被限制在 40 条文本消息或直接损坏。

第 1 轮修复用户实际遇到的问题。第 2 轮为 harness 提供它目前缺乏的保证。第 3 轮是前两轮解锁的工作。

## 已验证的发现

以下所有内容均于 2026-08-17 对照仓库核查。行号会随 `server/index.ts` 增长而漂移；符号名是稳定的参照。

**上下文重建被限制或缺失。**

- `server/index.ts:806` 将面向模型的 transcript 构建为 `activePath(threadId)` 过滤 `kind === "text"`，然后 `.slice(-40)`。
- 只有 `server/drivers/grok.ts` 读取 `SendTurnInput.transcript`。CLI 和 ACP 驱动通过各自的原生会话恢复。
- `server/index.ts:817` 仅为非 grok 驱动构建内联重放，且**仅当**设置了 `rewound` 时。
- `rewound: true` 恰好设置在两处——`server/index.ts:2058` 和 `:2075`——都是分支/编辑处理器。
- `resumeCursors` 是 `Record<instanceId, cursor>`（`server/store.ts:117`），在 `server/index.ts:979` 读取为 `rewound ? undefined : task.resumeCursors[instanceId]`。
- **后果：** 在线程中途切换 bot 的模型时，新实例找不到 cursor，不会设置 `rewound`，也不是 grok——因此新引擎只收到最新一条消息，别无其他。宣传的"在对话中途切换 bot 的模型"会静默丢弃对话。
- **第二个后果：** 重放过滤器仅限文本，因此所有工具调用和结果都被丢弃。重建的上下文显示人们在对话，却看不到任何工作证据。

**输入仅限文本。**

- `SendTurnInput` 携带 `text: string`，没有图像或文件字段。
- 消息类型为 `"text" | "options" | "activity" | "screen"`（`server/store.ts:53`）；`screen` 是 bot 自己的截图输出。
- Bot 可以向用户发送图像。用户无法回发一张。

**主对话路径没有重试。**

- `server/drivers/claude.ts`、`codex.ts`、`grok.ts` 中重试出现次数为零。
- 重试存在于 `server/boat.ts`、`server/computer-proxy.ts`、`server/drivers/acp/core.ts`、`server/webhooks.ts`、`server/drivers/antigravity.ts`。
- 外围系统有韧性；对话路径没有。

**没有存活检测，且只有一个粗粒度状态。**

- `busy?: boolean`（`server/store.ts:175`）就是整个运行时状态模型。
- `server/index.ts:1022` 在回合结束时清除它；`server/index.ts:1296` 在启动时为每个 bot 清除它——这是驱动静默死亡时的变通方案。
- 36 个文件中共 221 处引用 `busy`，其中 10 处是测试。

**两条独立的写入路径。**

- `server/store.ts:360` 的 `writeFileAtomic(BOTS_FILE, ...)` 和 `server/harness/bus.ts:32` 的 `EventBus.publish`。
- `server/index.ts` 包含 139 处 `store.` 调用和 39 处 `broadcast(` 调用。

**自动化有门控但不持久。**

- `botState` 返回 `"ready" | "busy" | "missing"`（`server/index.ts:1036`、`:1065`）；routine 和 webhook 共享一个排队执行器（`server/index.ts:1060`）；peer `ask_bot` 路径返回 `{ busy: true }`（`server/index.ts:1432`、`:1475`）。没有 missing 门控。
- `server/delegations.ts:31` 声明委托队列*"不持久化到任何地方——服务器重启会丢弃委托"*。Routine 和 webhook 有持久回执；委托没有。

**审批以散文形式作答。**

- 未应答请求路径（`server/drivers/claude.ts:122`）向模型发送一条自然语言指令以跳过该操作。其意图是 fail-closed，但不是调用方可以分支的类型化结果。
- `request.opened` 携带 `tool` 和 `summary`，作为已作为工具项流式传输的信息的自身副本；两者可能漂移。

**已存在——不要重建。**

- 消息树：`parentId`（`server/store.ts:69`）、`activePath`（`:480`）、`branchMessage`（`:519`）、`activeLeafId`。
- UI 中可用的版本切换器：`ChatView.tsx:388-403` 渲染 `1/3` 及前后箭头，由 `switchBranch` 和 `/active-branch` 端点支持。分支可发现且可达。
- 端口与适配器：`server/contracts.ts` 是端口接口，`server/drivers/` 是适配器，未知驱动降级为不可用的影子快照。
- `RuntimeEventBase.raw` 携带原生协议载荷。没有任何地方暴露它。
- `ProviderInstance.generateText`（`server/contracts.ts:245`）——摘要接缝已存在。
- 成本由三个主驱动填充，并通过 `turn.completed.cost` 和 `thread.token-usage.updated` 到达应用。UI 丢弃了它。

**任何地方都不存在。**

- 跨线程搜索。
- 每次调用的工具截止时间或重复调用检测。
- `ModelCatalog.options` 上的 `contextWindow`（`server/contracts.ts:223`）。

---

# 第 1 轮——损坏或缺失

## 1. 修复模型切换时的上下文丢失

**做什么。** 在线程中途切换 bot 的引擎时，必须为新引擎重建上下文，而不是让它空白启动。

**为什么。** 已验证损坏，且这是首页功能。

**设计。** 在派发时，根据*谁在任务上运行了上一个回合*来决定新鲜度，而不是根据此实例是否持有 cursor——cursor 只证明存在覆盖线程某前缀的会话，而此后另一个引擎运行的每个回合都不在其中。在派发时跟踪 `TaskRecord.lastInstanceId`，当它与所选实例不同时重放（`server/index.ts:817` 处 `rewound` 已走的同一分支）。该字段存在之前的任务回退到 cursor 映射：唯一且属于我们的 cursor 继续恢复，任何有歧义的情况都重放。在第 7 项之前发布此项；40 条消息的文本重放总比没有上下文好，而重建路径正是第 7 项稍后接入的地方。
*（#180 中的实现：`server/turn-context.ts` 中的 `engineIsFresh()`、`store.markTaskDispatched()`；先尝试了朴素的缺失 cursor 规则，在手动测试中未通过 A → B → A 场景。）*

保持重建标记与 `rewound` 区分。`rewound` 意味着"可见分支已改变"；此标记意味着"此引擎在此没有会话"。它们今天重合，但不会总是重合。

**完成标准。** 一个有历史的线程从 Claude 切换到 Codex 后，首次回复能明显知道之前发生了什么。

## 2. 附件——用户到 bot

**做什么。** 用户可以向 bot 发送图像和文件。

**为什么。** `SendTurnInput` 仅限文本。Bot 发送截图；用户无法回发一张。每个底层 CLI 都已接受图像输入。

**设计。**

- `SendTurnInput.images?: Array<{ mediaType: string; data: string }>`（base64），以及一个 `files?` 变体，如果基于路径的交接更适合 CLI 驱动。
- 与 `screen` 并列的用户附件消息类型，随消息存储并在 transcript 中渲染。
- 编辑器：粘贴、拖放和文件按钮。
- 按驱动编码——Claude 和 Codex 在其流格式中接受图像；ACP 有自己的内容块。
- `capabilities.images?: boolean`。无法接受图像的驱动声明 `false`，编辑器为这些 bot 禁用附件，而不是在发送时失败。
- 限制大小和数量；在驱动支持的情况下，大文件应引用路径而非内联 base64。

**完成标准。** 粘贴到编辑器中的截图在每个声明支持的驱动上到达模型，出现在 transcript 中，并在重新加载后保留。

## 3. 主驱动中的自动重试

**做什么。** 在 `claude.ts`、`codex.ts`、`grok.ts` 中对瞬时故障进行有界退避重试。

**为什么。** 今天零重试。一个 529、一个速率限制或一个断开的 socket 就会终止回合——包括无人看管时由 routine 和 webhook 启动的回合。

**设计。**

- 区分瞬时（429、5xx、过载、连接重置）与终止（认证、无效模型、配额耗尽）。终止故障不得重试。
- 带抖动的指数退避、小的尝试上限和硬性上限。
- 发出 `turn.retrying` 运行时事件，让 UI 显示它而不是看起来卡住。
- 中止路径：用户在重试期间中断会取消整个回合。
- 重试不得重复部分流式传输的助手消息——部分输出后的重试丢弃部分内容并重启回合。

**完成标准。** 模拟的 529 产生可见重试并完成回合而非失败，且无效 API key 立即失败而不重试。

## 4. 跨线程搜索

**做什么。** 搜索每个 bot 的消息和活动。

**为什么。** 一队 bot、持续数周的线程，却无法找到任何东西。

**设计。**

- 基于 store 的服务端端点：查询、可选 bot 过滤、可选类型过滤、带线程和消息 id 的分页结果。
- 从对内存消息加每线程 NDJSON 的线性扫描开始；只有在实测太慢时才加索引。
- 结果链接到消息，这意味着选择包含它的分支——`switchBranch` 已能做到。
- 也搜索 `activity` 类型，而不仅是文本，这样"哪个 bot 运行了那次迁移"可以回答。

**完成标准。** 三周前在分支线程中说过的一句话可被找到，点击结果落在其上且选中了正确的分支。

## 5. 成本与用量可见性

**做什么。** 暴露已在到达的每回合和每 bot 成本及 token 数据。

**为什么。** 数据从三个主驱动在线上传来却被丢弃。一个无人看管整夜运行 agent 的应用必须显示花费。

**设计。**

- 回合上的每回合成本、每线程累计、每 bot 累计。
- 区分计量计费与订阅计费。Claude on Max 和 Codex on ChatGPT Plus 报告的是名义成本；在那里显示美元数字会误导。标注来源："在你的订阅上"与实际金额。
- 个人资料中的每 bot 总计，以及某处可达的机队总计。

**完成标准。** 用户无需打开提供商仪表盘即可回答"我的哪个 bot 在花钱"。

---

# 第 2 轮——harness 保证

## 6. 存活回收器

**做什么。** 轮询活跃驱动进程并推导出丢失状态。

**为什么。** 驱动在没有终止事件的情况下死亡会让 bot 永远 `busy`；启动时清除每个 bot（`server/index.ts:1296`）是当前的恢复手段。

**设计。**

- 对 `server/procs.ts` 的一次 tick，探测每个拥有进行中回合的进程。
- 至少检测进程退出；在传输支持的情况下进行协议级 ping。
- 丢失时：发出 `runtime.error`，区分设置问题与崩溃，结算回合，并将 bot 移到丢失状态而非静默空闲——在任务中途死亡的 bot 应该说明。
- 启动对账取代一刀切清除：将每个 bot 与观察到的进程对账，而不是重置所有 bot。

**完成标准。** 外部杀死驱动进程在一个 tick 内暴露丢失的 bot，且重启不再需要一次性清除每个 bot 的状态。

## 7. 可移植上下文

**做什么。** Harness 可以为任何引擎、在树中任何位置重建忠实且有大小限制的上下文。

**为什么。** 这是引擎切换、分支、群组回合和委托背后的机制。它们目前都只得到 40 条文本消息或什么都没有。

**主导规则。** 有两个 transcript，只有一个会收缩。显示路径——`store.activePath()`，在 `server/index.ts:179` 完整发送——已经正确，无界增长，且不被触碰。记录中永不删除任何内容。只有面向模型的重建是有界的。

**设计，分四部分。**

*7a. 目录中的上下文窗口。* 向 `ModelCatalog.options` 添加可选的 `contextWindow` 和 `maxTokens`。十行代码，没有它就无法为重建定大小。

*7b. 重放包含工作，而不仅是对话。* 当前过滤器是 `kind === "text"`。工具活动必须在重建中表示，否则交接的引擎不知道做了什么。紧凑地渲染 `activity` 条目——工具名、关键参数、结果——而非完整输出。

*7c. 压缩取代 `.slice(-40)`。* 一个 `compaction` 消息类型，携带 `{ summary, firstKeptId, tokensBefore, at }`，像任何其他消息一样带 `parentId` 追加，因此它存在于现有树中且不删除其后的任何内容。`store.modelContext(threadId, target)` 将 `activePath()` 回溯到最近的压缩记录，并返回摘要加上从 `firstKeptId` 起的消息。

正确性规则，来自 pi 的实现：绝不在工具调用与其结果之间切割；处理单回合就超出保留预算的拆分回合情况，通过生成并合并两个摘要；将前一个摘要喂入每次后续摘要；累积地向前携带读取和修改的文件。

通过已存在的 `ProviderInstance.generateText` 进行摘要。

*7d. 重建是目标的函数。* 同一线程可能在 8k 本地模型和 200k Claude 之间交接。一个冻结的摘要无法同时服务两者。将摘要视为为目标模型窗口定大小的重建的缓存**输入**，而非固定输出。按比例缩放预留——pi 的固定 16384 假设约 200k 窗口，会让 8k 模型永久超线。

*测量。* 当请求信封与上次成功调用匹配时，锚定在提供商报告的用量上；否则回退到启发式。从 deepseek-harness 的 token 计量器取此原则，而不取其完整的表面节点机制。

*范围。* 这管辖每条重建路径：grok、分支后重放、第 1 项的引擎切换、委托和群组回合。继续自己原生会话的 CLI 驱动不受影响，继续在内部压缩。

*UI。* 发出 `thread.compacted`。渲染一个安静的分隔线——"上下文已压缩，较早的消息仍在这里"——可展开到摘要。向上滚动显示所有内容。

**完成标准。** 一个数百条消息的线程交给不同引擎后，产生反映对话和其中所做工作的回复；且向上滚动仍能到达第一条消息。

## 8. 持久委托

**做什么。** Bot 到 bot 的交接在重启后存活。

**为什么。** `server/delegations.ts:31`——不持久化到任何地方。所述理由对权限成立，那里无人能为无人看管的 bot 作答。排队的工作不是权限。

**设计。** 将每线程队列与线程一起持久化，采用 routine 和 webhook 已获得的相同持久回执处理。在第 6 项的存活对账之后于启动时排空，在排空时重新验证目标和 `approvePeerComms` 设置——现有代码已在排空时而非入队时重新检查，这保持不变。

**完成标准。** 重启前立即排队的委托在重启后运行。

## 9. 工具截止时间与循环检测

**做什么。** 每次调用的工具截止时间和重复相同调用的检测。

**为什么。** 没有东西监视 bot 循环。那是用户的钱。

**设计。**

- 每次工具调用武装一个截止时间，可配置，有合理默认值。到期时终止调用并告知模型工具超时。
- 在最近调用的滑动窗口上检测重复，以工具名加规范化参数为键。首次响应是建议性的——注入一条说明同一调用已进行 N 次。仅在超过硬阈值时才升级为终止。
- 两者都在 harness 侧，因此对每个驱动都有效，而非按 CLI。

**完成标准。** 挂起的工具被切断且回合继续，重复一次调用的 bot 在重复二十次之前被提醒。

## 10. 工具结果脱敏

**做什么。** 在持久化之前从工具结果中清除机密。

**为什么。** Bot 读取的任何内容都落入 transcript，并在每个后续回合重放给模型——在第 7 项之后，也进入每次重建。一次意外的 `.env` 读取变成永久。

**设计。** 在消息写入之前运行的结果侧 Hook。复用 `server/auto-approve.ts` 中已有的模式（`SENSITIVE` 覆盖 `.env`、ssh 密钥、AWS 凭证、netrc、npmrc、keychain、服务账户）。将匹配替换为说明有内容被脱敏的标记