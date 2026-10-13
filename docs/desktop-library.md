# 桌面端组织库

当组织向其成员共享包，或合作伙伴向其客户共享包时，Admin 会为每个接收组织构建一份目录。已连接的 OpenMausBot 桌面端会保留该目录及每个包发布文件的已验证副本，因此 **模板 → 来自 {Organization}** 可以显示并添加它们。本页描述该通道中桌面端负责的部分：Electron 主进程。运行时的货架、添加按钮和来源行在运行时部分描述（`server/org-library.ts`）。

该通道不会向任何人询问任何内容。组织提供什么由 Admin 决定；添加包是用户自己的点击操作。没有更新控制：目录始终指向最新发布版本。

## 何时使用

- **无组织账户**：什么都不做。没有请求、没有文件、没有消息。
- **Admin 未启用该库**：同样什么都不做。桌面端从 `GET /api/public/config` 读取 `capabilities.library`，这与它每次启动时用于续期读取的是同一个响应；除非该值为 `1` 或更高，否则不调用任何库路由。启动时无法读取配置的桌面端（离线），会在成功同步后最多每 10 分钟重试一次。
- **Admin 已启用该库**：每次成功的会话同步（`GET /api/desktop/session`，约每分钟一次）都会携带一个小的 `library: { version, digest }` 指针。当 digest 与该桌面端上次应用的目录不同时，主进程会拉取目录。会话本身不携带大内容，保持其 512 KiB 上限。

## 主进程如何处理新指针

1. `GET /api/desktop/library`（最多 256 KiB，20 秒）。目录必须是 **本** 组织的 `openmaus.org-library` 版本 1 信封。格式错误的信封或其他组织的目录会被整体拒绝，保留上一个有效目录。格式错误的条目会被单独丢弃；未知字段被忽略。
2. 对于每个非 **Off**、有发布版本、格式为本应用可读（版本 2 或更低）且文件缺失的条目：`GET /api/desktop/library/blobs/<sha256>`（最多 4 MiB，60 秒，且不超过目录的 `sizeBytes`）。字节必须与目录的 `sha256` 匹配，否则不会写入。失败会使该包不可用，并在后续同步中以递增延迟重试（`429` 会按其 `retry-after` 暂停下载）。已缓存的文件在每次新目录后重新哈希；不再匹配的文件会被重新拉取。每个目录的下载总量上限为 64 MiB。
3. 原始目录字节保存为 `catalog.json`，然后 `{portalOrigin, organizationId, deviceId, libraryVersion, digest}` 经操作系统加密保存到 `company-library.bin`。
4. 主进程将目录转发给本地运行时（见下文）。
5. 应用的 digest 是实际接收字节的 SHA-256，而非指针中的 digest，因此两次请求之间重建的目录会在下次同步时收敛。
6. 目录不再引用的发布文件会在 7 天后删除。

每个库请求都使用设备的 `omd_` 令牌（绝不使用模型令牌），拒绝重定向，不发送 Cookie，并与公司备份使用相同的连接代际检查：登出会中止它。

## 文件

| 位置 | 内容 | 写入方 |
|---|---|---|
| `<app data>/company-library.bin` | 已应用的目录，对应哪个 Admin、组织和设备，经操作系统加密 | Electron 主进程 |
| `<data dir>/org-library/catalog.json` | 上次应用的确切目录字节（仅所有者可读） | Electron 主进程 |
| `<data dir>/org-library/blobs/<sha256>.json` | 发布文件，仅所有者可读，原子写入 | Electron 主进程 |
| `<data dir>/org-library/state.json`、`presets.json` | 此安装添加的内容 | 运行时 |

`<data dir>` 默认为 `~/.openmausbot`，除非 `OMB_DATA_DIR` 另有指定。

工作区和公司备份会排除 `catalog.json` 和 `blobs/`，因为主进程会重新下载它们，且登出会删除它们。它们保留运行时的 `state.json` 和 `presets.json`。

## 启动、登出与重启

- **启动时**，在向 Admin 发出任何请求之前，主进程读取 `company-library.bin`。仅当 Admin、组织和设备均匹配、登录未过期，且 `catalog.json` 仍哈希到记录的 digest 时，才使用保存的目录。否则忽略该文件，首次同步会重新拉取。
- **Admin 许可证失效**或 Admin 不可达不会改变任何内容：保留上一个目录。
- **登出、过期或撤销**会转发 `library: null`（货架消失）并删除 `company-library.bin`、`catalog.json` 及缓存的发布文件。已添加的机器人、房间、例行任务和技能保留。
- **Admin 停止通告该库**按同样方式处理。
- **运行时重启**后，在其就绪时会再次收到已应用的目录。未确认目录的运行时会在下次同步时收到。日志对每个运行时和目录只记录一次，而非每次同步都记录。

## 与本地运行时的消息

Electron 主进程 → 运行时，通过私有实用进程端口（绝不使用 HTTP，绝不使用远程服务器）：

```jsonc
{ "type": "openmausbot:managed-library", "requestId": "…",
  "library": { "adminOrigin": "https://admin.example.com", "organizationId": "…", "organizationName": "Beta Clinic",
               "digest": "<sha256 of catalog.json>", "catalog": { /* the parsed catalog */ } } }
// or "library": null: hide the shelf, change nothing else
```

运行时替换其内存中的目录，并在任何安装工作之前立即应答 `{ "type": "openmausbot:managed-desktop-result", "requestId", "ok": true }`（中继在 15 秒后放弃）。它自行从 `blobs/` 读取发布文件，并在每次读取时校验其 SHA-256。

运行时 → Electron 主进程，主动发送，在其处理完转发的目录后，以及对其添加内容的任何更改后：

```jsonc
{ "type": "openmausbot:managed-library-state", "digest": "<the relayed digest>",
  "packages": [{ "packageId": "…", "release": "1.3.0", "sha256": "…", "state": "installed" }] }
```

主进程逐条检查每个条目（错误的被丢弃，最多 100 条），并忽略 digest 不是本次注册所转发的快照。

## 安装报告

主进程将最新快照 POST 到 `POST /api/desktop/library/report`（`{libraryVersion, digest, appVersion, packages}`，最多 64 KiB）。Admin 仅显示计数。`reason` 来自固定列表；桌面端不发送自由文本、路径或错误消息。报告在以下时机发送：

1. 运行时处理完新应用的目录后（其首个快照）；
2. 对此桌面端添加内容的任何更改后（运行时发送快照）；
3. 每次应用启动后，首次成功会话同步后；
4. 发送失败后，在下次同步时重试，直到成功。

主进程在快照到达后等待 5 秒，只发送最新的一个。

## 兼容性

| 桌面端 \ Admin | Admin 无 `capabilities.library` | Admin 有 |
|---|---|---|
| 此通道之前的 OpenMausBot | 同以前 | 忽略指针，保持连接 |
| 有此通道的 OpenMausBot | 无库请求、无货架；文件导入导出正常 | 货架 |
| 无组织账户 | 同以前 | — |

此通道与运行时对 `openmausbot:managed-library` 的处理程序（`server/org-library.ts`）在同一版本中发布。没有该处理程序的运行时永远不会确认，因此主进程会在每次同步时重新转发，并每次都等待 15 秒超时。

测试方式：[verification/desktop-library.md](verification/desktop-library.md)。