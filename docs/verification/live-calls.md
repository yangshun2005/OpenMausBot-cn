# 实时通话：通话 ID 与通话记录

一次实时通话（OpenAI GPT-Live 作为语音，机器人作为大脑）会在其聊天中留下两样东西。通话中说的每个请求都携带该通话的 ID，而一次成功接通的通话在结束时留下一条 "call" 行，桌面端将其绘制为机器人所做工作的记录。本配方在一次性工作区中，用假 GPT-Live（`server/testing/fake-openai-live.ts`）和假引擎对两者进行校验。此处不涉及音频、真实 OpenAI 密钥或电话。设计见 `docs/superpowers/specs/2026-09-25-live-call-bar-design.md` 中的 "The call id and the call row"。

## 子功能

- 通话中说的请求在每条传递路径上都携带 `callId` 和 `via: "call"`：新回合、引导、队列（重启后也是）以及对问题的延迟回答。手动输入的行从不携带。
- 一次成功接通的通话在聊天的活动叶子处恰好留下一条 `kind: "call"` 行，文本为 `Call with <bot> · m:ss`，并带一条 `call` 记录。从未接通的通话，或通话期间聊天或机器人被删除的通话，不留下任何行。
- 开启生成的会话标题时，机器人自身的引擎根据其语音请求为通话命名，标题以该行的补丁形式到达。关闭时，或没有任何请求时，该行保持 `Call with <bot>`。
- 该行永远不是机器人上下文，也永远不是下一次通话的语音历史。
- 桌面端将该行绘制为紧凑记录：标题、时长，以及通话语音请求所启动工作的步骤和审批。它按通话 ID 选择并读取整个转录，因此一条记录只列出自己通话的工作，也包括在该行之后落地的步骤。

## 来自电话的实时通话需要 Pro

来自桌面应用的通话（客户端 `desktop`，或本机上的 `web`）免费。来自电话的通话需要任意档位的有效 OMB Cloud 套餐（Personal、Pro 或 Max）：经由伴侣应用的通话（它携带配对设备，电话无法选择或丢弃），或指定客户端 `ios`/`android` 的通话（`server/live-call-controller.ts` 中的 `liveCallFromPhone`）。没有套餐时，`POST /api/live/session` 在索要密钥（`409 needsKey`）之前、在任何内容到达 OpenAI 之前，返回 `402 { "error": "Live calls from your phone need a Pro plan.", "needsPro": true }`；伴侣应用将其原样传给电话。桌面端随后在通话按钮下显示 Pro 卡片而非密钥表单，仅向已登出（先登录）或免费用户提供。正在关联的付款仍算作 Pro。当套餐无法检查时（尚未读取、正在登录、OMB Cloud 不可达，或本机登录已结束），以最后验证的套餐为准。Cloud home 始终算数。仅在电话通话开始时才询问套餐，因此套餐变更永远不会结束正在进行的通话。规则见 `electron/pro-plan.mjs` 中的 `liveCallsAllowed`；设计见上述 spec 中的 "Live calls need Pro"。

桌面应用将其答案交给服务器：启动时的 `OMB_PRO_PLAN`，之后每次 Cloud 登录变更时通过私有的 `openmausbot:pro-plan` 消息。非它启动的服务器（开发、测试、无头或 Docker，以及本 fixture）用 `OMB_PRO_PLAN=1` 表示 Pro，fixture 启动器只允许 `1` 通过。

```sh
node --test electron/pro-plan.node-test.mjs
pnpm exec vitest run server/pro-plan.test.ts server/live-call-controller.test.ts server/routes/live.test.ts companion/test/proxy-response.test.ts scripts/control-omb.test.ts
pnpm exec vitest run src/lib/live-call-media.test.ts src/components/CallView.live-mode.test.ts src/components/ProIntroduction.test.ts
```

`server/live-call.e2e.test.ts` 用 `OMB_PRO_PLAN=1` 启动其主服务器，再启动一个不带它的 Cloud home（Pro，因此只索要密钥），以及第三个按桌面应用方式启动的服务器（`OMB_PRO_PLAN=0`，然后通过替身父端口发送私有消息）：以 402 拒绝且无会话，消息说 Pro 后通话建立，消息说非 Pro 后仍保持，下一次启动被拒绝。

## 用户路径

从通话按钮的菜单启动一次实时通话，说一些机器人必须处理的内容，然后挂断。聊天保留你说的话（标记为 "via call"）、机器人的步骤和回答，并在其下方保留一条通话的小记录。

## 驱动与证据

服务器路径（包括引导、队列、重启和延迟回答）需要脚本化语音，因此其配方是端到端测试：

```sh
pnpm exec vitest run server/live-call.e2e.test.ts server/chat-followups-restart.test.ts
pnpm exec vitest run server/live-call-record.test.ts server/live-call-controller.test.ts server/steer-queue.test.ts server/thread-title.test.ts server/delta-context.test.ts server/live-call.test.ts server/store.test.ts
pnpm exec vitest run src/lib/call-record.test.ts src/lib/live-activity.test.ts src/lib/activity-runs.test.ts src/components/CallRecordRow.test.ts src/components/ChatView.via-call.test.ts src/components/ChatView.rows.test.ts
pnpm typecheck
pnpm i18n:check
```

`server/live-call.e2e.test.ts` 在一次性 home 中启动真实测试装置，带假引擎和假 GPT-Live，按客户端方式启动通话（`POST /api/live/session`），通过 sideband 播放语音一侧（一些 `session.input_transcript.delta` 事件，然后一个 `session.delegation.created`）并挂断（`POST /api/live/call/end`）。它检查新回合、引导行、排队行（也包括被 Steer 从队列中提起的一行）和延迟回答上的通话 ID；每次成功接通的通话一条行，从未接通或机器人被删除的通话没有行；开启生成标题时的标题补丁，关闭时没有；该行在下一次通话启动历史中缺席；以及该行和服务器日志都不含所说的话，日志和事件流都不含密钥。`server/chat-followups-restart.test.ts` 在一次派发中途杀死 fixture 服务器，并检查恢复的行：通话中说的一行带其通话 ID 回来，携带杂散通话 ID 的输入行不带它回来。

手动操作时，一次没有语音请求的通话会显示该行和记录。独立假服务无法从 shell 脚本化，因此不对它发出任何请求：

```sh
# terminal 1: the fake GPT-Live prints its base URL
node --experimental-strip-types server/testing/fake-openai-live.ts
# terminal 2: the renderer fixture, pointed at it (the key only ever reaches
# the fake), and Pro, which Live calls need
OMB_OPENAI_LIVE_URL=http://127.0.0.1:FAKE_PORT OMB_OPENAI_LIVE_KEY=sk-fake OMB_PRO_PLAN=1 \
  node --experimental-strip-types scripts/control-omb.ts ui launch
# terminal 3, with the url, botId and ui handle it prints
pnpm control:omb messages --bot BOT_ID --url http://127.0.0.1:PORT
curl -s -X POST http://127.0.0.1:PORT/api/live/session -H 'content-type: application/json' \
  -d '{"botId":"BOT_ID","sdp":"v=0\r\no=- 1 1 IN IP4 127.0.0.1\r\n","client":"desktop"}'
curl -s http://127.0.0.1:PORT/api/live/call
curl -s -X POST http://127.0.0.1:PORT/api/live/call/end -H 'content-type: application/json' \
  -d '{"callId":"CALL_ID"}'
pnpm control:omb ui wait-settle --ui /tmp/openmausbot-verify-data-XXXXXX/ui.json --timeout 60
pnpm control:omb ui snapshot --ui /tmp/openmausbot-verify-data-XXXXXX/ui.json
curl -s "http://127.0.0.1:PORT/api/threads/THREAD_ID/messages?limit=20"
```

第一条命令 JSON 中的 `taskId` 即 THREAD_ID。重复对 `/api/live/call` 的 `curl` 直到 `status` 为 `live`。预期：启动返回 `201` 和 `status: "connecting"`；挂断返回 `status: "ended"` 和 `endReason: "hung-up"`；最后一条命令列出恰好一条 `kind: "call"` 行，其 `call` 含 `callId`、`botId`、`client`、`startedAt`、`endedAt`、`seconds: 42`（假服务报告 42 秒用量，与实际时钟无关）和 `endReason`，且无 `title`，因为没有任何请求；快照的转录中有 `group "Live call: Call with Pepper, 0:42"`。用 Ctrl-C 停止 fixture。

## 注意事项

- `pnpm control:omb messages` 只投影固定字段集：它显示该行的 `kind` 及其文本，但不显示 `via`、`callId` 或 `call`。这些请读上面最后一条命令。
- 只有生成的会话标题才为通话命名（`features.llmThreadTitles`；`ui flag --set` 见 [Chat UI](chat-ui.md)），且仅根据通话中所请求的内容：没有任何请求的通话永远不会被命名。
- fixture 的假引擎从 `FAKE_CLAUDE_TEXT_ROUTES` 一次性回答标题，它像其他 `FAKE_CLAUDE_*` 变量一样进入 fixture。没有路由时它回答 "fake generated text"。
- 必须不接通的通话需要 sideband 以 404 拒绝（e2e 中的 `refuseNextAttach(404)`）。Node 的 WebSocket 会重试被拒绝的 401 升级，而重试会被接受。
- 一次只能运行一个通话：一个打开时，另一次启动返回 `409`。结束失败运行遗留的打开通话，或重启 fixture。
- `OMB_OPENAI_LIVE_URL` 仅当为回环 `http://127.0.0.1:PORT` 时才进入 fixture，`OMB_OPENAI_LIVE_KEY` 仅随之进入。
- 没有 `OMB_PRO_PLAN=1` 时，fixture 对每次启动都返回 `402` `needsPro`，桌面端在通话按钮下显示 Pro 卡片。这是手动查看该卡片的方式；下面记录的运行早于该门控。

## 边界

- 无音频、WebRTC、真实 OpenAI 密钥或账单。脚本化语音发送 e2e 所发送的内容；GPT-Live 可能以不同方式为转录和委派计时。每个客户端在 PR 前仍用真实音频和真实密钥测试，如通话栏的设计所述。
- 标题一次性回答是假引擎的脚本化回答。这证明门控、摘录（仅语音请求）和补丁，而非标题质量。
- 此处的渲染器是无头 Chrome 中的 web 渲染器，而非打包的 Electron shell。e2e 仅限 POSIX（其假服务是 shebang 脚本），因此未在 Windows 上演练。
- 电话构建读取相同的线上字段，当它们不认识 kind 时读取该行的文本；其自身记录在 SupaMaus/mausbot-mobile 中验证。
- 记录中的审批行（"Allowed"、"Denied"、"Expired"、"by voice"）、回合仍在工作时挂断、从未报告结果的步骤（仅在聊天工作时显示为运行中）以及加载页面之前开始的通话（记录说其中部分可能在更早的消息中）由 `src/lib/call-record.test.ts`、`src/components/CallRecordRow.test.ts` 和 `src/components/ChatView.via-call.test.ts` 以及 e2e 的审批测试中的语音 yes 覆盖。下面的 fixture 运行不绘制它们。

## 最后演练

2026-10-08，`feat/live-call-record` 上 0d761f09 的 `ui launch` 隔离 macOS fixture：测试装置在端口 22340，假 GPT-Live 在回环端口，假引擎，以及通过 agent-browser 0.37.0 的无头 Chrome for Testing 153。首次通话后用 `ui flag --set features.llmThreadTitles=true` 开启生成标题，一条 `FAKE_CLAUDE_TEXT_ROUTES` 条目回答了标题提示。有请求的通话的语音一侧由一个一次性脚本播放，未保留在仓库中，它像 e2e 那样调用 `startFakeOpenAiLive` 和 `emit`。

- 通话 A，标题关闭：语音请求 "what is six times seven" 作为用户行落地，带 `via: "call"` 和等于通话 ID 的 `callId`，机器人运行了一个 Bash 步骤并回答，语音被告知了答案。挂断（`endReason: "hung-up"`）后有一条行，`Call with Pepper · 0:42`，带 `seconds: 42`（假服务的用量，尽管从开始到结束实际时钟耗时 2.6 秒），无标题，12 秒内无补丁。
- 通话 B，标题开启：同上，外加通话期间输入的一条消息。输入行不携带 `via` 和 `callId`。该行先以无标题写入，随后 `Times table question` 作为同一行的 `message.patch` 到达，其 `text` 保持 `Call with Pepper · 0:42`。
- 通话 C，假服务以 404 拒绝 sideband 升级：通话以 `sideband-lost` 结束，日志中 `seconds=0`，不留行。
- 通话 D，接通但无请求且标题开启：一条行，无标题。
- 渲染器：三条 `region "Live call: …"` 记录，通话 C 没有。A 读作 "Call with Pepper, 0:42"，一行 "Running a command (Done)"。B 读作 "Times table question, 0:42"，一行，尽管输入回合在通话期间运行了自己的 Bash 步骤；该回合的 chip 和回复位于记录上方，未列出。D 读作 "Call with Pepper, 0:42"，无行。侧边栏行保持预览机器人的最后回复，而非记录。
- fixture 的日志对四次通话各有 `call started` 和 `call ended` 行，语音回合为 `text=(spoken)`，任何地方都没有所说的话或密钥。
- Ctrl-C 停止了 fixture 并删除其数据目录；没有遗留 fixture、预览或驱动进程。日志保留在其打印路径。
- `pnpm exec vitest run server/live-call.e2e.test.ts server/chat-followups-restart.test.ts`：2 个文件，18 个测试通过，耗时 258 秒，机器负载平均值高于 150。上面第二和第三条命令中的 13 个单元文件：404 个通过。
- 自那次运行以来，最终修复波改变了渲染器绘制的内容：记录是一个 `group`，因此快照列出 `group "Live call: …"`，而本次运行看到的是 `region`；步骤后屏幕阅读器听到的词是工具 chip 自己的（"Running a command (Completed)"，本次运行看到的是 "(Done)"）；长行在 pill 内截断；测试装置从未结算的步骤仅在聊天工作时读作运行中，之后读作破折号。

## 最终修复波后重新演练

2026-10-09，同一隔离 fixture（`ui launch`，假 GPT-Live 由同一一次性脚本驱动，无头 Chrome for Testing 153）在 `feat/live-call-record` 上 40099d93，其生产代码为 d81c3b05 的：一次通话，标题关闭，一个语音请求（"what is six times seven"），挂断。

- 该行如之前写入（`Call with Pepper · 0:42`、`seconds: 42`、`hung-up`，语音行携带通话的 `callId`）。`ui snapshot` 列出 `group "Live call: Call with Pepper, 0:42"`，含 `StaticText "Call with Pepper"`、`StaticText "Running a command"` 和 `StaticText "(Completed)"`，无 `region`。截图显示 pill 居中于回复下方，波形字形、标题、`· 0:42` 和一行带勾。
- 宽度，在页面中测量：一行约 300 字符时，pill 保持其 32 rem 限制，内部列表 478 px，该行截断；从列表移除 `max-w-full`（旧标记）后，列表测得 1816 px，从 -108 px 延伸到 1708 px，越过 pill 两侧。
- `text-ink-tertiary`，破折号的颜色，解析为真实颜色（`rgba(252, 252, 252, 0.565)`，对比 pill 自身文本的 `0.6`）。控制台有四个 debug 和两个 info 条目，无错误或警告。
- 此处未驱动，因为假引擎会结算它发出的每次工具调用，且超过 200 条消息的页面手动运行无法触及：无结果的步骤（破折号）、"Expired" 和部分记录提示。组件测试固定它们（`src/components/CallRecordRow.test.ts`、`src/components/ChatView.via-call.test.ts`）。
- 向确切的 `ui launch` pid 发送 SIGINT 删除了其数据目录，驱动被停止；`ps` 显示没有遗留 fixture、预览、驱动或浏览器进程。