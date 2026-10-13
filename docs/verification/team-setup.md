# Reviewed Chief 团队设置

使用隔离的测试夹具，绝不要用线上应用或工作区。确定性集成配方会启动自己的临时服务器，只安装仓库中的 fake Claude 和 fake Codex 引擎：

```sh
pnpm exec vitest run server/team-setup.e2e.test.ts
pnpm exec vitest run server/bot-deletion-write-failure.e2e.test.ts
pnpm exec vitest run server/team-setup-requests.test.ts server/store.test.ts
pnpm exec vitest run server/index.test.ts -t 'delet|Full and Custom bots'
pnpm typecheck
pnpm lint
```

集成测试会在打印的服务器日志旁保留一份 `.team-setup.json` 证据文件，然后删除其夹具数据。它使用真实的活跃回合能力来调用内部工具，而非 owner bot-settings API。模型规划在此确定性测试中是模拟的；fake 引擎无法证明提供商认证和真实任务质量。

## 示例请求

在 Clive 被选为 Chief of Staff 后，询问：

> 用 Mira 设置 Research，用 Patch 设置 Engineering，用 Quill 设置 Growth。
> 给每个团队一个简洁的专家角色和常驻指令。从已连接的引擎中选用合适的
> 模型，若可用则包含不同提供商。给我看一份合并的设置审查。

Clive 通过 `list_team_setup` 发现确切的引擎/模型和已授权团队，然后调用一次 `propose_team_setup`。审查会合并每个 bot 的所有字段。新团队必须包含一个已创建或已移动的专家。卡片会明确列出新团队名称，以及 Clive 将获得的权限——用于协调并在其中提议设置变更。它无法授予对已存在但未授权团队的访问权限。名称限制为 60 个字符，Chief 最多可管理 100 个额外团队，与持久化和 owner 设置一致。

批准后，尝试：

> 将 Patch 移到 Growth，并将其默认模型改为另一个已连接提供商中的合适模型。
> 保留现有对话和权限。

仅 bot 默认值会改变，影响新线程和群组工作。现有线程保留其模型、批准级别和允许列表，包括此前继承默认值的旧线程。与当前权限级别不兼容的提供商会取消整个提议；设置绝不会静默降级或提升执行权限。新专家使用 Ask，且初始时禁用已连接应用。现有对等允许列表继续有效。

领导层设置接受 `fields.chiefOfStaff`，用于新 bot 或现有 bot。测试在已授权团队中的晋升、单批次中的显式替换、拒绝、冲突或新当选的 Chief、对等限制，以及卡片打开期间权限范围的丢失。降级会移除额外的受管团队授权；晋升不会复制请求方 Chief 的授权。验证重新加载和 `bots.json` 写入失败后，角色、授权和回执保持一致。`list_team_setup` 包含当前 Chief 分配。

对于删除，单独询问：

> 删除 Patch。先给我看删除确认。

`propose_bot_deletion` 会生成一份单独的破坏性审查，指明确切的 bot。生命周期守卫会拒绝活跃工作。在移除 bot 之前，服务器会发现并删除其确切受管的 Boat、VPS 容器和每 bot 的 Local VM（包括该 VM 的私有工作区）。共享团队计算机保持不动。提供商故障、未解决的所有权、未受管的名称冲突或未确认的提供商删除都会保留该 bot，以便重试删除。随后 bot 对话、记忆、指令和技能会被移除；生成的项目文件保留。Chief 无法删除自身。

## 断言与失败边界

确定性场景检查未知模型、伪造的调用方身份、合并、拒绝、批准、重复决策、范围授权、默认模型变更、团队移动、过期目标编辑、单独删除，以及每次决策恰好一次请求方继续——除非用户停止对话。对于直接任务和房间，当 Clive 仍在运行时都会拒绝，然后按下 Stop 并检查既不会出现新的提供商生成，也不会出现回复，包括在重复决策之后。它还验证来自活跃 loopback bot 的伪造 Origin 头无法批准设置或删除。受信任的桌面或已配对 owner 可以审查；无头本地浏览器批准会等到每个 bot 空闲。拒绝始终可以安全接受。

Domain 和 Store 测试覆盖注入的权限字段、隐藏/未授权目标、对等限制、忙碌/已删除/过期目标和源对话、已关闭卡片、60 字符/100 团队边界、模型不兼容、写入失败、回执重放，以及重新加载时保留旧任务设置。停止或删除请求方对话不会复活其卡片。

所有 bot 变更、新团队授权和结果回执都通过一次原子 `bots.json` 写入完成，然后才发布 bot 变更。单独的团队注册表先保存：如果随后 bot 写入失败，可能残留一个空的命名团队，但不会应用任何 bot、模型、成员资格或 Chief 访问变更。bot 记录移除同样会先保存其回执，再删除 bot 数据，但外部计算机无法加入该文件事务。当存在多个独立提供商时，一个已拥有的计算机可能在后续提供商失败前被移除；bot 保留，已完成的清理不会回滚，重试会安全地重新发现并移除剩余部分。删除写入失败夹具没有已拥有的外部计算机：它会阻碍确切的临时 `bots.json` 目标，执行真实的 DELETE 生命周期，并检查完整 bot、对话、例程、webhook 以及 solo/shared 日历调用在内存和磁盘上均未改变。它恢复该夹具文件并成功重试；无关的调用访客保留。不使用生产故障钩子。后续清理失败会与失败/未应用的设置区分报告。

## 已记录的应用与真实模型检查 — 2026-09-13

真实渲染器在隔离夹具中通过 `control:omb ui` 驱动：Clive 的待处理卡片显示了确切的 Research/Mira 设置和范围授权；Cancel 获得初始焦点，Apply setup 是唯一的正向操作，且没有出现 Always allow 选项。点击 Apply setup 创建了 Mira，清空了待处理 composer，显示 Team setup applied，并渲染了一次自动继续。验证运行保存了截图和保留日志。这检查的是共享 React 审查 UI，而非原生桌面打包。

另一次隔离运行分别使用 **真实 Codex gpt-5.6-luna** 和 **gpt-5.6-sol**，收到普通语言请求（无工具指令）。每个都调用了一次 `list_team_setup` 和 `propose_team_setup`，使用已连接模型恰好提议 Mira/Research、Patch/Engineering 和 Quill/Growth，然后在一次批准后报告成功，未再次询问。这是两个带日期的模型遵循样本，不保证每个提供商或任务都如此。运行后临时认证副本和夹具数据已移除；仅保留有界结果证据。

在注册表/导入修复后，基于 main 的合并集成使用真实 Luna 重复：恰好一次审查用所选模型和 Chief 授权创建了全部三个请求的 bot，然后 Chief 继续。证据：`/tmp/omb-live-team-0913.MwRY2k/setup-gpt-5.6-luna-1789255036346/result.json`。