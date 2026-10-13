# 本地 Docker Compose

在仓库根目录运行 `docker compose up -d --build`，然后打开
http://localhost:8080。需要支持 Linux 容器的 Docker。
Compose 提供默认值，无需 `.env` 文件。
应用容器使用 Docker 的 init 进程回收孤立的 agent 子进程。

如需自定义，将 `.env.example` 复制到仓库根目录的 `.env`。
`.env` 文件被 Git 忽略。Shell 环境变量优先。
`OMB_HTTP_PORT` 修改主机端口和默认公共 URL。
内部服务端口在共享网络命名空间内保持固定。
`ENGINES` 选择以空格分隔的 npm 包；空值跳过安装。
镜像还包含 Grok Build，版本由 `deploy/local/Dockerfile` 中的
`GROK_VERSION` 固定；Grok 和 Codex 可在应用的引擎设置中用一次性验证码登录，
无需终端。

对于 Tailscale Serve，将 `OMB_PUBLIC_URL` 设为你的 HTTPS URL，
`OMB_HTTPS_HOST` 设为其主机名（不含协议和路径）。在主机上配置 Tailscale
Serve 转发到选定的 localhost HTTP 端口。
保持默认的 `OMB_BIND_ADDRESS=127.0.0.1`。当 `OMB_HTTPS_HOST` 与任何其他
绑定地址同时设置时，Caddy 会拒绝启动，因此直接远程 HTTP
客户端无法通过提供该主机名来声称 HTTPS 语义。
主机名映射为该主机保留 HTTPS
语义，而 localhost 访问继续使用 HTTP。
私有 tailnet webhook URL 仅该 tailnet 上的调用者可访问。
未设置 `OMB_HTTPS_HOST` 时，可更改绑定地址以进行 HTTP 访问。
仅在配合可信的本地 TLS 终止代理时使用 HTTPS 主机名映射。

登录并配对浏览器：

```sh
docker compose exec omb codex login --device-auth
docker compose exec omb node dist-server/openmausbot.js pair
```

在 Windows 上，`./maus.ps1` 使用仓库目录将参数转发给 Compose。
它遵循 Docker 选定的上下文和 `DOCKER_CONTEXT`。

数据和引擎凭据持久化在命名数据卷中。对于已有
卷，将 `OMB_DATA_VOLUME` 设为其名称，并设置 `OMB_DATA_EXTERNAL=true`。
全新安装会自动创建卷。

使用 `docker compose build --pull` 更新，然后运行 `docker compose up -d`。
使用 `docker compose stop` 停止。`docker compose down -v` 删除受管理的卷。