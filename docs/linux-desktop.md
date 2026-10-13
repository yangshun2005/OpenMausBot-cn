# Ubuntu Desktop

OpenMausBot 提供 Ubuntu 24.04 LTS x86_64 桌面测试版。Electron 包内嵌了 harness 服务器，因此安装后的构建版本在运行时不需要 Node、pnpm、Swift 或终端。若要在自己的服务器而非本机上为 bot 提供同类型的 Linux 桌面，请参阅 [byo-vps.md](byo-vps.md)。

## 可用功能

- 在 GNOME Xorg 和 GNOME Wayland 上的原生 Electron 窗口及内嵌 OpenMausBot 服务器。
- 本地 Claude、Codex、Grok、Gemini 及其他已配置的 agent CLI。
- 聊天、流式对话轮次、审批、bot 间通信及本地数据存储。
- Composio 已连接应用和 Boat 云计算机。
- 在默认浏览器中打开外部文档和 OAuth 链接。
- 在 GNOME Xorg 和 GNOME Wayland 上提供显式的、仅查看的本地屏幕预览。Wayland 路径使用原生 portal 选择器，并保持所选 PipeWire 流打开，直到用户停止共享。
- 在 GNOME Xorg 上通过显式选择加入的本地计算机控制，使用捆绑的固定版本 Cua Driver，且不启用其装饰性全屏光标叠加层。
- 在 GNOME Wayland 上保持故障关闭的本地控制状态，直至 issue #345 中独立的真实席位输入安全门控得到解决。

本地预览本身**不会**让 bot 控制本计算机。在 Xorg 上，本地控制需要同时满足全局 **Enable local control** 选项和将 bot 分配给 **This computer**；每个操作仍会进入审批流程。在 Wayland 上，本地控制被禁用，旧的加入选项会自动清除。Wayland 辅助程序自动安装、Linux 听写和 ARM64 仍不可用并保持故障关闭；请在 [issue #29](https://github.com/milind-soni/OpenMausBot/issues/29) 中跟踪进展，在 [issue #345](https://github.com/milind-soni/OpenMausBot/issues/345) 中跟踪安全暂停。捆绑 CUA 供应链工作跟踪于 [issue #113](https://github.com/milind-soni/OpenMausBot/issues/113)。Xorg 跟踪于 [issue #79](https://github.com/milind-soni/OpenMausBot/issues/79)，受保护的 GNOME/Wayland 支持跟踪于 [issue #109](https://github.com/milind-soni/OpenMausBot/issues/109)。

## 下载包

从最新发布版本中选择一个 Ubuntu 24.04 x86_64 包：

- [Debian 包（`OpenMausBot-amd64.deb`）](https://github.com/milind-soni/OpenMausBot/releases/latest/download/OpenMausBot-amd64.deb) — 推荐；APT 会安装其桌面依赖并配置捆绑 bot 浏览器的沙箱。
- [便携 AppImage（`OpenMausBot.AppImage`）](https://github.com/milind-soni/OpenMausBot/releases/latest/download/OpenMausBot.AppImage) — 不安装系统文件。
- [SHA-256 校验和](https://github.com/milind-soni/OpenMausBot/releases/latest/download/SHA256SUMS-ubuntu-x64.txt)

带版本号的包和之前的发布版本仍可在[发布页面](https://github.com/milind-soni/OpenMausBot/releases)获取。

## 构建包

从源码构建的要求：

- Ubuntu 24.04 LTS x86_64
- Node.js 24 或更新版本
- pnpm 10.33.0（Corepack 可安装项目声明的版本）

```sh
git clone https://github.com/milind-soni/OpenMausBot.git
cd OpenMausBot
corepack enable
pnpm install --frozen-lockfile
pnpm package:linux
```

构建会生成：

- `release/OpenMausBot-<version>-amd64.deb`
- `release/OpenMausBot-<version>-x86_64.AppImage`

AppImage 使用静态运行时，不需要旧版 `libfuse2` 包。

## 安装与运行

使用 APT 安装下载的 Debian 包，以便解析其桌面依赖：

```sh
sudo apt install ./OpenMausBot-amd64.deb
```

然后从 GNOME 应用启动器打开 **OpenMausBot**。卸载：

```sh
sudo apt remove openmausbot
```

便携 AppImage 不安装系统文件：

```sh
chmod +x release/OpenMausBot-*-x86_64.AppImage
./release/OpenMausBot-*-x86_64.AppImage
```

对于下载的发布版 AppImage，请用 `OpenMausBot.AppImage` 替换上面的带版本号路径。

### Ubuntu 24.04 上的捆绑 bot 浏览器

桌面包包含浏览器引擎和浏览器运行时；无需单独下载浏览器。在 Ubuntu 24.04 上使用 **`.deb` 包**进行浏览器就绪安装。其安装程序会保护浏览器由包拥有的文件，并为该确切可执行文件加载 AppArmor 规则。浏览器沙箱保持启用，全机用户命名空间限制不变。升级会刷新同一规则；卸载包会移除它。启用 bot 浏览器并批准其操作仍是显式选择。

AppImage 是便携的，无法安装特权系统策略。在限制非特权用户命名空间的主机上，其捆绑浏览器仍可能报告 `No usable sandbox`。请改用 `.deb` 安装；不要添加 `--no-sandbox`、全局禁用 AppArmor，或将主目录下的任意可执行文件加入允许列表。参见 [Chromium 对 Ubuntu 限制的说明](https://chromium.googlesource.com/chromium/src/+/main/docs/security/apparmor-userns-restrictions.md)。

应用数据保留在本地 `~/.openmausbot`。Electron 浏览器数据和窗口状态使用常规 XDG 配置目录（除非环境覆盖，否则为 `~/.config/openmausbot`）。

## 开发桌面 shell

开发模式使用三个进程。请在每个终端中保持各命令运行：

```sh
pnpm dev:server
pnpm dev
pnpm dev:desktop
```

若要进行包形态的构建而不生成 `.deb` 或 AppImage 构建产物：

```sh
pnpm package:linux:dir
./release/linux-unpacked/openmausbot
```

## Agent CLI 发现

从 GNOME 启动的应用不会继承与终端相同的交互式 shell `PATH`。OpenMausBot 保留继承的路径，并添加现有的常见位置，例如：

- `~/.local/bin`
- `~/.claude/local`
- `~/.volta/bin`
- `~/.bun/bin`
- `~/.asdf/shims`
- `~/.deno/bin`
- `~/.nvm/versions/node/*/bin`
- `/usr/local/bin`

它还会在后台探测登录 shell。如果仍未检测到 CLI，请在从终端启动应用前设置显式的附加路径，并在终端中验证：

```sh
OMB_EXTRA_PATH=/your/custom/bin ./release/OpenMausBot-*-x86_64.AppImage
```

安装或登录 CLI 后请重启 OpenMausBot。

## Xorg 与 Wayland

shell、聊天、云计算机、已连接应用和仅预览捕获在两种 GNOME 会话类型中均可工作。Wayland 的选择器/选择/持久流/取消/结束/重试生命周期已在真实的 Ubuntu 24.04 GNOME Wayland 会话中验证。当 `WAYLAND_DISPLAY` 和 `DISPLAY` 同时存在时，OpenMausBot 会在 XWayland 之前检测到 Wayland，因此捕获不会意外绕过 portal 中介行为。

打开 Computer 面板并使用单独的 **Preview this computer** 卡片。应用或面板打开时捕获绝不会启动。

- **Xorg：** **Start preview** 直接捕获主显示器。
- **Wayland：** **Choose a screen** 打开一次 GNOME portal 选择器。所选流保持打开，直到你按下 **Stop preview**、关闭面板、从 GNOME 结束共享或退出应用。

取消或结束 Wayland 共享会返回平静的 **Try again** 状态，且绝不会自动重新打开选择器。OpenMausBot 不捕获屏幕音频、重启后不记住所选显示器，也不在 Linux 上提供 **Open Settings** 操作。

本地计算机控制独立于预览。它在 Xorg 上经显式选择加入后可用，在 Wayland 上保持故障关闭。XWayland 的 `DISPLAY` 绝不会绕过 Wayland 安全门控。

## 启用本地控制

安装的 `.deb` 和 AppImage 构建包含经过认证的 **Cua Driver 0.19.3** CLI 和光标主题 sidecar。在 GNOME Xorg 上，打开 Settings，选择 **Enable local control (Beta)**，等待 **Ready**，然后显式将 bot 分配给 **This computer**。无需下载驱动、终端命令、`chmod` 或守护进程设置。自有守护进程以 `--no-overlay` 启动，因此绝不会创建 Cua 的装饰性全屏 X11 光标表面。OpenMausBot 还在 Linux 上使用 Electron 软件渲染，以避免已复现的 NVIDIA/libGLES GPU 进程故障——该故障可能导致不可见的聚焦应用窗口接收输入。

Cua 操作使用私有逻辑光标。禁用装饰性叠加层后，`move_cursor` 不会移动用户的物理指针；已批准的点击和输入操作仍会定位到请求的窗口，而用户自己的鼠标仍由其控制。

在 GNOME Wayland 上，**This computer** 仍不可用，且旧的持久化加入选项会以私有文件权限重置为关闭。请注销并从登录屏幕会话菜单中选择 **Ubuntu on Xorg**，或继续使用 Chat、仅预览捕获、Cloud 或 Local VM。Wayland 重新启用需要其自身的真实席位证据，且不会由环境覆盖控制。

上游发布版本没有签名或 GitHub 构建产物证明，且非不可变，因此构建使用显式审查过的摘要作为信任锚：

- 源码提交：`a1672e7b11951275ecfba3384264d4530185d0db`；
- 归档 SHA-256：`3db9d4257d84bacaf7eb104d225f85613ce67edbb20d6eeb83c1384b6d8a5b10`；
- 打包驱动 SHA-256：`ed5844fadf07b9b72c4a3b3802e1c47233c166d66d6198608d5991f807aab4ac`；
- 打包光标主题 SHA-256：`e589b2b7521bbfeaf9e2bfce668a38e80ed1b9790b1327b13d374fc331d8312a`。

打包过程在仅提取这两个可执行文件之前，会验证确切的归档大小、校验和、成员名称/类型/大小及内部哈希。应用不进行运行时驱动下载或自更新。Cua 的 MIT 许可证、内嵌 Inter 字体的 SIL OFL 1.1 声明、完整依赖许可证文本、MPL 源码位置及 CycloneDX 清单随二进制一同发布；审查过的源码记录位于 [`third_party/cua-driver`](../third_party/cua-driver/)。审查过的原生运行时会为压缩的 Ubuntu 构建产物增加约 11–13 MiB。ELF 需要 glibc 2.30 或更新版本，以及受支持的 Ubuntu 24.04 桌面上已有的标准 Ubuntu X11/XInput/xkbcommon 库；包验证器会从每种构建产物布局中执行确切二进制。

AppImage 固定的 SquashFS 工具链可能将 root 拥有的目录输出为 `0755` 或 `0775`；包验证器要求在审查过的资源树中一致地使用其中一种模式。执行前，AppImage 仅将固定的二进制复制到私有的 `0700` 暂存区，并再次验证其哈希。DEB 升级会自动将其确切的包拥有路径修复为 `root:root 0755`。

打包的运行时保留在 ASAR 之外，以实现确定性的来源和验证。在打包构建中，`CUA_DRIVER_PATH` 值或环境 PATH 候选都无法替换它；在 Wayland 上，两者也都无法绕过安全门控。

Xorg 运行时使用私有套接字、标准权限模式、按操作的 OpenMausBot 审批、遥测/更新检查抑制、严格的驱动身份、无叠加层启动及生命周期清理测试。这些防御仍然必要，但都不能替代启用 Wayland 所需的真实席位验收证据。Linux **Auto** 绝不会路由到用户桌面，任何 Cloud 或 Local VM 审批都无法授权它。

## 验证包变更

```sh
pnpm typecheck
pnpm test
pnpm check:electron
pnpm build:cua:linux          # networked, checksum-pinned staging
dbus-run-session -- xvfb-run -a pnpm smoke:cua-x11-input
pnpm package:linux:offline    # CUA staging is offline; builder caches must already be available
node scripts/verify-linux-package.mjs
pnpm smoke:linux-package
```

验证器会检查 `.deb` 元数据、桌面身份、确切的休眠 Cua 资源树及来源、SquashFS/DEB 目录模式、运行时路径策略，以及所有构建产物中匹配的二进制哈希。本地冒烟测试在不带 `--no-sandbox` 的情况下启动解包应用和 AppImage；CI 首先复现 `0.1.7` 就地 DEB 升级，然后对 `/opt/OpenMausBot/openmausbot` 运行相同的冒烟测试。这些通道证明内嵌服务器和 UI 在可选 Composio broker 停滞时仍可用，验证旧的本地控制加入选项被清除，并断言在 Xorg 或模拟 Wayland 上不会启动任何 Cua 可执行文件。底层运行时测试保留未来的私有守护进程契约，但不在打包应用中激活它。只有真实席位验收矩阵才能授权重新启用。

## 故障排除

### 缺少 agent CLI

在终端中直接运行 CLI，完成其登录流程，然后重启 OpenMausBot。如果它位于上述常见目录之外，请在测试时使用 `OMB_EXTRA_PATH`，并报告安装位置以便考虑自动发现。

### bot 需要计算机工具

选择 **Cloud**（Boat）并在 App Settings 中添加 Boat token，或使用 Local VM。Linux **This computer** 在 Wayland 上仍禁用。在 Xorg 上，请先从 **Local control** 卡片启用它。

### 本地控制未就绪

在 Xorg 上，按下 **Try again** 并使用卡片中显示的原因。捆绑包无需手动安装驱动或 `chmod`；升级的 DEB 会自动修复其确切的包拥有目录模式。在 Wayland 上，卡片会引导你使用 Ubuntu on Xorg，并有意不提供启用按钮。

### 屏幕预览无法启动

在 Xorg 上，用 `echo "$XDG_SESSION_TYPE"` 确认会话有活动显示；它应输出 `x11`。在 Wayland 上，确认 `xdg-desktop-portal` 和 GNOME portal 后端正在运行，然后点击 **Try again** 打开新的选择器。取消或停止共享绝不会导致自动第二次提示。

### AppImage 无法启动

确认可执行位和架构：

```sh
chmod +x OpenMausBot-*-x86_64.AppImage
file OpenMausBot-*-x86_64.AppImage
```

从终端运行一次以收集启动输出。不要仅为这个 AppImage 安装 `libfuse2`；该包使用静态运行时构建。