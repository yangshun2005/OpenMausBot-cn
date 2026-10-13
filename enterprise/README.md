# OpenMausBot Enterprise

面向托管和白标部署的源码可用功能。此文件夹有自己的 [LICENSE](./LICENSE)；其外的所有内容均为 Apache 2.0。

**删除此文件夹即得到开源版本。** 核心仅通过一个 Hook 点接入此层：`server/enterprise.ts`，它会在存在时加载 `enterprise/server/index.ts`（或打包后的 `index.js`），否则报告开源版本。核心文件不从此处导入任何内容。

核心的查找顺序：设置了 `OMB_ENTERPRISE_DIR` 时（且仅在该处查找）；否则在服务器根目录旁（检出的 `enterprise/`、npm 包的 `<package>/enterprise/`），然后在其内部（`dist-server/enterprise/`，Docker 镜像和打包桌面版携带的全部内容；`scripts/bundle-server.mjs` 将打包层和此 LICENSE 写入该处）。

## 部署如何启用企业版

在服务器上设置 `OMB_LICENSE_KEY`。密钥格式为 `omb1.<claims>.<signature>`：claims 是可见的 JSON（授权对象、包含的权益、过期时间），由 Ed25519 密钥签名，其公钥已内置到 `enterprise/server/license.ts`。验证离线完成，一个构建服务所有客户，因为功能集由密钥决定而非代码。

`GET /api/edition` 报告结果：

```json
{ "edition": "enterprise", "customer": "Acme", "features": ["budgets", "whitelabel"], "expiresAt": "2027-09-02", "expiresInDays": 12 }
{ "edition": "enterprise", "customer": "Acme", "features": ["budgets", "whitelabel"], "expiresAt": "2027-09-02", "expiresInDays": -2, "graceEndsAt": "2027-09-09", "notice": "OMB_LICENSE_KEY expired on 2027-09-02; enterprise features keep working until 2027-09-09 while it is renewed" }
{ "edition": "oss", "features": [], "notice": "enterprise layer disabled: OMB_LICENSE_KEY expired on 2027-09-02; renew it to keep enterprise features" }
```

只要密钥有到期时间，就会返回 `expiresInDays`；非管理员会话的 `/api/edition` 不包含 `expiresInDays`、`graceEndsAt` 和 notice。到期前 30 天起，服务器在启动时记录警告，管理员在设置中看到横幅。到期日后，功能继续工作 7 天宽限期，显示上述 notice 和横幅说明截止时间；之后停止，无需重启。密钥格式和签名校验不变：此层只是接受过期时间在宽限期内的密钥。

缺失、篡改或过期的密钥不会阻止普通独立服务器运行：它运行开源版本，notice 说明需要修复什么。显式配置为托管登录的工作区，当其 `admin` 权益或身份服务不可用时，远程访问会失败关闭。它绝不能静默回退到独立邮箱或配对凭据。

## 权益 ID

| id | 授予 | 状态 |
|---|---|---|
| `whitelabel` | 来自 `brand.json`（见下文）的产品名称、标语、强调色、logo、favicon 和支持链接 | 已发布 |
| `sso` | 通过公司身份提供商登录 | 计划中：尚无任何检查 |
| `admin` | 设置 → 安装 以及可选的托管工作区登录适配器 | 已发布 |
| `budgets` | 整个工作区一个每月支出上限，达到上限时拒绝新对话轮次，带警告百分比和管理员通知（非按机器人或按分区） | 已发布 |
| `billing` | 按模型售价：用量 → 历史 及其 CSV 中的可计费列，以及替换成本估算中列表价的按模型价格 | 已发布 |

[FEATURES](./FEATURES) 是权威列表。

核心通过 `server/enterprise.ts` 的 `entitled("id")` 来门控功能。未知 id 会携带在密钥中但不授予任何权限，因此密钥可先于功能发布。

## 签发密钥

```sh
node enterprise/scripts/issue-license.mjs keygen          # once; prints the public key to add to license.ts
node enterprise/scripts/issue-license.mjs issue --customer "Acme" --features whitelabel,sso --expires 2027-09-02
```

签名密钥存放在仓库外（默认 `~/.config/openmausbot-enterprise/signing-key.json`）。轮换方式是生成新密钥对并追加其公钥：旧密钥对签发的密钥在过期前继续有效。

## 什么放在哪里

- 任何开源用户可能需要的？放入核心，作为公开 PR。
- 组织、管理或层级相关的，或下一任企业负责人会看到的内容？放在此处，置于权益之后。
- 客户特定的品牌、技能、包、连接器？放在客户自己的仓库：数据和配置，绝不用 Fork。

## 托管工作区登录（`admin`）

`server/workspace-access.ts` 中的工作区侧适配器通过 HTTPS 连接到独立部署的身份服务。该服务的控制台、邀请、提供商网关和部署自动化不在此仓库中发布。桌面版和普通自托管服务器不启动也不依赖托管控制台。

运维人员通过 `OMB_ADMIN_URL`、`OMB_ADMIN_WORKSPACE` 和 `OMB_PUBLIC_URL` 让工作区选择加入此协议；需要有效的 `admin` 权益。默认保留本地登录允许列表作为额外的收窄检查。运维人员可通过 `OMB_ADMIN_MEMBERSHIP=portal` 为已验证的门户会话显式委托成员资格；这绝不豁免普通邮箱或配对会话。参见[协议和隔离验证方法](../docs/verification/hosted-workspaces.md)。

## 白标（`whitelabel`）

将 `brand.json` 放入服务器的数据目录（`OMB_DATA_DIR`，Docker 中为 `/data` 卷），或用 `OMB_BRAND_FILE` 指向它：

```json
{
  "name": "Reliable Platform",
  "tagline": "Back office, on autopilot",
  "accent": "#1D4ED8",
  "logo": "data:image/svg+xml;base64,…",
  "favicon": "data:image/png;base64,…",
  "supportUrl": "https://help.example.com"
}
```

仅 `name` 为必填。`logo` 是内联 `data:image/…` URI 或 `https://` URL；`accent` 是 6 位十六进制颜色，其上的文字颜色根据对比度推导。服务器在每次 `GET /api/brand` 时读取该文件，因此编辑在下次重新加载时生效；应用在首次绘制前获取它，因此窗口不会闪现默认名称。未授权的服务器或文件有误时，保留默认品牌，并在 `/api/brand` 和启动日志中说明原因。

`brand.json` 无法更改以下内容，因为它们在打包时已固化：桌面应用的 bundle 和菜单栏名称、安装程序名称、macOS 权限提示、iOS 应用名称以及辅助应用的路径。完全重新品牌的桌面构建是逐客户的打包工作，而非配置。