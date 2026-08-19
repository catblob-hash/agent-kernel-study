TL;DR：ROS/ROS2 的关键边界是把可互操作的通信语义、可观察的受管生命周期和受约束配置放进运行时，而把任务编排、恢复策略与领域行为留给节点/应用；topic、service、action 是以交互时程而非“消息形状”划分的最小原语组。

## 概览

- **研究对象**：ROS 1/ROS 2 的节点通信、managed/lifecycle node、parameters，以及 Nav2/BehaviorTree.CPP 的行为编排层。
- **范围说明**：ROS 是机器人中间件，不是通用 multi-agent runtime；本笔记提取的是其“跨组件协调核心”的边界经验。
- **文档事实**：ROS 2 将通信分为 topics、services、actions 三种形式；node 是它们的命名、执行、参数与图发现的宿主，而不是第四种业务通信协议。[A][B]
- **文档事实**：ROS 2 保留 ROS 1 的 node/topic/service 心智模型，却为多机器人、实时、嵌入式、弱网络、产品化和生命周期管理而重写底层架构。[E]
- **文档事实**：重写而非直接改变 ROS 1 的理由是：改动对大量既有用户过于高风险；两代预期通过 bridge 并行互通，且 ROS 2 不承诺 API 兼容。[E]
- **推断**：这不是“所有系统都必须四原语”的结论；它表明一旦用户需要流、请求、长任务控制三类不同失败/取消/进度语义，用一个通用 send() 会把语义转移成不一致的约定。
- **推断**：node 应理解为 runtime ownership/supervision 边界，而非强制要求每个 agent 都是进程、容器或网络身份。

### ROS 1 到 ROS 2：为什么是架构替换

- **文档事实**：ROS 1 的 single-master 模式没有标准多机器人做法，现有做法是叠加在 single-master 之上的 hack。[E]
- **文档事实**：ROS 2 选择 DDS 的一个直接收益是分布式 discovery，替掉 ROS master，因而没有单一发现故障点。[F]
- **文档事实**：DDS 在 ROS 2 设计中是 middleware implementation detail；ROS 层刻意以较简单的 Node/Publisher/Subscriber API 隐藏 participant、DataReader/DataWriter 等大部分 DDS 层次。[F]
- **文档事实**：ROS 2 希望可替换不同 DDS vendor，尽量不将 vendor API 泄漏进通用 client-library 接口。[F]
- **文档事实**：DDS QoS 提供从低延迟、类似 UDP 的行为到能容忍长时间断连、类似 TCP 的可靠行为的可调范围；这种灵活性也被设计文档承认为复杂性来源。[F]
- **文档事实**：ROS 2 的实时目标是支持跨进程、跨机器实时控制，但明确前提是合适 OS 和/或硬件支持；不是 runtime 自动给予硬实时保证。[E]
- **文档事实**：ROS 2 设计曾要求本进程内使用零序列化 pointer passing，因为不同 DDS 的共享内存优化不一致，也避免不必要的 `.msg`→IDL 路径。[F]
- **文档事实**：ROS 2 的安全模型建立在 DDS Security 上；一个 secure process 使用 enclave 中的证书、私钥、governance 与 permissions 工件。[G]
- **文档事实**：一个 DDS Participant 只能使用一个 security identity；多个 composition node 共用 context/Participant 时，权限必须合并为同一 enclave。[G]
- **推断**：可插拔 transport、QoS 和安全身份都不能只是 backend 细节；它们会反过来决定库 API 是否需要显式 delivery policy、隔离域和部署身份。
- **推断**：将安全身份按 process/context 而非 logical node 发证，反映了安全边界必须匹配真实内存隔离边界。

## 核心抽象与接口面

### 1. Node：名称空间、资源所有权与执行宿主

- **文档事实**：ROS graph 中 node 是创建 publisher、subscription、service、client、timer、parameter 等实体的 API 宿主；例如 `rclcpp::Node` 直接暴露 topic/service 工厂。[H]
- **文档事实**：典型发布 API 是：
  `create_publisher<MessageT>(const std::string & topic_name, const rclcpp::QoS & qos, const PublisherOptions... & options = ...) -> std::shared_ptr<PublisherT>`。[H]
- **文档事实**：典型订阅 API 是：
  `create_subscription<MessageT>(const std::string & topic_name, const rclcpp::QoS & qos, CallbackT && callback, ...) -> std::shared_ptr<SubscriptionT>`。[H]
- **文档事实**：典型服务 API 是：
  `create_service<ServiceT>(const std::string & service_name, CallbackT && callback, const rclcpp::QoS & qos = rclcpp::ServicesQoS(), CallbackGroup::SharedPtr group = nullptr)`。[H]
- **文档事实**：典型服务 client API 是：
  `create_client<ServiceT>(const std::string & service_name, const rclcpp::QoS & qos = rclcpp::ServicesQoS(), CallbackGroup::SharedPtr group = nullptr)`。[H]
- **文档事实**：action 不在 `rclcpp::Node` 的这些 factory 内，而在 `rclcpp_action` 的独立 `create_server/create_client` API 中；这是 API package 分层，不表示 action 不是 ROS 一等概念。[H][I]
- **推断**：最小 kernel 无须复制“Node 万能对象”；更小的 Rust 抽象可以是 `Kernel` 创建具备 identity、cancellation scope、state store 和 ports 的 `AgentHandle`。
- **推断**：但所有通信资源都应有明确 owner，才能可靠释放、列举、关联日志和应用 policy。

### 2. Topic：可持续的、多对多单向事实流

- **文档事实**：topic 是 publisher 向一个或多个 subscriber 广播的单向通信；它适合持续状态、传感器数据、事件流，而不是要求每条消息有业务答复的交互。[A]
- **文档事实**：topic API 将 `QoS` 放在 publisher/subscription 创建点，而非 `publish()` 每次调用；传输期望是 endpoint contract 的一部分。[H][F]
- **文档事实**：DDS QoS 能调可靠性和网络行为；ROS 2 的设计目标是将 DDS 的巨大层次与细节封装起来，而不是让普通用户直接操作 DDS 对象。[F]
- **文档事实**：action 的 status/feedback 也使用 topic，因此 topic 是可复用 transport primitive，而 action 是在其上定义更高交互协议。[A]
- **推断**：agent kernel 的 `publish(topic, event, DeliveryPolicy)` 应仅承诺 delivery/retention/ordering 等 transport 语义；不要隐含“消费成功”或“任务完成”。
- **推断**：主题数据最好区分 ephemeral event 与 retained/latest-state；否则 UI 重连、late joiner、审计回放的行为会靠猜。
- **推断**：是否支持任意 topic 与 wildcard routing 是产品取舍；最小库只需 typed named stream 和 subscription lifecycle，路由 DSL 可留给上层。

### 3. Service：短请求—响应，而非工作流

- **文档事实**：service client 发送 request 并接收 response；相对于 topic，它编码了明确的双向闭合交互。[A]
- **文档事实**：ROS 2 action 设计明确指出 service 本身不提供执行进度信息。[A]
- **文档事实**：ROS 1 的 service 是同步的；ROS 2 公共 C 层 service 支持异步，这使 action 可组合 services 和 topics。[A]
- **文档事实**：服务在 C++ API 中有专门的 `ServicesQoS` 默认值，并可关联 callback group，显示其调度/通信约束不同于单纯 subscription。[H]
- **推断**：kernel 的 request API 应有 deadline、cancellation、typed error/response 和 correlation，但不应把 long-running job 伪装成永不返回的 request future。
- **推断**：若 API 只暴露 `call()`，用户仍会自行发“started/progress/complete”事件；当这种模式变普遍时，应升级为 action/job 原语。

### 4. Action：可取消、可观测的长期目标协议

- **文档事实**：action 用于 response 可能耗时显著的请求；client 可跟踪 progress、获取 result，并在完成前可选取消。[A]
- **文档事实**：action server 接受或拒绝 goal、处理 cancel、执行、可发布 feedback，最后发送 result；action client 发送 goal、观察 feedback/status、请求 cancel、取得 result。[A]
- **文档事实**：`.action` interface 有 Goal、Result、Feedback 三段，以 `---` 分隔；任意一段可为空。[A]
- **文档事实**：一个 ROS 2 action 在 middleware 上由 **三项 service** 加 **两项 topic** 组成，不是新的底层 DDS primitive。[A]
- **文档事实**：三项 services 是 `SendGoal`、`CancelGoal`、`GetResult`；两项 topics 是 goal status 与 feedback。[A]
- **文档事实**：`SendGoal` 的 request 包含 goal description 和 UUID，response 返回接受与否和 acceptance timestamp。[A]
- **文档事实**：`CancelGoal` 以 goal ID/timestamp 请求取消，response 给出 code 和进入 CANCELING 的 goals。[A]
- **文档事实**：`GetResult` 以 goal ID 获取 terminal status 和用户定义 result；server 可配置 result cache timeout，`-1` 为永久、`0` 为立即丢弃（在处理 pending request 后）。[A]
- **文档事实**：status topic 为 introspection，默认 QoS 是 TRANSIENT LOCAL、history size 1；feedback 的频率由 action author 决定。[A]
- **文档事实**：goal 状态是 ACCEPTED、EXECUTING、CANCELING 三个 active state，加 SUCCEEDED、ABORTED、CANCELED 三个 terminal state；rejected goal 不入状态机。[A]
- **文档事实**：ROS 2 改为 client 生成 UUID，避开 ROS 1 多 client 产生相同 goal ID 的冲突。[A]
- **文档事实**：C++ action server 创建需要 `handle_goal`、`handle_cancel`、`handle_accepted` 三个 callback；官方注释要求它们全部 non-blocking。[I]
- **推断**：这正是 agent long task 的最小协议：`submit(input)->RunId/acceptance`、`watch(RunId)->stream`、`cancel(RunId)`、`result(RunId)`，外加显式 terminal reason。
- **推断**：RunId 必须由 caller 或 kernel 使用全局抗碰撞方案生成，并在所有进度、日志、结果中透传；不可借用自然语言 task title 当 key。
- **推断**：result retention 不是实现细节：UI 断线、恢复、审计、二次查询都依赖它，必须成为 policy/configuration 的一部分。

### 5. Lifecycle node：把“可管理性”设为协议

- **文档事实**：managed node 提供已知接口并遵循固定 lifecycle state machine，内部实现仍可视为 black box。[B]
- **文档事实**：其目标包括：launch/supervisor 能确认所有组件正确初始化后再开始执行，并可在运行中重启或替换 node。[B]
- **文档事实**：primary states 为 Unconfigured、Inactive、Active、Finalized；transition states 为 Configuring、CleaningUp、ShuttingDown、Activating、Deactivating、ErrorProcessing。[B]
- **文档事实**：supervisory transitions 为 create、configure、cleanup、activate、deactivate、shutdown、destroy；除了 Active 中错误外，离开 primary state 由外部 supervisor 发起。[B]
- **文档事实**：Inactive 允许 node 已配置但不处理数据；managed topics 不被读取、managed service request 会立刻失败，保留数据的行为由 QoS 决定。[B]
- **文档事实**：每个 transition（除 create）可作为同名 ROS service 暴露；lifecycle 管理接口本身不受 lifecycle state 对一般通信的限制。[B]
- **文档事实**：节点应在每次 transition attempt 后向 latched `lifecycle_state` topic 发布 end state、transition、result code，不论成功或失败。[B]
- **文档事实**：`onConfigure` 建立 buffer、稳定 pub/sub、parameters；`onActivate` 可做最终准备如取得硬件，但文档警告不应在此做长工作。[B]
- **文档事实**：`onError` 成功可回 Unconfigured，失败可到 Finalized；从 Active 进入 error 时必须覆盖 deactivate 与 cleanup 的职责。[B]
- **文档事实**：设计刻意不允许加自定义 lifecycle states，因为工具链必须支持所有 state；应用状态机应嵌套其内或置于其上。[B]
- **推断**：kernel 应内建小而稳定的 operation lifecycle（例如 Created/Ready/Running/Stopping/Stopped/Failed），不能把每个 agent 的业务 phase 固化为核心 enum。
- **推断**：transition 应可由外部 supervisor 驱动且产生 append-only events；agent 自治重启可支持，但应是受控 policy，而不是无法观测的内部副作用。
- **推断**：把“accept work”与“resource ready”分开：Inactive/Ready 是有价值状态，避免刚启动的 agent 接到无法完成的工作。

### 6. Parameters：配置所有权、类型与原子变更

- **文档事实**：ROS 1 的 global blackboard parameter model 允许所有 node 读写所有参数，造成 ownership/control 问题，driver configuration 还催生了单独的 dynamic_reconfigure。[C]
- **文档事实**：ROS 2 将参数托管在 node，参数寿命隐式绑定 node 寿命，且由 owning node 验证；标识为 full node name + parameter name。[C]
- **文档事实**：参数可为 bool/int64/float64/string/byte array 及对应 homogeneous arrays；拒绝复杂、异构、多维值以保持文本配置与 introspection 可理解。[C]
- **文档事实**：理想 API 包含 get、set、list、describe，既有 local API 也有 ROS service 路径。[C]
- **文档事实**：一组参数必须可 atomic update：任一值未过校验，则整组都不改变；逐项 success/failure 返回给 client。[C]
- **文档事实**：node 可按任意准则接受/拒绝 change；validation 应尽快返回且只处理 accept/reject，不承担下游性能影响。[C]
- **文档事实**：node 发布 parameter-events topic；client library 可在其上实现 per-parameter callback，事件也可被记录和以 remote set 重放。[C]
- **文档事实**：该设计曾列出未完成项：per-parameter subscription、pre-update validation registration、参数变更记录/回放、暴露预期 validation policy。[C]
- **推断**：agent kernel 应避免全局任意 JSON KV；配置应属于 agent/run/policy owner，具备 schema/description、validation、version 与 atomic patch。
- **推断**：runtime config event 与 “任务的业务消息”应是两条通道；前者要求审计、权限和反压/回放策略，后者由应用定义。

### 7. BehaviorTree.CPP 与 Nav2：编排在核心之上

- **文档事实**：Nav2 将 BT Navigator 作为 Core Server；导航以 ROS 2 action 提供，BT 是选择 planner/controller/recovery 等步骤的编排层。[J]
- **文档事实**：Nav2 支持每个 navigation action 选择 BT XML；BT node 按 action、condition、control、decorator 分类，且 custom node library 可作为 plugin 加载。[J]
- **文档事实**：Nav2 包含特有 control nodes，如 PipelineSequence、Recovery、RoundRobin、NonblockingSequence、PersistentSequence、PauseResumeController。[J]
- **文档事实**：BehaviorTree.CPP 的基础 `NodeStatus` 包含 IDLE、RUNNING、SUCCESS、FAILURE、SKIPPED；公开入口是 `executeTick()`，用户实现 `tick()` 与 `halt()`。[K]
- **文档事实**：只有可能返回 RUNNING 的 async node 应实现 `halt()` 来中断；Tree 还提供 `tickOnce()`、`tickWhileRunning()`、`haltTree()` 和 wake-up signal。[K][L]
- **文档事实**：BehaviorTreeFactory 从 XML text/file 创建 tree：`createTreeFromText(text, blackboard)` / `createTreeFromFile(path, blackboard)`；多树 XML 必须有 `main_tree_to_execute` 或单一 `<BehaviorTree>`。[L]
- **文档事实**：factory 通过 `registerNodeType<T>`、`registerBuilder(manifest,builder)`、`registerSimpleAction/Condition/Decorator` 注册 node；builder 的核心形状是 `std::function<std::unique_ptr<TreeNode>(const std::string&, const NodeConfig&)>`。[L]
- **文档事实**：`registerFromPlugin(file_path)` 动态加载库并调用导出符号 `BT_RegisterNodesFromPlugin`；插件必须显式导出该 function。[L]
- **文档事实**：BT ports 是 blackboard entries；`getInput` 读取、`setOutput` 写入，带 `{key}` 的 mapping 指向 blackboard key。[K]
- **文档事实**：blackboard pointer 指向的对象，经普通 `getInput<std::shared_ptr<T>>` 后修改并不 thread-safe；需要 `getLockedPortContent()` 并在锁对象生命周期内访问。[K]
- **文档事实**：precondition 中 `_failureIf/_successIf/_skipIf` 仅在 node 从 IDLE/SKIPPED 启动时检查；只有 `_while` 会在 node RUNNING 中 halt 并返回 SKIPPED。[K]
- **推断**：BT 适合成为前端/应用配置的“控制流解释器”，不应成为 transport 或 agent kernel 的必需核心；kernel 应提供 BT 能调用的 action/run、state、event、cancel 接口。
- **推断**：将编排图定义（XML/JSON/DSL）留给上层允许 Nav2 式按 goal 选择流程、灰度升级与领域 recovery，而不迫使所有库用户接受一个 workflow language。

## 它把什么放进核心、什么留给使用者

### 放进 ROS/ROS2 核心（文档事实）

- node、名称/namespace、discovery 与通信实体的基础模型。[F][H]
- topic 的 publish/subscribe 与在 endpoint 上选择 QoS 的能力。[F][H]
- service 的 request/response 通信实体与异步实现基础。[A][H]
- action 的 goal/cancel/result/feedback/status 协议和标准 goal state machine。[A]
- managed node 的通用 state machine、transition 管理 interface、transition event。[B]
- 参数的 node ownership、类型集合、get/set/list/describe、验证与原子组更新目标。[C]
- 通过 DDS discovery、QoS、transport 与 security enclave 对部署级需求提供机制。[F][G]

### 留给 package、应用、部署者（文档事实）

- topic/service/action 的业务 message schema、命名、何时 publish、feedback 频率及 result cache timeout。[A]
- action 接受策略、取消策略、执行实现和最终成功/中止判断。[I]
- lifecycle callback 内的资源建立、硬件访问、错误恢复策略与 supervisor policy。[B]
- parameter 的具体名称、默认值、约束、业务校验与持久化模型。[C]
- Nav2 的 BT XML、导航 tree 选择、领域 action/condition/control plugin 和 recovery 行为。[J][L]
- DDS vendor 的选择、QoS profile、安全 keystore/enclave 拓扑与部署时权限合并。[F][G]

### 对边界的归纳（推断）

- kernel 应提供“可组合机制 + 机器可读状态”，而非“机器人/agent 应该怎样决策”的 policy。
- 前端应渲染 graph、run/lifecycle、feedback、config schema 与 audit events；不应拥有 authoritative run state 或直接绕过 kernel 改内存。
- 应用负责把领域步骤映射为 handlers/plugins，并把成功标准、retry/backoff、路由和预算编码为 workflow/policy。
- deployment 负责 transport、安全 credentials、resource limits；库只暴露验证过的配置入口和 capability checks。

## 踩坑与教训

- **单一通信原语的语义泄漏（设计共识）**：service 无 progress，ROS 1 actionlib 又只能建在 topics 上，因为 ROS 1 service 同步；ROS 2 将 action 明确组合为 3 services + 2 topics。[A]
- **教训（推断）**：不要为所有 agent interaction 设计“万能 event”；至少在 API 文档中区分 stream、bounded RPC、managed run 的 cancellation/retention/terminality。
- **全局配置黑板失去 ownership（ROS 1→ROS 2 设计反思）**：任意 node 读写参数使 driver/control 变得不可控，ROS 2 改为 node-owned parameter。[C]
- **教训（推断）**：全局共享 state 不等于协作；需要明确 owner、写权限、schema、变更事件与 CAS/atomic group semantics。
- **QoS 的能力与复杂性并存（设计文档）**：DDS QoS 给出很大可调范围，同时被官方承认为需要 ROS 帮用户管理的复杂性。[F]
- **教训（推断）**：Rust API 先提供少量 policy preset（telemetry、state、command、audit），把 vendor-level knob 收在 advanced config；否则前端无法解释“为什么没收到”。
- **生命周期 state 不可随意扩张（设计文档）**：ROS 2 故意不支持 custom lifecycle states，因为工具链需要共同理解它们。[B]
- **教训（推断）**：核心 state machine 要短且稳定；workflow phase、模型思考阶段、业务 approval 等归入 metadata/上层 graph。
- **共享安全身份不等于最小权限（security design）**：composition context 共用 Participant 时必须合并 permissions，且同进程 code 能访问彼此 trusted credentials。[G]
- **教训（推断）**：agent capability token 的授权单位需对齐真正隔离单位；若多 agent 同进程，不能向用户宣称它们彼此密码学隔离。
- **共享黑板的并发陷阱（BehaviorTree.CPP API docs）**：普通 shared pointer port 访问后写入非线程安全，必须使用显式锁定接口。[K]
- **教训（推断）**：不要让 workflow/agent 通过无版本的可变共享 context 协作；优先 immutable event、owned output、versioned state 或 scoped lock。
- **非阻塞 callback 约束（rclcpp_action API docs）**：goal/cancel/accepted callbacks 必须 non-blocking。[I]
- **教训（推断）**：kernel control-plane callback 只能做 admission/transition，实际执行必须投递到 scheduler；否则 cancel、UI、supervisor 都会被长 handler 卡住。

## 对我们 Rust agent kernel 的启示

### 最小完备通信集（推断）

- **EventStream**：`publish(StreamId, Event, StreamPolicy)` 与 `subscribe(StreamId, Cursor, Handler)`；用于多对多进度、telemetry、audit、状态变化。
- **Request**：`request(Target, Req, RequestOptions{deadline, cancel}) -> Future<Result<Resp, CallError>>`；用于短、单结果、具备明确 target 的交互。
- **Run/Action**：`start(Operation, Input, RunOptions) -> RunHandle`，其中 handle 有 `id()`、`events()`、`cancel()`、`result()`；用于长期、progress、result retention、显式 terminal states。
- **不建议把 node 作为第四通信原语**：它是 identity/ownership/lifecycle 容器；可命名为 `Agent`/`Component`，但不应和 `send` 语义混淆。
- **可能的补充不是第四消息类型，而是 State/Config plane**：owner-scoped、schema-validated、versioned、atomic patch 的状态/参数 API，与业务 event 分离。

### 建议的窄 Rust 接口（推断，示意而非 ROS API）

- `Kernel::spawn(spec) -> AgentId`：创建带 identity 与 lifecycle 的受管 component。
- `Kernel::transition(agent, Transition) -> TransitionReceipt`：只能操作稳定核心 state，写入 state-change event。
- `Kernel::start(target, operation, input, opts) -> RunHandle`：统一长期工作而非给每个应用实现自制 job protocol。
- `RunHandle::{id, watch, cancel, await_result}`：watch 可重连，result 有明示 retention policy，cancel 返回 accepted/rejected/already-terminal 等可观测状态。
- `Kernel::call(target, request, opts) -> Result<Response, CallError>`：仅用于 bounded request/response。
- `Kernel::publish(stream, event, policy)` / `subscribe(...)`：明确 stream 与 run feedback 的区别，同时允许 run internally 映射到 events。
- `Kernel::patch_config(owner, expected_version, patch) -> ConfigReceipt`：原子 validate-or-reject；前端只走该 API，不直改 agent internals。
- `Kernel::observe(filter, cursor) -> EventStream<KernelEvent>`：让 UI、supervisor、audit 使用同一 authoritative event log/cursor，而非轮询多份内存状态。

### 语义必须早定（推断）

- Run terminal set 至少要区分 `Succeeded`、`Failed`、`Cancelled`、`Rejected`；`Cancelled` 与执行错误不能合并。
- cancel 是请求/协商，而不是同步 kill；需要定义在 queued、running、already terminal、deadline exceeded 各状态的回执。
- result/feedback/status 应按 `RunId` 可关联；对 late subscriber 的 replay/retention 要显式。
- 每种 stream 需要至少一个交付 preset；不要暗示“exactly once”除非整个存储、ack、去重协议都能兑现。
- lifecycle transition、config patch、run terminal event 是 control/audit plane 的强需求；应带 timestamp、actor、correlation ID、原因码。
- 业务 workflow、BT/DAG、recovery/retry、human approval、LLM provider/tool selection 均不要塞进 kernel；通过 handlers 和上层 orchestrator 调用 kernel。

## 来源

- [A] ROS 2 Design — “Actions”（2019，修订 2020）：https://design.ros2.org/articles/actions.html
- [B] ROS 2 Design — “Managed nodes / Node Lifecycle”（2015，修订 2021）：https://design.ros2.org/articles/node_lifecycle.html
- [C] ROS 2 Design — “ROS 2 parameter design”（参数 ownership、atomicity、限制）：https://design.ros2.org/articles/ros_parameters.html
- [E] ROS 2 Design — “Why ROS 2?”（重写动因与限定）：https://design.ros2.org/articles/why_ros2.html
- [F] ROS 2 Design — “ROS on DDS”（DDS、discovery、QoS、内进程、早期安全背景）：https://design.ros2.org/articles/ros_on_dds.html
- [G] ROS 2 Design — “ROS 2 Security Enclaves”（DDS Security identity、context、enclave）：https://design.ros2.org/articles/ros2_security_enclaves.html
- [H] rclcpp rolling source — `rclcpp/node.hpp`（publisher/subscription/service/client 真实 C++ signatures）：https://raw.githubusercontent.com/ros2/rclcpp/rolling/rclcpp/include/rclcpp/node.hpp
- [I] rclcpp_action rolling source — `create_server.hpp`（action server callback contract）：https://raw.githubusercontent.com/ros2/rclcpp/rolling/rclcpp_action/include/rclcpp_action/create_server.hpp
- [J] Nav2 docs — BT Navigator configuration / migration index（Nav2 的 BT/action/plugin 集成）：https://docs.nav2.org/configuration/packages/configuring-bt-navigator.html
- [K] BehaviorTree.CPP source — `tree_node.h`（tick、halt、ports、blackboard 并发限制）：https://raw.githubusercontent.com/BehaviorTree/BehaviorTree.CPP/master/include/behaviortree_cpp/tree_node.h
- [L] BehaviorTree.CPP source — `bt_factory.h`（XML、node factory、dynamic plugin API）：https://raw.githubusercontent.com/BehaviorTree/BehaviorTree.CPP/master/include/behaviortree_cpp/bt_factory.h

### 证据分级说明

- 标为“文档事实”的内容可直接在以上官方设计文档、官方 API header 或官方项目文档中核对。
- 标为“推断”的内容是将 ROS/BT 的取舍迁移到 Rust agent kernel 的设计建议，不是 ROS 官方承诺。
- [F] 是早期 ROS 2 DDS 设计论证，适合解释动机和边界，不应单独作为当前某 DDS vendor 行为或硬实时/安全实现保证的证据。
- [J] 的网页内容抓取受限，故本笔记只采用其可见的架构/迁移事实；未把未取得的参数默认值写成结论。
