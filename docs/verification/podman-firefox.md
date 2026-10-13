# Podman 本地 VM 中的 Firefox 沙箱

修复 #853。Podman 默认的 seccomp 策略将 `chroot` 限制在 `SYS_CHROOT` 上；Firefox 在建立自己的 Linux 沙箱时会使用该系统调用。受管 Podman 桌面现在保留该能力，与 `SETUID` 和 `SETGID` 并列。有效能力与边界能力的校验要求两者集合完全一致。Docker 和 VPS 策略保持不变。

使用旧能力集的现有 Podman 桌面需通过正常的本地 VM 生命周期重建。保留工作区绑定挂载；不要删除用户文件，也不要静默重建正在使用的桌面。此修复不需要也不接受特权模式和非受限安全配置文件。

旧容器会被报告为不安全/未就绪，并提示重建；状态检查和对话轮次不会自动替换它。当前工作完成后，将所需文件保存到 `/home/cua/workspace` 下。对于每个 bot 的桌面，使用该 bot 的 Computer 面板重建，并确认现有提示。对于共享桌面，使用本地 VM 设置中的 **Delete and recreate**。现有的移除/运行生命周期会保留相同的工作区绑定挂载。容器中其他位置的文件（包括工作区之外的 Firefox 配置文件）将被丢弃；请先将所需内容复制到工作区。重启旧容器不会添加新能力。

## 隔离的前后验收

显式设置 `OMB_VERIFY_PODMAN` 和 `OMB_VERIFY_MACHINE`，并在该测试引擎上准备好受管镜像，然后运行：

```sh
node --experimental-strip-types scripts/verify-podman-firefox.ts
```

该脚本会创建一次性桌面和引擎主机临时工作区。它比较带与不带 `SYS_CHROOT` 的相同生成参数，使用全新配置文件无头运行 Firefox，并校验 PNG 字节。打过补丁的容器还会在客户机回环上使用 Python 标准库和 Firefox 自带的 Marionette 服务器运行 `scripts/testing/firefox-sandbox.py`；不发布调试端口。它读取 [about:support 沙箱字段](https://searchfox.org/firefox-main/source/toolkit/content/aboutSupport.js)，并要求有效内容进程沙箱级别为正。单独的全新配置文件对照组设置 `MOZ_DISABLE_CONTENT_SANDBOX=1`，要求级别为零，且必须使相同的启用沙箱断言失败。两个级别都记录在 `receipt.json` 中。脚本只清理它创建的容器和工作区。日志和回执保留在 `.omb-scratch/firefox` 下（可用 `OMB_VERIFY_OUTPUT` 覆盖）。

2026-09-06：Windows/WSL2 Linux x86_64，无根 Podman 5.8.3，受管驱动 0.20.0-v4 镜像。基线：`chroot: EPERM`，超时退出码 124，无 PNG。打补丁后：Firefox 退出码 0，PNG 有效，无 `chroot: EPERM`。Firefox 报告配置/有效内容沙箱级别为 6/6；禁用对照组报告 6/0，被启用沙箱断言拒绝。相关容器/VPS 测试：71 项通过；服务器 TypeScript 检查通过。

这是浏览器启动/渲染证据，不是交互输入测试。原生 Linux、ARM64、SELinux 强制模式及其他运行时版本未在此次运行中认证；未重复运行完整仓库测试套件。