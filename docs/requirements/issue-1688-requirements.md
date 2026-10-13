# Issue #1688 — 支持选择 Claude Opus 5.5

## 目的

将 2026-09-22 发布的 Claude Opus 5.5（API id `claude-opus-5-5`，上下文 100 万 token）加入 Claude 引擎的模型列表。

## 验收标准

- 列表中把 `claude-opus-5-5` / "Claude Opus 5.5" 放在 `claude-opus-5` 之前。
- 默认仍为 `claude-sonnet-5`。不修改现有机器人的模型。
- `modelContextWindow("claude-opus-5-5")` 为 1,000,000。依据是 Anthropic 的模型页面（context window 1M）。`claude-opus-5` 和 `claude-opus-5-50` 为 200,000。`host::` 的本地注入不设为 1M。
- 该 id 视为官方云端模型，不进行本地注入的探测。
- 不修改 Droid 的固定列表。

## 非范围

- 修改默认模型、机器人迁移、Fast mode、Bedrock 的其他 id。
- 禁用 `thinking` 或强制 `tool_choice`。Claude 驱动不发送这些。