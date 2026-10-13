# server/routes

新的 HTTP 路由以模块形式放在这里。不要往 `server/index.ts` 里加路由 `if` 块。
`scripts/testing/index-route-ratchet.test.ts` 会在 `index.ts` 新增路径守卫时失败。移出一个路由后，
要在该测试里调低它的数量。

- 路由模块导出一个带显式依赖的工厂函数 `createXRoutes(deps): RouteHandler`
  （与 `server/workspace-backup-http.ts` 结构相同）。它绝不导入 `server/index.ts`。
- `index.ts` 在其余接线旁构建它（一处导入，一处 `ROUTES.push(createXRoutes({...}))`）；
  无需依赖的模块直接在 `table.ts` 的 `ROUTES` 中列出。
- 当请求不属于该处理器时，返回 `PASS`；返回其他任何值都表示它已作答。
  在同一模块内，字面路径要放在同样能匹配它的模式之前。
- `handleRequest` 在鉴权门之后立即运行路由表，因此处理器已通过鉴权。
  作用域规则仍按路径在 `server/request-auth.ts` 中配置。
- `RouteContext` 携带 `req`、`res`、`url`、`path`、`method`、`auth`、`json`、`readBody`；其余为 `deps`。
- 仅类型导入必须写成 `import type`（或内联 `type`）：服务器依赖类型剥离运行，因此
  裸类型导入能通过 `tsc`，却会在启动时崩溃。