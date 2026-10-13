# Cursor Agent CLI

Cursor 是可选的 OpenMausBot 引擎。OpenMausBot 以 ACP stdio 模式（`cursor-agent acp`）运行官方
[`cursor-agent` CLI](https://cursor.com/docs/cli)，因此会话、流式输出、编码工具、权限请求、MCP 集成、
恢复和取消都与其他 ACP 引擎使用相同的运行时。

使用此引擎的 Bot 消耗用户的 Cursor 订阅（或
`CURSOR_API_KEY` / `CURSOR_AUTH_TOKEN`），而非单独的 Anthropic/OpenAI/xAI
密钥。

## 设置

1. 安装 Cursor CLI：

   ```sh
   curl https://cursor.com/install -fsS | bash   # macOS / Linux
   ```

   Windows（原生）：`irm 'https://cursor.com/install?win32=true' | iex`

2. 使用 `cursor-agent login` 登录，或在 Cursor 实例的环境中设置 `CURSOR_API_KEY` / `CURSOR_AUTH_TOKEN`。

3. 确认 `cursor-agent --version` 可用。Cursor 的文档将该命令称为 `agent`；
   安装程序会在其旁边添加 `cursor-agent`，OpenMausBot 运行的就是这个名字，
   因为其他工具也会安装一个 `agent`。该二进制文件默认安装到
   `~/.local/bin`（Windows 上为 `%LOCALAPPDATA%\cursor-agent`）；
   OpenMausBot 已经会查找这两个位置，因此应用打开时安装的 CLI
   无需重启即可被找到。

在找到 `cursor-agent` 可执行文件之前，该引擎一直不可用。缺少登录会显示为未认证，而不会导致整个集群崩溃。

## 模型

选择器从一个小型静态目录开始，并从纯
`cursor-agent models` 输出（`slug - Label`，带有 `(default)` / `(current)` 标记）刷新。
实时 id 会合并到主云端栏（而非本地模型面板）。列表获取失败时保留
最后可用的目录（然后是静态回退），而不是清空该栏。

`--model <id>` 作为全局 CLI 标志在 `acp` 之前传入。当前 CLI
（以 2026.09.28 检查）在 ACP 会话中忽略该标志：每个会话
都以 `default[]`（Auto）启动。所选模型只能通过 ACP
`session/set_model` 生效，该方法只接受会话所公布的参数化 id，例如
`grok-4.7[context=256k,reasoning_effort=high,fast=true]`。
OpenMausBot 将选择器 slug 映射到该列表：`auto` 映射到 `default[]`，普通
slug 映射到其 Standard 变体，`-fast` 映射到 `fast=true`，而 effort
slug（如 `grok-4.7-high-fast`）映射到具有该 effort 的变体。
会话为每个基础模型公布一个变体，因此带有其他 effort 的 slug
（`grok-4.7-medium-fast`）会得到该变体，并在对话中显示一条通知。

如果缺少 `session/set_model`（`-32601`），则保留 argv 固定值
并继续该轮次。如果 Cursor 拒绝该 id（`-32602`），该轮次也会
继续，使用会话启动时的模型，并在对话中显示
一条指明该模型的通知。

## 自主性

为兼容直接驱动嵌入方，实例的 `fullAuto: true`
仅在某一轮次未提供 bot 审批级别时添加 `--force`（CLI 文档中记载的自动批准开关）。OpenMausBot 应用轮次始终提供
一个：**Ask for approval** 和 **Approve for me** 都会在不带
`--force` 的情况下启动 Cursor，然后 OpenMausBot 根据
bot 的当前级别处理其权限请求。

## 此驱动尚未支持的功能

- Cursor ACP 扩展方法（`cursor/ask_question`、`cursor/create_plan`、
  todos/tasks/images）没有专门的 UI。未知的 JSON-RPC 请求
  会以 method-not-found 拒绝，以免 CLI 被阻塞。
- 在 `session/new` 中传入的 MCP 服务器遵循 Cursor 的 ACP 限制；需要时
  优先使用项目或用户 `.cursor/mcp.json`。
- 在生产环境中依赖此功能之前，应在已安装并登录 CLI 的机器上
  运行实时冒烟测试（`cursor-agent login`、`cursor-agent models`、一次真实轮次）。

## 测试

常规单元测试和 ACP 协议测试使用脚本化的假 CLI，不需要
Cursor 订阅。不要打印凭据，也不要上传来自带凭据的实时运行的
原生协议日志。