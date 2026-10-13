# OpenMausBot 中的语音

历史决策文档，2026-08-14。记录最初仅支持 ElevenLabs 的形态；当前产品还支持 Fish Audio、Mac 内置语音和本地 Chatterbox。参见当前[语音指南](../apps/docs/content/docs/features/voice-and-memory.mdx)。

## 形态

```
Renderer (src/)                        Harness (server/)
├── lib/tts/index.ts   说话者          ├── tts/speech-text.ts  markdown → 可朗读文本
│     队列 · 预取 · 打断               └── tts/elevenlabs.ts   校验 · 音色 · 合成
└── components/CallView.tsx
      Apple STT 端点检测               POST /api/tts/prepare → 话语
                                       POST /api/tts/speak   → mp3 字节
```

只有一个语音提供商：**ElevenLabs，自带密钥**。没有本地模型，没有第二提供商，没有回退阶梯——没有密钥，语音就关闭，按钮也会如实显示。

## 为什么密钥留在 harness

渲染进程从不直接与 ElevenLabs 通信。`GET /api/config` 只返回是否已配置的布尔值，其他一概不返回，这与所有其他凭据遵循同一规则，其价值超过省下的一次往返。因此应用向 harness 请求音频，harness 持有密钥。

有两种状态值得区分，因为它们需要不同的提示：`configured`（已保存密钥）和 `ready`（密钥*和*已选音色）。两者缺一就朗读会抛出 `NoVoiceConfigured`，路由将其转为 409——"你还没配置好"不是提供商故障，也不应看起来像。

## 朗读语体

决定体验是否舒适的那一半。Agent 是为屏幕写作的：围栏代码、文件路径、表格、链接大杂烩。逐字朗读，一段 diff 就是四分钟的标点，而 `server/drivers/acp/core.ts` 会变成"server 斜杠 drivers 斜杠 a c p 斜杠 core 点 t s"。

`speech-text.ts` 朗读散文，念出产物名，丢弃语法。它还拆分为话语，因为那是工作单元——一次请求，一段音频，客户端在当前段播放时预取下一段。每个话语一次请求，而非流式输入的 WebSocket：感知延迟相同，活动部件少得多，且回合被打断时没有会泄漏的 socket。

## 通话模式

**有意采用半双工。** 听写助手是 `SFSpeechRecognizer`，基于原始 `AVAudioEngine` 输入，没有声学回声消除。播放期间麦克风保持开启，会把机器人自己的声音转写回对话，两者就会永远对讲下去。因此麦克风仅在机器人不说话时开启，打断方式是点击、空格键或 Escape。全双工插话需要在采集路径上做 AEC——这是真正的后续工作，不是脚注。

**回合检测保持原生且本地。** 基于缓冲的 `SFSpeechRecognizer` 不会仅因说话者安静下来就发出 `isFinal`；它只在其音频流结束后才 finalize。因此通话模式以静音超时启动原生助手。一旦非空转写停止变化达 850ms，助手停止采集并调用 `endAudio()`，产生发送给渲染进程的最终转写。输入框听写省略该超时，保留按下停止的行为。不涉及云端 STT 或内置 VAD 模型。

**旁白让它可忍受。** 一个 agent 回合是 5–60 秒的工具调用，这么长的沉默会被当成通话掉线。harness 旁白的每个活动 chip 都在发生时被朗读。短语在折叠时于服务端一次性计算为 `tool.spoken`——所以你看到的 chip 和你听到的短语不会漂移。

**审批会被朗读。** `request.opened` 卡片会被读出，并以"yes"/"no"作答。任何不明确是决定的内容都会被拒绝并重新询问：绝不能从仅仅包含"sure"一词的句子中推断同意。非权限类问题也会被读出，下一个完整的朗读回合作为答案返回，因此请求输入的 agent 不会让通话卡在看不见的卡片后面。

**延迟，如实说。** 端点检测为 300–700ms，首字节时间约 100–250ms，而 agent 回合为 5–60s。agent 以 50–100 倍主导，所以音色选择是质量决策，不是延迟决策。让通话感觉像对话的方法，是把你呼叫的机器人放在快速模型上，让它用 `coordinate_bots` 把实际工作交给专家——无需新机制。

## 已否决

| 选项 | 为何不选 |
| --- | --- |
| 系统语音（macOS/Windows） | 听感合成味重；会拉低该功能的档次 |
| Piper | 同样的毛病，高一个档次 |
| 渲染进程中的 Kokoro-82M | 确实好且免费，但它是第二提供商、2.2MB 分块、ONNX 运行时和首次运行模型下载。简洁胜出。 |
| Cartesia | 更便宜、首字节更快，但第二提供商只有在第一个不够用时才值得 |
| ElevenLabs Agents | 其自定义 LLM 的 `cascade_timeout_seconds` 上限为 15s，而 agent 回合超过该值；它还想接管回合管理和工具调用，而这些归 harness 所有 |
| OpenAI Realtime / Gemini Live（语音到语音） | 它们替换大脑，而大脑是你自己机器上的 Claude Code *就是*产品本身。（后来：GPT-Live 的*客户端委托*保留机器人为大脑，因此成为可选的 Live 通话模式——见 `server/live-call.ts`。） |

## 已知缺口

- **轮流通话仅限 macOS**，因为听写仅限 macOS。语音那一半到处都能用。Live 通话（GPT-Live，`server/live-call.ts`）不需要听写，因此在任何电脑的桌面应用和手机上都能用，使用用户自己的 OpenAI 密钥。A Cloud 的页面，在网页浏览器或桌面应用中，运行相同代码，但尚未在真实 Cloud 上端到端进行过 Live 通话。
- **房间还不能说话**，尽管每个机器人的音色已存在（`bot.voice`）。
- **没有消费计量。** ElevenLabs 按字符计费。自动朗读默认关闭部分出于此原因，但应用最终应显示用量。
- **没有语音插话**——见上文半双工。

## 失败边界

- 有意的麦克风停止（播放、挂断或替换）不会发出自然的 `speech:end`；否则渲染进程可能在机器人音频期间重新开启采集。
- 通话阶段与 React 状态同步更新，因此助手退出与最终转写处于同一事件循环回合时，不会观察到过期的 `listening` 阶段。
- 离开机器人视图会拥有并结束其通话。隐藏的覆盖层不会留下麦克风会话或过期的 `currentCall`。
- 合成请求可从渲染进程中止，单个话语在服务端设有上限，以限制意外的托管语音消费。