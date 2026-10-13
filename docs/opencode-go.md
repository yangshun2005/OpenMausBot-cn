# OpenCode

OpenCode 是可选的 OpenMausBot 引擎。OpenMausBot 通过 ACP stdio 接口运行维护中的 OpenCode CLI，因此会话、流式输出、编码工具、权限请求、MCP 集成、恢复和取消都与其他 ACP 引擎使用相同的运行时。

## 设置

1. 按照 [OpenCode 安装指南](https://opencode.ai/docs/) 安装官方 CLI。
2. 在 OpenCode 应用中连接所需的提供商，运行 `opencode auth login`，或在 OpenMausBot 中通过 设置 → API 密钥 → **其他 OpenCode 提供商的密钥** 保存提供商的密钥。在没有终端的 OMB Cloud 上，请在该处保存密钥。
3. 重启 OpenMausBot。它会自动复用 OpenCode 现有的连接和模型配置。

关于 OpenRouter、Fireworks AI、DeepSeek、Cline 或你自己的提供商，请参阅 [其他模型提供商](../apps/docs/content/docs/providers/model-providers.mdx)。

OpenCode 包含匿名免费模型。Zen、Go、OpenRouter 或其他提供商连接会根据已安装的 CLI 扩展目录。OpenCode API 密钥可选择保存在 设置 → API 密钥 下。它是只写的，仅作为 `OPENCODE_API_KEY` 注入 OpenCode 子进程；不会发送到渲染器、日志、分析、快照、错误消息或命令参数。

OpenCode 其他提供商的密钥放在同一位置的 **其他 OpenCode 提供商的密钥** 下，每个都使用 OpenCode 读取它的环境变量名：Venice 用 `VENICE_API_KEY`，Groq 用 `GROQ_API_KEY`，依此类推。这在桌面端和 My Cloud 上都有效，后者没有终端可以导出它们。每个密钥都是只写的（设置只显示其名称，从不显示密钥），保存在服务器自己的配置中，并且只传递给 OpenCode 进程；保存一个密钥会重新加载引擎，因此 OpenCode 会立即列出该提供商的模型。在 Cloud、托管团队或组织的桌面端上，OpenCode 仍然从不从服务器自己的环境读取提供商密钥；此处保存的密钥是例外。OpenMausBot 为自己保留的名称（`OPENCODE_*`、`OMB_*`，以及它为其他引擎保存的密钥，如 `XAI_API_KEY`）会被拒绝，最多可保存 20 个密钥。

OpenMausBot 不会复制或重写 `auth.json`。OpenCode CLI 仍然是提供商身份验证的所有者，OpenCode 桌面端/TUI 使用的同一个 Zen 或 Go 连接也会被 OpenMausBot 使用。

## 模型

模型选择器针对配置的二进制文件运行 `opencode models --verbose`，并保留 CLI 返回的每个精确 `provider/model` ID。这可能包括 Zen（`opencode/*`）、Go（`opencode-go/*`）、第三方提供商、自定义配置和本地端点。如果发现暂时失败，则使用上次成功的目录，然后是一个小的匿名模型回退。

在每次提示之前，ACP 会收到 `session/set_config_option`，其中 `configId: "model"` 和精确选定的提供商限定模型 ID。

## 测试

普通单元测试和 ACP 协议测试不需要订阅。实时测试必须显式启用，并且绝不能打印凭据或上传带凭据运行的本地协议日志。