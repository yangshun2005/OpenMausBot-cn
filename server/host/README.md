# server/host — 服务器的 host 模块

`server/index.ts` 正在被拆分成模块，放在这里（HTTP 路由放在 `server/routes/`）。规则如下，确保这个文件不会重新膨胀：

1. **一个模块只负责一件事，依赖显式声明。** 模块导出一个工厂函数 `createArea(deps: AreaDeps)`，返回服务器其余部分调用的函数。deps 是它需要的单例（`store`、`cfg`、`registry`、`bus`、`coordinator`、`sessions`、`broadcast`、`notify`……），由 `server/index.ts` 传入，后者始终是唯一的组合根。模块绝不导入 `server/index.ts`。
2. **状态随代码一起迁移。** 某个区域拥有的模块级 `Map` 和 `Set` 放在其工厂函数内部，只创建一次，顺序与现在一致。其他区域读取的共享状态以函数形式暴露，不暴露裸 `Map`。
3. **迁移就是迁移。** 把代码从 index.ts 移出的 PR 不改变行为：函数体逐字节相同（用 `git diff --color-moved=dimmed-zebra` 审查），行为变更放在之前或之后的独立 PR 中。注释随代码一起迁移。
4. **顺序重要的留在 index.ts。** 顶层 `await` 和 `bus.subscribe` 注册在 index.ts 中保持原有顺序：被移走的折叠逻辑变成函数，由 index.ts 在原来的位置注册。
5. **延迟绑定的单例靠构造顺序，不靠提升。** index.ts 在远离声明处赋值的 `let`（`routines`、`calendarCalls`、`orgLibrary`、`workspaceAccess`……）变成 deps 中的小 getter（`deps.routines()`），绝不引用被提升的名称。
6. **测试跟随代码。** 按源码文本切片 index.ts 的测试（`readFileSync(... index.ts)` + 标记）要么直接导入新模块作为真正的单元测试，要么在端到端场景已覆盖该行为时删除。重写 index.ts 的 loader Hook 改为重写新文件。
7. **大小受限。** `scripts/testing/file-size-ratchet.test.ts` 列出所有超过 1,000 行的源文件；已列出的计数只能下降，未列出的文件不得超过 1,000 行，任何文件不得超过 2,000 行。这里的新模块保持在 1,500 行以下，多数在 800 行以下。
8. **路由**遵循 `server/routes/README.md` 中的规则（`createXRoutes` 工厂、一次 `ROUTES.push`、不属于自己时 `PASS`），`scripts/testing/index-route-ratchet.test.ts` 中的路由棘轮随每次迁移下降。

拆分顺序（事实和行号映射见项目笔记）：先把路由从请求处理器中移出，然后是卡片和提案服务、computers、turn 准入和 `startTurn`、rooms 和 coordinator host Hook、事件折叠、身份和 wire 投影、启动和关闭。