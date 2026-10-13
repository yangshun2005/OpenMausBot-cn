# 本地 VM 停止与恢复

在一次性 home 目录中运行聚焦的生命周期检查：

```sh
pnpm exec vitest run server/container-computer.test.ts server/local-vm-stop-reason.test.ts server/local-vm-idle.test.ts server/group-local-vm.e2e.test.ts server/routes/desktop-viewer.test.ts src/components/LocalComputerSection.test.ts src/components/ComputerPanel.test.ts src/components/ComputerPanel.i18n.test.ts
node --experimental-strip-types scripts/verify-local-vm-resume.ts
```

浏览器方案使用 `launchVerificationServer`、fake engine 和私有浏览器配置文件。它通过合成的桌面传输挂载真实的 ComputerPanel；绝不接触宿主机的 Docker daemon。设置 `OMB_AGENT_BROWSER_PATH` 和 `AGENT_BROWSER_EXECUTABLE_PATH` 以复用已安装的工具。其回执会打印隔离的服务器 URL 和持久日志路径；除非 `OMB_UI_EVIDENCE_DIR` 指定其他目录，截图会保留在该日志旁边。在 PATH 中有 ffmpeg 的情况下设置 `OMB_UI_RECORD=1`，还会额外保存 `resume.mp4`。就绪断言会在桌面保持未就绪状态时等待两次实际的状态读取；不使用固定延迟。

浏览器检查证明：现有的共享 VM 显示"已停止"和空闲说明，Start 只发出一次请求，该请求返回后按钮保持禁用和忙碌状态（此时 Cua 仍在预热），就绪的桌面会替换空状态，且不会发出移除或重建请求。它还检查 Settings 会为已停止的 VM 提供 Start 并等待就绪。

服务器 fixture 运行实际的 HTTP 路由和空闲计时器，仅替换容器边界，并通过仅测试用的 loader 缩短空闲窗口。它验证空闲关闭会停止而不删除，停止原因在服务器重启后仍然保留，共享启动和按 bot 启动都能工作，即使达到按 bot 的容量上限，Auto 轮次也能恢复现有桌面。原因记录与运行时的完成时间戳匹配；之后的外部停止或未知时间戳会得到中性的已停止说明。

2026-10-01，Linux 上的 Docker：一个单独创建的、使用固定 driver-0.20.0-v5 镜像的一次性容器通过了两次停止/启动循环。每个循环都通过了生产就绪探针（driver 版本、健康报告和完整截图），并保留了 `/home/cua` 和 `/opt` 下的标记文件。另外两次突然关闭循环使用 `docker kill --signal KILL`，并通过 `docker cp` 确认在调用生产 Start 之前 `/tmp/.X1-lock` 仍留在已停止的容器中。两次恢复都通过了相同的就绪和标记检查。固定镜像的 `/usr/local/bin/start-vnc.sh` 会在启动 VNC 之前移除 `/tmp/.X1-lock` 和 `/tmp/.X11-unix/X1`，解决了此前一概拒绝恢复背后的陈旧锁问题。fixture 使用空的临时工作区，之后被移除。无需重建镜像。该实际验收不覆盖 Podman、Apple container、原生 iOS，或镜像替换后的持久性。

## 审查证据

[before 截图](evidence/local-vm-resume/before.png)是在上游 `9f0c33d00fc490491bf794704fd192046d2f31eb` 处使用未改动的生产代码捕获的，使用相同的一次性 fixture 和经过检查的已停止 VM 响应。fixture 的状态文本反映了基线服务器拒绝恢复。只有 fixture 传输被复制到该检出中。

[stopped](evidence/local-vm-resume/stopped.png)、[starting](evidence/local-vm-resume/starting.png) 和 [ready](evidence/local-vm-resume/ready.png) 截图展示了改动后的面板。[启动视频](evidence/local-vm-resume/resume.mp4)记录了 Start、按钮禁用时的两次就绪轮询，然后是就绪预览。截图捕获的是真实的面板元素；视频仅隐藏 fixture 控件。桌面预览明确是合成的，不声称是实时远程会话或原生移动端验收。