# 使用无根 Podman 自托管完整技术栈

用一个 Podman 引擎运行 OpenMausBot 服务器、Caddy 和每个 bot 的 Linux 桌面。无需 Docker Engine、Docker Desktop 和 Docker Compose；`podman-compose` 提供 Compose 命令。现有的 [`deploy/docker-compose.yml`](../docker-compose.yml) 仍可作为独立选项。

```text
浏览器 -> 回环 Caddy :8080 -> OpenMausBot :8799
                                    |
                             无根 Podman socket
                                    |
                         每个 bot 的桌面容器
                         （私有网络 + 工作区）
```

服务器和 Caddy 使用 Linux 主机网络，以便服务器访问每个桌面动态分配的回环端口。在 Windows 上，该主机即 Podman WSL2 机器。数据保留在其 Linux 文件系统中，而非 Windows 绑定挂载。

## Windows x64 / WSL2

安装 WSL2 和 Podman（`winget install --exact --id RedHat.Podman`），然后重新打开 PowerShell。Podman Desktop 可选。将仓库克隆到 WSL 可通过 `/mnt/c`、`/mnt/d` 等访问的 Windows 驱动器。在仓库根目录执行：

```powershell
.\deploy\podman\maus.ps1 setup
# 编辑 deploy/podman/.env。将 ENGINES 设为你想要使用的 npm CLI 包。
.\deploy\podman\maus.ps1 up -d --build
.\deploy\podman\maus.ps1 ps
```

Setup 会创建/启动名为 `openmausbot` 的 WSL2 机器（4 CPU、10 GiB 内存、请求 60 GiB 磁盘），在其中安装缺失的 `podman-compose`，启用用户 socket，并生成 `.env`。WSL 资源限制仍然适用。它会保留已有的 `.env`。用 `OMB_PODMAN_MACHINE` 选择其他机器。对于常规 Compose 命令，包装脚本会拒绝已停止的机器；重启后，先执行 `podman machine start openmausbot`，再执行 `up -d`。

Compose 在机器**内部**运行，因此 socket 和绑定路径对服务器和引擎含义相同。`OMB_PODMAN_ENV_FILE` 选择备用环境文件（相对于 `deploy/podman`），适用于隔离的测试夹具。

## Linux x64 / systemd

安装无根 Podman、`podman-compose`，并按发行版说明配置用户的 subuid/subgid 范围。以普通用户运行：

```sh
cd deploy/podman
sh setup.sh
# 编辑 .env；例如 ENGINES=@anthropic-ai/claude-code @openai/codex
PODMAN_COMPOSE_PROVIDER=podman-compose podman compose --env-file .env -f compose.yaml up -d --build
```

本方案面向 Podman 5.8.3 / podman-compose 1.6.0 和 Linux x86_64。远程客户端通过校验和固定；渲染器和服务器均从你的检出构建。引擎安装通过 `ENGINES` 选择加入；需要可复现性时请在其中固定包版本。实际验收环境是 Windows WSL2 机器。原生 Linux、启用 SELinux 的系统、ARM64、macOS 和非 systemd 主机需要单独验证；Containerfile 明确拒绝非 x86_64。启用 SELinux 的系统需要合适的 socket/绑定策略，而非一刀切禁用标签。Linux 上的重启持久化还需按主机管理策略配置用户的 linger 和 Podman 重启服务。

## 引擎与桌面

打开 <http://localhost:8080>。在容器内登录所选引擎，例如 `maus.ps1 exec omb claude`。其登录状态持久保存在 `OMB_DATA_ROOT` 下。对于 Linux 命令，用 `podman compose --env-file .env -f compose.yaml` 前缀替代 `maus.ps1`。

在 **App Settings -> Local VM** 中，准备托管镜像，选择 **Per bot**，并设置桌面最大数量。为每个 bot 指定 **Local VM** 作为其计算机，然后创建其桌面。首次准备镜像需要大量下载。每个桌面沿用现有的 4 GiB / 2 CPU 限制；需为服务器、构建以及所有并发桌面预留容量。参见 [Local VM 指南](../../apps/docs/content/docs/computers/local-vm.mdx)。

服务器的 `maus` 用户（1001）和每个桌面的 `cua` 用户（1000）通过各自的 `keep-id` 命名空间映射到同一个无根主机用户。服务器在两侧将数据根挂载到**相同的绝对路径**，以便桌面工作区路径在引擎主机上可解析。不要将该挂载替换为无关的命名卷或添加 `:U`：递归更改所有权可能导致桌面用户无法写入工作区。已受所有权更改影响的旧工作区需要显式、先备份的所有权修复；本方案不会递归 chown 现有数据。

## 访问与信任边界

Caddy 仅监听回环。要使用 Tailscale Serve，将 tailnet HTTPS 端点路由到 `http://127.0.0.1:8080`，将 `OMB_PUBLIC_URL` 设为该 HTTPS URL，`OMB_HTTPS_HOST` 设为其主机名，然后用 `up -d` 重建服务。用 `maus.ps1 exec omb node dist-server/openmausbot.js pair` 生成配对码。代理会转发客户端地址和协议，因此服务器的配对检查仍然生效。不要直接公开服务器、webhook 监听器或 Podman socket。对于公网域名 HTTPS，请使用现有的 Docker 部署，或为本方案设计并验证一个带认证的边缘。

应用的 socket 授予对**该无根用户拥有的所有容器**的控制权。请使用专用机器/用户。这并非对每个 agent CLI 的完全隔离：CLI 在共享的应用容器中执行。桌面容器只接收自己的工作区，绝不接收 socket。Web Computer 预览和 bot 操作经由服务器。直接的 "Open desktop" URL 使用主机本地动态 noVNC 端口；从其他设备手动访问 noVNC 需要单独的带认证中继。

## 更新、备份与共存

- 更新检出并运行 `up -d --build`；本方案没有发布的 Podman 应用镜像/更新通道。
- 数据、聊天、凭据和桌面工作区位于 `OMB_DATA_ROOT` 下。归档该 Linux 目录前，先停止应用及其桌面。不要复制运行中的 SQLite 数据库。恢复时保留所有权和绝对路径。
- Compose `down` 会停止应用和 Caddy，**不会**停止动态创建的 bot 桌面。在关闭技术栈前，请通过应用移除/停止这些桌面。
- Docker 部署保留其现有命令、卷和文件。本方案不会迁移它们。若要并行运行，请使用独立的数据根和不同的应用/webhook/代理端口；切勿共享运行中的数据目录。
- 验收检查请遵循[隔离验证方案](../../docs/verification/podman-self-hosting.md)。