---
title: "fix: 在权限代理中拒绝冲突的 ask id"
plan_type: fix
artifact_contract: ce-unified-plan/v1
artifact_readiness: implementation-ready
execution: code
product_contract_source: ce-plan-bootstrap
created: 2026-08-18
---

# fix: 在权限代理中拒绝冲突的 ask id

## 摘要

[server/drivers/claude.ts](server/drivers/claude.ts) 中的 `createPermissionBroker()` 会把每条收到的 `{t:"ask", id, ...}` 消息注册到以 `askId` 为键的共享 `pending` Map 中，且不检查是否已存在同键条目。两个 id 相同的 ask 到达时——可能来自代理上的任意连接，因为 `pending` 是服务端作用域而非按连接隔离——会静默冲突：第二次 `pending.set()` 覆盖第一次，`onAsk` 仍会为两者触发（可能出现两张 `request.opened` 卡片），而当*第一个* ask 的 `finish()` 稍后执行时，`pending.delete(askId)` 会把第二个（仍在显示的）条目一并删除，使其永久无法应答——即由 id 冲突导致的僵尸卡片，与 #211 修复的关闭后时序问题属于不同类别。

本计划增加一个摄入时守卫，在 `onAsk` 触发或 `pending` 被改动**之前**检测冲突 id，直接在其自身连接上自动拒绝冲突的 ask，并保持原有 pending 条目不变。同时说明为何现实中冲突极不可能发生（两个 ask 生产者都为每个 ask 生成新的 `randomUUID()`，不存在复用同一 id 的重试/重放路径），并为该守卫添加回归测试。

## 问题背景

**根因：** [server/drivers/claude.ts:245](server/drivers/claude.ts#L245) —— `pending.set(askId, { ask, finish })` 在每条解析出的 `ask` 行上无条件执行，不检查 `askId` 是否已是 `pending` 的键。

**为何冲突罕见却仍需处理：** 该故障模式静默且不对称——一个合法的首个 ask 可能被后续的 id 冲突永久孤立，而它无从察觉；用户面对一张无法应答的 UI 卡片，且没有任何诊断信息。一个低成本的摄入时守卫能把一个无边界、难以调试的故障变成可观测、故障安全的故障。

**范围：** 仅限 `server/drivers/claude.ts` 的 `createPermissionBroker()`。`server/permission-proxy.ts`（id 生产者）经调查但不修改——其 `randomUUID()` 生成方式本身正确。

## 调查结论（记录需求 1）

两个 ask 生产者都为每个 ask 生成新 id，不存在重发已有 id 的路径：

- [server/permission-proxy.ts:107](server/permission-proxy.ts#L107) —— `const askId = randomUUID();` 在 MCP `tools/call` 处理器内，每次 `tools/call`（`approve` 或 `ask_user`）执行一次。每次调用都会在代理本地 `waiting` map 中以同一 id 为键注册自己的 `Promise`；不存在超时或出错后复用 id 的重试循环——来自 CLI 的新 `tools/call` 总是生成新 id。
- [server/drivers/claude.ts:226](server/drivers/claude.ts#L226) —— 代理自身的兜底 `const askId = String(msg.id ?? newId())` 仅在传入消息完全省略 `id` 时触发，而 `newId()`（[server/contracts.ts:304](server/contracts.ts#L304)）是 `crypto.randomUUID()`——密码学随机，不是计数器。

因此现实中可能的冲突来源是：(a) 概率极低的 UUID 碰撞，或 (b) 权限 socket 上存在缺陷或恶意的客户端发送手工构造的重复 `id`。两者都不是合法的重试/重放路径——这确认该守卫是针对当前假设性但真实存在的一类 bug（以及未来任何不谨慎生成 id 的 ask 生产者）的纵深防御修复，而非针对已观测到的冲突。

## 关键技术决策

**KTD1 —— 守卫在摄入时触发，位于 `onAsk`/`pending.set` 之前，而非 `finish` 内部。** 解析行后立即检查 `pending.has(askId)` 并在此拒绝，意味着冲突的 ask 永远不会变成 UI 卡片，原有 pending 条目也永远不会被改动。被否决的替代方案：放行两个 ask，在 `finish`/`answer` 中给第二个条目一个合成键来消歧——这仍会在 UI 中产生僵尸卡片（因为 `onAsk` 已触发），并且为本不该到达 UI 的情况增加了 `answer()` 公共契约的复杂度。

**KTD2 —— 在冲突 ask 自身的连接上自动拒绝，而非静默丢弃。** 代理侧（`server/permission-proxy.ts:117`）是 `await new Promise`，等待其 `askId` 对应的 `t:"answer"`——如果代理丢弃该行而不应答，该 `tools/call` 会一直挂起直到 CLI 自身的超时（如果有），而不是快速失败。立即回写 `{t:"answer", id, behavior:"deny", message}` 保持了 socket 所有调用方已预期的契约（`server/permission-proxy.ts:49-53` 按 id 匹配 `t==="answer"` 并 resolve 其等待者），且故障关闭，与 [server/drivers/claude.ts:250-252](server/drivers/claude.ts#L250-L252) 处现有代理理念一致（"保持该轮次故障关闭，但留下可操作的诊断信息"）。

**KTD3 —— 通过 `console.error` 记录日志，与现有代理诊断模式一致。** [server/drivers/claude.ts:253-255](server/drivers/claude.ts#L253-L255) 已用此方式记录代理不可用错误；冲突 id 的日志行遵循同一约定，而不是为一个预期几乎不会触发的守卫引入新的日志通道或遥测事件类型。

**KTD4（会话已定：用户指示——选择不并入 #211 分支/PR：该 bug 与 #211 的关闭后时序 bug 属于不同的、超出范围的类别）** —— 本修复作为独立 PR 在基于 `main` 的新分支上发布，不并入 #211 僵尸卡片修复。

## 实现单元

### U1. 在 `createPermissionBroker()` 中防止 ask id 冲突

**目标：** 检测传入的 `ask` 其 `id` 在 `pending` 中已有活跃条目，在其自身连接上自动拒绝，不注册它也不调用 `onAsk`，并保持原有 pending 条目不变。

**需求：** 上述调查结论（说明为何该守卫是纵深防御）；KTD1、KTD2、KTD3。

**依赖：** 无。

**文件：**
- `server/drivers/claude.ts`（修改 `createPermissionBroker`，约 219–246 行）
- `server/drivers/claude.test.ts`（新增测试——见 U2）

**方法：**
1. 在 `const askId = String(msg.id ?? newId());`（226 行）之后、构造 `ask`/`finish`/`timer` 之前，立即检查 `if (pending.has(askId))`。
2. 冲突时：向 `conn` 写入 `{t:"answer", id: askId, behavior:"deny", message: <固定诊断字符串>}`（写入用 `try/catch` 包裹，与 232-234 行现有风格一致），用 `console.error` 输出一行诊断，包含 socket 路径和冲突 id，然后 `continue` `while` 循环——对该行完全跳过构造 `ask`、`finish`、`timer`、`pending.set` 和 `opts.onAsk`。
3. 不要以任何方式改动该 id 在 `pending` 中的现有条目；它的解析过程与没有冲突时完全一致。
4. 为拒绝消息引入一个导出或模块作用域常量（例如与 `QUESTION_TIMEOUT_NOTE`/`DENY_TIMEOUT_NOTE` 并列），而非内联字符串字面量，与该文件现有的面向用户的代理消息约定一致。

**遵循的模式：** 现有的超时拒绝消息 `QUESTION_TIMEOUT_NOTE`/`DENY_TIMEOUT_NOTE` 及其在 `finish` 超时分支中的用法（237-243 行）；`server.on("error", ...)` 诊断日志约定（253-255 行）；应答写入已使用的 `try { conn.write(...) } catch {}` 模式（232-234 行）。

**测试场景：**
- 正常路径（无回归）：正常的单个 ask 仍注册到 `pending`，仍触发 `onAsk`，仍可通过 `answer()` 解析——由 `claude.test.ts` 中现有测试覆盖；此处除确认现有测试仍通过外，无需新增正常路径测试。
- 同连接冲突：在同一 socket 连接上背靠背写入两条 `{t:"ask", id:"dup-1", ...}` 行，且在第一个被应答之前。预期 `dup-1` 恰好产生一个 `request.opened` 事件，第二条 `ask` 行在线上被应答为 `{t:"answer", id:"dup-1", behavior:"deny", ...}`，同时第一个 ask 自身的 `pending` 条目之后仍可通过 `answer("dup-1", "allow")` 解析。覆盖 KTD1/KTD2 的连接写入部分。
- 跨连接冲突：在一个连接上打开 id 为 `dup-2` 的 ask；在第一个解析之前，第二个连接（模拟第二个代理）发送相同 id `dup-2` 的 `ask`。预期第二个连接立即收到拒绝应答，不再触发第二个 `request.opened`，且第一个连接中 `dup-2` 的 pending 条目仍可通过 `answer()` 正确解析。这正是 bug 报告特别指出的场景（`pending` 是服务端作用域，而非按连接隔离）。
- 解析后复用*不*算冲突：`dup-3` 被应答后（其 `pending` 条目已通过 `finish` 移除），复用 id `dup-3` 的全新 ask 被正常接受（注册、触发 `onAsk`、可应答）。确认守卫只阻止*活跃*冲突，而非解析后的 id 复用——这是一个重要边界，因为 `pending.delete` 已释放该键。

**验证：** 上述四个场景全部通过；`claude.test.ts` 中现有权限代理测试（341-410 行）未经修改继续通过。

### U2. id 冲突守卫的回归测试

**目标：** 在 `server/drivers/claude.test.ts` 中为 U1 的守卫添加自动化覆盖，遵循权限代理现有的 socket 级测试模式。

**需求：** U1 的上述测试场景。

**依赖：** U1（守卫必须存在，测试才能对其断言）。

**文件：**
- `server/drivers/claude.test.ts`（在现有权限代理 `describe` 块中新增 `it(...)` 块，约 341-410 行附近）

**方法：**
1. 遵循 [server/drivers/claude.test.ts:341-380](server/drivers/claude.test.ts#L341-L380) 的现有模式：`create("hang")`，发送一个轮次，等待 `session.started`，`connect(permissionSocketPath(threadId))`，写入原始 `{t:"ask", ...}` JSON 行，并对 `recorder` 事件（`request.opened`）和原始 socket 读取（`{t:"answer", ...}` 行）进行断言。
2. 对于同连接冲突场景，两次 `ask` 写入复用同一个 `conn`，并通过在 `data` 上缓冲收集两条应答行（现有 `answered` promise 模式只 resolve 一次——将其扩展为收集解析行的数组，或按数组索引 resolve 两个独立 promise）。
3. 对于跨连接场景，针对同一 `permissionSocketPath(threadId)` 打开两个 `connect(...)` socket，各写入一个 ask。
4. 对于解析后复用场景，在写入相同 id 的第二条 `ask` 行之前，先通过 `instance.adapter.respondToRequest(...)`（如 372 行现有测试）应答第一个 ask，然后断言第二个*确实*产生 `request.opened`。
5. 用 `conn.end()` 清理每个打开的 `conn`，并以中断轮次、等待 `turn.completed` 结束每个测试，与现有测试一致。

**遵循的模式：** [server/drivers/claude.test.ts:341-410](server/drivers/claude.test.ts#L341-L410) —— 三个现有权限代理测试，尤其是原始 socket 写/读辅助函数和 `recorder.until(...)` 断言。

**测试场景：**（将 U1 的场景实现为可执行测试——无单独列表；逐字参见 U1 的"同连接冲突"、"跨连接冲突"和"解析后复用不算冲突"场景。）

**验证：** `npm test -- server/drivers/claude.test.ts`（或项目等效测试命令）通过，包括三个新用例，且无需改动文件中无关的现有测试。

## 范围边界

**范围内：** `server/drivers/claude.ts` 中 `createPermissionBroker()` 的摄入守卫、其回归测试，以及记录调查结论。

**范围外：**
- #211 关闭后时序修复（`fix/claude-turn-teardown-zombie-teardown-cards` 分支）——已是独立的、已合并/待处理的变更。
- 修改 `server/permission-proxy.ts` 的 id 生成——调查确认其生成方式已正确。
- 除现有 `console.error` 诊断约定之外的任何遥测/指标管道。

### 推迟到后续工作
未发现——这是一个小型、自包含的防御性修复。

## 验证契约

- `server/drivers/claude.test.ts` 中现有权限代理测试（341-410 行）未经修改继续通过。
- 新测试（U2）覆盖：同连接冲突、跨连接冲突、解析后 id 复用（非冲突）。
- 人工推理检查：确认守卫对冲突 id 永不触发 `opts.onAsk`（永不创建僵尸卡片，而非创建后被孤立）。

## 完成定义

- [ ] `createPermissionBroker()` 在摄入时拒绝冲突的 `askId`，立即返回拒绝应答并记录诊断日志，符合 KTD1-KTD3。
- [ ] 冲突 id 的原有 pending 条目可证明未受影响（仍正常解析）。
- [ ] 同连接冲突、跨连接冲突和解析后复用的回归测试已添加并通过。
- [ ] 未修改 `server/permission-proxy.ts`。
- [ ] 完整测试套件通过。