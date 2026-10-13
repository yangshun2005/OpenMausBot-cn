# OpenCode Go 集成方案

- 状态：提案
- 最近上游审查：2026-08-14
- 跟踪产物：添加本文档的 Pull Request

## 概述

通过官方 Agent Client Protocol（ACP）接口运行维护中的 OpenCode CLI，并从 `opencode-go/*` 提供商选择模型，将 OpenCode Go 作为 OpenMausBot 的一等引擎接入。

首个实现应复用 OpenMausBot 的 ACP 运行时，而非直接调用模型端点。这样可保留 OpenCode 的编码工具、会话、MCP 支持和权限请求。直接使用模型 API 驱动只能提供推理能力；OpenMausBot 目前没有与提供商无关的 agent/工具循环来替代 OpenCode CLI 提供的功能。

本方案中的 **OpenCode Go** 指 [opencode.ai/docs/go](https://opencode.ai/docs/go/) 所记录的当前订阅/API 产品，不针对 `opencode-ai/opencode` 下已归档的 Go 语言仓库——该仓库已迁移至其他项目，不再是当前的 OpenCode 实现。

## 需保留的上游事实

- OpenCode Go 是可选订阅，通过提供商 id `opencode-go` 暴露一组不断变化的编码模型。
- 公开模型目录可从 `https://opencode.ai/zen/go/v1/models` 获取。
- 模型推理分散在 OpenAI Responses、OpenAI 兼容的 chat completions 和 Anthropic Messages 端点。将所有模型视为单一 chat-completions API 是错误的。
- 维护中的 OpenCode CLI 支持 `opencode acp`，即基于 stdio 的 JSON-RPC 进程。其 ACP 实现包含会话、流式消息、工具活动、权限请求、取消、MCP 服务器和模型选择。
- 在 ACP 中，OpenCode 将模型选择暴露为 `model` 会话配置项。所选值为完整的 `opencode-go/<model-id>` 字符串，需在提示前通过 `session/set_config_option` 设置。
- OpenCode 接受 `OPENCODE_API_KEY` 环境变量。OpenMausBot 应仅将其注入 OpenCode 子进程。
- 官方跨平台 CLI 包为 `opencode-ai`；`opencode auth login` 仍是用户自行管理凭据的替代流程。
- Go 的模型阵容、限额、价格、提供商和保留条款可能变化。应用必须链接到 OpenCode 实时文档，而非在代码中复制易变的商业细节。

主要参考：

- [OpenCode Go 文档](https://opencode.ai/docs/go/)
- [OpenCode ACP 文档](https://opencode.ai/docs/acp/)
- [OpenCode CLI 文档](https://opencode.ai/docs/cli/)
- [OpenCode 服务器文档](https://opencode.ai/docs/server/)
- [维护中的 OpenCode 仓库](https://github.com/anomalyco/opencode)

## 建议架构

### 运行时：小型 OpenCode ACP 适配层

在现有 `createAcpDriver` 核心之上，添加 `server/drivers/acp/opencode-go.ts` 作为 `AcpSupport` 定义：

- 驱动类型：`opencodeGoAgent`
- 显示名称：`OpenCode Go`
- 可执行文件：`opencode`
- argv：`opencode acp`
- 模型 id：完整的 `opencode-go/<model-id>` 值
- 凭据：`OPENCODE_API_KEY`，作用域限于该子进程
- 安装命令：macOS、Linux 和 Windows 上使用 `npm install -g opencode-ai@latest`，并以官方安装/文档页面作为回退
- 设置链接：OpenCode Go 订阅和 API 密钥说明

初始集成不要 Fork OpenCode CLI、内嵌其 SDK 或启动网络监听服务器。ACP 已提供本地 stdio 边界，契合 OpenMausBot 用于 Grok 和 Gemini 的进程生命周期。

### 通用 ACP 变更

为 ACP 支持对象扩展一个可选的模型选择 Hook。在 `session/new` 或 `session/load` 之后、`session/prompt` 之前，核心应：

1. 检查返回的 `configOptions`；
2. 确认存在 `model` 选项且包含所请求的完整模型 id；
3. 以 `{ configId: "model", value: modelId }` 调用 `session/set_config_option`；
4. 在 `session.started` 中使用返回的当前值；以及
5. 以可操作的错误失败，而非静默运行其他模型。

该 Hook 必须可选，以便现有 Grok 和 Gemini ACP 驱动保持当前的 CLI 参数模型选择，直到有意迁移。

OpenCode 的 ACP 认证方法仅启动/确认其自身登录流程，并不证明 Go 订阅处于活跃状态。因此可用性应区分以下状态：

- CLI 缺失：不可用，显示 PR #97 中的驱动安装路径。
- 密钥缺失：不可用，深链到凭据行。
- 密钥存在：足以尝试一轮对话。
- 密钥无效、订阅未激活、配额或区域失败：规范化上游响应且不暴露密钥；认证/设置失败应携带 `setup: true`，而配额和临时提供商故障应保持为普通运行时错误。

### 凭据

在现有应用凭据旁添加只写的 OpenCode Go 凭据：

```json
{
  "opencodeGo": {
    "key": "..."
  }
}
```

`OPENCODE_API_KEY` 仍作为支持的环境变量回退。配置 API 只能返回 `opencodeGo.configured: boolean`；绝不能返回密钥。

仅 OpenCode Go 实例接收解析后的密钥。其他 CLI 子进程、原生日志、错误文本、分析和渲染进程载荷均不得接收。清除或替换密钥应使用现有的原子配置写入和提供商重载路径。

Connections 界面应提供：

- API 密钥的密码输入框；
- 订阅/管理 OpenCode Go 并复制密钥的链接；
- 明确提示这是可选的付费第三方服务；
- 与其他凭据一致的保存、替换和清除行为；以及
- 可同时表示"安装 CLI"和"添加密钥"的设置状态。

驱动可额外记录 `opencode auth login` 供偏好 OpenCode 自有凭据存储的用户使用，但首个实现不应依赖读取或编辑 OpenCode 的私有认证文件。

### 模型目录

从文档所述的 `/zen/go/v1/models` 端点获取当前目录。在 `ModelSelection` 中存储完整的提供商限定 id，同时在选择器中展示简洁标签。

目录行为必须故障安全：

- 使用短超时，不无限阻塞应用启动；
- 校验响应结构，仅接受字符串 id；
- 发现或健康检查期间绝不发起付费推理请求；
- 在进程生命周期内缓存最后一次有效目录；
- 离线时回退到小型固定目录；以及
- 在现有实例重新探测路径上刷新，使上游新增无需重启 OpenMausBot。

如果当前同步的 `ProviderInstance.models` 契约无法实现真实刷新，则为驱动/注册表契约添加小型异步目录方法，而非在注册表背后修改共享对象。

### 会话、工具和权限

以现有 ACP 映射为基线：

- 将 OpenCode 会话 id 保存为恢复游标；
- 在下一个 OpenMausBot 轮次加载该会话；
- 流式输出助手文本和推理一次，不重复回放；
- 将工具启动/完成更新转换为规范运行时事件；
- 将 `session/request_permission` 转换为现有审批卡片；
- 无匹配上游权限选项时拒绝或取消；
- 按选项 id 转发用户的允许/拒绝选择，绝不按数组位置；
- 中断时发送 `session/cancel`，宽限期后终止子进程树；以及
- 仅在 OpenCode 报告时展示用量；不要虚构成本值。

首个切片应支持与当前通用 ACP 核心相同的 agent 和 computer stdio MCP 集成。HTTP/SSE MCP 新增（包括直接 Composio 传输）应作为后续独立工作，待其 ACP Schema 和权限行为经测试覆盖后再进行。

## 预期代码区域

- `server/drivers/acp/opencode-go.ts`：OpenCode 专用 ACP 支持。
- `server/drivers/acp/core.ts`：可选的会话配置/模型选择，以及协议验证发现的任何 OpenCode 事件结构兼容性。
- `server/drivers/builtIn.ts`：注册。
- `server/contracts.ts` 和 `server/harness/registry.ts`：仅在验证需要时添加异步目录或设置元数据。
- `server/config.ts` 和 `server/index.ts`：只写密钥持久化、环境变量回退、状态报告和提供商重载。
- `src/components/ApiKeys.tsx`、设置组件和 `src/state/store.tsx`：凭据 UI 和仅配置状态。
- `src/components/ProviderIcons.tsx`、引导流程和模型选择器：提供商展示和设置入口。
- `server/testing/fake-acp-cli.ts` 及 ACP/配置测试：确定性的协议、凭据、目录和生命周期覆盖。

实现 PR 中不要包含无关的引擎设置、样式和提供商重构。

## 以小 PR 交付

### PR 1：协议验证和通用 ACP 模型选择

- 针对测试配置捕获一次脱敏的 `opencode acp` 会话。
- 为返回的会话配置项添加 fake-ACP 覆盖。
- 添加可选的 `session/set_config_option` Hook。
- 在不添加可见引擎的情况下验证所请求模型的选择、会话恢复、取消和权限行为。

退出条件：fake OpenCode ACP 进程在确认所请求的 `opencode-go/*` 模型之前无法提示。

### PR 2：驱动、目录和凭据管道

- 添加 OpenCode Go 驱动和内置注册。
- 添加只写密钥存储和 `OPENCODE_API_KEY` 回退。
- 添加带超时、校验、缓存和离线回退的模型发现。
- 添加安装/设置元数据和服务端测试。
- 在实时冒烟测试完成前，通过显式实例配置保持驱动可选。

退出条件：`/api/instances` 在 CLI 缺失、密钥缺失、就绪和目录离线状态下如实报告引擎，且不泄露凭据。

### PR 3：产品设置和选择器集成

- 添加 Connections 凭据行和 OpenCode Go 提供商展示。
- 复用 PR #97 的焦点/选择器重新探测，使安装和密钥变更无需重启应用即可生效。
- 仅在 macOS、Linux 和 Windows 设置路径验证后，将引擎加入引导/默认集群。
- 将易变的价格、用量、隐私和模型信息链接到 OpenCode 实时文档。

退出条件：新用户无需编辑配置文件即可从不可用变为可选择 Go 模型，且无虚假"就绪"状态。

### PR 4：实时冒烟覆盖和用户文档

- 运行可选的实时 CLI 冒烟测试，覆盖一次流式对话、一次权限请求、会话延续、模型切换和取消。
- 在所有支持的桌面操作系统上验证进程清理和可执行文件发现。
- 记录设置、密钥移除、配额/认证错误和上游数据政策链接。
- 决定引擎是否可加入默认集群。

退出条件：集成满足下方完成定义，并记录最低支持的 OpenCode CLI 版本。

## 测试矩阵

自动化测试必须覆盖：

- 配置解码和默认可执行文件；
- macOS、Linux 和 Windows 上可执行文件缺失及 PATH 刷新发现；
- 密钥缺失、环境变量回退、已保存密钥、替换和清除；
- 仅配置的 API 响应及日志/错误脱敏；
- 目录成功、JSON 格式错误、超时、HTTP 失败、空结果、缓存和回退行为；
- 完整的提供商限定模型 id 及拒绝 ACP 会话未通告的模型；
- 顺序：initialize -> authenticate -> new/load -> set model -> prompt；
- 流式文本无重复最终内容；
- 工具生命周期及权限允许、拒绝、超时和选项缺失行为；
- 会话加载、会话缺失回退及恢复后模型切换；
- 会话创建前后的取消、进程提前退出、JSON-RPC 格式错误和进程树清理；以及
- 并发 OpenMausBot 线程无共享 ACP 或凭据状态。

实时测试必须可选，因为它们需要私有订阅密钥且可能消耗配额。CI 绝不能打印密钥或上传带凭据运行的原始协议日志。

## 完成定义

- 在 CLI 和凭据均存在前，OpenCode Go 显示为不可用；否则 OpenMausBot 绝不将其选为可运行默认。
- 设置流程无需重启 OpenMausBot 即可生效，且绝不静默执行安装程序。
- 用户可选择当前通告的每个 `opencode-go/*` 模型，且 ACP 会话在提示开始前确认该确切模型。
- 机器人可流式回复、使用编码工具、请求审批、延续会话、切换模型并干净取消。
- Agent/computer MCP 集成行为与其他 ACP 引擎一致，或明确显示为不支持。
- 密钥缺失、密钥无效、订阅未激活、配额耗尽、区域限制、上游中断和目录中断具有不同且有用的错误。
- 凭据为只写，且不出现在渲染进程载荷、子进程参数、日志、分析、测试快照和错误消息中。
- macOS、Linux 和 Windows 的设置和运行时路径已验证。
- 集成记录最低支持的 OpenCode CLI 版本，并链接到实时 Go 价格、用量、模型和隐私条款。

## 推迟的工作

- 直接无 CLI 的 OpenCode Go API 驱动。这需要通用 agent/工具循环、工具调用执行、审批代理和三种上游传输格式。
- OpenCode 的 HTTP 服务器/SDK 集成。它引入端口分配、认证、就绪和服务器生命周期问题，且相对 ACP 无初始优势。
- 自动订阅购买、账单管理或用量充值。
- 持久化或修改 OpenCode 自身的认证/配置文件。
- 通过 ACP HTTP/SSE 传输提供 Composio。
- 支持已归档的 Go 语言 OpenCode CLI。

## PR 1 中需解决的问题

1. 哪个最低 OpenCode CLI 版本具有稳定的 `opencode acp` 配置项和权限事件结构以支持本集成？
2. CLI 是否暴露可靠的非计费凭据/订阅就绪检查，还是就绪仅意味着"CLI 和密钥存在"直到首轮对话？
3. `session/load` 是否在所有支持版本中返回当前 `model` 配置项，且能否在每次提示前切换？
4. 哪些 OpenCode Go 错误可识别无效凭据、订阅未激活、配额和区域限制，而无需匹配易变的英文字符串？
5. 目录回退应为捆绑元数据还是持久化到磁盘的最后一次成功目录？
6. 现有 ACP 用量元数据能否区分累计会话总量与每轮用量，使 OpenMausBot 不重复计数？