TL;DR：Rust 的主流 LLM crate 已能提供单 agent 的 provider/tool/RAG 封装，Swiftide 已接近可暂停的 agent harness + task graph；但可嵌入、可恢复、可观察且前端友好的多 agent kernel 仍是明显空位，公共接口必须从一开始为 `dyn`、`Send`、取消与事件流设计。

## 概览

- 研究范围：rig、Swiftide、llm-chain、AutoAgents、Kalosm，以及用户点名的 `graph-flow` 名称核验。
- 截止：2026-08-18；star、issue、最近 push 是仓库页面/API 当日快照，不是质量保证。
- 证据标记：`[事实]` 来自 README、源码、Rust Reference 或 issue；`[推断]` 是对 kernel 边界的设计解释。
- 结论不能以 star 数替代成熟度；本任务关心的是可复用运行时契约，而不是模型/provider 覆盖数。
- Rust 生态可粗分为四层：provider/model adapter、RAG/indexing pipeline、单 agent tool loop、workflow/task graph。
- 几个 crate 多把前三层混在一个 crate family 中；跨 agent 调度、持久化恢复、UI 事件协议通常不构成稳定核心。
- 这也是“agent framework”看似很多、真正可抽成 agent kernel 的部分仍少的原因。

### 成熟度快照（不作排名）

- `[事实]` Rig 仓库为 `0xPlaygrounds/rig`，约 8.3k stars，2026-08-18 仍有 push；README 明说未来更新会有 breaking changes，并有 `MIGRATING.md`。
- `[事实]` Swiftide 为 `bosun-ai/swiftide`，约 773 stars，2026-08-15 仍更新；README 明说 pre-1.0、agent harness 和 task graph API 仍在收敛。
- `[事实]` llm-chain 为 `sobelio/llm-chain`，约 1.6k stars；README 示例依赖 `llm-chain = "0.12.0"`，仓库页面未见 archive/deprecation 告示。
- `[事实]` AutoAgents 为 `liquidos-ai/AutoAgents`，约 734 stars、2026-08-18 更新，仓库创建于 2025-02；是相对新项目。
- `[事实]` Kalosm 为 `floneum/kalosm`，约 2.2k stars、2026-08-17 更新；其 Fusor backend 自称 early development、not ready for production。
- `[事实]` 以上项目都有活跃迹象，但没有一项的 README 声称提供完整的 durable、可恢复、多 agent kernel。
- `[事实]` GitHub repository search 对精确名 `graph-flow` 找到的是 max-flow/min-cut 图算法仓库（`SuperInstance/graph-flow`，0 stars）及非 agent 的 Reactive Graph Flow 系列。
- `[事实]` 未找到一个可核验、被广泛采用且名为 `graph-flow` 的 Rust LLM agent crate；不能据名称把它列作 Rust agent 先例。
- `[推断]` 若项目资料意指某个私有 crate、改名 crate 或 LangGraph 风格实现，需要补充 crates.io/GitHub URL 才能作可靠比较。

### 横向定位

- Rig：以 portable LLM/application primitives 为中心，agent 是“经典 runtime”之一。
- Swiftide：同仓提供 RAG streaming pipeline、agent harness、typed task graph，最接近本研究的 orchestration 对照。
- llm-chain：较早的 prompt/chain/executor 抽象；公开 README 的接口面主要是宏和单链执行。
- AutoAgents：把 Agent、Environment、Memory、Executor、protocol event 组合成 multi-agent 产品框架，能力面较宽但稳定契约尚难验证。
- Kalosm：本地/预训练模型和 typed structured generation 优先，不是 agent runtime。
- `[推断]` 因此最值得借用的不是任一项目的全栈，而是 Rig 的可移植 tool/model 契约与 Swiftide 的显式暂停/恢复状态机。

## 核心抽象与接口面

### Rig

- `[事实]` README 把 workspace 分为 `rig-core`（portable tools、memory、vector-store contracts）和 `rig-agent`（builder、prompt/streaming traits、typed hooks、contextual tools、extraction、可序列化 `AgentRun` state machine）。
- `[事实]` README quickstart 的构造链是：
  ```rust
  let client = openai::Client::from_env()?;
  let agent = client.agent(openai::GPT_5_2)
      .preamble("...")
      .build();
  let response = agent.prompt("Entertain me!").await?;
  ```
- `[事实]` 因此公开体验以 `Client -> AgentBuilder -> Agent -> prompt` 为中心，`prompt` 是 async 且返回 `Result`。
- `[事实]` 单一 vector-store 抽象下列有 10+ integration；README 所列包括 LanceDB、Qdrant、MongoDB、Neo4j、PostgreSQL、SQLite、SurrealDB、Milvus 等。
- `[事实]` `Agent::into_tool(self) -> DynamicTool` 将配置好的 agent 转成 runtime-defined tool，agent name 成为 tool name，未设置时默认 `agent_tool`。
- `[事实]` 该 adapter 的参数 schema 是单字段 JSON schema（`{ prompt: String }`），内部调用 `agent.prompt(args.prompt)`。
- `[事实]` 这个实现使用 `DynamicTool::new(name, description, parameters, closure)`，closure 内 `Box::pin(async move { ... })`，并用 `Arc<Agent>` 捕获 agent。
- `[事实]` README 宣称统一 20+ model provider、支持 streaming/prompting workflow、OpenTelemetry GenAI semantic convention；没有展示多 agent orchestration API。
- `[事实]` portable core 可用于 `wasm32-unknown-unknown`，README 明说 WASI 不支持、`rmcp` 是 native-only。
- `[推断]` `Agent -> Tool` 适合递归委派，却不等同于多 agent 生命周期管理：没有看到 supervisor、mailbox、run tree、budget 分配或恢复协议。

### Swiftide

- `[事实]` workspace 有 `swiftide-agents`、`swiftide-core`、`swiftide-indexing`、`swiftide-query`、`swiftide-tasks` 等，明确把 agent、RAG、任务图拆分。
- `[事实]` agent builder 的关键表面为：
  ```rust
  Agent::builder()
    .llm(&llm)
    .tools(tools)
    .on_new_message(hook)
    .limit(...)
    .build()
  ```
- `[事实]` builder 可接受 `context(impl AgentContext + 'static)`、`add_tool(impl Tool + 'static)`、`add_toolbox(impl ToolBox + 'static)` 和 before/on/after hooks。
- `[事实]` `Agent` 的执行面包括：
  ```rust
  pub async fn query(&mut self, query: impl Into<Prompt>) -> Result<(), AgentError>
  pub async fn query_once(&mut self, query: impl Into<Prompt>) -> Result<(), AgentError>
  pub async fn run(&mut self) -> Result<(), AgentError>
  pub async fn run_once(&mut self) -> Result<(), AgentError>
  pub async fn history(&self) -> Result<Vec<ChatMessage>, AgentError>
  pub async fn stop(&mut self, reason: impl Into<StopReason>)
  ```
- `[事实]` `Agent` 把 `state` 放在实例内，而对话存于 `Arc<dyn AgentContext>`；clone 后共享 context、重置 retry/pending message，因此 clone 被文档明确标为“不便宜”。
- `[事实]` 公开只读状态面是 `is_running()`、`is_stopped()`、`is_pending()`、`state()`、`stop_reason()`。
- `[事实]` lifecycle 至少可见 pending → running → stopped；重入会得到 `AgentError::AlreadyRunning`。
- `[事实]` stop reason 包括 `NoNewMessages`、`RequestedByTool`、`ToolCallsOverLimit`、`FeedbackRequired`、`AgentFailed`、`Error`。
- `[事实]` `ApprovalRequired` tool 返回 `ToolOutput::FeedbackRequired(payload)` 时，agent 不写 tool output、保留未履行 call、以 `StopReason::FeedbackRequired` 停止。
- `[事实]` caller 通过 context 交 `ToolFeedback::approved()` 或 `refused()`，再 `run_once`；这是一种真正的 interrupt/resume 语义而非 callback 弹窗。
- `[事实]` tool loop 的重试以 tool call（name+args hash）计数，默认上限 3；未履行 tool call 会产生 `AgentError::UnfulfilledToolCalls`。
- `[事实]` agent 内并发 tool invocation 用 `tokio::spawn` 并 join，join 失败映射为 `AgentError::ToolFailedToJoin`。
- `[事实]` hooks 被存为 boxed hooks；测试中的 hook 返回 `Box::pin(async move { ... Ok(()) })`。
- `[事实]` task layer 把任务定义成 typed graph of `TaskNode`：`Task`、`TaskRunOutcome::{Completed, Paused}`、`Transition`、`NodeId`、`TaskAgent`、`SimplePrompt`。
- `[事实]` graph 装配 API 包括 `register_node`、`starts_with`、`register_transition`、`transitions_with`、`Transition::next`、`transitions_to_finish`、`task.run()`。
- `[事实]` indexing pipeline 是另一类流式组合：`Pipeline::from_loader(...).then_chunk(...).then(...).then_in_batch(...).then_store_with(...).run()`。
- `[推断]` Swiftide 最有价值的边界是：agent loop 管短期运行状态/调用顺序，task graph 管编排，context 管消息持久域；但 durable checkpoint 和跨进程 transport 仍非已证实的核心保证。

### llm-chain

- `[事实]` README 的最小执行 API 是：
  ```rust
  let exec = executor!()?;
  let res = prompt!("system", "user")
      .run(parameters()!, &exec)
      .await?;
  ```
- `[事实]` 由示例可知 `executor!()` 返回可 `?` 的结果、`prompt!` 接收模板字符串、`.run(parameters, &executor)` 是 async。
- `[事实]` README 将概念列为 prompt templates、chains、tools/agents、vector stores，并称 chain 可按步骤完成复杂任务。
- `[事实]` README 声称 agent 可访问 Bash、Python、web search 一类 tool，但未在该页展示 Tool trait、agent loop 或多 agent 协议。
- `[事实]` README 没有 durable execution、checkpoint/resume 或 UI event wire protocol 的说明。
- `[推断]` 它是 prompt-chain building block，不宜反向作为 kernel 的生命周期/观察性设计模板；宏式快速路径也不应成为我们跨语言前端接口的中心。

### AutoAgents

- `[事实]` README 的一等概念是 `Agent`、`Environment`、`Memory`、tools、executors（ReAct/Chain-of-Thought）、output、guardrails、pipeline、hooks。
- `[事实]` `Environment` 被定义为管理 agent lifecycle 与 communication；README 还声称 typed pub/sub communication 与 environment management。
- `[事实]` builder 示例近似为：
  ```rust
  AgentBuilder::<_, DirectAgent>::new(...)
      .llm(llm)
      .memory(Box::new(SlidingWindowMemory::new(10)))
      .build().await;
  agent.run(Task::new("What is 1 + 1?"));
  ```
- `[事实]` tool 有 `#[tool(...)]`、`ToolInput`、`ToolRuntime`，示例执行面为 `async fn execute(&self, args: Value)`；README 还声明 sandboxed WASM tool runtime。
- `[事实]` workspace 有 `autoagents-protocol`，README 将其描述为 shared protocol/event types。
- `[事实]` memory 默认示例是 sliding window、可扩展 backend；只提及 Qdrant vector-store crate，未文档化 checkpoint、session resume 或 event sourcing。
- `[事实]` README 未文档化 scheduler、cron/task queue、human approval/interrupt；CLI 可从 YAML 跑 workflow、通过 HTTP 服务。
- `[事实]` provider 行为的明确降级包括：不支持 streaming 的 provider 返回 typed streaming-unsupported error，而不是 panic；一些 multimodal input 返回 typed error。
- `[推断]` Environment + protocol 是 multi-agent kernel 的有用命名，但若没有稳定 message ordering、delivery、idempotency、cancellation、durability 契约，就应视为应用框架层而非最小 kernel API。

### Kalosm

- `[事实]` Kalosm 的定位是 Rust 中 pre-trained model 的简单 interface，覆盖 language/audio/image，并有本地推理和 typed structured generation。
- `[事实]` README 示例模型交互是 `Llama::phi_3().await`、`model.chat()`、`.with_system_prompt(...)`。
- `[事实]` task API 为 `model.task("...").typed()`，再以 `task(&input, &model)` 取得 stream/typed output；结构通过 `#[derive(Parse, Schema)]` 表达。
- `[事实]` README 提供 context extraction、chunking、semantic search/vector integration 的例子链接，但未展示 tool-call trait。
- `[事实]` README 没有 dedicated agent、workflow orchestration、multi-agent 或 distributed abstraction。
- `[事实]` Fusor 是 WGPU quantized inference runtime，native/browser 目标；README 明说它尚不适合 production，计划在 0.5 成为 backend。
- `[推断]` Kalosm 适合成为 kernel 外部的 local model/structured-output adapter；不应把其模型下载、GPU/backend feature 或 parser grammar 放入 orchestration kernel。

## 它把什么放进核心、什么留给使用者

### 已有项目共同放入“核心”的内容

- `[事实]` provider-neutral LLM request/response、prompt/message、streaming、structured extraction 或 embedding/vector contracts。
- `[事实]` 单 agent 的 tool schema、tool call、tool result、turn limit/retry/stop condition。
- `[事实]` agent-local message history/context 与短生命周期状态。
- `[事实]` async execution hooks 和 observability integration（至少 tracing/OTel 的宣称或 hook 面）。
- `[事实]` Swiftide 进一步将 typed task graph、transition 与 `Paused` outcome 放入 library。
- `[事实]` AutoAgents 进一步把 environment、pub/sub、protocol event types 放入 framework。

### 项目通常留给使用者/应用层的内容

- `[事实]` Rig、llm-chain、Kalosm 的文档没有给出多 agent supervisor、mailbox、team state ownership 或公平调度 API。
- `[事实]` 没有上述项目的 README 把 durable run persistence、crash recovery、exactly-once tool call 写成核心承诺。
- `[事实]` UI 的 event schema、reconnect cursor、render policy、approval dialog、credential storage 均未见统一 API。
- `[事实]` AutoAgents 虽称 typed pub/sub，公开 README 未给出 delivery ordering/ack/retry/backpressure 的规范。
- `[事实]` Swiftide 的 approval 要求 caller/context 送回 feedback；显示与身份认证仍属调用者。
- `[推断]` kernel 应拥有“可观察的运行事实”和“暂停点”，前端应拥有“如何展示、何人批准、如何输入凭证”。
- `[推断]` provider SDK、RAG pipeline、MCP client/server、WASM sandbox 是 capability adapter，应该是可选 integration，不是 Rust kernel 的必需依赖。

## 踩坑与教训

### 运行时/产品教训

- `[事实，Swiftide #686]` 用户报告 indexing pipeline 在正常条件下滥用 WARN、DEBUG 有时只写摘要有时倾倒完整 prompt/response、fan-out 后缺 node ID，导致难以追踪一个 node 的端到端路径。
- `[事实，Swiftide #686]` 该 issue 仍 open；提议是 TRACE 放 payload、DEBUG 放带 node ID/timing 的 step summary、INFO 放 pipeline lifecycle，WARN 要给可行动解释。
- `[推断]` kernel 事件不能只输出字符串日志：每个 run/agent/step/tool call 应有稳定 ID、父子关系、时间与结构化 lifecycle event；日志是事件投影而非唯一 API。
- `[事实，Swiftide source]` tool invocation 已经会并发 spawn，且 join failure 是一个显式错误分支。
- `[推断]` 如果 kernel 暴露 parallelism，必须同时定义 cancellation、child failure aggregation、等待/abandon 语义；否则前端看到的“完成”不可靠。
- `[事实，Rig README]` Rig 自己预告 breaking changes；`rig-agent` 新旧 API 的迁移成本证明 rapidly evolving LLM abstraction 不宜渗入窄 kernel ABI。
- `[事实，AutoAgents README]` non-streaming provider 返回 typed error 而非 panic 是正确降级模式。
- `[推断]` kernel 需把 `Unsupported`、`Rejected`、`NeedsApproval`、`Cancelled`、`Failed` 做成可区分状态，不能把所有失败折为文本。

### Rust 语言/接口教训

- `[事实，Rust Reference]` `async fn` trait method 和 return-position `impl Trait` 含 opaque return type，trait object (`dyn Trait`) 不 dyn-compatible。
- `[事实，Rust Reference]` 带 `where Self: Sized` 的 method 可留在 trait 内但不能通过 `dyn Trait` 调用；generic method、裸 `Self` 参数/返回等亦会影响 dyn compatibility。
- `[事实，Tokio tutorial]` `tokio::spawn` 的 task 要求 `Future + Send + 'static`；只有跨 `.await` 保留的数据均为 `Send`，future 才是 `Send`。
- `[事实，Swiftide source]` 它同时用 `Arc<dyn AgentContext>`、`Box<dyn Tool>` 与 `tokio::spawn`；这让 context/tool 与其 future 的 thread-safety 成为实际 API 约束。
- `[推断]` 对需要 registry/插件/前端驱动动态装配的 kernel，核心 extension trait 不应仅写 native `async fn`；应显式提供 object-safe boxed-future ABI，或同时提供 generic fast path 与 dyn adapter。
- `[推断]` 不要让公共 trait 的 builder/generic helper 与 runtime vtable 混在一起：把 `Tool` 的 object-safe dispatch 面与 `ToolExt`/builder 的 `Self: Sized` convenience 面分开。
- `[推断]` 盲目把所有 trait 设为 `Send + Sync + 'static` 会排斥 WASM/single-thread/local UI capability；应分出 local executor/API 与 multi-thread host adapter，或把 `Send` 仅置于 spawn boundary。
- `[推断]` `&mut Agent` 形式如 Swiftide 易表达独占 loop，但跨任务/UI 控制会受借用期限制；更通用的 kernel 应让 run state 由 runtime 拥有并通过 command/event channel 交互。

## 对我们 Rust agent kernel 的启示

### 建议最小职责边界

- `[推断]` kernel 核心只负责：run/agent identity、状态机、命令受理、capability invocation、暂停/恢复/取消、父子 run、结构化事件、资源预算钩子。
- `[推断]` 不负责：prompt 模板、provider 选择、RAG/indexing、具体 agent persona/plan、UI 渲染、审批 policy、秘密管理、网络 transport、长期存储实现。
- `[推断]` kernel 可以定义 persistence `Snapshot`/`EventSink` port，但不应内置某数据库或承诺 exactly-once；tool 需有调用 ID，应用层可据此实现幂等。
- `[推断]` 以“kernel emits facts；host decides effects”替代“kernel prints logs/calls UI”。

### 建议的最小对象模型

- `[推断]` `RunId`、`AgentId`、`ParentRunId`、`CommandId`、`CapabilityId`、`InvocationId` 应为 first-class opaque IDs。
- `[推断]` `RunState` 至少区分 `Created | Running | WaitingForInput | WaitingForApproval | Completed | Failed | Cancelled`，并携带可序列化 reason。
- `[推断]` `KernelCommand` 只需 `Start`、`DeliverInput`、`ResolveApproval`、`Cancel`、`Resume`、`Shutdown`；不要把 prompt/RAG DSL 固化其中。
- `[推断]` `KernelEvent` 至少有 `RunStateChanged`、`MessageAppended`、`CapabilityRequested`、`CapabilityCompleted`、`ApprovalRequested`、`ChildSpawned`、`Diagnostic`；每条有 monotonic `sequence` 或 `EventId` 供前端重连去重。
- `[推断]` capability 调用输入/输出宜用 versioned, serializable envelope（例如 JSON/CBOR values + schema/version），不要把 provider-specific Rust type 穿过 kernel。
- `[推断]` `ApprovalRequested` 必须有 opaque approval/invocation ID、摘要/structured payload、截止/取消语义；UI 只回传 accept/reject/edit 的决策。

### 建议的 Rust trait 轮廓

- `[推断]` 动态 capability 的 object-safe 形式可类似：
  ```rust
  pub trait Capability: Send + Sync {
      fn describe(&self) -> CapabilityDescriptor;
      fn invoke<'a>(&'a self, call: CapabilityCall)
        -> Pin<Box<dyn Future<Output = CapabilityResult> + Send + 'a>>;
  }
  ```
- `[推断]` 如果要支持 WASM/local non-`Send`，另设 `LocalCapability` + local executor；不要把一个 `dyn Capability` 同时许诺为跨线程和 local。
- `[推断]` `describe` 必须纯同步/可缓存；schema 生成、I/O、模型调用不能隐藏在注册期，避免前端/registry 意外阻塞。
- `[推断]` kernel API 应返回/订阅 `Stream<Item = KernelEvent>` 或 event receiver，而不是接受 UI callback；callback 可在 host adapter 层实现。
- `[推断]` 多 agent 最小原语可只是 spawn child run、send addressed message、await child outcome；role/team/router/planner 都放到 library user。
- `[推断]` 同时提供 `Kernel` generic embedding API 与 `dyn KernelHandle`/channel control handle，避免 `&mut` 借用跨 await 绑定前端会话。

### 验收问题（设计评审可直接问）

- 一个 tool 在执行中被 cancel，事件序列、最终 state、是否可再次 resume 分别是什么？
- 一个 tool 先 side effect 后进程崩溃，恢复时怎样凭 `InvocationId` 交给 host 做幂等/人工修复？
- child agent 失败、超时、等待 approval 时，parent 是否自动取消、等待、继续，谁定义 policy？
- 同一 run 的两个前端重连时怎样按 sequence 补事件，怎样避免重复审批？
- provider 无 streaming、tool 无法在 WASM 使用、capability 拒绝请求时是否都是 typed、可显示的结果？
- 动态注册 capability 后，接口是否仍可通过 `Arc<dyn Capability>` 调度且不破坏 `Send`/WASM 策略？

## 来源

- Rig README（架构、quickstart、breaking/WASM 限制）：https://github.com/0xPlaygrounds/rig
- Rig `Agent::into_tool` source（agent-to-dynamic-tool adapter）：https://raw.githubusercontent.com/0xPlaygrounds/rig/main/crates/rig-agent/src/agent/tool.rs
- Swiftide README（agent/task/RAG 三层、pre-1.0 声明）：https://github.com/bosun-ai/swiftide
- Swiftide `Agent` source（builder、状态、tool concurrency、approval）：https://raw.githubusercontent.com/bosun-ai/swiftide/master/swiftide-agents/src/agent.rs
- Swiftide #686（logging/correlation 运行经验）：https://github.com/bosun-ai/swiftide/issues/686
- llm-chain README（`executor!`/`prompt!`/`.run` 示例）：https://github.com/sobelio/llm-chain
- AutoAgents README（Environment/protocol/tools/memory/限制）：https://github.com/liquidos-ai/AutoAgents
- Kalosm README（local model、typed task、Fusor 成熟度）：https://github.com/floneum/kalosm
- Rust Reference，Dyn compatibility（async trait/object safety 规则）：https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility
- Tokio tutorial，Spawning（`Send + 'static` 与跨 await 状态）：https://tokio.rs/tokio/tutorial/spawning
- `graph-flow` 搜索到的非 agent 同名仓库（名称歧义核验）：https://github.com/SuperInstance/graph-flow
