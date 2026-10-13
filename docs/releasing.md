# 发布

常规发布时，运行 **Actions → Prepare next release → Run workflow**，选择 patch、minor 或自定义版本。它会创建一个很小的版本号提升 PR；合并该 PR 会自动启动 **Release**，并基于确切的合并提交组装草稿。准备好后，审查并发布该草稿。

现有的 **Actions → Release → Run workflow** 按钮仍可用于重跑和恢复。它基于单个固定提交构建 macOS（arm64 + x64，已签名、已公证、已装订）、Windows 和 Ubuntu，按用户实际收到的方式验证每个构建产物，并在 [OpenMausBot releases](https://github.com/milind-soni/OpenMausBot/releases) 中组装规范草稿。完全相同的资产也会暂存到公开的旧版发布仓库，以便 0.1.46 及更早版本的已安装构建能在仓库迁移后继续更新。

勾选 **publish** 会先发布并验证规范发布版本，然后使完全相同的旧版更新桥接可见。不勾选则只审查规范草稿；当你在 GitHub UI 中发布该草稿时，**Sync published release** 工作流会自动验证并发布其旧版镜像。切勿只发布旧版草稿。

发布提交必须已通过 CI：**CI passed on the release commit** 任务会（在各平台构建期间）等待 ci.yml 在固定提交上的 `CI` 检查，若失败则在任何草稿生成前停止发布。如果该提交没有 CI 运行，或有更新的合并替换了它，该任务会在临时分支 `release-ci/v<version>` 上对该提交本身运行 CI。遇到偶发失败后，重跑失败的 CI 任务；仍在等待的发布会采用新的尝试，而已停止的发布只需执行 `gh run rerun <id> --failed`。`ship_without_ci` 会跳过等待，仅用于紧急情况。

工作流拒绝覆盖已发布的版本。手动 Release 运行仍要求所选 ref 上的 `package.json` 版本号已提升。如果任何安装程序、稳定下载名称、更新源、blockmap、大小或摘要缺失或不一致，发布将被拒绝。

GitHub 根据自上一个规范标签以来的 Pull Request 生成发布正文。文档更新日志将已发布的规范版本与旧版归档合并为一份完整历史，并缓存五分钟。配置下面的可选 Vercel Hook，即可在发布后立即重建文档；否则实时缓存或下一次常规文档部署会刷新它。

## 更新器迁移不变式

`app-update.yml` 被打包进每个桌面应用中。0.1.46 及更早的构建指向 `milind-soni/openmausbot-releases`；更新的构建指向 `milind-soni/OpenMausBot`。因此：

1. 桥接期内，每个新发布版本都会逐字节发布到两个仓库。
2. `openmausbot-releases` 必须保持公开。不要删除其最终的桥接发布版本、更新源或资产。
3. README 和文档中的下载指向规范仓库，旧版镜像仅供已安装的更新器客户端和历史发布版本使用。

npm 包单独发布，其带版本号的 `.tgz` 仅附加到规范发布版本。它不是桌面更新器资产；镜像检查允许这一个额外文件，同时仍验证完整、逐字节一致的桌面资产集。

## 为什么存在这些关卡

`release.yml` 中的每个验证步骤都对应手工发布（0.1.15–0.1.25）中的一次真实事故：过期的构建输出破坏代码签名、一个裸导入导致打包后的服务器启动即崩溃而所有检查仍显示绿色、打包后辅助路径解析到应用之外、装订静默使每个已发布哈希失效，以及已完成的发布作为草稿不可见。未阅读关卡上方的注释前，不要移除任何关卡。

## 捆绑浏览器关卡

桌面构建在打包前还会暂存固定版本的引擎和 Chromium Headless Shell。签名前检查会验证完整资源和上游哈希；原生浏览器冒烟测试和 macOS 签名检查在打包输出上运行。更新归属、许可证来源和 Linux 沙箱限制见[浏览器打包](browser-packaging.md)。缺少浏览器资源必须使构建失败，而不是发布一个在首次使用时才下载它们的安装程序。

## 一次性设置：发布密钥

在 **OpenMausBot → Settings → Secrets and variables → Actions** 中设置这些密钥。

**Prepare next release** 工作流还需要启用 **Settings → Actions → General → Workflow permissions → Allow GitHub Actions to create and approve pull requests**。该工作流只创建版本 PR；它从不批准或合并它。

### 1. `MAC_CERT_P12_BASE64` + `MAC_CERT_PASSWORD`

Developer ID Application 证书，从当前签署发布版本的 Mac 导出：

```sh
# Keychain Access → My Certificates → "Developer ID Application: Milind Soni
# (993D98NH4J)" → right-click → Export… → .p12 with a strong password, then:
base64 -i DeveloperID.p12 | pbcopy   # → MAC_CERT_P12_BASE64
# the export password             → MAC_CERT_PASSWORD
```

### 2. `APPLE_API_KEY_P8_BASE64` + `APPLE_API_KEY_ID` + `APPLE_API_ISSUER_ID`

用于公证的 App Store Connect API 密钥（比 CI 用的应用专用密码更好——可撤销、有范围限制、无需双重验证）：

1. [App Store Connect → Users and Access → Integrations → App Store Connect API](https://appstoreconnect.apple.com/access/integrations/api)
2. 生成一个具有 **Developer** 角色的 **Team Key**
3. 下载 `.p8`（仅一次机会），记下 Key ID 和 Issuer ID

```sh
base64 -i AuthKey_XXXXXXXX.p8 | pbcopy   # → APPLE_API_KEY_P8_BASE64
```

### 3. `RELEASES_PAT`

一个细粒度个人访问令牌，让工作流能写入旧版更新器镜像：**GitHub → Settings → Developer settings → Fine-grained tokens** → 仓库访问：仅 `openmausbot-releases` → 权限：**Contents: Read and write**。设置较长的有效期并加日历提醒。规范发布使用工作流的范围受限 `GITHUB_TOKEN`，不需要 PAT。

### 4. 可选 `DOCS_DEPLOY_HOOK_URL`

在 Vercel 中打开文档项目，为其生产分支创建一个 **Deploy Hook**。将该私有 URL 存为 `DOCS_DEPLOY_HOOK_URL` 仓库密钥。发布版本后即会请求一次新的文档部署，使生成的更新日志立即出现。不要将 Hook URL 放入源代码控制。

### 本地回退

当 Actions 不可用或发布需要手术时，手工路径仍然有效：`pnpm package:mac`，用 `codesign --verify --deep --strict` 把关，用本地钥匙串配置文件公证（`xcrun notarytool submit … --keychain-profile AC_PASSWORD`），装订，重新打包，重新生成 blockmap 和 `node scripts/regenerate-mac-feed.mjs`，将完全相同的完整资产集上传到两个仓库，先发布并验证规范发布版本再处理旧版镜像，并始终通过下载回来验证已发布的字节与已发布的更新源一致。