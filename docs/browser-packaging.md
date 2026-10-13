# 浏览器就绪的桌面安装包

桌面安装程序包含固定版本的 `agent-browser` 引擎和 Chromium Headless Shell。全新安装无需单独下载浏览器。这是自动化浏览器，不是有界面的 Google Chrome 应用。浏览器访问默认开启，工作区（设置）和每个机器人（其计算机面板）均如此；任一处关闭即禁用浏览器，打包可执行文件不会覆盖该开关。

## 包布局与查找

`pnpm package:prepare` 运行 `pnpm build:browser`。它下载经过验证的供应商归档，并暂存到 `dist-native/browser/PLATFORM-ARCH`，在 macOS 上复制为 `Resources/browser-engine`，其他平台为 `resources/browser-engine`。每个包只包含其目标架构：macOS ARM64/x64、Windows x64 或 Linux x64。

服务器从 Electron 接收 `OMB_RESOURCES_PATH`。解析顺序：先看显式的 `OMB_AGENT_BROWSER_PATH` 覆盖，然后是完整的捆绑引擎和浏览器，最后是单独安装的引擎或 PATH。捆绑不完整时直接失败，并提示重装/更新。显式的 `AGENT_BROWSER_EXECUTABLE_PATH` 仍会覆盖浏览器可执行文件。配置文件、Cookie 和下载内容属于用户数据，绝不打包为资源。

npm/自托管发行版保持精简，保留其显式的浏览器安装流程。桌面打包不会悄悄增大 npm tarball。

## 已登录站点与 My Chrome

每个内置浏览器都会列出其已登录站点（设置 → 浏览器配置文件，或机器人的浏览器面板），支持按站点退出登录和全部退出登录。桌面、服务器和 Cloud 上行为一致，无论浏览器打开还是关闭。共享配置文件可改用 **My Chrome**，即用户自己正在运行的 Chrome，用于拒绝自动化浏览器的站点，例如 Google 登录。这需要桌面应用和 Chrome 144+，并在 `chrome://inspect/#remote-debugging` 开启远程调试；在 Cloud 和服务器上配置文件仍为内置。详情见[浏览器引擎计划，第 5 步](plans/browser-engine.md)。

## 视频录制

macOS 和 Linux 引擎固定为 agent-browser 0.37.0。其 [`record start` 和 `record restart`](https://agent-browser.dev/recording) 命令默认以 30 fps 录制当前页面；`--fps 60` 选择 60 fps（有效范围：1–60）。这会保存视频文件，不会启用实时视图。录制需要服务器 PATH 中有 `ffmpeg`，WebM 需要 `libvpx`，MP4 需要 `libx264`。`agent-browser doctor` 会报告这些可选依赖。它们不捆绑，正常浏览也不需要。更高帧率消耗更多 CPU 和磁盘空间；实际不同帧数取决于页面重绘。OMB 的浏览器面板中没有录制控制。

## Linux 沙箱

在 Ubuntu 24.04 上使用 `.deb`。其包钩子为 `/opt/OpenMausBot` 下 root 所有的浏览器可执行文件安装范围狭窄的 AppArmor 策略。它们不会禁用浏览器沙箱，也不会更改全局用户命名空间限制。参见 [Linux 桌面](linux-desktop.md)。

AppImage 包含相同的二进制文件，但无法安装特权沙箱策略。受限主机可能需要 `.deb` 而非 AppImage。

## 发布验证与更新

版本、URL、大小和 SHA-256 摘要位于 `server/browser-bundle-release.ts`；引擎固定版本与 `server/browser-engine-release.ts` 共享。每次缓存的下载都会重新校验。准备阶段会清点整个供应商目录树，包括许可证。`afterPack` 门禁在签名前拒绝缺失资源、字节变更或可执行文件架构错误。macOS 签名会改变原生字节，因此签名后的包后续使用代码签名和运行时测试检查，而非原始上游可执行文件哈希。

对解包后的应用运行真实浏览器检查：

```sh
node scripts/smoke-browser-bundle.mjs --resources /absolute/app/resources
```

macOS 上使用 `/absolute/OpenMausBot.app/Contents/Resources`。Linux 上以非特权用户对已安装的 `.deb` 运行，路径为 `/opt/OpenMausBot/resources`。此检查会创建自己的空 home 和本地网页，检查自动发现、导航、输入/点击、截图交付以及双机器人 Cookie/存储隔离，然后只删除其测试夹具。不使用模型账户或用户浏览器配置文件。跨目标 `--check-only` 只检查布局，不能作为浏览器成功执行的证据。

发布工作流在发布前于原生 macOS、Windows 和 Linux runner 上运行此检查。macOS 还会验证浏览器二进制文件使用应用的团队身份签名。浏览器通过经过审查的应用发布更新：审查新的供应商目标/许可证内容，更新所有平台固定版本，运行准备/测试和原生包门禁，并在需要浏览器安全更新时尽快发布。桌面捆绑包中没有独立的自动浏览器更新器。

### 临时 Windows 引擎回移

Windows 使用明确标识的 OpenMausBot 构建版 agent-browser 0.36.0，包含来自[上游 PR #1781](https://github.com/vercel-labs/agent-browser/pull/1781) 的句柄继承修复。官方 0.37.0 发布版不包含此修复，因此 Windows 停留在原生验证的 `0.36.0-omb.1` 修订版，尚未获得新的录制选项。不要用未打补丁的 0.37.0 Windows 二进制文件替换它。原始二进制文件在新启动的后台浏览器保持工具调用输出连接打开时可能挂起。回移更改了该 Windows 启动行为；它不包含 PR 更广泛的输出读取器重写或上游 main 的无关更改。

仅产物的 **Windows 浏览器供应商构建**工作流构建固定源码和已检入的补丁，保留许可证和构建来源，并在 Windows 上通过同一 MCP 连接测试冷启动及关闭/重开。其候选版本必须经过审查，并在正常应用打包工作流使用前固定其确切大小和 SHA-256。候选测试不能替代打包应用测试。已发布的供应商字节不得覆盖；变更的构建需要新修订版和经过审查的固定版本。

Windows 修订版有单独的管理安装目录，以免旧的 0.36.0 下载被误认为已打补丁的引擎。桌面包使用其捆绑引擎。显式可执行文件覆盖仍由用户管理。当官方发布版包含该修复并通过相同的原生冷启动和重启测试后，移除回移。

完整的上游声明和来源见 [`third_party/browser`](../third_party/browser/README.md)。完整的 Google Chrome 发行版及其专有 Widevine 组件未捆绑。