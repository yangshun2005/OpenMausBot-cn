# 自带引擎

两种零代码方式，让 OpenMausBot 机器人跑在应用未内置的引擎上。
两者都位于 `~/.openmausbot/config.json` 的 `"instances"` 下；编辑后需重启应用（实例条目在启动时读取）。

## 提供商图标

在 **设置 → 引擎** 中，展开某个实例并选择其 **提供商图标**。
内置选项包括 OpenAI、Anthropic、Google Gemini、Microsoft Azure、Amazon Bedrock (AWS)、xAI、DeepSeek、Meta、Mistral AI、Qwen、Moonshot AI、Cohere 和 OpenRouter。你也可以上传 PNG、JPEG 或 WebP 图片，上限 128 KB、1024 × 1024 像素。**重置** 可恢复默认图标。

每个实例拥有独立图标，与其驱动或 API 协议无关。
在设置中所做的更改立即生效，无需重启引擎。
若使用基于文件的配置，请在 `driver` 和 `displayName` 旁添加 `"icon": { "kind": "preset", "preset": "azure" }`。自定义上传以嵌入图片数据形式存储；应用不会拉取远程图标 URL。

## 任意 ACP 代理（你启动的 CLI）

如果某个代理 CLI 通过 stdio 使用 [ACP](https://agentclientprotocol.com) 协议——`fx acp`、Zed 风格的代理服务器、你自己的封装——将 `customAcp` 实例指向它即可：

```json
{
  "instances": {
    "my-agent": {
      "driver": "customAcp",
      "displayName": "My Agent",
      "environment": { "MY_AGENT_TOKEN": "…" },
      "config": { "cli": "my-agent acp" }
    }
  }
}
```

- **`config.cli`** 是完整命令，含参数（`"npx -y some-agent acp"` 也可用）。你也可以在应用中设置：设置 → 引擎 → 实例所在行的 *设置 CLI…*。未配置命令的实例会显示该提示，而不会在首次发消息时失败。
- **先登录。** 该驱动没有自己的认证流程——在终端中运行一次 CLI 并登录；OpenMausBot 启动它时会保留你的登录状态。
- **模型选择留在代理内部。** 选择器只显示一个"代理默认"条目；CLI 配置运行什么就运行什么。
- **`environment`** 会传递给 CLI 子进程。外部提供商的密钥（XAI_API_KEY、OPENAI_COMPAT_API_KEY 等）会被刻意剥离，确保自定义 CLI 绝不会用其他引擎的登录计费。
- **权限** 走 ACP 自身的 `session/request_permission`——如果你的代理发起请求，该请求会变成聊天中的普通审批卡片。
- 多个实例没问题——每个代理一个。

## 任意 OpenAI 兼容端点（完全无进程）

内置的 `openai-compat` 驱动支持多个实例，因此本地 vLLM/LM Studio/Ollama-openai 端点或任意托管兼容 API 只需一个条目：

```json
{
  "instances": {
    "my-endpoint": {
      "driver": "openai-compat",
      "displayName": "My Endpoint",
      "environment": { "MY_ENDPOINT_KEY": "sk-…" },
      "config": {
        "url": "http://127.0.0.1:1234/v1",
        "apiKeyEnv": "MY_ENDPOINT_KEY",
        "model": "my-model"
      }
    }
  }
}
```

- `apiKeyEnv` 指定哪个 `environment` 值携带密钥，因此多个实例可持有不同密钥而不冲突。
- 该驱动在可行时会列出端点的 `/models`，无论如何都会将你的 `model` 保留为自定义选项。
- OpenAI 兼容实例支持结构化工具和图像输入。若模型两者都支持，harness 可挂载已批准的主机、Local VM、VPS、Boat 云或内置浏览器工具，并将截图返回给模型。
  现有的平台、计算机选择和审批检查仍然适用。
  Boat 轮次在直接聊天、房间和云例程中保留所选 API 模型，与所有带计算机工具的引擎一致。
- 对于不支持工具的模型，将 `config.tools` 设为 `false`。图像支持同样取决于所选模型；驱动无法为纯文本模型添加视觉能力。范围与测试参见 [工具验证](verification/openai-tools.md)。

## 注意事项

- `config.json` 以 0600 权限写入；`environment` 中的值以明文存储在该文件中。建议使用仅限单个引擎范围的密钥。
- `driver` 拼写错误或 `config` 无效绝不会导致应用崩溃：该实例会显示为不可用并附原因，其余实例正常加载。