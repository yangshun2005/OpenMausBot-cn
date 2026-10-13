# Group 与 Goal 本地 VM 路由

相关：#430。此改动为房间成员提供其配置的本地 VM；它不会创建房间自有的桌面或房间 Computer 面板。

既有的直接轮次在缺少终止事件后残留陈旧租约的问题，单独在 [#860](https://github.com/milind-soni/OpenMausBot/issues/860) 中跟踪。该后续工作需要区分不同的直接调用身份，并添加测试以保护替换轮次免受旧看门狗回调影响；这不在本次房间切片范围内。

`runGroupMemberTurn` 现在在认领与直接轮次相同的目标作用域租约后，挂载发言机器人的 computer MCP 和提示词。设置取消和终止清理会释放租约和设置所有权。超时或停滞的提供者会通过现有的中断宽限期保留桌面；随后回退逻辑在释放机器人前移除租约记录。Computer 能力绑定到确切的 VM 认领，并在过期或成员交接时停止工作。

## 可选隔离验收

在显式选定的 Podman 测试机器上准备托管桌面镜像。构建渲染器（`node node_modules/vite/bin/vite.js build`），将 `OMB_VERIFY_PODMAN` 设为可执行文件的绝对路径，`OMB_VERIFY_MACHINE` 设为该连接，然后运行：

```sh
node --experimental-strip-types scripts/verify-group-vm.ts
```

启动器会创建临时 home、数据目录、显式回环服务器和假 Claude 引擎。它授予该 fixture 访问所选引擎的权限，仅创建 fixture 专属的桌面目标，并在之后移除它们。不要替换为真实应用 URL 或数据目录。JSON 回执记录每个发言者的确切 MCP 目标、computer 关闭的情况，以及每个已结算发言者 computer 能力的撤销。它仅在清理成功后写入 `.omb-scratch/verification-logs/group-vm-routing.json`。脚本在每次发送前移除先前的转储，并要求每个 Goal（包括 computer 关闭）都结算。清理错误会导致运行失败。

2026-09-07 在 Windows/WSL2、rootless Podman 5.8.3 上：

- 两个 Goal 发言者收到不同的匹配桌面目标并结算。
- 一个切换为 `computer: off` 的发言者未收到 computer MCP。
- 每个已结算发言者的 computer 控制能力返回 HTTP 401。
- 六个回归场景运行真实的隔离服务器和假 Claude，仅替换容器边界：就绪失败、就绪期间 Stop、共享成员交接/冒充、租约过期、超时和停滞清理。
- 在后续修复之前，相同的就绪、过期和超时回归在 `a9ec061c` 上失败；修复后通过。
- 集成上游浏览器引擎第 2 步（`b3c2f74b`）后：Goal、浏览器、租约和 fixture 启动器覆盖通过 68 个测试，1 个既有 TODO。专用的真实服务器浏览器引擎房间挂载测试也通过。
- 服务器 TypeScript 检查和服务器打包通过。

此假引擎 fixture 证明的是路由，而非模型驱动的点击或屏幕流。Firefox 沙箱兼容性和日文客户机字体是单独的改动。完整仓库覆盖由 PR CI 提供；此本地配方是定向的。

直接轮次回归还会在机器人默认为 Cloud 时，通过本地 VM 固定的对话发送两个不同请求，检查实际的 MCP 描述符、提示词、匹配的预览界面和能力过期。频道成员的显式 This computer 和 Cloud（Boat 或 VPS）目标，与其机器人线程挂载方式完全一致：相同的检查、相同的控制门，以及唤醒或创建云计算机时相同的同意边界。fixture 证明主机描述符通过一个在轮次结算时关闭的工作门到达发言者，缺失的 CUA 驱动会被报告而不是在没有承诺工具的情况下派发，以及 Cloud 发言者唤醒其自有的已归档 Boat、在那里运行，并在下一轮次归还。VPS 挂载共享机器人线程代码路径，但尚无频道 fixture。频道仍没有对话固定，也没有到主机的 Auto 回退。

同一隔离服务器现在覆盖聊天驱动的 computer 选择：agents 工具发现就绪、可启动和可配置的目标，而不改变它们。选择一个会结束旧轮次，立即阻止其先前的 computer 和浏览器能力，并用新工具和一条用户历史记录恢复原始请求。Auto 优先选择就绪目标而非创建云计算机。假 Boat 边界验证唤醒、创建、复用，以及 computer 在发现和派发之间消失；不会创建真实的付费 computer。Stop、提供者失败、Off、新的排队请求，以及被拒绝的进行中手动界面更改都有回归覆盖。VPS fixture 单独证明仅在选择后启动已停止容器和创建缺失容器。

一个限制仍然明确：已停止的本地 VM 不会被破坏性重建以使选择成功。每个引擎在 Cloud 上保留自己的模型，以 Boat 作为其 computer 工具；没有引擎在 Boat 自己的 agent 上运行轮次。

在没有容器引擎的情况下运行回归覆盖：

```sh
node node_modules/vitest/vitest.mjs run server/group-local-vm.e2e.test.ts server/vps-routing.test.ts server/local-vm-lease.test.ts server/group-goal-run.test.ts server/group-goal-run.e2e.test.ts server/group-goal-wait-cap.e2e.test.ts server/control-omb.test.ts
```

`server/testing/group-local-vm-hooks.mjs` 中的仅测试 Node 加载器替换容器状态，并在该子进程内控制租约过期和看门狗/截止时间计时。生产启动器从不导入它。这些回归测试不使用任何提供者或实时桌面；上述可选 Podman 验收覆盖真实路由。

重启回归通过 fixture 的生命周期边界创建每机器人 VM，在移除容器的同时保留工作区文件，并在同一一次性 home 中重启服务器。Auto 在容量空闲后恢复先前的 VM，保留其文件，并且既不探测也不配置从未有过 VM 的机器人。容量占用的情况保持未配置。

池回归在一个座位的检查被阻塞时并发启动两个冷座位，然后显示第三个轮次等待并复用已释放的座位。它们在完整的既有每机器人清单下重复：每机器人上限仍拒绝其下一个桌面，而切换到池模式则允许其自己配置的座位。`maxInstances` 限制所选模式，而非跨模式更改保留的容器总数。两个场景都使用上述隔离容器边界，并要求实际的假引擎 MCP 描述符与分配的座位匹配；不联系任何真实桌面或模型。