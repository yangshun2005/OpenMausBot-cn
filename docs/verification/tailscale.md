# Tailscale 独立 CLI 发现

macOS 应用可执行文件根据环境选择 GUI 或 CLI 模式。
由 Finder 启动的 OMB 进程必须设置 `TAILSCALE_BE_CLI=1`，使用自托管服务器的 `tailscale serve` 支持时也一样。无需安装包装脚本、交互式 shell，也无需修改用户环境。

## 离线验证

```sh
pnpm exec vitest run companion/test/tailscale-cli.test.ts companion/test/listener.test.ts companion/test/endpoints.test.ts companion/test/control.test.ts server/tailscale.test.ts
```

测试夹具会启动一个真实的回环 companion 控制服务器，使用一次性设备存储和合成网络接口。每次 CLI 启动都路由到离线子进程；绝不运行已安装的 Tailscale，也不改动真实 tailnet。若缺少 CLI 环境变量，子进程会以 GUI 启动文本正常退出，复现所报告的问题。

断言覆盖：

- `POST /tailscale/refresh` 在探测失败后恢复 MagicDNS 名称。
- 在配置的 companion 端口上广播 `http://fixture.tail1234.ts.net:8787`，同时保留独立的托管 HTTPS 端点。
- 不将裸 CGNAT IP 广播为手机路由，也不打开配对窗口。
- 候选回退、清除过期名称，以及针对 GUI 启动、JSON 格式错误和无 DNS 名称状态给出不同诊断。原始输出不记录日志。
- 自托管状态、HTTPS Serve 和 Serve 关闭时显式使用 CLI 模式。

这验证的是发现/控制契约，而非某个客户的 Tailscale 安装、iOS 传输行为或蜂窝网络可达性。端到端验收时，使用一次性 Mac/手机配对，确认 Remote access 报告 MagicDNS 名称，然后在两台设备均连接 Tailscale 的情况下将手机从 Wi-Fi 切换到蜂窝网络。公共隧道/DNS 故障需单独排查：托管 URL 返回部署 404，不会因 MagicDNS 发现成功而修复。