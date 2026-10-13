# 组织库

本指南涵盖 **模板 → 来自 {Organization}**：Electron 转发的目录、带 `trust: "org"` 的 **添加**、标记、已撤回的发布版本、报告快照、**Bot → 技能 → 来自 {Organization}** 以及来源行。产品行为见 [../org-library.md](../org-library.md)。此处不涉及真实 Admin。目录通过 `POST /api/testing/org-library` 转发到一次性夹具中，该端点仅在夹具以 `OMB_TEST_ORG_LIBRARY_KEY` 启动时存在。

## 库单元

```sh
pnpm exec vitest run server/org-library.test.ts
```

此测试在一次性 home 中使用真实存储、例程管理器和技能存储。发布文件是已提交的夹具，与 Admin 提供的一致：按文件解析、打上发布者标记、使用规范字节。它检查以下内容：

- **目录（§7.3.1）。** 未知字段被忽略，错误条目被丢弃。错误的信封、其他组织的目录、摘要与字节不匹配、以及纯 http 的 Admin，均保留上一个目录。`formatVersion: 3` 的条目显示为"更新"，且其文件永不被读取。
- **文件（§7.3.2）。** 字节与名称不匹配的文件在读取时被拒绝：显示为未下载，添加返回 `503 blob_unavailable`，并报告失败。由其他发布者标记的文件被拒绝为 `invalid_package`。
- **转发（§7.3.3）。** `applyRelay` 在写入 `state.json` 或报告任何内容之前返回。随后进行协调和报告。
- **添加（§7.3.4）。** 技能开启，来源为 `org:`。例程暂停。Bot 携带 `installId`、`publisher`、`ref`、`sha256` 和 `agentKey`。每个 bot、群聊和例程部分都有 `r` 和 `w`，当"Scout"已被占用时 `w(name)` 为"Scout 2"。标记永不进入 `listRoutines()` 或技能列表。随后更新索引和报告。
- **幂等性（§7.3.5）。** 添加两次只产生一个团队。删除 `state.json` 后，下次转发会采纳记录，添加保持无操作。仅含技能的包第二次添加也是无操作，丢失 `state.json` 后会被采纳为仅含技能的包。登出会隐藏货架并保留副本。重新连接会识别安装。
- **应用中途停止的添加。** 测试在应用将停止的瞬间复制整个安装，并从副本重新开始。在第一个例程处停止，或在设置 leader 之前停止，下次启动会移除部分 bot、群聊和例程及其团队，报告 `failed`/`import_failed`，然后添加会以自己的名称添加整个团队及其 leader 和简介。在 leader 之后立即停止，团队被索引为已添加，添加为无操作。
- **已撤回（§7.3.6）。** 已撤回的发布版本其技能关闭、例程暂停、`status: "withdrawn"`，报告显示 `withdrawn_by_publisher`。之后重新开启的技能保持开启。消失的条目不改变任何内容。
- **移除（§7.3.7）。** 删除安装的最后一个 bot 会将其标记为 `removed` 并报告 `removed_locally`。然后添加再次生效。当团队提供的技能仍在某人自己的某个 bot 上时也是如此：技能保留在那里，添加会带回全部三个 bot。如果该发布版本随后被撤回，遗留技能被关闭一次，安装保持 `removed`。
- **人员删除的部分（`removedLocally`，§3.4）。** 删除一个 bot、群聊或一个例程会记录 `agent:<key>`、`room:<key>` 或 `routine:<key>`，团队保持 `installed`。删除 bot 也会记录其例程。再两次转发和一次直接重建不添加任何内容，也不写入任何新内容。重启保留列表，在没有库监听时删除的 bot 会在下次启动时被记录一次，即使列表已列出它。整个团队被删除时为 `removed`，所有六个部分均被记录，再次添加会启动空列表。再次有记录的部分会从列表中移除。
- **仅含技能的包。** 它注册时无记录，其技能按 bot 提供。添加一个会将其开启并打上标记。重复为 `409`。撤回包会关闭该技能。
- **无组织（§7.3.7）。** 不列出任何内容，不创建 `org-library/` 文件夹，不读取任何 bot 的技能状态，不报告任何内容。

## 通过夹具服务器的 HTTP

```sh
pnpm exec vitest run server/org-library.e2e.test.ts --silent=false
```

这会通过 `launchVerificationServer` 启动并清理自己的假引擎夹具，传入随机的 `OMB_TEST_ORG_LIBRARY_KEY`。它检查以下内容：

- 无转发时，`GET /api/org-library` 为 `{organization: null, packages: []}`。
- 转发路由在密钥错误时返回 `404`。
- 转发后，两个包均列为就绪，`GET /api/org-library/packages/:packageId` 返回已检查的文档。
- `POST /api/org-library/add` 返回 `201`，然后返回 `200 {alreadyAdded}`。
- `GET /api/bots`、`GET /api/routines` 和添加响应不包含 `packageBase` 或 `memberKeys`，因此标记留在服务器上。
- 技能开启，例程暂停，`state.json` 保存安装。
- `POST /api/org-library/skills` 将提供的技能放到 bot 上并开启。
- `null` 转发会隐藏货架并保留 bot。

## 渲染器

```sh
pnpm exec vitest run src/lib/org-library.test.ts
OMB_UI_E2E=1 pnpm exec vitest run scripts/testing/org-library-ui.e2e.test.ts --silent=false
```

第一条命令将货架卡片和预览渲染为标记。它检查：

- 添加仅对已检查的文件出现，且没有确认步骤；
- 已添加、"更新 OpenMausBot 以添加此项"、推荐徽章、等待发布版本的句子和"由……撤回"；
- "包含的技能 — 已开启"；
- 无团队的仅含技能预览；
- 来源行，从文件导入的 bot 不会获得该行。

第二条命令运行一次性的 `control-omb ui` 应用，密钥通过 `scripts/control-omb.ts` 传入。它按顺序检查：

- 任何转发之前，模板没有组织标签页。
- 转发后，**来自 Customer Co** 是第一个标签页并首先打开。
- **Sales desk 的详情**显示技能已开启。
- **添加 Sales desk** 一键添加团队。其技能开启，例程暂停。
- 卡片随后显示已添加。
- **添加 Sales 技能**就地添加。
- 第一个 bot 的 **身份**显示"来自 Sales desk 1.3.0 · Acme Partners"。
- **技能 → 来自 Customer Co → 为此 bot 添加 follow-up** 将技能放上并开启。

截图保存到 `.omb-scratch/verify-evidence/org-library-*.png`。

## 2026-09-24：实际运行内容

仅在 macOS（arm64）上针对一次性夹具运行。以下通过：

- `pnpm typecheck`、`pnpm lint` 和 `pnpm i18n:check`；
- 上述四条命令，包括带 `OMB_UI_E2E=1` 的无头渲染器运行；
- `server/package-import.test.ts`、`server/package-export.test.ts`、`shared/package-format.test.ts`、`server/bot-package.test.ts`、`server/skills.test.ts`、`server/routines.test.ts`、`server/store.test.ts`、`server/team-share.e2e.test.ts`、`src/components/ShareTeamDialog.test.ts` 和 `src/lib/team-import.test.ts`。

以上每一项都经过变异检查：破坏行为，观察指定测试失败，然后恢复修复。

- **添加：**
  - 组织技能落地时关闭；
  - 组织例程落地时开启；
  - bot 部分哈希未写入；
  - `w(name)` 取自文件而非已写入的内容；
  - 提供的技能以非 `org:` 来源添加。
- **索引：**
  - 丢失 `state.json` 后记录未被采纳；
  - 索引无操作被移除（由仅含技能的包捕获）；
  - 未监视删除；
  - 启动时在读取记录之前写入索引；
  - 在无组织的安装上读取技能状态。
- **已撤回的发布版本：**
  - 技能未关闭；
  - 对已撤回的安装再次操作。
- **文件和转发：**
  - 文件未对照其 SHA-256 检查；
  - 接受外部组织的目录；
  - 仍读取更新的格式；
  - 在确认之前运行协调；
  - 测试转发路由无密钥应答。
- **传输中的标记：**
  - 例程标记；
  - bot 的 `packageBase`；
  - 群聊标记。
- **渲染器：**
  - 卡片为未就绪的文件提供添加；
  - 预览显示技能到达时关闭；
  - 来源行缺少撤回信息；
  - 货架标签页被隐藏（由无头渲染器运行捕获）。

非生产资格：不涉及真实 Admin、Electron 转发、加密存储或打包应用。Electron 侧（获取、文件缓存、转发和发布报告）是单独的工作，由测试路由和测试写入的文件代替。

## 2026-09-24：PR #1768 的审查修复

修复了三个审查发现，仅在 macOS（arm64）上针对一次性夹具：

- 团队保持"已安装"，而它给另一个 bot 的提供技能被遗留，因此永远无法再次添加（并且，一旦修复，撤回也必须触及此类已移除团队遗留的技能）；
- 应用中途停止的团队添加被采纳为半成品；
- 从文件导入的 bot 显示新的来源行，这改变了无组织安装上的 Bot 设置。

以下通过：`pnpm typecheck`、`pnpm lint`、上述四条命令（包括带 `OMB_UI_E2E=1` 的无头渲染器运行，连同 `scripts/testing/team-share-ui.e2e.test.ts`），以及上一节列出的相邻测试加上 `src/lib/team-share.test.ts`。

以上每一项都经过变异检查，观察到指定测试失败：

- 提供的技能使团队保持存活（提供技能移除测试）；
- 导入前未写入 `adding` 条目（两个中途停止测试）；
- 部分团队被视为已完成（在第一个例程处停止）；
- 已完成的团队被视为部分（在 leader 之后停止）；
- 未检查 leader（在 leader 之前停止）；
- 部分团队的例程被保留（在 leader 之前停止）；
- 简介再次在 leader 之后写入（在 leader 之后停止）；
- 仅含技能的包被采纳为团队（丢失 `state.json` 测试）；
- 撤回跳过已移除团队遗留的技能，并在人员重新开启后再次操作它（已移除团队的撤回测试）；
- 文件标记显示来源行（`src/lib/org-library.test.ts`）。

未测试：实际进程终止。停止通过导入中途复制安装文件来模拟。每个 bot 的起始注释和部分哈希的写入顺序未被测试覆盖，因为夹具没有能区分它们的停止点。

## 2026-09-24：`removedLocally` 已填充（PR #1771）

包第二波的集成检查发现 `removedLocally`（契约 §3.2、§3.4、§6）始终为 `[]`：重建用剩余内容替换安装的 bot、群聊和例程，因此 v1.1 更新无法区分人员删除的部分和新发布版本添加的部分。重建现在在替换之前记录每个消失的键（`agent:`、`room:`、`routine:`），整个团队被删除时记录其所有部分并变为 `removed`。仅在 macOS（arm64）上针对一次性夹具运行。

以下通过：`pnpm typecheck`、`pnpm lint`、`pnpm i18n:check`，以及对 `server/org-library.test.ts`、`server/org-library.e2e.test.ts`、`src/lib/org-library.test.ts`、`server/package-import.test.ts` 和 `scripts/testing/verification-docs.test.ts` 运行 `pnpm exec vitest run`。

经过变异检查，观察到指定测试失败：

- 未记录键（全部六个 `removedLocally` 测试）；
- 整个团队被删除时未记录其部分（整个团队测试）；
- 再次存在的部分保留在列表中（恢复部分测试）；
- 列表未去重（重启测试）。

未测试：实际进程终止（重启测试处置库并在相同文件上打开新库）。丢失 `state.json` 后的采纳按设计启动空列表；见 [../org-library.md](../org-library.md)。非生产资格。