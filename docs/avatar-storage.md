# 头像附件生命周期

机器人头像有意复用图片附件存储。因此，上传和生成的图片与消息图片一样，会经过相同的大小检查、仅所有者可访问的文件系统权限、不可变的访问 URL，以及仅限光栅图的 MIME 白名单。

## 图片提供商设置

打开机器人的个人资料，找到 **头像 → 使用 AI 生成**，然后选择一个**图片提供商**。所选连接由所有机器人头像共享。保存连接，可选地描述图片，然后选择**生成头像**。图片提供商独立于机器人的聊天引擎。

- **OpenAI** 是现有安装的默认选项。保存 OpenAI API 密钥后，可使用 `gpt-image-2` 生成低质量 1024 × 1024 WebP 草稿。参见 [OpenAI 图片生成指南](https://developers.openai.com/api/docs/guides/image-generation)和 [GPT Image 2 参考](https://developers.openai.com/api/docs/models/gpt-image-2)。
- **Grok (xAI)** 使用 `grok-imagine-image-2.0`，采用正方形宽高比和 base64 输出。保存 xAI API 密钥，或复用已在设置中配置的 Grok 密钥。更改或删除此密钥也会影响其他 Grok 功能。参见 [xAI 图片生成指南](https://docs.x.ai/developers/model-capabilities/images/generation)。
- **自定义**接受兼容的 Images API。输入其 **Base URL**、**图片模型** ID 和可选的 API 密钥，然后选择**保存连接**。使用该网关支持的模型 ID，包括其要求的任何提供商前缀。OpenMausBot 仅向此连接发送自定义密钥。

直接使用 OpenAI 和 Grok 生成需要各自的 API 密钥和 API 访问权限。登录 ChatGPT、Codex 或 Grok 订阅不会配置这些密钥。Grok 订阅计费与 xAI API 计费也是分开的；参见 [xAI 账户常见问题](https://docs.x.ai/developers/faq/accounts#if-i-already-have-an-account-for-grok-can-i-use-the-same-account-for-api-access)。

### 自定义网关兼容性

对于 OmniRoute 等本地网关，base URL 可能是 `http://127.0.0.1:20128/v1`。OpenMausBot 会追加 `/images/generations`；粘贴完整端点也可以。`localhost` 和 `127.0.0.1` 指的是运行 OpenMausBot 服务器的机器，包括你远程打开应用时。公共端点需要 HTTPS。本地/私有端点支持 HTTP。URL 不得包含内嵌凭据、查询参数或片段。

网关必须接受同步的 OpenAI 风格 Images API 请求，包含 `model`、`prompt`、`size: "1024x1024"`、`n: 1` 和 `response_format: "b64_json"`。其 JSON 响应必须包含 `data[0].b64_json`，其中是原始 base64 编码的 PNG、JPEG 或 WebP 字节。仅兼容 Chat API 不够；不支持仅返回 URL 的响应、重定向和异步任务响应。每张解码后的图片限制为 10 MiB，响应限制为 15 MiB，生成限制为两分钟。

初次配置无密钥网关时，将自定义密钥留空。如果之前保存过自定义密钥，选择**删除已保存的密钥**可使连接变为无密钥；否则留空会保留已保存的密钥。OpenAI 和 xAI 密钥绝不会作为自定义连接的备用密钥。

## 延迟清理

替换或删除头像**不会**立即删除之前的文件。当前附件记录没有来源信息：同一个生成的文件名可能被机器人个人资料引用、被一条或多条持久化消息引用，或两者兼有。仅因当前没有机器人使用就删除文件，可能会破坏历史消息，因此宽泛的"未引用文件"清理不具备引用安全性。

未来的有界清理可能只删除创建时记录为头像所有的文件。在删除某个候选文件之前，仍必须验证：

1. 没有机器人使用该 `avatarUrl`；
2. 没有活动或归档的任务/房间消息引用其存储路径；以及
3. 该文件名属于头像所有注册表，而非旧版共享附件池。

清理每次运行应处理少量固定数量的候选文件，并保留宽限期。在该来源注册表存在之前，保留旧头像是安全的非破坏性行为。