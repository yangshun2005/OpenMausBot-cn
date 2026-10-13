# iOS 配套应用架构

iOS 应用是一个轻量原生客户端，连接运行在你 Mac 上的 OpenMausBot 实例。Mac 仍是唯一持久化智能体进程、凭据、SQLite 数据、对话记录和计算机的机器。iPhone 通过扫描桌面端 **设置 → 远程访问** 中显示的二维码来信任 Mac，无需自己的 OpenMausBot 账户。

## 在手机上使用 My Cloud

使用 MausBot Cloud 时，将手机与你的 Cloud 配对而非电脑，这样电脑关机后仍可继续工作。

1. 在桌面应用中，打开你名字下的菜单，选择 **获取手机应用**：iPhone 选 App Store，Android 选 APK。用手机相机扫描其代码。
2. 在同一菜单中，选择 **连接你的手机 · 到你的 Cloud（始终在线）**。（**设置 → MausBot Cloud → 在手机上使用 My Cloud** 效果相同。）应用会在其窗口中打开你的 Cloud，进入 Cloud 的手机配对页面。
3. 选择 **创建配对码**，用手机应用扫描二维码。

在电脑上，**连接你的手机** 会优先提供你的 Cloud，其次才是本机——前提是你的套餐已付费且 Cloud 已就绪。Cloud 就绪前，它只提供 *到本机*，并注明 Cloud 稍后会显示在那里。当应用窗口中打开的是 Cloud 时，条目显示为 *到你的 Cloud* 并直接与其配对。在其他服务器上则与该服务器配对，且仅对允许在该处生成配对码的会话显示。

## 当前状态

第一版包含：

- 从桌面端 **设置 → 远程访问** 进行二维码优先配对，连接前在 iPhone 上确认计算机名称。
- 桌面端所有者启用后，默认二维码使用托管 HTTPS；专用 Tailscale 和受信任本地二维码路由仅在明确选择后启用。
- 二维码路径不可用时，**其他连接方式** 下提供附近计算机、手动地址和六位数字码。
- 安全的按设备信任、设备列表和吊销。
- 多台已保存计算机，一键切换。每个配对有自己的 Keychain 凭据，同一时间只连接选中的计算机。
- 机器人和房间列表、分页对话记录、发送、中断和未读状态。
- 审批和提问，包括窄范围的"始终允许"授权。
- 可恢复的 SSE、流式回复文本、重连水合，以及可选的实时 Boat 计算机视图。仅回环的 VPS SSH 查看器仍为桌面端专属。
- 按需获取机器人 Local VM 的静态截图（空闲或工作中），供计算机所有者已允许计算机访问的手机使用。
- Markdown 渲染，以及手机配对信任的 Keychain 存储。
- 使用 iOS 密码自动填充和二维码固定的 HPKE 加密，安全完成受支持的凭据请求卡片。Apple 密码/iCloud 钥匙串即可；1Password、Bitwarden 等其他自动填充提供方为可选。

应用打开时或移至后台后保持连接的短暂期间内，提醒可正常工作。一旦 iOS 挂起或关闭应用，新提醒无法送达。关闭应用的推送投递、语音和 App Store 发布自动化不属于本版本。可选的托管传输连接到用户自己的计算机；它不是云端对话记录存储，也无法唤醒已终止的 iOS 应用。

Mac 必须运行 OpenMausBot 且不能休眠。桌面端 **设置 → 远程访问** 提供默认关闭的 **保持此计算机唤醒** 开关，在设备访问开启时阻止系统休眠；显示器仍可关闭。休眠或关机的计算机无法接收手机请求或运行本地例程，包括通过可选托管传输。

## Markdown

机器人和房间的回复会被渲染。用户输入的消息保持原样。

渲染的块包括段落、标题、列表、任务列表、引用、分隔线、围栏代码，以及缩进不超过三个空格的顶层表格。列表项、块引用或围栏内的表格保持为文本或代码，缩进四个空格或一个制表符的行也是如此。行内强调、代码、删除线和链接使用 Foundation 属性文本。表格在气泡内水平滚动。任务项显示复选框，有序任务在复选框旁保留其编号。

应用内朗读形式会读出每个表格行和每个任务的文字，跳过仅含分隔符的段落和复选框标记。关闭应用的语音不属于本版本。

## 运行时架构

```text
 iPhone (pairing trust in Keychain; credential plaintext only while editing)
       │                         │
       │ trusted LAN/Tailscale   │ optional hosted HTTPS
       ▼                         ▼
 sidecar :8810          Cloudflare Tunnel (outbound connector)
                                 │
                                 ▼
                        guardian gateway 127.0.0.1:8812
                                 │ exact per-launch socket/pipe
                                 └──────────────┐
                                                ▼
 companion sidecar (pairing auth, default-deny allowlist,
 response/SSE scrubbing, authenticated endpoint refresh;
 credential ciphertext only)
            │ loopback only
            ▼
 OpenMausBot harness :8799
   HTTP API + event stream
   agent processes and approvals
            │ private Electron utility-process channel
            ▼
 OS-encrypted desktop credential store
            │
            ▼
 SQLite message store + local configuration
```

有三个刻意分离的信任面：

| 信任面 | 绑定 | 用途 |
|---|---|---|
| Harness | `127.0.0.1:8799` | 现有应用 API；保持仅回环 |
| Companion | `0.0.0.0:8810` | 已配对的原生设备；需认证且在允许列表中 |
| Companion 控制 | `127.0.0.1:8811` | 开始配对、取消配对、列出设备、吊销 |
| 托管网关 | `127.0.0.1:8812` | Guardian 拥有的、指向某个确切 sidecar 代次的路由 |

桌面应用通过 `electron/companion.mjs` 管理 sidecar 生命周期。渲染进程只接收窄范围的 IPC 操作；它不直接访问控制端口。

## SQLite 兼容性

SQLite 不会移到手机上。它是 harness API 背后的实现细节：

- `server/message-db.ts` 和 `server/store.ts` 持久化和分页对话记录。
- 手机请求 `GET /api/bots?messages=50` 和 `GET /api/threads/:threadId/messages?before=…&limit=50`。
- 因此 SQLite 排序和游标在服务器边界测试，而 Swift 包使用通过真实 sidecar 捕获的响应测试解码和前置/去重。
- 存储迁移可能改变磁盘上的字节而不改变应用。如果 API 载荷变化，用 `node scripts/capture-companion-fixtures.mjs` 重新生成夹具并审查差异。

Sidecar 将设备注册表保存在 `~/.openmausbot/devices.json`。这是网络边界拥有的安全状态，不是对话记录数据，因此不属于消息数据库。

## 连接性

### 同一 Wi-Fi

同一 Wi-Fi 下二维码仍是主要路径。若不可用，用户可打开 **其他连接方式**，选择附近计算机或输入桌面端 设置 → 远程访问 中显示的地址。用户打开该回退方式前，附近发现不会运行。

附近发现使用 Bonjour 和直接 LAN 流量。仅在信任的网络上使用。

因此，选择附近计算机或手动输入 LAN 地址是明确的回退方式。一旦应用使用托管或 Tailscale 路由，自动重连会保持在那些受保护的传输内。回到直接 LAN 需要再次选择该计算机或地址。

### Tailscale

Tailscale 是离家在外以及客户端隔离的 Wi-Fi 网络上的可选路由。在桌面端 **设置 → 远程访问** 中，主卡片仍是 **安全 HTTPS 配对 — 推荐**。单独的 **Tailscale 配对** 卡片仅供已使用 Tailscale 的用户。在两台设备上安装或打开 Tailscale，登录同一 tailnet，保持 MagicDNS 启用，然后选择 **开启设备访问并检查**，再选择 **通过 Tailscale 配对**。第一个操作会明确启动远程访问，让手机有监听器可连。OpenMausBot 随后将计算机的 MagicDNS 名称放入该专用二维码；它绝不会静默替换默认的托管 HTTPS 路由。手动输入仍可作为回退。

URL 仍是 `http`，但路径由 tailnet 内的 WireGuard 加密和认证。使用 MagicDNS 名称而非 `100.64.0.0/10` 地址：App Transport Security 例外基于域名，`ios/project.yml` 仅对 `ts.net` 子域窄范围允许不安全 HTTP。

Tailscale 为可选。直接路径不使用 OpenMausBot 运营的中继，也不创建本地对话记录数据的云端副本。

### 可选托管 HTTPS

在桌面端 **设置 → 远程访问** 中，**随处使用你的手机** 接受免密码邮件验证码，并为该计算机配置一个 HTTPS 地址。此桌面登录仅用于托管 HTTPS。iPhone 从不登录；它通过相同的配对二维码信任计算机。附近、手动和 Tailscale 连接无需账户即可继续工作。

桌面端运行到 Cloudflare 的出站连接器，因此无需入站路由器配置或安装 Tailscale。托管地址仅在就绪后才包含在配对邀请中。默认设置等待该 HTTPS 地址，而非静默替换为 Tailscale；Tailscale 配对仍在其独立可选卡片中明确选择。

地址仅在需要时保留。桌面端在你登录、选择重试或开始配对时创建它（"正在创建安全远程访问"加载指示器下需几秒），在任何手机或桌面端配对期间保留，并在远程访问显示无配对且无进行中配对后一到四小时归还。下次配对会拿回同一地址。若无空闲 HTTPS 地址，面板会提示何时重试，Wi-Fi 或 Tailscale 配对仍可用。

Cloudflare 终止并代理到连接器的加密连接。OpenMausBot 控制平面在 D1 中存储账户和安装元数据以及不透明的隧道/DNS 标识符，但不存储机器人、对话记录、审批、屏幕帧、配对令牌或连接器令牌。数据和删除详情见 `docs/ios-privacy.md`。

连接器不指向可复用的 LAN 端口。Electron 启动一个私有 sidecar 套接字（或 Windows 管道）和一个 guardian，后者拥有固定的回环网关和 `cloudflared`。若 Electron 或该 sidecar 退出，guardian 先使转发不可用，确认连接器已死，然后才释放网关。之后绑定本地端口的其他进程无法继承公共路由。

## 配对与设备安全

1. 在 Mac 上，打开 **设置 → 远程访问** 并选择 **配对一个手机**。应用会在设置过程中启动设备访问。
2. 在 iPhone 上，选择 **连接我的电脑** 并扫描二维码。
3. 确认计算机名称和显示的传输方式——**HTTPS 连接**、**Tailscale 连接** 或 **受信任本地连接**。手机将信任安全存储在 Keychain 中；无需 iPhone 账户。相机二维码还会固定该计算机的公共凭据加密密钥。手动和旧配对仍可聊天，但在手机上输入密钥前必须扫描新二维码。
4. 若无法扫描，打开 **其他连接方式** 使用附近计算机、手动地址或六位数字码。
5. 在 Mac 上吊销手机将移除其访问权限，并允许其重新配对。

要添加另一台 Mac，在 iPhone 上打开 **设置 → 计算机 → 连接另一台计算机** 并扫描该 Mac 的二维码。若新配对失败或被取消，现有计算机仍可用。切换计算机会替换实时事件流和内存中的聊天状态，但保留所有已保存配对；移除一台计算机只从手机删除该计算机的 Keychain 凭据。应用升级会自动迁移之前的单个已保存配对。

Mac 必须保持唤醒且运行 OpenMausBot，聊天、审批和例程才能工作，包括通过托管 HTTPS 或 Tailscale。

配对后，手机定期读取已认证、sidecar 拥有的 `GET /api/companion/endpoints` 快照。这让现有手机无需再次配对仪式即可获知新的托管地址——或其撤销。该路由从不触及 harness，仅返回计算机名称和有界的连接源列表。

附近、手动或 Tailscale 连接无需 OpenMausBot 账户。仅桌面端所有者在启用可选托管 HTTPS 路由时登录；iPhone 始终使用相同的二维码信任流程。

### 安全凭据输入

这是密码自动填充，不是密码库集成。标记为密码的原生 `SecureField` 让用户明确选择 Apple 密码或任何已启用的第三方自动填充提供方。OpenMausMobile 不枚举密码库、不接收提供方令牌，也不将输入值保存在自己的 Keychain 中。

打包的桌面端创建一个稳定的 P-256 接收方密钥对，并将私钥 JWK 保存在操作系统加密的凭据文档中。只有 65 字节的未压缩公钥点被加入相机二维码。手机随连接验证并固定该点；手动和较旧配对仍可完全用于聊天，但在扫描新二维码前无法提交凭据。

凭据提交仅在手机使用托管 HTTPS 或 Tailscale 路由时启用。本地和 Bonjour 聊天配对仍照常工作，但其明文 HTTP 传输会将可复用设备令牌暴露给任何能观察该 Wi-Fi 网络的人，因此应用引导用户在计算机上完成凭据请求。

每次提交使用 RFC 9180 基础模式 HPKE，采用 P-256/HKDF-SHA256/AES-GCM-256，并认证以下确切换行分隔的上下文：

```text
openmausbot-phone-credential-v1
<key id>
<authenticated companion device id>
<bot id>
<thread id>
<message id>
<credential target>
<one-time request key>
```

Companion 将任何调用方提供的设备头替换为其认证的 bearer 令牌身份。Harness 仅接受确切待处理卡片上的允许列表目标，通过其私有 Electron 通道打开密文，并等待加密桌面存储和实时配置提交后才将卡片标记为完成。重放是幂等的；将密文移到另一设备、机器人、任务、卡片、目标或计算机会导致认证失败。本地加密后立即清除原生字段。若响应中断，应用仅在内存中保留该确切密文并复用于重试，因此 HPKE 随机性不会将网络重试变成第二次凭据写入。手机上不持久化任何内容。开发服务器刻意没有接收方密钥，会失败关闭而非宣传不可用的安全输入路由。

面向设备的套接字在读取令牌前拒绝浏览器 `Origin` 头。其在 `companion/src/routes.ts` 中的路由策略为默认拒绝：新的 harness 路由在刻意添加前不可达。

第一版允许：

- 读取机群、房间、实例、配置状态和对话记录。
- 获取已定格的屏幕图像，并选择加入实时屏幕帧。
- 仅当计算机所有者已为此特定配对手机启用该能力时，请求新的交互式云桌面查看器。
- 在相同能力下，获取机器人 Local VM 的静态截图（`POST /api/bots/:id/local-computer/screenshot`）。应用始终传递对话的 `threadId`，因此 harness 对不在 Local VM 上的对话返回 409。这读取图片；无法启动、停止或移除 VM。与桌面面板预览一样，每次捕获都算作使用 VM，因此打开的计算机视图会阻止其被作为空闲回收；离开视图或将应用置于后台会停止。
- 在相同能力下，驱动 Local VM：在设备控制租约下接管机器人的计算机（`POST /api/bots/:id/computer/control`），然后 `POST /api/bots/:id/local-computer/join?controlLeaseId=…`。Harness 仅对回环调用方、仅对 Local VM 上的对话、且仅对持有计算机的租约应答。Sidecar 将回环 noVNC 地址重写为其设备范围的 relay 路径，然后每隔几秒询问 harness 该租约是否仍持有（`action: "check"`，只读），不再持有时关闭 relay。手机通过 relay 使用 RFB。交还会关闭查看器并释放租约，将应用置于后台亦然。
- 直接与服务器配对的手机（`openmausbot serve` 位于 Tailscale Serve 或隧道后，无 sidecar）以相同方式驱动 Local VM，但 join 以服务器自己的已认证桌面代理（`/api/desktop-viewer/local/<target>/websockify`）应答，绑定到其控制租约和 VNC 密码；绝不用回环地址。代理每隔几秒重新检查租约和会话，任一失效即关闭套接字；交还会立即关闭。计算机访问是配对的范围：完全访问（`openmausbot pair`）可以，仅聊天（`--client`）返回 403，应用将计算机访问显示为关闭。
- 手机控制需要按机器人的 Local VM。共享和池模式在两条连接路径上均被拒绝，直到查看器能针对其他机器人预留其桌面；仅对单个机器人的控制持有无法做到。静态截图在所有模式下仍可用。
- 发送消息、中断机器人、回答审批/提问，并将聊天标记为已读。
- 创建基本机器人。
- 将受支持的待处理凭据卡片作为 RFC 9180 HPKE 信封提交，绑定到已认证的设备、机器人、任务、消息、请求和目标。只有打包桌面端的内嵌服务器和 Electron 私有进程通道接收私钥或明文；sidecar、托管中继、聊天记录和 SQLite 存储仅见密文或状态。

写入面使用专门构建的 `read` 和 `always-allow` 端点。通用机器人和房间 `PATCH` 端点无法通过 sidecar 访问。始终允许请求仅当其服务器签发的密钥仍在该机器人的待处理审批上时才成功，因此拥有设备令牌不足以凭空创建宽泛的执行授权。

刻意拒绝：

- 读取凭据、任意凭据目标和通用提供方配置。唯一的凭据写入是上述确切待处理卡片信封，它馈入现有桌面端操作系统加密保存路径。
- 配对、设备吊销或 companion 生命周期控制。
- Local VM 生命周期、webhook、连接器、例程、团队导入/导出和内部对等智能体路由。
- 云计算机配置、休眠、shell 执行和截图 API。手机仅接收新的 `join` 查看器 URL，绝不接收提供方密钥。
- 未经设备访问审查的新 harness 路由。

## 流与状态模型

`CompanionCore` 包含线路模型、客户端、原始字节 SSE 解析器和纯状态折叠。SwiftUI 目标仅拥有生命周期和呈现。

连接时，服务器发送 `hello` 帧，包含游标以及请求的间隙是否被重放。客户端：

1. 从其最后的 `<streamId>:<seq>` 游标恢复；
2. 间隙可用时