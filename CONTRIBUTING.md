# 为 OpenMausBot 做贡献

感谢你愿意帮忙——本仓库已经合并过社区 PR，也欢迎更多贡献。
本文档说明如何搭建可用的开发环境、代码库对一次变更的期望，以及什么样的 PR 容易合并。动手前请先读一遍；它刻意写得很短。

## 基本规则

- **PR 要小而聚焦。** 一个 PR 只做一件事。既移植平台、又加功能、还重构的 PR 会被要求拆分。大改动：先开 Issue 商定方案。
- **匹配代码的"高度"。** 本代码库刻意保持小而直接——服务端用纯 Node、不用框架，一个 store，一个事件总线。三十行代码能搞定的事，别引入依赖。新增运行时依赖必须在 PR 描述里说明理由。
- **保持绿灯。** `pnpm typecheck && pnpm test` 必须通过。服务端改动需要测试（见下文）。
- **UI 改动需要截图。** PR 正文里放前后对比图；有动画的放视频。配色和风格要匹配 [`src/styles.css`](src/styles.css) 中现有的设定。

## 开发环境搭建

要求：**Node 24+**、**pnpm**，以及至少一个 agent CLI（[`claude`](https://claude.com/claude-code) 或 [`codex`](https://github.com/openai/codex)）并已登录，才能真正和 bot 对话。macOS 是主要发布平台，Ubuntu 24.04 x64 是 Linux 桌面 beta；harness 服务端本身是可移植的 Node，测试套件在 macOS、Linux 和 Windows 上都能跑。

```sh
git clone https://github.com/milind-soni/OpenMausBot && cd OpenMausBot
pnpm install

pnpm dev:server    # harness 服务端 → 127.0.0.1:8799
pnpm dev           # 应用 → http://127.0.0.1:5199
pnpm dev:desktop   # Electron 外壳（macOS/Ubuntu；保持 server + Vite 运行）

pnpm typecheck     # 应用 + 服务端
pnpm test          # vitest 套件（服务端单元 + 驱动契约 + API 冒烟）
pnpm test:watch    # 同上，watch 模式
pnpm exec vitest run --shard=1/4   # 单个 CI 分片，正是 CI 在该分片中运行的文件
pnpm check:electron # 对纯 JS 的 Electron 入口做语法检查

pnpm package:mac   # DMG + ZIP；需要 Swift/Xcode 工具
pnpm package:linux # Ubuntu x64 .deb + AppImage；不需要 Swift
```

`pnpm dev:desktop` 会在 Electron 启动前，下载并校验当前平台和架构对应的固定版本 Cloudflare Tunnel 连接器。后续启动会重新校验并复用已暂存的二进制。打包仍使用 `pnpm build:cloudflared`，它会暂存宿主桌面打包构建所需的全部架构。若只想暂存当前开发目标而不启动 Electron，运行 `node scripts/prepare-cloudflared.mjs --current`。

Ubuntu 安装和真实桌面检查见 [`docs/linux-desktop.md`](docs/linux-desktop.md)。

## Ubuntu 发布检查清单

Ubuntu 发布包必须来自手动触发的 **Package Ubuntu** 工作流，且基于确切的发布提交或标签，不能来自开发者工作站。Ubuntu 24.04 runner 会构建并校验两种格式，启动解包后的应用和 AppImage，在 Xorg 上通过无覆盖层的捆绑 Cua 运行时路由 `click` 和 `type_text`，运行 fail-closed 的 Wayland CUA 冒烟测试，并产出一个包含以下内容的发布构建产物：

- 带版本号的 `.deb` 和 AppImage；
- 供最新下载链接使用的稳定副本 `OpenMausBot-amd64.deb` 和 `OpenMausBot.AppImage`；
- `SHA256SUMS-ubuntu-x64.txt`，覆盖带版本号和稳定名称两种文件。

发布前，确认 `package.json` 中的版本号正确，并针对与其他平台相同的提交派发工作流。将全部五个 Ubuntu 文件附加到 [OpenMausBot releases](https://github.com/milind-soni/OpenMausBot/releases) 中对应的发布版本。然后在干净的 Ubuntu 24.04 x86_64 GNOME 环境中校验校验和文件、安装 `.deb` 并启动 AppImage。切勿把不同提交构建的包合并到同一版本下。

## 仓库地图

| 路径 | 内容 |
|---|---|
| `server/contracts.ts` | 驱动 SPI 和规范的运行时事件类型。整个架构都在这一个文件里——先读它。 |
| `server/drivers/` | 每个 provider 一个文件（Claude、Codex、Grok、cloud computer）。新增 provider = 一个文件 + 在 `builtIn.ts` 中加一行注册。 |
| `server/harness/` | 注册表（配置 → 活实例，未知 → 影子）和扇入事件总线。 |
| `server/index.ts` | 应用通信所用的 HTTP + SSE API。 |
| `server/testing/` | 测试替身：一个内存驱动，以及脚本化的假 `claude` / `codex` CLI。 |
| `src/` | React 聊天应用。自身没有传输层——HTTP 命令出去，一条 SSE 流进来。 |
| `electron/` | 桌面外壳：听写、屏幕捕获、本地 computer-use 守护进程。macOS 专属代码在这里，带门控。 |
| `dist-server/` | **构建产物。** 永远不要手改，永远不要放进 PR——它在发布时由 `pnpm build:server` 重新生成。 |

数据存放在 `~/.openmausbot/`（bots、transcripts、每线程的 NDJSON 事件日志、含密钥的配置）。

## 测试

测试套件与源码同目录（`server/**/*.test.ts`），用 `pnpm test` 运行。分三层：

- **单元** —— 注册表、总线、store。纯进程内，使用 [`server/testing/fake-driver.ts`](server/testing/fake-driver.ts) 中的假驱动。
- **驱动契约** —— [`claude.test.ts`](server/drivers/claude.test.ts) 和 [`codex.test.ts`](server/drivers/codex.test.ts) 会启动 `server/testing/` 中脚本化的假 CLI，断言规范事件流、argv/env 卫生、中断和权限代理。失败模式通过环境变量切换（`FAKE_CLAUDE_MODE=exit-early` 等）——扩展这些假实现，而不是 mock `child_process`。
- **API 冒烟** —— [`index.test.ts`](server/index.test.ts) 针对一次性 home 目录启动真实服务端，跑一遍 HTTP 接口。

测试的内部规则：

- **不要 sleep。** 等待能证明行为的事件（见 `server/testing/events.ts` 的 `recordEvents(...).until(...)`）。需要靠超时才能通过的测试是错的。
- **永远不要碰真实的 `~/.openmausbot`。** setup 文件把 `HOME` 指向临时目录；保持这样。
- 假 CLI 的 shebang 脚本必须通过 `spawnCli`/`execCli` 启动，它们在 Windows 上会经由 Node 解析。只有当行为本身确实与平台相关时才给测试加门控。

## 新增 provider 驱动

[`server/contracts.ts`](server/contracts.ts) 中的 SPI 刻意做得很小。驱动 PR 应当：

1. 新增 `server/drivers/<name>.ts` 实现 `ProviderDriver`，并在 [`builtIn.ts`](server/drivers/builtIn.ts) 中注册。
2. `decodeConfig` 遇到无效配置要**抛出**；`create` 失败时要**reject**（绝不同步抛出）——注册表会把两者降级为不可用的影子，而不是让整个集群崩溃。不要移除或绕过这一行为；正是它保证了配置的前后兼容。
3. 只发出携带自己 `driverKind` 的规范 `RuntimeEvent`——总线会丢弃跨驱动的事件。
4. CLI 缺失或损坏必须表现为 `snapshot() → { state: "unavailable", reason }`，启动失败表现为失败的回合——绝不挂起，绝不崩溃。
5. 附带一个遵循假 CLI 模式的契约测试（脚本化假进程 + `recordEvents`）。

## 面向 agent 的控制 CLI

在声称某个服务端或对话改动可用之前，请使用 [`docs/verification/README.md`](docs/verification/README.md) 中的隔离流程。对于面向自动化的新 CLI：

- 复用已有的 MCP 或 API 操作；CLI 只负责参数解析和结果格式化。
- 变更类命令必须显式指定目标或使用隔离启动器。绝不静默操作用户正在使用的应用。
- 成功和失败都返回 JSON，失败用非零退出码，并提供 `--help`。
- 错误信息要指明失败的动作和下一步有效操作。
- 删除或覆盖数据的命令要提供 `--dry-run`；并测试它不改变状态。
- 优先使用任务级子命令，并为主要工作流加一个冒烟测试。

验证功能图刻意保持不完整。只有当共享控制面能覆盖它、且有永久测试证明时，才添加条目。

## MCP 工具 Schema

工具的 `inputSchema` 在模型看到之前，会经过每个引擎自己的 MCP 到 provider 的转换，而这些转换是有损的：组合关键字会被扁平化、丢弃，或被体积压缩流程裁剪（codex 直到 2026 年中才开始保留 `oneOf`；其他引擎简化得更狠）。没见过你 schema 分支的模型会永远猜形状——0.1.38 发布几小时后聊天例行提案在现场失败（#544）正是如此。

- **工具 `inputSchema` 中绝不使用 `oneOf`、`anyOf`、`allOf`、`const` 或 `format`。** 只暴露一个扁平对象；把各变体规则写进 `description`。普通字符串上的 `enum` 没问题。
- **先强制转换，再拒绝。** 模型会把嵌套对象字符串化、缩短枚举值、改变大小写。如果某个输入只有一种明显含义，就接受它并在传输时归一化。
- **错误必须能教学。** 拒绝输入时，消息要说明支持的形状，并给出模型能照抄的字面示例。"Invalid discriminator value" 浪费一个回合；一个示例能修好下一次调用。
- schema 测试应断言工具面保持扁平（见 `server/drivers/agents-proxy.test.ts`——它用正则守护序列化后的 schema）。
- agents 工具位于 `server/drivers/agents-catalog.ts`，其序列化的 `tools/list` 由 `server/drivers/agents-catalog-wire.test.ts` 逐字节固定并做体积预算。故意改动后，用 `UPDATE_AGENTS_CATALOG_GOLDENS=1` 运行该文件一次，审查 golden diff，并手动调整其 `BUDGET_BASELINE`。

## 新增语言

渲染器的字符串存放在 `src/locales/` 下的 JSON 目录中。英语（`src/locales/en.json`）是唯一事实来源（类型也从它推导）。一种语言就是一个文件加一行注册，和 provider 驱动完全一样：

1. 把 `src/locales/en.json` 复制为 `src/locales/<code>.json`。文件名用小写 BCP-47 标签（`de.json`、`pt-br.json`）。翻译值；省略的键会回退到英语，所以部分翻译的社区包也没问题。
2. 在 `src/locales/index.ts` 中注册。
3. 运行 `pnpm i18n:check`，然后在 **Settings → General** 中选择该语言。

已认证的本地 Claude CLI 可以生成初稿；它绝不在 CI 中运行，输出仍需人工审查：

```sh
# 使用本地已登录的 Claude CLI
node scripts/generate-locale.mjs it "Italian"
```

该辅助脚本在无仓库访问、无自定义指令、无写入工具的情况下运行模型。它只接受一个完整的 JSON 对象，并为每条已审查的翻译记录英语源哈希。工作流和安全规则见 [`docs/localization.md`](docs/localization.md)。

目前只抽取了 UI 的一部分。改动组件时顺手把字符串用 `t("…")` 移入目录——绝不要大批量扫荡，那会和所有东西冲突。

## 平台规则

- harness（`server/`）必须保持可移植的 Node。任何 macOS 专属的东西（TCC、Swift 辅助程序、`~/Library` 路径）都归 `electron/`，并置于 `process.platform === "darwin"` 门控之后。
- 渲染器代码必须消费桌面能力契约，而不是从 Electron、user agent 或 preload 桥的存在来推断支持。屏幕预览、听写和本地控制是相互独立的能力。
- Ubuntu 平台相关声明要在真实 GNOME 会话中测试。Xvfb 只能证明打包和假驱动编排，不能证明 Wayland portal 行为或真实 CUA 检查/输入投递。
- Linux 本地控制仅在 GNOME/Xorg 上、且用户明确选择加入后才启用。自有守护进程必须以 `--no-overlay` 启动：装饰性的全屏 Cua 光标层不属于产品契约，绝不能挡在人和桌面之间。GNOME/Wayland 必须清除遗留的持久化选择加入，报告 `linux-wayland-seat-safety-blocked`，并且在独立通过 #345 中的真实座位矩阵之前绝不启动 Cua。Xvfb 能证明无覆盖层参数、生命周期和输入路由；它不能免除真实座位证据。任何获批操作之前，无关应用必须仍可点击/输入。全局选择加入加每 bot 的 **This computer** 仍是强制要求；Linux Auto、全自动/绕过模式、记住的授权和云端批准绝不能授权操作用户桌面。
- CUA 发现必须无 shell，并固定接受的归档、内部文件、manifest 和驱动契约。打包的 Linux 构建必须优先使用经审查的 ASAR 外运行时，并 fail closed，而不是执行环境 PATH 中的代码；源码/开发构建可使用经校验的显式/用户本地路径。绝不添加运行时下载器/自更新器，也不静默安装 GNOME 扩展。GNOME/Wayland 就绪必须要求其确切的 compositor/helper/portal 健康契约；绝不从 `WAYLAND_DISPLAY` 或 XWayland 推断。
- 原生发布改动必须更新已签入的 Cua 许可证报告/SBOM，保留 MIT/OFL/MPL 声明，通过恶意归档测试，并证明 `linux-unpacked`、`.deb` 和 AppImage 产物中的哈希一致。AppImage 还必须证明在其私有 `0700` 执行阶段中复制后哈希正确；当工具链产出 `0775` 而非 `0755` 时，绝不削弱通用路径校验器去接受 root 属主、组可写的 SquashFS 路径。
- **绝不为 shell 构建命令字符串。** 不用 `shell: true`，不通过 `cmd.exe` 加引号字符串启动——模型名、persona 和 MCP 配置 JSON 都经由 argv 传递，而 cmd.exe 的元字符展开是真实的注入类别。在 Windows 上，把 `.cmd` shim 解析到其 JS 入口，改为启动 `process.execPath`。
- 仅 POSIX 的调用（`process.kill(-pid)`、unix socket）需要带门控的 Windows 等价物（`taskkill /T`、命名管道）——而不是静默失败。

## 密钥

API 密钥是只写的：它们经 `PUT /api/config` 写入 `~/.openmausbot/config.json`，API 只报告 `configured` 布尔值。保持这样——不记录密钥日志，不在响应或事件中回显，不把它们写进 argv 让其他本地进程读到。

## 下游 Fork 与发布归属

在下游 Fork 中准备的改动，应在 Pull Request 中保留来源信息，而不是向上游源码树添加 Fork 专属品牌或归属声明。记录用作对比基准的确切上游提交、head 分支，以及最终变基后运行的检查。

Fork 维护者拥有其发布的二进制和更新通道。分发 Fork 前，审查应用名称和标识符、签名配置、更新元数据，以及每个 `electron-builder` 发布目标。绝不把 Fork 产物或更新元数据上传到官方 OpenMausBot 发布仓库，也绝不在功能 PR 中更改上游发布目标，除非该发布迁移已与维护者明确商定。

上游 PR 应只包含可移植的产品改动。把本地构建路径、账号名、凭据、私有端点、机器专属配置和 Fork 专属发布说明排除在提交和截图之外。

## 贡献许可

- 不需要 DCO 签署。只提交你编写的、或你有权按适用项目许可证贡献的代码。
- `enterprise/` 下的改动（源码可见，见 [LICENSING.md](LICENSING.md)）需要 [CLA](CLA.md)，在 bot 要求时于 Pull Request 中评论一次即可签署。`enterprise/` 之外的改动不需要 CLA。
- `enterprise/`、云接缝和许可文件有代码所有者；那里需要维护者审查。

## CI 一览

每个 PR 都运行相同的检查，每项作为独立 job，这样失败会自己报出名字。快速的静态 job 先跑；只有它通过后，昂贵的测试和构建 job 才进入 runner 队列。这为有类型、lint 或构建错误的提交节省容量，同时不跳过可合并 PR 的任何检查。如果 preflight 失败或必需测试 job 被跳过，必需门禁仍会失败。它给原本空闲的 runner 池增加了 preflight 时长；它不能治愈 GitHub 全局的调度积压。

- **typecheck + lint** —— 类型检查、lint、语言目录（`pnpm i18n:check`）、Electron 语法检查、生产 UI 构建。在 Ubuntu 上跑一次；这些都不与平台相关。
- **vitest (os, shard n/4)** —— 套件按平台分成四个分片：PR 上跑 Ubuntu 和 Windows；main、合并队列和手动运行还跑 macOS（macOS runner 稀缺，且只有少数测试是 macOS