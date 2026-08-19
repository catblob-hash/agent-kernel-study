TL;DR：成熟 actor runtime 的共同“内核”不是替用户定义业务 agent，而是把隔离的消息执行、可观察生命周期和明确失败边界做成机制；OTP 将重启策略标准化得最深，Orleans 进一步把位置与 activation 托管进 runtime，而 Rust 框架的差异说明调度、邮箱和监督策略必须可替换或分层，不能假定一个万能默认值。

## 概览

### 研究问题与范围

- 本笔记比较 Erlang/OTP、Akka Typed、Microsoft Orleans、Rust 的 actix 与 ractor。
- 目标不是评判哪个框架“最好”，而是识别它们把哪些职责固化为 runtime/kernel、哪些交给应用。
- 这里的“kernel”指多 agent 运行时依赖库的稳定核心，不等同于 OS kernel，也不预设分布式集群是首发目标。
- [事实] OTP 将 worker 与 supervisor 组成的 supervision tree 定义为其设计原则的基本结构。[S1]
- [事实] Akka Typed 用 `Behavior[T]` 表示 actor 的消息处理行为，并把异常后的 stop/restart/resume 放在 supervision wrapper 中。[S4]
- [事实] Orleans 的 grain 是分布式、virtual、asynchronous 的实体；grain reference 表示逻辑 identity 而非物理位置。[S6]
- [事实] actix 将 actor、typed message、handler、address 和 per-actor `Context` 暴露为主要抽象。[S7]
- [事实] ractor 把 `Actor` trait、`ActorRef`、actor 生命周期、信号及 supervision event 暴露为 Rust API。[S9]
- [推断] 五者的共同最小交集是“拥有状态的单元 + 串行/受控地处理消息 + 可寻址句柄 + 生命周期结果”。
- [推断] 位置透明、集群成员管理和 durable state 属于更强的运行时产品承诺，不应悄悄混进本地 agent kernel 的最小接口。

### 两条架构路线

- OTP/Akka 的重点是显式 process/actor 拓扑与 failure domain。
- Orleans 的重点是应用只持有 logical identity，由 runtime 按需 activation、placement、回收与故障迁移。
- actix/ractor 都采取 Rust 类型系统表达消息/状态/生命周期，但在运行时选择上明显不同。
- [推断] 这不是 Rust 缺少 actor 理论，而是 Rust 生态对 executor、线程亲和性、`Send` 边界、背压和 supervision policy 的取舍没有统一。
- [推断] agent kernel 应先确定自身是否只管理本进程中的协作单元，还是承诺“同一个 agent id 在任意节点可被唤醒”；后者才需要 Orleans 类控制面。

### OTP 为什么能以很少 behaviours 覆盖很多场景

- [事实] OTP 将常见 process pattern 拆为 OTP 提供的通用 behaviour 模块和用户实现的 callback module。[S1]
- [事实] 文档列出的核心标准 behaviours 包含 `gen_server`（client-server）、`gen_statem`（state machine）、`gen_event` 与 `supervisor`。[S1]
- [事实] `-behaviour(Behaviour)` 让编译器能对缺失 callback 产生警告。[S1]
- [事实] 服务器名称和协议被 behaviour 隐藏在客户端之后，因此可替换实现而不改调用者。[S1]
- [事实] OTP 也明确承认手写 process 可能更高效，但代价是失去通用性和统一管理能力。[S1]
- [推断] 它的关键不在于 callback 数量少，而在于每个 behaviour 对“谁拥有循环、状态、退出与清理”给出了固定协议。
- [推断] 对 agent kernel，可仿效“少数可组合 behaviour”，而非为 planner、tool、memory、reviewer 等每个 agent 角色创造内核类型。

## 核心抽象与接口面

### OTP：`gen_server` 是一个受控消息循环

- [事实] 典型启动接口为 `gen_server:start_link/4`；它在 server 初始化并可接收请求前不会返回。[S2]
- [事实] 相应的非链接启动入口是 `gen_server:start/4`。[S2]
- [事实] 同步请求用 `gen_server:call/2`，异步通知用 `gen_server:cast/2`；`cast` 立即返回 `ok`。[S2]
- [事实] `start_link/4` 接收注册方式、callback module、传给 `init/1` 的参数与 options。[S2]
- [事实] callback 的核心形状如下：

```erlang
init(Args) -> {ok, State}.
handle_call(Request, From, State) -> {reply, Reply, State1} | {stop, Reason, State1}.
handle_cast(Request, State) -> {noreply, State1} | {stop, normal, State1}.
handle_info(Info, State) -> {noreply, State1}.
terminate(Reason, State) -> ok.
code_change(OldVsn, State, Extra) -> {ok, NewState}.
```

- [事实] `handle_info/2` 接收非请求消息，例如链接进程的 `{'EXIT', Pid, Reason}`。[S2]
- [事实] `call` 会封装请求为消息、由 server reply；`cast` 不产生回复。[S2]
- [事实] 在监督树内，一般由 supervisor 而非应用自己调用 stop；需要清理时，`terminate/2` 是回调点之一。[S2]
- [推断] 真实可复用接口是“typed/untyped envelope + correlation/reply route + state transition + termination reason”，而不是某门语言的 `gen_server` 命名。
- [推断] Rust kernel 可让 `Agent` handler 返回显式 `Effect`（reply、emit、spawn、stop）或调用受限 context；二选一应保持一致，避免双重控制面。

### OTP：`supervisor` 把恢复策略声明为数据

- [事实] child spec 是 map，必要字段为 `id` 与 `start`，后者为 `{M,F,A}`，以 `apply(M,F,A)` 启动。[S3]
- [事实] child spec 可声明 `restart`、`shutdown`、`type`、`modules`，新版还可声明 `significant`。[S3]
- [事实] supervisor callback 为 `init(Args) -> {ok, {SupFlags, ChildSpecs}}`。[S3]
- [事实] `supervisor:start_link/2,3` 在所有 child 启动后才返回。[S3]
- [事实] 策略包括 `one_for_one`、`one_for_all`、`rest_for_one` 与用于同构动态 child 的 `simple_one_for_one`。[S3]
- [事实] `permanent` 总重启、`temporary` 从不重启、`transient` 只在异常退出时重启。[S3]
- [事实] `intensity`/`period` 限制时间窗内重启次数；超出后 supervisor 停止所有 children 并以 `shutdown` 退出。[S3]
- [事实] shutdown 可以是 `brutal_kill`、有限超时或 `infinity`；默认 worker 为 5000ms、supervisor 为 `infinity`。[S3]
- [事实] 正常 shutdown 时按 child 启动顺序的逆序停止；动态同构 child 的关闭顺序例外。[S3]
- [推断] `ChildSpec` 的价值是把“恢复边界、依赖关系、关闭预算”变为可检查数据，而不是散落在 catch/retry 代码。
- [推断] agent kernel 需要一个小的 `ChildSpec`/`RestartPolicy` 或同等声明对象，即使 v1 只有 one-for-one，也应保留 reason、attempt 和窗口的可观测性。

### Akka Typed：把状态机与 supervision 组合

- [事实] Typed actor 没有定义 supervision strategy 时，未处理异常默认导致该 actor stop；Classic Akka 的默认则是 restart。[S4]
- [事实] Scala 中 stateful behavior 可写为 `def counter(count: Int): Behavior[Command] = Behaviors.receiveMessage { ... }`。[S4]
- [事实] `ActorRef[T]` 是按消息类型参数化的；Scala tell 用 `ref ! message`，Java 用 `ref.tell(message)`。[S4]
- [事实] supervision 的 API 形态为 `Behaviors.supervise(behavior).onFailure[Exception](SupervisorStrategy.restart|resume|stop)`。[S4]
- [事实] restart 可用 `withLimit(maxNrOfRetries, withinTimeRange)` 限制，亦可配置是否停止 children。[S4]
- [事实] actor restart 前收 `PreRestart`，但 restart 时不会收 `PostStop`；清理逻辑通常要兼顾两者。[S4]
- [事实] 父只有 `watch(child)` 后才会在 child 异常停止时收到 `ChildFailed`/`Terminated`；未处理 `Terminated` 可令父以 `DeathPactException` 失败。[S4]
- [推断] 对 agent 运行时，默认“失败即 stop 并报告”通常比自动 restart 安全，因为 LLM/tool effect 往往不可幂等；restart 应是声明过的 opt-in。

### Orleans：logical grain reference 与 runtime-managed activation

- [事实] grain interface 方法必须返回 `Task`、`Task<TResult>` 或 `ValueTask<TResult>`。[S6]
- [事实] 示例接口把 grain identity 放在 `IGrainWithGuidKey`，实现类继承 `Grain`。[S6]

```csharp
public interface IPlayerGrain : IGrainWithGuidKey {
    Task JoinGame(IGameGrain game, CancellationToken ct = default);
}
public class PlayerGrain : Grain, IPlayerGrain { /* state + methods */ }
```

- [事实] `client.GetGrain<IPlayerGrain>(playerId)` 或 `GrainFactory.GetGrain<IPlayerGrain>(playerId)` 取得 reference。[S6]
- [事实] grain reference 封装 type + unique key 的 logical identity，独立于 physical location，并可跨完整系统重启继续有效。[S6]
- [事实] runtime 会跨 host 传播 grain 方法异常；普通方法异常通常不会销毁 activation，`InconsistentStateException` 是特殊情形。[S6]
- [事实] grain 默认 non-reentrant 且 single-threaded；允许 reentrant 后仍是单线程但 task 可 interleave。[S5]
- [事实] 文档警告 call cycle 可死锁，并以 timeout 自动打破；reentrancy 增加易错的 interleaving 风险。[S5]
- [事实] Orleans 默认 response timeout 为 30 秒，接口方法上可用 `[ResponseTimeout(...)]` 覆盖。[S6]
- [事实] grain lifecycle 有 `SetupState`、`Activate` 等有序 stage；stateful grain 的 storage load 在 `SetupState`。[S8]
- [事实] 业务可 override `OnActivateAsync`、`OnDeactivateAsync`；activation hook 抛错会使 activation 失败。[S6]
- [事实] silo crash 等异常情况下 deactivation lifecycle 未必执行，不能在 deactivation 依赖关键持久化。[S8]
- [事实] 默认 idle collection 的 `CollectionAge` 是 15 分钟；在 memory pressure 下可按活跃度 shed activation。[S8]
- [事实] silo 失败后，该 silo 上 activation 会在集群别处自动 reactivated；消息保证为 at-most-once，调用者负责必要 retry。[S5]
- [推断] Orleans 将 location/activation 管进 runtime 的前提，是它也承担 identity directory、placement、membership、serialization、failure detection 与 persistence integration 的巨大复杂度。
- [推断] 本地 Rust agent kernel 可先只给稳定 `AgentId` 和 registry trait；不要宣称 `AgentRef` 跨进程/重启有效，除非一并实现并测试上述契约。

### actix：trait + context，但投递语义是显式选择

- [事实] 任意 Rust type 只要实现 `Actor` trait 就可成为 actor；`Context<A>` 仅在 actor 执行期间可用并控制生命周期。[S7]
- [事实] 消息实现 `Message`，返回类型由 `Message::Result` 表示，处理者实现 `Handler<M>`。[S7]
- [事实] actor 通过 `Addr<A>` 而非直接引用通信，`addr.send()` 返回可 await 的 future。[S7]
- [事实] lifecycle 是 Started → Running → Stopping → Stopped；若 `Context::stop()` 触发 Stopping，未处理消息会在无法恢复时被丢弃。[S7]
- [事实] `Context::set_mailbox_capacity(&mut self, cap: usize)` 可改 mailbox capacity，默认容量是 16。[S11]
- [事实] `Addr::do_send` 即使 mailbox 满也排队，但 receiver closed 时静默丢弃。[S12]
- [事实] `Addr::try_send` 在 mailbox 满或 closed 时失败，返回 `Result<(), SendError<M>>`。[S12]
- [事实] `Addr::send` 返回 `Request<A,M>`；丢弃这个 request future 会取消消息。[S12]
- [事实] `Context::notify` 对自身投递会绕过 mailbox capacity。[S10]
- [事实] `Supervisor<A>` 要求 `A: Supervised + Actor<Context=Context<A>>`；它重启 context 并调用 `restarting()`，不会重建 actor 实例。[S10]
- [事实] actix Supervisor 不保证消息成功处理；handler 失败时该消息不能恢复，caller 得到 `Err(Cancelled)`。[S10]
- [推断] 同一个 `send` 名字不足以定义可靠性：拒绝、等待背压、绕过上限、取消即撤回是四套不同契约，kernel 必须把它们命名/类型化。

### ractor：机制较全，但策略故意留给实现者

- [事实] ractor actor 关联 `Msg`、`State`、`Arguments`，典型 callback 为：

```rust
async fn pre_start(&self, myself: ActorRef<Self::Msg>, args: Self::Arguments)
    -> Result<Self::State, ActorProcessingErr>;
async fn handle(&self, myself: ActorRef<Self::Msg>, msg: Self::Msg,
                state: &mut Self::State) -> Result<(), ActorProcessingErr>;
```

- [事实] `Actor::spawn(None, actor, args).await` 返回 `(ActorRef, JoinHandle)`；`ActorRef::send_message` 投递，`stop` 请求优雅停止。[S9]
- [事实] 消息类型至少要求 `Send + 'static`；cluster feature 下另需实现 `ractor::Message`。[S9]
- [事实] ractor 处理优先级为 `Signal::Kill`、Stop、`SupervisionEvent`、普通 messages；Kill 可中断当前执行，Stop 不会终止已在执行的 work。[S9]
- [事实] child 启动、死亡、panic 会作为 `SupervisionEvent` 通知 supervisor；但具体 supervision policy “presently left to the implementor”。[S9]
- [事实] `pre_start` panic 会直接令 spawn 失败、尚未建立 link，因而不会通知 supervisor；其他特定 lifecycle callback 的失败才会通知。[S9]
- [事实] 若构建设定 `panic = 'abort'`，panic 会终止整个程序而不进入 supervision。[S9]
- [事实] ractor 另提供 opt-in `monitors` feature，近似 Erlang process monitor 而非直接监督。[S9]
- [推断] ractor 证明“暴露 supervision event”与“提供官方默认恢复树”是两个层级；agent kernel 可先实现前者，把 policy 交给 embedding application。

## 它把什么放进核心、什么留给使用者

### 明确应由 kernel 拥有的机制

- [综合事实] 每个框架都拥有 actor 的 message ingress、执行时机与 lifecycle 状态，尽管 API 形状不同。[S2][S6][S7][S9]
- [推断] kernel 应拥有 `AgentId`、`AgentRef`、mailbox/queue、投递结果、cancel/stop、join/status 与生命周期事件。
- [推断] kernel 应拥有一个与 executor 解耦的调度抽象，例如由 runtime 注入 spawn/sleep/clock，而非将 Tokio 多线程或单线程写死为 public contract。
- [推断] kernel 应拥有有界队列及可观测的 overload 结果；容量、优先级、等待策略可配置，但“无限静默累积”不可成为默认。
- [推断] kernel 应拥有 supervision linkage/event 与 restart budget 的通用数据模型，至少记录 child、exit reason、attempt、时间窗与最终 disposition。
- [推断] kernel 应保证停止的幂等性、定义 in-flight message 的 disposition，并产生可订阅的 audit event。
- [推断] 对 LLM agent，message envelope 应带 trace/correlation id、deadline/cancellation token、发送者 identity 与明确的 payload schema/version；这是比传统 actor 更必要的观测边界。

### 应作为可选 adapter 或 feature，而非最小核心

- [推断] 远程 transport、cluster membership、node failure detection、placement 和序列化协议应是独立 distributed feature；它们是 Orleans 位置透明承诺的配套物。[S5][S6]
- [推断] durable state/checkpoint、storage provider、exactly-once 幻觉与跨重启 identity 也应独立，因为 Orleans 仍只保证 at-most-once delivery。[S5]
- [推断] work-stealing、dedicated thread、current-thread executor、priority scheduler 都应是 runtime configuration/adapter，不能用一种框架的性能选择定义 kernel。
- [推断] Pub/sub、streaming、timer/reminder、cron、workflow retry 是可组合设施；核心仅提供它们需要的 message/lifecycle primitives。

### 应由库使用者/应用定义的策略

- [事实] OTP child restart 类型和 strategy 是用户写进 child spec/supervisor callback 的声明。[S3]
- [事实] ractor 的 supervisor policy 目前由 implementor 编写。[S9]
- [推断] 应用应选择每个 agent 的 restart/stop/escalate policy，而非 kernel 因错误类型猜测“重跑 LLM”是否安全。
- [推断] 应用应定义 agent graph、角色、prompt、model/tool provider、业务 state schema、权限和费用预算。
- [推断] 应用应定义 effect 幂等键、外部副作用补偿、重试条件和人工介入；这些无从仅靠 actor crash 判断。
- [推断] 应用应决定 mailbox 配额/优先级（interactive、background、system）与 overload 的业务降级策略。

### 应由前端拥有的职责

- [推断] 前端展示 tree/graph、状态、trace、mailbox depth、重启次数和 failure reason，但不应成为调度或监督决策的唯一执行者。
- [推断] 前端可发 declarative command（spawn、stop、send、approve），由 kernel 做 authorization、状态转换和审计。
- [推断] 前端保存 layout、筛选器、局部 UI state；agent identity、生命周期事实、事件序号必须由 kernel/backend 权威保存。
- [推断] 若支持人审 tool call，kernel 应暴露 pending-effect/approval event；前端只是 presenter，不能伪造执行完成。

## 踩坑与教训

### 失败不是“自动重试”

- [事实] Orleans 的 message delivery 是 at-most-once；失败重试由 caller 负责。[S5]
- [事实] actix Supervisor 不能恢复处理失败的消息，caller 可得到 `Err(Cancelled)`。[S10]
- [推断] 不要把 restart 与 delivery guarantee 混为一谈；对有 tool side effect 的 agent，自动重启前必须有幂等/去重边界。
- [推断] kernel API 应分别报告“消息未入队”“已入队但未开始”“执行中断”“handler 返回失败”“结果丢失/未知”，避免一个 `Error` 掩盖恢复选择。

### 生命周期 cleanup 只能是 best effort

- [事实] Orleans 在 silo crash 等异常时不会保证 grain deactivation lifecycle 执行。[S8]
- [事实] `OnDeactivateAsync` 不可用于关键 state persistence。[S6]
- [事实] OTP supervisor 可以用 `brutal_kill`，该模式不保证 child 有清理机会。[S3]
- [推断] kernel 不应把“停止回调已运行”包装成 durability；关键 agent checkpoint/effect journal 必须在正常执行路径持久化。

### 背压必须成为一等语义

- [事实] actix 默认 mailbox 只有 16，且 `do_send`、`try_send`、`send` 对满队列/取消的处理不同。[S11][S12]
- [事实] `notify` 可绕过 capacity。[S10]
- [推断] 若 kernel 只提供一个无返回的 `send`，调用方无法表达“必须送达/可以丢弃/必须等待/过载失败”，最终只会积压或隐性丢任务。
- [推断] 应提供至少 `try_send`（立即拒绝）与有 deadline 的 `send`（等待容量）；任何 bypass 必须显式命名并受 metric/权限约束。

### supervisor 本身也会阻塞或形成循环

- [事实] OTP 中启动、重启、手动终止在 supervisor process 内同步执行；child 的 start/stop 应尽量短。[S3]
- [事实] OTP 文档警告 child 从树内调用 `terminate_child/2` 停自己的 supervisor 会死锁，应使用自动关闭机制。[S3]
- [事实] Orleans non-reentrant grain 的 call cycle 可能死锁，timeout 只是打断而非使协议正确。[S5]
- [推断] agent graph 不应让 supervisor handler await child 的“停止确认”同时 child 又需要 supervisor 处理消息；控制命令要有无环、可超时、可旁路的协议。

### 默认恢复选择会塑造用户心智

- [事实] Akka Typed 默认 exception → stop，Classic Akka 默认 restart。[S4]
- [事实] actix 的 restart 不重建 actor，只调用 `restarting()`。[S10]
- [事实] ractor 只传递 supervision event，policy 留给 implementor。[S9]
- [推断] `restart` 必须精确定义为“重建状态”“复用实例”“重新投递/不重投递 in-flight 消息”中的哪一种；不要用一个词掩盖语义差异。

### 动态拓扑和自动关闭有反直觉边界

- [事实] OTP 动态添加的 child 在 supervisor 重建后丢失；删除静态 child 的效果也不会跨 supervisor 重启保留。[S3]
- [事实] 自动关闭的 OTP supervisor 若作为父的 permanent child，可能造成关闭/重启循环并耗尽 restart intensity。[S3]
- [推断] agent kernel 的动态 spawn 应记录“desired topology”还是仅记录“当前实例”；两者必须由不同 API/持久化模型表达。

## 对我们 Rust agent kernel 的启示

### 建议的最小稳定接口面（推断）

- `AgentId`：稳定、可序列化的逻辑标识；v1 只保证进程内可解析，除非启用持久化/分布式 adapter。
- `AgentRef<M>`：只负责投递，不暴露内部 state 的可变引用；提供 `try_send`、deadline-aware `send`、显式 `stop`。
- `Envelope<M>`：`message_id`、`correlation_id`、`sender`、`deadline`、`cancellation`、priority 与 trace context。
- `Agent`/`Handler`：接受受限 `AgentContext` 与 envelope，返回 typed outcome；context 可请求 spawn、emit event、reply、schedule、stop，但不可任意绕过 policy。
- `MailboxPolicy`：容量、overflow（reject/wait/drop-new/drop-oldest 仅在明确允许时）、priority、公平性；投递 outcome 可观测。
- `LifecycleEvent`：created/starting/running/stopping/stopped/failed，带 reason、time、agent id、parent id、trace id。
- `SupervisionEvent` 与 `RestartPolicy`：把 child failure 与 restart budget 交给 host policy trait；v1 可提供 stop/escalate/restart-child 的少数标准策略。
- `Runtime` trait：注入 task spawning、clock/timer、cancellation、event sink；避免公开 API 锁死到 Tokio 或特定 executor。

### 刻意不放入 kernel v1（推断）

- 不内置 LLM provider、prompt graph、tool protocol、memory RAG、model retry；它们是应用业务且各自有权限/幂等语义。
- 不承诺 transparent remote `AgentRef`、cluster placement、cross-node migration；以 transport adapter 后置。
- 不承诺 exactly-once、自动恢复 tool call 或“crash 后从任意点续跑”；提供 event/effect journal hook 供应用实现。
- 不内置前端状态机、布局或审批 UI；只输出有版本且可重放的 event stream。

### 实现优先级（推断）

- P0：本进程 actor/agent、bounded mailbox、typed outcome、cancel/deadline、lifecycle event、stop/join。
- P1：parent-child link、failure propagation、restart budget、supervision policy trait、metrics/tracing。
- P2：durable desired topology、effect journal/checkpoint adapter、timer/reminder adapter。
- P3：remote transport、registry/placement、cluster membership、migration；只有产品明确需要位置透明时进入。
- 验收不应只测“能并发跑”，还应测满邮箱、sender cancel、child panic、restart storm、stop 时 in-flight、supervisor/child call cycle 和 process abort 下的可观测结果。

## 来源

- [S1, 官方设计原则：Erlang/OTP Design Principles Overview](https://www.erlang.org/doc/system/design_principles.html)（behaviour、worker/supervisor、统一管理。）
- [S2, 官方教程：Erlang `gen_server` Behaviour](https://www.erlang.org/doc/system/gen_server_concepts.html)（真实 callback/API 与 call/cast/lifecycle。）
- [S3, 官方教程：Erlang Supervisor Behaviour](https://www.erlang.org/doc/system/sup_princ.html)（child spec、策略、restart/shutdown、已知限制。）
- [S4, 官方文档：Akka Typed Fault Tolerance](https://doc.akka.io/libraries/akka-core/current/typed/fault-tolerance.html)（默认失败、supervision、watch、restart signal。）
- [S5, 官方文档：Orleans Best Practices](https://learn.microsoft.com/en-us/dotnet/orleans/resources/best-practices)（virtual actor 使用边界、non-reentrancy、at-most-once、silo failure。）
- [S6, 官方文档：Develop a grain](https://learn.microsoft.com/en-us/dotnet/orleans/grains/)（grain interfaces/references、timeouts、exceptions、activation hooks。）
- [S7, 官方文档：Actix Actors](https://actix.rs/docs/actix/actor/)（Actor/Context/Message/Handler/Addr 与 lifecycle。）
- [S8, 官方文档：Orleans Grain Lifecycle](https://learn.microsoft.com/en-us/dotnet/orleans/grains/grain-lifecycle)（ordered lifecycle、collection、memory shedding、crash 限制。）
- [S9, 官方 API：ractor crate 0.16.5](https://docs.rs/ractor/latest/ractor/)（Actor trait、signal/stop/supervision event、policy 限制。）
- [S10, 官方 API：actix `Supervisor`](https://docs.rs/actix/latest/actix/struct.Supervisor.html) 与 [`AsyncContext`](https://docs.rs/actix/latest/actix/trait.AsyncContext.html)（restart 语义、self-notify、future 停止。）
- [S11, 官方 API：actix `Context`](https://docs.rs/actix/latest/actix/struct.Context.html)（mailbox capacity 默认值和配置。）
- [S12, 官方 API：actix `Addr`](https://docs.rs/actix/latest/actix/struct.Addr.html)（`do_send`/`try_send`/`send` 的容量、取消和 closed 语义。）

### 证据等级说明

- 标为“事实”的条目直接来自上述官方文档/API；各文档版本会演进，签名以链接对应版本为准。
- 标为“推断”的条目是将这些机制映射到 Rust agent kernel 边界后的设计判断，不应误读为任一框架的官方承诺。
- 本次未找到足以支持“Rust actor 框架为什么没有一统”的单一作者 postmortem；文中对此只作基于 actix/ractor API 分歧的谨慎推断，未声称为社区共识。
