# OMB Cloud：家用机器

OMB Cloud（Personal、Pro 和 Max 套餐）为个人提供一台专属的常开 OpenMausBot 服务器。各套餐在机器规格、磁盘和包含的额度上有所不同；本页所有内容对它们都适用。每位客户获得一个 Fly 应用，内含一台常开的 `home` 机器、一个挂载在 `/data` 的卷，以及 `https://<app>.fly.dev` 上的 TLS。桌面应用、手机和网页都是它的窗口。本地使用应用的方式不变，且免费。客户在各处（应用、管理端页面、邮件）看到的名称都是 **MausBot Cloud**；"OMB Cloud" 是本页的简称，应用本身仍叫 OpenMausBot。

OMB Cloud 不包含 AI 用量。用户在自己的机器上使用自己的 Claude 或 ChatGPT 订阅，或 API 密钥登录，登录流程与任何 OpenMausBot 服务器相同。Cloud 家用机器上的任何内容都不会路由到平台模型网关。唯一的例外是免费试用的 Claude 额度——管理端在自己的中继上计量的一笔小额一次性额度，仅在用户接入自己的 AI 之前使用（[试用 Claude 额度](#试用-claude-额度)）。

本页是三方契约中 OpenMausBot 这一半：

- **家用机器**：本仓库的 `cloud-home` 镜像（`Dockerfile`、`deploy/fly/`）；
- **管理端**（openmaus-cloud，其 `docs/consumer-cloud.md`）：负责开通应用、持有机器的签名密钥，并响应桌面端的 Cloud 会话；
- **桌面应用**：登录 Cloud，在 Servers 下列出该机器，并提供 **Open My Cloud**。

契约版本：`1`（传输层为 `cloudContractVersion`）。

## 用户看到什么

1. 用户在 Cloud 网站订阅。管理端创建 Fly 应用和机器。
2. 用户打开桌面应用，进入 **Settings → MausBot Cloud** 并登录（现有的设备登录）。**Your Cloud** 卡片显示 **Setting up**，直到机器就绪。
3. 就绪后，机器以 **My Cloud** 出现在 **Servers** 下，卡片提供 **Open My Cloud**。一键即可在应用窗口中打开该机器并已登录，无需二次确认。
4. Cloud 首先问的是 **What should your Cloud do while you're away?**（`src/components/CloudIntent.tsx`、`src/lib/cloud-intent.ts`）：一个外观类似输入框的框，文本前固定有 `/setup`，四个可填充它的想法（新闻摘要、监控某个页面、研究综述、每日计划）；悬停某个想法会在框中预览其完整文本。它绝不是一道关卡：**Skip for now** 就明摆在想法下方，框为空时按 Escape 可跳过，在侧边栏选择另一个 bot 会跳到该 bot，把问题留给设置卡片的 **Give it a job**。回答以 `/setup <job>` 发送（server/setup-mode.ts），因此 bot 最多问四个问题，并在聊天中为这项工作完成自我设置。如果已有引擎可运行，则立即发送；框在过程中变为输入框（一个具名 View Transition），请求成为聊天的第一条消息。否则该工作在此设备上等待（浏览器存储中的 `omb.cloudIntent.pending`），引擎登录界面会在顶部显示它，并带 **Edit**。Cloud 在其 onboarding 记录中保留该问题已被回答（`cloud-intent-asked`）以及已给出工作（`cloud-intent-given`），因此没有设备会再问一次；已经有一个 bot 完成过一轮的 Cloud 永远不会被问。该问题只对 Cloud 家用机器上所有者自己的会话提出。
5. 然后，在未登录引擎的情况下，进入引擎登录界面（`src/components/CloudEngineSignIn.tsx`）：提供商磁贴并排显示（两个，或镜像带 Grok CLI 时三个），选中的那个在下方面板中打开其登录，下方是 API 密钥：
   - **Claude**：现有的粘贴代码流程（打开 Anthropic 页面，把代码粘回来）；
   - **ChatGPT**：现有的 Codex 设备代码流程；
   - **Grok**：grok.com 订阅上 Grok Build 的同一设备代码流程（`grok login --device-auth`，在 Cloud 计算机上运行），仅在镜像带 Grok CLI 时提供；
   - **Use an API key**：**Settings → API keys** 中现有的模型提供商密钥（Anthropic、xAI，或 OpenRouter 等 OpenAI 兼容密钥）。

   它会明确说明：用户的 AI 套餐限制适用于全天候运行的 bot，且 Anthropic 的 Claude Max 套餐或 API 密钥最适合重度使用。
   当等待中的工作的引擎可运行时，屏幕会短暂显示 **Connected. Starting your job…**，选项退去，工作如上所述进入聊天。
6. 在这些引擎之一可运行之前，Cloud 上的每个 bot（包括默认 bot）都显示此登录界面，而不是一个首轮就失败的聊天。一旦有一个可运行，聊天便取而代之。登录信息保存在机器的卷上（`~/.claude`、`~/.codex`、`~/.grok`、服务器自身的配置）。在没有 Grok CLI 的旧镜像上，Grok 的设置卡片会用一行说明，并提供 xAI 密钥作为替代。

Cloud 的 `GET /api/auth/session` 对已配对的会话返回 `"cloudHome": true`；网页 UI 据此知道要打开引擎登录界面，而不是描述用户自己电脑的欢迎流程（后者仍可从 Settings 重放）。

### 在手机上使用 My Cloud

1. 获取手机应用：你名字下的菜单 → **Get the phone app**（iPhone 用 App Store，Android 用 APK）。
2. 同一菜单 → **Connect your phone · to your Cloud (always on)**，或 **Settings → MausBot Cloud → Use My Cloud on your phone**。Cloud 会在应用窗口中打开其手机配对页面。
3. **Create pairing code**，用手机应用扫描二维码。

它如何协同工作（`src/lib/phone-pairing.ts`）：

- **Connect your phone** 打开 Settings → Remote access，定位到适合窗口的配对，焦点在显示代码的按钮上：桌面应用在其自身电脑上是本机的手机流程，Cloud 家用机器上是 Cloud 自己的配对码（`ServerPairingCard`），其他服务器的配对码仅对可以生成它的会话显示（该机器上的所有者或管理会话，此时配对码开启）。
- 在本机上，当已验证的快照显示付费套餐（任意档位）且 Cloud 就绪时，菜单有两行 **Connect your phone**：先是 *to your Cloud (always on)*，其行为与 **Use My Cloud on your phone** 相同，然后是 *to this computer*。付费套餐但 Cloud 未就绪时，只保留单行 *to this computer*，并附注 Cloud 会显示在那里。切换失败会打开 Settings → MausBot Cloud。
- **Use My Cloud on your phone** 对付费套餐显示。Cloud 就绪时，它调用 `cloud-account:connectHomeForPhone`，该调用不带参数，连接方式与 **Open My Cloud** 相同，并附加一个固定请求 `?desktop-settings=phone`（在 `/pair` 上也一样，配对后会携带它）。Cloud 的页面打开其手机配对设置。它从不自行生成代码。在 Cloud 就绪之前，或打开失败时，卡片改为列出两个步骤。在 Cloud 本身上，Settings → MausBot Cloud 提供相同的按钮并直接打开配对。

### 仅限你自己的设备

Cloud 家用机器是私人的（`server/cloud-owner.ts`）：只有所有者自己的设备能连接（桌面应用、手机、从 Cloud 页面登录的浏览器），每个设备都持有一个管理会话，由管理端的签名配对或这些设备之一授予。服务器不铸造也不接受其他任何东西，并用一行说明："My Cloud is personal: only your own devices can connect."：

- `POST /api/auth/pairing` 拒绝没有管理范围的窗口（Remote access 在那里不提供仅聊天的选项），且 `POST /api/auth/pair` 和 `POST /api/pair` 从不兑换它；
- 邮箱登录（`/api/auth/email/start`、`/api/auth/email/verify`）关闭，Settings 的 People 部分及其邀请也消失（`PUT /api/config` 中的 `signIn` 列表会被拒绝）；
- 没有管理范围的会话不进行认证（其事件流票据也不认证），已配对设备列表只显示所有者的设备。

每次启动时，任何存储的没有管理范围的会话都会被撤销（日志会说明数量），它打开或写入的内容不属于任何人。它的密钥在任何其他事情发生之前就写入 `cloud-owner.json`，因此任何崩溃或保存失败都不会让它变成所有者的。

此前，每个设备携带自己的密钥（v0.1.91），因此所有者取消配对的一个设备会让所有者自己的对话被读作他人的。在个人 Cloud 家用机器首次启动时，以及每次恢复之后，此前命名的密钥会分两层被确定：

- **Adopted（已采纳）**：每个打开过对话、写入过一行或回答过卡片的密钥，被撤销的除外。其对话和房间按打开者归属所有者，连同其审批级别（Auto 或 Full 保持不变）及其文件夹。只写入过一行的设备也算。
- **Proven（已证实）**：只有能证明是所有者本人的密钥：在启动时仍处于配对状态的其设备，或回答过卡片的设备（在 Cloud 家用机器上只有所有者的设备能）。只有已证实密钥的发言才会进入借用的 Mac、记忆捕获、回忆和近期工作简报。

这对无法得知的事情是诚实的。所有者在升级前取消配对的访客（或其会话已过期）无法与所有者的旧设备区分，因此其对话被采纳：它们以 bot 的级别运行，在其文件夹中。但其发言永远不会进入借用或记忆，任何包含其发言的对话都被排除在两者之外。升级时仍处于配对状态的访客会被撤销，并保持不属于任何人。

例程默认拒绝。例程只有在有证明时才属于所有者：所有者作为其写入者的密钥、其上的所有者指纹（他们从自己设备编写的例程），或所有者发起的恢复（Copy this computer here，或在 Settings 中恢复的备份）；从所有者的 bot 模板创建的例程，或所有者批准的提案，在创建时即记录为其所有。其他所有例程都不属于任何人：它像访客的一样受限运行，并报告到一个不属于任何人的对话中。所有者的例程报告到一个属于所有者的对话中：bot 的主线程，只有所有者打开的对话才能是。

Webhook 属于所有者：只有他们自己的设备能创建、编辑或轮换它，因此其运行以 bot 自身的级别、在其项目文件夹中、用其 shell 进行，与桌面端相同。Webhook 带入的内容仍永远不会进入借用的 Mac，无论它是启动一次运行还是发布到聊天（见下文）。

被撤销会话编写的例程在启动时暂停，它打开的对话失去其工作文件夹：其下一轮在它自己的文件夹中工作，绝不是所有者的项目。

一个最终不属于任何人但本属于所有者的例程（例如 v0.1.91 的 bot 模板在服务器上创建的、未记录写入者的例程）受限运行，无法受限的一次运行会说明：打开它并保存一次，它就重新属于所有者，拥有完全访问权限。所有者在 bot 的提案卡片上批准的例程属于他们（卡片记录谁允许了它，以及它展示了什么：此前的例程仅当它仍精确地按此运行——相同的指令、bot、计划和位置，且无附件——才算数），在所有者自己的 Full-access 对话中立即创建的例程，或由所有者某个例程的一次运行创建的例程，也是如此。批准一项更改（编辑、暂停、恢复）仅在例程本已属于所有者时才保持其归属：它绝不会让他人的例程变成他们的。bot 从任何其他对话立即创建或更改的例程不属于任何人，如同其他人的编辑一样。恢复、移动或重新定时一个例程，当所有者是其写入者且其上尚无他们的指纹时（模板、恢复），保持其归属；一个不再与其运行内容匹配的指纹，只有所有者重写其指令才能更新。例程的每次运行都在一个像其结果对话一样打开的对话中工作，因此不属于任何人的例程的运行被限制在它自己的文件夹中。

一个被采纳但未证实的密钥仍有代价：其发言使该对话被排除在借用和记忆之外，如果那里的一轮更改了 bot 的记忆文件，整个 bot 在所有者审查该更改之前无法使用 Mac（bot 的 **Memory** 面板，**Mark reviewed**）。开启新对话可避免此问题。

在首次个人启动时，日志还会提示审查 **Settings → Remote access → Paired devices**，它现在只显示管理设备，并登出任何非所有者的设备：此前以完全访问配对的设备从此属于所有者，无法辨别为其他。

`cloud-owner.json` 仅属于此机器：它从不进入备份，恢复会将其留在原处并确定它带来的内容（它记录它确定的最后一次恢复，因此在提前结束的启动中应用的恢复会在下一次确定）。恢复仅对报告到一个尚未命名任何人或命名所有者的对话的例程构成证明：此前的备份可能包含访客的例程，它保持不属于任何人。下面的访客规则对访客留下的内容保持默认拒绝。

卡片显示以下之一：**Setting up**、**Ready**、**Stopped**、**Payment problem**、**Could not be set up yet**。只有 Ready 才能连接。登出 Cloud 后，应用不发出任何 Cloud 请求，本页内容都不运行。

### 设置清单

在 Cloud 家用机器上，一张小卡片 **Set up My Cloud** 位于左下角，直到其步骤完成或用户隐藏它（`src/components/CloudSetup.tsx`、`src/lib/cloud-setup.ts`）。只有所有者自己的设备（Cloud 家用机器上的管理会话）能看到它；桌面和自托管安装永远看不到，并保留其欢迎流程。当前要做的步骤领衔卡片，其说明和操作在其下，每个步骤一个分段，完成时填充。其他每个步骤在其下方一行显示；未完成的步骤点击后成为领衔项。折叠箭头仅在此设备上将卡片最小化为其分段（浏览器存储中的 `omb.cloudSetup.collapsed`），这不算隐藏。当最后一个必需步骤在打开的会话中完成时，卡片一次性让位给 **Your Cloud is ready**，带指南吉祥物；一个打开时已完成一切的 Cloud 不显示任何内容。每个步骤的状态来自 Cloud 或应用，绝不来自用户勾选的框：

1. **Create your Cloud**：从一开始就完成；Cloud 存在。
2. **Give it a first job**：一旦给第一个问题给出了工作（`cloud-intent-given`），或一旦某个 bot 的一轮在 Cloud 上完成，即完成：服务器仅在 Cloud 家用机器上，为在 bot 的对话或房间中完成的一轮（非失败或停止的）记录一次 `onboarding.firstTurnAt`。onboarding 记录从不随副本迁移，因此复制进来的聊天不计入。**Give it a job** 会再次提问。
3. **Connect your AI**，唯一必需的步骤：当 Cloud 上任何引擎可运行时完成。从其他视图，其 **Connect** 返回上文的引擎登录界面。
4. **Approve its plan**：一旦给出第一个工作就列出。bot 在聊天中提问并提议一个例程；一旦该例程存在、属于该工作的 bot 且在工作发送后创建，即完成（此设备记住哪个 bot 收到了它，`omb.cloudIntent.sent`；另一设备则计入任何例程）。**Open the chat** 跳转到该 bot。
5. **Bring your bots from your computer**：仅在桌面应用中，当 Copy this computer here 卡片会被提供时（空的 Cloud、有工作要带来的电脑；docs/copy-workspace.md）。**Copy to My Cloud** 就地打开该提供（大小、保留内容、**Copy** 和 **Not now**）。复制后完成；**Not now** 后跳过，Cloud 会保留它（其 onboarding 记录中的 `cloud-setup-move-skipped`），这也会隐藏那张一次性卡片。

工作一发送即计为已给出，早于 Cloud 的记录响应，因此没有步骤会闪回。当主窗格已显示领衔的步骤（问题，或聊天视图中的登录界面）时，卡片显示 **You're on this step** 而不重复它。当所有列出的步骤都完成但尚无一轮完成时，卡片说 bot 正在聊天中自我设置。将此 Mac 借给 Cloud 不是一个步骤；它留在 Settings → MausBot Cloud 和菜单栏项的 **Lending settings…** 中。

**Hide setup** 是唯一的关闭方式。Cloud 保留它（其 onboarding 记录中的 `cloud-setup-hidden`），因此它在每个设备上、在浏览器存储被清除后都保持，它也是该移动的 **Not now**。一旦引擎可运行且 Cloud 做过某事，卡片也会自行消失：给出第一个工作时，其例程存在（bot 的首轮只提问，因此完成一轮还不够）；没有第一个工作时，某个 bot 在那里完成过一轮。有例程时，**Your Cloud is ready** 会命名它及其下次运行，**Run it now** 立即启动它，使首个结果无需等待计划。没有任何确认请求。
卡片之后，一次性的 Copy this computer here 卡片的行为与任何其他服务器相同。

当窗口显示 Cloud 家用机器时，侧边栏的服务器切换器显示 **My Cloud · always on**；在浏览器中，一个普通标签显示相同内容。

### bot 在哪里工作

Cloud 家用机器是无头 Linux 服务器，因此其 bot 有两个位置：内置浏览器和云计算机。它从不提供 **This computer**（那会是服务器本身）或 **Local VM**（Fly 机器没有容器运行时）。用户的 Mac 仅在他们借用时（**Let My Cloud use this Mac**，见下文）通过共享计算机工具访问。

- 两个位置都不列在 Computer 面板、输入框的位置芯片、bot 的 Works on 设置或 Settings → Computers 中（配置返回 `"cloudHome": true`），也不在 `select_computer` 中，后者还会去掉 `vm_exec`。Auto 永远不会落到两者之一。
- 仍设置为两者之一的 bot（较旧或导入的设置）的每个任务都会被拒绝，并附一句说明，建议 Auto、Cloud 或 Browser，对于 This computer 则建议借用 Mac。
- 每一轮的系统提示都说 bot 在云中运行。当被问及用户自己的电脑时，带共享计算机工具的 bot 会检查是否有借用的 Mac，若没有则说明并告知如何借用；没有这些工具的 bot 则说无法访问。两者都