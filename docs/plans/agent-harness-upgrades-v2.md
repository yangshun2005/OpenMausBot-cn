# OpenMausBot Harness 升级 v2 实施计划

> **面向智能体工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 按任务逐项执行本计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 修复 harness 中损坏的上下文重建路径，补上缺失的韧性与可观测性原语，并解锁本地模型——按每一轮都为下一轮铺路的顺序推进。

**架构：** OpenMausBot 本身就是一个 harness；`server/store.ts` 是跨引擎的权威记录，每一条将其还原为模型上下文的路径都是关键路径。本计划保留 v1 的重新定框与轮次划分，但依据代码当前的真实状态修正了范围：有两项已部分实现（附件、脱敏），有一项对 CLI 驱动而言架构有误（工具超时），有一项属于推测性设计（generation fencing），还有一项缺失的 Round-1 事项一直藏在开放问题里（每个 bot 的 `cwd`）。

**技术栈：** TypeScript（`--experimental-strip-types`）、vitest、pnpm、Electron + Vite。测试：`pnpm vitest run <file>`；类型检查：`pnpm typecheck`。

**规格：** `docs/plans/agent-harness-upgrades.md`（v1）。本文档取代其排序与范围；v1 的**已验证发现**、**设计**章节以及**上游参考**（v1 末尾列出从 pi、deepseek-harness、mem0/Letta 借鉴内容的那一节）仍是本计划原样承接事项的设计细节。当本计划说"设计同 v1 第 N 项"时，执行者去读 v1 的对应章节。

## 全局约束

- Node 通过 `--experimental-strip-types` 直接运行服务端 TS；导入使用显式 `.ts` 扩展名。
- 绝不破坏单文件驱动承诺：新增一个 provider 就是 `server/drivers/` 下的一个文件加一次注册。
- store 记录中的任何内容都永不删除；只有面向模型的重建是有界的。
- `~/.openmausbot` 跨应用版本共享——每个新字段或消息类型都必须能被旧版本忽略。
- e2e 测试遵循 `server/branching.test.ts` 模式：真实服务器、假 CLI（`server/testing/fake-acp-cli.ts`）、POSIX 门控。
- 现有 59 个测试文件必须全部保持通过（`pnpm test`）；v1 的"82 个测试文件"数字有误。

---

## 相对 v1 改了什么，以及为什么

v1 的每一项事实性论断都在 2026-08-17 对照仓库重新核实过。几乎全部成立。以下这些不成立，并重塑了本计划：

| # | v1 说 | 仓库说 | 变更 |
|---|---------|-----------|--------|
| 1 | "输入仅限文本"，从零构建附件 | `src/lib/composer-attachments.ts` 及其测试已实现粘贴/文件附件、拖放处理、大小限制，以及 `composeMessage` 折叠进提示词 | 第 2 项重新定范围为**仅图片**，扩展现有模块 |
| 2 | 脱敏复用 `auto-approve.ts` 的模式 | `server/redact.ts` 已提供 `redactSecrets`，含掩码、形状保持及测试 | 第 10 项重新定范围为在现有函数上**新增调用点 + 内容规则** |
| 3 | 工具超时/循环检测"放在 harness 侧，这样对所有驱动都生效" | CLI 驱动在 CLI *内部*运行工具；harness 观察工具事件但只能中断整个回合，而建议注入需要回合中途转向（v1 第 17 项） | 第 9 项拆分：重复*检测*保持靠前（观察 + 呈现），强制执行移到转向**之后** |
| 4 | Round 2 中的 generation fencing | 未观察到该 bug 的实例；仓库已有两种陈旧性概念，且 v1 自己警告第三种会导致无法捕获的 bug | **推迟**，直到原始检查器显示确有陈旧事件落地 |
| 5 | 开放问题 1："`cwd` 是否按 bot 呈现？" | 没有——每个驱动都接受 `turn.cwd`（`claude.ts:399`、`codex.ts:107`、`acp/core.ts:239`、`antigravity.ts:141`），而 `src/` 中没有任何地方设置它 | 按 v1 自己的标准提升为 **Round 1**；驱动侧已完成 |
| 6 | 82 个测试文件 | 59 个（`node_modules` 之外的 `*.test.*`） | 风险中的迁移预算相应下调 |
| 7 | 原始检查器列在第 15 位，"尽早构建" | 同意——所以计划应真正把它放前面 | 移到 **Round 1**，紧接模型切换修复之后 |

---

## Round 1 — 损坏的、缺失的，或为后续一切铺路的

轮次内的顺序即执行顺序（1.3 和 1.6 于 2026-08-17 跳过，现位于 3.4 和 3.5）。

### 1.1 修复模型切换上下文丢失 *（v1 第 1 项——详细任务计划见下）*

已验证损坏：新实例没有 `resumeCursors` 条目、不设置 `rewound`、不是 grok——因此它只收到最新消息，别无其他（`server/index.ts:806–826`、`:979`）。修复复用现有的 rewind 重放分支，使用一个独立的 `fresh` 标记。完整 TDD 拆解见下方 **Task 1**；先执行它。

### 1.2 原始事件检查器 *（v1 第 15 项，上移）*

`RuntimeEventBase.raw` 已携带原生载荷；按线程呈现它，含工具调用、时间戳和原始开关。设计同 v1 第 15 项。现在就构建，因为 1.4–2.x 都通过它诊断，且推迟的 generation fencing 需要它的证据。

### 1.3 主驱动中的自动重试——**暂时跳过，移到 3.4**

2026-08-17 在 1.1 和 1.2 交付后决定：推迟到 Round 3 末尾（见 3.4）。发现仍成立；只是这一项不是下一个。

### 1.4 每个 bot 的工作目录 *（从 v1 开放问题 1 提升）*

本计划中最小的一项：每个驱动都已接受 `turn.cwd` 并默认 `homedir()`。向 `BotRecord`（`server/store.ts`）添加 `cwd?: string`，在 `server/index.ts` 分发时紧挨现有 `integrations` 接线传入，在 bot 配置 UI 中用文件夹选择器（Electron 对话框）暴露它，并在聊天头部显示活动文件夹。完成标准：指向某项目文件夹的 bot 在每个驱动上都在那里运行其 shell 工具。

### 1.5 成本与用量可见性 *（v1 第 5 项）*

已验证：`turn.completed` 上的 `cost`（`server/contracts.ts:92`）和 `thread.token-usage.updated` 由所有主驱动发出；`src/` 中唯一的 `cost` 引用是 `routines.ts` 里的一个休眠字段。设计同 v1 第 5 项，包括计量 vs 订阅的标注。

### 1.6 图片附件 *（v1 第 2 项，重新定范围）*——**暂时跳过，移到 3.5**

2026-08-17 决定：不是下一个。重新定范围后的设计记录在 3.5。

### 1.7 跨线程搜索 *（v1 第 4 项）*

设计同 v1 第 4 项（先服务端线性扫描，包含 `activity` 类型，结果通过 `switchBranch` 落地）。

---

## Round 2 — harness 保证

### 2.1 存活回收器 *（v1 第 6 项）*——设计同 v1。2.3 和 Round 3 转向的前置条件。
### 2.2 可移植上下文 *（v1 第 7 项）*——设计同 v1，全部四部分（目录 `contextWindow`、重放中的 activity、压缩类型、目标大小重建）。单项最大；原始检查器（1.2）和修复后的重建路径（1.1）是它的脚手架。
### 2.3 持久委派 *（v1 第 8 项）*——设计同 v1；在 2.1 的对账之后排空。
### 2.4 工具结果脱敏 *（v1 第 10 项，重新定范围）*——一个结果侧钩子，在 `store.appendMessage` 之前调用现有的 `redactSecrets`（`server/redact.ts`），在其当前*键名*规则之外扩展*内容*模式（文件读取中的键形值），外加 `server/auto-approve.ts` 中的 `SENSITIVE` 路径列表。脱敏存储副本。新测试扩展 `server/redact.test.ts`。
### 2.5 失败关闭式审批 *（v1 第 11 项）*——设计同 v1（`allowed-once | rejected | cancelled | unavailable`，`request.opened` 通过 `itemId` 引用工具项）。单独落地。
### 2.6 在 store 内发出 *（v1 第 13 项）*——设计同 v1：让 `appendMessage`/`patchBot` 在写入时发出；明确**不是**通用 `commit()` 重构。
### 2.7 服务端活动状态 *（v1 第 14 项）*——设计同 v1：五种状态，`busy` 保留为派生 getter，先仅 harness 读取方（221 处引用 / 36 个文件逐步迁移）。
### 2.8 重复调用**检测** *（v1 第 9 项，前半）*——仅观察：对流转的工具事件按名称 + 规范化参数做滑动窗口，在 UI 中呈现"同一调用 × N"（通过 1.2 的检查器和线程横幅），一键中断。不注入、不逐调用终止——在转向存在之前，harness 对 CLI 驱动两者都做不到。

---

## Round 3 — 由前两轮解锁

### 3.1 本地模型 *（v1 第 16 项）*——设计同 v1；阻塞于 2.2，之后从 `grok.ts` 起步几乎免费。
### 3.2 回合中途转向 *（v1 第 17 项）*——设计同 v1，包括 stdin-EOF 验证门（`claude.ts:536`——在设计队列**之前**回答开放问题 3）和 2.1 前置条件。
### 3.3 工具超时与循环**强制执行** *（v1 第 9 项，后半）*——有了转向，升级 2.8 的检测：在下一个安全点注入建议说明，超过硬阈值则终止。仅在 harness 真正拥有调用时才逐调用设超时；对 CLI 驱动，超时仍是"中断回合"，在 UI 中如实说明。
### 3.4 主驱动中的自动重试 *（v1 第 3 项，原 1.3——2026-08-17 跳过并移到这里）*——已验证：`claude.ts` / `codex.ts` / `grok.ts` 中零重试，而 `boat.ts`、`webhooks.ts` 和 `acp/core.ts` 都有。设计同 v1 第 3 项：区分瞬态（429、5xx、过载、连接重置）与终态（认证、无效模型、配额）；退避 + 抖动，尝试次数上限较小；一个 `turn.retrying` 运行时事件让 UI 显示它（1.2 的检查器免费使其可见）；一条中止路径，使重试期间的中断取消整个回合；部分输出后的重试丢弃部分而非重复它。完成标准：模拟的 529 产生可见重试并完成回合，无效 API key 立即失败且不重试。
### 3.5 图片附件 *（v1 第 2 项，原 1.6——2026-08-17 跳过并移到这里）*——粘贴/文件/拖放层已存在于 `src/lib/composer-attachments.ts`；只缺图片路径。设计：`POST /api/attachments`（图片 mime，≤10 MB）写入 `~/.openmausbot/attachments/<uuid>.<ext>` 并返回路径；composer 显示缩略图 chip，`composeMessage` 发出 `<attached-image path="…"/>`——每个 CLI 引擎按路径打开文件，无需逐驱动编码；转录将标签解析为缩略图；每个驱动的 `capabilities.images` 门控粘贴（Claude/Codex/Antigravity/ACP 为 true，grok/boxAgent 为 false）；API 驱动的 `SendTurnInput.images` 等待第二个 API 驱动（3.1）。完成标准：粘贴的截图到达 Claude、在转录中渲染、重载后仍在，且 grok bot 礼貌拒绝它。

---

## 推迟

v1 推迟列表中的所有内容因 v1 的理由继续推迟（共享记忆、技能即人格、溢出存储、树导航 UI、远程访问基础、拆分 `server/index.ts`），外加：

- **Generation fencing** *（原 v1 第 12 项）。* 罕见、无观察到实例，且会在 `rewound` 和 `resumeCursors` 之外增加第三种陈旧性概念——v1 自己的横切风险。仅当原始检查器（1.2）捕获到来自已处置实例的陈旧事件落到活跃回合上时才重新考虑。若发生，先统一现有两种概念。

## 流程改进（代码之外的"我们需要做什么"）

1. **在依赖项之前回答经验性开放问题，而非在其期间。** v1 开放问题 2（Codex 回合中途输入）、3（Claude CLI EOF 语义）和 4（Ollama `/v1/models` 上下文窗口）各是半天的探索，用一个一次性脚本；在 Round 1 期间运行它们，使 Round 2–3 的设计有据可依。答案记录在本文件中。
2. **每个 PR 一项，测试在同一 PR 中。** 触及契约的事项（1.6、2.5、2.7）会波及驱动测试——在事项内预算该迁移，而非事后。
3. **构建前重新核实。** v1 写于 2026-08-17，已因代码变动在两处（附件、脱敏）过时。下方每个任务都以验证步骤开头，出于同样原因。
4. **保持双转录规则可见。** 每个重建路径 PR 描述必须声明：显示路径未动、未删除任何内容、只有面向模型的重建是有界的。

---

# Task 1：修复模型切换上下文丢失

**文件：**
- 创建：`server/turn-context.ts`
- 创建：`server/turn-context.test.ts`
- 修改：`server/index.ts:806-826`（transcript/turnText 块）、`:979`（resumeCursor 行保持原样——仅验证）

> **实际构建（#180）：** 下方的朴素规则——"fresh = 此实例无游标"——先被实现，在 A → B → A 的手动测试中失败（第一个引擎有几天前的游标，恢复了一个陈旧会话）。交付的契约是**基于分发的**：`TaskRecord.lastInstanceId` 由 `store.markTaskDispatched()` 在每次 `sendTurn` 后设置，`engineIsFresh({ instanceId, lastInstanceId, resumeCursors, transcript })` 决定重放——上一个实例不同 ⇒ fresh；无该字段的遗留任务回退到游标映射（仅有一个自身游标则继续恢复；任何模糊情况则重放）；仅有种子问候永不触发重放。仅当 `resume` 为 true 时才传入 `resumeCursor`。下方步骤按原样保留存档；若有出入，以代码和 `server/turn-context.test.ts` 为准。

**接口：**
- 消费：`store.activePath(threadId)`（存在）、`task.resumeCursors: Record<InstanceId, unknown>`（存在，`server/store.ts:117`）、`bot.rewound`（存在）、`task.lastInstanceId`（新增，见上）。
- 产出：来自 `server/turn-context.ts` 的 `buildTurnContext(input: TurnContextInput): { turnText: string; resume: boolean }` 和 `engineIsFresh(...)`；`store.markTaskDispatched(botId, threadId, instanceId)`——第 2.2 项（可移植上下文）稍后替换 `buildTurnContext` 的内部；其签名即接缝。

```ts
// server/turn-context.ts — the produced interface, in full
export interface TurnContextInput {
  /** the user's new message */
  text: string;
  /** settled text turns on the active branch, oldest first, capped upstream */
  transcript: Array<{ role: "user" | "assistant"; text: string }>;
  /** the visible branch changed (edit / version switch) */
  rewound: boolean;
  /** this driver instance has no session cursor for this thread */
  fresh: boolean;
  /** transcript-replay drivers get history via SendTurnInput.transcript instead */
  replaysNatively: boolean; // driverKind === "grok" today
}
export function buildTurnContext(input: TurnContextInput): {
  turnText: string;
  /** false when the native session must not be resumed */
  resume: boolean;
};
```

- [ ] **Step 1：验证 bug 仍存在**

阅读 `server/index.ts` 中 `slice(-40)` transcript 块附近，确认：`rewound` 仍是内联重放的唯一触发器，且 `task.resume