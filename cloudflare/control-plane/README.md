# OpenMausBot 控制平面

本目录是一个独立的 Cloudflare Worker，用于云账户身份、安装归属，以及每个安装的托管配套端点。它**不**存储或传输本地机器人、聊天、桌面 SQLite 状态、提示词或工具输出。

## 包含内容

- Better Auth 1.7.1，支持邮箱 OTP、签名 bearer 会话、哈希 OTP 存储，以及基于 D1 的 IP 和收件人速率限制。
- Cloudflare Email Sending 绑定，同时生成 HTML 和纯文本 OTP 邮件。即使投递失败，认证响应也保持通用；邮箱地址、OTP、密钥和提供商错误绝不记录日志。
- 归属者范围的桌面安装，以及可独立撤销的 `omb_install_…` 凭据。账户 bearer token 绝不接受为安装凭据，反之亦然。
- 精确来源 CORS、有界 JSON 请求体、脱敏错误，以及每个响应上的 `no-store`。
- 每个安装一个远程管理的 Cloudflare Tunnel。其不透明公共主机名路由到 Electron 拥有的网关 `http://127.0.0.1:8812`（绝不使用 `8810` 上可复用的 LAN 监听器），后面跟一个强制的 `http_status:404` 兜底规则。代理 CNAME 指向 `<tunnel-id>.cfargotunnel.com`。
- 基于 D1 的生成/租约声明、通过稳定不透明隧道名恢复，以及可重试的部分清理。Cloudflare API 凭据和原始连接器 token 绝不写入 D1 或日志。
- 一个或多个 Cloudflare 账户中的端点（见**多个 Cloudflare 账户**）。新端点进入空间最大的账户；每个端点终生留在其创建时的账户中。

D1 schema 固定在 `migrations/` 中。`0001_better_auth_1_7_1.sql` 由确切的 Better Auth 配置生成。`0002_installations.sql` 仅包含云归属和凭据元数据。`0003` 添加收件人范围的 OTP 限流器，其键为 HMAC 而非邮箱地址，另加一个需认证的安装创建限流器。`0004` 添加托管端点资源 ID、生命周期状态、生成租约、脱敏错误码，以及安装范围的操作限制。`0005` 添加用于计划重试退避的清理尝试计数器。`0006` 添加空闲回收标记和一行容量快照（仅计数和时间戳），并为运维人员已移至 `deleting` 的活跃安装行提供与自动回收相同的重连保护。`0007` 将每个端点行绑定到持有它的 Cloudflare 账户（`provider_account`，一个账户 ID，其默认值为所有早期行所在的账户，因此在其之前写入的行，或由在其之前部署的 Worker 写入的行，保留其账户），并将容量状态移至每个账户一行（`managed_endpoint_account_capacity`，从 `0006` 行播种，该行保留但不再写入）。端点行故意不随安装硬删除而级联删除：丢失隧道和 DNS ID 会使运维人员无法清理。

## API 接口

| 方法 | 路径 | 认证 |
| --- | --- | --- |
| `GET` | `/healthz` | 无 |
| 任意 | `/api/auth/*` | Better Auth |
| `GET` | `/v1/me` | 账户 bearer |
| `GET`、`POST` | `/v1/installations` | 账户 bearer |
| `POST` | `/v1/installations/:id/credentials/rotate` | 归属账户 bearer |
| `DELETE` | `/v1/installations/:id` | 归属账户 bearer |
| `GET` | `/v1/installations/self` | 安装凭据 |
| `GET`、`POST`、`DELETE` | `/v1/installations/self/endpoint` | 安装凭据 |

安装注册需要稳定的 `clientInstanceId`、显示 `name`，以及 `platform` 为 `darwin`、`windows` 或 `linux`；`appVersion` 可选。客户端 ID 在一个账户的活跃安装中唯一。撤销后，该账户可重新注册该稳定 ID。其他账户可独立使用相同客户端 ID。一个账户最多可有 100 个活跃安装，与完整管理列表上限一致。创建也限制为每账户每小时 100 次尝试。

原始安装凭据包含一个随机查找 ID 加 32 个随机字节。仅存储 SHA-256 摘要，原始值仅在创建安装或轮换其凭据时返回。凭据即使未被撤销也在 90 天后过期；响应包含其过期时间，以便已登录桌面可提前轮换。`/v1/installations/self` 拒绝过期凭据，并记录凭据使用和安装 `lastSeenAt`。轮换以一分钟冷却串行化，因此并发请求不会在一个使另一个失效的同时都返回凭据。

### 托管端点契约

三个端点方法都要求 `Authorization: Bearer <omb_install_…>`。账户 bearer token 被拒绝。

- `GET` 在分配前或删除后返回 `{ "endpoint": null }`。否则返回 HTTPS URL、主机名、生命周期状态、生成、时间戳，以及脱敏的 `lastErrorCode`。绝不返回连接器 token。
- `POST` 无必需请求体。应用可发送 `{ "appVersion": "0.1.104" }`（`application/json`，可打印，最多 64 字符）：其运行的发布版本，记录在安装上，并决定哪些账户可持有新端点（见**多个 Cloudflare 账户**）。无请求体行为与之前完全一致；任何其他成员，或格式错误的请求体，为 `400 invalid_request`，非 JSON 请求体为 `415 unsupported_media_type`。它幂等地保留或协调端点，采用被中断的早期运行创建的隧道/DNS 记录，并返回 `{ endpoint, connectorToken }`。原始 token 仅在隧道配置和 DNS 就绪后获得。调用方必须将其直接放入操作系统的安全凭据存储；它无法从 GET 或 D1 恢复。空闲回收后，同一调用在**同一主机名**后分配新隧道，因此已配对的手机保留其地址；它也可收回回收仍待处理的端点。桌面应用（Remote access 开启）和 `openmausbot serve --tunnel` 每 15 分钟询问 `GET`，即使其连接器报告就绪，并在端点消失或处于 `error` 时发起此调用；`GET` 返回 `401`（90 天安装凭据过期）会使其进入账户恢复，或"登录已过期"提示。
- 当 Cloudflare 的隧道配额（`1045`）或区域的 DNS 记录配额（`81045`）拒绝分配时，`POST` 返回 `503 endpoint_capacity` 并带 `Retry-After: 600`。已满账户也可能以 `429` 响应隧道创建：控制平面随后读取账户的隧道计数（或当该读取失败时，其上次扫描若在 30 分钟内，减去计划清理自那以后删除的隧道），并在达到或超过账户隧道限制时，将 `429` 视为配额（`cf_tunnel_quota`）。在接下来十分钟内（或直到计划清理释放其一个资源），该账户中需要新隧道的进一步分配会以相同方式响应而不调用 Cloudflare，保护共享 API 预算。从未发放地址的端点改为在同一请求中移至另一个有空间的已配置账户（见**多个 Cloudflare 账户**）。来自 Cloudflare API 速率限制的任何其他 `429`（`cf_rate_limited`）返回 `503 endpoint_rate_limited`，Cloudflare 的 `Retry-After` 被限制在 30–300 秒（Cloudflare 未发送时为 60），且不关闭任何东西。所有其他提供商失败保持 `502 endpoint_unavailable`；有多个账户时，Cloudflare 拒绝其 token 的账户也会关闭该账户（见**被拒绝的 token**）。
- `DELETE` 先移除 DNS 再移除隧道。完成时或已删除时返回 `204`。部分 Cloudflare 失败返回 `503 endpoint_cleanup_pending`，仅保留重试所需的 ID。并发变更返回 `409 endpoint_busy` 并带 `Retry-After: 2`。

主机名在持有端点的账户后缀前恰好有一个不透明标签：主账户中为 `c-<32-lowercase-hex>.<COMPANION_HOST_SUFFIX>`。将后缀设为区域边缘证书覆盖的区域名（通常为区域顶点），使端点不依赖深层子域 TLS 覆盖。隧道名是稳定不透明标识符，不含账户邮箱、显示名或客户端提供的 ID。

端点配置限制为每安装每小时 20 次尝试；删除限制为 30 次。60 秒 D1 租约和单调递增的生成串行化并发请求。归属者在每次提供商调用前续订并围栏该租约，因此过期请求无法回滚其后继采用的资源。Cloudflare 调用有每请求五秒超时、拒绝重定向、有界响应体，并在持久化 ID 前验证响应形状。模糊的创建/更新响应通过稳定隧道名和精确 DNS 身份协调。在任何破坏性清理前，存储的 ID 和提供商侧名称/目标都会被重新验证；被重命名或改用途的资源保留给运维人员而非猜测。新创建的部分资源被回滚；采用的资源绝不被失败的协调删除。

撤销安装先撤销其本地安装凭据，然后安排尽力而为的端点清理。云清理失败不能恢复或延迟凭据撤销。重复归属者范围的安装 DELETE 是安全的，并重试保留的清理状态。五分钟 cron 还在每行已处于删除中、属于已撤销安装，或超出硬删除安装时，每次运行处理最多 `OMB_CLEANUP_SWEEP_LIMIT`（代码默认 20，`wrangler.jsonc` 随附；需要 Workers Paid，其每次调用子请求和 D1 限制是默认值所依赖的）过期租约行。每行在其自己的账户中清理，没有已配置账户处理的（见**多个 Cloudflare 账户**）被跳过。行每次运行五个（Workers 限制为六个等待头的连接），且一旦某账户的 API 响应 `429`，运行就停止在该账户启动新行。每行十次提供商调用时，默认约每次运行 200 次调用：API token 每五分钟 1,200 次请求的六分之一，远低于 Workers Paid 每次调用 10,000 子请求的限制。在 Workers Free（每次调用 50 子请求）上将限制设为 4。失败的定时清理从五分钟退避到 15 分钟、一小时、六小时，然后 24 小时。一旦删除待处理 24 小时，每次符合条件的扫描发出一个不同的聚合运维关注日志，不含安装或账户标识符。此有界扫描防止瞬时提供商失败永久孤立资源，而不创建无界定时调用。

### 隧道容量和空闲回收

Cloudflare 默认将账户限制为 1,000 个隧道，区域限制为固定数量的 DNS 记录。同一 cron 的每次运行还在清理前按配置顺序为每个已配置账户执行一个有界容量步骤。一个账户的失败被记录，绝不阻止其他账户：

1. 它为整个账户读取一页 100 个未删除隧道（账户的页游标跨运行行走和回绕）以及区域的 DNS 记录计数。隧道页的总数是账户范围的使用量。
2. 它通过将其端点行移至 `deleting` 并带 `reclaim_requested_at` 来回收空闲隧道，所有账户每次运行最多 20 个。它自身绝不删除任何东西；清理通过归属者 DELETE 相同的所有权验证路径进行。隧道仅在 Cloudflare 报告其：
   - `inactive`（从未运行），无激活时间，创建至少七天前；或
   - `down`，其最后连接结束至少 `OMB_TUNNEL_OFFLINE_RECLAIM_DAYS`（默认 21，最小 7）天前且无更晚激活。

   `healthy` 和 `degraded` 隧道、任何报告的连接、未知状态，以及任何无法解析的时间戳绝不空闲。D1 侧也必须同期安静：安装活跃（未撤销）且未调用控制平面，端点行属于此账户且未被协调或更新，存储的隧道 ID 仍命名所列隧道，且无请求持有该行的租约。这些守卫存在于标记 SQL 本身中，因此与扫描竞争的签到获胜。该账户中没有端点行的隧道仅被计数（`unmatched`），绝不触碰。
3. 在回收的每次破坏性调用前，清理从 Cloudflare 重新读取隧道。若其已重连，或安装在标记后签到，回收被取消，行返回 `ready`（或其 DNS 记录已被移除时返回可重试的 `error`）。归属者的 DELETE 或撤销清除回收标记并始终删除。
4. 它写入账户的容量快照（计数、时间戳、待处理回收，以及休眠端点：隧道已消失的活跃安装的端点），并记录一条带账户 `hostSuffix` 的 `managed endpoint tunnel scan` 摘要。直到下次扫描，计划清理将快照的隧道和 DNS 记录计数减去其删除的内容，因此它释放的槽位不被计为已用。当使用量达到账户隧道或 DNS 记录限制的 90% 时，它发出 `console.error`，带 `"alert": "managed_endpoint_capacity"`、`hostSuffix`、资源、已用、限制和百分比。在该字段上创建 Workers Logs 告警。有多个账户时，池状态（见下）为 `high` 或 `full` 的运行也发出 `"alert": "managed_endpoint_pool_capacity"`：一旦账户填满就永久报告 `full`，因此改为对池告警寻呼。

`OMB_TUNNEL_RECLAIM` 除非设为 `on` 否则为 `observe`（未设或无效值观察）：它记录 `idle`/`eligible` 计数而不标记任何东西。在 `observe` 中部署，检查一个完整扫描周期的真实计数，然后设为 `on`。

`OMB_TUNNEL_LIMIT` 和 `OMB_DNS_RECORD_LIMIT`（各默认 1000）是主账户的隧道配额及其区域的 DNS 记录配额；额外账户携带自己的。这些限制驱动告警和 `/healthz`，为新端点排名账户，并决定隧道创建 `429` 是否为配额。它们自身绝不拒绝分配：只有 Cloudflare 会。保持它们等于 Cloudflare 的真实配额，并在 Cloudflare 提高配额时提高它们（低于真实配额的限制会在该计数处将真正的速率限制变为该账户的十分钟容量门）。

`GET /healthz` 保持 `ok` 和 `service` 不变（桌面以它们为托管登录的门控；满配额不得隐藏登录或恢复），并添加一个无标识符或密钥的 `capacity` 对象。单账户时它是该账户的快照：

```json
{
  "status": "ok | high | full | unknown",
  "checkedAt": 1790000000000,
  "tunnels": { "used": 950, "limit": 1000 },
  "dnsRecords": { "used": 960, "limit": 1000 },
  "providerRejectedAt": null,
  "reclaim": { "mode": "on", "pending": 12 }
}
```

`status` 在近期配额拒绝（有多个账户时，还有被拒绝的 token）门控分配时为 `full`，快照超过 30 分钟时为 `unknown`。有多个账户时，相同字段描述新安装看到的池（接受新端点的账户），`accounts` 按主机后缀列出每个账户（绝不按账户或区域 ID）：

```json
{
  "status": "ok",
  "checkedAt": 1790000000000,
  "tunnels": { "used": 1010, "limit": 2000 },
  "dnsRecords": { "used": 412, "limit": 1200 },
  "providerRejectedAt": null,
  "reclaim": { "mode": "on", "pending": 12 },
  "accounts": [
    {
      "hostSuffix": "openmausbot.com",
      "newEndpoints": true,
      "status": "full",
      "checkedAt": 1790000060000,
      "tunnels": { "used": 1000, "limit": 1000 },
      "dnsRecords": { "used": 400, "limit": 1000 },
      "providerRejectedAt": 1790000030000,
      "reclaimPending": 12
    },
    {
      "hostSuffix": "mausbot.si",
      "newEndpoints": true,
      "status": "ok",
      "checkedAt": 1790000000000,
      "tunnels": { "used": 10, "limit": 1000 },
      "dnsRecords": { "used": 12, "limit": 200 },
      "providerRejectedAt": null,
      "reclaimPending": 0
    }
  ]
}
```

池是 `newEndpoints` 不为 `false` 的每个账户；主账户始终计入。其 `status` 是它们中已知最佳状态（`ok`，然后 `high`，然后 `full`；拒绝分配的账户计为 `full`）。`used` 是它们的总和，或当其中任何快照未知或陈旧时为 `null`；`limit` 和 `reclaim.pending` 是总和；`checkedAt` 是最旧扫描；`providerRejectedAt` 仅在它们全部拒绝时设置。池假设最新发布，因此账户的 `minAppVersion` 不会将其排除：旧发布上的安装在那里被设计拒绝（见**为什么 `minAppVersion`**），池告警不为它们寻呼。读取顶层数字的监控器在配置第二个账户后应读取 `accounts`。每个 Cloudflare 数据中心复用一次快照读取最多两分钟，因此 `capacity` 可能落后 D1 那么久；分配门控始终读取 D1。

### 多个 Cloudflare 账户

由 `CLOUDFLARE_ACCOUNT_ID`、`CLOUDFLARE_ZONE_ID`、`COMPANION_HOST_SUFFIX`、`CLOUDFLARE_API_TOKEN` 密钥、`OMB_TUNNEL_LIMIT`