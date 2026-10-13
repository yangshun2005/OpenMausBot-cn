# 阶段 0 — 基础：让 harness 看见 bot 做了什么，并开始度量

实现说明（9 月 19 日）：这仍是原始设计，不是已交付能力的清单。当前基础涵盖回执、观测工作摘要和有界 Claude Hook。已实际验证的行为与限制见[验证边界](../verification/digests.md)。启动预算、指标和重放改动是独立的后续工作。HTTP provider 可以暴露工具；证据覆盖按轮次判定，绝不依据下面的引擎家族表。

状态：计划（2026 年 9 月 14 日）。这是 `../../../harness-gap-analysis.md`（§17，"基础优先，团队最后"）所述 harness 升级计划的第一个阶段。与 `agent-harness-upgrades.md` 第 7 项（可移植上下文，未构建）和第 15 项（原始检查器，已完成）组合。不触及任何 bot 间行为：委派、房间、对等通信和共享内存在阶段 6 之前保持不变。

**常设规则（负责人，9 月 14 日）：OpenMausBot 与模型和 CLI 无关，此处每一项都必须对每个引擎成立。** main 上的引擎家族有：**Claude Code**（stream-json，`drivers/claude.ts`）、**Codex**（app-server JSON-RPC，`drivers/codex.ts`）、**pi**（rpc 模式，`drivers/pi.ts`）、共享 `drivers/acp/core.ts` 的 **ACP 家族**（Cursor、Gemini、Droid、Grok Build、Hermes、Kimi、OpenCode Go、Qwen、custom、Antigravity）、共享 `drivers/openai-chat.ts` 的 **OpenAI 兼容 HTTP 家族**（openai-compat 用于 OpenRouter/Groq/Together/llama.cpp、xAI Grok API、MiniMax，以及通过 `local-inject.ts` 接入的本地主机），以及 **boat agent**（`drivers/boatagent.ts`，轮次在云端 boat 上运行）。下面每一项都以按家族给出的决定结尾：**full**、**degraded**（说明丢失了什么）或 **not supported**（隐藏在能力标志之后，遵循 `contracts.ts` 中现有规则：绝不告诉 bot 它拥有其驱动无法挂载的能力）。引擎专属通道（Claude Hook）是加速器；每一项的基线都是每个驱动已经发出的规范 `RuntimeEvent` 流。§"每个引擎，每一项"汇总了矩阵，测试计划对 `server/testing/` 中的每个 fake 引擎（`fake-claude-cli`、`fake-codex-app-server`、`fake-acp-cli`、`fake-agy-cli`、`fake-pi-cli`、`fake-driver`）运行每一项。

## 为什么

关于当前 main 的六个事实，每个都附有证明它的文件：

1. **harness 存储的是对话，不是工作。** 一条 `activity` 行保留 `tool.name`、`ok`、200 字符的 `summary` 和 ≤ 6,000 字符的 `output` 预览（`store.ts:Message.tool`、`tool-summary.ts:toolDetailPreview`），每个 CLI 驱动都已从其协议填充该预览（Claude 的 `tool_result` 块见 `drivers/claude.ts:1538`，Codex 的 `item/completed` 见 `drivers/codex.ts:917`，ACP 的 `rawOutput` 见 `drivers/acp/core.ts:799`，pi 见 `drivers/pi.ts:707`）。HTTP 家族完全没有工具（`drivers/openai-chat.ts` 仅聊天）。没有任何东西把这些行*聚合*成轮次记录，完整结果和推理存在于引擎自己的会话中，harness 从不读取。因此每次上下文重建——引擎切换、回退、外部更新、房间轮次——都是最近 40 条（`index.ts:5187`）或 30 条（`index.ts:6815`）*文本*消息，而"被交接的引擎完全不知道之前做了什么"（`agent-harness-upgrades.md` 第 7 项）。
2. **Claude Code Hook 未被使用。** 驱动已为每次启动写入私有的 `--settings` 文件（`drivers/claude.ts:1209-1211, 1379-1381`），其中只包含认证 `env`/`apiKeyHelper`。Hook（带完整 `tool_response` 的 `PostToolUse`、`PreToolUse`、`PreCompact`、带 `source: compact` 的 `SessionStart`、`Stop`）可以在同一文件中声明。它们仅为 Claude 增加未截断的工具结果和对压缩的发言权。其他引擎通过其协议（0.1）获得同样的*信息*，通过 harness 拥有的压缩记录（0.7）获得同样的*压缩*钩子；Hook 是加速器，不是设计。
3. **bot 轮次无法返回类型化对象。** Claude 驱动运行 `--output-format stream-json` 且无 schema（`drivers/claude.ts:1048`）；目标房间决策是从散文正则解析的（`group-goal-run.ts:parseGroupGoalDecision`）。阶段 3 和 6 中的一切（验证者裁决、图边、决策节点）都需要经过 schema 校验的输出。
4. **没有任何东西融合进程启动。** `spawnCli`（`procs.ts:61`）被每个驱动、一次性辅助程序（`drivers/claude.ts:1836`）、例程和委派唤醒调用。唤醒预算按线程存在（`DelegationWakeBudget`），但没有用户级上限。ruflo 的历史（每 Hook 的 `npx` 冷启动导致机器内核恐慌；守护进程泄漏数万次 `claude --print` 启动）就是未能预先阻止的失败。
5. **成本被记账，效率没有。** `appendUsage` 按已结算轮次记录 `input/output/cachedInput/cost`（`index.ts` 中 `case "turn.completed"` 的折叠；`usage-ledger.ts`），但没有任何东西计算每任务 token、每 bot 缓存命中份额，或实时轮次中哪个提示词部分花费多少（`index.ts:1553` 的 `previewSystemPrompt` 报告的是*预览*的字节数，不是轮次的）。
6. **每个新功能都落在 `server/index.ts`**（16,247 行，约 40 个模块级 map，五调用 drain 咒语在六个出口重复）。两个现有计划都推迟了分解。阶段 0 也不分解它；它通过给新变更一种统一形态来阻止增长。

塑造此计划的负责人问题："你每次都发送前 200 行和 40 条消息——这不是在浪费上下文吗？"诚实的答案在差距分析 §3：笔记本只在首次和变更时投递（易变的一半），40 条消息重放是回退路径。真正的浪费是工具输出在会话内累积、破坏缓存的前缀，以及重放真正触发时的有损形态。有一个引擎家族是例外，也证明了规则：OpenAI 兼容 HTTP 驱动没有可恢复的会话，所以对它们来说 40 条消息重放是*每一*轮（`drivers/openai-chat.ts:194`），且完全没有压缩——硬切。下面的 0.1、0.2、0.6 和 0.7 项正是针对这些问题，0.7 对 HTTP 引擎最重要，0.6 让每个部分的成本按轮次、按引擎可见。

## 我们借鉴什么（以及从哪借）

- **从事件 + 检查点 diff 生成工作摘要** — 我们自己的折叠（`item.completed` 用 `output` 修补 activity 行，`turn.completed` 结算）、`checkpoints.ts` 按 `(botId, sha(cwd))` 的影子 git（每轮前在 `index.ts:5639` 快照）、`memory-journal.ts` 的轮次边界 diff。Anthropic 的子代理指南（返回 1-2k token 的浓缩摘要）、Manus（"可恢复压缩：保留标识符，丢弃载荷"）。
- **Hook 作为观测通道** — Claude Code Hook 事件及其 JSON 载荷（`PostToolUse` 携带 `tool_name`、`tool_input`、`tool_response`；`SessionStart` 携带 `source`；Hook 可返回 `hookSpecificOutput.additionalContext`）。ruflo 的 Hook 约定，是吃了苦头学来的：始终 exit 0、全局 5 秒定时器、按 `tool_use_id` 去重、本地辅助脚本（绝不在 Hook 内用 `npx pkg@latest`）、stdin JSON 做 snake/camel 归一化。OpenClaw 的静默预压缩内存刷新。herdr 的教训：绝不让 Hook 成为"已阻塞"的*唯一*权威（Esc 不留 Hook 痕迹）；我们的驱动已通过 stream-json 看到请求及其解决，所以这里的 Hook 只*观测*和*注入*，绝不决定状态。
- **类型化轮次** — `claude -p --output-format json --json-schema` 返回 `structured_output`；`codex exec --output-schema <file>`（有两个已知 Codex 注意事项：某些版本中 MCP 服务器活跃时 schema 被忽略，且 `resume` 不支持）。continual-harness 的规则：无论 provider 如何都在代码中校验，解析失败意味着"未完成"。
- **全局启动预算** — ruflo `services/global-ai-budget.ts`（文件锁许可、`maxConcurrentGlobal`、每小时/每天上限、配额错误时暂停）。我们的 `DelegationWakeBudget` 和 `room-post-budget.ts` 纯 `(budget, attempt) → decision` 形态。
- **命令 + 回执形态** — t3code（我们的上游：`packages/contracts`、`apps/server/src/orchestration/{decider,OrchestrationEngine}.ts`）：每个变更都是带幂等回执的类型化命令，与其事件在同一事务中提交；副作用是订阅者。我们自己的 `RoutineRequestReceipt` 和 `chat_followups.sendId` 已为两条路径这样做。
- **度量** — Manus（KV 缓存命中率作为第一指标）、Factory（每*任务* token，不是每请求）、Claude Code 的 `/usage` "Prompt cache (main)" 行、tbench 的 cost/tokens 列。

## 设计

### 0.1 工作摘要（`server/digest.ts`，新增 `Message.kind = "digest"`）

在 `turn.completed` 时，折叠计算一条持久、脱敏的记录，说明该轮*做了什么*：

```ts
export interface TurnDigest {
  turnId: string;
  botId: string; threadId: string;
  at: number; durationMs: number;
  tools: Array<{ name: string; count: number; failed: number; sample?: string }>; // sample = existing commandSummary
  files: { changed: string[]; added: string[]; deleted: string[]; truncated?: number };   // from the checkpoint diff
  memory: Array<{ path: string; kind: "created" | "updated" | "deleted" }>;                 // from memory-journal rows for this turn
  reply: string;                 // first sentence of the terminal assistant text, ≤ 200 chars
  usage?: { input: number; output: number; cachedInput?: number; costUsd?: number | null };
  hookCoverage: "full" | "chips";  // "full" when 0.2 delivered real tool results for this turn
}
```

- 存储为 `pushMessage({ role: "bot", kind: "digest", text: renderDigest(d), digest: d, turnId })`，其中 `text` 是紧凑的单段渲染，使现有 FTS 触发器为其建立索引供 `session_search` 使用，无需 schema 变更（`message-db.ts` 索引 `messages.text`）。
- **变更的文件**来自新的 `checkpoints.diffStat(botId, cwd, fromHash, toHash)`，它对影子仓库运行 `git diff --name-status <before> <after>`（`runGit`，现有 120 秒超时和按仓库 `serialize`）。"before" 哈希是派发时拍摄的快照（`index.ts:5639`）；"after" 是结算时的第二个快照，标记为 `settle <turn>`。当检查点被禁用或对 cwd 被拒绝（`refusalReason`）时，省略 `files`，摘要中说明这一点。路径相对于 cwd；路径中的秘密不是问题，内容从不读取。
- **工具计数**来自该轮的 `activity` 行（按 `tool.name` 分组，`ok === false` 计为失败）——对每个驱动都可用。当 0.2 活跃时，`sample` 和 `failed` 计数来自真实结果而非 chip，`hookCoverage` 翻转为 `"full"`。
- **内存行**来自 `memory-journal.ts` 中在派发和结算之间写入的、具有此 `threadId` 的条目（`endMemoryTurn` 已在 `turn.completed` 时运行；把摘要排在其后）。
- 渲染的摘要**包含在每次上下文重建中**：`index.ts:5185` 的直接重放过滤器和 `index.ts:6813` 的 `serializeRoomContext` 接纳 `kind === "digest"` 行，渲染为方括号行 `[What <bot> did in the previous turn: …]`。这是重建上下文首次携带*工作*，而不仅是对话。渲染器将摘要行显示为回复下方的折叠 chip（"did 30 things · 2 files"）；它不是新的消息气泡。
- 大小上限：渲染后 1,500 字节（工具列表按计数截断为前 8，文件截断为 20 并带 `+N more` 标记）。每个已结算轮次一条摘要；失败/中断的轮次仍得到一条，`reply: ""`，以免工具使用记录丢失。

**引擎。** *Full：* Claude、Codex、pi、ACP 家族——工具计数、失败和样本来自这些驱动都已写入的、带 `output` 预览的 `activity` 行；文件来自检查点 diff（任何有 `cwd` 的引擎）；内存来自日志。*Degraded：* box agent——轮次在 box 上运行，所以 harness 只看到助手文本；摘要携带回复、usage 和 `hookCoverage: "none"`，box 自己的工具活动是阶段 5 项（box 侧事件捕获）。*本质上 degraded：* HTTP 家族——无工具，所以摘要是回复 + usage + 内存行；仍然写入，因为这些引擎的重建每轮都发生，摘要是它们唯一的工作记录。

### 0.2 通过现有 `--settings` 文件的 Claude Code Hook（`server/hooks/`、`drivers/claude.ts`）

`authSettingsPath` 变为 `settingsPath` 并始终写入（不仅在认证设置存在时）。其内容增加一个 `hooks` 块，指向一个本地辅助程序 `server/hooks/omb-hook.ts`（用与 `PERM_PROXY_PATH` 相同的 `process.execPath --experimental-strip-types` 运行），它把事件 POST 到 `http://127.0.0.1:PORT/api/internal/hook`，并在 `env` 中携带该轮的能力令牌（与每个代理使用的同一个 `mintInternalCapability` 门，所以 Hook 只能报告到自己线程）。

事件及 harness 对每个事件的处理：

| Hook | 匹配器 | Harness 动作 |
| --- | --- | --- |
| `PostToolUse` | `*` | 摄取 `{tool_name, tool_input, tool_response, tool_use_id}`；把脱敏结果（`tool-summary.ts:toolDetailPreview`，6 KB 上限；超过时把全文以 0600 溢出到 `DATA_DIR/tool-results/<thread>/<tool_use_id>.txt`）存到匹配的 activity 行（`toolMessageByItem`）；供给 0.1。按 `tool_use_id` 去重（重试的 Hook 绝不重复计数）。 |
| `PreToolUse` | `Bash` | 返回 `updatedInput`，把已知嘈杂命令（`pnpm test`、`pytest`、`npm test`、不带 `-n` 的 `git log`、包安装）改写为过滤形式（`… 2>&1 \| tail -n 200`、`--quiet`），来自 `server/hooks/filters.ts` 中的小允许列表。绝不阻塞；绝不改写不认识的命令。按 bot 默认关闭，直到度量后（见 0.6）。 |
| `PreCompact` | `*` | 返回 `additionalContext` = 压缩指导："保留决策、未解决的错误、文件路径、验证命令、当前 PROGRESS 目标；丢弃工具输出"。在总线上记录 `compaction.pending` 事件。（OpenClaw 式的*静默刷新轮次*是阶段 1；这里只引导 CLI 自己的摘要。） |
| `SessionStart` | `source: compact` | 返回 `additionalContext` = 此线程最近的摘要（≤ 1,500 字节），使压缩后上下文携带工作记录。记录 `compaction.completed`。 |
| `Stop` | `*` | 总线事件 `turn.stop-hook`，带 CLI 的 `stop_hook_active`；仅用于 0.6 中摘要与折叠一致性计数器。无行为。 |

约定，由 `omb-hook.ts` 及其测试强制执行：每条路径都 exit 0；5 秒全局定时器；宽松解析 stdin JSON；除 loopback 外无网络调用；不在 `DATA_DIR` 外写入。`--settings` 文件已属于 `privateFileFlags`，所以 hooks 块变化不改变 `argsKey`（不重启健康会话）——但 Hook *辅助程序路径*和 harness 端口在服务器生命周期内稳定，所以这是安全的。功能标志 `OMB_HOOKS=0` 禁用全部；Codex bot 不受影响（其摘要保持 `hookCoverage: "chips"`