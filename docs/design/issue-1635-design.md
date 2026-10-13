# Issue #1635 — 设计

需求: `docs/requirements/issue-1635-requirements.md`

## 评审反馈

需求与设计的初次评审（Claude / Grok。Codex 在实现前纳入）中，与代码对照后保留的内容。

- 线程列表的比较使用新的 `orderedThreadList`。现有的 `orderedSidebarThreads` 保持注意顺序不变，铃铛、折叠中的其他线程、All-threads 的注意区块继续使用它。iOS 的 Updates / Live Activity 也不使用 `threadGroups`，而是使用注意顺序的专用列。
- All-threads 选择器（`ConversationTaskPicker` / `groupThreadTasks`）和频道列表也用 `orderedThreadList` 排序。
- 6 件不把置顶和"需要注意"计入数量，而是按更新从新到旧取 6 个已打开的线程。置顶无论关闭还是归档都保留。不把置顶送到移动端的 Archived 区块。
- 桌面端的文件夹顺序修正为可见线程的头部在上。现在的比较是反的，与注释不一致。
- 频道的置顶在作业中也能切换。标题变更照旧在作业中返回 409。客户端发送 `{ pinned, title: 当前标题 }`。旧服务器忽略未知的 `pinned`，只写回相同标题，不会变成空标题"Untitled"。
- `updatedAt` 只前进。启动时仅在未设置，或 SQLite 的 `MAX(at)` 更新时才修正。消息行不会删除，所以不下调。在导入旧版 JSON 之后执行。`appendMessage` / `insertMessageAfter` / `branchMessage` / `importTranscript` 推进内存。`saveBots` 不每条消息都调用。
- 客户端在快照时把 `updatedAt` 设为 `max(本地, 服务器)`。桌面端在后台线程的 early-return 和非活跃频道任务中也推进。乐观发送推进，取消则回退到剩余消息的最大值。iOS / Android 的 `Store` 应用 message 时也推进任务。
- `pinned: false` 在 `saveBots` / `saveGroups` 之前丢弃。把 `pinned` 加入 `TASK_PATCH_FIELDS`。
- 团队备份携带 `pinned`，恢复时的 `updatedAt` 为备份内消息的最大 `at`（没有则用 `createdAt`）。
- 文案使用现有的 `sidebar.bot.pin` / `sidebar.bot.unpin`。移动端的置顶标签与 Archive 一样使用界面侧的英文。

主设计文档不在本仓库中。本文件即决策记录。

## 决策

### 排序

显示顺序只用以下比较。不使用注意等级。

1. `pinned === true` 在前
2. `updatedAt ?? createdAt` 降序
3. 相同则按输入数组中的原始位置（稳定）

`activeId` 不用于排名。保留调用方的参数，仅用于可见性（把当前打开的行排除在 6 件限制之外）。

搜索结果也按此顺序。匹配条件不变。匹配到文件夹名的文件夹内容也按此顺序。

### 更新时间

`updatedAt` 是 `WireTask` 和 `GroupTask` 的可选 epoch ms。

- 创建时填入与 `createdAt` 相同的值。
- `Store.appendMessage` 和 `Store.insertMessageAfter` 确定消息后，把持有该 `threadId` 的机器人任务和频道任务的内存 `updatedAt` 设为消息的 `at`。此处不调用 `saveBots` / `saveGroups`。
- 启动时，仅当各任务的 `updatedAt` 缺失，或早于该线程的 `MAX(messages.at)` 时，修正为 SQL 的最大值（没有则用 `createdAt`）并保存一次。即使消息删除导致最新行消失，下次启动也会追上。
- 客户端在 `messageAdded`（包括进入后台线程缓冲区的情况）和乐观发送时，把本地 `updatedAt` 设为 `max(现有, message.at)`。下一次 bot/group 快照会用服务器值覆盖。

`updatedAt` 不加入 `TASK_PATCH_FIELDS`，也不加入 HTTP 的允许键。

### 置顶

把 `pinned?: true` 加入 `WireTask` 和 `GroupTask`。只保存 `true`，取消时删除属性。不把 `false` 留在磁盘上。

- 机器人: 把 `pinned` 加入 `PATCH /api/bots/:id/tasks/:threadId` 的允许键。非 boolean 返回 400。`patchTask` 之后若 `pinned !== true` 则 `delete task.pinned`。
- 频道: `PATCH /api/groups/:id/tasks/:threadId` 目前是标题专用。接受 `pinned`，没有标题的请求不调用 `renameGroupTask`。未知键返回 400。DM 照旧返回 400。
- 作业中也能切换置顶。归档和删除的 busy 限制不变。

### 桌面端的 6 件

`visibleSidebarThreads` 在既非搜索也非"全部显示"时，按更新时间降序查看后再计数。

- 置顶的行始终保留（无论关闭还是归档）
- 关闭的行、归档的行仅在需要注意时保留（现行的 `demandsAttention`）
- 其余打开的行按更新从新到旧取 6 件

之后 `orderedSidebarThreads` 按置顶＋更新顺序排列。群组的线程列表也经过同一函数（目前只有可见性过滤，连注意顺序都没过）。

### 行的操作

- 桌面端: `SidebarThreadRow` 的菜单加 Pin / Unpin。行上显示置顶图标。没有 `onPin` 的行（未传入的调用）不显示。
- iOS: 管理表的 context menu 和 swipe 加 Pin / Unpin。树和表的 `BotThreadRow` 显示置顶图标。相对时间用 `updatedAt ?? createdAt`。
- Android: 线程表的 Archive 旁加 Pin / Unpin。行的相对时间相同。显示置顶图标。

文案只加 `en.json` 的 `task.pin` / `task.unpin`。其他语言回退到英文。

### 不变的内容

- `ios/Sources/CompanionCore/AttentionInbox.swift` 和 `android/.../ui/AttentionInbox.kt` 的 attention 排名
- 文件夹的保存顺序（移动端）。桌面端的文件夹照旧跟随"可见线程的最头部"。顺序会变，因此包含置顶或近期更新的文件夹会排到上面。
- 例程内部执行（`routineRunId`）继续从列表中排除
- bot-to-bot DM 不持有任务数组

## 影响范围

### 变更对象

| 对象 | 变更 |
|---|---|
| `shared/wire.ts` `WireTask` / `GroupTask` | `pinned?: true`、`updatedAt?: number` |
| `server/store.ts` `TaskRecord`（extends `WireTask`）、`GroupTaskRecord` | 相同字段。创建、追加、启动时的填充 |
| `server/message-db.ts` | 每个线程的 `MAX(at)` |
| `server/index.ts` 任务 PATCH | `pinned` 的允许与校验 |
| `src/state/store.tsx` `Task` / `GroupTask` | 相同字段。收到消息时推进 `updatedAt` |
| `src/components/SidebarThreadRow.tsx` | 顺序与比较、菜单 |
| `src/components/Sidebar.tsx` | 搜索和群组列表也用相同顺序 |
| iOS `Models.swift` `BotTask`、`ThreadNavigation.swift`、`BotThreadRow.swift`、`TaskManagerView.swift`、`Client.swift` | 解码、顺序、置顶操作 |
| Android `Models.kt` `BotTask`、`ThreadNavigation.kt`、`BotThreadRow.kt`、`TaskSheet.kt`、`Client.kt` | 相同 |

`TaskRecord` 继承 `WireTask`，`toWireTask` 返回除去私有键后的剩余部分（`server/store.ts` 的 `TaskWireProjectionIsExact`）。新的公开字段加到 `WireTask` 即可上线到 wire。不要放入私有键。

### 影响点

顺序的消费者（显示会变）:

- `orderedSidebarThreads` 的调用: `Sidebar.tsx`、`SidebarBotActivity.tsx`、`TaskPicker.tsx`
- `visibleSidebarThreads` 的调用: `Sidebar.tsx`（机器人和群组）
- `Bot.threadGroups` 的调用: iOS `BotThreadTree.swift`、`TaskManagerView.swift`、`ChatListView.swift`、`Updates.swift`。Android `BotThreadTree.kt`、`TaskSheet.kt`、`TaskRules.kt`
- `orderedThreads` / `attentionRank`（用于线程列表）: `ThreadNavigation.kt`。Android `TaskRules.kt`

不能改变顺序的内容:

- `AttentionInbox.swift` 的 `attentionRank`、`AttentionInbox.kt` 的 `attentionRank`。是另一个函数。即使删除线程列表的 `attentionRank`，这边也保留。

持久化与 wire:

- `saveBots` 写入时从任务中除去 `busy` / `activity`。`pinned` 和 `updatedAt` 保留。`updatedAt` 不每条消息都保存。
- 伴侣的 JSON 忽略未知键（Android `CompanionJson.ignoreUnknownKeys = true`，iOS `JSONDecoder` 的默认）。旧应用即使附加新字段也不会崩溃。

### 影响类型

- 功能: 列表顺序改变。需要注意的旧线程，更新新则在上，旧则在下。6 件之外也保留可见性。
- 兼容: 仅新增可选字段。PATCH 的未知键目前也返回 400，因此新应用向旧服务器发送 `pinned` 会返回 400。失败按现有任务更新错误展示。
- 安全: 置顶与现有任务 PATCH 相同的授权。审批模式和模型不变。
- 性能: 启动时每个线程一次 `MAX(at)`。每条消息只更新内存。

### 应对方针

- 三个客户端的测试统一到相同示例（置顶在前、更新从新到旧、同时刻按原顺序、注意状态不影响排名）。
- 桌面端的现有测试，若时刻全部相同则保持原数组顺序，因此 6 件限制的多数可直接通过。期待注意顺序的 `orderedSidebarThreads` 测试需重写。
- 启动时的填充仅在"SQL 的最大值比保存值新，或未设置"时写入，不每次都污染 `bots.json`。

### 未受影响的依据

- Attention inbox 是另一个 `attentionRank`，不调用 `orderedSidebarThreads`。
- 例程定义和 webhook 的 `updatedAt` 是别的类型。任务的字段新增不会改变它们。
- DM 不持有 `group.tasks`，`createGroupTask` 会拒绝。

## 测试

- `src/components/SidebarThreadRow.test.ts`: 置顶、更新顺序、同时刻的稳定性、6 件按更新顺序、置顶和 attention 在 6 件之外也保留、关闭的置顶保留。
- `server` 的任务 API 测试: `pinned: true` 重新加载后保留、`false` 时键消失、非 boolean 返回 400、PATCH `updatedAt` 返回 400、追加消息推进 wire 的 `updatedAt`、无消息的任务用 `createdAt`。
- iOS `ThreadNavigationTests`、Android `ThreadNavigationTest`: 相同的顺序契约。
- 变异: 去掉置顶比较则置顶测试失败。去掉 `updatedAt` 比较则更新顺序测试失败。把 6 件恢复为数组顺序则"更新旧行后进入"测试失败。

## UI 确认

桌面端用隔离的 fixture，在侧边栏操作置顶和更新顺序。iOS / Android 用单元测试固定顺序和 PATCH 请求体，本环境不做真机的 headed 操作。