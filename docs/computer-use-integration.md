# OpenMausBot 中的计算机使用与浏览器使用

决策文档，2026-08-12。OpenMausBot 中的机器人如何获得本地计算机使用和浏览器使用能力。macOS 和打包的 Ubuntu x64 构建使用开箱即用、随发布版本锁定的提供程序；源码/开发版 Ubuntu 可使用单独安装的提供程序。基于对 OSS 聊天应用 MCP 宿主、macOS 控制服务器、浏览器自动化技术栈，以及本机 `cua` / `axstream` 代码的调研。

## TL;DR 架构

```text
Electron 主进程
├── CUA 宿主  ──启动──▶  cua-driver（macOS 和打包的 Ubuntu x64 上随包提供）
│     平台权限边界                               │ unix socket（私有）
├── WebContentsView 池（内嵌浏览器，按机器人分区持久化：persist:）
│     通过 webContents.debugger（CDP）驱动 —— 零安装的浏览器使用
└── server/ harness（驱动以 --mcp-config 启动 agent CLI）
      ├── computer-proxy-local.ts  ──▶ 将 MCP 工具调用转发到 driver socket
      └── computer-proxy.ts（现有） ──▶ 远程/云端 Boat 计算机
```

- **插件 = 基于 stdio 的 MCP 服务器。** 插件面板控制哪些 MCP 服务器被注入到每个机器人的 `--mcp-config`。与 Claude Desktop / Cherry Studio / LibreChat 的模式相同。
- **本地桌面使用 = `cua-driver`**。macOS 将 Rust Mach-O 打包进应用 Resources；Ubuntu x64 将认证过的 0.19.3 ELF 及其光标主题 sidecar 打包在 ASAR 之外。两者均与应用发布版本配对。这适用于 Ubuntu 24.04 GNOME/Xorg 测试版和受保护的 GNOME/Wayland 测试版；远程/云端 Boat 计算机和隔离的 Local VM 仍是独立的提供程序。**不是** Swift —— 大家记得的那个 Swift 文件（`examples/embedded-host-macos/ExampleAgentHarness.swift`）是一个 165 行的参考宿主，演示嵌入模式，而非驱动本身。
- **浏览器使用 = 优先使用应用自带的 Chromium。** Electron *就是* Chromium；将页面嵌入 `WebContentsView`，通过内置的 `webContents.debugger` CDP 传输驱动。无 Chrome 依赖，无需下载 281MB 的 Playwright，用户可在聊天中观看机器人浏览。

## 本地桌面使用：仅用 CUA —— Electron 掌控驱动生命周期

**决策（Milind，2026-08-12）：CUA 是唯一的本地桌面控制提供程序。不用 cliclick，不用 robotjs/nut.js，不用 Python computer-server，无回退方案。**
所有本地桌面控制和输入操作都经过已验证的 `cua-driver` 二进制。Linux 屏幕预览使用受支持的 Xorg 或用户主动触发的 XDG portal 捕获路径，它不是控制提供程序。此规则不替代远程/云端 Boat 或隔离的 Local VM 提供程序。已评估并否决的本地替代方案：

Ubuntu GNOME 测试版使用相同的官方 CUA 提供程序，并遵循 [#113](https://github.com/milind-soni/OpenMausBot/issues/113) 中跟踪的 Phase 5 供应链契约：锁定归档和内层哈希、精确的归档允许列表、ASAR 之外的资源、完整的 notices/SBOM、无运行时下载/更新，以及失败即关闭的打包发现机制。Electron 仍拥有私有的内嵌守护进程/socket，harness 只接收已验证的 MCP 代理契约。Xorg 在 [#79](https://github.com/milind-soni/OpenMausBot/issues/79) 中跟踪；GNOME/Wayland 额外需要 WinRects v8 以及 [#109](https://github.com/milind-soni/OpenMausBot/issues/109) 中跟踪的精确 Cua 健康报告契约。

| 方案 | 结论 |
| --- | --- |
| cua `computer-server`（Python/FastAPI） | ✗ 200MB+ 的冻结 Python，在错误身份下触发第二次 TCC 提示 |
| axstream / cliclick / robotjs 类 | ✗ 已否决 —— 仅用 CUA 政策 |
| **cua-driver 二进制，嵌入模式** | ✓ 唯一提供程序：一份契约，20+ 工具，自带 stdio MCP 代理 + socket 守护进程 + TS SDK（`@trycua/cua-driver`），agent 光标叠加层，权限工具 |

### 规则（来自 `cua/libs/cua-driver/rust/Skills/cua-driver/EMBEDDING.md` —— 请从头到尾阅读）

1. **从 Electron 主进程启动，绝不从 server/gateway 层启动。** macOS TCC 将派生的子进程归属到其"责任进程"。从 Electron 主进程启动 → 授权属于 OpenMausBot，用户只看到一次名为 OpenMausBot 的提示，随包提供的驱动继承该授权。从 Node gateway/daemon 启动 → 身份会静默变为 gateway 的，且 `check_permissions` 无法检测到这种错误归属。harness 必须通过 IPC 向 Electron 主进程请求 driver socket 路径，而不是自己启动驱动。
2. 使用 `@trycua/cua-driver` 中的 `EmbeddedCuaDriverHost`（`libs/cua-driver/typescript/src/embedded.ts`，Electron 辅助函数在 `src/electron.ts`：`requestMacOSPermissions`、`hasRequiredMacOSPermissions`、`openMacOSScreenRecordingSettings`）。可用的参考：`typescript/test/electron-main-fixture.mjs`。
3. 环境变量：`CUA_DRIVER_EMBEDDED=1`（精确值）+ `CUA_DRIVER_HOST_BUNDLE_ID`。权限模式 `standard`。
4. 生命周期：将 `before-quit` 推迟到 `await embedded.stop()` 之后；TCC 授权变更后，销毁客户端 → `restart()` → 重新连接（macOS 按进程缓存 TCC）。

### macOS 打包目标

- 将二进制放在 `OpenMausBot.app/Contents/Resources/cua-driver`，**在 ASAR 之外**，保留可执行位（electron-builder `extraResources`）。
- **在签名 + 公证应用之前，用我们的 Team ID 重新签名**（安装副本由 trycua `YCK386LBJ7` 签名）。这是最大的新增构建步骤。
- Info.plist：`NSAccessibilityUsageDescription`、`NSScreenCaptureUsageDescription`（与 `/Applications/CuaDriver.app` 的字符串保持一致）。
- 引导流程：检查 → 应用内说明 → 深链到设置面板（`x-apple.systempreferences:com.apple.preference.security?Privacy_Accessibility` 和 `?Privacy_ScreenCapture`）。预计 macOS 15 会约每月重新提示屏幕录制权限；`persistent-content-capture` 权限由 Apple 把关，我们实际上无法获得。

### MCP 暴露：官方 `cua-driver mcp` 代理（无自定义代理）

不要手写 socket 代理。驱动自带 stdio MCP 代理：

```
cua-driver mcp                              # 独立模式（连接到运行中的守护进程）
cua-driver mcp --embedded --socket <path>   # 嵌入模式（宿主拥有的守护进程）
```

它在 stdin/stdout 上使用行分隔的 JSON-RPC 2.0，自身不执行任何操作，转发到宿主拥有的守护进程。已针对安装的 `CuaDriver.app` 二进制验证往返：`tools/call get_screen_size` → `{"width":1512,"height":982,"scale_factor":1}`。

因此 harness 只需向机器人的 `--mcp-config` 添加一个条目：

```jsonc
{ "mcpServers": { "computer": {
    "command": "<cua-driver binary>",
    "args": ["mcp", "--embedded", "--socket", "<socketPath>"],
    "env": { "CUA_DRIVER_EMBEDDED": "1", "CUA_DRIVER_HOST_BUNDLE_ID": "com.opengrokbot.app" }
} } }
```

Electron 主进程将该描述符写入 `<userData>/cua-connection.json`（见 `electron/cua.mjs`）；harness 读取它并注入该块。驱动自身的非幂等操作安全机制，以及 `ax → ax_fg → cgevent → cgevent_fg → cgevent_hid` 投递阶梯（优先使用后台 pid 寻址输入 —— 不会抢占用户光标）都在二进制内部处理；宿主无需额外添加。

驱动工具面（依据 `cua-driver list-tools`）：start_session、click、double_click、right_click、drag、scroll、type_text、press_key、hotkey、move_cursor、get_window_state、get_desktop_state、get_accessibility_tree、list_windows、list_apps、launch_app、bring_to_front、check_permissions、get_screen_size、zoom、screenshot。优先使用 AX 元素路径而非像素坐标，且在后台/隐藏窗口上也能工作。

### 政策：CUA 是唯一的计算机使用路径

不用 axstream，不用 cliclick，不用 robotjs/nut.js，不用 Python computer-server。如果缺少某项能力（例如 OCR 锚定点击、宏录制/回放），则将其添加到上游 cua-driver 或作为驱动工具请求 —— 绝不在其旁边拼凑。这样保持一个 TCC 身份、一个需要签名/公证的二进制，以及一份行为契约。

## 浏览器使用：三个层级

1. **默认，零配置：内嵌浏览器。** 聊天 UI 内的 `WebContentsView`，`persist:bot-<id>` 会话分区（登录状态在重启后保留，按机器人隔离），规范化 Chrome UA。通过 `webContents.debugger` 驱动（内置 CDP：`Input.*`、`Runtime.*`、`Page.*`、`Accessibility.getFullAXTree` 用于 playwright-mcp 风格的快照引用）+ `capturePage()` 用于视觉。用户可在任务中途接管鼠标完成登录 / CAPTCHA，然后交还。已知限制：Google OAuth 会阻止内嵌 webview —— 将 Google 账号流程路由到第 2/3 层。
2. **可选启用"使用我的真实 Chrome"：扩展桥接。** Chrome 136+ 在默认配置文件上禁用了 `--remote-debugging-port`（不要构建旧的 CDP 重启流程）。现存的路径是 playwright-mcp `--extension` 模式（或 Browser MCP）+ Web Store 的 "Playwright Extension" —— 通过 `chrome.debugger` 驱动用户已登录的标签页。需要安装扩展，因此仅限可选启用。
3. **可选启用的高级层：随包的 `@playwright/mcp`** 启动系统 Chrome（`--browser chrome`，其默认值 —— 存在 Chrome 时无需下载），持久化配置文件目录使登录状态保留。可选 chrome-devtools-mcp 用于性能/Lighthouse/网络任务。

跳过 browser-use（Python；想掌控 agent 循环；连他们自己的桌面应用都不嵌入它）。

## 推出顺序

1. `computer-proxy-local.ts` + 在开发环境中直接从 Electron 主进程启动 `cua-driver mcp`（未签名的开发构建继承终端/Electron 的授权）。
2. 权限引导 UI（插件面板 → "Computer" 插件卡片：状态、授权按钮、深链）。
3. 内嵌浏览器面板 + 最小 CDP 工具集（navigate / snapshot / click-ref / type / screenshot），作为 "Browser" 插件暴露。
4. 打包：extraResources + 重新签名 + 公证；为生产环境接入 `EmbeddedCuaDriverHost`。
5. 后续：axstream 风格的宏教学/回放、扩展桥接、playwright-mcp 层。