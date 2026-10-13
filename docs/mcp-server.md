# OpenMausBot MCP 服务器

OpenMausBot 桌面应用内置一个本地 stdio MCP 服务器。桌面应用及其 harness 运行时，其他 MCP 客户端可通过它协调你的 OpenMausBot 团队。

## 功能

- 列出机器人和频道，包括其活动任务和当前动态；
- 读取有界对话记录分页，搜索本地对话记录，不返回截图像素；
- 创建并安全编辑机器人配置、频道和独立任务对话；
- 向机器人或频道发送工作，等待对话结束或需要帮助，并中断其活动轮次；
- 列出已配置的模型实例，将空闲机器人切换到某个确切可用的模型。

v1 服务器有意不支持审批请求、记住权限授予、删除数据、导入团队、更改凭据或控制计算机/VM 生命周期。这些操作仍保留在面向用户的应用中。

## 从源码检出运行

启动 OpenMausBot，然后配置 MCP 客户端运行：

```json
{
  "mcpServers": {
    "openmausbot": {
      "command": "pnpm",
      "args": ["--dir", "/absolute/path/to/OpenMausBot", "mcp"]
    }
  }
}
```

打包的桌面版本中，创建、更改、发送、切换、中断或运行任何内容的工具都需要配对会话。只读工具在回环地址上无需配对即可使用。托管或共享服务器若启用了服务回环信任（[自托管](self-hosting.md#loopback-trust-owner-or-service)），几乎所有工具（包括读取）都需要会话。授权外部 MCP 客户端：

1. 在 OpenMausBot 中打开 **Settings → Phone → Set up a phone**，显示一次性配对码。
2. 在本地交换配对码（去掉显示配对码中的空格）：

   ```sh
   curl -X POST http://127.0.0.1:8799/api/auth/pair \
     -H 'content-type: application/json' \
     --data '{"code":"YOUR-CODE","label":"Local MCP client"}'
   ```

   如果桌面选择了备用端口，将 `8799` 替换为 Settings 中显示的端口。复制返回的 `token`；配对码仅可使用一次。
3. 在 MCP 配置中固定同一端口和 token：

   ```json
   "env": {
     "ELECTRON_RUN_AS_NODE": "1",
     "OMB_PORT": "8799",
     "OPENMAUSBOT_TOKEN": "omb_sess_..."
   }
   ```

配对后的客户端会与手机一起显示在 **Settings → Phone** 下，并可在那里撤销。不要分享 token，也不要将其提交到仓库。

## 从已安装的桌面应用运行

发布版本捆绑了 `server/mcp-server.js`，可用 Electron 内置的 Node 运行时运行，因此用户无需安装 Node.js 或 pnpm。

macOS 示例：

```json
{
  "mcpServers": {
    "openmausbot": {
      "command": "/Applications/OpenMausBot.app/Contents/MacOS/OpenMausBot",
      "args": ["/Applications/OpenMausBot.app/Contents/Resources/server/mcp-server.js"],
      "env": { "ELECTRON_RUN_AS_NODE": "1" }
    }
  }
}
```

在 Windows 上，将已安装的 `OpenMausBot.exe` 作为 `command`，相邻的 `resources\\server\\mcp-server.js` 作为参数，并使用相同的 `ELECTRON_RUN_AS_NODE=1` 环境变量值。通常的按用户安装路径为 `%LOCALAPPDATA%\\Programs\\OpenMausBot`。

在 Ubuntu `.deb` 安装中，可执行文件通常为 `/opt/OpenMausBot/openmausbot`，脚本为 `/opt/OpenMausBot/resources/server/mcp-server.js`。使用相同的环境变量值。

## 连接发现

无配置时，MCP 进程会探测 OpenMausBot 的三个桌面端口（`8799`、`18799` 和 `28799`），仅接受标识自身为 OpenMausBot 的健康响应。这处理了桌面在端口 8799 已被其他本地进程占用时的正常回退。

设置 `OMB_PORT` 可强制使用一个本地端口，或设置 `OPENMAUSBOT_URL` 使用显式的 HTTP(S) 源。除非 `ALLOW_INSECURE_HTTP=true`，否则拒绝明文远程 HTTP；回环之外应使用 HTTPS。可选的 `OPENMAUSBOT_TOKEN` 作为 bearer token 发送，用于已认证的反向代理。设置 token 时，必须显式指定 `OPENMAUSBOT_URL` 或 `OMB_PORT`，以免在探测无关本地端口时发送凭据。`OPENMAUSBOT_MCP_TIMEOUT_MS` 可设置 HTTP 超时，范围 1,000 到 120,000 毫秒。在打包版本中，如上所述，变更类工具需要 `OPENMAUSBOT_TOKEN`；它不是通用的反向代理密钥。

## 工具

| 用途 | 工具 |
|---|---|
| 检查 | `get_system_health`、`list_bots`、`list_channels`、`get_bot_messages`、`get_channel_messages`、`search_messages`、`list_available_models` |
| 创建与组织 | `create_bot`、`update_bot_profile`、`create_channel`、`update_channel`、`create_task`、`switch_task`、`rename_task` |
| 运行工作 | `send_bot_message`、`send_channel_message`、`wait_for_conversation`、`interrupt_conversation`、`set_bot_model` |

`wait_for_conversation` 返回 `settled`、`needs-user`、`failed`、`stalled` 或 `timed-out`，以及一小段脱敏的对话记录尾部。若某对话中的机器人将工作交给队友（`coordinate_bots`，作业运行时上的分配：docs/runtime-contract.md §9），在这些分配未完成期间该对话不算结束；一旦其结果恢复该对话且该轮次结束，它便结束。工具等待期间会响应 MCP 取消。

## 安全与数据范围

对话记录读取分页，上限 200 条消息。搜索上限 100 条命中。截图像素和权限授予密钥会从 MCP 结果中移除。工具 Schema 拒绝未知字段和格式错误的值，机器人在工作时拒绝更改模型。

harness 本身是本地优先的，通常绑定到回环地址。若通过反向代理暴露它，该代理上的认证和 TLS 属于你部署的安全边界。