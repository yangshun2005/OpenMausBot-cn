# 规范：aside 通道的按驱动器中轮接缝

状态：生效。属于对等消息传递总括（#1803）的一部分；aside 通道本身（准入、持久队列、回执、降级到队列）随 S2 第 1 部分落地，本文档是每个引擎的驱动器在 harness 向其提供中轮 aside 时所遵循的契约。

## 接缝承载什么

aside 是交给线程正在运行的轮次的对等上下文——绝不是新轮次，绝不是中断，绝无操控权。harness 将封装后的文字持久入队（消息 id = 去重键），随后在驱动器声明 `capabilities.queueing` 时，对运行中的实例调用 `Adapter.steer(threadId, prompt)`。任何非 `refused` 的结果下，harness 为每个 aside 精确追加一行转录记录（封装文本、`aside: true`、对等来源、队列 id 标记）并注销持久行；`refused` 将文字退回队列等待下一个边界。`indeterminate` 计为已注入——文字可能已在运行，而重复运行正是该通道绝不能犯的错误。

转录记录是每个接缝的持久性契约：已注入的 aside 可能被后续压缩摘要掉（pi 会这样做），但记录让发送者和框架对任何后续读者保持可见，队列 id 标记使重启恢复跳过已注入的文字。

## 按驱动器契约

| 驱动器 | 中轮接缝 | 行为 |
| --- | --- | --- |
| codex | 带 `expectedTurnId` 的 `turn/steer` RPC | aside 随操控通道携带强制的非操控封装；轮次已推进导致的 RPC 错误映射为 `refused`（重新入队），投递后超时映射为 `indeterminate`。 |
| claude | stdin 用户消息（`writeUser`） | CLI 在下一次模型调用前投递；写入失败为 `refused`；绝不为了操控而杀死子进程。 |
| openai-chat 系列（openai-compat、grok、mistral、minimax） | 自有工具循环 | `steer` 将文字暂存于当前轮次的入口；下一轮循环开头将其作为用户消息拼入请求数组——这是数组处于轮次之间的唯一位置，绝不在工具批次中途。暂存的文字在该边界前不进入请求数组。 |
| pi | 原生 RPC `{type:"steer", message}` 帧 | 运行时将文字排入运行中的 agent，不中止进行中的步骤，接受后应答 `success`；`success:false` 是显式拒绝（`refused`），帧写入后死亡/超时为 `indeterminate`。 |
| ACP | **无——由协议决定，而非遗漏** | 见下文。 |

## ACP：设计上仅队列

ACP 没有中轮用户输入。一个轮次是一次 `session/prompt` RPC，流式发送 `session/update`，并在 RPC 结果解析时结束；协议在中轮接受的唯一输入是 `session/cancel`，它会中止。因此 ACP 驱动器不得：

- 声明 `capabilities.queueing` 或实现 `steer`；
- 中断并重新提示以模拟接缝（这违反 aside 通道存在所维护的非中断不变量）；
- 在提示中途发送规范外的帧并称之为接缝。

正确行为是 aside 通道的内置降级：对忙碌 ACP bot 的对等发送被持久入队，发送者收到诚实的 `queued` 回执，文字在运行中的轮次结束后开启下一轮。这与无接缝引擎一直使用的通道相同；ACP bot 只是从不离开它。仅当 ACP 协议新增中轮输入方法时才重新考虑。

## 为新驱动器添加接缝

1. 仅当引擎真正接受用户输入进入正在运行的轮次时才声明 `capabilities.queueing`——绝不为让 UI 控件出现而声明。
2. 实现 `steer` 时带完整三态；`refused` 必须意味着可证明未投递，否则 harness 将重放可能已运行过的文字。
3. 在引擎已遵循的步骤边界投递暂存的文字；绝不为了注入而中止、杀死或重启任何东西。