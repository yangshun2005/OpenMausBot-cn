# Issue 1706 — 设计

## 决策

让 `Markdown.blocks` 输出表格块和任务块。`MarkdownText` 负责绘制两者。`TextBubble` 不再把整条消息的管道表格替换为 `SQLResultTableView`，该视图将被删除。Walkie 朗读时跳过分隔行和复选框方括号。

Android 保留现有卡片。该部分由 #1707 处理。

## 块

`MarkdownBlock` 新增两个 case。现有 case 保持不变，因此当前 `.bullet(indent:text:)` 调用点仍可编译。

```swift
public enum MarkdownTableAlignment: Equatable, Sendable {
    case leading, trailing, center
}

public struct MarkdownTable: Equatable, Sendable {
    public var headers: [String]
    public var alignments: [MarkdownTableAlignment]
    public var rows: [[String]]
}

case table(MarkdownTable)
case task(indent: Int, number: Int?, checked: Bool, text: String)
```

不变式：`alignments.count == headers.count`，且每行单元格数相同。短行用 `""` 补齐。多行正文行若多出单元格，则用 `""` 扩展表头，对齐方式为 `.leading`。焊接的正文在构造结构体之前按表头宽度拆分为行，因此多出的单元格会开启下一行。分隔行不存储。

## 表格何时开始

表格行仅当其前导空白为 0–3 个空格且不含制表符时才被识别。四个空格或前导制表符仍视为文本。解析器维护一个打开的列表标记缩进栈。空行清空该栈。后续行若前导空白大于最内层标记，则属于列表。这些行不是表格：嵌套标记入栈，管道行保持为可见文本。缩进不超过最内层标记的行会弹出标记，直到某个标记仍包含它，然后独立解析。因此 `- parent`、嵌套的 `- child`、以及与 child 同缩进的管道行都保持为文本，而列表项之后位于第 0 列的表格仍是表格。顶层表格缩进 1–3 个空格仍是表格，因为栈为空。表头候选使用与正文行相同的排除条件：不是标题、引用、围栏、分隔线或列表项。`- [x] A | B` 后跟分隔行是任务，不是表格。

分隔行至少包含一个 `|`，且在去掉外侧空边缘单元格后至少有一个单元格，每个单元格都是分隔单元格。单独的 `|` 不是分隔行。`Pros | Cons` 后跟 `-` 是一个段落 `Pros | Cons -`，因为软换行将它们连接，且 `-` 不是项目符号或分隔线。

在现有的围栏、空行、分隔线、标题和引用检查之后，列表项之前：

1. **单行焊接。** 形状为 `splitInlineTable` 中的桌面救援逻辑，加上 `repairMarkdownTables` 对整个段落施加的反引号和反斜杠拒绝。`splitInlineTable` 本身不检查反引号。至多三个空格后，该行以 `|` 开头，不含反引号和反斜杠，包含分隔符序列，表头至少有两个单元格且以 `|` 结尾，剩余部分为空或以 `|` 开头。下一行本身不能是分隔行。`Example: | A | B | |---|---| | 1 | 2 |`、`| A | B | |---|---| then prose` 和 `| Step | --- | Done |` 保持为段落。焊接的正文单元格超出表头宽度时开启下一行，用空单元格补齐。行边界处恰好一个空单元格是连接点。行内真正的空单元格保留。后续管道行若未缩进在列表下，则延续同一表格。空行结束表格，与表头加分隔行路径相同，因此示例 2 的分隔行保持为段落。
2. **表头加分隔行。** 若本行是顶层行且下一行是顶层分隔行，则结束当前段落并输出表格。分隔行单元格数与表头不同时，适配表头宽度：保留提供的对齐方式，缺失单元格为 `---`，多余单元格丢弃。纯短横线为左对齐，包括为补齐短分隔行而合成的单元格。正文行在顶层行时继续。多行正文单元格超出表头宽度时，用空表头扩展表格。空行结束表格。不是正文行的行不被消费：外层循环重新解析它，因此 `After` 是段落，`- [x] done` 是任务。

正文行是顶层行，包含 `|`，去掉外侧空边缘后至少有一个单元格，且本身不是标题、引用、围栏、分隔线或列表项的开头。因此 `- a | b` 这样的行会结束表格。单元格按未被转义且不在反引号跨度内的 `|` 拆分。由外侧管道产生的首尾空单元格丢弃。中间空单元格保留。`\|` 变为字面管道，包括在代码跨度内，与当前 `tableCells` 辅助函数行为一致。其他反斜杠保留。

分隔单元格去除空白后匹配 `^:?-+:?$`。单元格以 `:` 结尾且不以 `:` 开头时对齐为 `trailing`，两者都有时为 `center`，否则为 `leading`。一个连字符即可，与 GFM 和 `src/lib/markdown-tables.ts` 中的 `DELIMITER_CELL` 一致。旧卡片要求三个连字符；该更严格检查随卡片一并移除。短分隔行补齐，长分隔行裁剪至表头宽度，即使单元格包含 `\|` 或代码跨度。桌面的修复会跳过这些段落，因为它不是分词器。本解析器会适配它们。焊接同样不是分词器，仍拒绝包含反引号或反斜杠的行。

未转义反引号数量为奇数的行不是表格。它保持为段落，这是流式和行内代码的失败模式，不会丢失内容。

围栏仍优先消费，因此打开的围栏运行到结尾，其中的管道是代码。单独的 `---` 仍是分隔线，因为该检查在表格检测之前运行。后面没有分隔行的管道行追加到段落。

本解析器不等待表格前的空行。桌面 CommonMark 不会中断段落，而 `repairMarkdownTables` 插入空行以使表格解析。在此识别表头加分隔行得到相同结果，无需第二遍。

## 任务项

`listItem` 仍识别 `- `、`* `、`+ ` 和有序标记（`1. ` 和 `1) `）。当剩余部分匹配 `[ ]`、`[x]` 或 `[X]` 后跟空白，或复选框就是整个剩余部分时，块为 `.task`。文本为去除空白后的剩余部分。有序任务在复选框旁保留其编号。`]` 后必须有空白，因此 `- [x]no-space` 保持 `.bullet`。`[  ]` 和 `[ x ]` 不是复选框。嵌套深度仍为 `min(leading / 2, 4)`。

## 绘制

`MarkdownText` 新增可选 `scrollIdentifier`。该视图中第一个表格将其放在水平 `ScrollView` 上。`TextBubble` 传入 `message-<message id>-scroll`。流式和文件预览传 nil。按块切换的方法仍为 `view(for:tail:)`。

表格是该滚动视图内的一个 `Grid`：表头、跨所有列的分隔线、然后正文行。可能没有正文行。每列宽度为该列中最宽的单行单元格。每个单元格使用该宽度、`lineLimit(1)` 和该列的对齐方式（`leading`、`trailing` 或 `center`），表头和正文均如此，因此无障碍框架是列而非文本自身宽度。单元格文本经过现有 `inline` 辅助函数。光标位于最后一个单元格，包括补齐的空单元格。表格保持在气泡内。

任务行显示有序编号（若有）为其自身文本 `"\(number)."`，位于任务的无障碍元素之外。`1.` 和 `1)` 都使用该格式。复选框图像和文字是一个元素，不是按钮，子元素被忽略。其标签为 `completed` 或 `not completed`，然后 `, ` 和渲染后的行内文本（当该文本非空时）。目录键位于 `ios/App/Localizable.xcstrings`：英文 `completed` / `not completed`，巴西葡萄牙语 `concluído` / `não concluído`。图像不是单独的 VoiceOver 元素，任何子标签都不包含 `[x]` 或 `[ ]`。光标使用项目符号行已有的同一 `tail`。

`FilePreviewView` 和 `StreamingBubble` 继续调用 `MarkdownText`。它们不新增第二个拆分器。

## 聊天气泡

删除 `TextBubble` 中的 `parsedTable`、`tableCells`、`isTableSeparator` 和 `SQLResultTableView` 分支。`customCard` 仅对 `parsedDiff` 保持 true，因此 diff 仍是 git 卡片，表格是普通气泡。删除 `ios/App/Cards/SQLResultTableView.swift`。iOS 目标中没有其他引用。

用户消息仍走 `mine` 分支和 `Text`，而非 `MarkdownText`。

## Walkie

`Walkie.speakable` 调用 `Markdown.blocks` 并朗读块。它不保留第二个焊接。表格每行贡献一句话，单元格用 `", "` 连接，空单元格省略，使用与解析器相同的外边缘丢弃。分隔行不存储，因此不朗读。段落仅当其整个文本（经相同边缘丢弃后）都是分隔单元格时才丢弃。即示例 2 的 `| :--- | ---: |`、`| - | - |` 和 `--- | ---`。埋在连接段落内的分隔行，例如列表项下的管道，不再是单独行，与该段落一起朗读。任何其他包含 `|` 的段落按单元格用 `", "` 连接朗读。标题、列表、引用和段落仍经过现有行内剥离，因此当前 Walkie 测试保持成立。任务仅朗读其文本：编号和复选框标记不朗读，单元格中单独的字母 `x` 会朗读。围栏代码块仍为 "Code omitted."。600 字符默认值和 `WalkieView` 的 400 字符调用保持不变。

## 主文本

在 `docs/ios-companion.md` 的 "Current status" 之后添加本节。它陈述当前行为，不提及 issue 编号。

```markdown
## Markdown

Bot and room replies are rendered. Messages the person typed stay literal.

Rendered blocks are paragraphs, headings, lists, task lists, quotes,
thematic rules, fenced code, and tables. Inline emphasis, code,
strikethrough, and links use Foundation attributed text. A table scrolls
horizontally inside the bubble. A task item shows a checkbox.

A top-level pipe table, indented by at most three spaces, is a table. A table
inside a list item, a blockquote, or a fence stays text or code, as does a
row indented by four spaces or a tab. The in-app spoken form reads each table
row and each task's words, and it skips a delimiter-only line and the
checkbox token. Closed-app voice is unchanged.
```

Current status 中现有的 "Markdown rendering" 项目符号保留。它仍然成立。

## 影响

| 变更内容 | 位置 | 类型 | 响应 |
|---|---|---|---|
| `Markdown.blocks`，新增 `MarkdownTable` / `.table` / `.task` | `ios/Sources/CompanionCore/Markdown.swift` | 每个 bot 气泡和 Markdown 预览的行为 | 在此实现，包括列表缩进记忆和未消费的表格终止符。 |
| 穷尽 switch | `MarkdownText.view(for:tail:)`、`MarkdownTests.text` | 缺少 case 时编译中断 | 添加两个 case。任务 case 传入 `tail`。 |
| `TextBubble.parsedTable`、`tableCells`、`isTableSeparator` | `ios/App/ChatView.swift` | 整条消息表格从 SQL 卡片变为气泡表格 | 删除它们。Diff 卡片保留。将 `message-<id>-scroll` 传入 `MarkdownText`。 |
| `MessageRow` | `ios/App/ChatView.swift` | 预览查询需要容器 | `accessibilityElement(children: .contain)` 和标识符 `message-<id>`。子元素保持独立。复制和选择文本仍在 `attachedContent.text` 上。 |
| `SQLResultTableView` | `ios/App/Cards/SQLResultTableView.swift` | 分支移除后成为死代码 | 删除文件。`Copy CSV` 目录条目可能保留未使用。 |
| 任务文字 | `ios/App/Localizable.xcstrings` | 新字符串 | 英文 `completed` / `not completed`。葡萄牙语 `concluído` / `não concluído`。 |
| `MarkdownText` 调用方 | `StreamingBubble`、`FilePreviewView` | 它们通过同一视图显示表格 | 不传滚动标识符。不新增第二个拆分器。 |
| `Walkie.speakable` | `Walkie.swift`，由 `WalkieView` 和 `WalkieController` 读取 | 朗读表格和任务 | 朗读 `Markdown.blocks`。不改变长度上限。 |
| 预览 fixture | 仅 `ios/App/ThreadPreview.json` | UI fixture | 需求中的父链。不要编辑 `ImagePreview.json` 或 `ThreadPreviewPages.json`。 |
| UI 断言 | `ThreadNavigationUITests` | CI 已运行该类 | 在此添加验收 14 检查，不要新建类。 |
| Android 注释 | `Transcript.kt`、`TranscriptTest.kt`、`ExecutionFeedbackTest.kt`、`TableReadingOrderTest.kt`、`TranscriptCardViews.kt` | 注释将提及已删除文件 | 描述 Android 自己的卡片并指向 #1707。不改变 Kotlin 行为。不要编辑 `Markdown.kt`。 |

不受影响：

- `TextBubble` 中的用户消息 `Text`。它从不调用 `Markdown.blocks`。新标识符仍包裹该行。
- `parsedDiff` / `GitPRDiffCardView`。仅为 diff 的消息仍走该卡片。
- 桌面 `src/lib/markdown-tables.ts` 和 `ChatMarkdown.tsx`。无共享代码。
- 服务器、sidecar 和 `ios/Tests/CompanionCoreTests/Fixtures` 下的 fixture。传输载荷不变。
- Widget 和 roster 预览。它们显示简短纯文本行。
- 消息搜索片段和暂存发送预览。它们保持原始源。
- `android/.../Markdown.kt`。它不引用已删除的 Swift 文件。

手段检查：`Markdown.blocks` 由 `MarkdownText` 和 `Walkie.speakable` 消费。滚动标识符仅由 `MarkdownText` 内第一个表格消费。消息标识符由预览 UI 测试消费。

## 测试

`MarkdownTests` 覆盖验收 1 到 10：两个散文加表格字符串、焊接空单元格、短尾、多行增长、焊接换行、短和长分隔行、居中 `:---:`、可选外侧管道、一个连字符、一列、`Pros | Cons\n-` 为一个段落、焊接否定、围栏（包括打开的围栏和围栏内 `- [x] literal`）、空行、表格后接 `After`、表格后接任务、四个空格、制表符、列表延续、引用、1–3 空格顶层表格，以及验收 9 中的每个任务行。流式检查是新测试，不是 `testNoPrefixOfAReplyLosesCharacters` 的扩展。它遍历这些源以及 `-5%` / `12:30` / `$3` / `x` 行的每个前缀。完整源的期望字符串手写，每个前缀按同一手写规则检查：仅在解析器消费处丢弃结构，不因字符类别而剥离 `-`、`:` 或 `$`。解析器用单个空格连接的空白规范化为该空格。`Quant baskets` 不得塌缩为 `Quantbaskets`。

`WalkieTests` 锁定两个精确朗读字符串。

变异，每个使指定测试失败，提交前记录：

| 变异 | 失败 |
|---|---|
| 不输出 `.table` | 验收 1 表格断言 |
| 不输出 `.task` | `+ [X]` 行和嵌套父/子行 |
| 将 `Pros \| Cons` 后跟 `-` 视为表格 | 验收 5 |
| 拆分代码跨度内的管道 | 验收 4 |
| 在朗读中保留分隔行或复选框 `x` | 验收 11 的第一个字符串 |
| 将 `:---:` 视为左对齐 | 验收 4 的居中列 |
| 在焊接多余单元格上增长第三列 | 验收 3 的换行 |

UI，位于 `ThreadNavigationUITests` 内，因此现有 CI 选择会运行它。使用 `-store-preview -threads-preview` 启动，不带 `-images-preview`。不要编辑 `ImagePreview.json`。`ThreadPreview.json` 链为 `preview-gmail-user` → `preview-gmail-grid` → `preview-gmail-tasks` → `preview-gmail-user-md` → `preview-gmail-reply`，`activeLeafId` 保持 `preview-gmail-reply`。查询保持在 `message-<id>` 内。网格检查单元格顺序、共享中心线、列宽、长 token 的高度、移动它的左滑、VoiceOver 顺序，以及不存在 `DATA TABLE`、`Copy CSV`、`rows` 和 `| --- | --- |`。任务消息检查 `Quant baskets`、`1.`、`completed, Ship the notes`（不是按钮）和 `not completed, waiting`。用户消息仍显示其分隔符和 `[x] done`。附上截图。现有导航和编辑器测试仍通过。恢复 `parsedTable` 会使网格检查失败。同一类也在 CI iPad 目标以及手机上运行。

## 验证

从 `ios/`：

```sh
swift test
```

然后按 `docs/verification/ios-threads.md` 所述生成项目