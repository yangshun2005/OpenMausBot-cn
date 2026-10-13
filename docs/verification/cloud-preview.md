# 云屏幕预览

在终端中直接启动隔离的渲染器测试夹具：

```sh
node --experimental-strip-types scripts/verify-cloud-preview.ts
```

在浏览器或隔离的 Electron 窗口中打开打印出的 `previewUrl`。
该夹具在临时 home 目录中启动标准的 fake-engine 测试框架，
通过映射的控制命令创建自己的测试 bot，并挂载真实的 `ComputerPanel`。
仅模拟云传输。它从不需要 Boat API key，也从不联系云服务商。Ctrl-C 会停止两个服务器
并删除其临时数据；打印出的测试框架日志会保留。

使用 computer use 验证以下状态转换：

1. 初始连接时，出现 **Cloud screen connected**。面板仅在其预览处于活动状态时
   接收实时帧，因此在其连接前发送的帧无法掩盖此截图。
2. 选择 **corrupt**，然后选择 **Reconnect panel**。面板必须显示图像
   错误和 **Retry preview**，而不是带 Open 按钮的空白图像。
3. 选择 **slow**，然后选择 **Retry preview**。面板立即显示
   **Connecting to the screen…**，然后在 12 秒后显示图像。
4. 选择 **failed**，然后选择 **Reconnect panel**。服务商错误出现在
   预览中。选择 **connected**，然后选择 **Retry preview** 以恢复。
5. 打开 **Busy**，然后选择 **Publish live frame**。新的实时帧出现。
   停止发布：14 秒内，截图轮询恢复并重新显示
   **Cloud screen connected**，即使 bot 仍处于忙碌状态。
6. 打开 **Busy**，然后选择 **Reconnect panel**。该夹具与服务器一样，
   在回合活动期间以 409 拒绝 `provision`。面板仍必须连接并显示屏幕
   （它附加到就绪的 boat 而无需 provision）——而不是显示 **Couldn't reach the computer**
   以及红色的 "being used by an active turn" 警报。关闭 **Busy**：面板保持
   连接。
7. 选择 **timeout**，然后选择 **Reconnect panel**。90 秒后，加载器
   变为超时错误并显示 **Retry preview**。选择 **connected** 并
   重试；连接必须在不重启应用的情况下恢复。
8. 当 **slow** 处于挂起状态时，切换到 **connected** 并选择 **Reconnect panel**。
   新连接必须立即显示；被取消的请求既不能阻塞它，也不能在之后覆盖其帧。
9. 选择 **held**，打开 **Busy**，等待下一次轮询。关闭
   **Busy**：挂起的捕获不得被取消。**Reconnect panel**
   确实会取消其客户端请求，但模拟的主机捕获会继续。
   替换的预览必须重试争用而不出现断开警报。
   选择 **connected**，然后选择 **Release held capture**；预览会自动恢复，
   旧捕获永远不会覆盖它。
10. 从已连接的帧开始，选择 **failed** 并打开 **Busy**。在
    错误出现后，选择 **contended** 和 **Retry preview**。重试期间最后一帧
    保持可见。十秒后，持续的争用变为可操作的错误，但重试会继续：
    选择 **connected** 即可恢复，无需再次点击 Retry。
11. 打开实时桌面。该夹具会保持其加入直到 **Release desktop
    join**；在此期间截图轮询必须暂停。使用
    **Panel → Remote desktop** 重复。查看器是模拟的，从不是原生窗口。
12. 在 **Busy** 打开时选择 **unconfigured**。其永久 HTTP409 必须
    显示 **VPS is not configured** 并带 Retry，而不是无尽的连接状态。

自动化浏览器回归针对同一夹具运行这些争用、取消、解码帧保留
和加入暂停检查：

```sh
OMB_UI_E2E=1 node node_modules/vitest/vitest.mjs run scripts/testing/cloud-preview.e2e.test.ts
```

它复用标准的隔离 UI 测试框架，并打印临时数据路径
和持久服务器日志。要复用已安装的测试二进制文件，请显式设置
`OMB_AGENT_BROWSER_PATH` 和 `AGENT_BROWSER_EXECUTABLE_PATH`。

该夹具测试实际的图像解码、请求取消、新帧
选择和渲染器反馈。取消后主机工作继续的情况在页面内模拟；
`server/vps-routing.test.ts` 单独使用 fake SSH/Docker 覆盖真实的 HTTP
路由。两个夹具都无法证明真实的 VPS 连接、Boat
provisioning、原生查看器窗口或账户认证。在更改这些路径时，
请使用显式隔离的服务商夹具单独测试它们。