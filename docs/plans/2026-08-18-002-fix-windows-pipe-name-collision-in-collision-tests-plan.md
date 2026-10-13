---
title: "修复权限代理测试中的 Windows 管道名称冲突 - 计划"
type: fix
date: 2026-08-18
artifact_contract: ce-unified-plan/v1
artifact_readiness: implementation-ready
execution: code
product_contract_source: ce-plan-bootstrap
---

# 修复权限代理测试中的 Windows 管道名称冲突

## 目标概要

让 PR #230 新增的四个权限代理冲突测试在 Windows 上通过，方法是消除它们之间的命名管道名称冲突：四个测试使用的线程 id 截断后都得到相同的 8 字符标签，产生相同的 Windows 管道名称，而前一个测试遗留的管道会让下一个测试的代理 `listen()` 失败并报 `EADDRINUSE`。修复方案是给每个测试分配唯一线程 id，使其产生唯一标签，并加入确定性的标签唯一性断言，这样再次冲突时会在所有平台失败，而不只是在依赖时序的 Windows 运行中暴露。`server/drivers/claude.ts` 中的生产环境冲突防护已验证无误，保持不变。停止条件：`pnpm exec vitest run server/drivers/claude.test.ts` 在 macOS 和 Ubuntu 上通过，且 Windows 的 `typecheck + test (windows-latest)` 任务连续两次运行均为绿色。

## 产品契约

### 摘要

Windows CI 任务在 PR #230 上失败，vitest 运行中恰好只有一个测试失败：`denies a colliding ask id from a second connection on the same broker`，报错为 `Error: connect ENOENT \\.\pipe\openmausbot-perm-1892-t-perm-d`。同一任务日志显示了真正原因：`permission broker unavailable on \\.\pipe\openmausbot-perm-1892-t-perm-d: listen EADDRINUSE: address already in use`。该测试的代理从未启动，因为管道名称已被前一个冲突测试占用。四个冲突测试产生相同的管道名称。（完整的 `pnpm test` 任务还会运行 broker:test、updater 和 packaged-server；这些在失败的运行中从未执行，因为 `vitest run` 先以非零状态退出，所以本次运行无法证明它们的 Windows 状态。）

### 问题背景

`permissionSocketPath(threadId)` 将线程 id 截断为 8 字符标签（`server/drivers/claude.ts:192-195`）。四个冲突测试使用的线程 id 为 `t-perm-dup-1`、`t-perm-dup-2`、`t-perm-dup-3`、`t-perm-dup-4`，截断后都是 `t-perm-d`。在 Windows 上，`brokerSocketPath` 构造 `\\.\pipe\openmausbot-perm-<pid>-t-perm-d`（`server/procs.ts:114-119`）。

在 POSIX 上，`createPermissionBroker` 在 `listen()` 之前调用 `unlinkSync(opts.socketPath)`（`server/drivers/claude.ts:208-210`），删除前一个测试的套接字文件，因此重新监听同一名称会成功。在 Windows 上，对 `\\.\pipe\...` 路径调用 `unlinkSync` 是空操作——命名管道不是文件系统条目。前一个代理 `server.close()` 后，操作系统会短暂保留管道名称，因此下一个测试的 `listen()` 失败并报 `EADDRINUSE`，代理从未启动。随后测试的 `connectSocket` 因没有管道在监听而报 `ENOENT`。观察到的运行中，四个同名测试只有 1 个失败，说明释放窗口很短，失败依赖时序。

已有的代理测试从未遇到此问题，因为它们使用不同的线程 id（`t-perm-abc`、`t-perm-stop`），标签也不同。生产代码在正常运行时不太可能遇到此问题，因为真实线程 id 是 UUID，不过 8 字符截断在那里仍是潜在冲突风险——不在本计划范围内。

### 需求

**Windows 管道名称隔离**

- R1. 四个冲突测试必须各自使用该测试独有的代理管道名称，使任何测试的代理 `listen()` 都不会与前一个测试遗留的管道冲突。

**确定性回归防护**

- R2. 一个与平台无关的断言验证四个冲突测试的线程 id 产生四个不同的 `permissionSocketPath()` 值，这样未来标签再次冲突时会在所有平台失败，而不是只表现为依赖时序的 Windows 失败。

**回归保持**

- R3. 四个冲突测试保留其精确断言：完整的拒绝载荷（`id`、`behavior: "deny"`、`message`）、每个冲突 id 恰好一个 `request.opened`、原始 ask 仍可解析、解析后 id 复用被接受。

**范围边界**

- R4. 不修改生产文件：本计划不改动 `server/drivers/claude.ts` 和 `server/permission-proxy.ts`。不修改 `permissionSocketPath` 中的 8 字符标签截断。

**验证**

- R5. `pnpm exec vitest run server/drivers/claude.test.ts` 在 macOS 和 Ubuntu 上通过（声明 33 个测试；32 个通过，1 个跳过——仅 Windows 的管道命名测试）。Windows 的 `typecheck + test (windows-latest)` CI 任务连续两次运行均为绿色。

## 计划契约

### 关键技术决策

- KTD1. 修复测试线程 id，而非生产标签逻辑。（会话已定：用户指示——选择此方案而非修改 `permissionSocketPath`/`brokerSocketPath`：生产行为对真实 UUID 线程 id 是正确的，改变标签方案会改变生产套接字名称，并可能破坏打包应用的管道契约。）
- KTD2. 给每个冲突测试分配一个 8 字符标签唯一的线程 id。使用截断后仍保持唯一的 id：`t-dup-1`、`t-dup-2`、`t-dup-3`、`t-dup-4` → 标签 `t-dup-1`、`t-dup-2`、`t-dup-3`、`t-dup-4`，这些标签唯一，不与文件中其他线程 id 冲突，且与每个测试已使用的 ask id 匹配。理由：测试中的 ask id 已经是 `dup-1`..`dup-4`；让线程 id 与 ask id 对齐可保持测试可读，并保证标签唯一。
- KTD3. 不重试、不休眠、不为冲突做变通式辅助函数修改。（会话已定：用户指示——选择此方案而非 bot 的带退避重试修复和辅助函数加固：日志证明失败是管道名称冲突导致的连接前 `EADDRINUSE`/`ENOENT`，任何连接重试或应答缓冲都无法修复。）对 `answerQueue` 的死代理快速失败改进推迟到后续，不属于本次修复。
- KTD4. 验证不能依赖单次绿色 Windows 运行，因为失败依赖时序（观察到的运行中四个同名测试只有 1 个失败）。要求确定性标签唯一性断言（R2）加上连续两次绿色 Windows 运行，才算满足完成定义。理由：空操作改动可能靠时序运气通过单次运行；确定性断言和两次运行门槛消除了这种歧义。

### 来源

- Windows 任务日志（管理员权限，通过已认证的 `gh` 获取）：`permission broker unavailable on \\.\pipe\openmausbot-perm-1892-t-perm-d: listen EADDRINUSE: address already in use`，随后在 `denies a colliding ask id from a second connection on the same broker` 测试上出现 `Error: connect ENOENT \\.\pipe\openmausbot-perm-1892-t-perm-d`。
- `server/drivers/claude.ts:192-195` — `permissionSocketPath` 的 8 字符标签截断。
- `server/procs.ts:114-119` — 使用 `process.pid` 构造 Windows 命名管道名称。
- `server/drivers/claude.ts:208-210` — `listen()` 之前的 `unlinkSync`（仅 POSIX 有效）。
- `server/drivers/claude.test.ts:438-561` — 四个冲突测试，线程 id 为 `t-perm-dup-1..4`。

## 实现单元

### U1. 给每个冲突测试唯一管道名称并断言标签唯一性

**目标：** 任何平台上都没有两个冲突测试共享代理管道名称，且未来任何标签再次冲突都会确定性失败。

**需求：** R1、R2、R3、R4。

**依赖：** 无。

**文件：** `server/drivers/claude.test.ts`（第 438-561 行四个测试中的线程 id；同文件中的新断言块）。

**方法：**
1. 修改四个冲突测试中的线程 id，使其 8 字符标签唯一：`t-perm-dup-1` → `t-dup-1`，`t-perm-dup-2` → `t-dup-2`，`t-perm-dup-3` → `t-dup-3`，`t-perm-dup-4` → `t-dup-4`。
2. 保持 ask id（`dup-1`..`dup-4`）不变；它们已与新线程 id 后缀匹配，保持测试可读。
3. 确认测试体中每个 `permissionSocketPath("<threadId>")` 调用都一致使用新线程 id（conn、respondToRequest、interruptTurn）。
4. 在同一文件中添加一个小型确定性断言（例如放在 `ClaudeDriver.decodeConfig` describe 块中或专门的 `it` 中）：`expect(new Set(["t-dup-1", "t-dup-2", "t-dup-3", "t-dup-4"].map(permissionSocketPath)).size).toBe(4)`。该断言在每个平台运行，未来任何标签再次冲突都会失败。

**测试场景：**
- 四个冲突测试在 POSIX 上仍以精确断言通过（R3）——改动仅是管道名称。
- 新的标签唯一性断言在 POSIX 和 Windows 上通过（R2）：`permissionSocketPath("t-dup-1")` → 标签 `t-dup-1`，依此类推，全部不同。
- 推理检查：`t-dup-1..4` 不与文件中其他线程 id 冲突（其他代理测试使用 `t-perm-abc`、`t-perm-stop`、`t-perm-2`；所有标签都不同）。

**验证：** `pnpm exec vitest run server/drivers/claude.test.ts` 在 macOS/Ubuntu 上通过；Windows CI 任务连续两次运行均为绿色。

## 验证契约

- `pnpm exec vitest run server/drivers/claude.test.ts` — 声明 33 个测试；在 Windows 上 33 个通过；在 macOS 和 Ubuntu 上 32 个通过，1 个跳过（第 66 行仅 Windows 的管道命名测试）。
- `pnpm typecheck` — 无错误。
- 在 macOS 上运行完整 `pnpm test`（vitest + broker:test + updater + packaged-server）。
- Windows 的 `typecheck + test (windows-latest)` 任务连续两次运行均为绿色。如果不是，获取任务日志（已认证的 `gh api repos/milind-soni/OpenMausBot/actions/jobs/<id>/logs`），在进一步改动前指出失败的测试；不要猜测第二个修复。

## 完成定义

- [ ] U1 已落地：四个冲突测试使用唯一线程 id/标签；没有两个共享管道名称。
- [ ] 标签唯一性断言（R2）已存在并在所有平台通过。
- [ ] 四个冲突测试保留其精确断言并通过。
- [ ] 未修改生产文件（diff 中 `server/drivers/claude.ts`、`server/permission-proxy.ts` 干净）。
- [ ] macOS 和 Ubuntu：claude 测试文件和完整 `pnpm test` 通过；typecheck 无错误。
- [ ] Windows CI 任务连续两次运行均为绿色。

## 推迟到后续

- `answerQueue` 错误/关闭拒绝：添加 `error`/`close` 处理器，使死代理以具名错误拒绝待处理的等待者，而不是挂起到 20 秒 vitest 超时。不属于本次修复——它不是失败模式（失败测试中代理从未启动，因此没有等待者挂在死连接上），且新分支将没有测试覆盖就发布。如果以后添加，需要一个实际在测试中途杀死代理并断言具名拒绝的场景。
- 代理仅 POSIX 的 `unlinkSync`-before-listen 的 Windows 等价物：平台不对称是根本原因；后续加固可以重试 `EADDRINUSE` 或以其他方式释放名称。此处不在范围内，因为唯一名称已避开冲突。