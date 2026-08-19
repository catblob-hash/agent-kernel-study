# Agent kernel 反模式清单：千万不要做什么

## 使用范围与证据纪律

本文面向一个将被抽成 Rust 依赖库、只向宿主和前端暴露有限接口的多-agent runtime core。它不是“好设计原则”合集，而是从先例中反推的**禁令清单**。

纳入一条反模式，必须至少有一种可核验代价：作者事后否定、官方重设计、官方明确警告/限制、已知兼容债、或带复现/讨论的社区故障报告。只有架构直觉而没有先例代价的判断不纳入。

证据强度：

- **A：作者 postmortem / 官方重设计反思**——可直接说明某个边界付出了代价。
- **B：官方规范、设计文档或 API 合约**——可证明限制、失败语义或为避免事故而存在的约束，但未必证明项目整体失败。
- **C：社区 issue 报告**——只证明报告者在特定组合/版本观察到故障；不能外推为所有版本的普遍事实。本文会明确标注“未获 maintainer 证实”。

“不要”不等于先例系统做错了。很多机制对其完整产品是合理的；反模式是**把该机制不加选择地固化进一个最小 agent kernel**。

---

## 一、不要让 kernel 变成 UI 的另一半

### AP-01　kernel 不要保存或生成前端展示 cache 的增量变换

**禁令**：不要让 kernel 维护卡片、行、折叠、viewport、timeline patch、token 动画等展示 cache，也不要要求前端必须按一串增量 patch 才能恢复正确状态。

**真实代价证据**：xi core 维护完整行视图，frontend 维护稀疏 cache，`update` 是 `copy/skip/invalidate/update/ins` 组成的 cache 变换程序。滚动时请求 cache 外文本、编辑与 rewrap 并发、以及 tearing 协调让作者称“把滚动做对花了数月”；其反事实判断是同进程数据结构会直接得多。作者最终明确说 core/frontend process separation “was not a good idea”，并把 async 称为 complexity multiplier。

- 来源性质：**A，作者 postmortem**：[Raph Levien, “Xi-editor retrospective”](https://raphlinus.github.io/xi/2020/06/27/xi-retrospective.html)
- 协议细节来源：**B，官方协议文档**：[xi Frontend Protocol](https://xi-editor.io/docs/frontend-protocol.html)

**kernel 边界**：kernel 只提供权威 snapshot、单调 revision/cursor 和领域事件；前端 projection 必须可丢弃、可从 snapshot 重建。高频展示 patch 不是 durable kernel event。

### AP-02　kernel 不要让业务正确性等待反向 UI RPC

**禁令**：不要让调度、状态转移或结果正确性依赖前端提供像素、字体、viewport、布局、本地格式化或同步回调。

**真实代价证据**：xi core 负责换行，却必须反向调用 frontend 的 `measure_width` 获取字体宽度。边界没有消除耦合，只把耦合变成请求时序、cache 一致性和与编辑操作的 race；作者记录了 rewrap race、tearing 和数月协调成本。

- 来源性质：**A+B，作者 postmortem + 官方协议**：[retrospective](https://raphlinus.github.io/xi/2020/06/27/xi-retrospective.html)、[Frontend Protocol](https://xi-editor.io/docs/frontend-protocol.html)

**kernel 边界**：UI 事实只能是可丢弃 hint。kernel 在前端断开、卡死或无窗口时仍须正确推进；真正需要人工决定时，进入显式 `Paused/InputRequested`，而不是悬在任意 UI callback 上。

### AP-03　kernel 不要为了假想多前端过早强制进程分离和 JSON-RPC

**禁令**：在只有一个同版本、同产品 frontend 时，不要先冻结 WebSocket/JSON-RPC/多进程协议，再让 Rust core 的内存模型迁就 wire DTO。

**真实代价证据**：xi 为跨语言 frontend 自建 `xi-rpc` 和 JSON 协议；作者报告 Swift JSON 解析“shockingly slow”、Rust `serde` 增加二进制体积、自建 RPC 增加维护成本，并最终明确否定该 process split。相对地，GPUI 的官方模型用同进程 `App`、`Entity<T>`、`Context` 和可失败的 async access 支持观察与异步，并不要求把所有应用状态序列化。

- 来源性质：**A，作者 postmortem**：[xi retrospective](https://raphlinus.github.io/xi/2020/06/27/xi-retrospective.html)
- 对照来源：**B，官方代码库文档**：[GPUI README](https://github.com/zed-industries/zed/blob/main/crates/gpui/README.md)、[GPUI Contexts](https://github.com/zed-industries/zed/blob/main/crates/gpui/docs/contexts.md)

**kernel 边界**：首版优先稳定进程内 Rust 的 command/snapshot/event 语义；IPC 是其上 adapter。只有独立发布、跨语言、远程、sandbox、权限隔离或真实多 consumer 出现后，才支付认证、版本、背压和重连的协议税。

### AP-04　kernel 不要把仍在发现期的产品交互冻结为稳定 wire contract

**禁令**：不要把 click/drag、角色卡、prompt 文案、token frame、某一工具 UI 的字段或产品内部 object shape 直接升格为 kernel ABI。

**真实代价证据**：xi 官方协议中 click/drag 正迁往 gesture，旧 selection/find style 与新 annotations 并存，plugin command 被标为 “in flux”，`replace.preserve_case` 存在但被忽略，`update` op 尚未使用。早期冻结使同一产品内部也需要 deprecated 字段、迁移期双表示和兼容代码。

- 来源性质：**B，官方协议/RFC**：[xi Frontend Protocol](https://xi-editor.io/docs/frontend-protocol.html)、[Annotations RFC](https://github.com/xi-editor/xi-editor/blob/master/rfcs/2018-11-23-annotations.md)

**kernel 边界**：稳定层只保留运行身份、状态转换、命令确认、事件顺序、effect/input 生命周期；产品交互 DTO 在 frontend adapter 演进。实验字段必须明确 capability-gated/experimental，而非伪装成稳定核心。

### AP-05　kernel 不要让第一个扩展用例定义整个插件协议

**禁令**：不要按当前最复杂的内置 agent、RAG、高亮器或 provider 的私有 cache/state machine 设计“通用” extension API。

**真实代价证据**：xi 的 plugin API 为大文件 syntax highlighting 的异步 cache 难题特化，叠加 OT/cache protocol；作者后来称该接口 “overly specialized”，希望形成标准协议但没有实现。贡献者功能也经常被大架构重构阻塞，维护者成为 gatekeeper；vi keybinding 项目未进入主项目。

- 来源性质：**A，作者 postmortem**：[xi retrospective](https://raphlinus.github.io/xi/2020/06/27/xi-retrospective.html)

**kernel 边界**：extension 边界只暴露窄 capability、typed request/response、取消和事件；provider/agent 私有状态留在 adapter。至少两个真实扩展证明共同语义后再扩大稳定 ABI。

### AP-06　kernel 不要把 agent 协作默认建模为 CRDT/自动 merge

**禁令**：不要假设 agent 的 plan、审批、budget、工具副作用和因果顺序都能靠“并发修改可合并”解决。

**真实代价证据**：xi 从 Android IME 跨进程并发问题泛化出 CRDT/OT；作者事后指出 indentation 这类依赖历史的决定“不自然适配 CRDT”，syntax highlighting 仍需复杂 OT/cache protocol。也就是说，通用合并抽象没有消除领域历史/意图语义，反而叠加了另一套协调协议。

- 来源性质：**A，作者 postmortem**：[xi retrospective](https://raphlinus.github.io/xi/2020/06/27/xi-retrospective.html)

**kernel 边界**：审批、授权、effect、预算、任务终态优先用 append-only facts、CAS/revision 和领域状态机；CRDT 只能是特定 artifact 类型的可选 adapter。

---

## 二、不要把编排策略和“agent 智能”固化成 kernel 本体

### AP-07　kernel 不要原生固定 Role/Crew/GroupChat/BDI 的选择策略

**禁令**：不要把 `Agent(role, goal, backstory)`、Crew manager、GroupChat speaker selection、“第一个匹配 plan”或 supervisor prompt 变成 Rust kernel 的必选对象和不可替换语义。

**真实代价证据**：Jason 官方教程明确说明递归 subgoal 会产生不终止 intention，无 applicable plan 会失败，默认只选第一个 context 成立的 plan；这是具体策略，不是通用最优性。AG2 issue #1756 报告 `GroupChat + human_input_mode="ALWAYS"` 的策略组合进入无限循环且不弹出输入。CrewAI 官方 API 同时把 role/backstory、sequential/hierarchical process、planning、memory、cache、embedder、callbacks 放进 Crew 配置，说明该对象已是上层产品/组织 DSL，而非窄执行原语。

- 来源性质：**B，官方教程/文档**：[Jason Hello BDI](https://jason-lang.github.io/jason/tutorials/hello-bdi/readme.html)、[CrewAI Crews](https://docs.crewai.com/en/concepts/crews)
- 故障来源：**C，社区 issue，未据此外推所有版本**：[AG2 #1756](https://github.com/ag2ai/ag2/issues/1756)

**kernel 边界**：kernel 只强制 run/task identity、spawn/join、预算、取消、失败传播和可插拔 scheduler/policy。角色、路由、speaker、proposal 评分、合成和 replan 属库使用者。

### AP-08　kernel 不要用完整 ACL/ontology/performative 词表冒充可执行契约

**禁令**：不要因为消息写着 `inform`、`request`、`Action` 或某个 ontology 名，就认定事实已发生或接收者能互操作。

**真实代价证据**：FIPA ACL 的 performative 以 feasibility precondition/rational effect 描述心智状态，本身不保证网络送达或真实执行；Jason 实际实现的是更小、直接映射 belief/goal/event 的 KQML-style 子集。现代重复案例是 CrewAI issue #3154：报告者观察到模型文本生成了完整 Thought/Action/Observation，但真实 `tool.run()` 未执行、trace 也没有 tool event。

- 来源性质：**B，规范/官方实现文档**：[FIPA ACL Message Structure](http://www.fipa.org/specs/fipa00061/SC00061G.html)（旧站可能限制抓取）、[Jason performatives](https://jason-lang.github.io/jason/tech/performatives.html)
- 故障来源：**C，社区 issue，未据此外推所有 provider**：[CrewAI #3154](https://github.com/crewAIInc/crewAI/issues/3154)

**kernel 边界**：消息种类保持窄；关键 command 需要 typed schema、调用 ID、executor 产生的 started/outcome 事实和明确 terminal。自然语言与模型自述永远不是执行审计证据。

### AP-09　kernel 不要提供无 owner、无 revision 的全局可变黑板

**禁令**：不要把共享 `HashMap<String, JSON>`、全局参数服务或任意可写 memory 叫作“协作内核”。

**真实代价证据**：ROS 1 的全局可写参数黑板失去 ownership，任意 node 可改 driver/control 参数；ROS 2 参数设计改为 node-owned、可验证并支持原子更新。BehaviorTree.CPP 的共享 blackboard pointer 普通访问后写入不是线程安全，必须使用显式锁接口。经典 blackboard 文献本身也把 control shell、效率和 heuristic control 当作核心难题，而不是免费共享字典。

- 来源性质：**A/B，官方重设计文档 + 官方 API**：[ROS 2 parameter design](https://design.ros2.org/articles/ros_parameters.html)、[BehaviorTree.CPP `tree_node.h`](https://raw.githubusercontent.com/BehaviorTree/BehaviorTree.CPP/master/include/behaviortree_cpp/tree_node.h)
- 原始架构来源：**B，经典论文**：[Hayes-Roth, “A Blackboard Architecture for Control”](https://doi.org/10.1016/0004-3702(85)90044-4)

**kernel 边界**：共享 workspace 必须 owner-scoped、schema-validated、带 revision/CAS、权限和背压。领域 merge、搜索、embedding、可见性与保留策略留在应用。

### AP-10　kernel 不要用一个 `send`/RPC 原语承载所有交互时程

**禁令**：不要让事件广播、短请求、长任务、人工等待、进度和取消全都退化成同一种“发消息”或一个永不返回的 Future。

**真实代价证据**：ROS 1 同步 service 无法表达长期任务的 progress/cancel，actionlib 又因同步 service 限制而只能全部建在 topics 上；ROS 2 专门把 Action 重设计为 goal/result/cancel 三类 service 加 feedback/status 两类 topic，并标准化 goal ID、进度、取消和终态。

- 来源性质：**A/B，官方设计反思与重设计**：[ROS 2 Actions design](https://design.ros2.org/articles/actions.html)

**kernel 边界**：至少区分 publish/subscribe 事件、带 deadline 的 request/response、以及有 `RunId + progress + cancel + terminal + retained result` 的长任务。Agent identity 不是第四种通信原语。

### AP-11　kernel 不要首版承诺“通用可变 agent graph”

**禁令**：不要把任意运行时增删节点、回边、嵌套 fan-out、可视化和恢复兼容一次性做成 kernel DSL；也不要反过来把所有 agent loop 强塞进静态 DAG。

**真实代价证据**：Airflow 的 runtime mapping 仍要求有限 iterable、数据库可索引的实例化和默认 `max_map_length=1024`；官方刻意不支持 mapped task group 内的 nested mapping，理由就是 UI complexity。Dagster dynamic graph 同样限制为声明好的 `map/collect` clone/fan-in。Prefect 放宽为普通 Python 动态 flow 后，durable input 去重仍需另存 seen keys。动态性没有消除身份、配额、展示和恢复成本。

- 来源性质：**B，官方文档/Apache 设计提案**：[Airflow Dynamic Task Mapping](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/dynamic-task-mapping.html)、[AIP-42](https://cwiki.apache.org/confluence/display/AIRFLOW/AIP-42+Dynamic+Task+Mapping)、[Dagster Dynamic Graphs](https://docs.dagster.io/guides/build/ops/dynamic-graphs)、[Prefect Interactive Workflows](https://docs.prefect.io/v3/advanced/interactive.md)

**kernel 边界**：v1 只承诺有稳定身份和配额的 `Spawn/Join/Wait/Signal` 及 parent-child lineage；图 DSL、循环策略和画布是上层 artifact/frontend。

---

## 三、不要伪造 durability、exactly-once 和生命周期保证

### AP-12　kernel 不要把 checkpoint/retry 宣称为外部副作用 exactly-once

**禁令**：不要用“状态已 checkpoint”“成功 step 不重跑”推导“邮件、支付、交易、文件写入只发生一次”。

**真实代价证据**：LangGraph 官方文档说明 resume 会从含 `interrupt()` 的 node 开头重跑，interrupt 前副作用必须幂等或后移。CrewAI issue #5802 报告 task retry 重执行已成功工具，可能重复 payment/email/trade，并指出内存 dedup 不能跨 worker redispatch。DBOS 官方把“step 至少一次尝试、完成后不重跑”与 transaction exactly-once commit 分开；本地 journal 与第三方确认之间仍有失败窗口。

- 来源性质：**B，官方文档**：[LangGraph Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)、[DBOS Workflow Tutorial](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial)
- 故障来源：**C，社区 open issue，未获 maintainer 证实**：[CrewAI #5802](https://github.com/crewAIInc/crewAI/issues/5802)

**kernel 边界**：诚实承诺至少一次边界；每个 effect 有稳定 operation/idempotency key、执行前 durable claim、attempt、receipt/outcome。真正 exactly-once 只能由下游幂等、事务或补偿实现。

### AP-13　kernel 不要允许外部 I/O 隐藏在普通 replay/async 代码里

**禁令**：不要让 reducer/workflow/node 任意访问 clock、random、DB、HTTP、LLM 或 filesystem，却仍声称可确定恢复。

**真实代价证据**：Inngest 每个 step 通过独立 HTTP 请求重新进入函数；官方明确说 step 外 DB/API 调用会在每次 reinvocation 重复。Temporal 修改 command 顺序会报 non-deterministic error，外部 I/O 必须进入 Activity。DBOS 将 DB、第三方 API、random、local time 留在 workflow body 会破坏确定性。Restate 甚至禁止在 `ctx.run` closure 内再调用 context，以保持 journal 边界可解释。

- 来源性质：**B，官方执行模型**：[Inngest execution model](https://www.inngest.com/docs-markdown/learn/how-functions-are-executed)、[Temporal Workflow Definition](https://docs.temporal.io/workflow-definition)、[DBOS Workflow Tutorial](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial)、[Restate durable steps](https://docs.restate.dev/develop/ts/durable-steps.md)

**kernel 边界**：采用 sans-I/O reducer/drive 与显式 `EffectRequest → EffectResult` 回灌。kernel 不尝试“自动判断 closure 是否纯”。

### AP-14　kernel 不要用随机/位置漂移的 step identity，也不要忽略长运行版本迁移

**禁令**：不要让 effect/step ID 每次随机生成，不要把重排/改名当普通重构，也不要在旧 history 上悄悄执行新代码。

**真实代价证据**：Temporal 新代码发出的 command 与旧 history 不匹配会触发 non-deterministic error。DBOS 改变 step 集合/顺序会产生 checkpoint mismatch；patch 必须经历等待旧 run、deprecate、再等待 marker run、最后删除，过早移除报 `DBOSUnexpectedStepError`。Inngest 改 step ID 会强制重新执行，官方推荐描述性、稳定、唯一的 ID；大改用新 function 并 drain 旧运行。

- 来源性质：**B，官方升级/版本文档**：[Temporal Workflow Definition](https://docs.temporal.io/workflow-definition)、[DBOS Upgrading Workflows](https://docs.dbos.dev/typescript/tutorials/upgrading-workflows)、[Inngest Versioning](https://www.inngest.com/docs-markdown/learn/versioning)

**kernel 边界**：历史记录 revision 与稳定 logical effect key；不兼容时显式 `NeedsMigration`。首版优先 revision pinning + drain，不假装通用 patch DSL 免费。

### AP-15　kernel 不要把恢复失败静默降级成新 run

**禁令**：`resume/restore` 找不到、损坏、版本冲突时，不要自动从空状态启动并返回“成功”。

**真实代价证据**：CrewAI Flow 官方文档中的 unknown restore state ID 会静默 fall back；因此调用方无法仅凭成功返回区分“恢复了原 run”和“新建了 run”。该行为把数据缺失/ID 错误变成语义完全不同但表面成功的执行。

- 来源性质：**B，官方文档记录的恢复语义**：[CrewAI Flows](https://docs.crewai.com/en/concepts/flows)

**kernel 边界**：存储/恢复必须返回 machine-readable `NotFound/Corrupt/VersionConflict/NeedsMigration`；fresh 与 resume 使用不同命令或显式 mode。

### AP-16　kernel 不要依赖 cleanup/deactivation 回调保存关键状态

**禁令**：不要把 checkpoint、effect receipt、lease 释放或审计终态只写在 destructor、`on_stop`、deactivation 或 graceful shutdown hook。

**真实代价证据**：Orleans 官方说明 silo crash 等异常情况下 grain deactivation lifecycle 不保证执行。OTP supervisor 可使用 `brutal_kill`，不给 child 清理机会。两者都说明 cleanup 只覆盖优雅路径，不能成为 durability 线性化点。

- 来源性质：**B，官方生命周期/监督文档**：[Orleans Grain Lifecycle](https://learn.microsoft.com/en-us/dotnet/orleans/grains/grain-lifecycle)、[Erlang Supervisor Behaviour](https://www.erlang.org/doc/system/sup_princ.html)

**kernel 边界**：关键事实执行前/后在 durable store 显式提交；cleanup 仅做 best-effort 资源回收，恢复逻辑必须容忍它没运行。

### AP-17　kernel 不要把 restart/supervision 等同于可靠投递或安全重试

**禁令**：不要因为 actor/agent 会自动重启，就承诺消息不丢、in-flight operation 重放安全或业务状态恢复完整。

**真实代价证据**：Orleans 官方只保证 at-most-once delivery，失败重试由 caller 负责；silo 失败后 activation 可在别处重建，但这不提升投递语义。actix 的 `do_send` 在 mailbox 满时仍排队、receiver closed 时静默丢弃；`try_send` 和 `send` 又有不同失败/取消语义。监督机制与 mailbox/effect 可靠性是不同层。

- 来源性质：**B，官方文档/API**：[Orleans Best Practices](https://learn.microsoft.com/en-us/dotnet/orleans/resources/best-practices)、[actix `Addr`](https://docs.rs/actix/latest/actix/struct.Addr.html)

**kernel 边界**：restart policy、message delivery、effect idempotency 和 state reconstruction 分别建模；每个 send 返回明确的 accepted/rejected/closed/timeout 结果。

---

## 四、不要把“流”和“并发”当成免费基础设施

### AP-18　kernel 不要用无界 fan-out、mailbox、collect 或 UI event queue

**禁令**：不要让模型输出直接决定无限子 agent/任务数，不要提供绕过 mailbox capacity 的默认发送，不要为慢前端永久缓存 token/trace。

**真实代价证据**：Airflow 对 runtime mapping 默认限制 1024，AIP-42 要求 finite iterable、单项可装入 worker 内存并设置最大展开量；reduce 使用 lazy proxy 防止一次加载全部输出。actix 默认 mailbox 容量 16，但 `do_send` 可绕过“满”并继续排队。Tokio 官方说明 unbounded mpsc 在 producer 更快时内存无界增长，receiver drop 会丢弃未读消息。

- 来源性质：**B，官方文档/API/设计提案**：[Airflow Dynamic Task Mapping](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/dynamic-task-mapping.html)、[AIP-42](https://cwiki.apache.org/confluence/display/AIRFLOW/AIP-42+Dynamic+Task+Mapping)、[actix `Context`](https://docs.rs/actix/latest/actix/struct.Context.html)、[actix `Addr`](https://docs.rs/actix/latest/actix/struct.Addr.html)、[Tokio mpsc](https://docs.rs/tokio/latest/tokio/sync/mpsc/)

**kernel 边界**：强制 depth/width/concurrency/payload/collect budget；发送区分 `try_send` 与带 deadline 的等待；token 可合并/丢弃，但权威状态必须另存。

### AP-19　kernel 不要把 live event stream 当作永久可靠的状态真相

**禁令**：不要假设 watch 永不丢事件、cursor 永不过期、subscriber cache 可反向写回，也不要用消息 channel 的“已发送”代替状态持久化。

**真实代价证据**：Kubernetes API watch 历史有限，旧 revision 返回 `410 Gone/Expired`，标准恢复动作是 relist + rewatch；bookmark 没有频率保证。client-go 明确警告 informer Store 只能 Get/List，直接 Add/Modify/Delete 会破坏 event notification；`DeletedFinalStateUnknown` 甚至可能携带陈旧对象。Tokio receiver drop 会丢未读消息。

- 来源性质：**B，官方 API 概念/API 文档**：[Kubernetes API Concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/)、[client-go `tools/cache`](https://pkg.go.dev/k8s.io/client-go/tools/cache)、[Tokio mpsc](https://docs.rs/tokio/latest/tokio/sync/mpsc/)

**kernel 边界**：snapshot 与 `watch(from_revision)` 必须成对；事件有 seq/cursor、retention 和 `RevisionExpired/GapDetected`，固定恢复动作是重新 snapshot。前端 cache 永远不是 authority。

### AP-20　kernel 不要把 provider raw token/chunk 当稳定前端协议或完成信号

**禁令**：不要让前端绑定某 provider 的 delta JSON、LangGraph tuple shape 或“最后一个 token”；不要由流暂时安静推断 run terminal。

**真实代价证据**：LangGraph issue #5249 报告带 tools 后 token stream 从逐 token 退化为整段；官方文档还要求 subgraph token 显式启用，v1 chunk shape 会随 mode/组合变化。OpenAI Agents SDK 官方说明最后 token 后仍可能进行 session persistence、approval bookkeeping、compaction，必须 drain stream 并检查 `is_complete`。Jupyter 规范承认 `idle` 后异步 output 行为未定义。

- 来源性质：**B，官方文档/规范**：[LangGraph Streaming](https://docs.langchain.com/oss/python/langgraph/streaming)、[OpenAI Agents SDK Streaming](https://openai.github.io/openai-agents-python/streaming/)、[Jupyter Messaging](https://jupyter-client.readthedocs.io/en/latest/messaging.html)
- 故障来源：**C，社区 issue**：[LangGraph #5249](https://github.com/langchain-ai/langgraph/issues/5249)

**kernel 边界**：规范化区分 `TokenDelta`（best-effort UX）、`MessageCommitted`、`Step/Operation`、`Checkpoint` 和权威 `RunFinished`；provider raw event 仅在 experimental adapter namespace。

### AP-21　kernel 不要把 HITL 做成 console input、异常字符串或裸 bool callback

**禁令**：不要让 worker 阻塞等 stdin，不要靠解析异常/自然语言发现审批，不要用无 ID 的 `approve: bool` 恢复任意暂停点。

**真实代价证据**：Jupyter 规范要求不支持输入的 frontend 预先设置 `allow_stdin=false`，否则 kernel 可能阻塞。AG2 issue #1756 报告 GroupChat 与 `human_input_mode="ALWAYS"` 组合无限循环而未 prompt。LangGraph 的 `interrupt()` 是特殊控制流异常，官方警告 broad `try/except` 会吞掉它，同 node 的 interrupt 还必须保持确定顺序。

- 来源性质：**B，官方规范/文档**：[Jupyter Messaging](https://jupyter-client.readthedocs.io/en/latest/messaging.html)、[LangGraph Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)
- 故障来源：**C，社区 issue**：[AG2 #1756](https://github.com/ag2ai/ag2/issues/1756)

**kernel 边界**：一等 `Paused { requests }`，每项有 request ID、schema、deadline、actor/audit context 和绑定 revision；resume/approve/reject 幂等并检查 stale request。

### AP-22　kernel 不要把取消做成 UI 本地 flag、普通低优先级消息或悬空 Future

**禁令**：不要只关闭前端 spinner，不要让取消排在长任务后面，也不要在发出 cancel 后永远不闭合原始请求。

**真实代价证据**：LSP 规范规定即使收到 `$/cancelRequest`，原 request 仍必须返回 response；否则 caller pending/future 无法闭合。Jupyter 将 Control 放独立 socket/线程，目的就是避免 shutdown/restart/debug 被长 Shell 请求饿死。UniFFI 官方没有 builtin async cancellation，若库只导出 `async fn run()`，外语 frontend 的取消不会自动中止 Rust 工作。

- 来源性质：**B，官方规范/文档**：[LSP 3.17](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/)、[Jupyter Messaging](https://jupyter-client.readthedocs.io/en/latest/messaging.html)、[UniFFI futures/cancellation](https://raw.githubusercontent.com/mozilla/uniffi-rs/main/docs/manual/src/futures.md)

**kernel 边界**：取消走高优先级 control path；区分 requested/accepted/actually-stopped/timed-out；原 `StartRun` 最终必须由 `Completed|Failed|Cancelled` 闭合。

---

## 五、不要把安全与生命周期押在 prompt、名称或隐式对象存活上

### AP-23　kernel 不要把自然语言、tool 名、agent label 或前端事件当授权凭据/执行事实

**禁令**：不要因为模型输出 `Action: X`、消息说 `inform`、tool 名在 allowlist 字符串中或前端显示“已批准”，就让 effect 越过不可绕过的授权点。

**真实代价证据**：CrewAI issue #3154 中模型可生成完整 Action/Observation 文本而真实工具未执行，证明文本不是 execution fact。FIPA performative 也不保证执行。WASI 以“无 ambient authority”、preopen/handle 和只能减权的 rights 说明真正授权是资源引用/能力，不是名字和请求内容。

- 来源性质：**B，官方安全设计/规范**：[WASI Capabilities](https://github.com/WebAssembly/WASI/blob/main/docs/Capabilities.md)、[WASI P1 spec](https://github.com/WebAssembly/WASI/blob/v0.2.0/legacy/preview1/docs.md)、[FIPA ACL](http://www.fipa.org/specs/fipa00061/SC00061G.html)
- 故障来源：**C，社区 issue**：[CrewAI #3154](https://github.com/crewAIInc/crewAI/issues/3154)

**kernel 边界**：调用必须原子地解析 opaque capability、校验 operation + typed input，再由 executor 产生审计事件；UI/prompt 只能请求授权，不能授予 master authority。

### AP-24　kernel 不要把“预批准列表”误作强制能力白名单，也不要只在晚期 callback 做安全检查

**禁令**：不要把 allow/preapprove 和 visibility/deny 混成一个开关；不要让早期 auto-approval 绕过必须执行的 policy hook。

**真实代价证据**：Claude Agent SDK 官方文档明确 `allowedTools` 是预批准而非可见能力白名单；`bypassPermissions + allowedTools` 仍可放行未列工具。`canUseTool` 可能被更早 auto-approval shadow，必须审计/阻止的检查应放在不可绕过的 `PreToolUse`。subagent 还可能继承高权限 mode。

- 来源性质：**B，官方权限文档**：[Claude Agent SDK permissions](https://code.claude.com/docs/en/agent-sdk/permissions)、[hooks](https://code.claude.com/docs/en/agent-sdk/hooks)、[subagents](https://code.claude.com/docs/en/agent-sdk/subagents)

**kernel 边界**：固定并测试 `deny → ask/allow → execute` 等求值顺序；不可绕过的 pre-action mediation、子树预算/权限上限由 kernel 强制，UI 只实现询问体验。

### AP-25　kernel 不要把 root/master capability 复制给 child，也不要把 revoke 承诺成时间机器

**禁令**：不要让 child 自助扩权，不要用一段“请勿滥用” metadata 代替缩权 facet，也不要宣称撤销能收回已泄露数据或已发生副作用。

**真实代价证据**：WASI `fd_fdstat_set_rights` 只能减权，扩权必须回到上级。seL4 手册明确 revoke 可 partial complete，撤销 page table 不等于撤销 frame cap，授权图、对象生命周期和清理并非同一件事。OCapN Model 的引用传递天然委托，但草案没有内建 attenuation/revocation。

- 来源性质：**B，官方手册/规范；OCapN 为社区草案**：[seL4 CSpace manual](https://github.com/seL4/seL4/blob/master/manual/parts/cspace.tex)、[WASI P1](https://github.com/WebAssembly/WASI/blob/v0.2.0/legacy/preview1/docs.md)、[OCapN Model](https://github.com/ocapn/ocapn/blob/main/draft-specifications/Model.md)

**kernel 边界**：parent 保留 root，child 只拿 operation/resource/budget 子集 facet；可实现基线是 revoke 后拒绝未来经 kernel 的调用，并分别说明未开始、进行中、已完成和已泄露数据的语义。

### AP-26　kernel 不要允许 hook 递归 drive 同一 run，也不要构造控制 await 环

**禁令**：policy/event/storage hook 不得在同一线性化点递归 mutate/drive 当前 run；child 不得同步等待一个又在等待 child 的 supervisor/controller。

**真实代价证据**：SQLite 官方禁止 progress/authorizer callback 修改同一 connection，说明嵌入式 core 必须限制 callback reentrancy。OTP 官方警告 child 从 supervision tree 内调用 `terminate_child/2` 停自己的 supervisor 会死锁。Orleans non-reentrant grain 的 call cycle 也可能死锁，timeout 只能打断，不能使协议正确。

- 来源性质：**B，官方 API/监督文档**：[SQLite progress handler](https://sqlite.org/c3ref/progress_handler.html)、[SQLite authorizer](https://sqlite.org/c3ref/set_authorizer.html)、[Erlang Supervisor Behaviour](https://www.erlang.org/doc/system/sup_princ.html)、[Orleans Best Practices](https://learn.microsoft.com/en-us/dotnet/orleans/resources/best-practices)

**kernel 边界**：单 run 可变状态单 owner/串行提交；hook 返回 decision/effect intent，不能同步重入。控制协议用异步 signal/receipt，显式检测 parent-child await cycle。

### AP-27　kernel 不要让 background child 的控制/权限通道绑定 parent turn 的文本生命周期

**禁令**：不要在父 agent 输出一条“最终文本”后关闭仍运行 child 的 approval、hook、cancel 或 event channel。

**真实代价证据**：Claude Agent SDK TypeScript issue #361 报告 background subagent 在 parent turn 结束后共享 stdin/control stream 被关闭，后续 MCP permission request 得到 `Stream closed`，tool hooks 也不再发；报告者只能阻止 parent Stop 作为 workaround。

- 来源性质：**C，2026-07 仍 open、未获 maintainer 证实；只作为生命周期风险证据**：[claude-agent-sdk-typescript #361](https://github.com/anthropics/claude-agent-sdk-typescript/issues/361)

**kernel 边界**：控制/权限/event channel 绑定 `RunId/TaskId`，直到 task terminal 或明确 detach；parent turn、前端订阅和 child execution 是三个独立生命周期。

### AP-28　kernel 不要只对一种 text 字段做输出配额

**禁令**：不要只截断 `content: text`，却让 structured payload、artifact、tool result、MCP schema 或嵌套表示不受最终 context budget 约束。

**真实代价证据**：Claude Agent SDK issue #389 给出复现：MCP `structuredContent` 据报告可绕过 oversized-result 截断，compaction 后 1–3 turn 又填满 context，反复 compact，最终以 `Autocompact is thrashing` 中止；临时 hook 截断可缓解。

- 来源性质：**C，2026-07 仍 open、未获 maintainer 证实；不外推为稳定产品事实**：[claude-agent-sdk-typescript #389](https://github.com/anthropics/claude-agent-sdk-typescript/issues/389)
- 相关官方边界：**B**：[Claude Agent SDK MCP docs](https://code.claude.com/docs/en/agent-sdk/mcp)

**kernel 边界**：配额在“最终进入模型/持久化/前端队列的完整表示”上统一计量；同时限制 bytes、items、schema size、nesting 和可展开 artifact，不按字段名猜测成本。

### AP-29　kernel 不要把 durable 约束只放在初始 prompt/transcript

**禁令**：权限、安全规则、预算和不可变任务约束不得只存在于会被压缩、裁剪或总结的自然语言历史中。

**真实代价证据**：Claude Agent SDK 官方文档明确 automatic compaction 会用 summary 替换旧 history，早期 prompt 中的具体要求可能丢失；官方建议把持久规则通过设置源每轮注入，并提供 `PreCompact` 生命周期事件。

- 来源性质：**B，官方 agent-loop/session 文档**：[How the agent loop works](https://code.claude.com/docs/en/agent-sdk/agent-loop)、[sessions](https://code.claude.com/docs/en/agent-sdk/sessions)

**kernel 边界**：硬限制和授权是 typed kernel state/policy，不依赖模型记忆；需要模型遵守的 durable instructions 由宿主每轮重注入，并把 compaction 作为一等事件审计。

### AP-30　kernel 不要暴露没有状态绑定、expiry 和 dispose 的 opaque handle

**禁令**：tool-call token、approval ID、cursor、artifact ref、run/session object 不能无限期有效，也不能只靠 GC 猜何时停止 native work。

**真实代价证据**：DAP 规范规定 variables/scopes reference 只在当前 suspended state 有效，resume 后失效；这避免前端把旧引用用于新状态。flutter_rust_bridge 官方允许 `StreamSink` 在函数返回后长期持有；若没有 terminal/dispose 约束，native-to-UI stream 可以在 UI 已销毁后继续存活。GPUI async context 则把 app/window 已消亡显式建模为可失败访问。

- 来源性质：**B，官方规范/文档**：[DAP Specification](https://microsoft.github.io/debug-adapter-protocol/specification)、[flutter_rust_bridge streams](https://github.com/fzyzcjy/flutter_rust_bridge/blob/main/website/docs/guides/types/translatable/stream.md)、[GPUI Contexts](https://github.com/zed-industries/zed/blob/main/crates/gpui/docs/contexts.md)

**kernel 边界**：每个 handle 绑定 run revision/state 和 expiry；过期返回 `StaleHandle`。run terminal、stream close、subscriber dispose 和 native resource release 的关系写入契约并可测试。

---

## 跨先例反复重现的“旧概念换新名字”

以下不是新增禁令，而是说明哪些问题已被多次重新发明，不能当作“LLM agent 新问题”而省略机制：

| 今天的名称 | 历史名称/先例 | 反复付出的代价 | 对最小 kernel 的含义 |
|---|---|---|---|
| multi-agent handoff / delegation | FIPA CNP、actor send、child workflow | 协议次序不等于候选质量；投递也不等于执行 | correlation、deadline、terminal、取消进核；评分/路由留外 |
| shared memory / workspace | blackboard、ROS parameter server、Kubernetes object store | 无 owner 会失控；无 revision 会 stale/conflict；共享 pointer 有竞态 | owner + schema + CAS/revision + audit |
| human approval | Jupyter stdin、workflow signal、LangGraph interrupt | 未协商会阻塞；异常可被吞；恢复可能错配 | typed request ID + paused state + deadline + idempotent response |
| retries / durable agents | actor restart、DAG retry、Temporal/DBOS/Inngest step | restart 不保消息；retry 重复外部 effect；代码升级破坏 history | effect key、attempt、receipt、revision、NeedsMigration |
| streaming UI | Jupyter IOPub、LSP progress、provider token stream、Kubernetes watch | late output 未定义、取消未闭合、chunk shape 漂移、cursor 过期 | best-effort delta 与权威 terminal/snapshot 分层 |
| agent roles / reasoning loop | BDI intention、GroupChat、Crew manager | 递归不终止、固定 selection 僵化、策略组合死循环 | budgets/cancel/failure 进核；deliberation policy 留外 |
| tool security | object capability、WASI preopen、permission hooks | 名称/文本不是权限；预批准可绕过晚期 callback；revoke 不能倒带 | opaque attenuated cap + pre-action mediation + audit |
| core/frontend split | xi RPC、Jupyter/LSP/DAP/CDP | 协议能降耦，也会制造序列化、版本、背压和 cache 一致性债 | 先稳定 domain semantics；仅在真实隔离/多宿主需求下 wire 化 |

---

## 对 v1 评审的硬性否决问题

以下任一问题回答“是”，应暂停把该设计放进 kernel 稳定面：

1. 删掉 frontend 后，run 是否无法正确推进或判断终态？
2. frontend 是否必须永久保存一串 UI patch 才能恢复，而不能重新取 snapshot？
3. Rust API 是否直接出现 role/backstory/manager prompt/provider raw event/viewport 等产品对象？
4. 是否把 checkpoint、retry 或 actor restart 写成“exactly once”？
5. 是否存在不经显式 effect/capability mediation 就能访问外部 I/O 的路径？
6. 是否有 unbounded spawn、mailbox、token stream、collect 或持久化 payload？
7. cancel 是否只改变 UI，或原 start/request 永远没有 terminal closure？
8. approval 是否没有 request ID、revision、deadline、actor 与 stale-handle 语义？
9. restore not found/version mismatch 是否会静默新建 run？
10. 代码升级后，旧 history 是否会在没有 revision/migration 检查的情况下由新逻辑继续？
11. child 是否拿到 parent/master 的全部 authority，而 revoke 语义又没有区分未来调用与既成副作用？
12. hook 是否能递归 drive 当前 run，或 child/controller 可能形成同步 await 环？
13. token/structured/artifact 中是否只有一个字段受预算限制？
14. 是否用“最后 token”“流关闭”“最近 error”“cleanup hook 已调用”推断权威终态？
15. 当前是否只有一个 consumer，却已经承诺长期维护跨进程公共协议的认证、版本、背压和重连？

这份否决表刻意不要求 v1 内建通用 DAG、actor cluster、数据库、provider、UI 或工作流语言；它只要求 kernel 不重复先例已经付过代价的边界错误。
