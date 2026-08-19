# Agent Kernel 开发者关注点规范税系

> 研究对象：将多 agent 协作运行时抽成仅暴露有限接口的 Rust 依赖库。  
> 输入：14 个架构先例（P1–P14）与 10 个领域解剖（D1–D10），共 24 个来源。  
> 判定日期：2026-08-18。

## 0. 判定口径

### 0.1 五种边界标签

- **kernel**：若不由内核统一维护，就无法保证跨调用、跨恢复或跨并发参与者仍成立的运行时不变量。内核只负责最小机制、状态与强制点，不负责领域策略。
- **lib-user**：库使用者应通过内核原语组装或注入实现；内核最多定义 trait/DTO/验证边界。其选择通常依赖部署、存储、供应商或产品策略。
- **frontend**：可从权威 snapshot/event 重建的展示和交互投影；前端绝不能成为生命周期、授权或调度事实的第二真相源。
- **app**：成功标准、业务含义、专业约束、优化目标或 agent“怎么想”；换领域就会改变，不能伪装成通用机制。
- **contested**：成熟先例或领域需求对归属有实质冲突，或同一关注点必须跨 kernel/使用者共同完成，强行归入一侧会作出虚假保证。

### 0.2 对抗式筛选规则

1. **“很多应用都需要”不等于“内核必须内置”**。只有内核能独占维护的不变量才进入 kernel。
2. **内核可以强制上限，不应决定优化政策**：硬预算、租约 fencing、状态转移校验可进内核；候选排序、重试条件、价格/安全阈值不能进。
3. **协议语义与实现后端分离**：恢复、revision、cursor 是内核契约；数据库、队列、Tokio、HTTP、MCP、模型供应商是 adapter。
4. **事实与投影分离**：run 终态、effect outcome、approval decision 是事实；聊天气泡、agent 头像、图布局、token 动画是投影。
5. **不承诺无法端到端兑现的语义**：checkpoint 不等于第三方副作用 exactly-once；capability revoke 不等于收回已泄露数据；取消接受不等于物理动作已停止。
6. **证据计数是“该来源明确支持该关注点或明确给出边界反证”的数量**，不是投票。横跨先例与领域的重复出现优先于单一路径的高频出现。

## 1. 总览

| # | 关注点簇 | 裁决 | 证据强度 |
|---|---|---|---|
| 1 | Run/Task 生命周期与终态 | kernel | 24/24：14P + 10D |
| 2 | 稳定身份、父子血缘与因果关联 | kernel | 24/24：14P + 10D |
| 3 | 命令—事件—快照的一致观察协议 | kernel | 24/24：14P + 10D |
| 4 | 持久状态、恢复、重放与 revision 兼容 | kernel | 22/24：12P + 10D |
| 5 | 外部 effect/tool 调用边界与事实账本 | kernel | 21/24：11P + 10D |
| 6 | 副作用幂等、重试、补偿与 exactly-once | contested | 22/24：12P + 10D |
| 7 | 暂停、Signal、人工输入与审批握手 | kernel | 20/24：10P + 10D |
| 8 | Capability、委托、撤销与执行前策略强制 | kernel | 22/24：12P + 10D |
| 9 | 预算、配额、并发、背压与 admission | kernel | 22/24：12P + 10D |
| 10 | Deadline、Timer、Lease、Heartbeat 与 fencing | kernel | 18/24：8P + 10D |
| 11 | 多 agent 消息、Spawn/Join、监督与失败传播 | kernel | 19/24：9P + 10D |
| 12 | Workflow/DAG/协议 FSM/组织编排 | contested | 20/24：10P + 10D |
| 13 | 版本化 workspace、artifact 与 provenance | kernel | 20/24：10P + 10D |
| 14 | “强事实/弱观测”、新鲜度、置信度与领域有效性 | app | 14/24：5P + 9D |
| 15 | Agent 认知与业务决策语义 | app | 24/24：14P + 10D |
| 16 | Store/Executor/Provider/Transport/Sandbox 实现 | lib-user | 24/24：14P + 10D |
| 17 | 展示、交互、视图状态与产品工作台 | frontend | 24/24：14P + 10D |
| 18 | Rust 公共 API、版本化 DTO、sans-I/O 与 FFI 纪律 | kernel | 18/24：14P + 4D |

---

## 2. 逐簇判定

## 1) Run/Task 生命周期与终态

**裁决：kernel**  
**证据强度：24/24；P1–P14，D1–D10；双路全覆盖。**

### 关注点

`Pending/Running/Paused/Cancelling/Completed/Failed/Cancelled` 等小而稳定的状态机；合法转移；start/cancel/join/inspect；明确 terminal reason；取消接受、停止完成和取消超时的区分。

### 证据清单

- 先例：P1 的可观察 run、P2 的 session/turn/终止、P3 的暂停恢复取消、P4 的 durable run、P5 actor stop/join、P6 task 合法转移、P7 action lifecycle、P8 请求必闭合、P9 权威运行状态、P10 显式 Result、P11 cancel/revoke、P12 API 对象状态、P13 step/run 状态、P14 sans-I/O 状态机。
- 领域：D1–D10 均要求任务/计划/命令的持久状态、取消、接管或终态，虽业务状态名不同。
- 代表出处：[ROS 2 Actions](https://design.ros2.org/articles/actions.html)、[LSP 3.17 cancellation](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/)、[Temporal workflow history](https://docs.temporal.io/encyclopedia/event-history/)。

### 边界论证

只有内核能保证所有命令、事件、等待者和恢复路径看到同一终态，并确保 terminal event 恰好闭合 run。若由前端或业务代码各自推断，“最后一个 token”“Future 被 drop”“worker 消失”会被错误当成完成。

**不进入内核**：业务 phase（例如“已退款”“起飞批准”“DRC clean”）、成功判据、取消费、安全复位动作。它们映射到通用状态机，但不扩充核心 enum。

**反方挑战**：“状态机因领域差异太大，应全部交给应用。”  
**回应**：领域状态确实在应用；但能否接受命令、是否正在等待、是否 terminal、取消是否完成是运行时资源与句柄安全的共同不变量，不能外包。

## 2) 稳定身份、父子血缘与因果关联

**裁决：kernel**  
**证据强度：24/24；P1–P14，D1–D10；双路全覆盖。**

### 关注点

`RunId/TaskId/AttemptId/CallId/AgentId`、parent-child lineage、`correlation_id/causation_id`、稳定逻辑 effect key、actor/subject 归因；ID 不等于授权凭据。

### 证据清单

- 先例全部要求 run、goal、actor、resource、message、delegation 或 call 的稳定身份及关联。
- 领域全部需要把决策、命令、artifact、审批和执行证据锚定到同一任务/版本/主体。
- 代表出处：[FIPA ACL message structure](http://www.fipa.org/specs/fipa00061/SC00061G.html)、[Claude Agent SDK sessions](https://code.claude.com/docs/en/agent-sdk/sessions)、[WASI capabilities](https://github.com/WebAssembly/WASI/blob/main/docs/Capabilities.md)。

### 边界论证

身份是去重、取消传播、审计、预算树和恢复匹配的连接键；如果由 adapter 或前端临时生成，同一 effect 可能在重放时换 ID，破坏幂等与归因。内核必须生成或校验其唯一性和作用域。

**不进入内核**：用户账号体系、组织结构、司机/机器人/芯片 block 的业务主键语义、显示名与头像。外部主体映射到 `SubjectId` 的过程由宿主负责。

## 3) 命令—事件—快照的一致观察协议

**裁决：kernel**  
**证据强度：24/24；P1–P14，D1–D10；双路全覆盖。**

### 关注点

版本化 `Command`、有序 `KernelEvent`、权威 `Snapshot`、opaque cursor/revision、订阅容量与 overflow、断线重放、`GapDetected/RevisionExpired` 后 resnapshot；命令事实与 token/telemetry UX 流分层。

### 证据清单

- 先例：P1/P2 agent stream，P4 event history，P6 dispatch→events，P8 Jupyter/LSP/DAP，P9 xi 增量 cache 教训，P12 list→watch，P14 bounded stream 等共同收敛。
- 领域 D1–D10 都要求时间线、审计、订阅或重放；物理领域还要求乱序、去重与 TTL。
- 代表出处：[Kubernetes API list/watch](https://kubernetes.io/docs/reference/using-api/api-concepts/)、[Jupyter messaging](https://jupyter-client.readthedocs.io/en/latest/messaging.html)、[xi frontend protocol](https://xi-editor.io/docs/frontend-protocol.html)。

### 边界论证

前端重连是否能恢复一致状态取决于事件序列、snapshot revision 和 cursor 生命周期，必须由事实源统一定义。只暴露 callback 或日志字符串会让 UI 成为隐式状态机；只暴露 event 又没有 snapshot/resync 会在任何丢包后永久漂移。

**最小契约**：`start/command/inspect/subscribe(cursor)`；terminal、排序范围、overflow、handle expiry 明示。  
**排除项**：每个 token、动画帧、markdown patch 不必 durable；live delta 可有界合并或丢弃，但不可冒充权威事件。

## 4) 持久状态、恢复、重放与 revision 兼容

**裁决：kernel**  
**证据强度：22/24；P1、P2、P3、P4、P6、P7、P8、P9、P10、P12、P13、P14 + D1–D10。**

### 关注点

checkpoint/journal 的语义、恢复入口、历史 completion 回灌、revision pinning、schema/artifact version、历史不兼容错误、drain/migration；不保存任意进程栈。

### 证据清单

- P4 的 durable history、P12 的 revision/CAS、P1 的 interrupt 重放、P10/P14 的 snapshot/journal 分层构成直接先例。
- D1–D10 均要求故障恢复、回放、冻结版本或重连对账。
- 反证也重要：P5 actor 系统把 durable checkpoint 放到可选层，说明**存储实现**不应焊死在最小内核。
- 代表出处：[Temporal workflow versioning](https://docs.temporal.io/develop/typescript/workflows/versioning#patching)、[DBOS upgrading workflows](https://docs.dbos.dev/typescript/tutorials/upgrading-workflows)、[LangGraph interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)。

### 边界论证

恢复后的状态转移能否与历史相容是内核语义；否则库无法保证 pause/resume、去重或等待句柄有效。但数据库、序列化介质与 retention 是 lib-user 的 adapter/政策。

**内核最小责任**：定义可序列化 snapshot/journal、revision、恢复校验与 `NeedsMigration`。  
**不承诺**：任意 Rust closure 可确定重放；v1 可用显式 command/effect 和 revision pinning，patch DSL 后置。

## 5) 外部 effect/tool 调用边界与事实账本

**裁决：kernel**  
**证据强度：21/24；P1、P2、P3、P4、P5、P6、P7、P10、P11、P13、P14 + D1–D10。**

### 关注点

所有 LLM/tool/sub-agent/外部命令统一表示为受调度 effect：`propose → authorize → execute → record outcome`；包含稳定 call/effect key、typed input/output、attempt、deadline、cancel、资源提示和 machine-readable error。

### 证据清单

- P4 durable command、P10 host request/response、P11 capability invoke 是最强机制证据；P1/P2 的真实 tool event 证明不能从模型文本推断执行事实。
- D1–D10 全部要求外部 Git/CI、抓取、CRM、机器人、派单、空管、电网或 EDA 操作的回执与证据。
- 代表出处：[Restate durable steps](https://docs.restate.dev/develop/ts/durable-steps.md)、[OpenAI Agents streaming](https://openai.github.io/openai-agents-python/streaming/)、[SQLite authorizer](https://sqlite.org/c3ref/set_authorizer.html)。

### 边界论证

副作用前的安全强制点与副作用后的权威 outcome 必须由内核串联；否则模型可声称“已执行”而 executor 未运行，或自动批准绕过 policy hook。executor 实现仍由宿主提供，内核不做 HTTP、shell、模型调用本身。

**最小接口提示**：`EffectRequest` 与 `resolve_effect(call_id, Result<typed_output, ErrorKind>)`，禁止同一 Run hook 递归 drive/mutate。

## 6) 副作用幂等、重试、补偿与 exactly-once

**裁决：contested**  
**证据强度：22/24；P1、P2、P3、P4、P5、P6、P7、P10、P11、P12、P13、P14 + D1–D10。**

### 内核侧论据

- 重放会重复进入 step；稳定 `EffectKey`、attempt ledger、claim-before-execute、去重 completion 和 retry budget 若不在内核，跨 worker 崩溃后无法一致。
- 工作流/持久执行先例（P4、P13）和 10 个领域都需要有限重试、分类错误、outbox/inbox 或补偿钩子。
- CrewAI 重试重复工具、Inngest step 外 I/O 重复等案例说明“让 agent 自己小心”不成立。

### lib-user/app 侧论据

- 第三方系统是否支持 idempotency key、业务 upsert、receipt、事务或补偿，内核无法知道。
- 退款、派单、机器人移动、开断电网、修改版图的补偿含义完全不同；错误分类和“可安全重试”是领域政策。
- actor、durable execution、LLM framework 先例都拒绝从 checkpoint 推导端到端 exactly-once。

### 裁决理由

这是一个**跨边界义务**，不能诚实地整体归入 kernel 或 lib-user：

- kernel 必须提供：稳定 effect key、intent/claim/attempt/outcome 账本、去重与最大尝试硬限、恢复时历史结果回灌；
- lib-user/app 必须提供：下游幂等实现、retry classifier、补偿动作及人工介入语义；
- 公共 API 必须明确标注交付语义，禁止宣传通用 exactly-once。

代表出处：[Inngest execution model](https://www.inngest.com/docs-markdown/learn/how-functions-are-executed)、[CrewAI issue #5802](https://github.com/crewAIInc/crewAI/issues/5802)、[Orleans best practices](https://learn.microsoft.com/en-us/dotnet/orleans/resources/best-practices)。

## 7) 暂停、Signal、人工输入与审批握手

**裁决：kernel**  
**证据强度：20/24；P1、P2、P3、P4、P6、P7、P8、P10、P11、P13 + D1–D10。**

### 关注点

`Paused/WaitingForSignal` 一等状态；`request_id`、typed payload、deadline、expiry、controller ownership；approve/deny/respond 幂等；断线重派；人工接管时冻结冲突自动写入。

### 证据清单

- P1/P2 的 tool approval、P4 durable signal、P8 reverse input 与 stale handle、P13 interactive workflow 直接支持。
- D1–D10 全部出现人工审批、接管、升级、见证或签核。
- 代表出处：[OpenAI Agents HITL](https://openai.github.io/openai-agents-python/human_in_the_loop/)、[Prefect interactive workflows](https://docs.prefect.io/v3/advanced/interactive.md)、[DAP specification](https://microsoft.github.io/debug-adapter-protocol/specification)。

### 边界论证

等待输入会冻结 run、持有资源、影响 deadline 与恢复，故握手状态和 ID 必须归 kernel。console input 或异常字符串无法支持断线、超时和重放。

**不进入内核**：谁有权批准、文案、审批 SLA、拒绝后的业务路线、双人审批规则。授权政策由 app/lib-user，弹窗与表单由 frontend。

## 8) Capability、委托、撤销与执行前策略强制

**裁决：kernel**  
**证据强度：22/24；P1、P2、P3、P5、P6、P7、P8、P9、P10、P11、P13、P14 + D1–D10。**

### 关注点

opaque capability handle、按资源/operation 缩权、delegate、subtree revoke、subject/delegation attribution、执行前不可绕过 policy hook、审计；agent/tool 名和 JSON 参数不是凭据。

### 证据清单

- P11 seL4/WASI/object-capability 给出最直接原则；P2 的 permission precedence 和 bypass 风险给出 agent 场景反证。
- D1–D10 全部要求短期、作用域化、可撤销授权及审计。
- 代表出处：[seL4 CSpace](https://github.com/seL4/seL4/blob/master/manual/parts/cspace.tex)、[WASI design principles](https://github.com/WebAssembly/WASI/blob/main/docs/DesignPrinciples.md)、[Claude Agent SDK permissions](https://code.claude.com/docs/en/agent-sdk/permissions)。

### 边界论证

内核是所有 effect 必经的中介，只有它能保证“未持有 capability 就不执行”及撤销后拒绝未来调用。若只由 prompt 或 UI 检查，后台任务、恢复路径或高权限模式可绕过。

**不进入内核**：企业 RBAC/ABAC 规则、OAuth 登录、密钥保管、租户组织结构、谁可审批。宿主把这些决定翻译为 capability grant/revoke。内核也不能撤回已泄露数据或已发生物理副作用。

## 9) 预算、配额、并发、背压与 admission

**裁决：kernel**  
**证据强度：22/24；P1、P2、P3、P4、P5、P6、P7、P8、P10、P12、P13、P14 + D1–D10。**

### 关注点

整棵任务树的 token/成本/turn/深度/并发/字节/队列硬限；bounded mailbox/event stream；拒绝、等待、降级的显式结果；logical budget 与 wall-clock deadline 分开。

### 证据清单

- P2 子 agent 树限额、P5 mailbox、P8 慢前端 gap、P10 Wasmtime fuel/epoch、P13 ready lease/配额、P14 bounded channel 是直接证据。
- D1–D10 均有成本、资源、并发、许可、带宽或安全容量约束。
- 代表出处：[Tokio mpsc](https://docs.rs/tokio/latest/tokio/sync/mpsc/)、[Airflow scheduler](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/scheduler.html)、[Wasmtime Store limits](https://docs.wasmtime.dev/api/wasmtime/struct.StoreLimits.html)。

### 边界论证

硬限若由 agent 或前端自愿遵守，在递归 spawn、重试风暴和断线时必然失效；内核必须原子记账并拒绝超限操作。

**不进入内核**：模型选择、质量/成本权衡、队列排序公式、客户优先级和降级内容。这些是 lib-user/app policy；kernel 只执行上限和返回 `BudgetExceeded/Backpressured/Rejected`。

## 10) Deadline、Timer、Lease、Heartbeat 与 fencing

**裁决：kernel**  
**证据强度：18/24；P4、P5、P6、P7、P8、P11、P12、P13 + D1–D10。**

### 关注点

持久 timer、deadline propagation、TTL lease、renew/release/revoke、fencing token、heartbeat/suspect/draining、失联回收；使用注入 clock，支持模拟与 replay。

### 证据清单

- P4 durable timer、P12 ownership/lease、P13 ready lease、P5 lifecycle/mailbox、P8 handle expiry 支持机制。
- D1–D10 全部把资源、任务、控制权或人工接管绑定到 lease/deadline/heartbeat。
- 代表出处：[Kubernetes controllers](https://kubernetes.io/docs/concepts/architecture/controller/)、[ROS 2 node lifecycle](https://design.ros2.org/articles/node_lifecycle.html)、[FIPA Contract Net](http://www.fipa.org/specs/fipa00029/SC00029H.html)。

### 边界论证

租约是否仍有效、旧持有者是否被 fencing 是并发安全事实，不能让每个 worker自行判断。timer 若不持久，重启会丢失 SLA、审批和 offer 过期。

**不进入内核**：什么资源需要锁、TTL 数值、心跳健康阈值、抢占优先级和物理安全恢复。它们由 lib-user/app 配置。kernel 不把 lease 自动解释为物理动作已停止。

## 11) 多 agent 消息、Spawn/Join、监督与失败传播

**裁决：kernel**  
**证据强度：19/24；P1、P2、P3、P4、P5、P6、P7、P12、P13 + D1–D10。**

### 关注点

typed envelope（from/to/correlation/deadline/cancel/trace）、bounded delivery、spawn/join、parent-child cancellation、failure event、restart budget、in-flight message disposition；本地 identity 与生命周期优先。

### 证据清单

- P5 actor、P6 ACL/CNP、P2 subagent、P4 child run、P13 dynamic tree 是主要先例。
- 10 个领域都依赖层级任务、黑板、协商或监管链；其共同部分是身份、消息时程和失败传播，不是具体组织图。
- 代表出处：[Erlang supervision principles](https://www.erlang.org/doc/system/sup_princ.html)、[ractor](https://docs.rs/ractor/latest/ractor/)、[Claude Agent SDK subagents](https://code.claude.com/docs/en/agent-sdk/subagents)。

### 边界论证

父任务取消是否传播、child 失败是否可见、满 mailbox 是拒绝还是等待，都会改变运行时正确性，应由 kernel 定义。内核可提供 `Spawn/Join/Send/Failure`，但不决定谁当 supervisor、何时 fan-out、如何综合答案。

**明确排除**：v1 不承诺跨节点位置透明。若没有 registry、placement、membership、serialization、故障检测，`AgentRef` 仅在本地/当前 kernel 实例有效。

## 12) Workflow/DAG/协议 FSM/组织编排

**裁决：contested**  
**证据强度：20/24；P1、P2、P4、P6、P7、P9、P10、P12、P13、P14 + D1–D10。**

### 放进 kernel 的论据

- P4/P13 证明 durable step identity、ready 判定、wait/signal、动态 spawn 与恢复必须由运行时知道。
- P6 表明数据驱动 protocol FSM 可校验 CNP 等交互合法性。
- D1、D2、D10 明确需要 DAG 依赖、fan-out/fan-in、失效传播；D6/D9 需要版本化行动计划与硬前置条件。

### 留给 lib-user 的论据

- P1 四框架差异巨大：LangGraph/CrewAI 内建图，OpenAI Agents 以 loop/handoff 为核，AG2 以会话角色为核；没有稳定统一 DSL。
- P2、P7、P10、P14 明确把 workflow、BT/DAG、graph DSL 作为上层 artifact；P12 用 CRD/reconciler 扩展而非把领域控制器写进核心。
- 静态 DAG 不自然表达 agent loop、临时 fan-out、人类 signal 和动态恢复；过早冻结 DSL 会像 xi frontend protocol 一样累积兼容债。

### 裁决理由

应拆成两层，但“Workflow”作为整体仍有实质归属冲突：

- **kernel 固定原语**：稳定 step/run identity、Spawn/Join/Wait/Complete、依赖满足的原子事实、取消和恢复；
- **lib-user 提供 artifact/解释器**：DAG、协议 FSM、角色组织、speaker selection、重试/升级路线、图迁移政策；
- v1 不应把 `Agent/Task/Crew/GroupChat` 或某个 graph DSL 固化为 Rust ABI。

代表出处：[Airflow dynamic task mapping](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/dynamic-task-mapping.html)、[LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview)、[Kubernetes Operators](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)。

## 13) 版本化 workspace、artifact 与 provenance

**裁决：kernel**  
**证据强度：20/24；P1、P2、P4、P6、P7、P9、P10、P12、P13、P14 + D1–D10。**

### 关注点

revision/CAS workspace、不可变 artifact reference、content digest/manifest、owner、producer run、input lineage、validity/invalidation、冻结 snapshot；blob 与领域 schema 外置。

### 证据清单

- P6 黑板 CAS、P12 API object/revision、P10 artifact ABI、P13 state projection、P1/P2 transcript/checkpoint 支持通用事实层。
- D1–D10 都要求把补丁、来源、工单证据、地图/观测、命令、计划或签核结果绑定到版本与 producer。
- 代表出处：[W3C PROV-DM](https://www.w3.org/TR/prov-dm/)、[Kubernetes API concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/)、[Git worktree](https://git-scm.com/docs/git-worktree)。

### 边界论证

没有内核统一的 revision、ownership 和 lineage，多个 agent 会覆盖事实，前端也无法重建“该结论基于哪一版输入”。因此内核应保存**引用与关系事实**。

**杀掉的伪内核需求**：

- 不内置 Git object、论文 ontology、GIS、PDK、向量库或业务数据库 schema；
- 不做 artifact 自动 merge、embedding/search、claim 真伪判断；
- 大 blob、索引和 retention 由 lib-user 的 artifact/store adapter 管理。

## 14) “强事实/弱观测”、新鲜度、置信度与领域有效性

**裁决：app**  
**证据强度：14/24；P4、P6、P7、P12、P13 + D2–D10；领域侧显著更强。**

### 关注点

来源质量、event time、watermark、staleness、confidence、unknown/conflict；决定某观测能否支持命令、何时结果 stale、何为“强承诺”。

### 证据清单

- 先例仅提供通用时间、revision、provenance 与 current/desired state 机制。
- D2–D10 强烈要求区分网页证据、GPS、遥测、气象、量测和签核结果的新鲜度/可信度；D1 主要以 SHA/环境精确绑定替代置信度模型。
- 代表出处：[W3C PROV-DM](https://www.w3.org/TR/prov-dm/)、[FAA UTM ConOps](https://www.faa.gov/uas/research_development/traffic_management/media/UTM_ConOps_v2.pdf)、[NERC EOP-005-3](https://www.nerc.com/pa/Stand/Reliability%20Standards/EOP-005-3.pdf)。

### 边界论证

这是最容易披着“kernel must”外衣进入核心的应用需求。内核可以保证时间戳未被悄悄改写、revision 可比较、provenance 可追踪，却无法判断：GPS 30 秒是否过期、论文转载是否独立、频率量测是否足以合闸、哪个 signoff corner 因 ECO 失效。这些规则取决于领域模型和安全案例，故整体裁决为 app。

**kernel 仅提供原料**：typed metadata slot、event/receive time、revision、provenance、CAS 与 policy hook。  
**app 负责判定**：质量码含义、staleness 阈值、冲突消解、是否阻断操作。

## 15) Agent 认知与业务决策语义

**裁决：app**  
**证据强度：24/24；P1–P14 的排除项/反例 + D1–D10 的 app_layer；双路全覆盖。**

### 关注点

prompt/persona/role/goal/backstory、planner、speaker/worker 选择、模型与 effort、RAG/memory summarization、候选评分、停止条件、成功标准、领域优化、安全/法规/商业规则。

### 证据清单

- 先例路线分裂本身是反证：Crew/GroupChat/Graph/Loop/BDI/Controller 各有不同认知模型；嵌入式库、Rust API、capability 和 actor 先例均要求策略外置。
- D1–D10 的 app_layer 全部把任务拆分、证据标准、退款政策、路径规划、派单目标、ICS/空域/电网专业规则和签核门限留在应用。
- 代表出处：[CrewAI crews](https://docs.crewai.com/en/concepts/crews)、[AG2 GroupChat](https://docs.ag2.ai/latest/docs/user-guide/basic-concepts/introducing-group-chat/)、[Jason BDI](https://jason-lang.github.io/jason/)。

### 边界论证

这些内容决定“做什么/为什么/何谓好”，而不是“运行事实怎样被一致执行”。若把它们写进 kernel ABI，任何新领域或模型范式都会迫使核心 breaking change，并把不可验证的 prompt 政策误包装成安全保证。

**严厉排除**：Rust kernel 不定义 `Agent { role, goal, backstory }`、Crew、bidder scoring、supervisor prompt、研究可信度、ETA、路径规划、潮流校验、DRC/LVS 判据。库使用者可构造相应 app artifact 或 policy 插件。

## 16) Store/Executor/Provider/Transport/Sandbox 实现

**裁决：lib-user**  
**证据强度：24/24；P1–P14，D1–D10；双路全覆盖。**

### 关注点

数据库、journal backend、LLM provider、tool/MCP、HTTP、Git/CI、queue、artifact blob store、Tokio/本地 executor、WASM/容器 sandbox、secret provider、telemetry sink、网络 transport、跨节点部署。

### 证据清单

- 14 个先例一致要求通过 adapter/trait 解耦供应商与部署；P14 明确 `types ← core ← runtime ← platform adapter`。
- 10 个领域都依赖完全不同的基础设施和行业协议，因此不可能成为统一 Rust kernel ABI。
- 代表出处：[quinn-proto sans-I/O](https://docs.rs/quinn-proto/latest/quinn_proto/)、[Rust dyn compatibility](https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility)、[WIT interfaces](https://component-model.bytecodealliance.org/design/wit.html)。

### 边界论证

内核必须定义窄、object-safe、可测试的 trait 和版本化 DTO，并规定回调并发/重入/取消契约；但实现和运营归 lib-user。把 Postgres、Tokio、OpenAI、MCP 或 Kubernetes 直接放进 core 会扩大依赖、破坏 WASM/local 路径，并把部署故障模型固化为 ABI。

**Rust 约束**：dyn 扩展需 boxed future/object-safe；只在真正 spawn 边界要求 `Send + 'static`；另保留 local/WASM 路径。

## 17) 展示、交互、视图状态与产品工作台

**裁决：frontend**  
**证据强度：24/24；P1–P14，D1–D10；双路全覆盖。**

### 关注点

聊天/IDE/CLI、token/markdown/artifact 渲染、agent tree/graph layout、timeline 虚拟化、筛选折叠、审批表单、错误文案、通知、dashboard、窗口/路由生命周期和本地 transient state。

### 证据清单

- P8/P9 是最直接的 core/frontend 分离证据；其余先例均把 UI 投影排除在核心。
- D1–D10 均要求领域工作台，但同时警告前端不能成为第二真相源。
- 代表出处：[xi retrospective](https://raphlinus.github.io/xi/2020/06/27/xi-retrospective.html)、[GPUI contexts](https://github.com/zed-industries/zed/blob/main/crates/gpui/docs/contexts.md)、[Tauri calling Rust](https://v2.tauri.app/develop/calling-rust/)。

### 边界论证

这些状态可由 snapshot/event 重建，且依赖产品、屏幕和用户角色；放进 kernel 会导致 viewport、像素、格式化和 cache consistency 反向污染运行时。前端只能提交版本化 command/signal，不得直接改 run、lease、approval 或 effect outcome。

**接口后果**：优先同进程 Rust API；只有真实跨语言、sandbox 或独立宿主需求成熟时才加 JSON/WS/FFI adapter，不能为唯一前端过早冻结宽协议。

## 18) Rust 公共 API、版本化 DTO、sans-I/O 与 FFI 纪律

**裁决：kernel**  
**证据强度：18/24；P1–P14 + D2、D3、D5、D7；先例侧极强、领域侧为 typed envelope/事件间接支持。**

### 关注点

小型 nominal handles；owned、serializable、versioned tagged DTO；machine-readable error；`Kernel::apply(now,input)`/`poll_output()` 或等价窄腰；core 不拥有 socket/thread/UI callback；generic convenience API 与稳定 dyn/FFI ABI 分层。

### 证据清单

- P14 是直接规范；P3/P10 说明 dyn/object safety、失败清理、Result；P8/P9 说明协议版本和 UI cache 债；P11 WIT resource 展示稳定资源 ABI；其余先例都依赖 typed event/command。
- D2、D3、D5、D7 明确要求 schema envelope、去重和 typed event，间接验证 DTO 边界。
- 代表出处：[Rust dyn compatibility](https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility)、[UniFFI futures](https://raw.githubusercontent.com/mozilla/uniffi-rs/main/docs/manual/src/futures.md)、[flutter_rust_bridge opaque Rust types](https://github.com/fzyzcjy/flutter_rust_bridge/blob/main/website/docs/guides/types/arbitrary/rust-opaque.md)。

### 边界论证

这是 Rust **库本身**的正确性和可演进性，不是 adapter 可事后补救的细节。若公开 trait object、borrow、Tokio channel 或 provider type 泄露进 ABI，后续无法在 Tauri/napi/UniFFI/WASM 间复用，也无法保持有限接口面。

**建议的最小窄腰**：

```text
initialize(capabilities) -> KernelInfo
start(StartRequest) -> RunHandle
command(RunId, expected_revision, KernelCommand) -> CommandAck
inspect(RunId) -> Snapshot
subscribe(RunId, Cursor) -> EventStream
resolve_effect(CallId, EffectOutcome)
shutdown(...)
```

其中 runtime facade 可包装成 async `Stream/Future`；core 仍保持 executor/transport 中立。FFI 只传 owned DTO 和 opaque handle，并明示 cancel/dispose/terminal 关系。

---

## 3. 从税系收敛出的最小职责边界

### 3.1 Kernel 必须拥有

1. run/task/effect/signal/capability 的稳定身份、状态机、合法转移与终态；
2. command/event/snapshot/cursor/revision 的一致语义；
3. journal/checkpoint/recovery 的协议及历史兼容检查；
4. effect 的 propose-authorize-record 强制点；
5. capability、硬预算、取消、deadline、lease/fencing 和父子传播；
6. typed envelope、artifact reference/provenance、可恢复人工 signal；
7. 小型版本化 Rust DTO/handle 和 sans-I/O core。

### 3.2 Lib-user 必须组装/注入

1. store、executor、clock、LLM/tool/provider、transport、sandbox、secret 与 telemetry adapter；
2. workflow/DAG/协议解释器、调度/重试政策、下游幂等与补偿；
3. 身份系统到 capability 的映射、审批者选择、保留和迁移运营政策；
4. 跨节点 registry/placement/membership/故障检测（若产品需要）。

### 3.3 Frontend 负责

1. snapshot/event 投影、聊天/timeline/tree/graph、artifact 渲染；
2. approval/input/config command 的表单与文案；
3. listener cleanup、窗口/路由生命周期、筛选折叠等 transient view state；
4. Gap/RevisionExpired 后请求 resnapshot，而不是自行修补事实。

### 3.4 App 负责

1. prompt、角色、模型、planner、任务分解、停止/成功标准；
2. 领域 ontology、质量/置信度/新鲜度规则、优化与安全约束；
3. 业务权限和审批政策、retry classifier、补偿含义；
4. 外部 effect 的真实幂等实现与专业验证。

## 4. 最容易误判的五条边界

1. **Durability 是 kernel 语义，但数据库不是。** 不定义恢复契约就没有 kernel；绑定 Postgres/SQLite 则不是最小库。
2. **Approval handshake 是 kernel，审批政策和弹窗不是。** 等待状态必须可恢复，谁来批与怎么展示是外层。
3. **Capability enforcement 是 kernel，RBAC/IAM 不是。** kernel 守住执行路径；企业身份与授权规则由宿主给出。
4. **Effect ledger 是 kernel，exactly-once 不是。** kernel 记录 claim/attempt/outcome；下游幂等和补偿才决定端到端结果。
5. **Spawn/Join 是 kernel，Crew/DAG/角色组织不是。** 内核保证树和失败传播；应用决定为何分工、向谁分工、如何综合。

## 5. 来源索引

### 5.1 先例（P）

- **P1** [Python 主流 LLM agent 框架](../precedents/llm-frameworks-py.md)：[LangGraph](https://docs.langchain.com/oss/python/langgraph/overview)、[OpenAI Agents SDK](https://openai.github.io/openai-agents-python/)。
- **P2** [Claude Code 与 Claude Agent SDK](../precedents/claude-agent-sdk.md)：[Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview)。
- **P3** [Rust LLM agent 生态](../precedents/rust-agent-eco.md)：[Rig](https://github.com/0xPlaygrounds/rig)、[Swiftide](https://github.com/bosun-ai/swiftide)。
- **P4** [持久执行引擎](../precedents/durable-execution.md)：[Temporal](https://docs.temporal.io/workflow-definition)、[Restate](https://docs.restate.dev/foundations/key-concepts.md)、[DBOS](https://docs.dbos.dev/typescript/programming-guide)。
- **P5** [Actor 系统](../precedents/actor-systems.md)：[Erlang/OTP](https://www.erlang.org/doc/system/design_principles.html)、[Akka fault tolerance](https://doc.akka.io/libraries/akka-core/current/typed/fault-tolerance.html)。
- **P6** [经典 MAS](../precedents/classic-mas.md)：[FIPA ACL](http://www.fipa.org/specs/fipa00061/SC00061G.html)、[Jason](https://jason-lang.github.io/jason/)。
- **P7** [ROS/ROS2](../precedents/ros-robotics.md)：[Actions](https://design.ros2.org/articles/actions.html)、[Lifecycle](https://design.ros2.org/articles/node_lifecycle.html)。
- **P8** [Kernel/frontend 协议分离](../precedents/kernel-frontend-split.md)：[Jupyter](https://jupyter-client.readthedocs.io/en/latest/messaging.html)、[LSP](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/)。
- **P9** [xi 与 Zed/GPUI](../precedents/xi-and-editors.md)：[xi retrospective](https://raphlinus.github.io/xi/2020/06/27/xi-retrospective.html)、[GPUI](https://github.com/zed-industries/zed/blob/main/crates/gpui/README.md)。
- **P10** [嵌入式核心库](../precedents/embedded-libs.md)：[SQLite architecture](https://www.sqlite.org/arch.html)、[Wasmtime Store](https://docs.wasmtime.dev/api/wasmtime/struct.Store.html)。
- **P11** [微内核与 capability](../precedents/os-capability.md)：[seL4](https://github.com/seL4/seL4/blob/master/manual/parts/intro.tex)、[WASI](https://github.com/WebAssembly/WASI/blob/main/docs/Capabilities.md)。
- **P12** [Kubernetes 控制平面](../precedents/control-plane.md)：[API concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/)、[Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)。
- **P13** [DAG 编排引擎](../precedents/workflow-engines.md)：[Airflow tasks](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html)、[Prefect flows](https://docs.prefect.io/v3/concepts/flows)。
- **P14** [Rust API patterns](../precedents/rust-api-patterns.md)：[quinn-proto](https://docs.rs/quinn-proto/latest/quinn_proto/)、[UniFFI](https://raw.githubusercontent.com/mozilla/uniffi-rs/main/docs/manual/src/types/interfaces.md)。

### 5.2 领域（D）

- **D1** [多 agent 软件工程](../domains/multi-agent-coding.md)：[Git merge](https://git-scm.com/docs/git-merge)、[GitHub protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)。
- **D2** [网络深度调研蜂群](../domains/deep-research.md)：[W3C PROV-DM](https://www.w3.org/TR/prov-dm/)、[RFC 9309](https://www.rfc-editor.org/rfc/rfc9309)。
- **D3** [客服工单分诊](../domains/support-triage.md)：[Zendesk SLA](https://support.zendesk.com/hc/en-us/articles/4408836335386-Defining-and-using-SLA-policies)、[NIST SP 800-207](https://csrc.nist.gov/pubs/sp/800/207/final)。
- **D4** [仓储机器人集群](../domains/warehouse-robots.md)：[Open-RMF](https://openrmf.readthedocs.io/en/latest/)、[rmf_traffic](https://github.com/open-rmf/rmf_traffic)。
- **D5** [网约车/即时配送](../domains/ride-dispatch.md)：[DoorDash logistics engine](https://careersatdoordash.com/blog/how-doordash-built-a-high-performance-logistics-engine/)、[Google SRE overload](https://sre.google/sre-book/handling-overload/)。
- **D6** [地震后城市救援](../domains/quake-rescue.md)：[FEMA NIMS](https://www.fema.gov/emergency-managers/nims)、[Incident Action Planning Guide](https://www.fema.gov/sites/default/files/2020-07/fema_incident-action-planning-guide_2015.pdf)。
- **D7** [城市交通数字孪生](../domains/traffic-twin.md)：[ARC-IT](https://www.arc-it.net/)、[FHWA TMC](https://ops.fhwa.dot.gov/trafficanalysistools/tmc.htm)。
- **D8** [全国空域管理/空中出租车](../domains/airspace-utm.md)：[FAA UTM](https://www.faa.gov/uas/research_development/traffic_management)、[EU U-space regulation](https://eur-lex.europa.eu/eli/reg_impl/2021/664/oj)。
- **D9** [电网黑启动与自愈](../domains/grid-blackstart.md)：[NERC blackstart resources](https://www.nerc.com/pa/Stand/Pages/Blackstart-Resources.aspx)、[IEC 61850](https://www.iec.ch/basecamp/iec-61850)。
- **D10** [先进芯片 tape-out](../domains/chip-tapeout.md)：[OpenROAD Flow Scripts](https://openroad-flow-scripts.readthedocs.io/en/latest/)、[OpenLane 2](https://openlane2.readthedocs.io/en/latest/)。

## 6. 结论

跨 24 个来源最稳定的“窄腰”不是 Agent/Crew/Graph，而是：**稳定身份 + 小型生命周期 + 版本化命令/事件/快照 + 可恢复 effect/signal + capability/预算/取消/lease 强制点 + adapter 中立的 Rust DTO**。

最应该被从 kernel 杀掉的是：角色/persona、领域 workflow DSL、模型/供应商、业务成功标准、领域新鲜度与置信度规则、UI 投影、数据库/队列实现，以及任何通用 exactly-once 宣称。Workflow 和副作用安全保留为 contested，是因为它们确实需要内核机制，但完整语义只能由 lib-user/app 补齐。
