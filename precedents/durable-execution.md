TL;DR：持久执行引擎把「一次运行可在进程死亡后从已确认边界继续」做成基础设施；Rust agent kernel 应借其 checkpoint、等待、幂等和可观测性，但不应把通用工作流 DSL、数据库/队列、部署版本路由或 LLM 策略一并内核化。

## 概览

### 研究范围与证据等级

- 目标是比较 Temporal、Restate、DBOS、Inngest 四个持久执行引擎，聚焦 workflow-as-code、确定性重放、activity/step 边界、版本迁移以及其 agent 实践。
- 本文的「文档事实」均来自各产品官方文档或官方示例，链接置于末尾。
- 本文的「推断」是针对一个仅暴露有限 Rust API 的多-agent kernel 的设计判断，不是上述项目的承诺。
- 没有采用二手文章来证明语义；因此「踩坑」以官方明确警告和运行模型可直接推出的风险为主。
- 四者共同提供 durable execution，但实现模型分两类，接口后果显著不同。
- Temporal、Restate、DBOS 更接近 event history / journal 驱动的恢复执行。
- Inngest 明确采用 step-level memoization：每一步由独立 HTTP 请求执行，函数从头再次进入，已完成 step 直接返回保存的结果。
- 对 agent 而言，四者都把 LLM 请求、工具调用、数据库写入等不确定或有副作用的工作放入可持久化边界。
- 它们解决的是「不重复已完成工作、可等待、可重试、可审计」，不是让 LLM 推理本身变成确定性的。

### 四个先例的定位

- Temporal：成熟的 workflow-as-code 平台；Workflow 生成 commands，服务端 Event History 成为重放真相来源。
- Temporal 的核心纪律是 deterministic workflow code；外部 I/O 必须放入 Activity 或通过 SDK 的记录型 API 完成。
- Restate：以 handler/context 为中心的 durable execution runtime；execution log journal 每个已完成的 `ctx.run` 结果。
- Restate 同时提供 service、Virtual Object、Workflow 等服务模型；AI 文档直接将 durable LLM calls 与 durable tools 作为应用模式。
- DBOS：把 workflow/step checkpoint 建在 Postgres system database 上，并把 durable transaction 作为一等抽象。
- DBOS 的文档称 step 至少一次尝试、已完成 step 不再执行，transaction 只提交一次。
- Inngest：面向事件和 serverless HTTP 的 durable functions；step ID 的哈希和计数器定位已保存输出。
- Inngest 将 agent 定位为「runtime-defined、non-deterministic control flow」但通过已记录 step 的短路得到恢复时的相同行进路径。

### 共通运行模型

- 一次逻辑运行有稳定 run/workflow ID，持久存储记录其步骤、等待和终态。
- 可恢复性不是保存任意进程栈；通常是重跑用户函数，或由 event history 驱动重新解释用户代码。
- 因而用户代码必须把有副作用/非确定性操作明确切分为 activity、step 或 `run`。
- 成功完成的边界需要稳定身份；否则新代码无法把历史结果对应到当前代码位置。
- 失败的当前边界可能会被重试；下游系统必须接受至少一次调用，或使用引擎提供的事务/幂等键。
- 长等待不是持有线程：timer、event/signal、human approval 都持久化后挂起，触发时再恢复。
- 代码升级是持久执行的核心难题：旧运行的历史与新代码的 command/step 图不再匹配。

## 核心抽象与接口面

### Temporal：Workflow、Activity、Worker、History、Signal/Query

- 【文档事实】Workflow Definition 是普通语言函数；TypeScript 基本形状为：

```ts
export async function WorkflowExample(
  args: BasicWorkflowArgs,
): Promise<{ result: string }> { /* ... */ }
```

- 【文档事实】同一输入和同一 history 下，Workflow 必须以相同顺序发出相同 Workflow API calls/Commands，才能 replay。
- 【文档事实】一个 activity 调用会生成例如 `ScheduleActivityTask` command；重放时该 command 要与 History 内的 `ActivityTaskScheduled` event 匹配。
- 【文档事实】不匹配即 non-deterministic error；运行中的 workflow 被改变代码是主要触发原因之一。
- 【文档事实】会影响历史顺序的 command-producing 操作包括：schedule/cancel Activity、timer、child workflow、外部 signal、Nexus operation，以及 complete/fail/cancel/continue-as-new。
- 【文档事实】`Patched` / `GetVersion`、`SideEffect` / `MutableSideEffect`、upsert Search Attributes/Memo 也受 replay/patch 规则约束。
- 【文档事实】Activity 在 replay 路径外执行，并自动重试；官方明确列举 API calls、LLM/AI invocations、数据库查询应移入 Activity。
- 【文档事实】这使 API 的自然分层为：Workflow 只编排；Activity 执行现实世界的工作；Worker 同时注册并轮询/执行二者。
- 【文档事实】Worker 或 Temporal service 宕机后，平台恢复 state 并继续；Workflow 代码不需要在进程崩溃处自行捕获该故障。
- 【文档事实】Workflow 对 time、random 等不能用宿主机不稳定来源；应使用 SDK 提供的 replay-safe API。
- 【文档事实】部分改动安全，例如 Activity/Child Workflow 的参数、返回值和超时（但不是 type 或 ID）可改变。
- 【文档事实】定时器 duration 通常可改；Java/Python/Go 中 0 与非 0 互改是非确定性的特例，.NET 还包括 -1。
- 【文档事实】可新增从未收到过的 signal type handler；不发 command 的查询信息调用通常可增删。
- 【文档事实】Temporal 提供两类演进策略：Worker Versioning 与 patching；文档推荐前者，并允许组合使用。
- 【文档事实】patching 的思想是将 `Patched`/`GetVersion` 记录为历史分叉标记，令旧 history 走旧分支、新运行走新分支。
- 【推断】Temporal 的最小可借接口不是其 server/worker 体系，而是「runtime 持久化事件 + 用户可见的稳定版本门」的语义。

### Restate：Handler、Context、durable step、service 模型

- 【文档事实】Restate 用 execution log 在 failure/suspension 后 replay operations；为保证 replay，DB call、HTTP request、UUID generation 等不确定工作必须被 journal。
- 【文档事实】TypeScript durable step 最小 API：

```ts
const result = await ctx.run<string>(async () => doDbRequest());
```

- 【文档事实】带稳定名称和 retry policy 的形状为：

```ts
await ctx.run("write", () => writeToOtherSystem(), {
  initialRetryInterval: { milliseconds: 500 },
  retryIntervalFactor: 2,
  maxRetryInterval: { seconds: 1 },
  maxRetryAttempts: 5,
  maxRetryDuration: { seconds: 1 },
});
```

- 【文档事实】`ctx.run` 成功结果默认 JSON 序列化并写入 journal，恢复时使用历史值；自定义 serialization 另有配置接口。
- 【文档事实】`ctx.run` closure 内不得使用 Restate context：不能 `ctx.get`、`ctx.sleep`、嵌套 `ctx.run`。
- 【文档事实】普通 handler error 与 run error 会按策略重试；显式抛 `TerminalError` 则为非重试终止。
- 【文档事实】无新 journal entry 超过默认一分钟，Restate 会 suspend 并重试 invocation；multi-minute LLM 调用需要调 service inactivity/abort timeout。
- 【文档事实】Restate 提供 replay-safe helpers：`ctx.rand.uuidv4()`、`ctx.rand.random()`、异步的 `await ctx.date.now()`。
- 【文档事实】这些 helper 从 invocation ID 派生，在重试中值相同；随机 UUID 不可用于密码学用途。
- 【文档事实】Restate 的服务概念还包括 Service、Virtual Object（有持久状态的按 key 实体）与 Workflow；它们是服务编程模型而非单纯函数调用。
- 【文档事实】其 agent 示例将 handler、persisted LLM calls、durable tool executions 作为三部分。
- 【文档事实】Vercel AI SDK 集成用如下 middleware 持久化模型响应：

```ts
const model = wrapLanguageModel({
  model: openai("gpt-5.4"),
  middleware: durableCalls(ctx, { maxRetryAttempts: 3 }),
});
```

- 【文档事实】该示例称每个 LLM response 保存于 Restate Server，恢复时从 journal replay。
- 【文档事实】工具实现必须拿到 `ctx` 并用 `ctx.run()`，例如 ``ctx.run(`get weather ${city}`, () => ...)``。
- 【文档事实】官方 Restate-only 示例将手写 agent loop 中每个 LLM call 包进 `ctx.run("LLM call", ...)`。
- 【文档事实】官方表述：工具 side effect 只执行一次；崩溃后从第一个未完成 step 继续，因此可避免重复 LLM 账单、重复发邮件/预订。
- 【文档事实】官方示例含 `parallelToolCalls: false`；文档未将其解释为普遍限制，不能据此断言平台不支持并行工具。
- 【推断】`ctx.run(name, closure, retry)` 是最值得借鉴的紧凑接口：把 checkpoint identity、effect boundary、retry policy 放在一次调用中。

### DBOS：Workflow、Step、Transaction、Queue、Application Version

- 【文档事实】DBOS 将 workflow/step state checkpoint 到 Postgres system database，恢复时从最后完成的 step 继续。
- 【文档事实】函数式注册 API 包括：

```ts
DBOS.registerWorkflow(workflowFunction);
await DBOS.runStep(() => stepOne(), { name: "stepOne" });
await DBOS.sleep(ms);
```

- 【文档事实】也可用 `@DBOS.workflow()` 和 `@DBOS.step()` 装饰 static class methods。
- 【文档事实】workflow 输入/输出必须可 JSON 序列化。
- 【文档事实】后台调用形状：

```ts
const handle = DBOS.startWorkflow(taskWorkflow, {
  queueName: "example_queue",
})(arg);
const result = await handle.getResult();
```

- 【文档事实】相关 API 有 `DBOS.registerQueue`、`DBOS.retrieveWorkflow`、`DBOS.workflowID`、`DBOS.setConfig`、`DBOS.launch()` 与 `DBOS.shutdown()`。
- 【文档事实】调用者可指定 `{ workflowID: myID }`；它必须全局唯一，重复使用同一 ID 仅执行一次，因而可作为幂等 key。
- 【文档事实】workflow 必须 deterministic：同输入下按同序调用相同 steps、使用相同 inputs。
- 【文档事实】数据库访问、第三方 API、random、local time 不可直接写在 workflow；应包进 step。
- 【文档事实】并行可用 `Promise.allSettled`，前提是 step 启动顺序确定；不得在 async closure 中嵌套 await，否则顺序取决于时序。
- 【文档事实】文档不建议 `Promise.all`：一个 promise reject 时其余 unresolved，之后可能造成 Node 未处理异常；并发序列建议 child workflow + handles。
- 【文档事实】DBOS 声称 workflow 会 run to completion、step 至少一次尝试但完成后不再执行、transaction exactly once commit。
- 【文档事实】未捕获异常将 workflow 标为 `ERROR` 且不恢复；transient failure 应落在可配置 retry 的 step。
- 【文档事实】durable timeout 可取消 workflow 及所有 child；`DBOS.sleep` 持久化等待。
- 【文档事实】其升级定义：改变「哪些 steps 运行」或「steps 顺序」是 breaking change，恢复时可发生 checkpoint mismatch。
- 【文档事实】patch 需 `enablePatching: true`；`DBOS.patch()` 对更新后启动的 workflow 返回 true、此前启动的返回 false。
- 【文档事实】patch 会在 `operation_outputs` 中放 patch marker；错误的未 patch 改动或过早移除会得到 `DBOSUnexpectedStepError`。
- 【文档事实】移除 patch 的严谨顺序是：等旧运行结束，先用 `DBOS.deprecatePatch()` 过渡，再等含 marker 的运行结束，最后才删代码。
- 【文档事实】另一策略为版本隔离：每个 workflow 标注其启动时 application version，DBOS 只恢复与当前进程 version 相配的 workflow。
- 【文档事实】默认 version 是 workflow source hash，也可在配置设置 `applicationVersion`。
- 【文档事实】推荐 blue-green：保留旧进程 drain 旧运行，新流量进新版本；用 list API 确认旧版本无 active workflow 后再下线。
- 【推断】DBOS 把 durable transaction 绑到 Postgres 是强大但特定部署承诺；作为通用 agent kernel 默认依赖过重。

### Inngest：Event Function、Step、持久等待、flow control

- 【文档事实】Inngest 的 function API 基本形状：

```ts
const fn = inngest.createFunction(
  { id: "import-contacts", triggers: { event: "contacts/csv.uploaded" } },
  async ({ event, step }) => {
    const rows = await step.run("parse-csv", async () => parseCsv(event.data.fileURI));
    return { results: rows };
  },
);
```

- 【文档事实】每个 `step.run` 是独立可运行、可 retry 的单位；成功后不会再执行，返回保存的输出。
- 【文档事实】初次运行先发现一个 step、执行 callback、保存结果并中断函数；后续 HTTP request 会重新调用 handler。
- 【文档事实】重进时 SDK 用 step identifier 的 hash 查找已保存结果，跳过 callback、将结果作为 `step.run` 返回，直到发现未执行 step。
- 【文档事实】step ID hash 还包含 counter，从而区分 loop 内同名 step 的多次调用。
- 【文档事实】函数外或 step 外的 DB/API call 会在每次 re-invocation 重复；官方要求所有非确定性工作位于 `step.run`。
- 【文档事实】step 抛错被 SDK 捕捉、序列化上报；剩余 attempts 时，带着之前 step state 从失败点恢复。
- 【文档事实】attempt 耗尽后，函数会以该 error 再次运行，用户可 catch 后做 fallback。
- 【文档事实】持久控制 API 包括 `step.sleep()` / `step.sleepUntil()`、`step.waitForEvent()`、`step.waitForSignal()`、`step.invoke()`。
- 【文档事实】`waitForEvent` 用于 approval/webhook/agent message，挂起时不占 process、connection 或资源。
- 【文档事实】`step.invoke()` 调用另一 Inngest function/sub-agent 并等待 result；它是官方 orchestrator-worker 组合模式的基础。
- 【文档事实】平台另有 concurrency、throttling、batching、rate limiting、singleton/debounce、priority 等 function-level flow-control 特性。
- 【文档事实】其 code evolution 不用显式 version marker：完成 step 跨 deployment 不重跑；新发现的 step 会执行，调换顺序记 warning 而不使运行失败。
- 【文档事实】保持 step ID 修改逻辑对进行中的运行安全：旧运行取得 memoized output，新运行走新 logic。
- 【文档事实】改 step ID 会强制重新执行；官方建议 ID 描述性、稳定且唯一，例如 `calculate-risk-score-v2`。
- 【文档事实】删 step 时旧 memoized data 留在 state 但会被忽略；加 step 时新 step 不得依赖尚未执行的 step 数据。
- 【文档事实】大的不兼容重写推荐 new-function pattern：两个 function 订阅同 event，按 event timestamp 或 version 作 cutover routing，等待 v1 run 排空。
- 【文档事实】Inngest 的 durable-agent 指南称 agent 的每个 step/tool call/decision 都 checkpoint；若第 7/12 步死亡，从第 7 步而非第 1 步继续。
- 【文档事实】agent loop 的 observe→think→act 每轮可为独立 checkpoint；图随 agent 运行生成，而不是预先静态 DAG。
- 【文档事实】agent/sub-agent 可通过 `step.invoke` 持久委派，或异步 events 协作；human-in-the-loop 可用 `waitForEvent` 暂停。
- 【文档事实】当前正在执行的 step 不会被保存，故最多丢失当前 step；文档也限定「从 failure point resume」只在某些场景可用。
- 【推断】Inngest 的 replay 封装较轻，适合 HTTP/serverless；但 step identity 与重入纪律仍是应用作者不可忽略的正确性契约。

## 它把什么放进核心、什么留给使用者

### 放进核心的职责（四者共识）

- 【文档事实】为 run/workflow/step 分配或接收稳定 ID，并保存 lifecycle、history/journal/checkpoint 和终态。
- 【文档事实】把成功边界的 outputs/durable waits 持久化，进程故障后重调度并恢复。
- 【文档事实】提供 retry、timer/sleep、外部 event/signal/await、child/invoke 等控制面 primitives。
- 【文档事实】提供可查询 handle、运行状态和步骤 trace/observability；Restate UI 与 Inngest traces 都显示 step 级信息。
- 【文档事实】控制并发/队列/限流是 DBOS 与 Inngest 的产品能力；Temporal/Restate 亦有其 worker/service 调度模型。
- 【文档事实】版本迁移必须由平台参与，至少提供 patch/history marker 或运行版本隔离。

### 留给库使用者/应用层的职责

- 【文档事实】用户写 workflow/handler 的正常控制流、数据 schema、step/activity 名称、retry 耐受策略和 compensation。
- 【文档事实】用户决定哪些操作是 effect boundary，并确保未完成 effect 可以重试或幂等。
- 【文档事实】用户管理下游 API 的幂等 key、外部 credential、支付/邮件等业务补偿；引擎不自动令任意第三方 API exactly-once。
- 【文档事实】用户要设计版本兼容/patch 生命周期或 blue-green drain，平台不能从语义上自动修复不兼容的流程改写。
- 【文档事实】LLM model、prompt、工具集合、停止条件、审批策略、agent memory 内容，均在官方示例中属于应用代码。
- 【推断】前端应拥有 run 可视化、人工介入、重试/取消触发与 trace 展示的产品体验；kernel 提供结构化状态/事件，不应硬编码 UI。
- 【推断】跨 agent 的角色分配、模型选择、预算、上下文压缩、质量评估不属于 durable runtime 的最小职责，应由上层 policy/orchestrator 决定。

## 踩坑与教训

### 确定性不是口号，而是 API 约束

- 【文档事实｜Temporal】修改仍在运行 workflow 的 command 顺序会触发 non-deterministic error；不能只把 workflow 当普通可热更新函数。
- 【文档事实｜DBOS】直接读取 local clock/random/DB 或按异步完成顺序开启 step 都破坏 deterministic workflow；并行启动顺序必须可复现。
- 【文档事实｜Restate】`ctx.run` 内嵌 context call 被禁止；否则 journal/replay 的边界语义无法保持简单。
- 【文档事实｜Inngest】step 外副作用在每次 HTTP re-invocation 执行；即使所谓「函数从头重跑」在源码上看不明显，也会重复发请求。
- 【推断】Rust kernel 需要将 effect boundary 设计得显式且类型可见，不能希望 agent 作者仅凭约定记住「不要在普通 async 代码里发请求」。

### 版本迁移的成本常被低估

- 【文档事实｜DBOS】patch 不是一行条件即可结束：必须等待旧运行、deprecate 过渡、再等含 marker 运行结束，过早删除产生 unexpected-step recovery error。
- 【文档事实｜Temporal】patch 与 worker versioning 都存在，说明长期运行需要同时考虑单个流程内部兼容和 worker code revision 路由。
- 【文档事实｜Inngest】稳定 step ID 让小改更宽容，但换 ID 则是明确的重新执行；添加 step 还不能依赖尚未产生的数据。
- 【推断】agent kernel 若没有 migration API，至少要把 `kernel_api_version`、step/effect key 与 replay incompatibility error 写成稳定协议，而非隐式实现细节。

### 「已完成不重跑」不等于端到端 exactly once

- 【文档事实｜DBOS】其限定语是 step 至少一次尝试、完成后不再执行；事务才有 exactly-once commit 的独立保证。
- 【文档事实｜Restate】其 agent 文档将 journaled tools 描述为 effect exactly once，但实际下游 side effect 仍应按应用的失败窗口与幂等语义评估。
- 【推断】不能把本地 checkpoint 成功和远端 API 确认视为天然原子；kernel 的 effect API 应支持 caller 提供 idempotency key 和 result receipt。

### LLM/agent 特有的执行问题

- 【文档事实｜Restate】长 LLM call 可超过默认 inactivity window，需配置 timeout；否则 durable engine 的故障检测本身会干扰调用。
- 【文档事实｜Restate/Inngest】LLM responses 只有被 durable middleware/`ctx.run`/`step.run` 包住才可避免恢复时重付费。
- 【文档事实｜Inngest】agent 的路径可 runtime-defined，故仅阅读代码不足以解释生产行为；step trace 是调试所需的第一等产物。
- 【推断】模型输入、输出和工具参数往往包含敏感数据；持久化 trace 的 retention、redaction、访问控制必须由应用/部署策略提供，kernel 至多暴露 hook。

## 对我们 Rust agent kernel 的启示

### 应学习：最小而完整的 durable protocol

- 【推断】核心应将一个 `RunId` 对应到 append-only 的 `RunEvent`/checkpoint log，并允许从指定 checkpoint 恢复；不要要求业务保存进程栈。
- 【推断】最小控制接口可围绕：`start`、`poll/next_command`、`complete_effect`、`fail_effect`、`deliver_signal`、`cancel`、`inspect`，而不是直接暴露数据库表或 worker internals。
- 【推断】将执行划为两种：纯 reducer/决策（可 replay）与带稳定 `EffectKey` 的 external effect（可记录结果）。
- 【推断】`EffectKey` 必须由 run 内稳定 logical position + 调用者显式 name 组成，而不是随机 UUID；它对应 Temporal command、Restate run name、DBOS step name、Inngest step ID 的共同本质。
- 【推断】effect completion payload 应持久化 `{key, attempt, input_digest, output/result_or_error, timestamps}`；重放只把已记录 completion 喂回 reducer。
- 【推断】接口应含 `Wait { until | signal_filter }`、`Spawn { child_run }`、`Join`/message delivery，而不是把 timer/event polling 让每个前端自行造轮子。
- 【推断】run status 至少区分 `Running`、`Waiting`、`RetryScheduled`、`Succeeded`、`Failed`、`Cancelled`、`NeedsMigration`，供前端直接渲染和操作。
- 【推断】副作用执行器、存储、队列、HTTP 客户端、LLM provider 都应为 adapter/trait，由库使用者注入；kernel 只判定 command 与记录状态。

### 应学习：演进和可观测性作为一等接口

- 【推断】定义可检查的 `KernelRevision`/`WorkflowRevision`，在恢复时比较历史 revision；不兼容时明确进入 `NeedsMigration`，绝不悄悄按新逻辑吞掉历史。
- 【推断】轻量方案可先支持「revision pinning + drain」：老 run 留给旧 executor，新 run 指向新 revision；这是比通用 patch DSL 更低复杂度的起点。
- 【推断】如日后加入 patch，API 应强制带命名 patch ID、activate/deprecate/retire 生命周期和「仍有多少 run 使用它」的查询。
- 【推断】记录结构化 lifecycle/effect events，并稳定暴露 `inspect_run`/event stream；前端自行组成 time line、agent graph 和审批界面。
- 【推断】trace 默认只存 metadata/digest，完整 prompt/tool I/O 经应用批准才由加密或外部 blob store 保存，避免 durable trace 变成无边界数据仓库。

### 不应照搬：对我们的最小库过重的部分

- 【推断】不内嵌 Temporal 式 cluster、task queue、worker polling 协议、search attributes、Nexus 或跨语言 SDK；这些是部署平台能力。
- 【推断】不内嵌 DBOS 的 Postgres system schema、SQL transaction runtime 和 Conductor；可提供 storage trait，让 adopter 选择 SQLite/Postgres/远端服务。
- 【推断】不内嵌 Inngest 的 event bus、HTTP serving、multi-tenant rate-limit/batching 产品；前端/宿主或部署层可选择实现。
- 【推断】不把 Service/Virtual Object actor 语义与 agent run 强制合一；持久 agent memory 可由一个可选 state-store trait 或上层 actor 系统提供。
- 【推断】不把 Vercel AI/LangChain/具体 provider middleware 变成 kernel API；只将 LLM 调用表达为普通 effect，并给 provider adapter 机会设置 idempotency/retry/timeout。
- 【推断】不尝试在 v1 自动证明任意 Rust closure 的确定性；让 host 显式声明 effect，配合 replay test harness、lint 和运行时 mismatch detection。

### 可落地的 Rust 轮廓（推断，不是现成产品 API）

```rust
pub trait RunStore {
    fn append(&self, run: RunId, event: RunEvent) -> Result<(), StoreError>;
    fn history(&self, run: RunId) -> Result<Vec<RunEvent>, StoreError>;
}

pub enum KernelCommand {
    Effect { key: EffectKey, kind: EffectKind, input: Bytes, retry: RetryPolicy },
    Wait { key: WaitKey, condition: WaitCondition },
    Spawn { child: RunSpec },
    Complete { output: Bytes },
}

pub trait Kernel {
    fn drive(&self, run: RunId) -> Result<KernelCommand, KernelError>;
    fn resolve_effect(&self, run: RunId, key: EffectKey, result: EffectResult)
        -> Result<(), KernelError>;
    fn deliver_signal(&self, run: RunId, signal: Signal) -> Result<(), KernelError>;
}
```

- 【推断】该轮廓故意不出现 HTTP、LLM SDK、SQL、Tokio task 或 UI widget；它只定义 durable state transition 的窄腰。
- 【推断】`drive` 应只能基于 run history 和稳定输入作决定；任何宿主 I/O 必须由 `Effect` command 返回给 host 后再 `resolve_effect`。
- 【推断】为防止 key 漂移，可让 macro/host 显式提供 human-readable effect name，并在 development mode 保存调用路径 digest 作 mismatch 诊断。
- 【推断】多-agent 协作可先降解为 child run、message signal、shared state adapter 三种，不要一开始设计独立的 agent graph scheduler。

## 来源

### Temporal（官方文档）

- Workflow Definition、determinism、command/history 对齐、Activity 边界、safe changes：https://docs.temporal.io/workflow-definition
- Temporal Event History 概念：https://docs.temporal.io/encyclopedia/event-history/
- TypeScript Workflow basics：https://docs.temporal.io/develop/typescript/workflows/basics
- TypeScript patching/versioning：https://docs.temporal.io/develop/typescript/workflows/versioning#patching
- Worker Versioning：https://docs.temporal.io/worker-versioning

### Restate（官方文档与官方示例）

- Durable steps（`ctx.run`、replay-safe rand/time、closure 限制、retry/timeout）：https://docs.restate.dev/develop/ts/durable-steps.md
- Key concepts：https://docs.restate.dev/foundations/key-concepts.md
- TypeScript services：https://docs.restate.dev/develop/ts/services.md
- Service models / Virtual Objects：https://docs.restate.dev/foundations/services.md
- Durable agents（Vercel AI middleware、manual loop、tool durability）：https://docs.restate.dev/ai/patterns/durable-agents.md
- Service versioning：https://docs.restate.dev/services/versioning.md
- 官方 Vercel AI template：https://github.com/restatedev/ai-examples/blob/main/vercel-ai/template/src/app.ts
- 官方 Restate-only agent template：https://github.com/restatedev/ai-examples/blob/main/typescript-restate-only/template/src/agent.ts

### DBOS（官方文档）

- TypeScript programming guide（注册、queue、恢复、Conductor）：https://docs.dbos.dev/typescript/programming-guide
- Workflow tutorial（determinism、parallelism、workflow guarantees、IDs）：https://docs.dbos.dev/typescript/tutorials/workflow-tutorial
- Workflow code upgrading（patch、deprecate、version pinning、blue-green）：https://docs.dbos.dev/typescript/tutorials/upgrading-workflows
- Transaction tutorial：https://docs.dbos.dev/typescript/tutorials/transaction-tutorial
- DBOS class reference：https://docs.dbos.dev/typescript/reference/dbos-class

### Inngest（官方文档）

- How functions are executed（独立 HTTP step、memoization、side-effect 边界）：https://www.inngest.com/docs-markdown/learn/how-functions-are-executed
- Versioning（step IDs、safe/unsafe change、new-function migration）：https://www.inngest.com/docs-markdown/learn/versioning
- Durable agents（agent loop、sub-agent、human wait、限制）：https://www.inngest.com/docs-markdown/learn/durable-agents
- Inngest steps：https://www.inngest.com/docs-markdown/learn/inngest-steps
- Wait for event：https://www.inngest.com/docs-markdown/features/inngest-functions/steps-workflows/wait-for-event
- Error retries：https://www.inngest.com/docs-markdown/features/inngest-functions/error-retries/retries
- Flow control：https://www.inngest.com/docs-markdown/guides/flow-control
