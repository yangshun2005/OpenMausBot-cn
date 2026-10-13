# 服务器提供商登录与自定义域名

仅使用隔离的测试夹具。切勿更改运行中用户的提供商账户、域名或配置来验证这些控制项。

## 自动化检查

```sh
pnpm exec vitest run server/custom-domain.test.ts server/provider-auth-sessions.test.ts server/drivers/codex-device-auth.test.ts server/drivers/acp/grok-device-auth.test.ts server/grok-sign-in-api.test.ts server/request-auth.test.ts server/config.test.ts src/components/DeviceSignIn.test.ts src/components/CustomDomainSettings.test.ts src/components/EngineSetup.test.ts src/components/EnginesSettings.test.ts
pnpm typecheck
pnpm build
pnpm build:server
node scripts/smoke-packaged-server.mjs
```

域名辅助测试会替换 DNS 和 HTTPS 响应。它们检查公网地址限制、固定 DNS、TLS 选项、重定向、超时、有界响应、工作区身份以及独立的一次性证明。它们不能证明客户的真实 DNS、证书或反向代理已正确配置。

## 使用离线提供商的真实设置 UI

1. 按照[夹具启动器](README.md#launch)操作，保留其打印的 URL、临时数据目录和持久日志路径。针对该显式 URL 运行 Doctor。
2. 在另一个空闲端口上启动 Vite，并将 `OMB_PORT` 设为打印的测试框架端口。不要让它代理到默认端口或用户的实时应用。
3. 对于域名 UI，打开 设置 → 远程访问 → 连接你的域名。输入 `bots.example.com`：确认 Type/Name/Value 行，并确认复制按钮不会提交表单。技术指南必须保持折叠在 **高级服务器设置** 下。启动器会刻意剥离操作员环境变量。要在浏览器自动化中测试可复制的 IP，仅拦截夹具的 GET `/api/settings/custom-domain` 响应，并提供 `serverIpv4: "8.8.8.8"`，保留其他字段。切勿将 DNS 指向此合成地址，也不要拦截验证 POST。公网接口检测和 `OMB_PUBLIC_IPV4` 覆盖由辅助测试覆盖。如果没有公网接口或覆盖，确认 UI 会要求管理员提供 IP，且不提供虚假值供复制。展开高级指南，确认其应用端口和 webhook 端口与夹具一致。提交 `http://bots.example.com`，确认出现 HTTPS 验证提示且不更改当前地址。真实 API 同样必须拒绝 localhost、IP 地址和包含配对码的 URL。
   在 390px 视口下，设置使用分区下拉菜单而非侧边栏。切换分区，复制一个 DNS 值，确认表单保持可读。Tab 和 Shift-Tab 必须停留在对话框内，跳过隐藏的侧边栏和折叠的高级指南中的链接。
4. 要测试 Codex，仅向**夹具的 config.json** 添加一个实例：驱动 `codex`，CLI 设为 `server/testing/fake-codex-login-cli.ts` 的绝对路径，环境变量 `OMB_DEVICE_AUTH_FIXTURE=1`，`HOME` 设为夹具的临时目录，`CODEX_HOME` 设为其 `.codex` 子目录。重新加载该隔离的提供商注册表并刷新预览。切勿替换为真实的 Codex 可执行文件。该夹具需要显式选择加入，且不发起网络请求。
5. 设置 → 引擎 → 连接 ChatGPT 应显示 `TEST-12345`、官方链接、过期时间和取消。**不要在 OpenAI 输入虚假代码。** 取消并重启，然后切换设置分区再返回；连接应恢复同一活动流程，而不启动第二次登录。
6. 在夹具的临时 HOME 中创建空的 `.omb-fake-codex-login-approved` 标记文件。离线进程会记录一个虚假的已登录状态。确认真实设置 UI 变为“ChatGPT 已在此服务器上连接”。
7. 停止 Vite 进程并中断前台夹具启动器。仅移除其拥有的子进程和一次性数据；保留打印的日志。

对于部署验收，需单独测试真实的公网域名和所有者发起的真实提供商登录。离线运行通过不得报告为真实的 OpenAI 账户登录或公网 DNS/TLS 可达性证明。