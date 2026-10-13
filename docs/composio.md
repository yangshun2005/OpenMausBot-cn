# 通过 Composio 连接应用

OpenMausBot 使用一个 Composio 项目 API 密钥和一个可复用的 Composio Session。该密钥是用户唯一需要提供的 Composio 凭据。Session 启用 Composio 的多账户模式并要求显式选择账户，因此一个 OpenMausBot 安装可以同时保持多个 Slack、Gmail、Calendar 或其他账户连接，而不会静默替换第一个账户。

## 打包的桌面应用

1. 打开 [Composio Dashboard](https://dashboard.composio.dev)。
2. 选择 **Platform**，选择或创建一个项目，然后打开 **Settings → API Keys**。
3. 复制以 `ak_` 开头的项目密钥。
4. 在 OpenMausBot 中，打开 **App Settings → Connections**，将其保存到 **Composio project key**。
5. 打开 **Connected apps**，选择 Gmail、GitHub、Slack 或其他服务。输入唯一标签，如 `work` 或 `personal`，然后在常用浏览器中完成授权。
6. 要为同一应用连接另一个账户，选择 **Add account**，给它一个唯一标签，然后在浏览器中完成第二次授权。

Connected 标签页会分别列出每个账户。**Disconnect** 只撤销该行对应的账户。OpenMausBot 会为每个新账户要求一个标签，并配置 Composio 在多个账户可能运行同一工具时要求显式选择；新的 OAuth 流程绝不会静默成为现有连接的默认账户。

桌面应用在保存前会验证密钥。密钥使用 Electron 基于操作系统的 `safeStorage` 加密；本地 JSON 配置只存储非机密的 Composio 用户和 Session 标识符。

## 受限密钥权限

默认项目 API 密钥无需额外配置即可使用。若使用最小权限的受限密钥，需授予：

- **Sessions：** 读和写
- **Toolkits：** 读
- **Connected accounts：** 读和写

需要连接账户的写权限，以便 **Disconnect** 在移除连接前撤销上游提供方的授权。

## 从源码运行

在服务器环境中设置密钥：

```sh
COMPOSIO_API_KEY=ak_your_project_key pnpm dev:server
```

仅浏览器的开发 UI 也可以将密钥保存到仅所有者可访问的 `~/.openmausbot/config.json` 文件。在无头环境和共享开发机器上，推荐使用环境变量。

OpenMausBot 会为该安装创建一个稳定的随机用户标识符，存储返回的 Session 标识符，并在每次启动时复用该 Session。OpenMausBot 不存储任何 Gmail、GitHub、Slack 或其他提供方的令牌；其连接生命周期由 Composio 管理。

旧版 OpenMausBot 创建的 Session 会就地升级：为同一稳定的 Composio 用户创建多账户 Session。已连接的账户属于该用户，因此现有授权保持可用，同时新 Session 增加了显式的多账户路由。每个工具包最多支持五个可用账户。

## 多个 Google 和 Slack 账户

可以。Gmail、Google Calendar、Google Drive 及其他 Google 工具包均可持有多个带标签的授权，Slack 也可持有多个带标签的工作区/账户授权。账户归属于 OpenMausBot 安装的稳定 Composio 用户，并在 **Connected apps** 中按别名和连接账户 ID 显示。

如果提供方或受限的 Composio 项目策略阻止再次授权，安全的回退方案是使用单独的 OpenMausBot 安装/配置及其自己的 Composio 用户。重新授权同一个单账户 Session 不是安全的变通方法：它可能改变所选授权。不要共享原始提供方令牌，也不要将其放入机器人提示词中。

托管/管理的连接应用代理暴露与自托管项目密钥模式相同的账户感知响应结构和账户专属移除路由；它不会将代理或提供方凭据发送到渲染进程。

## 默认连接不请求的权限

你连接的每个应用使用该应用的 Composio 项目授权。除非你自行设置，否则使用的是 Composio 管理的授权，它只向提供方请求 Composio 的默认权限。某些操作需要更多权限。在 Gmail 中：

- **过滤器、假期自动回复、IMAP 和 POP 设置** 需要 `https://www.googleapis.com/auth/gmail.settings.basic`。
- **转发地址、代发别名和委托人** 需要 `https://www.googleapis.com/auth/gmail.settings.sharing`。

没有该权限时，Google 会以 `403 ACCESS_TOKEN_SCOPE_INSUFFICIENT` 拒绝调用。重新连接账户无济于事：同意屏幕每次都会请求相同的权限。OpenMausBot 会在该拒绝上附加说明，让机器人告诉你需要更改什么，而不是重试。

授予权限：

1. 在 [Composio Dashboard](https://dashboard.composio.dev) 中，打开项目的 **Auth Configs**，为该应用（例如 Gmail）创建一个使用你自己 OAuth 应用凭据的认证配置。
2. 向其添加所需权限，保持启用，并让它对 Tool Router 可用。
3. 在 OpenMausBot 中，打开 **Connected apps** 并重新连接账户。同意屏幕现在会请求新增的权限。

OpenMausBot 会自动使用你项目自己的应用认证配置，替代 Composio 管理的配置。如果同一应用有多个配置，它会使用最近更新的，并跳过已禁用和为 Tool Router 关闭的配置。Google 可能要求你的 OAuth 应用在其同意屏幕上列出新增权限，并在组织外部人员授予前完成验证。

在 MausBot Cloud 方案中，连接的应用通过 OpenMausBot 的托管连接服务运行，而非你自己的 Composio 项目，因此你无法自行添加权限；请联系 OpenMausBot 支持。

## 浏览器授权恢复

Web UI 会在你点击时、请求链接之前预留一个授权标签页。如果浏览器仍然阻止它，请使用 Apps 模态框或连接器卡片中的 **Open authorization page**。**Continue** / **Open again** 会复用待处理的链接，而不是创建另一个账户。链接仅保存在 UI 内存中；十分钟后，重试会请求新链接，这是 [Composio 的连接生命周期](https://docs.composio.dev/kb/guide/platform-connected-accounts)所要求的。

如果该应用的所有现有账户都处于 `INITIALIZING`、`INITIATED` 或 `EXPIRED` 状态，无别名的重试会获得唯一的 `omb-retry-…` 别名。它绝不会删除或替换账户：`EXPIRED` 也可能描述一个曾经活跃的授权。活跃或未知状态仍需要显式的唯一标签，且五个可用账户的限制仍然有效。达到该限制时，请检查账户行，只断开你打算撤销的账户。托管安装需要更新后的代理以及更新后的应用；这不是客户端侧的绕过。

## 渲染器无关的连接清单

桌面、Web 和移动客户端可以通过一个请求加载完整的账户清单：

```http
GET /api/connectors/connected
```

```json
{
  "configured": true,
  "services": {
    "gmail": {
      "connected": true,
      "pending": false,
      "status": "ACTIVE",
      "accounts": [
        { "id": "ca_123", "alias": "work", "status": "ACTIVE" }
      ]
    }
  }
}
```

此操作直接对 Session 工具包状态和用户的连接账户进行游标分页。它将无认证工具包和 Session 选定的账户与完整的多账户清单合并，而不从市场卡片推导服务 slug，因此账户可见性独立于目录排序和分页。如果受限项目密钥能读取 Session 但无法列出原始连接账户，响应会安全回退到 Session 选定和无认证工具包清单，而不是让这些服务显示为未连接。托管代理在 `GET /v1/connectors/connected` 提供相同的行为和响应；本地服务器在未配置连接服务时添加常规的 `configured: false` 空响应。响应仅暴露连接账户 ID、用户提供的别名和生命周期状态——绝不暴露项目密钥、代理令牌、提供方令牌或只写授权字段。

现有的受限 `GET /api/connectors?services=gmail,slack` 操作仍可用于轻量级的 OAuth 后轮询和向后兼容。

## 每个机器人的工具授权

上述工作区级连接说明存在哪些账户。第二个按机器人划分的层级说明每个机器人实际可以调用某个应用的哪些**工具**。在桌面端打开机器人的设置，在 **Connected apps** 下，每个服务可设置为：

- **All tools** — 该服务提供的所有工具（通配授权）。
- **An exact list** — 仅指定的工具，如 GMAIL_SEND_EMAIL，显示为已授权数量。
- **No tools** — 列出该服务但不授予任何工具。

完全没有授权记录的机器人保持旧行为：它能看到的每个已连接应用上的所有工具。分配第一个授权会将机器人切换为精确工具模式——不在列表中的服务不授予任何工具，即使工作区已连接它。

**在你分配之前不会有任何变化。** 升级 OpenMausBot 不会改变任何机器人的访问权限：现有机器人保持全工具默认值，直到有人编辑其授权。清空的授权列表是刻意的，意味着"任何已连接应用上都没有工具"。

**导入的机器人不带任何授权。** 可共享包和团队导入从不携带授权——导入的机器人启动时已连接应用为关闭状态，它之后获得的任何授权由导入的工作区选择。授权也绝不会出现在导出中；只有工作区自己的私有团队备份会保留它们。

授权在调用时对真实工具名强制执行（包括 MULTI_EXECUTE 元工具内的批量调用），提供给模型的工具列表会过滤为已授权集合，每次允许/拒绝都会写入决策日志行。拒绝不会枚举其他已授权内容。配对的手机在其配置文件中只读查看每个机器人的授权（What this bot does → App tools）；授权在电脑上分配，绝不能从手机分配。