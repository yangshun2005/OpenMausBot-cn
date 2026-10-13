# 实时浏览器与配置文件

显式使用已安装的原生引擎和 Chrome 二进制文件；测试夹具绝不使用操作员的浏览器配置文件、OMB 主目录、提供商登录信息或 API 密钥。

```sh
OMB_VERIFY_BROWSER_BINARY=/absolute/path/to/agent-browser \
OMB_VERIFY_BROWSER_CHROME=/absolute/path/to/chrome-headless-shell \
node --experimental-strip-types scripts/verify-browser-live.ts
```

启动器使用 `launchVerificationServer`，配合临时主目录和伪造的模型 CLI，创建 Pepper，并打印后端、预览和本地测试页 URL。**只**打开打印出的预览 URL。Ctrl-C 会关闭其原生浏览器、UI 和测试框架，然后删除其临时数据；服务器日志保留。

1. 面板包含两行浏览器 chrome 和一个实时空白页。没有"即将推出"卡片。当机器人持有浏览器时，在你交互或按下 **Take control** 之前不会发送任何内容；悬停永远不会夺取控制权。
2. 输入打印出的测试页 URL 并按 Enter。首次交互会从机器人手中夺取浏览器（页面顶部出现一个胶囊提示，显示"You're using the browser"，在所有面板宽度下均如此），然后进行导航；它只发送一次。如果机器人正在执行操作，会先显示"Waiting for Pepper to finish…"；当机器人的操作先完成时，等待期间产生的点击和按键会被丢弃，不会在可能已变化的页面上重放（胶囊提示显示"Pepper was busy, so that wasn't sent. Try again."），而你输入的地址仍会打开。工具栏按钮会等待夺取完成，因此重复点击只执行一次。
   在名称字段中输入 `AdaX`，按 Backspace，然后按 Enter。流式页面必须显示 `Hello, Ada`。检查方向键和 Delete，Tab 进入备注字段并输入多行，然后打开对话框并用 Escape 关闭。这些必须影响远程页面，而不仅仅是周围的 UI。Shift+Escape 将焦点返回地址字段，而不向页面发送 Escape；仅使用键盘的用户仍必须能访问工具栏。在面板中 Tab 切换会经过实时页面而不夺取它；只有在页面中点击或输入才会将 Tab 发送到那里。
3. 停止 8 秒：控制权返回机器人，胶囊提示消失。Cmd+C 之类的 Cmd 快捷键不得延迟该过程。让 **Type or paste text…** 保持打开超过 8 秒：控制权仍归你，直到它关闭。按 **Take control**，然后等待一分钟：控制权仍归你，直到按 **Hand back**，它会立即归还。**Type a password or code…** 显示一个掩码字段；其文本到达选定的页面字段，且不出现在任何日志、响应或其他视图中。
   按住鼠标按钮或按键绝不能归还控制权。重新连接视图，然后在同一静态图像上保持连接至少 30 秒。它不得因等待 ACK 而停滞。页面保持完整，机器人继续持有它；点击进入页面并确认重新连接后编辑和 Enter 仍然有效。
4. 单个配置文件按钮打开切换器，如果你正在使用浏览器则立即归还，以便切换器解锁。创建一个共享配置文件，切换到它（一个干净的浏览器），然后切回 Own browser。之前的页面保留。重命名共享配置文件而不改变其身份。确认删除会清除其机器人引用，而不删除另一个配置文件的数据。
5. 在预览上打开第二个隔离的浏览器标签页。在一个中交互；另一个不得接收新的页面帧或接受输入。归还和断开连接绝不能释放另一个查看者的控制租约。
6. 测试窄（390 px）和桌面宽度、全屏、标签页、溢出输入以及显式浏览器重启。不应出现水平文档溢出或永久设置面板。控制期间禁用配置文件更改。完整的浏览器图像必须适配面板，而不改变远程页面的分辨率。在每个尺寸下点击相同的测试页控件：输入必须跟随所包含的图像，而不是其周围的信箱区域。空白边距不得点击页面；在图像上开始的拖拽可以在那里结束。
7. 在重启仍在完成时重新连接，或在较早的命令完成前更改配置文件。迟到的响应不得断开新视图、改变其错误状态，或将旧帧确认为新查看者。
8. 登录请求：编写 Pepper 的回合以调用测试页的 `request_sign_in`（下面的 e2e 使用伪造引擎执行此操作）。会到达一张 **Sign in needed** 卡片和一条通知。**Open** 选择已在你控制下的 Browser 标签页；页面仍归机器人所有，或者当机器人将其留空时打开该站点，其上方的横幅提供 Done 和 Cancel。请求打开期间，空闲时间永远不会归还控制权。卡片上的 **Done**（或从第二个窗口）会归还浏览器并以"the person finished signing in"恢复 Pepper；**Cancel** 以"declined"恢复它。
9. 已保存的登录：在同一张卡片上选择 **Save a login**，为测试页的主机保存一个测试账户（伪造值）。Pepper 恢复并调用 `fill_login`；页面的用户名和密码框被填充，工具结果只列出所填充的内容，**Settings → Logins** 列出该登录而不含其密码。在另一个主机（子域，或非本地地址上的纯 http）下打开同一页面并再次调用 `fill_login`：它会拒绝且不输入任何内容。让 Pepper 拍摄快照并对密码框的值执行 `eval`：两个答案都显示 `••••••`。
10. 1Password（仅使用测试账户和测试保险库，绝不使用真实的）：连接一个仅限于一个保险库的服务账户，该保险库为测试页的主机持有一个测试 Login 项。其行以只读方式出现，标记为 From 1Password；`fill_login` 像已保存的登录一样填充它。单元测试使用伪造 SDK，不访问网络。

聚焦的自动化覆盖：

```sh
pnpm exec vitest run server/browser-engine.test.ts server/browser-runtime.test.ts \
  server/harness-mcp-proxy.test.ts server/browser-live.test.ts \
  server/browser-live-routes.test.ts server/browser-codex-path.integration.test.ts \
  src/lib/browser-input-queue.test.ts src/lib/browser-control.test.ts src/lib/browser-profiles.test.ts \
  src/components/BrowserProfilesManager.test.ts src/components/BrowserViewport.test.ts \
  src/components/BrowserPanel.test.ts src/components/SignInRequestCard.test.ts \
  server/sign-in-request.e2e.test.ts server/login-fill.test.ts server/login-store.test.ts \
  server/totp.test.ts server/routes/logins.test.ts src/components/LoginsSettings.test.ts \
  server/saved-login.e2e.test.ts server/onepassword.test.ts scripts/prepare-duckdb.test.mjs
```

这些测试覆盖原生传输生命周期、所有者/会话作用域、门控竞态、过期的列表/流事件、原生按键路由、有界输入/背压、已撤销的能力以及精确的已保存状态清理。原生工作流测试仍然必需：仅凭一个通过的模拟帧测试并不能证明浏览器输入或恢复有效。

## 一分钟空闲与重连回归

使用已安装的 Playwright 模块，同一个隔离启动器可以运行自动原生验收流程，关闭其浏览器并退出：

```sh
OMB_VERIFY_BROWSER_BINARY=/absolute/path/to/agent-browser \
OMB_VERIFY_BROWSER_CHROME=/absolute/path/to/chrome-headless-shell \
OMB_VERIFY_PLAYWRIGHT=/absolute/path/to/playwright/index.mjs \
node --experimental-strip-types scripts/verify-browser-live.ts --recovery
```

这会让一个原生文档和未保存的表单使用生产运行时超时空闲 65 秒，验证文档身份未改变，然后通过下一次 agent 工具调用提交表单。同时，它将真实的 Browser 面板挂载到隔离服务器上，中断其实时 EventSource，并验证自动重连、保留的导航、无重放命令以及仅监视控制。中断是注入的；Chrome 和替换的服务器流是真实的。检查窄/桌面溢出，并在夹具日志旁保留一张预览截图。

运行时回归还模拟了一个必须比其 MCP 父进程存活更久的浏览器子进程，包括忽略 EOF 的 MCP 父进程。在 Windows 上也运行这些测试：仅 macOS 原生验收不能证明 Windows 进程行为。

### 生命周期参考

- [agent-browser 的 MCP 循环](https://github.com/vercel-labs/agent-browser/blob/v0.37.0/cli/src/mcp.rs)
  在 stdin EOF 时退出；空闲传输退役不得杀死配置文件的守护进程。
- [agent-browser 守护进程启动](https://github.com/vercel-labs/agent-browser/blob/v0.37.0/cli/src/connection.rs)
  分离守护进程。丢失 MCP 传输并不能证明已接受的浏览器操作已停止；中断仍需要显式重启。
- [Pi 作者的浏览器工具](https://github.com/badlogic/agent-tools/blob/main/browser-tools/browser-nav.js)
  在导航后断开自动化客户端，而不是关闭 Chrome。
- [OpenClaw 的 CDP 连接](https://github.com/openclaw/openclaw/blob/main/extensions/browser/src/browser/cdp-websocket.ts)
  在命令产生副作用之前限制连接重试。OMB 同样只重试观察，绝不重试点击、文本、导航或提交。

未复制任何上游实现；OMB 已随附 agent-browser，并保留其现有许可证声明和配置文件/控制边界。

于 2026-09-19 在 macOS arm64 上使用 agent-browser 0.37.0 和打包的 Chrome Headless Shell 验证：214 项聚焦检查、类型检查和限定范围的 lint 均通过。两次隔离的原生验收运行在 65 秒空闲时间内保留了文档身份和未保存的表单，然后完成了下一次工具操作。真实预览恢复了其注入的流故障，而没有重放操作或恢复人工租约。Windows 原生验收仍未完成。最终夹具证据：启动器打印的 `openmausbot-verification-evidence` 目录中的 `server-1789811974163-38144.log` 和 `server-1789811974163-38144-browser.png`。

## 真实 Codex 聊天到操作验收（可选加入）

这会使用真实模型配额。显式提供已安装的 CLI、登录文件和模型；该配方只将该登录复制到一次性主目录中。它不会导入个人聊天、提供商设置、技能或浏览器配置文件。切勿将其请求指向用户正在运行的应用。

```sh
OMB_VERIFY_CODEX_CLI=/absolute/path/to/codex \
OMB_VERIFY_CODEX_AUTH=/absolute/path/to/auth.json \
OMB_VERIFY_CODEX_MODEL=your-supported-model \
node --experimental-strip-types scripts/verify-codex-surface-live.ts
```

打开打印出的预览，选择 Ziggy，并发送不带工具名称的普通请求：打开打印出的测试页，填写名称，点击 Say hello 并验证问候语。确认实际截图，而不仅仅是模型的回答。使用编辑器选择 **Approve for me**，然后让它更改名称并打开/读取对话框。常规工具操作不应产生重复的批准卡片；提供商审查仍可能询问其他操作。

在未配置的云计算机上请求 Chrome。它必须检查真实选项并报告阻碍，而不是在主机上操作并声称它是 VM。编辑器和面板在发送期间必须保持真实目标高亮。

Ctrl-C 关闭确切的夹具并删除复制的登录和浏览器数据。VM/云传输和回合绑定切换由 `server/group-local-vm.e2e.test.ts`、`server/vps-routing.test.ts` 和 `server/index.test.ts` 使用隔离的提供商单独覆盖。这些不是真实云供应的证据。