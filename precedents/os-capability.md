TL;DR：先例共同把「不可伪造、可最小化授予且可在边界上拦截的能力」和极小的调用/生命周期机制放进核心；具体工具选择、授权决策、工作流与兼容层留在外面，因此 Rust agent kernel 应以 capability graph + mediation + audit 为核，而不是以 agent 框架为核。

## 概览

- 研究范围：seL4 微内核的 capability/CSpace 机制、WASI 从 Preview 1 到 Component Model / Preview 2 的接口重构、以及 OCapN 对 object-capability（ocap）引用传递的可组合性说明。
- 本文的「事实」均以各节所列原始文档为准；标注为「推断」的条目是面向 agent kernel 的设计建议，不是这些项目的原话。
- 三个先例处在不同层次，不能混同：seL4 是经形式验证的 OS kernel；WASI 是宿主—组件之间的标准 API；OCapN 是分布式 capability transfer protocol 草案。
- 但三者共享一个边界观：调用者不因“身份”或全局名字空间自动获得权力；权力来自一个明确交付、可传递的对象/导入/handle。
- seL4 的强项是把 authority 的存储、派生与回收做成小内核机制；WASI 的强项是把宿主给组件的外部世界 API 做成可组合的接口契约；OCapN 的强项是说明 authority 可以随着 object reference 跨边界传递。
- 对 agent 系统的类比应保持克制：LLM 的文本、tool schema、token、文件句柄和网络连接都不是天然等价的 capability，只有不可伪造且每次使用均经内核中介的引用才是。

## 核心抽象与接口面

### seL4：对象、capability、CSpace、invocation

**[事实｜内核抽象]**

- seL4 手册把内核描述为提供“小数目的 mechanisms”，例如 virtual address spaces、threads 和 IPC；该小体积使 integrity/confidentiality enforcement 的形式化证明成为可能。
- capability（cap）是获得内核对象服务的令牌；线程的 capability space（CSpace）由其 TCB 中 CSpace root capability 可达的 CNode 图构成。
- CNode 是 slot 表；一个 slot 至多存一个 cap。CNode cap 还能指向另一 CNode，故 namespace 是图而非单一全局表。
- cap lookup 把 CSpace 做成 guarded page table：CNode cap 携带 guard，先比较地址高位，再以 radix 位索引 slot；不匹配即 lookup failure。
- 此设计的直接含义是：内核只按“当前线程持有的 cap 路径”解析对象，而不是按进程可猜测的全局 object ID 解析对象。
- 权利（rights）是对象类型相关的细粒度位：例如 Endpoint 的 Read=receive、Write=send、Grant=可随消息发送 cap；Notification 的 Read=wait、Write=signal；page 的 Read/Write 控制映射权限。
- capability derivation tree（CDT）记录由一个 cap 派生出的子 cap。它是后续递归 revoke 的依据，而不是业务层的组织结构。
- `Mint` 能从源 cap 产生带较少 rights、不同 guard、或 badge 的 cap；`Copy` 保持 badge/guard；请求超过源 cap 的 rights 会被静默降级。
- badge 是 endpoint/notification cap 携带的不可由发送者自行选择的标识，可让服务端区分通过哪个 cap 进入；它不是通用用户身份认证。
- `Delete` 删除一个 slot 中的 cap；`Revoke` 删除目标 cap 的全部 CDT children，目标本身保留。最后一个 typed cap 删除时，对象被销毁。
- untyped memory 也是 capability：对象由 `seL4_Untyped_Retype` 从其代表的物理内存创建；这把资源创建权也纳入 capability 体系。

**[事实｜代表性 C API，名称/参数次序来自 seL4 手册的 API 描述；具体 typedef 随架构/配置而变]**

```c
seL4_Error seL4_CNode_Mint(
  seL4_CNode dest_root, seL4_Word dest_index, seL4_Uint8 dest_depth,
  seL4_CNode src_root,  seL4_Word src_index,  seL4_Uint8 src_depth,
  seL4_CapRights_t rights, seL4_CapData_t badge_or_guard);

seL4_Error seL4_CNode_Copy(/* dest CSpace/slot/depth, src CSpace/slot/depth, rights */);
seL4_Error seL4_CNode_Move(/* dest CSpace/slot/depth, src CSpace/slot/depth */);
seL4_Error seL4_CNode_Mutate(/* Move + rights reduction + cap data */);
seL4_Error seL4_CNode_Delete(seL4_CNode root, seL4_Word index, seL4_Uint8 depth);
seL4_Error seL4_CNode_Revoke(seL4_CNode root, seL4_Word index, seL4_Uint8 depth);
```

- 手册列出的 `seL4_Untyped_Retype` destination 参数包括 `root`、`node_index`、`node_depth`、`node_offset`、`num_objects`；目标连续 slot 必须全为空，否则报错。
- cap lookup failure 通过 IPC buffer 报告，错误种类包括 `seL4_InvalidRoot`、`seL4_MissingCapability`、`seL4_DepthMismatch`、`seL4_GuardMismatch`。

**[事实｜revoke 的边界与陷阱]**

- `Revoke` 并非无条件原子完成：文档明确列出两种 partial completion 情形——执行 revoke 的 TCB 在操作中被销毁，或容纳目标 cap 的存储在操作中被删除。
- revoke page table/directory 会 unmap frames，但不会 revoke frame caps；“撤销一个容器”并不自动表示撤销所有相关资源类别。
- 删除 root CNode 可能留下不可达 CNode；手册建议经覆盖它们的 untyped cap revoke 清理。这说明 authority reachability、对象存活和资源回收不是同一个谓词。

**[推断｜对机制/策略边界的解读]**

- seL4 不是替用户决定谁可访问什么；它把“持有何 cap、如何缩权派生、何时回收派生树”的可验证机制放进核，把 CSpace 布局、服务协议、进程角色与授权政策交给 userland。
- 因此它为 agent kernel 提供的不是“把 agent/plan 放入 kernel”的理由，而是“把授权图的不可伪造性和撤销原语放入 kernel”的理由。

### object-capability：reference 就是 authority，传 reference 就是 delegation

**[事实｜OCapN Model 草案]**

- OCapN 将可接收消息的 `Reference` 定义为 capability；它区分 `Target`（本地或远端 live object/presence）与 `Promise`（未来结果）。
- peer export local target、import remote target；传递 reference 即完成 grant。reference 是 first-class value，可放进 container、参数与 promise resolution，因此 delegation 可多跳传播。
- Target 满足 pass-invariant equality：传给远端再传回，仍与收到的 target 相等；这使其能作为稳定的 authority token。Promise 不保证该 equality。
- 该模型把 inert data（atom/container/error）与 authority-bearing reference 区分；container 可携带 reference，因而授权流在数据结构中是显式的。
- delivery 的结果为 fulfillment 或 rejection；草案明确仍有未定项，例如 error 内容要求很弱（文中指向 issue #142）。
- OCapN Model **没有**定义 attenuation、revocation、membrane 或 facet；这些不能被误报为该协议自带的保证。

**[事实｜极小消息模型的示意]**

```text
export(local_target) -> remote reference       # grant
send(reference, message/arguments) -> promise  # invoke
pass(reference, another_peer)                  # delegate
```

- 上面是对模型叙述的简化消息格式，不是 OCapN wire-format 的逐字编码；其要点是 capability 是 reference，而非 RPC method 名或 bearer string。

**[推断｜可组合授权与可收回的实现原则]**

- delegation 的可组合性来自“任何持有者可再传递同一 reference”，但这也意味着单纯发出裸 reference 后，原授予者无法靠删除本地变量收回远端副本。
- 要实现撤销，应交付由 kernel 控制的 forwarding facet/membrane：每次调用检查一个 shared live/revoked state；撤销改变该 state，未来调用被拒绝。已发出的结果、已开始的副作用和已泄露的数据通常不能倒带。
- attenuation 应创建只暴露操作子集、参数范围或预算的 facet，而不是把强 cap 和一段“请不要滥用”的 metadata 一起交给 agent。

### WASI Preview 1：POSIX-shaped 单模块 ABI，仍以 handle/right 管制

**[事实｜接口与 capability]**

- Preview 1 的导入模块是 `wasi_snapshot_preview1`，并要求 Memory import；参数/返回数据通过线性内存中的 pointer、record layout 与 out-parameter 交换。
- 它使用 Witx 风格的显式 size/alignment/offset ABI；例如 `fd` 是 4-byte Handle，许多调用返回 `Result<T, errno>`。
- filesystem authority 是 rights-bearing FD：base rights 用于该 FD，自身的 inheriting rights 约束由 `path_open` 创建的后代 FD。
- `fd_fdstat_set_rights` 只能移除 rights；若试图添加，返回 `errno::notcapable`。这是明确的单调 attenuation 原语。
- 目录访问锚定在 pre-opened directory FD；调用方用 `fd_prestat_get` / `fd_prestat_dir_name` 发现它们，随后 path operation 相对该 directory FD 进行，而非任意绝对环境路径。
- P1 的 FD 返回值随机化，以免应用在多线程情形依赖其编号；这是“handle 是能力，不是稳定数组索引”的实用防御。

**[事实｜真实 Preview 1 签名节选]**

```witx
fd_read(fd: fd, iovs: iovec_array) -> Result<size, errno>
fd_write(fd: fd, iovs: ciovec_array) -> Result<size, errno>
fd_seek(fd: fd, offset: filedelta, whence: whence) -> Result<filesize, errno>
fd_prestat_get(fd: fd) -> Result<prestat, errno>
path_open(fd: fd, dirflags: lookupflags, path: string, oflags: oflags,
          fs_rights_base: rights, fs_rights_inheriting: rights,
          fdflags: fdflags) -> Result<fd, errno>
poll_oneoff(in: ConstPointer<subscription>, out: Pointer<event>,
            nsubscriptions: size) -> Result<size, errno>
random_get(buf: Pointer<u8>, buf_len: size) -> Result<(), errno>
proc_exit(rval: exitcode)
```

- P1 同时保留大量 POSIX-shaped surface 与 errno；文档承认有些 errno 仅为对齐 POSIX 而不会被本 API 返回。
- P1 socket surface 很小（如 `sock_accept`/`sock_recv`/`sock_send`/`sock_shutdown`），没有 `connect`/`bind`/`listen`/address types；这显示“兼容 familiar API”不等于完整复刻 host OS。

### WASI Preview 2 / Component Model：从单 ABI 改为显式契约组合

**[事实｜演进]**

- Preview 2（WASI 0.2 / WASIp2）明确称自己“fully rebased on the Wit IDL and the component model”，目标是 modular、fully virtualizable，且让多种源语言可访问。
- 0.2.0 发布于 2024-01-25；Component Model 文档将其称为一组 stable WIT definitions。当前 WASI release process 后续按数字版本发展，勿把 Preview 2 当作永久最终接口。
- P2 包含独立 package/interface 组：`wasi-io`、`wasi-clocks`、`wasi-random`、`wasi-filesystem`、`wasi-sockets`、`wasi-cli`、`wasi-http`。
- 0.2 P2 README 称 Witx 可按 Canonical ABI 从 Wit 导出，使 engine 可利用既有 Witx machinery 实现 single-module components；这是桥接策略的一部分，但该 README 没有承诺 P1 API 的逐调用无损迁移。
- P3 的计划 async `future` / `stream` 将替换 P2 `wasi-io` streams/polling；README 允许实现通过同时支持二者，或把 P2 virtualize/polyfill 在 P3 上。接口演进的稳定手段是 adapter/interposer，而不是让 kernel 永不改变。

**[事实｜WIT 抽象与真实语法]**

- WIT 只定义 contract，不定义 behavior；`interface` 是具名 types + functions 集合，`world` 是组件的 imports/exports contract。
- import/export 在 interface/function 层，不在 package 层；`include other-world;` 可合成其他 world 的 import/export。

```wit
interface tools {
  record tool-result { text: string, truncated: bool }
  invoke: func(name: string, input: list<u8>) -> result<tool-result, string>;
}

world agent-worker {
  import tools;
  import audit: func(event: string);
  export run: func(task: string) -> result<string, string>;
}
```

- 上述 `tools` / `agent-worker` 是本文面向 agent 的示例；`interface` / `world` / `func` 语法与 import/export 规则是真实 WIT。
- WIT resource 是 component 外部存在、不可随意复制实体的 handle；行为只能经其 method 暴露。

```wit
resource blob {
  constructor(init: list<u8>);
  write: func(bytes: list<u8>);
  read: func(n: u32) -> list<u8>;
  merge: static func(lhs: blob, rhs: blob) -> blob;
}
```

- resource method 有隐式 `self`；constructor 至多一个；`borrow<blob>` 是调用期临时借用，而 owned handle drop 时销毁 resource。这个所有权接口是 Rust capability API 很有价值的参照。
- WASI design principles 明确分两类 capability：runtime handle（first-class、unforgeable、可在运行期传递）和 link-time capability（import 的函数/instance）。
- 对一时间需要多个独立 resource 的场景应使用 runtime handle；若通常只有一个（例：clock），可使用 link-time capability。
- WASI 声称没有 ambient authority：没有 runtime global namespaces，也没有 link-time global functions；link-time capabilities 可 interpose，替换组件可 virtualize/attenuate 底层功能。
- WASI 明确将 compatibility concerns 尽量留在 library/tool（如 WASI libc），避免给不需要兼容的应用施加负担；worlds 使不同 host 可暴露不同 API 集。

## 它把什么放进核心、什么留给使用者

### 先例的核心内职责

| 先例 | 放进核心/标准边界的机制 | 没有放进核心的政策 |
|---|---|---|
| seL4 | object、cap slot/lookup、rights、派生树、mint/copy/delete/revoke、IPC 与调度等基础 mechanism | CSpace 的业务布局、服务发现协议、谁该拿何 cap、应用功能与组织政策 |
| OCapN Model | reference/promise、export/import、跨 peer 传递、identity/invariant | attenuation/revocation/membrane 策略、error 语义细节、对象业务协议 |
| WASI P1 | FD/right、preopen root、相对路径和 ABI | host 给哪些 preopen、哪些网络/环境能力、应用的业务授权 |
| WASI P2/Component Model | WIT contract、world import/export、resource ownership、interposition 的组成能力 | host 实现哪些 interface、把哪些 capability 链给哪个 component、compatibility adapter |

### 对 agent kernel 的职责划分（推断）

- **kernel 必须管**：opaque `CapabilityId` 的签发/验证；parent→child delegation edge；rights/constraint 的只减不增检查；每次 tool/resource invocation 的强制中介；撤销 epoch 或 membrane；调用/拒绝/撤销的 append-only audit event；资源对象的生命周期/取消。
- **kernel 必须管**：不能让模型产生的 `tool_name`、JSON 参数、agent label 或前端传来的 string 本身成为授权凭据。它们最多是 routing hint，最终须以 cap resolve。
- **kernel 可暴露而不替使用者决定**：创建 agent scope、mint attenuated child cap、pass/borrow cap、invoke、revoke subtree、查询审计游标、注册/注销受控 resource provider。
- **库使用者/后端管**：真实 policy（谁可创建何种 agent、审批条件、预算上限、tenant isolation、工具实现、密钥保管、持久化、审计保留期与告警）。
- **前端管**：意图收集、capability 请求 UI、用户确认/审批、可视化 delegation/audit、展示 cancellation；前端不保存 master cap 或自行裁定授权。
- **agent/workflow 层管**：task decomposition、prompt、角色、agent 之间消息语义、重试、fallback、计划与业务状态机；它们调用 kernel 的授权机制，不能绕开该机制。
- **兼容层管**：把 legacy tool registry、MCP/HTTP/CLI 或宽泛权限模型包装为 provider/interface；不要为了兼容把所有工具都塞入 kernel API。

### 建议的最小 Rust 接口面（推断，不是来源 API）

```rust
pub trait CapabilityKernel {
    fn delegate(&self, parent: CapabilityId, attenuation: Attenuation,
                subject: SubjectId, ttl: Option<Deadline>) -> Result<CapabilityId, Denied>;
    fn invoke(&self, cap: CapabilityId, request: Invocation) -> Result<InvocationId, Denied>;
    fn revoke(&self, root: CapabilityId) -> Result<RevocationReceipt, KernelError>;
    fn cancel(&self, invocation: InvocationId) -> Result<(), KernelError>;
    fn audit_since(&self, cursor: AuditCursor) -> impl Iterator<Item = AuditEvent>;
}
```

- `CapabilityId` 应是不可从名称/参数伪造的 opaque handle；仅序列化为 bearer token 会失去“本地不可伪造 reference”的性质，若必须跨进程则要有签名、受众绑定、过期、revocation check 和泄漏模型。
- `Attenuation` 建议表达 operation set、resource selector、参数约束、最大调用数/字节数、deadline、delegable 与否；新增权力应一律拒绝，不做 P1 风格 silent downgrade，除非返回最终实际 granted scope。
- `Invocation` 应把 `(cap, operation, typed input)` 作为原子授权点；不要提供先 `authorize(tool)` 后任意次数裸 `tool.execute()` 的 TOCTOU 旁路。
- `revoke` 的语义须写清：对尚未开始、执行中、已产生外部副作用、已转交数据的不同效果；仅宣称“future kernel-mediated calls fail”是可实现且诚实的基线。
- provider 的接口宜类似 WIT `world`：按窄 capability group 拆分（filesystem read、filesystem write、HTTP to allowlist、secret use、model call），由 embedding 选择 imports；避免一个 `Tool::execute(any JSON)` 成为永久万能洞。

## 踩坑与教训

### 1. “capability”不等于一串权限字段

- **事实**：OCapN 的 capability 是可收消息的 reference；WASI 的 runtime capability 是 unforgeable handle；seL4 cap 位于 CSpace slot 并经内核 lookup。
- **教训/推断**：若 agent 平台只把 `{agent_id, allowed_tools}` 放在 prompt、前端 state 或可猜 ID 中，恶意 prompt/agent 可构造等价请求；必须使 resource provider 只接受 kernel 验证后的 invocation context。

### 2. attenuation 是单调的，扩权必须回到上级

- **事实**：WASI P1 `fd_fdstat_set_rights` 只能移除 rights；seL4 Mint 从现有 cap 派生且 rights 不得超过源 cap。
- **教训/推断**：child agent 不应能请求 kernel 将其 cap 合并、补齐为 parent 的 scope；扩权应形成一条新、可审计、可能需要 human approval 的 delegation edge。

### 3. revoke 不是时间机器

- **事实**：seL4 文档记录 revoke 可 partial complete，并指出 revoke page tables 不会 revoke frame caps；OCapN Model 并不提供 revocation。
- **教训/推断**：不可写“revoke 保证消除所有影响”。须明确 resource class：中断未开始调用、取消可取消的 future、关闭 stream、拒绝后续请求、或只记录已不可逆副作用。复杂 provider 需要 idempotency/cancellation contract。

### 4. 授权图、资源生命周期、可达性应分别建模

- **事实**：seL4 删除最后一个 typed cap 才销毁对象；删除 root CNode 可能留不可达 CNode；untyped revoke 才能覆盖某些清理。
- **教训/推断**：Rust kernel 不要把“撤销 cap”“drop provider resource”“删除 agent state”“清空 UI 可见性”合成一个含糊 API；分别记录状态、保证顺序并返回 per-step outcome。

### 5. 静态 import 与动态 handle 适用面不同

- **事实**：WASI 规定通常单例资源（如 clock）可用 link-time capability；多并存资源应使用 runtime handles；component interface 可 interpose。
- **教训/推断**：agent kernel 可把固定的 `AuditSink` / `PolicyOracle` 设计为 embedding-time trait/import，把每个 secret、browser tab、repo checkout、transaction、delegated tool scope 设计为动态 resource cap。

### 6. API 稳定来自契约/adapter，不来自把一切冻结进内核

- **事实**：P2 以 WIT/world 重构 P1 风格 ABI；P3 计划通过同时支持或 P2-on-P3 polyfill 延续兼容；WASI 将 compatibility 尽量留在 library/tool。
- **教训/推断**：稳定 Rust kernel 的少量 capability/audit/lifecycle trait；agent SDK、tool schema、模型供应商和 legacy compatibility 都置于适配层。为 provider 版本协商 typed contracts，而非 kernel 匹配任意 JSON。

## 对我们 Rust agent kernel 的启示

- 核心判据不是“代码是否通用”，而是“若该机制不在 kernel，任何一个使用者能否绕过它取得 tool/resource authority”。能被绕过的授权检查不应只在前端或 workflow 层。
- 把内核设计成 reference monitor：所有外部副作用须经 `invoke(cap, operation, input)`；cap 已撤销/过期/预算耗尽/operation 不匹配时，在 provider 前失败并审计。
- authority 的 data model 至少需要：cap identity、subject/tenant、resource kind+locator、allowed operations、constraints、parent/delegation edge、creation/expiry、revocation state、delegation flag、budget，以及审计关联 ID。
- 采用 “parent retains root, child receives facet” 而非把 master tool cap 直接复制给 child；这样 `revoke(parent-child subtree)` 有明确对象。跨 host/进程时应在每次调用强制校验，而不能依赖 Rust 内存封装。
- `badge` 的经验可转为不可由 agent 自报的 `SubjectId`/`DelegationId`：由 kernel 填入 audit/provider context，供配额、归因、策略分流使用；不要把它误作可自由填写的 prompt tag。
- 为 privilege attenuation 提供强类型 builder，并返回 `GrantedScope`；以免调用者误以为获得了请求的宽权限（seL4 的 silent rights downgrade 在 agent UX/安全审计中容易造成混乱）。
- capability 能力边界应是 resource-oriented，不是 agent-oriented：同一个 agent 可以持多个互不等价的 read-repo、write-branch、send-email、model-call cap；agent ID 只用于策略与归因。
- 简化首版：单进程 opaque handle + kernel-side map + subtree revoke + structured audit，可先不支持可转移 bearer token；跨进程 delegation 是单独安全设计，不要由 JSON 序列化“自然获得”。
- 对前端仅下发描述性 `CapabilityView`（label、scope 摘要、状态、是否可转授），从不下发可行使 master handle；UI 的“批准”应变成后端/kernel 的一次可审计 `delegate` 调用。
- 定义 4 个可测试不变量：无 cap 不可调用；child scope 是 parent scope 子集；revoke 后任何未来经 kernel 的调用均失败；每个 allow/deny/revoke 都可归因到 subject 与 delegation edge。

## 来源

1. seL4 Manual，Introduction（内核最小 mechanisms、formal verification）：https://github.com/seL4/seL4/blob/master/manual/parts/intro.tex
2. seL4 Manual，Capability Spaces（CNode、guard、rights、CDT、Mint/Copy/Revoke 与 partial revoke caveat）：https://github.com/seL4/seL4/blob/master/manual/parts/cspace.tex
3. seL4 公共 API 头文件（用于核对 C syscall wrapper 名称）：https://github.com/seL4/seL4/blob/master/libsel4/include/sel4/functions.h
4. WebAssembly/WASI，Design Principles（capability、interposition、compatibility、portability、modularity）：https://github.com/WebAssembly/WASI/blob/main/docs/DesignPrinciples.md
5. WebAssembly/WASI，Capabilities（handle vs link-time capability、无 ambient authority、attenuation/interposition）：https://github.com/WebAssembly/WASI/blob/main/docs/Capabilities.md
6. WASI Preview 1 snapshot spec（`wasi_snapshot_preview1`、FD rights、preopen、真实签名）：https://github.com/WebAssembly/WASI/blob/v0.2.0/legacy/preview1/docs.md
7. WASI Preview 2 README（WIT/Component Model rebase、0.2 packages、P3 polyfill 方向）：https://github.com/WebAssembly/WASI/blob/v0.2.0/preview2/README.md
8. Bytecode Alliance Component Model docs，WIT（interface/world/import/export/resource/borrow 的语法与含义）：https://component-model.bytecodealliance.org/design/wit.html
9. Bytecode Alliance Component Model introduction（platform supplies capabilities、WASI 0.2.0 stable WIT context）：https://component-model.bytecodealliance.org/
10. OCapN draft Model（reference capability、target/promise、pass-invariant equality、delegation；草案/社区协议而非 seL4 级证明）：https://github.com/ocapn/ocapn/blob/main/draft-specifications/Model.md
11. WASI Release Process（数字版本与 release train；用于避免把 P2 误作停滞接口）：https://github.com/WebAssembly/WASI/blob/main/docs/Release.md

- 访问说明：以上 GitHub 文档以仓库主分支或标明的 `v0.2.0` tag 为准；P1/P2 对照特意钉在 `v0.2.0`，避免主分支后续重组文档造成历史表述漂移。
