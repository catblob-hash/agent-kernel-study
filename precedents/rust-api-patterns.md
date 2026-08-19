TL;DR：Rust agent kernel 宜采用「sans-I/O 状态机 core + 小型 runtime facade + 可序列化命令/事件 DTO」；把 socket、executor、前端线程与 UX 策略留在外层，FFI 边界不直接暴露 Rust trait/Stream/channel。

## 概览

- 研究问题：为多 agent 协作运行时抽取 Rust 依赖库时，如何既保留可替换的调度/transport，又让前端能稳定接入。
- 本文把「事实」限定为所列官方文档/源码手册明确陈述的内容。
- 标作「推断」的条目是基于这些事实对 agent kernel 的设计建议，不是来源的原话。
- 最强的先例是 Quinn 的 `quinn-proto` 与 `quinn` 两层：前者 sans-I/O，后者提供 runtime/UDP 整合。
- `h2` 是较弱但有用的对照：它和 TCP/TLS 建连解耦，但在 `client::handshake`/`server::handshake` 后接管 async transport，故不是完全纯的 sans-I/O core。
- 对公开库而言，接口不仅决定性能，也锁定了宿主的 runtime、可测试性、FFI 可达性和版本演进空间。
- 一个跨语言前端并不应成为 Rust 内部类型系统的镜像；它需要一个受限的值类型协议和明确的对象句柄语义。
- 因此建议分三层而非一个「万能 crate」：`kernel-core`、可选 `kernel-runtime`、各平台 adapter/command 层。
- 这个分层也把高频内部可泛型化的路径与低频、长期兼容的公共边界分开。

### 四类边界

- 逻辑/环境边界：状态转移、调度决策和持久化意图在 core；时钟、任务执行、网络、文件系统在 host。
- 同步/异步边界：core 用显式输入、输出和 deadline；runtime 把它驱动成 `async`/task。
- Rust/FFI 边界：FFI adapter 只传 DTO、错误、opaque handle 和平台的 async/stream 原语。
- kernel/产品边界：权限确认、窗口生命周期、渲染、国际化和用户策略是产品/前端，不是 kernel。

## 核心抽象与接口面

### 1. Quinn：严格的 sans-I/O protocol core

- 【事实】`quinn-proto` 自称为 fully deterministic QUIC protocol logic。
- 【事实】它不含 networking code，也不从 OS 取得相关 timestamp。
- 【事实】其主入口 `Endpoint` 是 per-socket protocol state；`Connection` 是单连接 protocol state and logic。
- 【事实】这使调用方拥有 socket、event loop 和 clock，适合 C bindings 或非 Tokio event loop。
- 【事实】core 不是把一个 socket 藏在对象里，而是通过普通数据类型交换外界事件和待发送 packet。
- 【事实】`Connection` 接收 `ConnectionEvent`，并输出 `EndpointEvent` 和 application-facing `Event`。
- 【事实】关键 polling API 如下：

```rust
pub fn poll_transmit(
    &mut self, now: Instant, max_datagrams: usize, buf: &mut Vec<u8>
) -> Option<Transmit>
pub fn poll_timeout(&mut self) -> Option<Instant>
pub fn poll_endpoint_events(&mut self) -> Option<EndpointEvent>
pub fn poll(&mut self) -> Option<Event>
pub fn handle_event(&mut self, event: ConnectionEvent)
pub fn handle_timeout(&mut self, now: Instant)
```

- 【事实】`poll_transmit` 交出一个 `Transmit`；宿主自行将其送至 UDP socket。
- 【事实】`poll_timeout` 返回下一个应调用 `handle_timeout` 的时刻。
- 【事实】文档要求输入的时间单调递增；`handle_timeout` 可以被虚假调用，额外调用是 no-op。
- 【事实】`max_datagrams` 约束 GSO batching 且必须至少为 1，表明包聚合策略也留给驱动者。
- 【事实】流/数据报不是全局 runtime，而是经短生命周期 view 暴露：

```rust
pub fn streams(&mut self) -> Streams<'_>
pub fn recv_stream(&mut self, id: StreamId) -> RecvStream<'_>
pub fn send_stream(&mut self, id: StreamId) -> SendStream<'_>
pub fn datagrams(&mut self) -> Datagrams<'_>
```

- 【事实】core 提供 `ConnectionIdGenerator`、`ConnectionIdParser`、`TokenLog`、`TokenStore` 等策略 trait。
- 【事实】crypto、拥塞控制也以 trait/module 形式可替换；配置用 `EndpointConfig`、`TransportConfig`、`ClientConfig`、`ServerConfig` 等值对象表达。
- 【推断】agent core 可以仿照此形状：`Kernel::apply(now, KernelInput) -> ...`，而非在 core 内直接 `spawn`、读 socket 或调用 UI callback。
- 【推断】`KernelInput` 应含 host 已完成的事实，例如 `ToolFinished`、`UserDecision`、`TimerFired`、`Persisted`，不是 `dyn Tool` 的隐式调用。
- 【推断】输出可按语义分流为 `KernelEvent`（给观察者）、`Effect`（请求 host 执行）和 `Deadline`（何时再驱动）。
- 【推断】显式 `now` 让回放测试、模拟 clock、deterministic trace 和浏览器/移动端宿主成为可行选项。

### 2. Quinn facade：core crate + runtime-facing crate

- 【事实】`quinn` 文档明确说它 build on `quinn-proto`，后者独立于任何 particular runtime。
- 【事实】`quinn` 提供 `Runtime` trait 以抽象 I/O 与 timer operations，并有 `TokioRuntime`、`AsyncStdRuntime`、`SmolRuntime`。
- 【事实】`quinn` 的公开面含 `AsyncUdpSocket`、`AsyncTimer` 及 `Accept`、`OpenBi`、`ReadDatagram` 等 future。
- 【事实】多数应用应使用 futures-based `quinn` API；`quinn-proto` 面向自定义 runtime/event-loop 嵌入者。
- 【推断】应使 `agent-kernel-core` 的 feature/dependency 尽量小且无 Tokio，`agent-kernel-runtime` 才依赖 Tokio、线程池或具体 storage transport。
- 【推断】对普通 Rust 集成者，facade 可以提供 ergonomics；对 Tauri、Node、Flutter adapter，直接驱动 core 或自行实现窄 Host trait。
- 【推断】facade 只能是 convenience layer，不能成为 core 的唯一可用入口；否则 FFI 和 deterministic test 会被 runtime 反向绑死。
- 【推断】crate 之间应单向依赖：`types` ← `core` ← `runtime` ← `tauri/napi/uniffi/frb adapter`。

### 3. h2：有价值的「不负责建连」界线，但非 pure sans-I/O

- 【事实】`h2` 是以 futures 为基础的 asynchronous API，且实现与 TCP/TLS details 解耦。
- 【事实】用户必须自行完成 ALPN 或 HTTP/1.1 upgrade；crate 不提供这些 facilities。
- 【事实】ready-for-HTTP/2 的连接才传给 `client::handshake` 或 `server::handshake`。
- 【事实】从 handshake 之后，h2 驱动该 async transport；因此其 I/O 分离弱于 `quinn-proto`。
- 【事实】交互的 per-stream handle 包括 `SendStream`、`RecvStream`、`FlowControl`、`PingPong`。
- 【推断】若 kernel 必须提供一站式 `run()`，仍应在文档中准确声明谁拥有 I/O、谁驱动 future、谁负责 shutdown；不应把「可替换 transport」误说成 sans-I/O。

### 4. trait object 与泛型：公共 API 选择的是扩展模型

- 【事实】泛型 trait bound 使用 monomorphization/static dispatch，编译器为每个具体类型生成实现。
- 【事实】`dyn Trait` 通过指向实例与方法表的 trait object 做 dynamic dispatch。
- 【事实】dynamic dispatch 有运行时查表成本，且会阻止内联及部分优化。
- 【事实】同一 `Screen<T: Draw>` 内只有一个 `T`，适合 homogeneous collection；`Vec<Box<dyn Draw>>` 可放多种运行时类型。
- 【事实】对于编译期知道的固定集合，enum 是适当方案；trait object 适合 library user 可扩展类型集合。
- 【事实】trait object 受 dyn compatibility/object-safety 规则限制；不是所有 trait 都可成为 `dyn Trait`。
- 【推断】公开的 `Kernel<Tool, Store, Clock>` 会把每一种组合传播到调用者、adapter 和 FFI wrapper，虽可高性能但不宜当主入口。
- 【推断】内部热路径可保留泛型；公开 stable API 优先使用 concrete config/DTO，或小而 object-safe 的 capability trait。
- 【推断】若选择 `dyn Host`，不要把大而易变的「全能 Host trait」稳定化；按 capability 分成例如 `EffectSink`、`Clock`、`SnapshotStore`。
- 【推断】trait 里应避免 generic method、返回 `impl Trait`、关联泛型等会使其不能 `dyn` 化的项目；必要时放到 extension trait 或 generic helper。
- 【推断】插件/工具注册若要求第三方在运行时新增实现，Rust 内部可用 `Arc<dyn ToolExecutor + Send + Sync>`；但不要把该 trait 原样跨 FFI。
- 【推断】事件的 schema 应使用 `enum KernelEvent`；相比 `dyn Event` 它能稳定序列化、生成 TS/Dart/Swift union，并支持 exhaustive UI rendering。

### 5. 事件流：channel、Stream、callback 三种不同契约

- 【事实】Tokio bounded `mpsc::channel(capacity)` 满时 send 等待 receiver，因而提供 backpressure。
- 【事实】Tokio `unbounded_channel` 的 send 总能立即完成；生产者快于消费者时内存会无界增长。
- 【事实】bounded `Sender::send` / `Receiver::recv` 是 async；同步端要用 `blocking_send` / `blocking_recv`。
- 【事实】unbounded sender 因不 await，可从同步或异步代码使用。
- 【事实】所有 sender drop 后，buffer drain 完 `recv` 返回 `None`；receiver drop 后未读消息会被 drop，后续 send 报错。
- 【事实】Tokio 建议的 clean shutdown 是 receiver `close` 拒收新消息，再 drain 至结束。
- 【事实】Tauri 文档称 `Channel<T>` 是 streaming data 的推荐机制；它适合 ordered/high-throughput chunk pushes。
- 【事实】Tauri events 不 type-safe、总是 async、不能 return value，且只支持 JSON payload。
- 【推断】core 不应把 `tokio::mpsc::Receiver<KernelEvent>` 当唯一公共事件面：它泄露 runtime、buffer 策略和 consumer ownership。
- 【推断】core 的最低层可提供 pull API：`fn poll_event(&mut self) -> Option<KernelEvent>`，或由单一 driver 调用 `drain_events`。
- 【推断】runtime facade 才可按用户选择包装为 `Stream<Item = KernelEvent>`、`broadcast`、mpsc 或 callback。
- 【推断】跨语言 UI 默认应是「命令返回一次 Promise/Future + 订阅 typed event channel/stream」；不要要求 UI 传常驻 Rust callback 作为主路径。
- 【推断】每个事件流必须写清：排序范围、容量/overflow policy、terminal event、取消语义、是否可重放、slow consumer 的行为。
- 【推断】重要状态不可只依赖 lossy push event；应提供 `get_snapshot()`/`list_runs()` 使 UI 重连后可 resync。
- 【推断】若需要 callback，限定其为 host capability（例如 approval/request credential）且要定义重入、超时和线程语义；它不等价于可广播的 telemetry。

### 6. UniFFI：语言中立值模型 + `Arc` 对象代理

- 【事实】UniFFI 内建可跨界的 Rust 类型包括 `bool`、有符号/无符号整数、`f32/f64`、`String`、`Vec<u8>`、`SystemTime`、`Duration`、`Option<T>`、`Vec<T>`、`HashMap<K,V>`、`()`, `Result<T,E>`。
- 【事实】UDL 对应 `boolean`、数值、`string`、`bytes`、`timestamp`、`duration`、nullable、`sequence<T>`、record、`void`。
- 【事实】`HashSet<K>` 在其 built-in UDL 表中为 N/A；`Vec<u8>` 的 `bytes` 与 `sequence<u8>` 在外语映射不同。
- 【事实】Python 与 Swift 转换 timestamp/duration 可能丢失精度。
- 【事实】interface object 是 live Rust struct，背后由 `Arc<>` 持有，foreign side 得到 proxy object。
- 【事实】普通 UniFFI interface 要求 `Send + Sync`，并且不允许暴露接收 `&mut self` 的方法。
- 【事实】跨界共享可写对象需使用同步的 interior mutability；文档例示 `RefCell` 非 `Sync` 不可用，`AtomicU64` 可用。
- 【事实】interface trait 可作为 `Arc<dyn Trait>` 传递；foreign-implemented trait 可用 `[Foreign]`/`with_foreign` 声明。
- 【事实】callback interface 已 soft-deprecated，文档推荐 foreign traits；旧 callback 的 Rust 参数为 `Box<dyn Trait>`。
- 【事实】UniFFI 可把 Rust `async fn`/`Future` 映射为外语 native futures，外语绑定提供 executor/event loop，不要求 Rust event loop。
- 【事实】UniFFI 没有 builtin future cancellation；文档建议自行暴露 `cancel()`/flag/channel，并映射成库错误或自定义结果。
- 【推断】UniFFI adapter 应暴露纯 DTO 方法和少量 `Arc` session/run object，不应导出 `Kernel` 的泛型参数、borrowed graph 或 Rust stream。
- 【推断】对于 agent run，采用 `start_run(request) -> RunHandle`、`RunHandle.snapshot()`、`RunHandle.cancel()`、`RunHandle.events...` 的粗粒度形状，比让前端长期持有 `&mut Kernel` 更可行。

### 7. napi-rs：Node/TS 边界允许丰富值，但 JS 生命周期是约束

- 【事实】napi-rs 可将基本数、`String`、`bool`、`Option<T>`、`Buffer`、typed array、`Vec<T>`、`#[napi(object)]` 和 class 映射到 JS/TS。
- 【事实】`Option<T>` 参数可收 `T`/`null`/`undefined`，但 `None` 返回 `null`。
- 【事实】`#[napi(object)]` 所有 field 必须 public，且从 JS object clone；Rust 侧修改不会改变原 JS object。
- 【事实】typed array 是 Reference，不 copy/clone；其 mutable access 是 unsafe，修改对 JS 可见。
- 【事实】`Vec<T>`/Array 变换 O(n)，文档称 Object/Array 的 JS round-trip 有成本。
- 【事实】BigInt 输入有精度限制；部分整数参数不直接接受，以避免由 JS BigInt 转换时丢精度。
- 【事实】导出的 `async fn` 在 napi-rs supplied Tokio runtime 上运行，TS 端返回 `Promise<T>`：

```rust
#[napi]
pub async fn read_file_async(path: String) -> Result<Buffer>
// TypeScript: (path: string) => Promise<Buffer>
```

- 【事实】async napi method 的 `&mut self` 必须标 `unsafe`，因为对象也由 Node runtime 拥有，Rust 无法保证独占。
- 【事实】`&self`、`&mut self`、`This<T>` 会在 await 期间自动转为 JS `Reference`，防止 JS GC 提前回收。
- 【事实】napi-rs 的 `ThreadsafeFunction` 是 Rust thread 调 JS callback 的专用机制；它不是普通 Rust closure 的等价替身。
- 【推断】Node adapter 应优先导出 `Promise<Dto>` 与 `AsyncIterable`/明确 event subscription，使用 `ThreadsafeFunction` 只作为内部桥接实现。
- 【推断】禁止 kernel 把 JS `Object` 或 typed array reference 存入长期状态；它们的 environment/lifetime/thread affinity 属于 N-API adapter。

### 8. Tauri：command 是 JSON RPC，不是 Rust ABI

- 【事实】`#[tauri::command]` 函数参数须实现 `serde::Deserialize`；返回值须实现 `serde::Serialize`。
- 【事实】从 JS 调 command 通常得到 resolved/rejected Promise；`Result<T,E>` 的 `Err` 拒绝 Promise，`E` 也必须可 Serialize。
- 【事实】默认参数键为 camelCase，`rename_all = "snake_case"` 可改名。
- 【事实】大 binary 可通过 `tauri::ipc::Response` 避开 JSON；原始请求也能传 `ArrayBuffer`/`Uint8Array` body。
- 【事实】async command 以 `async_runtime::spawn` 在另一 task 执行；非 async command 默认在 main thread，重工作会冻结 UI。
- 【事实】async command 目前不支持 borrowed argument（例 `&str`、`State<'_, T>`）；使用 owned type 是文档列出的 workaround。
- 【事实】Tauri 推荐 channel 用于 ordered/high-throughput streaming；events 是较弱的 JSON-only broadcast 风格机制。
- 【推断】Tauri 层应只做 `invoke` request DTO ↔ kernel command、`Channel<UiEvent>` ↔ runtime event 的转译。
- 【推断】前端权限 modal、路由、listener cleanup、notification batching 应留在前端；kernel 只能发 `ApprovalRequested` 之类的语义事件。
- 【推断】Tauri command 错误应定义 versioned tagged error DTO（如 `{kind, message, retryable}`），不直接序列化 anyhow/backtrace。

### 9. flutter_rust_bridge：最宽的类型覆盖不等于应暴露全部类型

- 【事实】FRB 把接口类型分为 translatable Dart value 与 arbitrary/opaque native handle 两类。
- 【事实】转译表覆盖 primitive、`Vec<T>`、`HashMap`、`HashSet`、array、struct、unit enum、payload enum、`Option<T>`、`Box<T>`、tuple、type alias，以及 feature 支持的 chrono/uuid/serde_json。
- 【事实】示例 `struct A { name: String, children: Vec<A> }` 在 Dart 端成为普通 `class A { String name; List<A> children; }`。
- 【事实】不便编码、不可 clone 或需要 native resource 的 Rust 类型可以成为 opaque handle；Dart 能将它作为参数/返回值并调用方法。
- 【事实】手工 opaque 例为 `RustOpaque<ArbitraryData>`；Dart 端用后应 `dispose()`。
- 【事实】trait object 可置于 opaque pointer 后，例如 `RustOpaque<Box<dyn Debug>>`。
- 【事实】Rust `fn f(sink: StreamSink<T>, ...) -> Result<()>` 生成 Dart `Stream<T> f(...)`。
- 【事实】`StreamSink` 可在 Rust 函数返回后长期持有，也可嵌入 struct/enum/vector；stream error 当前用 `anyhow::Error` 送至 Dart exception。
- 【推断】FRB 的 opaque 强大但会把长生命周期、dispose、并发和版本兼容责任带给移动端；仅将 run/session 等有身份的对象 opaque 化。
- 【推断】agent events 优先为可转译的 `KernelEvent` sealed union；opaque 内部只持有 engine state，不承载 UI 必须 inspect 的数据。
- 【推断】对长期 `StreamSink` 必须定义 run end 和 native handle disposal 的关系，避免 UI 已 dispose 而 kernel 继续无限推送。

## 它把什么放进核心、什么留给使用者

| 责任 | 先例的落点 | 对 agent kernel 的归属 |
|---|---|---|
| 状态机与不变量 | `quinn-proto::Connection` | core：run graph、agent turn、消息状态、cancel transition |
| 时间与 timeout 决策 | core 计算 deadline，host 喂 `now` | core 产出 `NextWakeup`；host 安排 timer |
| 网络/socket/ALPN | Quinn/h2 的宿主或 facade | host/runtime：模型 HTTP、tool I/O、websocket |
| task/executor | `quinn::Runtime`、napi-rs/Tauri runtime | runtime/adapter，不进入 core |
| 实际 effect 执行 | sans-I/O host 发送 `Transmit` | host：调用模型、tool、文件、credential store |
| 策略实现 | Quinn 小 trait（CID/token/crypto） | 可选窄 capability trait；避免全能 host |
| 应用事件 | Quinn `Event` polling | core 定义 typed domain event；runtime 决定分发机制 |
| backpressure 缓冲 | Tokio/Tauri channel 选型 | runtime/adapter 配置；core 可定义 priority/compaction hint |
| 持久化介质 | core 不绑 OS I/O | host 提供 snapshot/journal effect；core 定义序列化 schema |
| UI 生命周期 | Tauri/FRB/Node GC | frontend/adapter：subscribe、unlisten、dispose |
| 授权 UX | 不属于 protocol core | 前端呈现；host 将结果作为 `UserDecision` 输入 |
| FFI 类型映射 | UniFFI/napi/FRB codegen | adapter：DTO converter，不污染 core domain model |

### 可作为最小 Rust 核心的候选轮廓

```rust
pub struct Kernel { /* private deterministic state */ }

pub enum KernelInput {
    StartRun(StartRun),
    EffectCompleted { id: EffectId, result: EffectResult },
    UserDecision { request: RequestId, decision: Decision },
    TimerFired { at: Timestamp },
    CancelRun { run_id: RunId, reason: CancelReason },
}

pub enum KernelOutput {
    Event(KernelEvent),
    Effect(EffectRequest),
    WakeAt(Timestamp),
    SnapshotRequested(SnapshotIntent),
}

impl Kernel {
    pub fn apply(&mut self, now: Timestamp, input: KernelInput);
    pub fn poll_output(&mut self) -> Option<KernelOutput>;
    pub fn snapshot(&self) -> KernelSnapshot;
}
```

- 【推断】此为建议的形状，不是来自 Quinn 的可直接复制 API。
- 【推断】`EffectRequest` 应为 tagged enum，包含 stable ID、输入 DTO、deadline、idempotency/retry metadata；host 返回相同 ID 的 `EffectCompleted`。
- 【推断】`KernelEvent` 和 `EffectRequest` 必须和内部 struct 分离，才可做 serde/FFI schema 稳定化。
- 【推断】不要让 `apply` 直接返回单一 event；一次输入可连锁产生多项 effect/event，queue + poll/drain 可显式保持顺序。
- 【推断】高层 facade 可以额外提供 `async fn run_with_host(...)`，但它应只是循环 `apply/poll_output/await effect`。

## 踩坑与教训

- 【事实，UniFFI 文档】UniFFI 的 async 无内建取消；若 API 只有 `async fn run()`，前端取消按钮并不会神奇中止工作，必须另有 cancel protocol。
- 【事实，UniFFI 文档】foreign interface 可被多线程调用，故必须 `Send + Sync` 且不可暴露 `&mut self`；把单线程 actor 假设跨 FFI 导出会失败或被迫加不安全同步。
- 【事实，napi-rs 文档】await 期间 JS 值可能 GC，napi-rs 要自动持 `Reference`；这说明跨界 borrowed/reference value 不能进入 kernel 的长期状态。
- 【事实，Tauri 文档】async command 不支持 borrowed argument；这和 FFI 的 owned DTO 优先原则一致。
- 【事实，Tokio 文档】unbounded channel 没有 backpressure；把 model token/trace 直接无限广播给慢 UI 会造成内存累积。
- 【事实，Tokio 文档】receiver drop 会丢弃未读事件；所以「事件送达」不能被误当作「状态已保存」。
- 【事实，Tauri 文档】event 无返回值、非 type-safe 且 JSON-only；不能以 event 代替 request/response、ack 或 command protocol。
- 【事实，FRB 文档】`StreamSink` 可在函数返回后永久持有；若没有 terminal/dispose 语义，很容易形成 native-to-UI 泄漏。
- 【事实，FRB 文档】opaque handle 需要 Dart `dispose()`；把所有 domain state opaque 化会把资源释放与调试困难转嫁给产品层。
- 【推断】一次性把 `tokio::Runtime`、`mpsc::Receiver`、`Arc<dyn Host>`、`serde_json::Value`、FFI 宏混进一个 public `Kernel`，会让用户同时承受 runtime lock-in、所有权复杂度、schema 漂移和不可重放测试。
- 【推断】应把每条外部 effect 记录为可关联的 request ID；没有 correlation ID 时并发 tool/model 完成顺序会令复现与取消竞态不可诊断。
- 【推断】对 live token 采用有界且可合并策略（例如按 run/latest chunk coalesce），而 audit event/snapshot 走可持久化路径；不要许诺所有 telemetry lossless。
- 【推断】公开 enum 要预留演进：考虑 non-exhaustive/unknown variant 或显式 schema version；各 FFI generator 对新增 enum variant 的兼容性不同。

## 对我们 Rust agent kernel 的启示

1. 采用真正 sans-I/O 的 `kernel-core`：不依赖 Tokio、不 `spawn`、不做 HTTP/FS/IPC、不持有 UI callback，所有外部事实通过 `KernelInput` 进入。
2. core 的最小主动接口应是 command/input、`poll_output`、snapshot/recovery；时间以参数/deadline 往返，不让 core 读取系统时间。
3. core 输出分三种语义：面向 UI/audit 的 `KernelEvent`、面向 host 的 `EffectRequest`、面向 driver 的 wakeup/persistence intent，避免混为一种 channel。
4. 定义一个可序列化、版本化的 DTO 子集作为跨语言 contract；Rust 内部泛型、trait object、borrow、Tokio receiver、error source chain 不可越界。
5. 保留可选 `kernel-runtime` facade：它将 core drive 成 Rust `Stream`/async API，并选择 Tokio channel、HTTP client、task spawning；不要让其 API 反向决定 core。
6. tool/model/storage 的注入用细粒度 object-safe capability trait 或 `EffectRequest` host loop；性能关键的同 crate 模板可用泛型，稳定公开面不要泛型爆炸。
7. 前端 adapter 以各平台惯用模式翻译：Tauri command + `Channel<UiEvent>`、napi Promise + async iterator/subscription、UniFFI async + handle/cancel、FRB Future/Stream + opaque run handle。
8. 对每个 event subscription 明确 capacity、overflow、ordering、terminal、cancel、reconnect snapshot；UI 流用于体验，snapshot/journal 才是权威状态。

## 来源

### 一手文档：sans-I/O、facade 与事件

- Quinn Proto crate docs（sans-I/O/deterministic/类型索引）：https://docs.rs/quinn-proto/latest/quinn_proto/
- `quinn_proto::Connection` 方法签名与 polling 合约：https://docs.rs/quinn-proto/latest/quinn_proto/struct.Connection.html
- Quinn facade crate docs（runtime-independent proto 的明确说明）：https://docs.rs/quinn/latest/quinn/
- h2 crate docs（TCP/TLS/handshake 的责任界线）：https://docs.rs/h2/latest/h2/
- Tokio `sync::mpsc` docs（bounded/unbounded、关闭和 backpressure）：https://docs.rs/tokio/latest/tokio/sync/mpsc/
- Rust Book，trait objects 与泛型/static-dynamic dispatch：https://doc.rust-lang.org/book/ch18-02-trait-objects.html
- Rust Reference，dyn compatibility/object-safety 规则：https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility

### 一手文档：跨语言边界

- UniFFI built-in type mapping（仓库原始手册）：https://raw.githubusercontent.com/mozilla/uniffi-rs/main/docs/manual/src/types/builtin_types.md
- UniFFI interfaces/objects/traits/concurrency：https://raw.githubusercontent.com/mozilla/uniffi-rs/main/docs/manual/src/types/interfaces.md
- UniFFI async/futures/cancellation：https://raw.githubusercontent.com/mozilla/uniffi-rs/main/docs/manual/src/futures.md
- UniFFI callback interfaces（soft-deprecated）：https://raw.githubusercontent.com/mozilla/uniffi-rs/main/docs/manual/src/types/callback_interfaces.md
- napi-rs Values/type conversion：https://napi.rs/docs/concepts/values
- napi-rs async functions and lifetime rules：https://napi.rs/docs/concepts/async-fn
- Tauri v2 Calling Rust（commands/channels/events）：https://v2.tauri.app/develop/calling-rust/
- flutter_rust_bridge type overview：https://github.com/fzyzcjy/flutter_rust_bridge/blob/main/website/docs/guides/types/overview.md
- flutter_rust_bridge translatable type table：https://github.com/fzyzcjy/flutter_rust_bridge/blob/main/website/docs/guides/types/translatable/simple-correspondence.md
- flutter_rust_bridge Rust opaque handles：https://github.com/fzyzcjy/flutter_rust_bridge/blob/main/website/docs/guides/types/arbitrary/rust-opaque.md
- flutter_rust_bridge `StreamSink` → Dart `Stream`：https://github.com/fzyzcjy/flutter_rust_bridge/blob/main/website/docs/guides/types/translatable/stream.md
