# Issue #1688 — 设计

## 变更

在 `STATIC_CLAUDE_MODELS` 中，将 `{ id: "claude-opus-5-5", label: "Claude Opus 5.5" }` 加到 `claude-opus-5` 之前。`default` 不变。

`resolveClaudeTurnModel` 将静态列表中的 id 从本地注入的搜索中排除。新 id 已在该列表中，因此无需额外分支。

`modelContextWindow` 仅在末尾恰好为 `claude-opus-5-5` 或 `claude-opus-5.5` 时返回 1,000,000。允许斜杠或点之前的部分作为转发来源（`openrouter/anthropic/claude-opus-5-5`、`anthropic.claude-opus-5-5`）。依据是 Anthropic 的模型页面（2026-09-22，id `claude-opus-5-5`，context window 1M tokens，https://platform.claude.com/docs/en/models/opus-5-5/overview ）。`claude-opus-5`、`claude-opus-5-50`、`claude-opus-5-5-local`、`claude-opus-5-5garbage` 仍沿用通用 Claude 规则。`host::model` 的本地注入不在范围内。目录行中也放置 `contextWindow: 1_000_000`。`contextWindowFor` 优先使用目录声明，而非模式表。

同类排查：标签写为 1M 的行中，Cursor 的 `claude-sonnet-5-thinking-high` 有 1 条。由于没有模型页面，本 PR 中保持 200,000。Fable 5.1 和 Sonnet 5 在本 PR 中也保持原有的 200,000（带 `[1m]` 的除外）。是否改为 1M 另作处理。

Claude 的 harness compaction 仍保持 `autoCompactWindow` 默认的 200,000。1M 是使用量标签的分母（`server/index.ts` 中的 `modelContextWindow` 调用），以及 `contextWindowFor`（`server/context-budget.ts`）读取目录声明时的宽度。`--autocompact` 的默认值不变。

旧版 Claude Code 若不认识 `claude-opus-5-5`，该模型的回合会失败。标志用的 version floor 不适用于模型 id，与添加 Fable 或 Opus 5 时相同。

## 影响范围

| 对象 | 位置 | 类型 | 方针 |
| --- | --- | --- | --- |
| Claude 的静态列表 | `server/drivers/claude.ts` `STATIC_CLAUDE_MODELS` | 功能 | 添加 1 行。默认与现有 bot 不变 |
| 上下文宽度 | `server/model-context-window.ts` | 功能 | 仅 Opus 5.5 及以斜杠或点分隔的转发形式为 1M |
| 使用量标签 | `server/index.ts` 的 `modelContextWindow` | 功能 | 目录未返回窗口时的分母。compaction 不变 |
| 窗口解析 | `server/context-budget.ts` `contextWindowFor` | 功能 | 优先使用目录的 `contextWindow`，而非模式表 |
| Droid 的 `MODELS` | `server/drivers/acp/droid.ts` | 无 | Factory 侧的快照，不触碰 |
| 目录检查 | `claude-catalog.test.ts` | 测试 | 固定位置与默认值 |
| 宽度检查 | `model-context-window.test.ts` | 测试 | 5.5 为 1M，5 为 200k |

调用方将列表原样输出到选择器。排序契约不变。

## 测试

在隔离的 vitest 中运行上述 2 个文件。变异测试为从静态列表中移除 `claude-opus-5-5`，观察目录检查是否失败。确认后恢复。