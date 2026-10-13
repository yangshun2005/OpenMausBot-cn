# 终端设置

安装 [Node.js](https://nodejs.org/) 24 或更高版本，然后任选一种方式运行 OpenMausBot：

```sh
# 安装一次，之后用短命令：
npm install -g openmausbot
openmausbot
```

或者不全局安装：

```sh
npx openmausbot
```

下次使用同一条命令即可。首次启动会引导你完成设置；之后启动会复用已保存的 AI 连接和手机访问选择。`openmausbot start` 与不带参数的命令相同。如果该工作区已在运行，OpenMausBot 会打开它，而不是再启动一个服务器。

## 首次启动

设置向导使用 Clack。用 ↑/↓ 和 Enter 选择选项。普通终端会显示编号选项。

1. 选择 ChatGPT/Codex、Claude Code 或 API 服务。已支持的现有连接也会列出。
2. 登录提供商，或将 API 密钥粘贴到隐藏提示中。安装缺失的 Codex 或 Claude CLI 前，设置会先询问。Codex 还提供设备码登录，适用于远程终端。API 选项包括 OpenAI、OpenRouter、Groq 及其他 OpenAI 兼容端点。
3. 选择模型并保存。API 连接会请求许可发送一条简短测试消息，提供商可能对此收费。原生 CLI 设置只确认登录，不会用消息测试所选模型。模型访问权限在你发送消息时才会检查。
4. 可选：连接手机，或选择 **暂时跳过**。此步骤前，你的 AI 设置已保存。

随后工作区启动，并在这台电脑的浏览器中打开其本地地址。在 SSH 或无图形桌面的电脑上，请改用合适的连接方式打开打印出的地址。自动打开浏览器只使用本地地址；`--no-open` 可禁用它。

使用机器人期间请保持终端打开。这运行的是前台服务器，不是已安装的后台服务。Ctrl-C 会停止服务器，但不会删除已保存的工作；仅关闭浏览器不会停止它。只有在电脑和服务器运行时，机器人才能继续工作。

Codex 和 Claude Code 支持代理工具。API 密钥连接目前仅支持聊天。ChatGPT 或 Claude 订阅不包含单独计费的 API 用量。更改已保存的默认值**仅影响新机器人**；现有机器人和对话保留其设置。

## 可选的手机访问

手机无法连接这台电脑的 `localhost` 地址。该工具也不监听你的局域网地址。在配置好的 HTTPS 地址通过此工作区检查之前，指南不会生成手机二维码。

选择一种连接方式：

- **托管 HTTPS 地址：** 设置会明确请求许可，通过 Cloudflare 建立公共端点，并可能下载连接器。设备配对保护聊天和设置；配对页面和基本服务器身份仍可公开访问。使用邮件验证码登录 **OpenMausBot 账户**，或复用本机已保存的账户。此账户独立于 ChatGPT、Claude 或 API 提供商账户。OpenMausBot 运行期间连接保持活跃。
- **现有 Tailscale：** 电脑和手机必须已登录同一 tailnet，并启用 HTTPS 证书。设置会在启用向该 tailnet 提供 HTTPS 服务前询问；它不会替你安装或登录 Tailscale。
- **现有 HTTPS 地址（高级）：** 提供你已配置的反向代理源，例如 `https://maus.example.com`。不要粘贴密码、路径、查询参数或配对码。输入地址不会创建代理，也不会打开局域网监听。

连接就绪后：

- **iPhone/iPad：** 用相机扫描二维码打开 Safari。如果已安装 OpenMausBot iOS 应用，用其配对扫描器或在那里粘贴完整链接。
- **Android：** 二维码是应用链接，请在 OpenMausBot 应用内扫描。若改用浏览器，请打开其上方打印的网址并输入代码。

在手机上选择 **连接**。仅扫描不算配对成功。代码是私密的、一次性使用，五分钟后过期。引导式手机配对授予聊天和审批的客户端访问权限，不包括设置或配对管理。

如果工作区启动但 HTTPS 检查失败，本地工作区仍可使用，且不会生成手机代码。修复连接后，在另一个终端运行 `openmausbot pair` 重试。缺少账户、连接器或 Tailscale 前提条件可能导致启动失败；按打印的错误操作，或使用 `openmausbot --local`。若跳过手机访问后想添加，请停止服务器，运行 `openmausbot setup`，然后重新启动。

## 命令

以下示例假设已全局安装；否则请在命令前加 `npx`。

| 命令 | 用途 |
| --- | --- |
| `openmausbot` | 设置一次，之后用已保存的设置启动。 |
| `openmausbot setup` | 重新进行 AI 和可选的手机设置，保存后退出，不启动。这不是重置。 |
| `openmausbot --no-open` | 启动时不打开浏览器。 |
| `openmausbot --local` | 本次启动忽略已保存的远程访问；保留已保存的选择供下次使用。 |
| `openmausbot --no-pair` | 抑制手机设置提示和配对邀请。这**不会**关闭已保存的远程访问；如需关闭请用 `--local`。 |
| `openmausbot pair` | 在已配置的工作区和 HTTPS 连接运行时，创建另一个手机邀请。 |
| `openmausbot sessions` | 列出已配对设备；`openmausbot sessions revoke ID` 注销其中一个。 |
| `openmausbot serve` | 启动时不显示引导提示，也不自动打开浏览器；为服务显式指定远程访问标志。 |
| `openmausbot login` | 为 `--tunnel` 登录 OpenMausBot 账户；这不会登录 AI 提供商，也不会启动隧道。 |

`start` 接受与 `serve` 相同的服务器选项，包括 `--port`、`--data-dir`、`--tailscale`、`--tunnel` 和 `--public-url`。启动或配对时请继续使用你的自定义数据目录和端口：

```sh
openmausbot setup --data-dir /path/to/omb-data --port 8799
openmausbot --data-dir /path/to/omb-data --port 8799
```

设置需要交互式终端。设置完成后，后续启动无需终端。更改设置或访问模式前，请先停止正在运行的服务器：`--local` 不会关闭已运行服务器的远程连接。重新配置无需删除配置、机器人或对话。

## 凭据与取消

API 密钥在输入或粘贴时隐藏。新的 API 连接将密钥以**明文、未加密**形式保存在数据目录的 `config.json` 中，在 Unix 上权限为仅所有者可访问（`0600`）。`tunnel-account.json` 中的托管访问账户凭据同样是明文，在 Unix 上权限为 `0600`。请妥善保管这些文件及其备份。原生提供商登录凭据由提供商自己的 CLI 管理。

AI 设置期间按 Ctrl-C 会放弃未保存的 OMB 更改。已完成的安装和提供商登录仍然保留。后续手机步骤期间按 Ctrl-C 会保留你已保存的 AI 设置，退出且不启动服务器，也不会撤销已完成的账户登录。运行 `openmausbot setup` 继续；无需破坏性重置。

远程访问和后台部署选项，请参阅[自托管](self-hosting.md)和 [VPS 指南](deploy-vps.md)。