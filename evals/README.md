# 行为评估

OpenMausBot 测试框架的离线行为评估（上游 Issue #1503，第 1-3 层）。单元测试和端到端测试套件测试代码路径；这些场景测试测试框架*做了什么*：某一轮对话的模型被允许调用哪些工具、工作被派发到哪里、交接树长什么样、例程如何延迟、以及计算机声明何时触发。

原则：评估测试框架，绝不评估模型。脚本化引擎从计划文件重放确定性的轮次（工具调用、文本、拒绝）；不调用任何外部 API；每次运行都是封闭且可重复的。

## 运行

    pnpm eval                     # 所有场景，JSON + markdown 报告输出到 evals/reports/runs/
    pnpm eval --scenario lazy-computer-claim
    pnpm eval --golden            # 第 2 层：针对已提交的追踪基线重放脱敏的黄金线程
    pnpm eval --live              # 第 3 层：可选的实时模型冒烟测试（除非 OMB_EVAL_LIVE=1 否则跳过）
    pnpm typecheck && pnpm exec tsc -p evals/tsconfig.json   # 评估也独立进行类型检查
    npx vitest run --config evals/vitest.config.ts   # 将相同场景作为测试门禁运行

## 目录结构

- `scenarios/` — 夹具文件，每个场景一个 JSON。纯数据：机器人、脚本化轮次、驱动步骤、断言。
- `providers/mock/` — 确定性脚本化引擎：构建假 provider 逐轮重放的计划，其中 `@key` 机器人引用解析为实时 id。
- `runners/` — 为每个世界启动真实的测试框架服务器（脚本化协调服务器；Local VM 夹具世界），解释步骤，冻结证据。
- `scorers/` — 针对冻结快照的纯断言评估；评分器从不接触服务器。
- `reports/` — 每次运行的 JSON 和 markdown 构建产物（已 gitignore）。
- `golden/` — 第 2 层：脱敏流水线（输入原始样本，输出合成场景）、行为追踪提取、以及针对已提交追踪基线的重放。
- `live/` — 第 3 层：可选的实时模型冒烟场景、阈值和报告；实时基线已 gitignore。

## 场景剖析

每个夹具声明一个 `world`、要创建的 `bots` 及其脚本化 `turns`、可选的门控名称（保持某轮次打开的文件，用于确定性的忙碌窗口）、驱动真实服务器的 `steps`、以及针对冻结证据检查的 `assertions`。断言覆盖工具调用序列和参数、派发目标、交接树形状、系统提示、例程延迟戳、计算机门控答案和转录事实。

## 当前固定项

- `dispatch-supersede` — 当协调工作尚未完成时向空闲 Chief 发送的消息会立即作为其自身轮次运行（不会被引导到派发轮次中，也不会排在工作之后），分配保持附加状态，且该轮次被告知哪些工作尚未完成。（当前语义：新消息不是取消。）
- `routine-deferral` — 在忙碌目标之后到期的例程在保持排队的同时被打上 `deferredAt` 戳，然后在目标空闲时派发并完成。
- `lazy-claim-screenless`（Issue #1361）— 无屏幕的 Auto 轮次挂载计算机工具而不声明 VM，并在另一线程持有它时完成；不出现等待活动。
- `lazy-computer-claim`（Issue #1361）— 第一次屏幕调用触发延迟声明，在另一线程持有 VM 时被诚实拒绝，显示现有的等待活动，并在释放后继续。
- `lazy-claim-rejection`（Issue #1369）— 在派发和第一次屏幕调用之间死亡的 VM 拒绝触发的声明：调用被诚实拒绝而非作为争用处理，恰好一个终端计算机不可用错误落地，轮次结束而非保持忙碌，且后续屏幕调用在拆卸撤销桥接能力后失败关闭。

## 世界

- `coordination` — 打包的验证服务器，带脚本化房间引擎：普通聊天、`coordinate_bots` 交接、例程。
- `localVm` — 真实服务器，容器边界被仓库内 VM 夹具（`server/testing/group-local-vm-hooks.mjs`）替换：封闭的 Local VM 生命周期，无 Podman。运行器扮演桥接的计算机控制轮询，这是延迟声明所在的接缝。

## 第 2 层 — 黄金线程重放

将真实协调线程转换为合成夹具，然后通过脚本化引擎重放：

    node --experimental-strip-types evals/golden/redact/redact-cli.ts \
        --input evals/golden/samples/<raw-thread>.json \
        --out evals/golden/scenarios/golden-<name>.json
    pnpm eval --golden --update-baseline --scenario golden-<name>

脱敏步骤将 id、名称和自由文本重写为稳定的合成占位符，并在写入任何内容前运行泄漏扫描，因此真实线程变为可审查、可提交的数据。将脱敏场景和重新生成的基线都提交到 `evals/golden/baselines/` 下；重放针对该基线断言稳定的工具追踪和结果——轮次顺序、工具调用名称、派发目标、交接树形状。

## 第 3 层 — 可选的实时模型冒烟测试

实时模型运行默认从不执行：除非门控打开，`pnpm eval --live` 打印一行跳过信息并以 0 退出。

    OMB_EVAL_LIVE=1 OMB_EVAL_LIVE_CONFIG=/path/to/instance.json pnpm eval --live

实例文件（或内联 `OMB_EVAL_LIVE_INSTANCE` JSON）使用产品自身的实例形状——`instanceId`、`driver`、`model`、可选的 `config` 和 `environmentFrom`——因此凭据变量从启动 shell 复制（`OMB_EVAL_LIVE_PASS_ENV` 列出要转发的名称），绝不内联。阈值位于已提交的 `evals/live/config.json`：最低套件分数、与上次记录基线的最大漂移、以及轮次和评判超时。场景使用基于状态的等待运行廉价的真实模型、确定性检查（`botReplied`、工具追踪）、以及一个版本化评判器，其提示文件通过清单中的 sha256 固定；针对已 gitignore 的 `evals/live/baselines/` 基线的漂移会使套件失败。

## 技能基准（alpha）

技能基准通过第 1 层相同的脚本化机制，针对夹具技能自身的提示对其进行测量。每个提示运行两次——一次将技能作为用户技能安装在已启动服务器的数据目录下，一次不安装——因此对比覆盖真实接缝：触发词选择、搭载系统提示的 `<openmaus-skill>` 块、以及脚本化跟随者在无技能基线下的行为。它评估测试框架，绝不评估模型。

    node --experimental-strip-types evals/runners/run-skill-bench.ts [--fixture bench-triage-handoff] [--replicates 2]

alpha 报告给出每臂通过率、持续时间和估计 token 数（证据轮次上的字符数/4；脚本化 provider 不进行 API 调用），含均值和标准差、每断言增量表、以及两个标志：无区分度断言（两臂判定相同——是对照，不是效应）和高方差断言（同一提示和臂的重复中判定翻转，从两次重复起可评估）。`evals/skill-bench/fixtures/` 存放基准夹具；`evals/skill-bench/triggers/` 存放触发评估集（应触发/不应触发，含训练和留出划分，加上近似未命中），供后续路由切片验证——仅数据，无路由代码。