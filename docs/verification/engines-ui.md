# 引擎库：引导与设置

使用真实渲染器，配合示例引擎状态和一次性服务器：

```sh
node --experimental-strip-types scripts/verify-engines-ui.ts
```

打开它打印的 `previewUrl`。底部工具栏可在真实的设置模态框与引导流程之间切换，应用 Midnight/Atelier 皮肤，并切换一个合成的 Antigravity 连接。**引导预览**会直接打开欢迎流程的引擎环节。提供商安装、登录、路径保存和账户管理请求都会被仅用于 fixture 的中间件拒绝；不涉及真实的提供商登录或用户配置。被拒绝的设置请求可用于检查错误展示，但不能证明提供商认证可用。图标更新是例外：fixture 中间件仅在预览进程生命周期内将其存入合成实例。**切换示例 ChatGPT 套餐**只改变合成账户状态；不会登录或联系 OpenAI。Vite 生成的源码缓存保留在检出目录下被忽略的 `.omb-scratch/engine-preview-vite` 目录中；fixture 账户、主目录和应用数据均为一次性。将它们分开可防止延迟的缓存写入重新创建已删除的 fixture 目录。

## 检查项

1. 在 1280px 下，设置将卡片分为 Ready / Needs setup 两组，两列布局；引导使用一个紧凑列表，带状态胶囊和折叠的设置行。两者都保留提供商标记和所选皮肤的颜色。在 390px 下，设置卡片变为单列；页面无水平溢出；引擎列表滚动时，引导的 Continue 和设置的 Close 仍可触达。
2. 在设置中展开 Antigravity，展开 **CLI path and updates**，选择 **Set CLI…**，输入 `/preview/keep-this-draft`。点击 **Toggle sample connection**。其卡片移入 Ready，同时保持展开并保留该未保存路径。折叠/重新打开卡片；草稿必须保留。
3. 在引导中，切换示例连接。分组计数和 Antigravity 的状态立即变化，无需重新打开引导或改变焦点。`Claude · Local` 尽管没有云登录，仍保持 Ready。
4. 当 Antigravity 需要设置时，点击 **Sign in with Google**。合成错误出现，按钮重新变为可用。设置必须仍可关闭。此 fixture 有意不触发真实的 Google 浏览器、回调和账户流程。
5. 展开 Codex/Claude，检查账户身份和受保护的登出控件是否保留。展开高级 CLI 折叠区以访问路径、重置和更新控件。Tab 到卡片并按 Enter/Space；其原生折叠区必须可用，并有可见的焦点指示器。
6. 展开 Kimi 找到 **Install Kimi on this server**，展开 OpenCode 找到 **Update OpenCode on this server**。分别点击，确认 fixture 错误出现且按钮重新可用。终端命令保留在 **Prefer a terminal?** 下。这些点击永远不会执行真实安装。
7. 展开 Claude，使用其卡片内的 **Add Claude account**。输入账户名，折叠/重新打开卡片，确认草稿保留。取消，然后在 ChatGPT 套餐卡片中用 **Add ChatGPT account** 重复。提交一次，检查仅 fixture 的错误留在该卡片内且表单仍可编辑。这两个操作都不应出现在引擎网格下方、无关提供商中或只读托管卡片中。提供商图标编辑控件不应出现。在 390px 视口重复。在 Antigravity 中用 **Add Google account** 重复。切换其示例连接，确认 `google@example.test` 出现且不丢失账户名草稿。保存失败必须使草稿仍可编辑。提示应说明在模型选择器中选择账户，而非配额轮换。
7. 展开 Codex，在 **Provider icon** 下选择 **Google Gemini**，然后重新加载。所选图标应持久保留，而同级实例保留各自的图标。上传一个小型 PNG、JPEG 或 WebP，检查其在卡片和选择器中渲染。拒绝不支持或无效的图片，并显示可见错误。**Reset** 恢复默认图标。在桌面和窄宽度下检查两种主题中的控件。服务器 API 测试单独验证磁盘持久化；此预览的合成状态不会在进程重启后保留。
8. 展开 **ChatGPT plan**。其设置卡片提供 **Continue with ChatGPT**，而非设备码或终端登录。切换示例套餐连接：首次使用对话框应聚焦 **Got it**，在 390px 下适配，并将 ChatGPT 套餐用量与 MausBot Cloud 分开说明。关闭它，断开并重新连接示例：它不应再次出现。已连接的卡片保留 **Using ChatGPT plan**、**Manage usage**、合成账户地址和受保护的登出。检查 Midnight 和 Atelier；无水平溢出。

自动化覆盖：

离线设置生命周期还驱动真实 store/卡片，从缺失 CLI 到刷新失败、安装、需要登录和就绪，全程无需重新打开应用。它使用合成提供商响应，并验证隔离服务器报告自己的安装器平台；它从不安装或登录真实提供商。证据和 390px 截图写入 `.omb-scratch/provider-setup` 下：

```sh
node --experimental-strip-types scripts/verify-provider-setup.ts
pnpm exec vitest run src/state/engine-refresh.test.ts src/components/EngineSetup.refresh.test.ts server/drivers/acp/probe-errors.test.ts
```

```sh
pnpm exec vitest run scripts/verify-engines-ui.test.mjs
pnpm exec vitest run server/antigravity-account-api.test.ts server/drivers/antigravity-account.test.ts src/components/ChatGptPlanSignIn.interaction.test.ts
pnpm exec vitest run src/components/EngineLibrary.test.ts src/components/EnginesSettings.test.ts src/components/EngineSetup.test.ts src/components/ClaudeAccountSettings.test.ts src/components/CodexAccountSettings.test.ts src/components/ClaudeSignIn.test.ts src/components/DeviceSignIn.test.ts src/components/GrokSignIn.interaction.test.ts src/components/EngineSetup.grok.test.ts src/components/EngineUpdateNotice.test.ts src/components/SettingsModal.appearance.test.ts src/components/ModelPicker.test.ts
pnpm typecheck
pnpm i18n:check
pnpm build
```

中断前台启动器只会停止其拥有的子进程并移除其临时数据。保留打印的服务器日志作为证据。响应式检查后恢复任何临时浏览器视口覆盖。此配方不证明真实提供商安装或登录；真实认证边界请使用单独的[离线服务器登录配方](server-settings.md)。

Antigravity HTTP fixture 创建两个独立账户，拒绝无效和非管理员请求，切换机器人的模型账户，并重启其隔离服务器以验证持久化。身份测试模拟 Google 的 token 和 userinfo 响应，覆盖缓存、凭据替换和查找失败。两者都不使用真实 Google 账户，也不证明实时 Google 同意或配额行为。

在未完成的 Vite 依赖转换期间停止，仍可能从 `ui.close()` 报告退出码 13。应用服务器及其一次性数据会先被清理；请验证这些结果，而不是将该开发工具退出码视为已安装应用的失败。就绪预览关闭退出码为 0；启动取消报告 launch-cancelled 错误。