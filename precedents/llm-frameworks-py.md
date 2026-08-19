TL;DR：四个 Python 先例都把「一次可观测、可暂停的 agent 运行」做成 runtime，但编排范式分别为显式 state graph、消息对话/角色、任务/crew/flow、agent handoff；可靠持久化、幂等副作用、UI 与业务授权仍必须由使用者承担，Rust kernel 应优先统一 Run/State/Event/Interrupt/Tool 边界而非复制任一上层 DSL。

## 概览

### 研究范围与证据等级

- 对象：LangGraph（当前 OSS Python 文档）、AG2（AutoGen 继承分支）、CrewAI、OpenAI Agents SDK Python。
- 「文档事实」来自各项目官方文档，版本能力以页面当前描述为准。
- 「issue 事实」只描述报告者观察到的问题、项目状态和讨论量；它不是对所有版本/模型都成立的证明。
- 「推断」是面向本项目 Rust dependency library 的设计归纳，明确标记。
- 不把 LangGraph 的 LangSmith、CrewAI 的 CLI、OpenAI 的 tracing backend 等托管/产品层误当作最低 runtime 必需项。

### 横向结论（文档事实）

- LangGraph 的原语是带 reducer 的 shared state、node、edge、checkpoint 和 interrupt；multi-agent 是 graph composition，而非独立「团队」运行时。
- AG2 的原语是 `ConversableAgent`、message、reply/turn 与 GroupChat/Swarm 模式；多 agent 的自然单位是对话及 speaker selection。
- CrewAI 同时有角色化 `Agent` + `Task` + `Crew` 的工作流 DSL，和可编排方法的 `Flow`；前者偏任务队列，后者更像事件驱动 graph。
- OpenAI Agents SDK 的原语是 `Agent`、Runner、tool、handoff、session 和 `RunState`；handoff 被编码为模型可调用的 transfer tool。
- 四者均有某种增量输出；但都没有把前端协议简化为唯一稳定、跨框架的 event schema。
- 持久化在四者中都不是「任意闭包/工具执行的 exactly-once」保证：保存的主体是 conversation/state/checkpoint/serialized run，而外部副作用仍需应用层幂等化。

### 适合作为 kernel 还是应用层的对照

- 可下沉 kernel：run 生命周期、状态快照与版本、事件订阅、暂停/恢复令牌、工具调用的计划/结果、取消、错误、审计关联 ID。
- 不宜强加 kernel：角色 prompt、选择哪位 agent、任务分解、LLM provider、向量 memory、具体数据库、网页/CLI 审批 UI。
- 需要留扩展点而非内置策略：state merge/reducer、调度/路由、授权决策、checkpoint store、event sink、tool execution、token counter。

## 核心抽象与接口面

### LangGraph：state graph + durable execution

#### 核心模型（文档事实）

- `StateGraph(State)` 建图；state 可用 `TypedDict`、Pydantic 或 dataclass，node 收 state 并返回 partial update。
- reducer 决定并发/多 node 更新同一 key 时的合并语义；消息 state 常使用 `MessagesState`。
- `START` / `END` 是特殊边界，node 与 `add_edge`/`add_conditional_edges` 显式定义控制流。
- 最小 API 形态：

```python
from langgraph.graph import StateGraph, MessagesState, START, END

def node(state: MessagesState):
    return {"messages": [{"role": "ai", "content": "hello"}]}

builder = StateGraph(MessagesState)
builder.add_node(node)
builder.add_edge(START, "node")
builder.add_edge("node", END)
graph = builder.compile(checkpointer=checkpointer, store=store)
graph.invoke({"messages": [{"role": "user", "content": "hi"}]}, config=config)
```

- `invoke`/`ainvoke` 得最终状态；`stream`/`astream` 逐步 yield；新版还推荐 `stream_events` 的 typed projection。
- agent 可作为 node/subgraph 插进父图；图能混合确定性 hand-written step 与 LLM-driven step。
- 因而其「协作」的核心不是角色对象：开发者可建 supervisor/router，也可让每个 agent 是一个 subgraph，调度策略完全外置。

#### 状态与持久化（文档事实）

- compile 时传 `checkpointer`；每个 `thread_id` 对应一条 thread scoped checkpoint history。
- invocation 典型 config：`{"configurable": {"thread_id": "thread-1"}}`。
- `InMemorySaver` 仅 RAM，进程重启丢失；SQLite/Postgres saver 可持久化，Postgres 要 `checkpointer.setup()` 建表。
- checkpointer 服务短期会话连续性、fault tolerance、HITL 和 time travel；app-defined `Store` 则跨 thread 存长期事实/偏好/共享知识。
- checkpoint 会随长会话增长，pruning/retention（例如 cron 删除）由使用者负责。
- subgraph 有自己的 checkpoint namespace；父 graph 不会自动即时得到子图 state 更新，文档给出的替代是共享 Store 或写回 parent checkpoint。

#### 流式事件（文档事实）

- `stream_mode` 有 `values`（每 step 全 state）、`updates`（每 node update）、`messages`（`(LLM token, metadata)`）、`custom`、`checkpoints`、`tasks`、`debug`。
- v2 chunk 是 `{ "type": mode, "ns": namespace_tuple, "data": payload }`；v1 的 tuple/dict 形状会随单/多 mode 和 subgraph 组合而变。
- `messages` metadata 含 `langgraph_node`、tags，可过滤；带 `nostream` tag 的调用仍执行但 token 不发出。
- 要收到 subgraph 的消息，必须 `subgraphs=True`；否则父图 `messages` stream 看不到其内层 token。
- `tasks`/`debug` 要 checkpointer；`custom` 允许 node 通过 writer 发业务进度，非 LangChain 模型也可由此转发流。

#### Human-in-the-loop（文档事实）

- node 内 `from langgraph.types import interrupt; answer = interrupt(payload)`，payload 必须 JSON serializable。
- interrupt 通过 special exception 停止、持久化、无限等待；使用 checkpointer 与同一 thread ID 才能恢复。
- 以同 thread ID 调 `graph.invoke(Command(resume=value), config)`；该 `value` 成为 `interrupt()` 的返回值。
- 多个并行 interrupt 可 `Command(resume={interrupt_id: answer, ...})`，供 UI 对每一项答复。
- stream 可以暴露 `interrupts`（含 `Interrupt(value, id)`）、`interrupted` 与最终 `output`，所以 UI 可不用解析 prompt 文本。
- static `interrupt_before` / `interrupt_after` 是 debugging breakpoint，文档明确不建议拿来做生产 HITL。

#### LangGraph 的职责边界（文档事实）

- 核心负责：graph execution、state reducer、checkpoint/state history、interrupt/resume、stream/progress、subgraph namespace。
- 使用者负责：state schema、node 函数、edge/route、工具/外部 IO、checkpointer 的部署与 retention、审核 UI、side-effect 幂等性。
- LangGraph 以 durable runtime 自居，但没有规定业务对象/角色层；这种克制使它更像 kernel 而非 agent application framework。

### AG2（AutoGen）：conversation + agent roles

#### 核心模型（文档事实）

- `ConversableAgent` 是 AG2 的 foundational building block，兼具「brain/personality」；其他 agent 继承或等同于它。
- agent 的 `name`、`system_message`、`llm_config` 表达身份、指令和模型配置；它会同 agent/human 传消息、执行 tool、决定/接收 reply。
- 核心交互可形为 `agent.run(message, max_turns, user_input)`；返回 lazy iterator，开始迭代才执行，逐步产出 event/message/metadata。
- `process()` 是把该 iterator 消费并在 console 印出、处理 input 的便利层，不是必需 execution primitive。
- GroupChat / Swarm 等 orchestration 把一组 conversational agent 放进共同消息历史与 speaker/termination 机制；应用程序在 pattern/manager/selector 层决定谁发言与何时停。
- `human_input_mode`（例如 `"ALWAYS"`）是 agent 级 human participation 策略，不是通用的 durable interrupt token protocol。

#### 状态、流与持久化（文档事实）

- AG2 对话历史和 event iterator 是可观察的运行信息；`run()` 的 lazy event iterator 是它适合接自定义 UI 的主要入口。
- 所查基础 `ConversableAgent` 文档未定义 core-level checkpointer、线程 ID、serialized resume 或 storage protocol。
- 因此不要因为「对话有历史」推断为 crash-safe persistence；应用需要选择自己的 conversation storage / resume 设计。
- 文档页未定义统一 token-stream event union；调用方需消费其 event/message 流，或按所用 LLM adapter 的能力处理。

#### AG2 的职责边界（文档事实）

- 核心负责：agent persona、turn/message/reply、LLM/tool 接入、群聊语义与基础 human-input mode。
- 使用者负责：role prompt、agent roster、group speaker/termination policy、模型 provider 配置、对话长期存储、生产审批和 UI。
- 与 LangGraph 相比，它把「谁在对话」的抽象放进核心，但没有由该基础页承诺通用 durable workflow runtime。

### CrewAI：roles/tasks/crew 与 event-driven Flow 双层 DSL

#### Crew / Task 核心（文档事实）

- `Agent(role, goal, backstory, tools, llm, ...)` 代表角色化执行者；`Task(description, expected_output, agent, context, ...)` 是可交付任务。
- `Crew(agents=[...], tasks=[...], process=...)` 是协作容器；默认 `Process.sequential`。
- `Process.hierarchical` 用 manager 做 delegation/validation，要求 `manager_llm` 或 `manager_agent`；这把调度策略固化为 Crew DSL 选项。
- `kickoff()` / `akickoff()` 返回结果；另有 `kickoff_for_each` 和 thread-based async API。
- `CrewOutput` 提供 `raw`、`pydantic`、`json_dict`、`tasks_output`、`token_usage`，即应用层结果对象并非单一 state snapshot。
- Crew 配置还混有业务运行策略：cache、memory、max_rpm、planning、callbacks、embedder；其接口面比 kernel 更宽。

#### Crew checkpoint 与流（文档事实）

- `Crew(..., stream=True)` 后 `streaming = crew.kickoff(inputs=...)`，迭代 chunk 的 `content`，流耗尽后从 `streaming.result` 拿最终值。
- `checkpoint=True` 每 task 保存到 `.checkpoints/`；`CheckpointConfig` 指定 location、events、Json/SQLite provider、max_checkpoints。
- 可 `Crew.from_checkpoint(".checkpoints/latest.json")` 再 `kickoff()`；CLI 也可按 task id replay。
- 此 checkpoint 的文档语义是 task 进度/replay，不能据此推导外部 tool 副作用的 deduplication。

#### Flow 核心（文档事实）

- `Flow[StateModel]` 是另一套显式控制流；`@start()` 标入口，`@listen(upstream)` 订阅步骤输出，`@router(fn)` 返回 label 决定下游。
- `or_(a,b)` / `and_(a,b)` 表示任一/全部 upstream 完成才触发；多个 start 常可并行。
- state 可以是自动带 `id` 的 dict (`self.state["key"]`)，或 Pydantic 泛型 state (`self.state.key`)；文档推荐后者取类型安全。
- `flow.kickoff()` / `flow.kickoff_async(inputs={...})` 返回最后完成的方法输出，而不是规定统一 run-result envelope。
- `@persist` 可标在 class 或 method；默认 `SQLiteFlowPersistence`，自定义 backend 实现 `FlowPersistence`。
- 以 `kickoff(inputs={"id": uuid})` 从最新 snapshot 恢复；`restore_from_state_id` 是 fork（载入历史但给新 `state.id`）。unknown ID 会静默回退，`restore_from_state_id` 与 `from_checkpoint` 混用报 `ValueError`。

#### Flow 的流与 human feedback（文档事实）

- class `stream = True` 时 `kickoff()` 产生 session，sync `for` 或 async `async for` 消费 `StreamFrame`。
- `StreamFrame` 暴露 `content`、`channel`（文档例子为 `llm`/`tools`）、`type`、`event`（包含 `method_name`、`task_name`、`agent_role` 等）。
- session 有 `result`、`frames`、`is_completed`、`is_cancelled`、`close/aclose`；必须把所有 frame 迭代完才取 result。
- `@human_feedback(message=..., emit=[...], llm=..., default_outcome=...)`（文档称 v1.8.0+）暂停给人反馈。
- 有 `emit` 时 LLM 把自由文本映射到 outcome label，匹配对应 `@listen`；没有 `emit` 时 listener 收 `HumanFeedbackResult`。
- `last_human_feedback` / `human_feedback_history` 可读；Slack/webhook 等非阻塞 provider 的实现另在专页，不在 Flow 主文档 contract 内。

#### CrewAI 的职责边界（文档事实）

- 核心负责：角色/task/crew 一次运行、sequential/hierarchical policy、Flow event graph/state persistence、stream session、feedback decorator。
- 使用者负责：任务/角色语义、manager 提示词、外部工具安全与幂等、数据库/部署选择、审批通道与权限/timeout policy。
- 其上层 DSL 很有生产力，但 Agent/Task/Crew 不宜成为 Rust kernel 的强制公共对象：它们已包含组织结构和产品偏好。

### OpenAI Agents SDK：runner + handoff + resumable run state

#### 核心模型（文档事实）

- `Agent(name=..., instructions=..., tools=..., handoffs=...)` 是 LLM 加 instructions/tools 的实例。
- `Runner.run_sync(agent, input)` / async `Runner.run(...)` 执行 loop，结果常用 `result.final_output`。
- 多 agent 的首选协调原语是 handoff：给 `handoffs=[target_agent]` 时，SDK 将其给模型暴露成 `transfer_to_<target_agent>` tool。
- `handoff(agent, tool_name_override=..., tool_description_override=..., on_handoff=..., input_type=..., input_filter=..., is_enabled=...)` 可定制 transfer。
- `input_type` 是本次 handoff tool-call JSON schema；模型返回 JSON 会被本地验证后给 callback，但它不替换下一 agent 的完整 conversation input，也不动态选择目的 agent。
- handoff destination 固定；需要动态选 target 时必须自定义 `Handoff` dataclass。只需调用 specialist 而不移交聊天时用 `Agent.as_tool()`。
- 默认下一 agent 看完整 prior history；`input_filter` 可变换，而 `HandoffInputData` 区分 input history、pre-handoff items、new items 与 run context。

#### Session / persistence（文档事实）

- `session = SQLiteSession("conversation_123"); await Runner.run(agent, "prompt", session=session)` 是基本接法，同步 runner 亦可。
- runner 运行前取 history 并 prepend input；本 turn 新的用户/assistant/tool item 自动写回 session，免去手工 `.to_input_list()`。
- 一个 session ID 是一个 isolated conversation thread；不同 agents 可共享同一 session，看同一历史。
- built-in backend 包含 SQLite/async SQLite、Redis、SQLAlchemy、MongoDB、Dapr、server-managed OpenAI conversations、advanced SQLite、encrypted wrapper 等。
- `Session` 是 structural protocol：须具 `session_id`、`session_settings` 和 history 的四个方法；若要 runner context，四个方法须显式有 keyword `wrapper: RunContextWrapper`。
- `SessionSettings(limit=N)` 限制取回 history；`RunConfig.session_input_callback` 控制 history/new input 如何合并，persist 的仍是 new turn items。
- session 不可与 server-side `conversation_id` / `previous_response_id` / `auto_previous_response_id` continuation 混用。

#### Streaming（文档事实）

- `result = Runner.run_streamed(agent, input)`，`async for event in result.stream_events(): ...`；可用 `RunState` 作为 input 恢复。
- stream 没结束就不算 run complete：最后 token 后仍可能有 session persistence、approval bookkeeping、compaction；应用必须 drain iterator 后检查 `result.is_complete`。
- event union 包三类：`raw_response_event`（provider 原始 event，如 `response.output_text.delta`）、`run_item_stream_event`、`agent_updated_stream_event`。
- raw text delta 典型为 `event.data.delta`；这给 UI 快速 token 显示，但它暴露 provider-specific transport 细节。
- run item 在完整生成时发出，`event.name` 覆盖 `message_output_created`、`handoff_requested`、`handoff_occured`（历史拼写）、`tool_called`、`tool_output`、`mcp_approval_requested` 等。
- agent 更新 event 用 `event.new_agent` 表达 handoff 后当前 agent；很适合前端做「当前执行者」视图。
- `result.cancel()` 立即停，`cancel(mode="after_turn")` 让当前 turn 收尾；若在 tool turn 后取消，文档要求以 normalized input 和 `last_agent` 恢复，不能盲目补一条新用户消息。

#### HITL / approval 与 run state（文档事实）

- `function_tool`、`Agent.as_tool`、Shell/patch tool 可配置 `needs_approval=True` 或 async predicate；MCP 有自己的 approval 配置。
- pending approval 出现在 `RunResult.interruptions` / `RunResultStreaming.interruptions` 的 `ToolApprovalItem`，含 agent name、tool name、arguments。
- 按顺序：drain stream → `state = result.to_state()` → `state.approve(interruption)` 或 `state.reject(interruption)` → `Runner.run(agent, state)` / `run_streamed` 恢复。
- 可只处理一部分 interruption；未处理的下一次仍会 pause。嵌套 `Agent.as_tool` 和 handoff 内的 approval 都浮到外层 run state，且须以最初 top-level agent 恢复。
- `always_approve` / `always_reject` 对本 run 内同 tool identity sticky，并会被 state serialization 保留。
- state 可 `to_json()` / `to_string()` 保存；context 有 serializer/deserializer/strict options，但文档警告 serialized state 带 app context/runtime metadata，勿不经意把 secret 放入 context。
- malformed JSON/非 object/NaN arguments 下，approval callable fail-closed，转为人工审批。

#### SDK 的职责边界（文档事实）

- 核心负责：agent loop、tool/handoff invocation、session integration、stream event、run-state interruption/serialization、trace hooks。
- 使用者负责：agent instructions、tool implementation、session backend 的数据生命周期、approval UI/身份认证/timeout/escalation、业务 side-effect 的 idempotency。
- 它为「tool approval」提供很完整的中断状态机，却仍不提供通用业务 workflow graph；这是有意选择 handoff-first 协作模型。

## 它把什么放进核心、什么留给使用者

### 共同放进 runtime 的最小闭环（文档事实）

- 输入/状态 → step/agent/tool 选择 → item/state 更新 → terminal/paused/failed/cancelled 的生命周期。
- 逐步可观察性；全部至少能把 token 或 message 输出给调用者，较成熟者还给 tool/handoff/task/state event。
- 可恢复身份：LangGraph 的 `thread_id` + checkpoint，Crew Flow state ID/snapshot，OpenAI Session ID + `RunState`；AG2 基础页没有同级契约。
- 人机介入：LangGraph 的 general interrupt，Crew Flow 的 feedback decorator，OpenAI 的 tool approval state；AG2 是 conversation-level human input mode。

### 明确留给应用/前端的部分（文档事实 + 推断）

- [文档事实] 选择角色、prompt、routing/speaker strategy、最大轮次和终止判定大都由调用方配置或编码。
- [文档事实] 生产 persistence provider、保留/清理、schema migration、secret management 不是 execution core 自动解决。
- [文档事实] 审批的展示、认证、RBAC、超时/升级策略通常在 SDK 外；即使 SDK 有 pause/resume，也只给状态与 API。
- [推断] 前端应仅订阅 kernel 规范化 event 并向 kernel 提交 command；不要直接知道 Python object、graph node、provider token event 或 Pydantic class。
- [推断] 人/agent/tool 都应在 event 中以 stable entity ID 表达；display name、avatar、prompt、组织图是 UI/application metadata。
- [推断] prompt/角色/crew/task 是可选 adapter 层，不能冻结为 library 必选 trait，否则 Rust kernel 被某一框架的 authoring model 锁死。

## 踩坑与教训

### 可恢复不等于 exactly-once（文档事实）

- LangGraph：resume 会从含 `interrupt()` 的 node 开头重跑，interrupt 前代码也重跑；文档要求前置 side effect 要幂等（upsert）或移动到 interrupt 后/独立 node。
- LangGraph：同一 node 中 interrupt 的调用顺序按 index 严格匹配；条件跳过或不确定循环会错误匹配恢复输入。不要 `while True + interrupt()`，应把 pending question 存 state、用 edge 循环。
- CrewAI 高讨论 open issue #5802（97 comments，2026-05）报告 task retry 会重执行已成功 `@tool`，可能重复 payment/email/trade；报告者指出 in-memory dedup 无法跨 worker redispatch，需执行前以 durable external key claim。（issue 报告，不是普适实证。）
- [推断] kernel 的 ToolCall 必须从一开始携带 durable `operation_id`/idempotency key，并可先写 `started` 再执行；不能只在最终 event 留 tool result。

### 流式接口会被组合性击穿（文档事实）

- LangGraph issue #5249（27 comments）报告 `create_react_agent` 加 tools 后 `astream(..., stream_mode="messages")` 从逐 token 变为整段输出；说明 token stream 对 model/tool adapter/agent composition 很脆弱。（issue 报告。）
- LangGraph 文档也要求 subgraph tokens 用 `subgraphs=True`，且 v1 chunk shape 随 mode 数/子图而变；调用 UI 若只匹配裸 tuple 将很脆弱。
- OpenAI Agents SDK 明说最后 token 后 stream 仍可做持久化/approval bookkeeping；前端不能以「没有 delta」判 run 已完成。
- [推断] 前端协议应分离 `TokenDelta`、`StepStarted/Finished`、`ToolCall`、`StateCheckpoint`、`RunTerminal`，每个有 version、run_id、seq、origin/path；runtime 不承诺 provider raw event 稳定性。

### 人机审批需要明确的状态机（文档事实）

- AG2 issue #1756（10 comments）报告 GroupChat + `human_input_mode="ALWAYS"` 会无限循环而从不 prompt；role/termination/human policies 的组合可死锁。（issue 报告。）
- OpenAI 的 approval 因而要求 streaming 先 drain、再 inspect interruptions，且只能以原 top-level agent + RunState resume；把 pause 当成新 turn 会破坏历史。
- LangGraph 禁止 broad `try/except` 包 `interrupt()`，否则会吞掉控制流 exception；还要求 JSON-serializable payload。
- [推断] kernel pause 是 first-class `Paused{requests:[Interrupt]}` terminal-adjacent state，而不是 throw string exception 或 console input；resume command 要用 request ID/expected state version 做乐观并发控制。

### 模型/tool adapter 不可靠时必须审计执行事实（issue 事实）

- CrewAI issue #3154（66 comments）报告 custom LLM + WebSearchTool 场景中模型生成了貌似完整的 Thought/Action/Observation，但 `tool.run()` 从未执行、trace 也无 tool event。（issue 报告；不能外推到所有 provider。）
- [推断] 前端不可把模型文本中的 `Action:` 当作执行证据；kernel 应只由 executor 发 authoritative `ToolCallStarted/ToolCallFinished/ToolCallFailed` 事件。
- [推断] 需要 policy hook 在 tool execution 前后做 allow/deny、参数 validation、审计，而不是只能修改 agent prompt。

### 状态边界和资源生命周期（文档事实）

- LangGraph checkpoint 会无界增长，retention 是调用方职责；subgraph checkpoint namespace 也会使 parent-visible state 与真实执行状态不同步。
- Crew Flow 的 unknown restore state ID 会静默 fall back；因此上层必须区分「fresh run」与「restore failed」，不能用同一种成功返回。
- OpenAI session 与 provider continuation options 不可组合；其 compaction 还可能使 token 输出后 stream 开数秒，关注低延迟 UI 时需显式规划。
- [推断] Rust kernel 的 persistence trait 应返回显式 `NotFound`/`VersionConflict`/`Corrupt`，不应静默 new state；backpressure/retention/compaction 必须可见于 metrics/event。

## 对我们 Rust agent kernel 的启示

### 建议的最小公共模型（推断）

- `Kernel::start(spec, input, options) -> RunHandle`：kernel 只接可序列化 `RunSpec` 或注册的 plan ID，不接 Python closure/前端组件。
- `RunHandle::events(cursor) -> Stream<EventEnvelope>`：envelope 至少有 `run_id`、monotonic `seq`、timestamp、causation/correlation ID、path、schema version。
- `RunHandle::command(expected_revision, Command) -> CommandResult`：Command 包 `Resume { interrupt_id, value }`、`Approve/Reject { operation_id }`、`Cancel`；revision 防双击/并发审批。
- `StateStore`：load/checkpoint/append-event/claim-operation 四类语义，显式结果 `NotFound/Conflict`；短期 thread state 与跨 run memory 刻意分开。
- `Executor`：把 LLM、tool、sub-agent 统一为可调度 step，但 tool 的 effect type、timeout、retry、idempotency policy 必须显式。
- `Policy`：在 `BeforeOperation`、`AfterOperation`、`BeforeHandoff`、`BeforeResume` 插入 allow/deny/transform，而非把治理写成 prompt。

### 不应放进核心的设计（推断）

- 不强制 `Agent(role, goal, backstory)`、`Crew(Task[])`、GroupChat speaker selection、supervisor prompt、React/terminal UI。
- 不绑定 provider raw streaming event、OpenAI-style session、Postgres/SQLite、vector memory、tracing SaaS。
- 不把 approval decision 简化成 bool：保留 request kind、display payload、arguments preview、decision、actor、reason、deadline 与 audit record；认证由外层提供。
- 不承诺 exactly-once 外部 effect；明确至少一次重试模型，并给 durable operation claim/idempotency-key hook。

### 面向前端的事件与前端职责（推断）

- 前端读取 stable domain events：`RunStarted`、`StepScheduled/Started/Finished`、`TextDelta`、`MessageCommitted`、`OperationProposed/Started/Finished`、`InterruptRequested/Resolved`、`Checkpointed`、`RunPaused/Completed/Failed/Cancelled`。
- 前端维护 presentation projection（timeline、current agent、approval dialog、token buffer、retry banner），不得从 prompt 文本或 provider raw JSON 推理 state。
- 前端提交带 run revision 的 command；display labels/role colors/markdown rendering 由前端或 application metadata 管理。
- 前端可在断线后用 event cursor 重放，再拉取 latest state/checkpoint；这比持有 process-local iterator 可靠。

### 取舍建议（推断）

- 以 LangGraph 的「显式 state + checkpoint + interrupt」作为 durability/HITL 参考，而不是照抄其 Python reducer/graph DSL。
- 以 OpenAI SDK 的 `RunState` + typed interruption + agent-changed event 作为 run/approval/UI 参考，而不是照抄 handoff-only routing。
- 将 CrewAI 的 roles/tasks/flow 和 AG2 的 conversational groupchat 作为上层 adapter：它们可编译/翻译到 generic plan，而非 kernel native ABI。
- 优先定义可版本化、可重放 event log 和 command protocol；图、对话、Crew 都是 producer，Web 前端只是 consumer/commander。

## 来源

### 官方文档：LangGraph

- LangGraph overview（StateGraph 示例、长期运行/streaming/HITL 定位）：https://docs.langchain.com/oss/python/langgraph/overview
- Persistence（checkpointer/thread/store/retention/subgraph namespace）：https://docs.langchain.com/oss/python/langgraph/persistence
- Interrupts（`interrupt`/`Command(resume)`、node replay 与 idempotence 规则）：https://docs.langchain.com/oss/python/langgraph/interrupts
- Streaming（modes、v1/v2 chunk、subgraphs、async caveats）：https://docs.langchain.com/oss/python/langgraph/streaming

### 官方文档：AG2

- ConversableAgent（基础抽象、`run` lazy event iterator、`process`、human input）：https://docs.ag2.ai/latest/docs/user-guide/basic-concepts/conversable-agent/
- GroupChat introducing guide（所涉 GroupChat/HITL 示例的上游文档）：https://docs.ag2.ai/latest/docs/user-guide/basic-concepts/introducing-group-chat/

### 官方文档：CrewAI

- Crews（Agent/Task/Crew、process、checkpoint、kickoff/stream）：https://docs.crewai.com/en/concepts/crews
- Flows（decorators/state/persist/fork/human feedback）：https://docs.crewai.com/en/concepts/flows
- Streaming Flow Execution（StreamFrame/session 生命周期）：https://docs.crewai.com/en/learn/streaming-flow-execution

### 官方文档：OpenAI Agents SDK

- SDK overview（Agent/Runner primitives）：https://openai.github.io/openai-agents-python/
- Handoffs（tool 表示、input filter/type、history）：https://openai.github.io/openai-agents-python/handoffs/
- Sessions（session contract/backends/limits/caveats）：https://openai.github.io/openai-agents-python/sessions/
- Streaming（`run_streamed`、event union、drain/cancel/resume）：https://openai.github.io/openai-agents-python/streaming/
- Human-in-the-loop（approval/state serialization/fail closed）：https://openai.github.io/openai-agents-python/human_in_the_loop/

### Issue / 社区讨论样本（均为报告者主张，非文档承诺）

- LangGraph #5249，tools 存在时 message streaming 不逐 token（27 comments）：https://github.com/langchain-ai/langgraph/issues/5249
- AG2 #1756，`human_input_mode="ALWAYS"` groupchat loop 不提示输入（10 comments）：https://github.com/ag2ai/ag2/issues/1756
- CrewAI #3154，模型文本伪造 tool observation、tool 未实际执行（66 comments）：https://github.com/crewAIInc/crewAI/issues/3154
- CrewAI #5802，retry 可能重复 external tool side effect（open、97 comments）：https://github.com/crewAIInc/crewAI/issues/5802
- OpenAI Agents SDK #636，早期请求 native HITL/checkpoint/audit（closed、38 comments；现文档已有 tool approval）：https://github.com/openai/openai-agents-python/issues/636
