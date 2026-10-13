# 新用户引导

状态：已在 `feat/onboarding` 上构建完成。这是最终实现的设计；它取代的探索性方案曾考虑录制片段和一份现场指南页面，两者均被放弃，改用代码绘制的场景和真实界面上的引导式导览。

## 目标

新安装后，用户应在首次会话中了解 OpenMausBot 是什么（有手的智能体、已连接的应用、频道、自动化、终端），以及每一项在应用中的位置，无需阅读手册。每个部分都可跳过、可从设置中重放，且绝不会在已配对的远程客户端上显示。

## 三层结构

1. **欢迎流程**（`src/components/onboarding/WelcomeFlow.tsx`）。一张卡片通过 View Transitions API 在各节拍之间变形，吉祥物作为唯一向导。节拍依次为：问候、短片、引擎、权限（仅在听写可用时）、手机、认识你的机器人。顺序是 `src/lib/onboarding.ts` 中的纯逻辑（`beatsFor`、`nextBeat`、`previousBeat`），并有单元测试覆盖。
2. **功能短片**（`src/components/onboarding/reel/`）。六个用代码绘制的场景，每个都是一条小时间线，报告吉祥物提示和自身结束：智能体、手、应用、频道、自动化、终端。用代码绘制意味着无需附带媒体文件，每种皮肤和语言环境看到相同的画面，UI 变更也不会留下过时的录像。`scenes/index.ts` 中的 `REEL` 是播放顺序。
3. **引导式导览**（`src/components/onboarding/GuidedTour.tsx`、`src/lib/guided-tour.ts`）。欢迎流程之后，在实时界面上用聚光灯高亮，每一步都有"下一步"。导览按用户的操作路径行进：打开工具菜单并指向目标项，再打开其背后的页面，并关闭已打开的内容，使最后一张卡片停留在聊天界面上方。步骤：输入框、模型标签、Computer 按钮、面板的视图标签页（存在时显示 Browser 标签页）、工具菜单、已连接应用项、应用市场、自动化项、自动化页面、结束卡片。

第四个小部分 `FirstConversationTour.tsx` 在真实对话中首次出现审批卡片和首次连接应用卡片时，对它们进行说明。

## 持久化

服务器配置携带一个 `onboarding` 块（`server/config.ts`）：`completedAt`、`version`、`reelSeen`、`hintsSeen[]`。当当前 `WELCOME_VERSION` 没有完成时间戳时，欢迎流程即应触发；已完成旧 localStorage 门槛的安装获得一个版本的宽限期，不会看到新流程。导览步骤和聚光灯是 `hintsSeen` 中的 id，在界面推进前通过 `PUT /api/config` 写入，因此刷新后会回到同一步骤。位于前一步骤所打开内容内部的步骤，在进入时会重新打开它，原因相同。

设置 → 通用有两个重放入口：欢迎导览和应用导览。手动重置：

```
curl -X PUT http://127.0.0.1:<port>/api/config -H 'content-type: application/json' \
  -d '{"onboarding":{"completedAt":"","version":0,"reelSeen":false,"hintsSeen":[]}}'
```

## 聚光灯

`Spotlight.tsx` 通过 `data-tour` id 查找锚点，绝不使用 class，因此重构不会悄无声息地破坏导览。暗层是一个奇偶裁剪路径多边形，从整个窗口缓动收缩到锚点；镂空处带有强调色圆环和光晕。卡片位于锚点上方、下方或旁边（侧边栏控件用旁边），锚点填满窗口时位于其内部，无锚点时居中。布局、滚动、调整大小和 DOM 变更都会重新测量。按 Escape 结束导览。

锚点：`composer`、`model`、`computer`、`computer-tabs`、`computer-browser`、`tools`、`nav-apps`、`nav-automations`、`apps-panel`、`apps-close`、`automations-page`、`approval`、`connector`。

侧边栏工具弹出层忽略导览卡片（`[data-tour-card]`）上的指针按下事件，因此按"下一步"不会关闭它正在描述的菜单。当用户按下高亮的控件而非"下一步"时，导览不会重复该控件自身的操作。

## 动效

仅使用 transform 和 opacity，标准曲线 `cubic-bezier(0.22, 1, 0.36, 1)`，40 ms 错开，没有任何元素从 `scale(0)` 进入。令牌和关键帧位于 `src/styles.css`（`rise`、`spot-in`、`orbit`、`ripple`、`bolt`）。通过 `src/lib/onboarding.ts` 中的 `reducedMotion()` 遵循减弱动效设置，它读取媒体查询和 `data-reduced-motion` 开发钩子：短片场景渲染其最终帧，聚光灯出现时无扫过效果，节拍切换时无视图过渡。

## 预览与验证

- `onboarding-preview.html` + `src/onboarding-preview.tsx` 是开发画廊，展示每种皮肤下的每个节拍（`?beat=&skin=&reduced=1&all=1`）。
- 纯逻辑由 Vitest 覆盖：`onboarding.test.ts`、`guided-tour.test.ts`、`first-conversation.test.ts`，以及 `server/config.test.ts` 中的配置合并。
- 导览已在无头 Chrome 中针对隔离夹具（`node --experimental-strip-types scripts/control-omb.ts launch`）端到端走查，每步截图。注意记录是按工作区存储的，因此同一服务器上的第二个客户端会运行相同的导览并跳过它看不到的步骤；请从单个窗口测试。

## 范围之外

- 手机和原生客户端：它们看不到此分支的任何内容。
- 欢迎流程内的皮肤选择器。
- 按用户的引导状态（已为 `feat/rbac-users` 准备，并移至该分支）。
- `server/cli-setup.ts` 中的 CLI 设置向导。