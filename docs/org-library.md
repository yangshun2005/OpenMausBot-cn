# 组织库

组织的 Admin 可以将包（整个团队，或单独的技能）共享给属于该组织的桌面端，以及其客户。在 OpenMausBot 中，它们显示在 **Templates → From {Organization}** 下的第一个标签页。点击一次 **Add** 即可添加一个包。没有确认步骤，因为点击本身就是决定。**Details** 会打开与共享文件相同的预览。

没有组织账户时一切不变：标签页不存在，下面所有路由都返回 `{organization: null}`，不写入任何文件，也不上报任何内容。

## Add 做了什么

包通过与文件相同的导入器（`server/package-import.ts`）处理，仅由 `server/org-library.ts` 以 `trust: "org"` 调用，其他任何地方都不会这样调用。与文件相比：

- **技能到达时已开启。** 组织的 Admin 发布了它们，所以 Admin 就是审核。其来源显示为 `org:<publisher>/<package>@<release>`，文件自身的 `source` 被忽略。
- **例程到达时仍为暂停。** 机器人仍以 **Ask** 启动，没有已连接应用的访问权限，使用安装的默认模型。连接仍以关闭状态创建，不带值。
- **每条记录都带有其安装的标记。** 机器人的线路上可见的 `installedPackage` 增加 `source: "org"`、`installId`、`publisher`、`ref` 和 `sha256`。仅在服务端，每条记录还为每个部分保留两个哈希：`r`，发布时的部分；`w`，此处写入的部分。哈希存在于机器人（`packageBase`）、群聊（`installedPackage`）、例程（`installedPackage`）和技能状态（`package`）上。后续的自动更新用它们区分你的编辑与发布者的更改。值的结构见 `server/package-parts.ts`。
- **仅技能的包**不创建记录。其技能是*被提供*的：**Bot → Skills → From {Organization}** 会列出它们，在那里点击 **Add** 会将一个技能放到该机器人上，开启并打上标记。团队中没有任何机器人使用的技能也以同样方式提供。提供的技能从已添加的发布文件中读取。Electron 在目录不再列出某个发布文件一周后删除它，之后这些技能不再提供，但已在机器人上的技能保留。
- **添加两次无效果。** 安装 id 由 Admin 地址、组织和包派生，因此同一个包在每次重连时都是同一个安装。第二次 Add 返回 `200 {alreadyAdded: true}`。

从组织库添加的机器人会在 **Bot settings → Identity** 中显示其来源："From Sales desk 1.3.0 · Acme Partners"。从文件导入的机器人不显示此行，与之前一样。此版本从不更改已添加的内容。当目录列出更新的发布时，卡片显示 "Version X available. Updates arrive automatically in an upcoming OpenMausBot update."

**Required** 在此版本中按 Available 处理，带有 "Recommended by …" 徽章。组织**关闭**的条目会被隐藏，其副本保留。

## 已撤回的发布

当目录在 `withdrawnReleases` 下列出已添加的发布时，其技能被关闭（在其自己的机器人上、在添加了所提供技能的机器人上，以及有人从其预设创建的机器人上），其例程被暂停，其预设从 New bot 中移除，安装被标记为 **withdrawn**，机器人的来源行增加 "Withdrawn by <publisher>"。
这在变更时发生一次。如果你之后重新开启某项，它会保持开启。你已移除的团队保持移除，但它留在另一个机器人上的所提供技能会以同样方式关闭一次。从目录中消失的包不改变任何东西：副本保留。

## 目录如何到达

此运行时从不与 Admin 通信。Electron main（`electron/managed-desktop.mjs`）负责 Admin 连接：它获取目录和每个发布文件，校验其 SHA-256，并将文件写入 `DATA_DIR/org-library/blobs/<sha256>.json`。然后它通过私有工具端口转发目录：

```jsonc
// Electron main → runtime
{ "type": "openmausbot:managed-library", "requestId": "…",
  "library": { "adminOrigin": "https://admin.example.com", "organizationId": "<uuid>",
               "organizationName": "Customer Co", "digest": "<sha256 of the catalog body>",
               "catalog": "<the raw catalog body, or the parsed object>" } }   // or library: null
// runtime → Electron main, at once, before any other work
{ "type": "openmausbot:managed-desktop-result", "requestId": "…", "ok": true }
```

- 以字符串发送的目录正文必须哈希到 `digest`。
- 运行时按契约 §5.3 的定义读取目录：未知字段被忽略，错误条目被丢弃。错误的信封，或属于另一个组织的目录，会被拒绝（`ok: false`），最后一个有效目录保留。
- `library: null`（登出、过期或撤销）隐藏书架，其他不变。
- 发布文件在每次读取时都校验其 SHA-256，且绝不通过符号链接。不匹配的文件视为未下载。

确认后，运行时协调其索引，处理已撤回的发布，并发回完整快照：

```jsonc
// runtime → Electron main, which posts it to POST /api/desktop/library/report
{ "type": "openmausbot:managed-library-state", "digest": "<applied digest>",
  "packages": [ { "packageId": "<uuid>", "release": "1.3.0", "sha256": "…",
                  "state": "installed" | "failed" | "removed" | "withdrawn",
                  "reason": "blob_unavailable" | "invalid_package" | "import_failed" | "newer_app_required"
                          | "removed_locally" | "withdrawn_by_publisher" } ] }
```

运行时在三个时间点发送此快照：

- 每次转发目录后；
- 每次 Add 后，无论成功或失败；
- 在人员删除某个安装的最后一条记录后。

去抖、重试和每次启动一次的上报由 Electron main 负责。上报只列出当前组织的安装。它从不包含名称、路径或错误文本。

## 文件

`DATA_DIR/org-library/`：

| 文件 | 写入者 | 内容 |
|---|---|---|
| `state.json` | runtime | 已添加内容的索引（契约 §3.4），加上每个安装的 `kind`、`name` 和 `withdrawnHandled`，以及 `adding`：已开始但未完成的团队 Add。模式 0600。 |
| `blobs/<sha256>.json` | Electron | 发布字节，每次读取时校验。 |
| `catalog.json` | Electron | 最后应用的目录正文。运行时不会读取它。 |
| `presets.json` | runtime（`server/presets.ts`） | 预设机器人，来自文件和此处的安装（[presets.md](presets.md)）。某个安装的预设列在 `state.json` 的 `presets` 下。 |

`presets.json` 中的预设行也带有其安装 id。当某个预设和技能包（无机器人）的 `state.json` 条目丢失时，它们用于重新识别该包，并为被采用的安装恢复其 `presets`。它们从不保持团队已添加，从团队预设创建的机器人也不保持（该机器人的技能也不保持，尽管它们带有安装的技能标记 `via: "preset"`），因此人员删除的团队可以再次添加；其预设随后原地刷新，绝不重复。
New bot 从内存中的状态为每个安装查询库状态（`OrgLibrary.installStatuses()`）。

记录是事实来源。`state.json` 在以下情况下从它们重建：

- 启动时；
- 每次 Add 前；
- 每次转发目录后；
- 每当机器人或群聊被删除，或团队被重命名时。

团队的区段从其机器人现在所在的位置读取，因此重命名团队无需钩子。只要其任何机器人、群聊或例程尚存，团队就算已添加。有人放到另一个机器人上的所提供技能是副本：它保留，但不保持团队已添加，因此一旦团队自己的记录消失，安装即为 `removed`，**Add** 会带回整个团队。

每个安装的 `removedLocally`（契约 §3.4）列出人员删除的已添加团队的部分：机器人用 `agent:<key>`，群聊用 `room:<key>`，例程用 `routine:<key>`。自动更新（v1.1）读取它以区分人员删除的部分与新发布添加的部分，绝不带回已删除的部分。它这样维护：

- **何时记录。** 每次重建索引时，索引曾持有但不再有记录的键会被加入列表一次。再次重建，或重启，不会添加任何内容。例程在其机器人被删除时停止计数，因此删除机器人也会记录其例程。删除例程不是存储变更，因此在下次重建（下次目录、Add 或启动）时记录。
- **整个团队被删除**变为 `removed`，其每个部分也被记录。这样无论人员是一次删除团队还是一个机器人一个机器人地删除，列表都相同。
- **再次有记录的部分**（从备份恢复的记录）会从列表中移除，因此它绝不列出存在的部分。
- **移除后 Add** 带回整个团队，因此它开始一个新的空列表。

如果应用在记录写入后、索引写入前停止，一旦目录列出其包，记录就会被采用，第二次 Add 仍是无操作。被采用的安装没有团队部分哈希，因此后续更新将这些部分视为已编辑并保留它们。其 `removedLocally` 从空开始：剩余记录无法显示人员在 `state.json` 丢失前删除了什么，因此后续更新可能带回那些部分。`state.json` 无法读取时同样如此。

如果应用在添加团队中途停止，下次启动会在 `adding` 下找到其条目（在第一条记录前写入）并查看已写入的内容。导入器在机器人的技能和起始笔记之后写入其部分哈希，简报在群聊和例程之前，领导者最后。因此：

- **每个机器人、群聊和例程都在且领导者已设置：** 团队被索引为已添加，如同上面的崩溃情况。
- **有任何缺失：** 部分添加的机器人、群聊、例程及其新团队被再次移除，上报为 `failed` 并带 `import_failed`。**Add** 随后带回整个团队。可能留下两样东西：其连接槽（关闭，无值，在其 MCP 服务器名称下），以及其机器人的图片文件，失败的导入也会留下这些。

## 路由

这些路由使用 admin 作用域，与导入相同。

| 路由 | 响应 |
|---|---|
| `GET /api/org-library` | `{organization, packages: [entry + blob: "ready"\|"unavailable"\|"unsupported" + installed], installs}`。无组织时为 `{organization: null, packages: []}`。 |
| `GET /api/org-library/packages/:packageId` | `{package, document}`：预览，从已校验的文件读取。 |
| `POST /api/org-library/add {packageId}` | `201` 带已添加的记录 · `200 {alreadyAdded: true}` · `404` 未列出或已关闭 · `409 newer_app_required` · `503 blob_unavailable` · `422 invalid_package` |
| `GET /api/org-library/skills?botId=` | 已添加包的所提供技能，带每个机器人的 `added`。 |
| `POST /api/org-library/skills {botId, installId, name}` | `201 {skill}`：将一个所提供技能放到机器人上，开启。 |

测试夹具没有 Electron 父进程。测试时，`POST /api/testing/org-library {library}` 改为转发目录。该路由仅在服务器以 `OMB_TEST_ORG_LIBRARY_KEY` 启动时存在，请求在 `x-openmausbot-test-org-library` 中携带该键。生产环境从不设置该键。

## 此版本不包含

这些在 v1.1 到达，格式不变：

- 自动更新；
- 带自动安装的 Required；
- 遵循 `offAction: "remove"`；
- Admin Withdraw 按钮。

来自包的预设机器人进入 **New bot** 下的 **From {Organization}**，从预设创建的机器人其技能会开启。已撤回发布的预设不再提供。见 [presets.md](presets.md)。

关于如何测试，见 [verification/org-library.md](verification/org-library.md)。