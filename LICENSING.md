# 许可

OpenMausBot 基于 [Apache License 2.0](LICENSE) 开源，但有一项例外和几点说明。

## 例外：`enterprise/`

`enterprise/` 下的所有内容基于 [OpenMausBot Enterprise License](enterprise/LICENSE) 提供源码，而非 Apache 2.0。你可以阅读、构建和评估它，并在开发和测试中自由运行。在生产环境运行其功能，需要为你的组织获取许可密钥。为第三方托管该产品或将产品白标化，需要签订合作伙伴协议。

删除该文件夹后，剩余部分即为开源版：服务器报告 `{"edition":"oss"}`，普通独立运行不受影响。显式配置为托管登录的工作区，在没有该可选适配器时会拒绝远程访问；移除企业层不得绕过其已配置的登录授权。`open-source edition builds without enterprise/` CI 任务验证了 OSS 构建、启动响应和适配器工厂缺失；它并不覆盖所有托管 HTTP 路由。隔离的托管工作区测试单独验证这些登录和吊销路径。服务器可识别的权益 ID 列表见 [`enterprise/FEATURES`](enterprise/FEATURES)。

新功能的归属规则：开源用户会需要它吗？那就放进核心，以公开 Pull Request 提交。属于组织、管理或分层特性？那就放在 `enterprise/` 下，通过权益控制。客户特定的品牌、技能、包或连接器，应作为数据和配置放在该客户自己的仓库中，绝不作为 Fork。

## 贡献

- `enterprise/` 之外：基于 Apache 2.0 贡献。无需 DCO 签署或 CLA。仅提交你编写或有权贡献的代码。
- `enterprise/` 之内：在你的 Pull Request 中评论一次，签署[贡献者许可协议](CLA.md)。它让项目能在自己的许可下保留该文件夹，同时仍接受你的贡献。
- 开放核心边界、云接缝和本文件由 [`CODEOWNERS`](.github/CODEOWNERS) 覆盖：由维护者审查相关变更。

## 第三方组件和商标

捆绑的第三方软件保留其自身许可；声明、许可文本、源码位置和 SBOM 列于 [NOTICE](NOTICE) 和 [`third_party/`](third_party/)。OpenMausBot 和 MausBot 名称及吉祥物是 Supamaus Software Private Limited 的商标；Apache License 不授予商标权（第 6 条），因此基于 OpenMausBot 构建的产品需要自己的名称，除非合作伙伴协议另有约定。