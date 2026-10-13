# 一个浏览器引擎：agent-browser

**决策（2026 年 9 月 6 日）：** 机器人的浏览器统一为 [agent-browser](https://github.com/vercel-labs/agent-browser)（Vercel Labs，Apache-2.0），锁定版本，覆盖所有平台：macOS、Windows 和 Linux 桌面，以及无头服务器。它取代 Electron 浏览器界面。决策背后的对比（Playwright MCP、Chrome DevTools MCP、Steel、Stagehand、browser-use、browserless、Lightpanda）见文末总结。

## 原因

- **Windows 目前没有浏览器。** Electron 界面需要渲染器沙箱，而 Electron 43 在 Windows 上启用沙箱后会在就绪前退出（electron/electron#51761）。一个作为独立 Chrome 进程、通过 CDP 驱动的浏览器不依赖这一点。
- **服务器目前没有浏览器。** `openmausbot serve` 和 Docker 栈运行时没有 Electron，因此 `availableBrowserConnection()` 为 null，开关置灰。
- **一个引擎，一份契约。** 在 Mac 上编写的技能可在 VPS 上运行。
- **登录墙。** 所有已发布的 agent 产品（ChatGPT agent、Browserbase Live View、Cloudflare Browser Run）在登录或 CAPTCHA 时把浏览器交给人工，然后恢复。agent-browser 的流把投屏帧送出、把鼠标/键盘输入送入，因此我们只需一个 canvas 就能在桌面和服务器上实现观看与接管。

## agent-browser 提供的能力

- 基于 CDP 的 Rust 守护进程；Chrome for Testing 由 `agent-browser install` 下载并校验（会检测已有的 Chrome/Chromium/Brave）。
- 隔离、并行的**会话**（`--session <id>`），`--restore` 在稳定键下自动保存 cookies 和 localStorage；持久化配置文件；桌面端可复用用户自己的 Chrome 配置文件；认证状态导入/导出；加密认证保险库（`AGENT_BROWSER_ENCRYPTION_KEY`）。
- 一个 MCP stdio 服务器（`agent-browser mcp --tools core`），其动词与我们的 17 个 `browser_*` 工具几乎一一对应。
- 一个流服务器：JPEG 帧、当前 URL、标签页和视口元数据。上游**不**强制人工优先。该闸门由 OMB 掌控。

## 架构

```
bot turn → scoped OMB MCP proxy → profile control gate → agent-browser MCP → Chrome
owner UI ← authenticated SSE ← OMB ← loopback WebSocket frames ← agent-browser
owner UI → authenticated POST → control gate → acknowledged native input → Chrome
```

`server/index.ts` 中的 `browserIntegration()` 挂载一个回合作用域的 OMB 代理，由工作区标志、机器人开关和提供方的 `browserMcp` 能力共同守卫。原生会话身份和加密密钥保留在服务器中。

## 步骤

### 1. 无头引擎（服务器，以及所有平台的回退方案）——已交付

- `server/browser-engine.ts`：解析锁定的 agent-browser 二进制（`OMB_AGENT_BROWSER_PATH` → `$OMB_DATA_DIR/tools/agent-browser/<version>/` → PATH）；从 GitHub release 下载，各平台 SHA-256 锁定在 `server/browser-engine-release.ts`（与 `antigravity-release.ts` 和 `prepare-cloudflared.mjs` 相同模式）；用 `agent-browser install` 确保 Chrome 就绪；报告 `unavailable` 并附原因，而非静默降级。
- `browserIntegration()`：当不存在桌面连接且引擎可用时，返回 `{command: <binary>, args: ["mcp", "--tools", "core", "--no-webmcp"], env: {AGENT_BROWSER_SESSION, AGENT_BROWSER_RESTORE: <stable-key>, AGENT_BROWSER_ENCRYPTION_KEY, AGENT_BROWSER_HEADLESS: "1"}}`。会话 id = 机器人的浏览器配置文件分区，或机器人 id（独立会话）。
- 加密密钥：一次性生成到 `$OMB_DATA_DIR/browser-engine-key`（0600），与隧道凭据相同。
- 能力：环境描述符新增 `capabilities.browser: "desktop" | "headless" | "unavailable"`（+ 原因），设置开关和每个机器人的开关据此判断，而非 `window.ogb.browser`。
- Docker：构建时以 root 身份、在 `USER maus` 之前执行 `npm install -g agent-browser@<pinned>` 和 `agent-browser install --with-deps`。
- 测试：解析器和下载锁定（stub 服务器）、集成规格（对守卫做变异检查）、针对模拟 `agent-browser` 二进制（支持 MCP）的端到端回合。

### 2. 桌面，三个平台——已交付

- 桌面端使用相同的引擎和规格；Electron 界面（`electron/browser-surface.cjs`、`browser-host.cjs`、`browser-platform.cjs`、分区清理、连接/控制同步、Windows 闸门、preload `browser:*` IPC、快照包）以及服务器的每回合能力注册表均已删除。机器人和配置文件删除通过同一持久化清理日志在服务器本身清除引擎的会话状态。设置和机器人的浏览器面板提供一键安装引擎（`POST /api/browser-engine/install`）。
- 应用保持浏览器无头；实时面板是可见界面。导入操作员的 Chrome 配置文件仍不在范围内；配置文件可以改用该用户正在运行的 Chrome（步骤 5，My Chrome）。

### 3. 观看与接管——已实现，待发布

- 两条紧凑的 chrome 行：标签页和顶部配置文件按钮；导航、地址、接管和溢出菜单在下方。配置文件管理和输入/粘贴按需打开。页面周围没有常驻状态仪表盘。
- `BrowserLive` 仅代理帧/状态/标签页/URL 事件。ACK 跟随解码渲染，包括相同的图像数据。最多一个待处理帧等待 SSE 背压；缓慢/断开的查看者会被关闭。默认 15 fps。
- `BrowserRuntime` 为共享会话门控**所有**浏览器工具调用和转录捕获。接管先阻止新调用，再排空已接受的调用。人工输入使用固定的原生 HTTP 命令，等待 CDP 完成，而非即发即忘的 WebSocket 输入。交还等待这些命令完成。
- 超时的操作结果不确定。显式重启浏览器在恢复前关闭原生会话；保留已保存的登录。当共享该浏览器的机器人正在工作时，该操作被拒绝。
- 仅所有者/管理员的端点使用现有认证、CSRF 和会话撤销检查，桌面和自托管一致。浏览器帧永不进入客户端可读的全局事件流。原生流端口仅限回环。人工控制期间隐藏其他查看者。
- 自有、共享和临时配置文件在 Web 上同样可用。临时浏览器仅存活一次服务器运行，不保存，在删除/配置文件退出时关闭。配置文件编辑包含过期列表检查；删除仅移除该确切配置文件的已保存状态。不要为此使用上游的 `state clear --all`。
- 这些控制门控的是 OMB 的浏览器工具，而非任意主机 shell 访问。浏览器配置文件是登录隔离，**不是**操作系统安全沙箱。

参见[原生验证配方](../verification/browser-live.md)。录制的 `--fps 60` 与实时流分开，需要 ffmpeg；录制控制有意不加入实时浏览器 chrome。

### 4. 工具名适配器——已放弃

引擎的 `agent_browser_*` 名称现在就是各处的词汇；系统提示会教授它们。无需适配器。

### 5. 已登录站点和 My Chrome——已实现

- **已登录站点。** 每个内置浏览器（共享配置文件，或机器人自己的浏览器）列出其持有 cookies 或存储的站点，支持按站点退出登录和全部退出登录，位于设置 → 浏览器配置文件和机器人的浏览器面板。`server/browser-sites.ts` 用引擎自身的 `state save` 读取运行中的浏览器，从恢复文件读取已停止的浏览器（`<restore key>-<session>.json.enc`：AES-256-GCM，密钥为 `browser-engine-key` 文本的 SHA-256，12 字节 nonce 在前，16 字节 tag 在后）。退出登录通过回环 DevTools socket 删除该站点的 cookies 并清除其来源的存储，然后重写所有已保存副本，使下次启动两者都不恢复。站点是浏览器自身存储记录的最宽主机（`.google.com` 使 `google.com` 成为 `accounts.google.com` 的站点）；不使用公共后缀列表。密钥永不离开服务器。路由：`GET /api/browser-sites`、`POST /api/browser-sites/sign-out`（管理员，与配置文件相同）。
- **My Chrome。** 共享配置文件的浏览器为内置或 My Chrome。My Chrome 将 agent-browser 附加到用户自己正在运行的 Chrome（`AGENT_BROWSER_AUTO_CONNECT`：Chrome 的 `DevToolsActivePort`，然后是端口 9222/9229），使机器人在其真实、已登录的浏览器中操作。它永不将 OMB 的已保存状态恢复到该 Chrome，也不保存其 cookies，关机时不关闭它，且没有已登录站点列表：其登录是 Chrome 自己的。用户在 `chrome://inspect/#remote-debugging` 开启远程调试（Chrome 144+）；Chrome 在接受连接前会询问。切换配置文件的浏览器会关闭其已打开的浏览器（这会保存内置浏览器的登录）并等待其机器人的回合。
- **适用范围。** 仅在机器人运行于用户计算机上的地方：桌面应用。服务器或 Cloud 没有用户的 Chrome，因此该选项显示为不可用，那里的每个配置文件都保持内置。config.json 中的 `browserEngine.attachCdpUrl` 现在用于 My Chrome 无法自行找到时（另一端口或机器）；设置它也会使 My Chrome 在服务器上可用。它不再附加每个机器人。
- **为何用 My Chrome 而非有头内置浏览器。** 10 月 12 日用 agent-browser 0.37.0 和 Chrome 154 在 macOS 上测量，在 accounts.google.com 输入一个不存在的账户：无头内置浏览器和有头启动已安装 Google Chrome 的 agent-browser 都得到"无法让你登录。此浏览器或应用可能不安全"（两者的 `navigator.webdriver` 均为 true）。同一个人启动 Chrome 并开启远程调试、再由 agent-browser 附加，通过了检查并到达"找不到此账户"（`navigator.webdriver` 为 false）。LinkedIn 仍可能拒绝附加的浏览器（#894）；My Chrome 避免的是 Google 的拒绝，而非每个站点的机器人检测。

## 不做

- 捆绑 Steel、browser-use 或 Playwright MCP。高级用户今天即可将它们作为自定义 MCP 服务器添加。
- 反检测、代理、CAPTCHA 破解。

## 对比（2026 年 9 月 6 日）

| 候选 | 不选原因 |
|---|---|
| Playwright MCP（Microsoft） | 次选；两者都是 MCP 服务器，配置即可切换。需裁剪 70+ 工具，每个持久化配置文件一个客户端，无流。 |
| Chrome DevTools MCP（Google） | 侧重调试/性能，默认向 Google 发送使用统计，无每机器人会话，无流。 |
| Steel | 有可接管的实时视图，但自托管服务器一次运行一个会话且需要已安装的 Chrome；一个带 nginx 和 UI 的 Node/Fastify 服务。 |
| Stagehand、browser-use、Skyvern | Agent 或模型驱动的 SDK：在我们的 agent 之下再加一层模型循环。 |
| browserless | SSPL / 商业许可。 |
| Lightpanda | 不是 Chrome（Web 兼容性不完整），AGPL。 |