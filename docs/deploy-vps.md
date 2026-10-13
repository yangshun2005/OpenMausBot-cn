# 在 VPS 上部署 OpenMausBot

从一台空白 Linux 服务器，到 OpenMausBot 全天候运行其上，可从你的笔记本电脑、桌面应用和手机访问，所有笔记本电脑关闭时机器人仍在工作。本文假设你只会打开终端并粘贴命令。大约二十分钟，大部分时间都在等待。

本文涵盖三种让服务器可访问的方式。任选其一，其余内容相同。

| | 你需要 | 谁能访问 | 最适合 |
|---|---|---|---|
| **A. 公网地址，无需域名**（`serve --tunnel`） | 一个 OpenMausBot 账户（邮箱验证码） | 任何持有配对码的人，通过 HTTPS | 最快路径；蜂窝网络下的手机 |
| **B. 你自己的域名**（Docker + Caddy） | 一个域名，80/443 端口 | 任何持有配对码的人，通过 HTTPS | 你拥有的永久地址 |
| **C. 你的 Tailscale 网络**（`serve --tailscale`） | 服务器和你的设备上都安装 Tailscale | 仅你的 tailnet | 最私密；完全不公开 |

无论选哪种，登录方式相同：每台设备用短码**配对**一次，之后保持登录。会话持续 30 天；剩余时间不足一半时使用会续期至完整 30 天，自配对起最长 180 天（`OMB_SESSION_TTL_DAYS` 和 `OMB_SESSION_MAX_DAYS` 可修改这两个数值）。没有密码。

## Hetzner 启动检查（2026-09-07）

路径 A 在一台空白 Hetzner CX43（x86-64，Ubuntu 24.04）上验证，使用 Node 24.20.0 和 Codex 0.153.4。已发布的 0.1.60 基线之后应用了本次变更的本地候选修复；这些结果并不证明未经修改的已发布 0.1.60 包可用。

配对后的 Web UI 完成了真实的 GPT-6-Astra 对话、两次自动调度的例程运行，以及浏览器标题/截图测试。浏览器需要下方 Ubuntu 沙箱设置。完整重启后服务与托管隧道自动恢复；已配对设备、机器人、对话和暂停的调度均保持可用。新的真实对话轮次也成功。

停止应用数据后进行的备份恢复后文件哈希一致，SQLite 完整性通过。恢复的副本未启动；提供商重新认证和浏览器登录迁移未测试。Docker、Tailscale、ARM64、其他提供商和容量限制不在本次运行范围内。

## 服务器上运行什么，不运行什么

完全在服务器上运行：所有引擎 CLI（Claude Code、Codex、Grok、自定义 ACP 引擎）、对话、房间、机器人间协调、例程、已连接应用和自定义 MCP 服务器、Webhook、Company Brain、云端或容器计算机上的计算机使用、文本转语音，以及 Web UI（服务器自行提供）。

需要桌面应用：听写和控制服务器自身的桌面。安装浏览器引擎后（见下文），机器人也可在服务器上浏览。

## 开始之前

1. **一台服务器。** 任意 Linux VPS：Ubuntu 22.04 或 24.04，2 核 CPU 和 4 GB 内存足够。你需要 `sudo` 或 root。Hetzner、DigitalOcean、Lightsail、柜子里的 Mac mini：都可以。
2. **Node 24 或更新版本**，用于路径 A 和 C（`node --version`）。在 Ubuntu 上：

   ```sh
   curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash - && sudo apt-get install -y nodejs
   ```

   路径 B 改用 Docker，主机上无需 Node。
3. **你的引擎账户。** 机器人运行在你笔记本电脑上使用的相同引擎 CLI 上，使用你的订阅。在服务器上以相同方式登录一次。

以下所有操作都连接到服务器：

```sh
ssh root@YOUR_SERVER_IP
```

路径 A 和 C 请使用同一个非特权 Linux 账户进行设置和运行服务。以下示例使用 `maus`，主目录为 `/home/maus`。从管理员 shell 创建（若尚不存在），然后切换过去：

```sh
sudo useradd --create-home --shell /bin/bash maus
sudo -iu maus
```

登录 shell 也会切换到 `/home/maus`；在该目录下运行设置，而非 `/root`。在此账户中运行下方的 `npx` 命令和引擎登录。系统包和 systemd 请保留单独的管理员 shell。如果你已以 root 登录，请重新以 `maus` 登录；这些账户的主目录和凭据不同。

## 路径 A：一条命令获得公网地址

无需域名、无需代理、无需开放端口。服务器通过 Cloudflare 隧道获得类似 `https://c-7f3a9c.openmausbot.com` 的地址；只有经过隧道的流量才能到达，且该流量仍需配对。

```sh
npx openmausbot setup          # 一次：选择 AI 访问方式、连接并选择模型
npx openmausbot login          # 一次：邮箱验证码登录本机并保留其地址
npx openmausbot serve --tunnel # 在此运行服务器并打印带二维码的配对链接
```

`setup` 连接 AI 提供商；它与 OpenMausBot 账户是分开的。通过 SSH 使用 Codex 的设备码选项，或输入隐藏的 API 密钥进行仅聊天连接。之后可添加更多引擎。参见 [CLI 设置](cli-onboarding.md)。

`login` 询问你的邮箱，发送 8 位验证码，并打印为本机保留的地址。`serve --tunnel` 首次运行时下载 `cloudflared`（固定版本并校验摘要，存入 `~/.openmausbot`），启动服务器，连接隧道，几秒后打印 `tunnel: live at https://…`。保持运行；服务化参见"保持运行"。

账户凭据位于 `~/.openmausbot/tunnel-account.json`，仅你的用户可读。`npx openmausbot logout` 释放该地址。

跳到"安装并登录引擎"。

## 路径 B：使用 Docker 和自己的域名

一个容器运行服务器，加 Caddy 在 `https://maus.example.com` 提供 HTTPS。

1. **将域名指向服务器。** 在 DNS 提供商处添加 **A 记录**（名称 `maus`，值为服务器公网 IP）。几分钟后 `ping maus.example.com` 应返回该 IP。80 和 443 端口必须开放；大多数提供商默认开放。
2. **安装 Docker：**

   ```sh
   curl -fsSL https://get.docker.com | sh
   docker compose version   # 打印版本；若提示 "command not found"，注销后重新登录
   ```

3. **获取部署文件并设置域名：**

   ```sh
   git clone https://github.com/milind-soni/OpenMausBot && cd OpenMausBot/deploy
   cp .env.example .env
   nano .env                # DOMAIN=maus.example.com ; ENGINES=@anthropic-ai/claude-code @openai/codex
   ```

   `ENGINES` 列出打包进镜像的引擎 CLI，以空格分隔。之后添加引擎需修改并重新构建。

4. **启动：**

   ```sh
   docker compose pull omb && docker compose up -d
   docker compose ps        # omb "healthy"，caddy "running"
   ```

   Caddy 自行申请证书；稍等片刻。然后 `https://maus.example.com` 显示要求配对码的页面。这是正常的：未配对前一切不可用。

此路径下，下方所有 `npx openmausbot …` 命令改为在容器内运行：

```sh
docker compose exec omb node dist-server/openmausbot.js pair --label "My MacBook"
```

## 路径 C：仅你的 Tailscale 网络

从管理员 shell 在服务器上安装 Tailscale 并登录（`curl -fsSL https://tailscale.com/install.sh | sh && sudo tailscale up`）。在管理控制台为 tailnet 启用 HTTPS 证书一次（DNS → HTTPS Certificates），然后从 `maus` shell 运行：

```sh
npx openmausbot serve --tailscale
```

Tailscale 使用自己的证书终止 HTTPS，配对链接使用服务器的 MagicDNS 名称（`https://maus.tail1234.ts.net`）。仅你的 tailnet 上的设备可访问，对于能运行工具的服务器来说这是非常好的特性。

## 给机器人一个浏览器（可选）

机器人可在服务器上浏览。Docker（路径 B）镜像中已包含引擎和 Chrome。路径 A 和 C 需从管理员 shell 安装一次 Linux 系统库：

```sh
sudo -H npx --yes openmausbot browser install --with-deps
```

然后在服务账户主目录中安装浏览器，从 `maus` shell 运行：

```sh
npx --yes openmausbot browser install
```

管理员的浏览器下载位于不同主目录；它不会安装服务账户的浏览器。两条命令都必须成功完成。

在 Ubuntu 24.04 上，AppArmor 可能还需要[管理员沙箱设置](#ubuntu-2404-browser-sandbox)。一键安装器下载浏览器；它不授予这些操作系统权限。

然后在设置 → 计算机下开启浏览器，并按机器人开启。每个机器人获得独立的隔离会话，其登录在重启后保持。

## 安装并登录引擎

npm 的 OpenMausBot 包不安装模型引擎 CLI。路径 A 和 C 需在服务账户中安装所用引擎并登录。例如，从 `maus` shell 安装 Claude：

```sh
npm install --global --prefix "$HOME/.local" @anthropic-ai/claude-code
export PATH="$HOME/.local/bin:$PATH"
claude
```

Codex 使用 npm 包 `@openai/codex`，运行 `codex login --device-auth` 从无头服务器登录。在浏览器中完成 CLI 的账户流程。仅对你使用的引擎重复。路径 B 在容器内运行已安装的 CLI，例如 `docker compose exec omb claude`。

引擎登录属于服务用户的主目录（例如 `~/.codex` 和 `~/.claude`），与 OpenMausBot 的 `~/.openmausbot` 分开。重启或升级时保留该主目录。下方 systemd 示例在 PATH 中包含 `~/.local/bin`。

## 配对第一台设备

`serve` 启动时已打印配对链接和二维码。之后为另一台设备配对：

```sh
npx openmausbot pair --label "Kitchen iPad"
```

```
pairing code:  RR8Y-BLR6-H939
expires:       10:59:45 AM (single use)
open or scan:  https://c-7f3a9c.openmausbot.com/pair#code=RR8Y-BLR6-H939
```

- **浏览器：** 打开链接。验证码已填入；按 **Connect**。该浏览器配对 30 天，使用时按上述规则续期。
- **桌面应用：** 复制链接，然后在应用的 **Server** 菜单中选择 **Add Server from Copied Pairing Link…**。菜单可在本机和已添加的每台服务器间切换。
- **手机：** 从 iOS 应用的配对屏幕扫描二维码，或将整个链接粘贴到其地址栏。手机可聊天、审批和阅读；创建机器人、更改模型和连接应用仍在服务器 UI 中由你操作。

须知：验证码**一次性**使用，**五分钟**后过期；链接仅在该地址有效（手动输入验证码：打开 `…/pair` 并输入）；同一地址一分钟内十次错误验证码会暂停该地址配对一分钟；`--client` 为可聊天和审批但不能更改设置或配对其他设备的设备生成验证码。

## 管理设备

每台已配对设备都是一个会话：

```sh
npx openmausbot sessions              # id、设备、范围、最后活动、过期时间
npx openmausbot sessions revoke ID    # 立即登出该设备并关闭其流
npx openmausbot status                # 服务器对自身的报告
```

## 保持运行

`npx openmausbot serve` 是普通前台进程。对于 systemd（路径 A 和 C），先从 `maus` shell 安装选定版本。将 `X.Y.Z` 替换为你想运行的已发布版本：

```sh
npm install --global --prefix "$HOME/.local" openmausbot@X.Y.Z
```

启用服务前用 Ctrl-C 停止前台服务器。从管理员 shell 将以下内容保存为 `/etc/systemd/system/openmausbot.service`：

```ini
# /etc/systemd/system/openmausbot.service
[Unit]
Description=OpenMausBot server
After=network-online.target

[Service]
User=maus
WorkingDirectory=/home/maus
Environment=HOME=/home/maus
Environment=PATH=/home/maus/.local/bin:/usr/local/bin:/usr/bin:/bin
ExecStart=/home/maus/.local/bin/openmausbot serve --tunnel --no-pair
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```sh
sudo systemctl daemon-reload && sudo systemctl enable --now openmausbot
journalctl -u openmausbot -f            # 服务器日志，包括 "tunnel: live at …"
```

路径 C 用 `--tailscale` 代替 `--tunnel`。`--no-pair` 跳过每次重启时打印验证码；需要时用 `npx openmausbot pair` 生成。Docker（路径 B）自行重启（`restart: unless-stopped`）。

若账户、主目录和 PATH 不同请相应调整；该 PATH 上必须有 Node 24。服务直接运行已安装的 CLI，因此重启使用同一版本，不会出现 npm 安装提示或隐式升级。

## 更新

- **使用上述服务的路径 A 和 C：** 选择新的已发布版本，然后从管理员 shell 运行以下命令，替换 `X.Y.Z`：

  ```sh
  sudo systemctl stop openmausbot
  sudo -iu maus npm install --global --prefix /home/maus/.local openmausbot@X.Y.Z
  sudo systemctl start openmausbot
  ```

  启动前确认安装成功。仅重启服务不会更新已安装的包。前台 `npx` 用法请指定所需版本：`npx --yes openmausbot@X.Y.Z serve --tunnel`。
- **路径 B：** `cd OpenMausBot/deploy && docker compose pull omb && docker compose up -d`。

例程和排队的工作在重启后保留。当时正在运行的队友轮次会在下次启动时重跑一次，并被告知检查已完成的部分，请求它的机器人也会收到通知。用户自己的轮次和正在运行的例程不会重跑：对话会显示被中断。尽量在两次运行之间更新。

## 备份

复制 SQLite 数据库前停止服务器：前台进程用 Ctrl-C，上述服务从管理员 shell 用 `sudo systemctl stop openmausbot`。停止仍在写入你要备份文件的任何引擎进程和托管桌面。

路径 A 和 C 备份应用状态（机器人、对话、例程、隧道凭据和已配对会话）。从服务账户的 shell 运行：

```sh
umask 077
backup_dir=$(mktemp -d "$PWD/openmausbot-backup.XXXXXX")
tar czf "$backup_dir/openmausbot-data.tgz" -C "$HOME" .openmausbot
```

完整备份还需要引擎凭据/配置路径，如 `~/.codex`、`~/.claude` 和 `~/.claude.json`，以及使用时的 `~/.agent-browser` 下的浏览器会话状态。包含已安装 CLI 存在的路径、任何配置的主目录/数据目录覆盖，以及应用目录外的工作区。这些不在上述命令中。

路径 B 的整个 `/data` 卷包含容器的 CLI 主目录。从 `OpenMausBot/deploy` 归档前停止应用；`deploy_data` 是默认卷名，若更改了 Compose 项目请使用实际卷名：

```sh
docker compose stop omb
backup_dir=$(mktemp -d "$PWD/openmausbot-backup.XXXXXX")
docker run --rm -v deploy_data:/data:ro -v "$backup_dir":/b alpine sh -c 'umask 077; tar czf /b/openmausbot-data.tgz -C /data .'
```

每条命令在当前目录创建新的私有文件夹，因此不会以更宽松的权限覆盖旧归档。将归档私密保存在该文件夹中并放到另一台机器。恢复时停止服务器，使用相同路径和原始所有权。备份或恢复后，用 `sudo systemctl start openmausbot` 或 `docker compose start omb` 启动服务。

## 设置所依赖的规则

在服务器前放置任何其他东西之前请阅读。

- 服务器只监听回环地址（`127.0.0.1:8799`）。切勿自行发布该端口。隧道、Caddy 和 Tailscale 都从同一台机器访问它。
- 经代理或隧道到达的请求被视为远程，无论携带什么头都需要会话。你自己的代理（nginx、Traefik、Cloudflare Tunnel）必须转发真实的 `Host` 并添加 `X-Forwarded-For` 和 `X-Forwarded-Proto`，不得缓冲事件流，且**不得**将 `Host` 重写为 `127.0.0.1`。
- 配对即登录。想要再加一道墙？路径 B 的 `Caddyfile` 有注释掉的 `basic_auth` 块用于共享密码。
- 会话 Cookie 标记为 `Secure`；不要在公网以纯 HTTP 提供此服务。
- 陌生人唯一能读到的是 `/.well-known/openmausbot/environment`（服务器 id、标签、版本、能力）和 `/api/health`（仅应用名）。其他一切返回"配对此设备"。

## 故障排查

**`serve --tunnel` 提示 "no account on this machine yet"。** 先在本机运行 `npx openmausbot login`；凭据按机器存储。

**隧道一直显示 "retrying"。** 服务器正在运行且本地可用；公网跳转尚未验证。等一分钟（Cloudflare 对新地址需要一点时间），然后查看 `journalctl`/终端中的原因。若始终无法建立，`