# OpenMausBot 连接应用代理

该 Worker 将共享的 Composio 项目密钥隔离在桌面构建之外。每个安装实例获得一个随机 bearer token，仅以 SHA-256 哈希形式存储在 D1 中。Worker 为该安装实例分配独立的 Composio 用户/会话，代理 MCP 流量，并按需向本地应用返回短期有效的 Connect Link。

桌面端永远不会收到项目密钥。授权链接仅在请求时返回，且不会持久化到聊天消息中。

本仓库的部署步骤：

1. `pnpm broker:types`
2. `pnpm exec wrangler d1 migrations apply openmausbot-composio --remote --config cloudflare/composio-broker/wrangler.jsonc`
3. 对于已有的 Worker，运行 `pnpm exec wrangler secret put COMPOSIO_API_KEY --config cloudflare/composio-broker/wrangler.jsonc`，然后运行 `pnpm broker:deploy`。
4. 首次部署时，将 `COMPOSIO_API_KEY=...` 写入被忽略的 `.dev.vars.production` 文件，然后运行 `pnpm exec wrangler deploy --config cloudflare/composio-broker/wrangler.jsonc --secrets-file .dev.vars.production`。完成后立即删除该文件。

Fork 本仓库的用户应创建自己的 D1 数据库和限流命名空间，替换 `wrangler.jsonc` 中的 ID，以自己的 Worker 名称部署，并在打包构建中设置 `OMB_COMPOSIO_BROKER_URL`。仅使用本地服务器加 Composio 项目密钥运行，仍是不依赖 Cloudflare 的自托管方案。

将 `REGISTRATION_MODE` 设为 `closed` 可停止签发新的安装 token，且不影响现有用户。