# CI 任务选择

每个 PR 仍会运行工作流及其静态检查：语言区域校验、类型检查、lint、Electron 检查、UI 构建、CI 选择测试和验证文档检查。选择逻辑绝不会跳过整个工作流。

- 仅根目录 Markdown 文件、`docs/` 下的 Markdown 以及 `.github/FUNDING.yml` 不运行运行时任务。
- 任何其他变更都会运行运行时测试套件。
- 推送到 main 按其自身提交范围（`before..after`）判定：仅文档变更，或仅提升 `"version"`（发布版本号）的 `package.json` 变更，会跳过运行时任务。仅当推送前的提交自身有绿色 CI 运行时才如此。main 的等待运行会被每次推送替换，因此没有该检查，被替换运行的代码将永远不会被测试。
- 合并组和手动运行始终运行所有任务。空或不可读的 diff，以及前一个提交的 CI 无法读取时，也回退到运行所有任务。

PR 选择使用本地 merge-base diff 并禁用重命名检测，因此将源文件移入文档目录仍会选择其原始运行时路径。不存在变更文件 API 截断，也没有新增 action 依赖。

单一的 `CI` 门禁保留现有的必需依赖。仅当静态验证通过且明确选择了仅文档路径时，才接受被跳过的运行时任务。失败、取消、缺少选择以及意外跳过都会使门禁失败。现有的建议性任务仍为建议性。

独立的共享终端冒烟工作流仅限手动：其测试已在 Windows Vitest/Electron 任务中运行。

## Vitest 分片

测试套件一次运行一个文件，因此一个分片的耗时等于其文件耗时之和。Vitest 自带的 `--shard` 按文件数量均分，2026 年 10 月时这导致四个最慢的 e2e 文件中有三个落在同一分片：Windows 上约 24 分钟测试，而其他分片为 13 到 16 分钟，每个 PR 都要等它。`scripts/testing/duration-sequencer.ts` 改为按记录时间分配文件，最慢的优先，每个分给当前累计时间最少的分片。时间是 `scripts/testing/vitest-shard-weights.json`：每个文件在 Windows runner 上的中位秒数，涵盖所有超过五秒的文件和所有 e2e 文件。不在其中的文件计一秒，若是 e2e 文件则计 e2e 中位时间。每个文件仍恰好运行一次，每个分片任务会记录一行 `duration-sequencer:` 及其份额。

当分片耗时出现偏差时，从最近几次绿色运行刷新时间：

```sh
node scripts/testing/update-shard-weights.mjs --run <run id> --run <run id> --run <run id>
```

在权重中删除或重命名测试文件会导致 `scripts/testing/duration-sequencer.test.ts` 失败，直到其条目被移除或刷新。刷新会将部分文件移到其他分片；仅在刷新后出现的失败是测试文件之间真实的顺序依赖。

## macOS runner

该账户最多同时运行五个 macOS 任务，而一个 PR 过去会排队七个（四个 Vitest 分片、两个冒烟、带一小时模拟器套件的 iOS 任务）。有 25 个打开的 PR 时，macOS 任务中位等待六个半小时。现在：

- PR 在 Ubuntu 和 Windows 上运行 Vitest 分片；main 推送、合并组和手动运行才加上 macOS 分片。只有少数测试块是 macOS 专属。
- 每个 PR 的 macOS 检查是一个任务：打包服务器冒烟和 Electron 冒烟。
- `ci-stop-closed.yml` 在 PR 被合并或关闭时取消其 CI 运行。PR 运行共享以 PR 编号命名的组（`ci-pr-<n>`），绝不使用 `github.ref`：已合并 PR 的 closed 事件会将基础分支报告为 `github.ref`，这曾导致每次合并都取消 main 的 CI（2026 年 10 月 3 日修复）。

## main 与发布

main 保持一个 CI 运行进行中、一个等待中。每次合并替换等待中的运行，进行中的始终完成，因此一波合并只消耗两次完整运行，而非每次合并一次（2026 年 10 月有十五个排队，发布版本排在它们后面）。

`release.yml` 只发布代码通过 CI 的提交（`scripts/release-ci.mjs`，与平台构建重叠）。仅当其 `CI` 门禁通过且每个 vitest 任务确实运行并通过时，一次运行才算数，因此有范围限定的运行（仅文档或仅版本号，跳过 vitest）永远不算证明。脚本按顺序接受：

1. 发布提交自身的一次已完成运行；
2. 任意具有相同 git tree 的提交上的一次已完成的推送、合并队列或手动运行。这是 `prepare-release.yml` 在 `release/v<version>` 分支上启动的全 OS 手动运行，前提是 main 在合并前未移动；
3. 当发布提交相对其父提交的唯一变更是 `package.json` 的 `"version"`（版本号提升的全部改动）时，父提交按 1 或 2 的证明。父提交运行仍在进行时会等待；红色父提交，或跳过测试的父提交，永远不算数。

若以上都不存在，且 main 对该提交的运行被替换、从未运行或跳过测试，脚本会在自己的通道中对该提交启动 CI：一个任何合并都无法触及的 `release-ci/v<version>` 分支，之后删除。红色判定会停止发布；重新运行失败的 CI 任务（`gh run rerun <id> --failed`），等待中的发布版本会采用新的尝试。所有失败任务都从未启动任何步骤的红色（macOS runner“获取失败”）不是判定。在发布通道自身的运行上，脚本会自行重新运行失败任务一次（`POST /actions/runs/<id>/rerun-failed-jobs`，任务的 `actions: write` 允许此操作）并继续等待。main 的运行绝不会以该方式重新运行：重新运行会加入 main 的并发组，在那里它会取消最新合并的等待运行，并从较旧的 tree 重新运行仅 main 的部署。脚本改为在通道中启动 CI。手动发布可用 `ship_without_ci` 跳过等待，仅限紧急情况。

## 不稳定测试与 Windows 超时

已知不稳定测试列在 `scripts/testing/ci-retry-list.json` 中：文件、完整测试名（describe 块与测试用 ` > ` 连接）、负责人、证据以及列入日期。仅在 CI 中（`CI=true`），仓库的 vitest runner（`scripts/testing/ci-retry-runner.ts`）为每个列出的测试提供两次重试；其他都不重试，本地运行从不重试。重试按文件受限：一旦列出的测试三次尝试全部失败，即为真实失败，该文件后续列出的测试不再重试。否则，破坏整个 fixture 的回归（六个列出的测试共享 `server/delta-context.e2e.test.ts`）会使其墙钟时间增至三倍，并将 Linux 分片推过其 20 分钟上限，而超时的任务会丢失失败测试的注解和此摘要。代价是：同一文件中列在损坏测试之后的真实不稳定测试在该次运行中不会重试。需要重试的列出测试会写入 vitest 任务的摘要并作为警告注解抛出（`scripts/testing/ci-retry-summary.mjs`），因此通过的不稳定测试仍可见。修复测试后删除其条目；`scripts/testing/ci-retry-list.test.ts` 会在条目对应的测试已不存在时失败。

Windows runner 运行套件比 Linux 慢约 1.45 倍。那里每个测试和 hook 超时都翻倍（`server/testing/host-timeout.ts`）：`vite.config.ts` 中的默认值，以及测试自身以 `hostTimeout(ms)` 书写的超时。Linux 和 macOS 保持严格数值，例如 3,000 文件检查点测试的 120 秒预算。

打包服务器冒烟向操作系统请求空闲端口（监听端口 0），而非随机选取：在 Windows 保留范围内随机选取会以 `listen EACCES 127.0.0.1:497xx` 失败。

## 必需检查

`main-ci-gate` 规则集要求单一的 `CI` 检查（它于 2026 年 9 月取代了三个旧有的 `typecheck + test (<os>)` 名称）。重命名 `gate` 任务需先更新规则集，否则每个 PR 都会永远等待。