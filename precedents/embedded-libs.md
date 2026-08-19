TL;DR：四个成熟嵌入式库共同把「可复用、可验证、资源可控的执行/解析循环」留在 core，把 I/O、策略、身份、UI、业务语义经窄回调或数据对象交给 host；Rust agent kernel 应模仿这种边界而非做成全栈 agent 框架。

## 概览

- 研究对象：SQLite、libgit2、tree-sitter、Wasmtime。
- 方法：优先阅读官方 C/Rust API、官方手册和公开 API header；本笔记的「事实」均对应来源链接。
- 本笔记的「推断」是面向 Rust agent kernel 的设计建议，不是原项目的声明。
- 它们都不是 agent runtime，但都解决了同一类嵌入式内核问题：host 保持控制权，core 提供确定的机制。
- SQLite 是「事务/查询核心 + OS/VFS 与策略回调」的范例。
- libgit2 是「纯 Git 对象/传输库 + C 回调与错误码」的范例。
- tree-sitter 是「稳定的通用解析 runtime + 独立生成的 language artifact」的范例。
- Wasmtime 是最贴近目标的「受控执行 core + host imports + per-store budget / interruption」范例。
- 共同特征一：核心概念少且命名直接，不把上层工作流编码进 public API。
- 共同特征二：可扩展点是反向依赖：core 调 host callback，而非 core 自带所有外设、网络或业务策略。
- 共同特征三：资源/取消/错误不被藏进日志，均是调用方可观察和处理的协议。
- 共同特征四：兼容性重点放在边界对象、版本号和数值状态，不承诺抽象业务行为不变。
- 共同特征五：回调的生命周期、线程与 reentrancy 限制必须文档化，否则嵌入会成为未定义行为来源。

## 核心抽象与接口面

### SQLite：connection、statement、VFS 和 policy hook

- 【事实】SQLite C API 总览将 `sqlite3` database connection 与 `sqlite3_stmt` prepared statement 称为最重要的两个对象。
- 【事实】总览同时说明 API 函数很多，但大多数应用只会使用其中少数几个。
- 【事实】其 public constants 在 `sqlite3.h` 中以 `#define` 给出；函数围绕对象并使用/返回这些 constants。
- 【事实】结果码为 signed 32-bit integer；低 8 位是 primary result code，高位可携带 extended detail。
- 【事实】`SQLITE_OK`、`SQLITE_ROW`、`SQLITE_DONE` 是三个非错误结果；其它 result code 都表示 error。
- 【事实】历史兼容默认返回 primary code；可启用 extended codes 或用 `sqlite3_extended_errcode()`读取最近扩展码。
- 【事实】现有 SQLite result-code 名称与数值「fixed and unchanging」，以后可能新增 code。
- 【事实】错误详情可从 `sqlite3_errmsg()` / `sqlite3_errmsg16()` 得到。
- 【事实】`SQLITE_MISUSE` 的检测是 probabilistic；文档明确说不能依赖它来检测错误 API 用法。
- 【事实】SQLite 的 disk file format 被承诺 stable、well-defined、forward-compatible；该来源的承诺范围是文件格式而非整套 C API。
- 【推断】对 agent kernel，最值得模仿的稳定性颗粒度是 status code、事件 schema、持久化 log schema 与 capability ID，而不是承诺每个 helper 的内部行为不变。
- 【事实】VFS 是 SQLite 的 OS interface；`sqlite3_vfs` 是由应用提供并注册的结构，而非 core 直接依赖某个 OS。
- 【事实】`sqlite3_vfs` 有 `iVersion`；新增 method 时只追加字段，当前文档所述版本为 3。
- 【事实】`szOsFile` 指定 VFS 使用的 `sqlite3_file` 子类大小，`mxPathname` 指定它支持的最长 path。
- 【事实】`zName` 必须在 VFS modules 中唯一，`pAppData` 是 application-specific data 指针。
- 【事实】`pNext` 由 SQLite 维护注册 VFS 链表；文档说应用和 VFS 实现不应使用它。
- 【事实】VFS 的 `xOpen` 负责打开 file；即使失败也必须设置 `sqlite3_file.pMethods`。
- 【事实】VFS 的 v1 method 还包括 `xDelete`、`xAccess`、`xFullPathname`、动态库加载钩子、randomness、sleep、clock、last-error。
- 【事实】v2 只追加 `xCurrentTimeInt64`；v3 追加 system-call override hooks。
- 【事实】v3 的 `xSetSystemCall` / `xGetSystemCall` / `xNextSystemCall` 不被 SQLite core 使用，可用于测试模拟 fault；应用须先检查 `iVersion` 并容忍 NULL。
- 【推断】agent kernel 的宿主接口可采用「versioned trait/table + append-only optional methods」；core 只认识 `Host` capability，不认识具体前端、文件系统或模型厂商。
- 【推断】Rust 不需裸 `pAppData`，可在 `Kernel<HostState>` / `RunCtx<'_, H>` 中持有 typed state；但仍需明确谁拥有 state、何时可回调。
- 【事实】progress handler 声明为：`void sqlite3_progress_handler(sqlite3*, int, int(*)(void*), void*);`。
- 【事实】progress callback 非零返回会 interrupt 当前 operation；N 是近似 VM-instruction 间隔，N<1 或 callback NULL 会 disable。
- 【事实】每个 connection 只有一个 progress handler，新的注册会替换旧的。
- 【事实】progress callback 不得修改触发它的 database connection；`sqlite3_prepare_v2()` 和 `sqlite3_step()` 都算修改。
- 【事实】SQLite 3.41.0 起，复杂 query 的 analysis/code-generation 期间也可能从 `sqlite3_prepare()` 调用 progress handler。
- 【推断】kernel cancellation callback 必须是明确的安全点（safe point），并声明「callback 内不得 drive 同一 run / mutate 同一 session」，不能靠用户猜 reentrancy。
- 【事实】SQLite authorizer 的公开函数是 `sqlite3_set_authorizer(sqlite3*, xAuth, void*)`，其回调在 SQL operation 发生前收到 action code 和上下文。
- 【事实】authorizer 的标准 decision 是 `SQLITE_OK`（允许）、`SQLITE_DENY`（拒绝，prepare 失败）或某些 read 情形的 `SQLITE_IGNORE`（用 NULL 替代）。
- 【事实】authorizer callback 同样禁止修改其 database connection。（该条以 set_authorizer 官方页面为准。）
- 【推断】agent 的 ToolCall / delegation 不应只有「允许/抛异常」；可用 `Allow`、`Deny{reason}`、`Transform`/`Redact` 三态，且把 policy decision 留给 host。

### libgit2：handle 生命周期、negative status 与 callback payload

- 【事实】libgit2 的 `git_error_code` 以 `GIT_OK = 0` 表示成功，以 `GIT_ERROR = -1` 表示错误。
- 【事实】它还定义明确的负值类别，如 `GIT_ENOTFOUND=-3`、`GIT_EEXISTS=-4`、`GIT_EAMBIGUOUS=-5`、`GIT_EBUFS=-6`。
- 【事实】`GIT_EUSER=-7` 不由 libgit2 代码生成；callback 返回它可区分「用户 callback 造成的失败」。
- 【事实】错误的详细记录是 `git_error { char *message; int klass; }`，后者以 subsystem 分类。
- 【事实】`git_error_last(void)` 返回 library-owned 的最近错误，绝不返回 NULL，调用者不应释放它。
- 【事实】官方 header 警告：不能用 `git_error_last` 做错误检测；须先检查函数 return code，且成功调用后它可能仍是 stale information。
- 【事实】若编译时定义 `GIT_THREADS`，last error 是 per-thread；否则是单一 global instance。
- 【推断】kernel 应以每次 command/run 的 `Result` 或 terminal event 为唯一判定，不把 thread-local/global「最后错误」当 API；诊断信息必须绑定 `RunId`。
- 【事实】libgit2 remote API 使用版本化 callbacks struct（`git_remote_callbacks`）和 caller-provided payload 的 C 回调模式。
- 【事实】其 error header 特意预留 `GIT_EUSER` 来表达「回调主动终止」而不是把它伪装为 core bug。
- 【推断】Rust 中 callback 错误至少分为 `HostRejected`、`HostFailed`、`KernelFault`、`BudgetExceeded`、`Cancelled`，避免只有字符串 `Error`。
- 【推断】所有 external callbacks 都应把 `RunId`、`CallId`、deadline 与 cancellation token 显式带入，而不是只传 untyped `void *payload`。
- 【推断】libgit2 的 C 形态证明 payload escape hatch 很实用；Rust API 应以 typed context 实现同样的可嵌入性，不应为每种 UI/credential/telemetry 设计一个 kernel feature。

### tree-sitter：通用 runtime 与独立 language artifact

- 【事实】tree-sitter C runtime 的四个主要类型是 `TSLanguage`、`TSParser`、`TSTree`、`TSNode`。
- 【事实】`TSLanguage` 是定义某种语言如何 parse 的 opaque object，代码由 Tree-sitter 生成。
- 【事实】`TSParser` 是 stateful object；为它设置一个 `TSLanguage` 后，才能把 source 解析为 `TSTree`。
- 【事实】典型 grammar export 的实际声明是：`const TSLanguage *tree_sitter_json(void);`。
- 【事实】将 language 装入 parser 的 API 是 `bool ts_parser_set_language(TSParser *self, const TSLanguage *language);`。
- 【事实】basic parse API 为：`TSTree *ts_parser_parse_string(TSParser *, const TSTree *old_tree, const char *string, uint32_t length);`。
- 【事实】通用 input API 为：`TSTree *ts_parser_parse(TSParser *, const TSTree *old_tree, TSInput input);`。
- 【事实】`TSInput` 通过 `read(payload, byte_offset, position, bytes_read)` callback 使 parser 可读 rope/piece-table 等非连续 source。
- 【事实】`TSInput` 自带 input encoding 和 optional custom decode function，而 runtime 不要求 source 先复制为某个自有 string 类型。
- 【推断】agent kernel 的 event/history/input 也应接受 host-provided stream/source，而非强迫前端把所有 token、附件和 tool output materialize 成 kernel 私有格式。
- 【事实】`old_tree` 支持 incremental parsing：先 edit old tree，再 parse 并传入 old tree，得到的新 tree 与旧 tree 内部共享结构。
- 【事实】edit data 是 `TSInputEdit { start_byte, old_end_byte, new_end_byte, start_point, old_end_point, new_end_point }`。
- 【事实】真实签名为：`void ts_tree_edit(TSTree *, const TSInputEdit *);` 与 `void ts_node_edit(TSNode *, const TSInputEdit *);`。
- 【事实】如果使用者在 tree 外保存 `TSNode`，并且 edit 后继续使用它，必须单独以同一 edit 更新 node；通常应从 edited tree 重新取 node。
- 【事实】`ts_tree_copy(const TSTree *)` 只增加 atomic reference count。
- 【事实】单个 `TSTree` instance 不 thread-safe；文档要求多线程并用时先 copy tree。
- 【推断】agent kernel 的 immutable `RunSnapshot` / `EventCursor` 比可变全局 session 更适合前端并发读取；若暴露可变 handle，需复制或借用纪律。
- 【事实】runtime header 定义 `TREE_SITTER_LANGUAGE_VERSION=15` 为当前 library 支持的最新 ABI。
- 【事实】同一 header 定义 `TREE_SITTER_MIN_COMPATIBLE_LANGUAGE_VERSION=13` 为最早支持的 ABI。
- 【事实】`uint32_t ts_language_abi_version(const TSLanguage *self);` 用来验证 generated language compatibility。
- 【事实】`ts_parser_set_language` 遇到 CLI 生成版本不兼容会返回 false。
- 【事实】`TSLanguageMetadata` 的 major/minor/patch 是另一套语义版本信号，用于提示 parser release 可能与 existing queries 不兼容。
- 【推断】agent 的「角色/策略/工作流定义」应作为可版本化 artifact 与 core 分离；kernel load 时只验证 ABI/schema compatibility，不应把每种编排 DSL 编入 Rust crate。
- 【事实】grammar 用 JavaScript DSL 的 `rules` object 编写，再由 generator 生成 parser；runtime 和 grammar authoring 因此分层。
- 【事实】Tree-sitter 把 parsing 与 lexing 分两阶段，lexing 是 context-aware/on-demand；这属于 core algorithm，而某语言的 rule/precedence/conflict 属于 grammar artifact。
- 【推断】对应 agent runtime：调度、event ordering、cancellation、budget accounting 属于 core；角色 prompt、模型选择、approval policy、展示格式属于可替换 artifact/host policy。

### Wasmtime：Engine、Store、Linker、host function、limiter 与 interruption

- 【事实】Wasmtime 的 `Linker<T>` 绑定 import name 到 host definition，主要对象是 Engine、Store、Linker、Module/Component 和 host `Func`。
- 【事实】typed convenience host registration 为：`pub fn func_wrap<Params, Args>(&mut self, module: &str, name: &str, func: impl IntoFunc<T, Params, Args>) -> Result<&mut Self>`。
- 【事实】dynamic host function registration 为：`pub fn func_new(&mut self, module: &str, name: &str, ty: FuncType, func: impl Fn(Caller<'_, T>, &[Val], &mut [Val]) -> Result<()> + Send + Sync + 'static) -> Result<&mut Self>`。
- 【事实】async variant 存在但要启用 crate 的 `async` feature，且对 state 有 `Send + 'static` 要求。
- 【事实】通过 Linker 定义的 host function store-independent，同一 Engine 内可 instantiate 到多个 Store。
- 【事实】多线程时它们可能从不同 Store concurrent 调用。
- 【事实】默认不允许同 module/name/type 的重复 definition；`allow_shadowing(true)` 才让后定义优先。
- 【事实】`func_wrap` 在不允许 shadowing 的同名冲突时返回 error；type 不属于同一 Engine 时会 panic。
- 【事实】Linker 可把 unknown imports 定义为 trap 或 default values，这是一种显式的 missing-dependency policy。
- 【推断】agent kernel 的 `ToolRegistry` 应能配置 duplicate-name policy 和 missing-tool policy（fail、pause-request-host、synthetic denial），不要让 hashmap 覆盖语义暗含在实现里。
- 【事实】Wasmtime 的 host function closure 取得 `Caller<'_, T>`，因而能访问 per-store typed host state，而无需全局 singleton。
- 【推断】设计 `HostFn(Context<'_, H>, Request) -> Result<Response, HostError>`：每次调用从 run-local context 取 state，天然隔离多个前端窗口/会话。
- 【事实】`StoreLimits` 是 `ResourceLimiter` 的便利实现，通常用 `Store::limiter` 附加。
- 【事实】它限制 linear memory growth、table growth、instance count、table count 和 linear-memory count。
- 【事实】limiter 的 growth hooks 是 permission-shaped API，例如 `memory_growing(&mut self, current, desired, maximum) -> Result<bool>`。
- 【事实】limiter 还会收到 `memory_grow_failed(error)` 与 `table_grow_failed(error)` 通知。
- 【推断】agent runtime 必须区分「请求预算」「预检查拒绝」「执行时超限/失败」三件事；只提供 `max_steps: u32` 不足以支持 host 作审计和 UI 提示。
- 【事实】fuel 需先在 `Config::consume_fuel` 启用；否则 `get_fuel` 和 `set_fuel` 返回 error。
- 【事实】fuel API 是 `get_fuel(&self)->Result<u64>` 与 `set_fuel(&mut self, fuel:u64)->Result<()>`。
- 【事实】store 初始 fuel 为 0，文档说这表示 wasm 会立即 trap；fuel 全部耗尽也会 trap。
- 【事实】多数 wasm instruction 消耗 1 fuel，`nop`、`drop`、`block`、`loop` 等消耗 0。
- 【事实】async 模式可设 `fuel_async_yield_interval`，在耗费 interval fuel 时 cooperative yield；fuel 耗尽仍 trap。
- 【事实】官方文档将 fuel 作为固定区间限制的 deterministic 替代品，并与 epoch interruption 区分。
- 【推断】agent kernel 应提供 deterministic logical quota（例如 transitions、tool-cost units、model-token reservation）以便可重放测试；wall-clock 是补充而非唯一 limit。
- 【事实】epoch interruption 需编译时 `Config::epoch_interruption()` instrumentation，且某个 driver 必须调用 `Engine::increment_epoch()`。
- 【事实】epoch deadline 默认是 0；配置 epoch 后若不 `set_epoch_deadline`，store 会立即 trap。
- 【事实】`set_epoch_deadline(&mut self, ticks_beyond_current:u64)` 以当前 epoch 相对设定 deadline。
- 【事实】默认到 deadline 后，在 instrumented code 到达 epoch check 时以 trap 终止；它是 coarse-grained interruption，不是 deterministic fixed interval。
- 【事实】`epoch_deadline_callback` 接受 `FnMut(StoreContextMut<'_,T>)->Result<UpdateDeadline> + Send + Sync + 'static`。
- 【事实】callback 返回 error 会以 trap 终止 execution；async 下 `UpdateDeadline::Yield` 可在更新 deadline 前让 future yield。
- 【事实】`epoch_deadline_async_yield_and_update(delta)` 使 execution 到期 yield，恢复后把 deadline 设为 current epoch + delta。
- 【推断】agent kernel 可分成两种停止机制：可重放的 `BudgetExhausted` 与外部时钟驱动的 `DeadlineElapsed` / `Cancelled`；二者不能混为一个「timeout」。

## 它把什么放进核心、什么留给使用者

- 【事实】SQLite core 管 SQL execution、connection/statement lifecycle、result codes 与 VFS contract；具体 filesystem、clock、randomness 和某些 system-call test override 来自 VFS。
- 【事实】SQLite core 管 progress safe points；是否取消和 authorizer policy 由 application callback 决定。
- 【事实】libgit2 core 管 Git object/transport operations 与 error taxonomy；credential、network/UI 指引通过 caller callbacks/payload 进入。
- 【事实】tree-sitter core 管 parser runtime、tree structure、incremental reuse 与 ABI check；语言规则由独立 grammar 生成物提供。
- 【事实】Wasmtime core 管 instantiation/execution、import linking、per-store resources、fuel/epoch enforcement；host functions、state 和调度时钟在 embedder。
- 【推断】我们的 kernel core 应管：run state machine、event append/order、agent/tool invocation contract、capability check、budget ledger、cancellation state、snapshot/replay metadata。
- 【推断】库使用者应管：LLM provider、prompt/role/graph definition、tool implementation、认证/秘密、持久化 backend、观测 sink、租户隔离策略。
- 【推断】前端应管：呈现、输入收集、approval UI、stream backpressure UX、用户可见错误文案和 local state；前端不得成为预算/权限的唯一执行者。
- 【推断】kernel 不应管：默认人格、planner 算法、特定模型 SDK、HTTP client、数据库、浏览器 protocol、React/Vue state。
- 【推断】kernel 可暴露 `Host` trait、`Tool` trait、`Policy` trait、`Clock` trait、`EventSink` trait，但每个 trait 要小且以 data contract 互通。
- 【推断】把可能被替换的东西做 artifact/trait；把必须跨所有 embedding 一致的东西做 kernel-owned invariant。

## 踩坑与教训

- 【事实/官方文档】SQLite progress handler 被替换而非累加；若 kernel 允许多个 observer，必须定义 fan-out ownership 或提供单一 dispatcher，不能默默覆盖。
- 【事实/官方文档】SQLite progress/authorizer callback 禁止 reenter 修改同一 connection；agent 的 event hook 若能递归推进同一 run，可能破坏 state ordering。
- 【事实/官方文档】SQLite VFS xOpen 即使失败也须建立 `pMethods`；低层接口的失败路径也必须满足调用方能安全清理的结构不变量。
- 【事实/官方文档】SQLite VFS optional v3 methods 可为 NULL；版本字段和 capability probing 应一起用，不能只看版本号就无条件调用。
- 【事实/官方 header】libgit2 last error 可 stale，且不是成功/失败的真相来源；agent kernel 不应以「最近一条 error event」代替 run terminal status。
- 【事实/官方 header】libgit2 last-error 的 scope 取决于 `GIT_THREADS` build option；隐式 global/thread-local 诊断会使 embedding 行为随构建配置漂移。
- 【事实/官方文档】tree-sitter tree copy 是廉价 refcount copy，但同一 tree instance 不 thread-safe；cheap clone 不等于可共享可变对象。
- 【事实/官方 header】tree-sitter 将 runtime ABI 与 grammar semantic compatibility 分开；agent workflow schema version 也应与 kernel ABI 独立。
- 【事实/官方文档】Wasmtime Linker host function 可被多个 Store 并发调用；host callback 必须默认考虑 Send/Sync、isolation 和 shared-state contention。
- 【事实/官方文档】Wasmtime fuel 初值为 0、epoch deadline 默认 0，未正确配置会立即 trap；安全默认必须同时配套可用的 builder 默认值或启动 validation。
- 【事实/官方文档】Wasmtime fuel 是 deterministic，而 epoch 检查是 coarse-grained；绝不能承诺 cancellation 在任意 instruction/任意 tool I/O 边界即时生效。
- 【推断】将 policy callback 放在 mutation 后面会产生不可撤销副作用；工具调用至少要有「propose -> authorize -> execute -> record outcome」分界。
- 【推断】将 UI approval 当权限执行点不安全；kernel/host capability guard 必须在真正执行 tool 前二次执行。
- 【推断】callback 返回的任意字符串 error 不利于重试与前端映射；应建立 closed-ish error kind + optional diagnostic chain。

## 对我们 Rust agent kernel 的启示

- 【推断】以少数 nominal handle 作为 API 中心：`Kernel`、`Run`/`RunId`、`RunSnapshot`、`Event`、`ToolCall`、`Budget`、`Cancellation`、`Host`。
- 【推断】推荐最小启动面：`Kernel::new(config, host)`、`kernel.start(request)->RunId`、`kernel.poll/run_until_stalled`、`kernel.resume(run_id,input)`、`kernel.cancel(run_id, reason)`、`kernel.snapshot(run_id)`。
- 【推断】不要把 async executor 固化进核心：提供 poll/async adapter，executor 与 UI event loop 留给宿主，类似 Wasmtime 将 async 作为 feature/embedding mode。
- 【推断】Host callback 以 request/response enum 而非大量独立方法暴露，例：`Host::call(ctx, HostRequest)->Result<HostResponse, HostError>`；保持可版本化的 schema。
- 【推断】而 capabilities 应独立于 callback 名称，例：`ToolDescriptor { id, input_schema, required_caps, cost_hint }`；policy 在 invocation 前返回 structured decision。
- 【推断】每个回调收到 `RunContext { run_id, call_id, cancel, deadline, budget_view }`，并明确它是否可重入、是否可阻塞、是否可并行。
- 【推断】为 host function/tool callback 建立明确的 outcome：`Completed`、`Denied`、`Cancelled`、`BudgetExceeded`、`HostError`、`KernelError`；事件流与 terminal result 共享该 taxonomy。
- 【推断】预算至少分 logical steps、LLM tokens/cost、tool count、wall time；logical quota 用于确定性测试，wall time 用 clock/interrupt path。
- 【推断】将 budget 检查置于 agent transition 和 tool dispatch 的安全点；长期 tool 自身必须接收 cancel/deadline，kernel 无法抢占任意 blocking foreign code。
- 【推断】`RunSnapshot` 只读、可 clone、带 version；可变 `Run` 保持单 owner 或内部串行化，借鉴 tree-sitter 的 copy/线程限制纪律。
- 【推断】event log 是 kernel 的稳定外壳：append-only `Event { seq, run_id, kind, payload, schema_version }`；前端由 cursor 消费，而不是读取核心内部结构。
- 【推断】workflow/agent definition 作为外部、版本化 artifact；用 `supported_min/max` 做 load-time 检查，借鉴 tree-sitter language ABI。
- 【推断】trait 版本化应优先新建可选 subtrait 或 `HostRequest` enum variant，并让 kernel 能协商/报告 unsupported capability；不要频繁破坏 `Host` 主 trait。
- 【推断】无论用 `anyhow` 还是自定义 error，公开层必须提供 machine-readable `ErrorKind`；不得依赖 thread-local last error。
- 【推断】所有 Rust callback panic 都须在 FFI/task boundary 捕获并归类 `HostPanic` 或转换 failure，避免越过 kernel executor 边界。
- 【推断】启动时 validate：所选 model/tool、budget、deadline、artifact ABI、host capabilities；避免像未设 Wasmtime fuel/epoch deadline 一样运行后立刻失败。

## 来源

- SQLite C API overview（对象、常量、函数的组织）：https://www.sqlite.org/c3ref/intro.html
- SQLite result and error codes（primary/extended、稳定数值、errmsg、MISUSE）：https://www.sqlite.org/rescode.html
- SQLite VFS（`sqlite3_vfs` fields、version append、optional methods、xOpen contract）：https://sqlite.org/c3ref/vfs.html
- SQLite progress handler（签名、取消、单 handler、禁止 reentrancy）：https://sqlite.org/c3ref/progress_handler.html
- SQLite authorizer（policy callback 的 decision 语义）：https://sqlite.org/c3ref/set_authorizer.html
- SQLite architecture（disk file format forward compatibility 的明确范围）：https://www.sqlite.org/arch.html
- libgit2 `errors.h`（return codes、`git_error_last`、thread/stale caveat）：https://raw.githubusercontent.com/libgit2/libgit2/main/include/git2/errors.h
- libgit2 `remote.h`（versioned remote callback struct/payload 的 API 所在 header）：https://raw.githubusercontent.com/libgit2/libgit2/main/include/git2/remote.h
- tree-sitter getting started（core types、language assignment、parser lifecycle）：https://tree-sitter.github.io/tree-sitter/using-parsers/1-getting-started.html
- tree-sitter basic parsing（`TSInput` pull callback、parse signatures、node APIs）：https://tree-sitter.github.io/tree-sitter/using-parsers/2-basic-parsing.html
- tree-sitter advanced parsing（edit/reparse、structure sharing、tree concurrency）：https://tree-sitter.github.io/tree-sitter/using-parsers/3-advanced-parsing.html
- tree-sitter grammar writing（grammar DSL/generator 与 runtime 的分层）：https://tree-sitter.github.io/tree-sitter/creating-parsers/3-writing-the-grammar.html
- tree-sitter public API header（language ABI min/max、metadata）：https://raw.githubusercontent.com/tree-sitter/tree-sitter/master/lib/include/tree_sitter/api.h
- Wasmtime Linker API（host function、store-independence、shadowing/import policy）：https://docs.wasmtime.dev/api/wasmtime/struct.Linker.html
- Wasmtime Store API（fuel、epoch deadline、async yield/callback）：https://docs.wasmtime.dev/api/wasmtime/struct.Store.html
- Wasmtime StoreLimits API（`ResourceLimiter`、growth/account limits）：https://docs.wasmtime.dev/api/wasmtime/struct.StoreLimits.html
