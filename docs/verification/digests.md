# 工作摘要与引擎 Hook

这里记录的是观察到的工作，而非任务成功的证明。命令回执只对同步 SQLite 写入去重；它不会让文件或提供商操作变为事务性，也不授权部署。

## 驱动与证据

```sh
pnpm exec vitest run server/digest-control.e2e.test.ts
pnpm exec vitest run server/commands.test.ts server/store.test.ts server/message-db.test.ts server/digest.test.ts server/checkpoints.test.ts server/hooks/omb-hook.test.ts server/hooks.e2e.test.ts server/digest.e2e.test.ts
```

第一个测试启动 `launchVerificationServer`，然后用共享的 `control:omb` 接口创建两个 bot，发送连续的直接对话轮次，等待结算，创建房间并向其发送消息，再读取两份记录。它检查每轮的 hook 结果（包括复用的工具 ID）、唯一持久化的回执以及房间发言人。命令、等待结果和记录都保留在打印的服务器日志旁边。其临时主目录在 `finally` 中删除；不涉及真实账户或用户工作区。

其余隔离测试套件覆盖事务回滚、嵌套保存点、广播前提交、hook 脱敏、有界预览与私有结果文件、压缩上下文、退出选项、检查点文件列表以及重放到另一个模拟引擎。多引擎和较旧的 hook 套件目前运行在 POSIX 上；共享控制测试是跨平台的。CI 必须确立 Windows 行为，而不能把本地 macOS 运行当作 Windows 验证。

## 无记录工作的回复

Android 和桌面端会省略空回执。两个移动端解析器都在末尾的 `reply:` 字段处停止，包括回复中包含 ` · ` 分隔符的情况。已记录的工具、文件、记忆变更、截断通知和未知工作区段仍然可见；存储的摘要不会被删除。

回归检查：

```sh
pnpm exec vitest run src/components/DigestChip.test.ts
(cd ios && swift test --filter RoutineRunDigestTests)
(cd android && ./gradlew :core:test --tests '*TurnDigestTest' --tests '*ChatPreferencesTest' :app:testDebugUnitTest --tests '*TranscriptPresentationTest')
```

对于真实桌面渲染器，请遵循[聊天 UI 配方](chat-ui.md)，以 `--tool-calls '[]'` 启动并设置 `features.showToolCalls=true`。发送 `hello` 并等待结算：回复保持可见，记录中不包含 `[data-testid="digest-chip"]`。在一个新的临时 fixture 中用 `--tool-calls '[{"name":"Bash","input":{"command":"echo hi"},"ok":true}]'` 重复：此时工作回执必须保持可见。保留两份等待结果和 DOM 快照。Android 的记录套件在确认空回执不存在后打开一个有用的回执，并保存 `digest-only-recorded-work.png`。

## 边界

- 摘要包含可见的工具调用、有界的记忆变更和简短回复。它基于观察到的行，而非推断的提供商能力。
- 原生会话自身的摘要与其已完成的回复一起计入，不会作为未读消息发回。重建或切换的会话仍会收到活动分支的摘要。
- 文件列表仅覆盖可检查点的本地工作文件夹。被排除的文件、远程计算机以及失败/超时的捕获不予体现。
- 文件捕获有三秒预算，到期即取消。工作区在捕获结算前保持占用；迟到的结果只能更新其原始的、仍处于活动状态的消息和任务代次。
- `full` 表示每条已记录的工具行都通过 hook 交付了未截断的结果。它不表示测试通过。否则证据为 `preview` 或 `none`。Hook 有自己的四秒超时、有界输入且不跟随重定向。`OMB_HOOKS=0` 会禁用它们。
- 桌面端 chip 和原生移动端呈现需要各自的渲染器检查。这些服务器测试不证明布局或真实提供商的 hook。