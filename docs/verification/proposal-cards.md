# Profile 提案与凭证卡片

运行启动器管理的 fake-engine 测试夹具：

```sh
pnpm exec vitest run server/profile-card-lifecycle.e2e.test.ts
```

该测试启动一个一次性服务器，从其 fake engine 获取真实的 turn capability，并提交通知和语音回复偏好。它验证：确认后两个开关均被保存；旧 profile 卡片过期后不含任何选项；重复回答无法使其复活；八张过期卡片既不会阻塞已完成的等待，也不会占用新提案的配额。两次凭证请求生成两张不同的卡片；被取代的卡片拒绝 provide、resume 和 dismiss，而最新的卡片仍可操作。

该夹具将 action/response 证据、`wait` 和 `messages` 输出保留在 `<server-log>.profile-cards.json` 中，然后停止其子进程并删除临时数据。它不记录任何内部 bearer token。这验证的是服务器行为；渲染由 ApprovalCard 和 SecretRequestCard 组件测试单独覆盖。