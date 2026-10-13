# 从应用外部打开聊天

在侧边栏中打开某个聊天行的 **···** 菜单，选择 **复制链接**（机器人对话、房间或频道均可）。它会复制一个 `openmausbot://` 链接，在桌面应用中打开该聊天，因此外部仪表盘、浏览器书签或脚本都能把人直接带到正确的对话：

```text
openmausbot://thread/<thread-id>?bot=<bot-id>
```

只知道机器人（或房间）的仪表盘也有对应链接，会打开其当前对话：

```text
openmausbot://bot/<bot-id>
```

## 链接

- **`openmausbot://thread/<thread-id>?bot=<bot-id>`** — 一个确切的聊天。`?bot=` 部分指明其归属（机器人 id 或房间 id）。打开时可省略（裸的 `openmausbot://thread/<thread-id>` 会像粘贴应用内引用一样，在本机聊天中解析），但 **复制链接** 始终会写入它，且应用会严格校验：如果该 thread 不属于该机器人，则不会打开任何内容。
- **`openmausbot://bot/<bot-id>`** — 机器人的当前对话，效果等同于在侧边栏点击该机器人：隐藏 thread 时，打开其主 thread，即此人最后写入的对话。适合 Chat 按钮只知道机器人 id 的团队地图。房间 id 在此同样适用。

## 打开链接的效果

- 将窗口置于最前。如果当前显示的是已连接的服务器（**Server** 菜单），会先切回 **Local** —— 聊天在本机 —— 待该页面就绪后再打开链接。
- 打开指定的聊天，并在侧边栏中显示其所在行。**仅此而已**：绝不发送消息、打断正在进行的回合，或开启新对话。
- 启动时同样有效：链接会保留到应用加载完机器人，因此可以启动已关闭的应用并仍能找到对应聊天。在 macOS 上，应用窗口关闭后仍在运行，链接会打开一个新窗口。
- 从任意页面均可使用 —— 团队地图、Routines、房间 —— 并离开该页面进入聊天。设置和键盘快捷键列表会关闭，以免聊天被遮挡；带有未保存输入的对话框（如 New Bot）则保留。
- 未知或不匹配的 id 会在聊天中显示一条简短提示，而不打开任何内容：未知机器人链接显示 "That bot is no longer here."；未知 thread 显示 "That thread is no longer here."；若链接指向的机器人存在，则显示 "That thread is no longer here. Showing <bot> instead." —— 窗口切换到该机器人，与应用内失效 thread 引用的行为一致。

与 `openmausbot://cloud` 和 `openmausbot://organization` 链接一样，这些都是本地 shell 操作：从远程服务器加载的页面永远不会收到它们，浏览器版本则完全没有此类链接。作为另一台计算机的伴侣连接的应用（**Settings → Remote access → Connect to another computer**）会针对该计算机的聊天（即其显示的聊天）打开这些链接。