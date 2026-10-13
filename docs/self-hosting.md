# 自托管 OpenMausBot 服务器

在一台常开的 Linux 机器（VPS、家用服务器、壁橱里的 Mac mini）上运行 harness 服务器，让浏览器、桌面应用或手机与之配对。npm CLI 支持托管公网隧道、Tailscale 或你自己的代理。

> **安全优先：** 默认情况下，自托管服务器将回环地址视为所有者——任何能访问 `127.0.0.1:8799` 的进程都拥有完全控制权，包括机器人可用的 shell。**切勿直接暴露该端口，切勿将其绑定到公网接口。** 通过 SSH 隧道、你信任的私有网络或下方经过认证的远程路径访问它。通过托管隧道或正确配置的代理发出的请求需要已配对的会话。如果多人共用一个服务器，请阅读下方的[回环信任](#回环信任所有者或服务)。

如果你还没有服务器，请逐步操作：[在 VPS 上部署 OpenMausBot](deploy-vps.md) 介绍了三种接入方式（公网地址、自有域名、Tailscale）、引擎登录、配对、保持运行、更新和备份。本页是其背后的参考文档。

## 无头模式下可用的功能（以及不可用的功能）

完全在服务器上运行：

- 每个引擎 CLI（Claude、Codex、Grok、自定义 ACP 引擎——**在服务器上**安装并登录）
- 聊天、房间、机器人间协调、例程（即使全球所有笔记本电脑都关机它们也照常运行——这正是重点）
- 已连接的应用 / 自定义 MCP 服务器、Webhook、Company Brain
- **云或容器计算机**上的计算机使用（机器人的计算机本来就运行在服务器端）
- 文本转语音（需要密钥）、Web UI（服务器自行提供）

- 机器人的浏览器，前提是引擎已安装在服务器上（`npx openmausbot browser install`，Docker 镜像中无需操作，已内置）：每个机器人获得自己隔离的持久会话。从应用中实时观看是下一步（docs/plans/browser-engine.md）。

目前仅限桌面端（需要 Mac/Linux 应用）：

- 听写/语音、控制宿主机桌面

## 最快方式：用 Node 一条命令

在任何装有 Node 24 或更高版本的机器上（VPS、Mac mini、树莓派）：

```sh
npx openmausbot start
```

首次启动会要求你选择 AI 访问方式、连接账户或 API 密钥，并为新机器人选择默认模型。已有的登录可以复用；Codex 还提供适用于 SSH 的设备代码登录。API 密钥连接目前支持聊天，不支持代理工具或计算机使用。[设置指南](cli-onboarding.md)解释了各项选择、密钥存储以及如何安全地重新运行设置。

随后它会启动服务器，数据保存在 `~/.openmausbot`。如果你选择手机访问，它会在检查 HTTPS 连接后才打印配对链接和二维码。选择 **Skip for now** 则工作区仅限本地，不创建配对邀请。使用 `npx openmausbot setup` 仅配置不启动，或使用 `npx openmausbot serve` 以现有配置非交互式启动（用于服务和脚本）。

npm 包不包含引擎 CLI（`claude`、`codex` 等）；设置过程可以提议在本机安装并登录受支持的引擎。请以同一个非特权操作系统用户运行设置、引擎认证和服务器。引擎凭据存放在该用户的 CLI 专属目录中，并非全部位于 `.openmausbot` 下。

对于 Linux 服务，[VPS 指南](deploy-vps.md#before-you-start)展示了账户设置、引擎安装和浏览器依赖安装。以管理员身份安装浏览器库后，还需以服务用户身份运行 `npx openmausbot browser install`，以便该用户的浏览器就位。有三种方式让服务器可从其他地方访问：

- **在你的 Tailscale 网络上，无需域名：**
  `npx openmausbot serve --tailscale`。Tailscale 使用自己的证书终止 HTTPS，链接使用本机的 MagicDNS 名称，因此只有你 tailnet 上的设备能访问。需要 Tailscale 已登录并为 tailnet 启用 HTTPS 证书（管理控制台 → DNS）。
- **公网地址，无需域名、无需代理、无需开放端口：**

  ```sh
  npx openmausbot login          # 一次：通过邮件验证码登录本机
  npx openmausbot serve --tunnel
  ```

  `login` 为本机预留一个类似 `https://c-….openmausbot.com` 的地址；`serve --tunnel` 通过 Cloudflare 隧道（桌面应用为其伴侣使用的同一条隧道）连接它，并打印该地址下的配对链接。首次运行会下载 `cloudflared`（固定版本和摘要）到数据目录。只有经过隧道的流量才能到达服务器，且仍需配对：隧道落在服务器视为"经过代理"的独立监听器上，绝不会被视为所有者。`npx openmausbot logout` 释放该地址。账户凭据存放在 `~/.openmausbot/tunnel-account.json`（权限 0600）。
  从集群或容器中启动，没人能输入邮件验证码？将 `OMB_INSTALLATION_CREDENTIAL` 设为集群发放的安装凭据并跳过 `login`：地址和连接器令牌在每次启动时获取，不写入磁盘。凭据被拒绝时会以明确消息停止启动，而非本地提供服务。
- **你自己的域名，仍是一条命令：**

  ```sh
  npx openmausbot serve --domain maus.example.com
  ```

  将该域名的 A 记录指向本机并开放 80 和 443 端口。服务器会下载一次固定版本的 Caddy 到数据目录，写入与 Docker 栈相同的 Caddyfile，作为子进程运行，Caddy 从 Let's Encrypt 获取并续期证书。在 Linux 上，普通用户绑定 80 和 443 端口需要一项权限授予；当 Caddy 报告拒绝时，`serve` 会打印需运行一次的准确 `setcap` 命令。
- **在你自己的代理或域名后面：** `npx openmausbot serve --public-url https://maus.example.com`，代理规则见"在前面放置代理"。

之后：`npx openmausbot pair --label "Kitchen iPad"` 为另一台设备配对（`--client` 用于可聊天但不能更改设置的设备），`npx openmausbot sessions` 查看或撤销它们。`openmausbot serve` 是普通的前台进程。无人值守使用时，请遵循 [systemd 示例](deploy-vps.md#keep-it-running)，它安装选定的发布版本并直接运行其二进制文件。重启该服务不会隐式下载新版本。

## 从浏览器连接 ChatGPT

以 Full access 配对的浏览器无需打开终端即可连接已安装的 Codex CLI：**Settings → Engines → Codex → Connect ChatGPT**。OMB 在服务器上启动 `codex login --device-auth` 并显示一次性代码。选择 **Open ChatGPT sign-in**，在 OpenAI 页面输入代码，用你自己的账户完成登录。OMB 检查完成情况并刷新模型列表。你可以取消或在过期后请求新代码。

服务器仍需安装 Codex，并以与 OMB 相同的操作系统用户运行。你的密码绝不经过 OMB；Codex 将凭据存储在服务器上。请将服务器访问和备份视为敏感信息。设备代码登录可能需要在 ChatGPT 安全设置中或由工作区管理员启用；参见 [OpenAI 的无头认证指南](https://learn.chatgpt.com/docs/auth#login-on-headless-devices)。订阅限制仍然适用。此浏览器流程目前仅适用于 Codex；其他提供商保留其现有的登录方式。

连接后，当 Codex 能报告时，Settings 会显示账户邮箱。要切换账户，在该行下打开 **Manage account and sign-in** 并选择 **Sign out of ChatGPT**：OMB 以同一用户在服务器上运行 `codex logout` 并用 `codex login status` 确认。新的 ChatGPT 任务需要已连接的账户。切换前请停止正在运行的 Codex 任务：登出不会取消进行中的工作。API 密钥登录不会被此 ChatGPT 专属操作移除。另一个浏览器仍在完成的登录绝不会被拉走；请先完成或取消它。

## 在 Settings 中连接自定义域名

对于自托管服务器，从以 Full access 配对的浏览器打开 **Settings → Remote access → Connect your domain**。这是地址设置和验证流程，不是 DNS 或托管服务。输入域名后，会看到一条紧凑的 DNS 记录，带有 **Type**、**Name / Host** 和 **Value / IP** 的复制按钮。显示完整主机名；已自动附加 DNS 区域的提供商只需相对名称（或区域根部的 `@`）。服务器/代理说明在 **Advanced server setup** 下。

IP 来自本服务器的网络接口，绝不来自浏览器、隧道主机名或 IP 回显服务。仅显示单个明确的公网 IPv4。对于容器/NAT 或具有多个公网地址的主机，管理员可将 `OMB_PUBLIC_IPV4` 设为 HTTPS 代理的公网 IPv4 并重启 OMB。这是显示提示，不是可达性证明；验证仍会检查 HTTPS 和工作区身份。如果 IP 缺失或无效，UI 会请求管理员帮助，而不是编造 DNS 值。

配置连接：

1. 将所选名称的 DNS **A** 记录指向服务器的公网 IPv4 地址。仅当 IPv6 路由到同一服务器时才添加 **AAAA**。
2. 使用 Caddy 等反向代理配置 HTTPS。Settings 示例使用你实际的应用和 Webhook 端口；[提供的 Caddyfile](../deploy/Caddyfile) 是参考。保持 OMB 监听回环，转发原始 Host 和代理头，并保持事件流不缓冲。Caddy 需要入站端口 80/443 用于其常规证书设置。对于容器，请遵循下方的 Docker 配方，以便 Caddy 能访问回环监听器。
3. 输入 `bots.yourcompany.com`（或其裸 `https://` 源）并选择 **Connect domain**。OMB 在保存前检查该域名下的 HTTPS 和工作区身份。域名不正确则现有地址保持不变。

保存的自定义地址在**新的服务器配对链接**中优先于服务器配置的公网地址。移除它会恢复该回退地址（如果有）；两种操作都不改变 DNS、代理、机器人、对话或现有会话。不同的浏览器源需要自己的配对，因此在新标签页可用前请保持原标签页打开。该设置不改变 `OMB_WEBHOOK_PUBLIC_URL`、现有 Webhook URL 或桌面伴侣的托管连接。此功能用于自托管服务器，而非桌面应用的托管手机端点。

## Docker（在你自己的域名上使用 HTTPS）

对于运行服务器、Caddy 和每机器人桌面的单个无根 Podman 引擎，请参阅可选的 [Podman 全栈配方](../deploy/podman/README.md)，适用于 Windows/WSL2 和 Linux x64。它与下方的 Docker 部署相互独立。

对于本地 Docker Desktop 或无公网域名的私有 Tailscale 访问，请使用[本地 Compose 设置](../deploy/local/README.md)。它默认使用 `http://localhost:8080` 并支持可选的 `.env` 覆盖。

一个租户 = 一个服务器容器加一个用于 HTTPS 的 Caddy。
要求：带 Compose 的 Docker、指向该机器的 DNS 名称、开放的 80/443 端口。

```sh
git clone https://github.com/milind-soni/OpenMausBot && cd OpenMausBot/deploy
cp .env.example .env            # 设置 DOMAIN
docker compose pull omb && docker compose up -d
```

这使用 CI 在每次 `main` 推送时发布的镜像（`ghcr.io/milind-soni/openmausbot`，标签为 `latest`、`sha-…` 和 `v…`）。要从你的检出构建：`docker compose up -d --build`。

然后在**容器内**登录引擎 CLI（其登录信息位于 `data` 卷上，因此重启和镜像升级后依然保留），并为第一台设备生成配对码：

```sh
docker compose exec omb claude                       # 你在 ENGINES 中列出的每个 CLI
docker compose exec omb node dist-server/openmausbot.js pair # 打印代码、链接和二维码
```

在浏览器中打开链接（`https://<DOMAIN>/pair#code=…`）即完成配对；会话是什么见"从你的计算机使用它"。Webhook URL（`https://<DOMAIN>/hooks/wh_…`）无需会话即可工作，由于该栈设置了 `OMB_WEBHOOK_PUBLIC_URL`，这是应用在新 Hook 上打印的基础地址。

该栈所做的工作，供你参考调整：

- [`Dockerfile`](../Dockerfile) 构建 UI 和自包含的服务器包，并以非特权用户运行，`HOME=/data`。`--build-arg ENGINES="…"`（或 `.env` 中的 `ENGINES=`）将引擎 CLI 烘焙进镜像。
- [`deploy/docker-compose.yml`](../deploy/docker-compose.yml) 在**服务器的网络命名空间**中运行 Caddy，因此 Caddy 通过 `127.0.0.1` 访问服务器，服务器绝不绑定任何公网内容。
- [`deploy/Caddyfile`](../deploy/Caddyfile) 终止 TLS 并转发真实的 `Host` 以及 `X-Forwarded-For`/`X-Forwarded-Proto`；服务器自身的配对就是登录。共享密码的 `basic_auth` 块已注释掉，如果你想在配对前再加一道墙。

使用 `docker compose pull omb && docker compose up -d`（或 `git pull && docker compose up -d --build`）升级。状态（聊天、例程、引擎登录、已配对会话）位于 `data` 卷上；请备份它。

发布的镜像会在新的 Stable 版本发布时安装当前的 Chrome for Testing。从源码构建会保留首次安装的 Chrome（以及任何 `ENGINES`），因为这些层位于应用之下，在代码变更间保持缓存。要获取当前的版本，请用新标签重新构建：

```sh
docker compose build --build-arg CHROME_CACHE_TAG=$(date +%F) omb && docker compose up -d
```

## 从源码运行

要求：Node 24+、pnpm，以及至少一个在服务器上安装并登录的代理 CLI。

```sh
git clone https://github.com/milind-soni/OpenMausBot && cd OpenMausBot
pnpm install

# 选择数据存放位置并启动服务器
OMB_DATA_DIR="$HOME/.openmausbot" OMB_PORT=8799 \
  node --experimental-strip-types server/index.ts
```

要持久运行，让 CLI 为你编写服务：

```sh
npx openmausbot service install --domain maus.example.com   # 或 --tunnel、--tailscale，或不带参数
```

它会渲染一个 systemd 单元（Linux）或 launchd 代理（macOS），以你的选项运行相同的 `openmausbot serve …`，停止时重启，对于 `--domain`，还会授予该单元无需 root 即可绑定 80 和 443 端口的能力。文件写在你数据旁边，并打印安装和启动它的两条命令（在 Linux 上需要 `sudo`）。`openmausbot service uninstall` 打印反向操作。请先永久安装该包（`npm install -g openmausbot`）：服务不能指向 npm 可能清理的 `npx` 缓存。

引擎 CLI 从服务用户的主目录读取登录信息：在依赖无人值守运行的例程之前，请从 Settings → Engines（见下）或以该用户在终端中登录它们。

## 无需终端安装引擎

安装程序为 npm 包的引擎（Claude Code、Codex、OpenCode、MiniMax、pi）可在 npm 位于服务器 PATH 时从 **Settings → Engines** 安装和更新。OMB 以自身用户运行 `npm install -g` 到 `<data dir>/tools/npm`，因此无需 sudo，也不触碰全局前缀；该文件夹在引擎 PATH 上优先于其他所有内容，因此 OMB 安装的副本就是机器人运行的副本。包名来自引擎自身的安装描述符，绝不来自浏览器。通过 `curl | bash` 脚本安装的引擎仍需在服务器上有该命令。

## 提供商密钥，按 token 计费

**Settings → Connections → Model providers** 接受整个工作区运行所需的密钥，适合宁愿按 token 付费而不愿每个用户登录的人。密钥只写：页面显示是否已连接以及一个 **Test** 按钮，从服务器向提供商发出一次只读请求。

- **Anthropic API 密钥**：保存后，每个 Claude 机器人都在其上运行，Claude Code 向用量账本报告每轮的真实成本。无人需要登录，Settings → Engines 显示"workspace API key"而非个人。移除密钥即恢复个人登录。服务器自身的 `ANTHROPIC_API_KEY` 环境变量被有意忽略；请使用页面、`config.json` 或 `OMB_ANTHROPIC_API_KEY`。
- **OpenAI 兼容 API 密钥和基础 URL**：默认 OpenRouter，或 Groq、Together、网关，或 OpenAI 自身的 `https://api.openai.com/v1`。这驱动 OpenAI 兼容引擎。Codex 按设计没有密钥路径，始终使用个人 ChatGPT 登录。
- **xAI API 密钥**：Grok API 引擎和 xAI 图像生成。

保存的密钥只发往其自己的引擎，且只发往工作区的端点：`config.json` 中带有自己基础 URL 或自己密钥（路由器或代理）的引擎实例绝不会收到它。

**OpenCode 是"服务器自身环境被忽略"的例外。** 与终端中的 `opencode` 一样，它从环境读取提供商密钥（`ANTHROPIC_API_KEY`、`OPENAI_API_KEY`、`GEMINI_API_KEY`、`GOOGLE_API_KEY`、`KIMI_API_KEY`、`MOONSHOT_API_KEY`、`MINIMAX_API_KEY`）并提供这些提供商的模型，计入该密钥。它仅在服务器环境属于个人时如此：绝不在 MausBot Cloud、托管团队工作区、组织管理的桌面，或登录列表允许他人进入的服务器（用户、多个管理员或整个域名）上。通过 `opencode auth login` 添加的提供商在任何地方都有效。

## 一台服务器上的多个客户端工作区

`openmausbot fleet` 在单个 Linux 服务器上为每个客户端运行一个工作区，各自作为独立的 OS