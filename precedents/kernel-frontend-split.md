TL;DR：四个先例共同说明：把 runtime 与 frontend 用协议隔开能换来语言/进程独立、可观测和多客户端，但 kernel 只应承诺带关联 ID、能力协商、取消/生命周期与有界事件流的“执行语义”，不应把 UI、编排策略或不稳定对象句柄塞进稳定 API。

## 概览

### 研究问题与结论范围

- 本笔记比较 Jupyter Kernel Messaging、LSP、DAP 与 CDP 四种“核心运行时—工具/前端”边界。
- 重点是协议为何分层、能力如何演进、以及对 Rust agent kernel 的最小接口面意味着什么。
- 【文档事实】Jupyter 是多 socket / 多消息流的 kernel 协议；LSP、DAP 是以 JSON-RPC 风格消息为主的工具协议；CDP 是 Chromium/Chrome 的领域化调试协议。
- 【推断】它们不是 agent system，不能直接证明 agent 的最佳架构；但其异步执行、前端竞争、流式输出、取消和协议演进问题高度同构。

### 横向速览

| 先例 | 主要目标 | 前后端边界 | 演进方式 | 对本题最有价值的设计 |
|---|---|---|---|---|
| Jupyter | 多 frontend 操作计算 kernel | ZeroMQ 五通道、签名消息 | 消息类型与可选能力 | 将执行控制、广播副作用、用户输入、紧急控制拆开 |
| LSP | 编辑器 × 语言实现的 N×M 降耦 | stdio/socket 上 JSON-RPC | initialize capabilities、dynamic registration | 先协商、未知字段忽略、标准化取消和 progress |
| DAP | 通用 debug UI × 各 debugger | 工具—adapter—debugger | capability flags，version 固定为 1 | adapter 隔离异构后端；状态句柄的寿命明确 |
| CDP | DevTools/自动化工具 × Chrome runtime | HTTP 发现 + WebSocket JSON | schema 生成、stable/tot 分支 | target/session、domain 边界、版本精确匹配 |

### 协议化相对于进程内 library 的取舍

- 【文档事实】DAP 明确选择 wire protocol 而非 API/client library，以允许 adapter 用任何语言实现。
- 【文档事实】Jupyter 的 shell ROUTER 能接受多个 frontend 连接；IOPub 是广播的 side-effect 流。
- 【文档事实】CDP 允许前端通过 remote debugging endpoint 连接到 browser/page target；DevTools frontend 本身也是其 client。
- 【推断：收益】协议把 Rust ABI、内存所有权和发布节奏从 frontend 脱钩，可令 Web/桌面/CLI、远程进程或第三方 adapter 复用一个 kernel。
- 【推断：收益】消息有自然的 trace/correlation 边界，适合日志、录制、replay、故障隔离和跨进程权限收口。
- 【推断：收益】能力协商可让旧 frontend 不认识的新 feature 时安全降级，避免库 API 的同步升级锁步。
- 【推断：代价】序列化、复制、队列、背压、断连恢复与双向 RPC 会引入延迟和大量状态机；小而同进程的调用不应无故远程化。
- 【推断：代价】协议一旦公开，字段、时序和错误码都会成为兼容性承诺，重构成本高于私有 Rust trait。
- 【推断：决策】核心可先提供进程内 Rust API，并让同一份 command/event schema 驱动可选 IPC bridge；避免把“只能经网络使用”误当成“前后端分离”。

## 核心抽象与接口面

### Jupyter Kernel Messaging：五条具有不同 QoS 的通道

#### 连接与包信封

- 【文档事实】connection file 向 client 提供 transport、ip、五个 port、key、signature_scheme 与 kernel 名称等连接资料。
- 【文档事实】协议消息的逻辑部分为 `header`、`parent_header`、`metadata`、`content`、可选 `buffers`。
- 【文档事实】`header` 至少含 `msg_id`、`session`、`username`、`date`、`msg_type`、`version`；5.5 还可有 `subshell_id`。
- 【文档事实】ZeroMQ wire envelope 在 identity frames 后以 `<IDS|MSG>` 分隔，随后为 HMAC 签名和四个序列化 dict，raw buffers 可附在末尾。
- 【文档事实】HMAC 默认 sha256，签名覆盖四个序列化 dict 的串接；空 key 时验证被禁用。
- 【推断】`msg_id`/`parent_header` 是 agent kernel 必需的 causality primitive：一条 tool output、approval 或 child-agent event 必须能回指发起 run/command。

#### Shell：有回复的正常请求路径

- 【文档事实】Shell 是 request/reply 通道，用于执行代码、查询对象、请求 prompt 等；kernel 端为 ROUTER，能接多 frontend。
- 【文档事实】最低必需 kernel 行为是 `kernel_info_request` 与 `execute_request`，以及 IOPub 的 busy/idle 状态。
- 【文档事实】典型 execute content 包含：

```json
{
  "code": "print(1)",
  "silent": false,
  "store_history": true,
  "user_expressions": {},
  "allow_stdin": true,
  "stop_on_error": true
}
```

- 【文档事实】对应 reply 可带 `status: "ok" | "error"`、`execution_count`、`payload`、`user_expressions`；error 有 `ename`、`evalue`、`traceback`。
- 【文档事实】执行消息的惯例时序是：先在 IOPub 发 `status: busy`，处理、回 shell reply，最后 IOPub 发 `status: idle`。
- 【文档事实】`idle` 表示与该 request 关联的 IOPub 消息已发出。
- 【推断】agent kernel 可对应为 `Command -> CommandResult`，但不要只靠最终 result：应定义 terminal event 与 result 的先后/幂等规则。

#### IOPub：广播副作用而非请求结果

- 【文档事实】IOPub 是 kernel 发布 stdout、stderr、display 和其他 side effect 的 broadcast channel。
- 【文档事实】其多 frontend 理由是：所有 frontend 都应知道其他 frontend 向 kernel 发了什么及产生什么副作用。
- 【文档事实】5.5 将 PUB 改为 XPUB，以支持 `iopub_welcome`。
- 【文档事实】Jupyter 规定 execute 的 busy/idle 在该通道；也在此流中发布 `stream`、`execute_result`、`display_data`、`error` 等消息。
- 【文档事实】spec 明说在 `idle` 之后的异步 output 行为目前未定义。
- 【推断】这揭示“run 已完成”与“仍有 background agent/tool output”必须分开建模：给 agent run 分配 `run_id`，把 late event 定义为允许或禁止，并规定订阅重连语义。
- 【推断】IOPub 类事件流需要 bounded buffer、drop policy/sequence number 或 ACK；否则慢前端可拖累执行或无声丢事件。

#### stdin：核心反向向前端索取交互

- 【文档事实】stdin 让 kernel 在 `raw_input()` 类场景向 active frontend 发 input request，再收 input reply。
- 【文档事实】执行 request 中若 frontend 不支持输入，必须设置 `allow_stdin: false`，否则 kernel 可能阻塞等待 reply。
- 【文档事实】stdin 与 shell 需要同一 ZMQ IDENTITY，保证 kernel 路由回正确 client。
- 【文档事实】Jupyter 明确说 prompt 是 frontend feature，kernel 没有 prompt knowledge。
- 【推断】agent kernel 只应发结构化 `InputRequest {run_id, request_id, schema, reason, deadline?}`；表单渲染、approval 文案、凭据存储和默认值策略由 frontend/application 管。
- 【推断】必须把“无人响应/客户端断线/拒绝”作为一等结果，绝不可无限期卡住一个 worker。

#### Control：不能被长任务饿死的控制面

- 【文档事实】Control 在语义上与 Shell 相同但走独立 socket，目的正是避免排在执行请求后面。
- 【文档事实】它服务 shutdown、restart、debug；spec 建议以独立线程处理，使长 shell 请求不阻挡控制。
- 【推断】agent kernel 至少应将 `CancelRun`、`Shutdown`、`Health`/`Drain` 归入高优先级 control plane，而不是普通 command queue。
- 【推断】control 的线性化点必须文档化：取消是“已请求”、还是“保证未开始后续 step”，以及何时给 terminal state。

#### Heartbeat：仅存活，不等于正确

- 【文档事实】heartbeat 使用 REQ/REP simple bytestring ping/echo，不是完整 JSON 消息。
- 【文档事实】它确认前端与 kernel 的连接性。
- 【推断】liveness heartbeat 不能推导 run 完成、事件未丢或 worker 健康；Rust API 若已有 superviser，可把 health/status 单独暴露而不要把 ping 当业务状态。

### LSP：先建立会话契约，再按 feature 通信

#### N×M 问题与基础消息模型

- 【文档事实】LSP 的目的是让语言 server 与开发工具通过统一协议通信，减少每种 editor/IDE 对每种语言重复集成的成本。
- 【文档事实】base protocol 是 HTTP-like framing：ASCII header、必需 `Content-Length`、空行、UTF-8 content。
- 【文档事实】content 是 JSON-RPC 2.0：request 有 `id/method/params?`，response 有 `id/result?/error?`，notification 无 response。
- 【文档事实】LSP 有标准错误码，如 `ServerNotInitialized (-32002)`、`RequestCancelled (-32800)`、`ContentModified (-32801)`。
- 【推断】agent kernel 应采用同构的 `request / response / notification` 区分；“发生了 event”不应伪装成对一次请求的 response。

#### 初始化与 capability negotiation

- 【文档事实】`initialize` 必须是首个且只能发一次 request；在成功回应前，服务器拒绝其他 request（exit 例外）并丢弃 notification。
- 【文档事实】初始化参数含 `capabilities: ClientCapabilities`、`clientInfo?`、`processId`、`workspaceFolders?`、`initializationOptions?` 等。
- 【文档事实】返回是 `InitializeResult { capabilities: ServerCapabilities; serverInfo? }`。
- 【文档事实】客户端应忽略不理解的 server capabilities；server 应忽略未知 client capability；缺失 capability 即不支持。
- 【文档事实】新增 position encoding 用 client preference `general.positionEncodings`，由 server 在 `capabilities.positionEncoding` 选择，且 server 必须支持 utf-16。
- 【文档事实】一些 server capability 可通过 dynamic registration 在会话中变更，不限于 initialize 时静态宣布。
- 【推断】不要只暴露一个膨胀的 `KernelOptions`。定义一次握手：frontend 声明可渲染的 artifact/input schema、可承受 event volume、是否能执行 client action；kernel 返回可提供的 tool/stream/replay/approval feature。
- 【推断】能力字段应 additive、默认 false、未知忽略；为 Rust enum 预留 `Unknown(String)`/开放字符串，而非关闭式 enum 导致新 server 崩溃。

#### cancellation、progress 与顺序

- 【文档事实】客户端以 `$/cancelRequest` notification 加 `CancelParams { id }` 取消；被取消的 request 仍必须返回 response。
- 【文档事实】`$/progress` 有 `token` 和 `value`；work-done progress 使用 begin/report/end 三种 kind，可标 `cancellable`、`percentage`。
- 【文档事实】server-initiated progress 要 client 宣布 `window.workDoneProgress`，并先请求 `window/workDoneProgress/create`；token 应只使用一次。
- 【文档事实】response 应大致按 request 顺序，但仅在不影响 correctness 时可重排。
- 【推断】`CancelRun` 最好是独立 control request，返回 accepted/not-found；原始 `StartRun` 最终仍返回 `Cancelled` terminal result，保证每个 command id 闭合。
- 【推断】progress 是 UI 提示而非状态机真相；frontend 不应从百分比决定是否可重试或扣费。

### DAP：用 adapter 将后端特性翻译到最小调试语义

#### 分层与协议格式

- 【文档事实】DAP 解决“各种 debugger API 各自配一套 UI”的重复劳动；debug adapter 夹在 development tool 和 concrete debugger 之间。
- 【文档事实】DAP 特意是 wire protocol 而不是 API/client library，使 adapter 可使用任意语言实现。
- 【文档事实】单会话模式由 tool spawn adapter，经 stdin/stdout 通信；多会话模式则连接已在监听 port 的 adapter。
- 【文档事实】base protocol 同样是 `Content-Length` framing 加 UTF-8 JSON；消息分 request、response、event。
- 【文档事实】request 的真实形状例如：

```json
{"seq":153,"type":"request","command":"next","arguments":{"threadId":3}}
```

- 【文档事实】其核心字段为 `seq`、`type`、`command`、`arguments`。
- 【推断】若 agent kernel 必须适配不同 LLM provider、sandbox、queue 或 worker runtime，可把它们置于 adapter/provider trait 后面，但不能把 provider-specific payload 写进稳定 frontend protocol。

#### capability 与会话时序

- 【文档事实】DAP 故意维持协议 version 1，新 feature 通过 capability flag 做向后兼容；缺 flag 等于不支持。
- 【文档事实】`initialize` 中 tool 传 supports* 能力、path format、行列基准、locale；adapter 在 response 的 `Capabilities` 返回它能做的事。
- 【文档事实】adapter 在可配置时发 `initialized` event；client 才设置 breakpoints 等，再发 `configurationDone`。
- 【文档事实】`launch`/`attach` 的参数故意不被 DAP 统一定义，属于 debugger-specific。
- 【推断】这是强边界范例：kernel 可定义 `StartRun` 与状态，但模型供应商的 credentials、temperature、部署名、网络拓扑应由 application 的 adapter/config 解释。

#### 状态生命周期

- 【文档事实】在 stopped 后典型请求链为 `threads -> stackTrace -> scopes -> variables`。
- 【文档事实】variables/scopes 的 reference 整数只在当前 suspended state 有效，resume 后失效；evaluate/output refs 尽可能保留更久。
- 【文档事实】session 结束必须有 `terminated` event；launch 与 attach 的 disconnect 语义不同（前者可 kill、后者 detach）。
- 【推断】agent runtime 中 `AgentHandle`、stream cursor、tool-call token、checkpoint id 也都要标注有效期；不要让前端缓存 opaque ID 后在状态转换时获得静默错误或错误对象。

### CDP：面向一个具体 runtime 的宽域、schema 驱动协议

#### domain 与传输

- 【文档事实】CDP 按 domain（DOM、Debugger、Network 等）定义 commands 和 events，均是固定结构的序列化 JSON。
- 【文档事实】其定义来自 Chromium 维护的 `browser_protocol.pdl` 与 `js_protocol.pdl`；镜像仓库生成 JSON、TypeScript 与 Closure 定义并发包。
- 【文档事实】browser 以 `--remote-debugging-port` 暴露 HTTP discovery；`/json/version` 给 browser WebSocket URL，`/json/list` 给每个 target 的 WebSocket URL。
- 【文档事实】实际协议走如 `/devtools/page/{targetId}` 的 WebSocket；HTTP 还提供 new/activate/close target endpoint。
- 【推断】当一个 kernel 会管理多个 project/run/worker 时，`Kernel -> Run/Target -> Session` 的资源层次值得明确，而不是一个全局 WebSocket 混杂所有消息。

#### 版本与前端竞争

- 【文档事实】CDP 有 tip-of-tree（频繁变化且无 backward compatibility 保证）、stable 1.3（Chrome 64 的较小子集）和 v8-inspector（Node 调试/分析子集）。
- 【文档事实】`GET /json/protocol` 可取运行中 browser 的精确 schema，含 `experimental` 等标志。
- 【文档事实】Chrome 63 起有多 client 支持；但嵌入式 DevTools 打开时会终止 remote connection，client 会收到 `Inspector.detached`，reason 可为 `replaced_with_devtools`。
- 【推断】多前端连接不等于多 controller：agent kernel 要声明 controller lease/owner、observer 权限、抢占规则和 detach 后未完成 approval 的命运。
- 【推断】CDP 的 stable/tot 分叉说明：若快速扩张 feature，可提供明确标为 unstable/experimental 的 namespace；不要暗中把不兼容变更塞入“稳定”核心。

## 它把什么放进核心、什么留给使用者

### Jupyter

- 【核心】wire envelope、认证签名、socket 角色、execute/info 的最小语义、busy/idle、输入反向请求、控制和存活探测。
- 【使用者/前端】prompt 的呈现与含义；display 的富渲染；是否允许 stdin；UI 如何显示 stdout/error。
- 【使用者/前端】kernel language 的具体代码、package 环境与执行策略由 kernel implementation 自己决定。

### LSP

- 【核心】JSON-RPC framing、初始化状态机、标准 request/event 形状、capability 语义、取消/progress/错误码、文档位置规则。
- 【使用者/前端】server 的进程管理和生命周期由 client/tool 决定；用户体验、具体 UI 与大部分 configuration 不由协议规定。
- 【使用者/实现者】自定义 `$/'` message 可用，但 request 必须正确回 `MethodNotFound`；不能假装是标准互操作 feature。

### DAP

- 【核心】调试命令/事件的抽象、adapter handshake、会话生命周期、source/thread/variable 的最低语义。
- 【使用者/adapter】`launch`/`attach` 参数特意保持 debugger-specific；实际 debugger API、子进程与环境解释由 adapter 实现。
- 【使用者/前端】如何排版字符串型展示值、哪些 controls 可见、如何安装或选择 adapter。

### CDP

- 【核心】browser target discovery、WebSocket 命令/事件、domain schema、browser/page session 边界。
- 【使用者/客户端】界面、自动化策略、对 experimental domain 的采用、与特定 Chrome 版本的匹配。
- 【边界警示】它的 domain 面很宽；这适合同一 runtime 的深度工具化，不是所有 agent kernel 都应复制的“全能协议”范式。

## 踩坑与教训

- 【规范事实，Jupyter】`idle` 后异步 output 未定义；若完成事件含义不严，frontend 会错误地把“暂时安静”当作“永远结束”。
- 【规范事实，Jupyter】IOPub 的欢迎消息不能识别是哪位 client 的订阅触发；广播协议若缺 per-subscriber identity，审计和精细重连会受限。
- 【规范事实，Jupyter】前端若未禁用 stdin，kernel 可能等待输入而阻塞；interactive capability 必须提前协商且超时。
- 【规范事实，LSP】取消仍须回 response；只发 cancel event 而不闭合原请求会造成 caller future 泄漏和 UI pending。
- 【规范事实，LSP】协议不支持一个 server 被不同工具共享；“多 client/多 tenant”不是免费副作用，要另设 ownership/isolation。
- 【规范事实，DAP】object reference 在 resume 后失效；opaque handle 不声明 lifetime 会诱发竞态和陈旧 UI。
- 【规范事实，CDP】tip-of-tree 无向后兼容保证，且 frontend 可替换其他连接；版本策略和连接仲裁必须写入契约。
- 【社区/事故资料结论】本轮未找到可核实、且直接针对上述协议核心设计的作者 postmortem；不把泛化的社区抱怨伪装为因果结论。
- 【推断】“事件流”需有重放 cursor、序号、buffer 上限和 drop signal；四份高层协议文档都不能替使用者解决实际背压。
- 【推断】安全不能只靠 local transport 假设：Jupyter 的 HMAC、CDP remote debugging exposure 都表明连接端点和认证必须可配置、默认最小暴露。

## 对我们 Rust agent kernel 的启示

### 推荐的职责边界

- kernel 负责：run/agent/task 的生命周期状态机、调度执行、取消传播、结构化事件、correlation、持久化/恢复的抽象边界、resource ownership 与权限 hook。
- kernel 不负责：聊天 UI、token/markdown 渲染、审批措辞、具体模型 provider 配置、产品级 agent role、业务工作流策略和最终用户身份产品逻辑。
- application 负责：组装 model/tool/sandbox/credential adapter，设定预算、重试、角色、授权和数据保留策略。
- frontend 负责：订阅及重连、把 event 投影为 UI、采集并提交 InputResponse、显示 capabilities/错误、维护本地视图状态。
- 【推断】Rust crate 的第一个稳定面宜是 domain types + async traits + event stream，而不是直接把 WebSocket/JSON-RPC 作为唯一 public API。

### 最小、可协议化的概念集

```rust
// 示意签名，非来自上述规范。
pub trait Kernel {
    async fn initialize(&self, client: ClientCapabilities) -> KernelCapabilities;
    async fn start_run(&self, request: StartRun) -> Result<RunHandle, KernelError>;
    async fn cancel_run(&self, run_id: RunId, reason: CancelReason) -> CancelAccepted;
    async fn respond_input(&self, response: InputResponse) -> Result<(), KernelError>;
    fn subscribe(&self, from: EventCursor) -> EventStream;
    async fn shutdown(&self, mode: ShutdownMode) -> Result<(), KernelError>;
}
```

- `StartRun` 的同步返回只确认接受/创建；terminal `RunFinished` 必由 event stream 给出，携带 `run_id`、outcome、error 分类和最终 cursor。
- `KernelEvent` 至少区分：`RunStateChanged`、`StepStarted/Finished`、`OutputChunk`、`ToolCallRequested/Resolved`、`InputRequested`、`Warning`、`RunFinished`、`GapDetected`。
- 每个 event 应有单调 `EventCursor`、时间、`run_id`、`causation_id` 和可选 `actor_id`；输出 chunk 不能只靠 arrival order 归属。
- `ClientCapabilities` 至少有 `input`、`approval`、`artifact_mime_types`、`replay`、`max_event_rate`/batching、`client_actions`；不支持即安全降级。
- `KernelCapabilities` 只声明 kernel 真能保证的 feature，如 durable replay、pause/resume、multi-controller、background outputs、tool approval。
- 取消分两层：高优先级 control path 接受取消；run 本身随后保证产生一个 terminal `Cancelled | Completed | Failed`，并定义超时的 `CancellationTimedOut`。
- 明确 ownership：同一 run 的 controller 是否唯一、observer 能否 `respond_input`/cancel、controller 断开后是 lease 超时、自动取消还是转为 orphaned。
- 明确 handle lifetime：tool-call/input token 附绑定 run-state 和 expiry；过期返回可机器处理的 `StaleHandle`，不复用 ID。
- 事件背压：kernel 不能为慢前端无限缓存；支持 cursor replay 时应报告 earliest available cursor，不能重放时发送 `GapDetected` 并要求 snapshot/resync。
- 协议适配层：可在 `kernel-protocol` 中把同一 types 映射为 JSON-RPC/IPC；transport 层另设 auth、framing、connection 与 rate limit，避免污染纯 Rust 核心。

### 不应过早复制的东西

- 不因 Jupyter 有五 socket 就创建五个 Rust trait/port；先按 control、command、event、input 的 QoS 与背压实需切分，进程内可仍是同一个 executor。
- 不因 CDP 有几十 domain 就暴露所有 provider/sandbox 内部能力；稳定 core 应小，扩展放 capability-gated namespace。
- 不把 DAP 的字符串优先表示搬到 kernel；agent state/error/artifact 应是 typed data，文本是 frontend rendering 的一种投影。
- 不把 LSP 的 workspace/document 模型硬套到 agent；借其协商与生命周期纪律，不借其领域对象。

## 来源

### 一手规范与官方文档（本笔记主要事实依据）

1. Jupyter Client Documentation, “Messaging in Jupyter”, protocol version 5.5：
   https://jupyter-client.readthedocs.io/en/latest/messaging.html
2. Jupyter Client Documentation, “Connection files”：
   https://jupyter-client.readthedocs.io/en/latest/kernels.html#connection-files
3. Microsoft, Language Server Protocol Specification 3.17：
   https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/
4. Microsoft, Debug Adapter Protocol Overview：
   https://microsoft.github.io/debug-adapter-protocol/overview
5. Microsoft, Debug Adapter Protocol Specification（消息与 capability 定义）：
   https://microsoft.github.io/debug-adapter-protocol/specification
6. Chrome DevTools, DevTools Protocol（官方 overview、版本、discovery、生成 schema）：
   https://chromedevtools.github.io/devtools-protocol/
7. ChromeDevTools/devtools-protocol（PDL/JSON/TS 生成工件）：
   https://github.com/ChromeDevTools/devtools-protocol

### 证据等级说明

- “【文档事实】”均来自上述一手规范/官方 overview，必要处交叉使用 specification。
- “【推断】”是将先例映射到 Rust agent kernel 的架构建议，不是来源声称的结论。
- “【社区/事故资料结论】”只在明确取得可核验资料时采用；本轮未采到足以作为核心证据的作者 postmortem 或高热度 issue，故未展开。
