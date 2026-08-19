TL;DR：xi 的经验表明，把高频、强耦合且仍快速演化的交互状态切成 RPC kernel 会把局部复杂度升级为全局一致性问题；对单一已知前端，优先提供 Rust 进程内 API + 可观察事件流，只在确有独立故障域、远程/多宿主或不可信扩展需求时协议化。

## 概览

- 研究对象：xi-editor 的 Rust core + 多个 GUI frontend + plugin 进程架构；以作者 Raph Levien 2020 年 retrospective 为主证。
- 对照对象：Zed/GPUI 当前公开的进程内应用状态与异步 API；用于说明同样需要响应式 UI 时，可把状态访问留在类型化内存边界。
- 重要校正：未找到可信一手材料证明「Zed 从 xi 式多进程架构转向单进程 + GPUI」。
- 因而本文不把该命题写成历史事实。
- 能核实的事实是：xi 作者明确否定了自身 core/frontend process separation；GPUI 的公开 API 则以单个 `App` 拥有实体状态、通过 Rust 句柄/Context 操作为中心。
- 这构成架构对照，不等于 Zed 是为回应 xi 而作的直接继承或“转向”。
- 资料时间：xi retrospective 发布于 2020-06-27；GPUI 仓库文档为持续更新的 main 分支，未在此把其 API 当作历史版本证据。

### xi 的初衷（文档事实）

- xi 起步时的主要动机是「用 Rust 写 GUI 应用」；core 用 Rust，GUI 可用每个平台合适/原生的技术实现。
- Levien 说其灵感部分来自 Neovim，后者把 GUI 也当成 plugin。
- 他把 `xi-core` 想成「database-like engine」。
- 因此 core 与 frontend 分成进程，以 JSON-RPC 通信；这不是仅仅的 crate 边界。
- 此结构允许 Swift/macOS、GTK、Electron 等不同 frontend 围绕同一个编辑内核运作（项目历史与协议文档均以 frontend/client 对 core 叙述）。
- 选择 JSON 的预期是每一种可行语言都会有高质量实现。
- 项目自行实现了基于 JSON-RPC 的 `xi-rpc` 层。
- 作者后来指出 Swift 的 JSON 解析「shockingly slow」。
- 他也指出 Rust `serde` 会显著膨胀代码体积。
- 这些是具体实现/性能代价；更根本的反思是接口分割引入了 async 状态协调。

### xi 的协议设计目标（文档事实）

- 官方 frontend protocol 明确把 core 视为 server、frontend 视为 client，连线是 JSON-RPC。
- 视图更新协议的目标是全程 asynchronous、最小网络流量、并让大文档的 frontend 仍可只使用少量内存。
- core 维护概念上的完整行视图：文本、光标、样式。
- frontend 维护该视图的可稀疏 cache；cache 内存在的行必须和 core 对应行一致。
- 可见区域由 frontend 的滚动编辑命令上报；core 决定并推送窗口及缓冲区所需行。
- 换言之，显示层并不单纯 render snapshot：它保存部分真相，且必须应用增量 patch 才能重新同步。
- 协议文档安全警告：该协议可加载、保存任意文件，不能暴露给 frontend 以外的 agent。
- 对 agent kernel 的类比：一旦 transport command 能调用文件/工具，其认证与 capability 不可由“只是 UI 通道”假设替代。

## 核心抽象与接口面（它暴露了哪些概念和 API，尽量给出真实签名或消息格式）

### xi：顶层 JSON-RPC 生命周期

- 创建视图是请求—响应：

```json
// frontend -> core
{"id":0,"method":"new_view","params":{}}
// core -> frontend
{"id":0,"result":"view-id-1"}
```

- `view-id` 是随后的视图命令和通知关联键。
- frontend 到 core 的顶层能力包括：`client_started`、`new_view`、`close_view`、`save`。
- 还有配置/主题/语言设置、查找替换、plugin 控制等命名空间/方法。
- `new_view` 与 `save` 在该协议版本中没有错误上报机制；这是官方文档列出的缺口。
- 同一文件当时也只支持一个 view；同样是官方列出的限制。

### xi：编辑命令是 view-scoped 的协议对象

- 协议文档的典型编辑消息为：

```json
{"method":"insert","params":{"chars":"A"},"view_id":"view-id-4"}
```

- `edit` 命名空间的命令还包含滚动、点击、选择、移动、删除/插入等编辑语义。
- `scroll` 参数为 `[first_line, last_line)`，例如 `[0,18]`。
- `click` 的旧格式为 `[line, col, modifiers, click_count]`，例如 `[42,31,0,1]`。
- 迁移中的通用 gesture 例：`{"line":42,"col":31,"ty":"toggle_sel"}`。
- 协议文档明确说 click/drag 会迁往 gesture，纯移动/选区命令也计划合并。
- 这显示：即便一个产品内部边界，交互语义一旦做成 wire contract，也会产生兼容期、双协议和弃用成本。

### xi：core -> frontend 的增量视图同步

- core 发送 `update` 通知，关键字段包括 `rev?`、`ops: Op[]`、`view-id`、`pristine`、`annotations`。
- `pristine` 表示文档是否没有未保存修改。
- `rev` 在文档所述构建中尚不存在；不能把它当成一个稳定可用的并发控制 API。
- `ops` 是对 frontend cache 的变换程序，而非完整文档快照。
- `Op.op` 的枚举为 `copy | skip | invalidate | update | ins`。
- 每项有正整数 `n`；`copy` 从旧 cache 取行，`skip` 只推进旧索引，`invalidate` 令输出行失效。
- `ins` 包含新行，且 `n == lines.length`。
- `update` op 在该协议文档中称尚未由 core 使用。
- `Line` 可含 `text?`、`ln?`、`cursor?`、`styles?`。
- `cursor` 是递增的 UTF-8 offset；`styles` 是扁平三元组（相对 span 起点增量、长度、style id）。
- 样式定义 `def_style` 包含 `id`、可选 `fg_color`/`bg_color`（ARGB）、`weight`、`italic`、`underline`。
- `measure_width` 则反向由 core 请求 frontend 计算某些字符串按指定样式渲染后的宽度，结果用于换行。
- 该反向 RPC 说明 core 的正确输出依赖 frontend 的字体/渲染事实，边界并没有消除耦合，只把它变成请求时序与缓存一致性问题。

### xi：annotations 与兼容负担

- 旧保留 style id 0（selection）和 1（find result）被标记 deprecated。
- selection 和 find match 改由 `annotations` 表示，旧字段为兼容而暂存。
- plugin command 与 Command 描述格式在文档中标为 “in flux”。
- `replace.preserve_case` 被文档标明忽略。
- 这些并非异常，而是把仍在发现期的产品语义过早冻结在跨进程公共格式后的正常维护成本。

### xi：plugin 接口与 CRDT/OT 相关抽象

- xi 的 plugin 是另一个进程边界；syntax highlighting plugin 只维护 document cache，而不是全部文档。
- 为避免编辑和异步高亮结果竞争造成错误着色，更新经 OT（operational transformation）处理。
- 作者称这个 cache protocol 最终 “quite sophisticated”。
- 项目把 CRDT 用作 concurrent modification 的机制；它来自对 Android IME 与 `EditText` 跨进程并发 bug 的泛化尝试。
- CRDT 不是本研究中建议的 agent state API；这里只记录其作为“让分布式修改一致”的先例。
- 文中未给出足够稳定、可直接复用的 xi CRDT wire API 签名，因此不编造 `Delta`/`Engine` 的外部契约。

### Zed/GPUI：公开的进程内状态接口（文档事实）

- GPUI README 将启动入口写为：`gpui_platform::application().run(|cx: &mut App| { ... })`。
- 它把 `Application`/`App` 作为桌面程序进程内的根运行时，而非独立 RPC server。
- `App` 拥有所有 entity 的数据；`Entity<T>` 是带类型标签的句柄，不直接暴露内部状态。
- 新建实体的真实示例：`let counter: Entity<Counter> = cx.new(|_cx| Counter { count: 0 });`。
- 读取/修改需提供 `App` 或具体 `Context<T>`：

```rust
counter.update(cx, |counter: &mut Counter, cx: &mut Context<Counter>| {
    counter.count += 1;
    cx.notify();
});
```

- `Entity<T>` 类似受约束的 `Rc`：clone/drop 管引用计数，但需要 context 才能访问真实状态。
- 观察式更新为 `cx.observe(&entity, callback)` 与 `cx.notify()`。
- 有类型事件的接口为 `impl EventEmitter<Event> for Model {}`、`cx.subscribe(...)`、`cx.emit(event)`。
- UI 侧：View 是实现 `Render` trait 的 `Entity`；每帧 root view 生成 element tree，再由 GPUI 布局/渲染。
- README 还说明 GPUI 既有 entity/view 的高层声明式 UI，也有 imperative `Element`；后者用于大列表与 editor custom layout。
- 这表示 Zed 没有以“所有内核状态必须序列化”为换取多 UI，而是让性能关键的 UI/模型共享类型与进程内所有权。

### Zed/GPUI：async 不是无边界共享状态

- GPUI 文档区分 `App`/`Context<T>` 与 `AsyncApp`/`AsyncWindowContext`。
- 前者是借用引用；后者通过 `to_async` 可跨 `await` 持有，具 static lifetime。
- async context 上实体操作返回 fallible，因为 window 甚至 app 可能已消亡。
- 这是一条值得借鉴的接口细节：异步句柄不应假定同步对象存活；把取消/关闭变成类型上可见的 `Result`/错误路径。
- README 也明确提供与 platform event loop 集成的 async executor。
- 这些事实支持“单进程并非同步阻塞”的结论；不支持“Zed 没有并发”或“所有工作都在 UI thread”。

## 它把什么放进核心、什么留给使用者

### xi 放入 core 的内容（文档事实）

- 文档文本与编辑状态、undo/修改、selection/cursor、查找替换。
- 将文本组织成供 frontend cache 同步的行视图，并生成 patch ops。
- 配置、主题/语言变化与部分 plugin 生命周期。
- 换行计算的决定权在 core，但字体文本宽度由 frontend `measure_width` 提供。
- syntax highlighting 的协调、style/annotation 定义及异步结果对齐。
- CRDT/OT 相关的并发修改与异步结果重定位机制。

### xi 留给 frontend/使用者的内容（文档事实）

- 原生窗口、绘制、输入事件采集、字体测量与具体 platform UI。
- frontend 要保存并正确更新行 cache、将 RPC patch 变成屏幕。
- frontend 还要按 core 请求返回文本宽度，故它是算法回路的一部分。
- plugin 本身的功能实现、进程管理与其 cache 行为不等同于 core 内置业务逻辑。
- 协议暴露的 load/save 权限暗示运行/部署者必须处理 transport 暴露与信任边界；core 文档仅作警告，不替代宿主安全策略。

### Zed/GPUI 放入框架的内容（文档事实）

- `App` 生命周期、实体所有权、窗口、subscription/event、渲染 tree、platform 服务与事件循环集成的 executor。
- 同一地址空间的共享模型通过 `Entity<T>` + `Context` 受控访问，而不是 JSON DTO/cache。
- `AsyncApp` 和 window context 将对象死亡显式化为可失败访问。

### Zed/GPUI 留给应用的内容（文档事实）

- 各 `Entity<T>` 的领域状态与业务不变量。
- UI 的 root view、`Render` 实现、actions 及 element 布局细节。
- background task 的任务语义及其错误处理；框架只提供 executor/context 基础设施。
- 不能从这些 GPUI 文档推导出 Zed 全产品（remote collaboration、language server、extension sandbox）的全部职责分割。

## 踩坑与教训

### 1. async 是复杂度乘数（作者 postmortem）

- Levien 章节标题即为 “Async is a complexity multiplier”。
- 窗口宽度连续变化会触发 rewrap；编辑动作与 word-wrap update 之间会 race。
- 即使无 race，也难避免 tearing-style artifacts：交互程序必须在中间状态显示“某些东西”。
- 滚动时 frontend 要请求 cache 外文本；作者称把这件事做对花了数月。
- 他的反事实判断是：若数据在同一进程的数据结构中，实现会很直接。
- 原文结论：“adding async made everything more complicated, in some cases considerably so.”
- 推断：agent kernel 的 event stream 若前端须增量重建 task/agent/tool 状态，必须从第一版定义 revision、snapshot/replay、断线重连和最终状态；否则只是把 xi 的行 cache 换成 agent timeline cache。

### 2. process separation 不是免费的模块化（作者 postmortem）

- Levien 的明确结论：“I now firmly believe that the process separation between front-end and core was not a good idea.”
- 他质疑“core crash 时 frontend 仍运行”的收益，认为频繁 checkpoint 更实用。
- 他认为模块化软件的许多承诺服务于 project management，而非 technical excellence。
- JSON 的跨语言便利未抵消性能、二进制体积和维护自建 RPC 的成本。
- 推断：如果我们的唯一 consumer 是同版本前端，同语言/同进程 Rust crate 通常比稳定 RPC/IDL 更小、更可演化；不要为假设中的多前端先支付协议税。

### 3. 跨边界的反向依赖会暴露“假 core”（文档事实 + 推断）

- xi core 虽负责 wrap，却需 frontend 测量宽度；它没有完全拥有决定结果所需的现实条件。
- 因而需反向 `measure_width` RPC、等待、缓存和与编辑交错的协调。
- 推断：agent kernel 不应承担依赖 UI 呈现信息才能正确作出的决策（例如 viewport、像素尺寸、local formatting）；这些应是前端的派生呈现，或作为明确可选 hint 而非 kernel 真相。

### 4. 分布式一致性工具会漏掉历史/意图语义（作者 postmortem）

- xi 引入 CRDT 是为泛化跨进程并发编辑问题。
- 但作者指出 indentation 等依赖历史的决定 “doesn't fit nicely in the CRDT model at all”。
- syntax highlighting 的异步对齐还叠加了 OT 与复杂 cache protocol。
- 推断：不要默认把 agent 协作等同 CRDT；agent 的审批、授权、工具副作用、budget、因果顺序通常是业务/安全语义，需要 append-only event log、显式 coordinator 或 domain-specific state machine，而不是“merge 就正确”。

### 5. 为一个最强用例定制 extension protocol，会限制生态（作者 postmortem）

- xi plugin 接口为 syntax highlighting 的大文件异步 cache 难题而特化。
- 作者结论：其 plugin interface “had become overly specialized”。
- 他希望形成标准 protocol，但该愿望没有实现。
- 另一方面，他认为 plugin 的进程分割仍可接受，类似 LSP。
- 推断：kernel 的 extension 边界可比 UI/core 边界更值得协议化，因为其目标包含隔离、独立发布和不可信/不同语言实现；但只应暴露稳定、窄的 capability/事件 API，不把某个内置 agent 的整个内部状态机外泄。

### 6. 过度架构会伤害贡献和交付（作者 postmortem）

- 作者认为 xi “optimized for learning rather than shipping”。
- contributor 的功能经常阻塞于大架构重构；维护者变成 gatekeeper。
- 例子是 vi keybinding project 未合并主项目。
- 作者结论：“monolithic architectures, perhaps ironically, are better for community.”
- 推断：初期将 kernel 与前端放同一 workspace/仓库，以普通 Rust types 和 feature gates 迭代；在已有两个独立 consumer、稳定性需求和真实版本偏差后，再提炼公共 crate/协议。

## 对我们 Rust agent kernel 的启示

### 建议的职责边界（推断，非 xi/Zed 原文）

- kernel 应拥有：agent/run identity、生命周期状态机、调度决策、消息/事件序列、tool invocation 的 capability check、取消、预算/配额、持久化 checkpoint/replay 边界。
- kernel 应输出：版本化的 domain events 与可重建 snapshot，而不是为某个 UI 准备的 patch tree、像素/滚动/排序结果。
- kernel 不应拥有：window、渲染、布局、输入绑定、notification 样式、具体 timeline 虚拟化策略、frontend 临时筛选/折叠状态。
- 前端应以订阅 events + 读取 snapshot 的方式派生视图；它可丢弃和重建 cache，不影响 kernel 正确性。
- UI 为流畅性可保留本地 optimistic state，但必须与 kernel 的 authority/revision 规则明确区分。
- 应用/宿主应拥有：具体 LLM provider、tool 实现、密钥、文件/网络 policy、插件安装、部署拓扑与 UI framework 选择；kernel 只判定抽象 capability，不内置平台细节。

### 最小 Rust 进程内接口面（推断性草案）

- `Kernel::start_run(StartRun) -> Result<RunId>`：启动可持久化的运行，不暴露前端 DTO。
- `Kernel::command(Command) -> Result<CommandReceipt>`：向状态机提交显式命令；receipt 至少含 command/run id 与 accepted revision。
- `Kernel::snapshot(SnapshotQuery) -> Result<KernelSnapshot>`：恢复或首次接入时获得权威状态。
- `Kernel::subscribe(Subscription) -> impl Stream<Item = KernelEvent>`：订阅有序领域事件；event 携带 `seq`/`revision`，支持从 cursor 恢复或明确返回“cursor 过期，需 snapshot”。
- `Kernel::cancel(RunId, CancelReason) -> Result<()>`：取消是一等命令而非 UI 本地 flag。
- `Kernel::register_tool(ToolDescriptor, impl ToolExecutor)` 或宿主注入 `ToolRegistry`：将 tool 执行与 kernel 调度解耦，同时把 capability grant 作为 kernel 可审计输入。
- 方法名是建议，不是已存在 API；关键是 ownership、可恢复事件流、命令确认和 capability 边界。

### 不应过早协议化的部分（推断）

- 不为当前唯一前端定义 JSON-RPC/HTTP/WebSocket 的完整 public contract。
- 不把每个 transient UI 更新做成 kernel 事件，也不让 kernel 计算 frontend 行/卡片 cache patch。
- 不把所有 agent 内部思考/流 token 误认为 durable domain state；需要按产品/隐私 policy 分层。
- 不为“未来多语言 frontend”先冻结 enum、字段与版本协商；先让 Rust types 和 integration tests 发现真正稳定的概念。
- 若日后需要远程 UI，先在进程内 API 之上做 adapter；wire schema 应从稳定的 snapshot/event/capability 语义投影，而非反过来塑造 core。

### 何时协议化边界反而是正确的（推断，受 xi plugin 结论支持）

- consumer 是独立发布、不同信任级别、不同语言或真正远程的；并且重启/版本偏差是常态，不是理论可能。
- 进程边界带来必要价值：sandbox、crash containment、权限隔离、跨机器协作或多租户。
- 数据流可用有限稳定的命令、snapshot、append-only event 表达；无需 UI 回调来完成核心正确性。
- 已有至少两个真实 consumer 证明共同语义，而不是单一 frontend 的假设。
- 可以承诺 versioning、auth、backpressure、reconnect、observability、compatibility test 的长期维护预算。
- 对外 plugin/tool host 是最接近这个条件的地方；即使如此，capability-first、超时/取消和最小 stable ABI 仍优先于“把所有 core object 都 RPC 化”。

### 可执行验收问题（推断）

- 断开并重连 frontend 后，只凭 `snapshot + events(cursor)` 能否得到与 kernel 一致的状态？
- frontend 完全不响应时，kernel 的调度、审批和恢复是否仍正确？若否，是否其实把 UI 事实错误放进 core？
- 同一状态是否在 Rust API、未来 wire adapter、测试 harness 中使用同一个 domain model，而非三份逐渐漂移的 DTO？
- tool 权限是否在 UI 被绕过时仍由 kernel/host capability enforcement 拒绝？
- 一个 async task 在 run/window/subscriber 已关闭后回写，API 是否返回明确错误/忽略陈旧结果，而非隐式写入？
- 若要新增第二种前端，真正必须跨边界的概念能否缩为 command、snapshot、event、capability，而无需镜像完整内存对象？

## 来源

### 一手/官方资料

- Raph Levien, “Xi-editor retrospective”, 2020-06-27：<https://raphlinus.github.io/xi/2020/06/27/xi-retrospective.html>
- xi-editor, “Frontend Protocol”（官方协议文档）：<https://xi-editor.io/docs/frontend-protocol.html>
- xi-editor, “Annotations RFC”（旧 selection/find style 与 annotations 的迁移背景）：<https://github.com/xi-editor/xi-editor/blob/master/rfcs/2018-11-23-annotations.md>
- xi-editor, issue #1187 “Towards a text editor construction kit”（组件化讨论；不是 CRDT postmortem，避免误引）：<https://github.com/xi-editor/xi-editor/issues/1187>
- Zed/GPUI README（`Application`、`App`、`Entity`、`Render`、executor）：<https://github.com/zed-industries/zed/blob/main/crates/gpui/README.md>
- Zed/GPUI “Contexts”（`App`/`Context` 与 `AsyncApp`/`AsyncWindowContext`、跨 await 的可失败访问）：<https://github.com/zed-industries/zed/blob/main/crates/gpui/docs/contexts.md>
- Zed/GPUI ownership/data-flow source docs（entity ownership、`update`、`observe`、`subscribe`、`emit` 示例）：<https://github.com/zed-industries/zed/blob/main/crates/gpui/src/_ownership_and_data_flow.rs>
- Zed “Zed is now open source”（可核实 GPUI 支撑 Zed 且以 Apache-2.0 发布；不含进程模型结论）：<https://zed.dev/blog/zed-is-now-open-source>

### 来源性质与使用说明

- 最强证据是 xi 作者本人 retrospective；“process separation 不是好主意”“async complexity multiplier”“plugin overly specialized”等均由此直接支持。
- frontend protocol 是设计/参考文档，适合核实消息字段、弃用项和协议目标；不应把其中 in-flux 字段误作长期稳定承诺。
- GPUI 三份资料是当前代码仓库的官方随版本文档，适合说明其公开的进程内 API 模型；不构成 Zed 历史架构变迁的 postmortem。
- 本文标为“推断”的 agent kernel 建议由上述事实类比得出，需再以 agent runtime 的实际负载、持久化和安全需求验证。
