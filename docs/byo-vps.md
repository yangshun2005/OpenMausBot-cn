# 使用自有 VPS

OpenMausBot 可以把你已有的 Linux 服务器变成 Bot 的电脑。Agent 进程留在你本机；Docker 自带的 SSH 传输连接到 VPS 上的守护进程，每个 Bot 在那里获得一个受管、加固的 Cua 容器——一个它能看见并控制的 Linux 桌面。SSH 是唯一涉及的凭据，也是唯一暴露的入口：OpenMausBot 从不在 VPS 上开放公网端口，从不存储你的 SSH 密钥或口令，也从不远程运行 Agent。

## 可用功能

- 在 VPS 上的受管容器中，每个 Bot 一个 Linux 桌面，通过官方 Cua 工具驱动。
- Computer 面板和对话记录中的实时屏幕预览，与 Boat 相同。
- 显式选择 **Cloud** 并搭配 **Self-hosted VPS** 后端时，会创建或启动容器。**Auto** 默认复用已就绪的容器；一个默认关闭的 **Start VPS automatically** 开关可让该 Bot 在需要时准备或唤醒其受管容器。定时和手动触发的例行任务始终会启动它，且在无法连接 VPS 时绝不回退到本机桌面。
- 可在本地或从配对的桌面客户端进行交互式 **Take control**。主机仅将 noVNC 绑定到随机的 `127.0.0.1` 端口，关闭查看器时一并关闭隧道，从不在 VPS 上发布 VNC。只有在主机上为该设备启用 **Cloud desktop access** 后，配对客户端才会通过受管 HTTPS 或 Tailscale 获得短时、设备范围的转发。

## 前提条件

- **本地：** `docker` CLI，版本 18.09 或更高（`docker -H ssh://` 从该版本开始提供）。Docker 守护进程无需在本地运行——只用到 CLI。
- **VPS 上：** 在 x86_64 Linux 上运行 Docker 守护进程（`dockerd`）。
- **SSH 用户**必须位于 VPS 的 `docker` 组中，这样 `docker info` 无需 sudo 即可运行。

对最后一点要有清醒认识：在该机器上，属于 `docker` 组等同于 root。任何能与守护进程通信的人都能把主机文件系统挂载进容器。使用此功能意味着完全信任该 VPS——以及任何其他能访问其 Docker 守护进程的人。给 Bot 一台专用服务器，不要用还存放着你不愿交给 Agent 的东西的服务器。

## 必需的 SSH 配置别名

OpenMausBot 只通过 `~/.ssh/config` 中的命名别名连接——你在 App Settings → Connections 中输入该别名，仅此而已。每个 Bot 操作都变成通过 SSH 执行的 `docker exec`，因此应用自行提供连接复用和快速失败超时：它通过自己的 `ssh_config`（位于 `~/.openmausbot/ssh/` 下）运行每条 VPS 命令，该文件先包含你的文件，并在你的别名未设置 `ControlMaster`、`ControlPersist`、keepalive 和连接超时的地方补上。你的别名设置的任何内容优先。启动、预览、Bot 工具及其连接探测都使用相同设置。
对于较长的数据目录路径，应用将其控制套接字放在 `/tmp` 下私有的、用户拥有的目录中，以符合 OpenSSH 的套接字路径长度限制；配置文件仍留在应用数据目录中。
Windows OpenSSH 不支持连接复用，因此该平台保留其常规 SSH 设置。
下面的配置块仍是推荐形式，也是你在应用之外使用自己的 `ssh` 时的配置：

```
Host my-vps
  HostName 203.0.113.7
  User deploy
  IdentityFile ~/.ssh/id_ed25519
  ControlMaster auto
  ControlPath ~/.ssh/cm-%r@%h-%p
  ControlPersist 60m
  ServerAliveInterval 15
  ServerAliveCountMax 3
  ConnectTimeout 10
```

- `ControlMaster`/`ControlPath`/`ControlPersist`——每个操作都是 docker-over-SSH exec；复用把每条命令的连接变成一条持久连接上的毫秒级操作。
- `ServerAliveInterval`/`ServerAliveCountMax`/`ConnectTimeout`——VPS 掉线必须快速失败（一分钟以内，连接十秒），而不是让一轮对话挂起等待已死的 TCP 会话。

**先手动确认主机密钥。** 在让 OpenMausBot 指向该别名之前，先手动连接一次：

```sh
ssh my-vps true
```

这样主机密钥就按你的意愿写入 `known_hosts`。应用从不自动接受主机密钥——主机未知的别名会直接失败，直到你完成这一次操作。

## 安全

- **无公网端口。** 受管容器创建时不发布任何端口，且 OpenMausBot 拒绝使用任何发布了端口的容器——该检查在每次附加前运行，而不只是在创建时。实时视图通过 SSH 到达容器的私有网桥地址，且在你本机上仅限回环。
- **将 VPS 防火墙限制为仅 SSH**，最好只允许你的 IP。OpenMausBot 的任何操作都不需要开放其他入站端口，因此开放其他端口纯属攻击面。
- **不存储任何敏感信息。** OpenMausBot 唯一持久化的是别名本身（`~/.openmausbot/config.json`）；密钥、口令和 Agent 状态都留在 SSH 中。别名也不会同步到配对的手机——配套应用只报告是否已配置，从不报告名称。
- 容器本身以加固方式运行：丢弃 capabilities、私有 network/IPC/cgroup 命名空间、无主机挂载或设备、不自动删除。复用容器时仍会执行所有权、固定镜像和隔离检查；未通过检查的容器会被拒绝，而不是修复。

## 容器生命周期

每个 Bot 在 VPS 上拥有一个容器，命名为 `openmausbot-vps-<bot>-<hash>`——跨重启稳定，且与 Bot 的显示名称无关。

新容器默认 4 GiB 内存且无额外 swap、2 个 CPU、512 个 PID、512 MiB 共享内存，重启策略为 `unless-stopped`。这些是创建时的默认值，不是兼容性要求。已有的受管容器可以保留自定义的资源和 OOM 设置、无限制预算或其他重启策略，包括旧式的 `no` 策略。OpenMausBot 不会重置这些设置，也不会为强制其默认值而重建容器。重启行为遵循所选的 Docker 策略；例如，`always` 可在 Docker 守护进程重启后唤醒已休眠的容器。

- **Provision**（为 Bot 选择 **Cloud**，或点击面板按钮）：如有需要在 VPS 上构建固定版本的 Cua 镜像，容器缺失则创建，已停止则启动，并等待桌面响应。
- **Start** 仅唤醒已存在的停止容器；从不创建容器。
- **Sleep** 停止容器。VPS 不再为它消耗 CPU；文件系统保持不变。
- **Remove** 由你手动执行，当 Bot 不再需要该服务器时：
  `docker -H ssh://my-vps rm -f <container>`。OpenMausBot 从不自行删除容器。

什么能保留、什么会丢失：sleep/start 保留容器文件系统；删除——包括 Cua 镜像升级后的重建，因为固定到旧镜像的容器会被拒绝而非复用——会清空它。把容器文件系统视为**一次性**：Bot 必须保留的任何内容都应在容器被删除前离开 VPS（推送、上传或粘贴回聊天）。

设为 **Auto** 的 Bot 默认对生命周期只读。它仅在容器已运行且已验证时才附加。如果不存在本地回退，该轮仍会运行，因此不需要屏幕的工作（队友的撰写、Chief 的跟进）绝不会因它不使用的 VPS 而中断，且 Bot 会被告知 VPS 为何不可用以及该怎么做，从而如实说明而不是声称做了屏幕工作。定时任务则会失败并说明原因。按 Bot 启用 **Start VPS automatically** 可让 Auto 准备或唤醒该 Bot 的受管容器；该开关默认故意关闭。

每轮开始时都会检查容器，因为计算机工具在引擎启动它们的瞬间就连接到运行中的容器。同时开始的轮次共享一次检查，而不是各自排队。

## 同时进行多轮

每个 Bot 有一个容器，但容器不是瓶颈——里面的桌面才是。该 Bot 的任意数量线程（聊天、任务、例行任务）都可以在 VPS 挂载的情况下同时运行。桌面归第一个调用计算机工具（截图、点击、通过 Cua 驱动执行命令）的线程所有，并保持到该轮结束；从不接触计算机工具的线程绝不会被阻塞。当另一个线程持有桌面时，试图访问屏幕的线程会显示

> 正在等待轮到它使用这台电脑——队列中第 *position* 位——*bot* 正在运行 *thread*。前面的轮次结束后会自动开始。

其计算机调用会等待桌面，最长 45 秒，并在空闲的瞬间得到响应；超时的调用会告诉模型桌面仍被占用，应继续用另一次调用等待，而不是结束本轮。等待超过五秒的调用即使到那时也不会执行——开始得那么晚的操作可能超出引擎对单次工具调用的时限，并在模型重试时运行两次——因此模型会再次调用，而那次调用会立即执行：此时桌面已归该轮所有。当桌面到手时，提示条会变为 "Computer free — continuing"，若该轮先结束则变为 "Stopped waiting for the computer"。长时间等待会把该轮停在封顶值（默认 30 分钟），仍会指明持有者，并在桌面空闲时自行恢复。有人接管控制时发起的调用会以同样方式等待他们；一旦他们交还，该调用不会执行（屏幕可能已变化），模型会重新截图。超过同一封顶值的持有不再等待：Bot 会停止并说明正在等待交还控制权。容器生命周期操作（创建、启动、停止）仍按容器逐个进行。

## 故障排查

正常的屏幕刷新不应以 "the VPS is being prepared" 中止新轮次。
轮次设置会等待待处理的预览，重叠的准备请求共享同一操作。
Stop 和 Delete 仍保持独占；它们不会与待处理的捕获竞争，也不会悄悄重建其容器。
如果 docker-over-SSH 命令超时，应用会在释放该命令前清理其 Docker/SSH 进程。重试是显式的：可能产生变更的命令绝不自动重放。

按应用所走的同一路径排查，从最省事的信号开始：

1. **别名手动可用：** `ssh my-vps true` 静默返回。出现密码提示说明密钥/agent 未设置；出现主机密钥提示说明尚未进行首次手动连接。
2. **Docker over SSH 能到达守护进程：** `docker -H ssh://my-vps info` 打印服务器详情。权限错误说明 SSH 用户不在 `docker` 组中。
3. **Provision：** 在 Bot 的 Computer 面板中选择 **Cloud** 并搭配 **Self-hosted VPS** 后端。首次 provision 会在 VPS 上拉取并构建 Cua 镜像，可能需要数分钟；之后会很快。
4. **阅读状态。** 面板按检查顺序准确呈现服务器发现的情况：别名未配置 → 守护进程不可达 → 镜像缺失 → 容器缺失/已停止 → 容器未受管或不安全（端口、挂载、加固）→ 桌面未就绪。每条消息都会指明需要修复的步骤。