# Rust Agent Kernel 最小接口面提案（v2）

> 状态：经 12 路领域对抗走查修订；研究日期：2026-08-18。  
> 目标场景：同一进程、同一版本、单一前端；kernel 作为 Rust 依赖库嵌入可信宿主。  
> 判定依据：[规范税系](./taxonomy.md)、[反模式清单](./anti-patterns.md)、[kernel/frontend 先例](../precedents/kernel-frontend-split.md)、[xi 与编辑器先例](../precedents/xi-and-editors.md)，以及本文件末尾的逐条走查裁决。

## 0. 一句话裁决

稳定窄腰不是 `Agent/Crew/Graph/Tool/UI`，而是：**一个单 owner、可持久恢复的运行事实机；宿主用带显式时间与权限的幂等命令驱动它，用显式 effect 回调连接外界，用足以恢复的 snapshot + 有序 event 观察它。**

仍只提供五个宿主操作：`open / submit / drive / snapshot / subscribe`；仍只要求三个宿主端口：`Store / Policy / Effects`。修订没有加入 workflow DSL、队列、世界模型、分布式共识或 UI 协议；它只补齐反复出现的事实闭环：提交时间、恢复投影、effect 结果与重试、可审计授权、原子租约集合、租约—effect fencing、消息相关性，以及 control work lane。

---

## 1. 概念模型与责任边界

### 1.1 核心名词（仍为 10 个）

| 名词 | kernel 拥有的最小含义 | 明确留给库使用者/前端 |
|---|---|---|
| **Kernel** | 单一线性化点；校验命令、推进事实、原子持久化后发事件 | 线程、进程、HTTP、分片、跨站共识 |
| **Run** | 稳定 ID、父子血缘、原始 `RunSpec`、预算、deadline、终态结果 | role、persona、业务 step 类型系统、调度算法 |
| **Command** | 幂等意图，含 ID、显式提交时间、actor、authority、causation、CAS | UI click、自然语言自述、身份认证协议 |
| **Observation** | 可恢复 `Snapshot` 与同一事实模型的有序 `Event` | UI cache、日志、token 流、历史分析仓库 |
| **Effect** | propose→authorize→claim/attempt→dispatch/record→receipt 的外界事实账本 | HTTP/MCP/LLM/shell 实现、领域 retry 分类、exactly-once 承诺 |
| **Wait** | 可恢复 join/timer/message/input，保留条件、请求和响应 | 文案、本地化、业务 quorum 规则 |
| **Capability** | 可缩权、可撤销、可限定命令类与 effect scope 的未来动作凭据 | 企业 IAM、登录、签名验证、恶意宿主隔离 |
| **Budget** | Run 树硬上限、通用 meter 预留与结算 | 模型选择、优化目标、账单系统 |
| **Lease** | 原子资源集合、时间窗、共享/独占、容量、续租与资源 fence | 冲突几何、设备健康、业务日历生成 |
| **Artifact** | immutable ref、workspace revision/CAS、producer/input provenance | blob store、Git、向量库、领域 schema、自动 merge |

`Message` 仍是有界、持久 coordination fact，不是第 11 个 manager。`ResourceDefinition` 只给通用容量仲裁提供事实，不是资产目录。`WorkClass` 只给推进隔离，不是领域调度器。

### 1.2 三层责任

**kernel 必须管：**

1. 单 Kernel 内 command/state/event 原子线性化、ID 幂等、cursor/revision；
2. Run/Wait/Effect/Capability/Lease/Artifact 的完整恢复投影；
3. effect attempt、receipt、Unknown、显式 retry/reconcile；
4. capability 的原子核验、actor/响应/政策证据审计；
5. 资源集合的全有或全无授予，以及 effect 对资源 fence 的绑定；
6. bounded drive、control/interactive/background 工作类和硬预算。

**库使用者必须管：**

- handler registry 与 `RunSpec.kind` dispatch；领域状态、世界模型、freshness、空间冲突、优化器、队列、SLA 日历；
- Store/Policy/Effects、身份认证、blob、数据库事务、外部 job callback；
- retry 分类、补偿、审批 quorum、广播/ack 协议、跨 Kernel saga；
- 多线程 actor facade、进程隔离、分片、升级 drain/migrator、紧急物理保护。

**前端只管：**

- 从 Snapshot/Event 投影界面；按 `InputRequest` 渲染表单；经宿主提交 Respond/Cancel；
- token/progress 等可丢 UX 流；不得把 UI cache、stream close 或按钮状态当权威事实。

### 1.3 关系图

```mermaid
flowchart LR
  H[宿主] -->|submit(now, Command)| K[Kernel]
  K -->|state + events atomic CAS| S[(Store)]
  S -->|recover / replay| K
  K --> R[Run tree]
  R --> W[Wait ledger]
  R --> E[Effect ledger + receipt]
  R --> L[Lease sets + resource fences]
  R --> A[Artifacts + provenance]
  C[Capabilities] --> R
  L -->|required lease fences| E
  E --> P[Policy]
  P --> E
  E --> X[Effects]
  X --> E
  K --> O[Complete Snapshot + ordered Event]
  O --> F[Frontend]
  F --> H
```

---

## 2. 为什么是进程内 Rust 库，而不是先冻结协议

xi 为跨语言 frontend 把 core 分进程并自建 JSON-RPC，最终暴露了序列化、双 cache、race/tearing 和反向 RPC 的高税；作者明确认为该分离不是好选择：[xi retrospective](https://raphlinus.github.io/xi/2020/06/27/xi-retrospective.html)、[xi frontend protocol](https://xi-editor.io/docs/frontend-protocol.html)。当前没有“多个独立发布 consumer × 多语言 server”的 N×M 问题，因此 core 不拥有 socket、Tokio、认证或 wire compatibility。

Jupyter 值得借的是不同交互时程的语义分层，而不是五个 socket：控制、stdin、请求/回复、广播与 heartbeat 不能退化为一个 send；`idle` 也不是 terminal：[Jupyter messaging](https://jupyter-client.readthedocs.io/en/latest/messaging.html)。v2 因此给 control work 独立推进类、给人机交互 durable Wait、给 Run 唯一 Terminal。

LSP 值得借的是稳定 ID、取消后仍闭合原请求、capability negotiation 与 additive evolution，而不是直接套 JSON-RPC：[LSP 3.17](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/)。未来只有在真实出现独立版本 consumer、跨语言、跨机器或不可信插件隔离时才加 `kernel-protocol` adapter。

本提案不宣称跨 Kernel 顺序、离线 fork 合并、跨分片 fencing 或硬实时 deadline。需要这些能力时，部署边界已不再是“同进程 Rust 依赖库”，应引入共识/事务型控制平面，而不是把 CRDT 或 placement API偷渡进 core。

---

## 3. v2 稳定公共 API 全集

### 3.1 总体约束

- 以下代码块共同构成完整稳定 public surface；没有额外 manager/registry/provider state。
- DTO 全部 owned，`Payload.bytes` 是宿主 schema 的 canonical bytes。
- `Kernel`/callback future 仍 local-first、非 `Send`；多线程宿主用专用 owner task/thread + channel facade。
- 同一 Kernel 的事实提交串行；外部 callback 期间不持有可变提交借用。多个 `drive` future 可并存，但每个 work item 先 durable claim，且只可推进请求的 work class；callback 不得重入当前 Kernel。
- Kernel 不读系统时钟。Create、submit、drive 和外部完成报告都必须带宿主时间；同一 Kernel 时间不得倒退。

### 3.2 基础类型、预算、错误

```rust
use core::{future::Future, pin::Pin, task::{Context, Poll}};
use std::{num::{NonZeroU32, NonZeroUsize}, rc::Rc};

pub type LocalBoxFuture<'a, T> = Pin<Box<dyn Future<Output = T> + 'a>>;

pub struct KernelId(pub [u8; 16]);
pub struct CommandId(pub [u8; 16]);
pub struct ArtifactId(pub [u8; 32]);
pub struct RunId([u8; 16]);
pub struct EffectId([u8; 16]);
pub struct WaitId([u8; 16]);
pub struct CapabilityId([u8; 16]);
pub struct LeaseId([u8; 16]);
pub struct SubjectId(pub String);

pub struct Revision(pub u64);
pub struct Cursor(pub u64);
pub struct TimestampMs(pub u64);
pub struct Fence(pub u64);
pub struct StableKey(pub String);
pub struct SchemaId(pub String);
pub struct ResourceId(pub String);
pub struct OperationId(pub String);
pub struct MeterId(pub String);

pub struct Payload { pub schema: SchemaId, pub bytes: Vec<u8> }

pub struct MeterLimit { pub meter: MeterId, pub max: u64 }
pub struct MeterAmount { pub meter: MeterId, pub amount: u64 }

pub struct BudgetLimit {
    pub max_depth: NonZeroU32,
    pub max_children: u32,
    pub max_concurrent: NonZeroU32,
    pub max_effect_attempts: NonZeroU32,
    pub max_messages: u64,
    pub max_items: u64,
    pub max_bytes: u64,
    pub max_payload_bytes: u64,
    pub meters: Vec<MeterLimit>,
}

pub struct BudgetUsage {
    pub children: u32,
    pub concurrent: u32,
    pub effect_attempts: u32,
    pub messages: u64,
    pub items: u64,
    pub bytes: u64,
    pub meters_reserved: Vec<MeterAmount>,
    pub meters_consumed: Vec<MeterAmount>,
}

pub struct KernelConfig {
    pub id: KernelId,
    pub code_revision: String,
    pub limits: BudgetLimit,
    pub event_page_size: NonZeroUsize,
    pub effect_claim_ttl_ms: u64,
}

pub struct BootstrapAuthority {
    pub actor: SubjectId,
    pub scope: CapabilityScope,
}

pub enum OpenMode {
    Create { now: TimestampMs, bootstrap: BootstrapAuthority },
    Recover,
}

#[non_exhaustive]
pub enum ErrorCode {
    NotFound, AlreadyExists, Conflict, InvalidTransition, StaleHandle,
    CursorExpired, BudgetExceeded, Backpressured, Unauthorized,
    DeadlineExceeded, CancellationTimedOut, NeedsMigration, CorruptState,
    Busy, HostUnavailable, Internal,
}

pub struct KernelError {
    pub code: ErrorCode,
    pub message: String,
    pub retry_after_ms: Option<u64>,
}

#[non_exhaustive]
pub enum HostErrorKind { Conflict, Unavailable, InvalidData, Permanent }

pub struct HostError {
    pub kind: HostErrorKind,
    pub code: String,
    pub message: String,
    pub retry_after_ms: Option<u64>,
}
```

通用 meter 只强制宿主在 proposal 时声明的 reservation；receipt 结算真实消耗并释放差额。外部系统中未经过 kernel 的消耗、突发速率和价格换算仍不在 kernel 保证内。每域并发可建成有容量的 Resource；每秒速率桶由 Effects/外部限流器负责。

### 3.3 Store、Policy、Effects 端口

```rust
pub trait Store {
    fn load<'a>(&'a self, kernel: &'a KernelId)
        -> LocalBoxFuture<'a, Result<Option<StoredKernel>, HostError>>;

    fn commit<'a>(&'a self, batch: StoreCommit)
        -> LocalBoxFuture<'a, Result<(), HostError>>;

    fn read_events<'a>(&'a self, query: EventQuery)
        -> LocalBoxFuture<'a, Result<EventPage, HostError>>;
}

pub struct StoredKernel {
    pub revision: Revision,
    pub last_cursor: Cursor,
    pub code_revision: String,
    pub state_format: u32,
    pub state: Vec<u8>,
}

pub struct StoreCommit {
    pub kernel: KernelId,
    pub expected_revision: Revision,
    pub new_revision: Revision,
    pub state_format: u32,
    pub state: Vec<u8>,
    pub events: Vec<KernelEvent>,
}

pub struct EventQuery { pub kernel: KernelId, pub after: Cursor, pub limit: NonZeroUsize }

pub enum EventPage {
    Batch { events: Vec<KernelEvent>, next: Cursor, caught_up: bool },
    Expired { earliest: Cursor, latest: Cursor },
}

pub trait Policy {
    // 每次 effect attempt 都调用；不得重入同一 Kernel。
    fn authorize<'a>(&'a self, intent: EffectIntent)
        -> LocalBoxFuture<'a, Result<PolicyRecord, HostError>>;
}

pub struct PolicyRecord {
    pub decided_at: TimestampMs,
    pub decision: Authorization,
    pub evidence: Option<Payload>,
}

pub enum Authorization {
    Allow,
    Deny { reason: Payload },
    Ask { request: InputRequest },
}

pub trait Effects {
    // 必须是有界 dispatch：长时物理作业返回 Pending，不得把 future 挂到作业结束。
    fn execute<'a>(&'a self, call: EffectCall)
        -> LocalBoxFuture<'a, Result<TimedEffectDispatch, HostError>>;

    fn cancel<'a>(&'a self, effect: EffectId, attempt: u32, fence: Fence)
        -> LocalBoxFuture<'a, Result<TimedCancelDisposition, HostError>>;
}

pub struct TimedEffectDispatch {
    pub recorded_at: TimestampMs,
    pub dispatch: EffectDispatch,
}

pub enum EffectDispatch {
    Completed(EffectReceipt),
    Pending { external_ref: Payload, next_check: Option<TimestampMs> },
}

pub struct EffectReceipt {
    pub outcome: EffectOutcome,
    pub external_receipt: Option<Payload>,
    pub charges: Vec<MeterAmount>,
}

pub enum EffectOutcome {
    Succeeded { output: Payload },
    Failed { kind: EffectFailureKind, detail: Payload },
    Unknown { detail: Payload },
}

pub enum EffectFailureKind {
    Rejected, InvalidInput, Unavailable, TimedOut, Cancelled, Permanent,
}

pub struct TimedCancelDisposition {
    pub recorded_at: TimestampMs,
    pub disposition: CancelDisposition,
}

pub enum CancelDisposition { Accepted, AlreadyFinished, NotInterruptible, Unknown }
```

`Pending` 只表示外部系统已受理且仍在运行。push 型 adapter 完成后提交 `EffectOp::Record`；poll 型 adapter 把 poll 本身建成新的 Effect 或由自己的 durable job service 在完成时 Record。Kernel 不把外部 job handle 藏在 side channel：`external_ref`、最终 receipt、失败 detail 都进入权威投影。

### 3.4 五个宿主操作

```rust
pub struct Kernel { _private: () }

pub struct OpenedKernel {
    pub kernel: Kernel,
    pub bootstrap_authority: Option<CapabilityId>, // 仅 Create 返回 Some
}

impl Kernel {
    pub async fn open(
        mode: OpenMode,
        config: KernelConfig,
        store: Rc<dyn Store>,
        policy: Rc<dyn Policy>,
        effects: Rc<dyn Effects>,
    ) -> Result<OpenedKernel, KernelError>;

    pub async fn submit(
        &self,
        now: TimestampMs,
        command: Command,
    ) -> Result<CommandAck, KernelError>;

    pub async fn drive(&self, request: DriveRequest)
        -> Result<DriveReport, KernelError>;

    pub async fn snapshot(&self, query: SnapshotQuery)
        -> Result<Snapshot, KernelError>;

    pub fn subscribe(&self, subscription: Subscription) -> EventStream;
}

#[derive(Clone, Copy, PartialEq, Eq)]
pub enum WorkClass { Control, Interactive, Background }

pub struct DriveRequest {
    pub now: TimestampMs,
    pub classes: Vec<WorkClass>,
    pub max_transitions: NonZeroU32,
    pub max_effects: NonZeroU32,
}

pub struct DriveReport {
    pub transitions: u32,
    pub effects_attempted: u32,
    pub more_work: bool,
    pub next_wakeup: Option<TimestampMs>,
}
```

`submit(now, …)` 的 `now` 是本次 commit time，也是 Lease 相对 TTL 被改成绝对窗口前的合法时间来源。相同 `CommandId` 重试即使传入更晚 `now` 也返回原 ack，不生成第二个时间事实。`drive.now` 负责 timer、not-before、lease expiry 和 claim expiry；Policy/Effects 的返回自带完成时刻。

Control 包含 cancel、lease expiry/fence 失效、deadline、Unknown reconciliation；Interactive 包含 Wait/Respond 与普通短交互；Background 包含普通 effect dispatch、artifact housekeeping。实现必须先 durable claim 再 await callback，所以一个 Background callback pending 时，另一个 `drive(classes=[Control])` 可以推进取消；不同 work class 不意味着外部设备有硬实时上界。SCADA/机器人/航空的急停仍须由独立安全链路保证，并向 kernel 补记事实。

### 3.5 Command、Run、Wait、消息

```rust
pub struct Command {
    pub id: CommandId,
    pub expected_revision: Option<Revision>,
    pub actor: SubjectId,
    pub authority: CapabilityId,
    pub causation: Option<CommandId>,
    pub kind: CommandKind,
}

pub enum CommandKind {
    Run(RunOp),
    Coordinate(CoordinationOp),
    Effect(EffectOp),
    Authority(CapabilityOp),
    Resource(ResourceOp),
}

pub enum RunOp {
    Start { spec: RunSpec },
    Spawn { parent: RunId, key: StableKey, spec: RunSpec },
    Finish { run: RunId, outcome: RunOutcome },
    Cancel { run: RunId, reason: Payload },
}

pub struct RunSpec {
    pub kind: SchemaId,
    pub input: Payload,
    pub budget: BudgetLimit,
    pub deadline: Option<TimestampMs>,
}

pub enum RunOutcome {
    Succeeded { result: Payload, artifacts: Vec<ArtifactId> },
    Failed { kind: String, detail: Payload },
}

pub enum CoordinationOp {
    Send {
        from: RunId,
        to: RunId,
        key: StableKey,
        payload: Payload,
        deadline: Option<TimestampMs>,
    },
    Wait { run: RunId, key: StableKey, condition: WaitCondition },
    Respond { run: RunId, wait: WaitId, response: WaitResponse },
}

pub struct MessageMatch {
    pub schema: SchemaId,
    pub from: Option<RunId>,
    pub key: Option<StableKey>,
    pub expires_at: TimestampMs,
}

pub enum WaitCondition {
    Children { runs: Vec<RunId>, mode: JoinMode },
    Message(MessageMatch),
    Input(InputRequest),
    Until(TimestampMs),
}

pub enum JoinMode { All, Any }

pub struct InputRequest {
    pub schema: SchemaId,
    pub prompt_key: String,
    pub context: Option<Payload>,
    pub expires_at: TimestampMs,
}

pub enum WaitResponse {
    Input(Payload), Approve, Deny { reason: Payload }, Cancel,
}
```

Kernel 核验 `Send.from` 属于 command authority 的 run scope，且目标 Run 可接收消息；不能仅靠自报 `from`。`Wait::Message` 可按 sender + StableKey + schema 相关，解决同 schema 串扰；复杂谓词、置信度、签名和 freshness 仍由 child verifier Run 解析 Payload。

### 3.6 Capability：所有可变命令的原子授权点

```rust
#[non_exhaustive]
pub enum CommandClass {
    StartRoot, Spawn, Finish, CancelRun, Send, Wait, Respond,
    ProposeEffect, RetryEffect, RecordEffect, CancelEffect,
    GrantCapability, RevokeCapability,
    DefineResource, AcquireLease, RenewLease, ReleaseLease,
    PublishArtifact, InvalidateArtifact,
}

pub enum RunScope {
    Kernel,
    Subtree(RunId),
    Exact(Vec<RunId>),
}

pub struct CapabilityScope {
    pub runs: RunScope,
    pub commands: Vec<CommandClass>,
    pub resources: Vec<ResourceId>,
    pub operations: Vec<OperationId>,
    pub budget: BudgetLimit,
}

pub enum CapabilityOp {
    Grant {
        target: Option<RunId>,
        parent: CapabilityId,
        scope: CapabilityScope,
        expires_at: Option<TimestampMs>,
    },
    Revoke { capability: CapabilityId },
}
```

Create 时 kernel 原子生成 bootstrap authority；此后没有 `parent=None` 的普通 Grant。每个 Command 在同一状态 revision 内核验 authority 未撤销、未过期、actor 绑定、run/command/resource/operation scope 与 attenuation。Effect attempt 还必须核验 `EffectProposal.capability` 并调用 Policy。宿主仍负责“这个登录用户确实是 SubjectId”的认证，但旧指挥员被 revoke 后不能靠自报 actor 提交 Cancel/Respond/Lease。

Quorum 不加入 kernel：应用为每位独立批准者建 child Run/Input Wait，capability 将每个 Respond 限定到对应 Run/CommandClass；父 Run `Children(All)` 汇合。这样身份、顺序、角色分离由可审计的通用原语组成，而不是固化 `two_person_rule`。

### 3.7 Effect 完整闭环

```rust
pub struct RequiredLease {
    pub lease: LeaseId,
    pub resource: ResourceId,
    pub fence: Fence,
}

pub struct EffectProposal {
    pub run: RunId,
    pub key: StableKey,
    pub operation: OperationId,
    pub resource: ResourceId,
    pub input: Payload,
    pub capability: CapabilityId,
    pub required_leases: Vec<RequiredLease>,
    pub reservations: Vec<MeterAmount>,
    pub class: WorkClass,
    pub not_before: Option<TimestampMs>,
    pub deadline: Option<TimestampMs>,
}

pub enum EffectOp {
    Propose(EffectProposal),
    Retry {
        effect: EffectId,
        not_before: TimestampMs,
        reason: Payload,
    },
    Cancel { effect: EffectId, reason: Payload },
    Record {
        effect: EffectId,
        attempt: u32,
        fence: Fence,
        receipt: EffectReceipt,
        recorded_at: TimestampMs,
    },
    ReconcileUnknown {
        effect: EffectId,
        receipt: EffectReceipt,
        recorded_at: TimestampMs,
        evidence: Payload,
    },
}

pub struct EffectIntent {
    pub effect: EffectId,
    pub run: RunId,
    pub operation: OperationId,
    pub resource: ResourceId,
    pub input: Payload,
    pub capability: CapabilityId,
    pub required_leases: Vec<RequiredLease>,
    pub idempotency_key: StableKey,
    pub attempt: u32,
    pub reservations: Vec<MeterAmount>,
    pub class: WorkClass,
    pub not_before: Option<TimestampMs>,
    pub deadline: Option<TimestampMs>,
}

pub struct EffectCall { pub intent: EffectIntent, pub fence: Fence }
```

语义：

1. `(run, EffectProposal.key)` 唯一标识逻辑 Effect；重复 Propose 返回同一 EffectId/ack，不会创建随机 attempt。
2. `Retry` 只允许显式领域决定；保留同一 EffectId/key，attempt 与 effect fence 递增。Failed 不会自动重试；Unknown 必须先 Reconcile，或由应用在 reason/evidence 中明确接受重复风险后再 Retry。
3. `Record` 只接受当前 attempt/fence，解决异步外部 job；过时 callback 返回 `StaleHandle`。
4. 每个 attempt 在 authorize、claim 和 dispatch 前原子验证 capability、预算 reservation、deadline、not-before 以及所有 required lease fences。任一 lease 失效，旧排队 effect 不得执行。
5. effect fence 是 attempt ownership；resource fence 来自 RequiredLease，二者不可混用。设备 gateway 应同时转发所需 resource fence。
6. receipt/output/detail/external receipt/charges 都持久化并出现在 Snapshot；不允许 Effects adapter 自建不可对账的第二结果账本。

### 3.8 资源、原子租约集合与 Artifact

```rust
pub enum LeaseMode { Shared, Exclusive }

pub struct LeaseWindow {
    pub valid_from: TimestampMs,
    pub expires_at: TimestampMs,
}

pub struct ResourceClaim {
    pub resource: ResourceId,
    pub mode: LeaseMode,
    pub units: NonZeroU32,
    pub window: LeaseWindow,
}

pub struct ResourceDefinition {
    pub resource: ResourceId,
    pub capacity: NonZeroU32,
}

pub enum ResourceOp {
    Define(ResourceDefinition),
    Lease(LeaseOp),
    Artifact(ArtifactOp),
}

pub enum LeaseOp {
    AcquireSet { run: RunId, key: StableKey, claims: Vec<ResourceClaim> },
    Renew { lease: LeaseId, expected: Vec<RequiredLease>, expires_at: TimestampMs },
    Release { lease: LeaseId, expected: Vec<RequiredLease> },
}

pub enum ArtifactOp {
    Publish {
        run: RunId,
        expected_workspace: Revision,
        artifact: ArtifactRef,
        inputs: Vec<ArtifactId>,
    },
    Invalidate { run: RunId, artifact: ArtifactId, reason: Payload },
}

pub struct ArtifactRef {
    pub id: ArtifactId,
    pub media_type: String,
    pub size_bytes: u64,
    pub locator: String,
}

pub struct LeaseClaimGrant {
    pub resource: ResourceId,
    pub mode: LeaseMode,
    pub units: NonZeroU32,
    pub window: LeaseWindow,
    pub fence: Fence,
}

pub struct LeaseGrant {
    pub lease: LeaseId,
    pub claims: Vec<LeaseClaimGrant>,
}
```

`AcquireSet` 对整个 claims 向量全有或全无。冲突规则仅作用于相同 ResourceId 且重叠时间窗：Exclusive 与任何 claim 冲突；Shared 的 units 总和不得超过已定义 capacity。动态空间/航路重叠必须由应用先规范化成共享 ResourceId/时槽，或交给一个外部事务型 Effect；kernel 不内置几何求交。

ResourceDefinition 的修改在仍有相关 lease 时拒绝。未来预约用绝对窗口；即时 heartbeat 由 `valid_from=submit.now` + Renew 表达。合成 bundle ResourceId 不再是多资源互斥的推荐做法。

本版不加入“Spawn + Acquire + Propose 的任意事务 batch”。Run 创建本身无外部副作用；宿主应按 StableKey 恢复后对账、让短期 lease 自动过期，并在取得完整 LeaseSet 后才 Propose effect。若两个以上实际实现仍证明跨不同账本必须有可引用新 ID 的事务，再设计 typed transaction，而不是先公开无法安全引用中间 ID 的 `Vec<CommandKind>`。

### 3.9 Ack 与完整恢复 Snapshot

```rust
pub struct CommandAck {
    pub command: CommandId,
    pub revision: Revision,
    pub cursor: Cursor,
    pub created_run: Option<RunId>,
    pub created_wait: Option<WaitId>,
    pub created_effect: Option<EffectId>,
    pub created_capability: Option<CapabilityId>,
    pub lease: Option<LeaseGrant>,
}

pub struct SnapshotQuery {
    // None：列举 roots；Some：取该 root。include_descendants 决定是否展开。
    pub root: Option<RunId>,
    pub include_descendants: bool,
    pub include_terminal_roots: bool,
}

pub struct Snapshot {
    pub kernel: KernelId,
    pub revision: Revision,
    pub at_cursor: Cursor,
    pub code_revision: String,
    pub roots: Vec<RunId>,
    pub runs: Vec<RunView>,
    pub resources: Vec<ResourceView>,
}

pub struct RunView {
    pub id: RunId,
    pub parent: Option<RunId>,
    pub spec: RunSpec,
    pub status: RunStatus,
    pub outcome: Option<RunOutcome>,
    pub deadline: Option<TimestampMs>,
    pub budget_limit: BudgetLimit,
    pub budget_usage: BudgetUsage,
    pub children: Vec<RunId>,
    pub messages: Vec<MessageView>,
    pub waits: Vec<WaitView>,
    pub effects: Vec<EffectView>,
    pub capabilities: Vec<CapabilityView>,
    pub leases: Vec<LeaseView>,
    pub workspace_revision: Revision,
    pub artifacts: Vec<ArtifactView>,
}

pub enum RunStatus { Pending, Running, Waiting, Cancelling, Succeeded, Failed, Cancelled }

pub struct MessageView {
    pub from: RunId,
    pub to: RunId,
    pub key: StableKey,
    pub payload: Payload,
    pub deadline: Option<TimestampMs>,
    pub committed_at: TimestampMs,
}

pub struct WaitView {
    pub id: WaitId,
    pub key: StableKey,
    pub condition: WaitCondition,
    pub state: WaitState,
    pub resolution: Option<WaitResolution>,
}

pub struct WaitResolution {
    pub actor: Option<SubjectId>,
    pub response: Option<WaitResponse>,
    pub resolved_at: TimestampMs,
}

pub enum WaitState { Open, Resolved, Expired, Cancelled }

pub struct EffectView {
    pub id: EffectId,
    pub proposal: EffectProposal,
    pub state: EffectState,
    pub attempt: u32,
    pub fence: Option<Fence>,
    pub policy: Option<PolicyRecord>,
    pub dispatch: Option<EffectDispatch>,
    pub receipt: Option<EffectReceipt>,
}

pub enum EffectState {
    Proposed, WaitingForApproval { wait: WaitId }, Authorized, Claimed,
    InFlight, Succeeded, Failed { kind: EffectFailureKind }, OutcomeUnknown,
    RetryScheduled { at: TimestampMs }, CancelRequested,
}

pub struct CapabilityView {
    pub id: CapabilityId,
    pub actor: SubjectId,
    pub target: Option<RunId>,
    pub scope: CapabilityScope,
    pub expires_at: Option<TimestampMs>,
    pub revoked: bool,
}

pub struct LeaseView {
    pub id: LeaseId,
    pub run: RunId,
    pub key: StableKey,
    pub active: bool,
    pub claims: Vec<LeaseClaimGrant>,
}

pub struct ResourceView { pub definition: ResourceDefinition }

pub struct ArtifactView {
    pub artifact: ArtifactRef,
    pub producer: RunId,
    pub inputs: Vec<ArtifactId>,
    pub workspace_revision: Revision,
    pub valid: bool,
}
```

Snapshot 是**当前权威运行投影**，不是历史时光机。它必须足以在 cursor gap 后重新派发未终态 Run、渲染 InputRequest、识别 Unknown effect、取得已完成 result/receipt，以及列举未知 root。大 Payload 可放 Artifact locator，但引用、schema、状态和关联 ID 必须在 Snapshot 内；不能以“宿主应该恰好另存一份”作为恢复前提。

精确历史冻结由应用发布 immutable evidence manifest（含 cursor、ArtifactId、输入版本）完成；不加入 `snapshot(as_of)`，因为 Store retention 与任意历史状态重建是另一套成本模型。

### 3.10 Event 与订阅

```rust
pub struct Subscription {
    pub after: Cursor,
    pub filter: EventFilter,
    pub max_batch: NonZeroUsize,
}

pub enum EventFilter {
    Kernel,
    Run { root: RunId, include_descendants: bool },
}

pub struct EventStream { _private: () }

impl futures_core::Stream for EventStream {
    type Item = Result<EventBatch, StreamError>;
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<Option<Self::Item>>;
}

pub struct EventBatch { pub events: Vec<KernelEvent>, pub next: Cursor, pub caught_up: bool }
pub enum StreamError { Gap { earliest: Cursor, latest: Cursor }, Closed, Host(HostError) }

pub struct KernelEvent {
    pub cursor: Cursor,
    pub revision: Revision,
    pub recorded_at: TimestampMs,
    pub run: Option<RunId>,
    pub command: Option<CommandId>,
    pub causation: Option<CommandId>,
    pub actor: Option<SubjectId>,
    pub authority: Option<CapabilityId>,
    pub kind: EventKind,
}

pub enum EventKind {
    RunCreated { run: RunId, parent: Option<RunId>, spec: RunSpec },
    StateChanged { from: RunStatus, to: RunStatus, reason: Option<Payload> },
    MessageCommitted { message: MessageView },
    WaitChanged { wait: WaitView },
    PolicyDecided { effect: EffectId, record: PolicyRecord },
    EffectChanged { effect: EffectView },
    CapabilityChanged { capability: CapabilityView },
    ResourceDefined { resource: ResourceDefinition },
    LeaseChanged { lease: LeaseView },
    ArtifactChanged { artifact: ArtifactView },
    BudgetCharged { usage: BudgetUsage },
    Terminal { outcome: RunOutcome },
}
```

Event 使用与 Snapshot 相同的 view DTO，避免“事件告诉你变了却无法知道变成什么”。允许 payload 较大时以 ArtifactRef 间接引用，但不能删除恢复所需字段。actor/authority、Wait response、Policy evidence、Effect receipt 都成为原子审计事实。

顺序保证：

1. 同一 KernelId 内 cursor 严格递增；同一 revision 的事件连续；不同 Kernel 无顺序保证。
2. submit/drive 仅在 state + events 同一 Store CAS 成功后返回；重复读取按 cursor 去重。
3. 每个 Run 恰有一个 Terminal；stream close、provider EOF、job accepted、idle 都不是终态。
4. `snapshot.at_cursor=C` 后订阅 `after=C`；Gap 后重新 snapshot。
5. EventStream 有界，只保存 wakeup；Store retention、加密与归档由宿主负责。
6. raw token/progress 是可丢 UX 流，不进入 stable EventKind，也不得驱动状态。

---

## 4. 关键语义与惯用组装

### 4.1 StableKey 作用域

- Spawn：`(parent RunId, key)`；重复返回同一个 child。
- Send：`(from, to, key)`；重复返回原接受事实。
- Wait：`(run, key)`；已终态 Wait 不因同 key 重开，重建 timer 必须用新 key。
- Effect：`(run, key)`；Retry 保持同一个 EffectId。
- AcquireSet：`(run, key)`；重复返回同一个 lease set。

StableKey 是逻辑动作键，不是随机 attempt ID。canonical URL frontier 可以用固定 parent + URL key + child Run；更复杂的 near-duplicate/claim ledger 用应用 Artifact/数据库。

### 4.2 验证门、审批 quorum 与污染隔离

验证门：每个 check 为 child Run，结果/输入 revision 发布 Artifact，父 Run `Children(All)`，应用确认 artifact inputs 仍指向冻结 manifest。新 merge 使旧结果 stale 时 Invalidate，再 spawn 新 key。flaky policy 属 app，effect Retry 提供连续审计链。

审批 quorum：每个批准者一个受不同 capability 约束的 child/Input Wait；父 Join(All/Any)。Kernel 记录 actor、authority、response 和时间。复杂“双人且角色不同”“不可同人”由 Policy/IAM 发放互斥 capability；不新增 quorum DSL。

污染隔离：Wait::Message 按 sender/key/schema；Send.from 受 authority 核验。未验证证据与已验证证据分别由不同 producer Run/Artifact manifest 表示；synthesis Run 的 capability/输入只指向 verified manifest。Kernel 不做 prompt-injection 分类或字段级 visibility label。

### 4.3 长时作业与紧急控制

Effects::execute 只 dispatch。LSF job、12 小时炉实验、机器人移动或设备命令返回 `Pending { external_ref }`，随后 adapter Record 最终 receipt；因此 background drive 不占据整个物理生命周期。Cancel 是 EffectOp 且属于 Control；单独 control drive 可在 background callback 等待时推进。

硬实时 PLC/急停/保护继电器必须绕过通用 kernel，直接作用设备，再以 Record/Cancel/Message 补记。v2 承诺的是事实优先级与不会被单个长物理 job 阻塞，不承诺操作系统调度、网络或设备响应的延迟上界。

### 4.4 外部观测、世界模型与 compare-and-use

GPS、遥测、ETA、CUSRES、地图、置信度与 observed_at/sequence 属领域状态，不进入通用 kernel state。gateway Run 可以 Send 低频关键事实；高频流进入外部版本化状态库，决策 Run 的 input/Artifact manifest 固化所读版本。需要原子“读取实体版本并派单”时，把数据库 compare-and-use 做成一个 Effect，并在 receipt 中记录采用版本。

Kernel 只对自己拥有的前置条件原子强制：expected_revision、Artifact validity/workspace revision、Capability、Budget 与 RequiredLease fences。它不能对未纳入账本的司机 GPS v42 或地图 v17 作 CAS，也不应伪装能做到。

### 4.5 恢复、升级与跨 Kernel

Recover 后先 `snapshot(root=None, include_descendants=true)`，按 `RunView.spec.kind/input` 重建 dispatcher，再依据 WaitView/EffectView/outcome 续跑。Store event retention 可有限，因为 snapshot 已闭合当前事实。

`NeedsMigration` 仍是稳定失败。v2 不公开 opaque state patch 或 Temporal 式 workflow patch DSL；首选 pin + drain。真实出现无法 drain 的长期部署后，Store/版本 adapter 可由同版本 kernel crate 提供离线 whole-state migrator，但该运维 API不属于运行窄腰。

不同 Kernel 仍无顺序、共享 capability/lease 或 fork merge。跨区/跨 USS/断网自治用外部 durable inbox/outbox、签名 correlation、saga 与控制权协议；若必须防双主，采用共识存储或单一权威调度服务，而不是复制本地 Kernel 后尝试合并。

---

## 5. 税系簇溯源与新增项门槛

| Public 项 | 至少两个依据 |
|---|---|
| `open/submit` 显式时间 | #3 事件事实；#10 timer/lease；航空、多 agent coding |
| 完整 Run/Wait/Effect snapshot + roots | #3 observation；#4 recovery；编码、研究、客服、救灾、交通、实验、芯片等 |
| EffectId ack、receipt view、Retry/Record/Reconcile | #5 effect ledger；#6 attempt；#4 recovery；几乎全部领域 |
| sender/key message match | #7 signal；#11 messaging；研究污染隔离、港口相关、grid 证据门 |
| command capability + actor audit | #2 authority/causation；#8 revoke；救灾、空域、grid、芯片、仓储 |
| atomic AcquireSet + window/mode/capacity | #9 admission；#10 lease/fence；仓储、派单、grid、港口、实验、空域 |
| RequiredLease on Effect | #5 effect enforcement；#10 resource fence；交通、grid、实验 |
| WorkClass + concurrent class drive | #1 cancel；#5 effect；#9 QoS；空域、grid、交通、救灾、实验 |
| generic meter reservation/settlement | #9 hard budget；#5 effect accounting；研究 token/cost 与共享配额 |
| Artifact/Workspace | #2 provenance；#3 revision；#13 workspace；编码/研究/芯片 |

仍拒绝扩面：workflow DAG/FSM、queue/dequeue、broadcast、domain signal store、historical snapshot、quorum DSL、cross-kernel transaction/CRDT、资源几何、模型/provider、UI DTO、数据库/telemetry。

---

## 6. 异步、错误与一致性摘要

1. `submit` 是 cancel-safe 线性化操作；Future 返回前，命令要么未提交，要么 state/event 已一起提交。相同 CommandId 得原 ack。
2. `drive` bounded；每个 work item 在 await 外部 callback 前先 claim，callback 后再 CAS。不同 class drive 可并存；同一 item 由 fence 防重复 owner。
3. Policy/Effects 不得 reenter。Effects 必须有界 dispatch；长期运行使用 Pending + Record。
4. `HostError.kind` 和 `KernelError.code` 决定控制流，不解析 message。Conflict 要重新 snapshot；CursorExpired/Gap 要重新 snapshot；NeedsMigration 不得静默 Create。
5. effect 至少一次 attempt，可能 Unknown；Failed 不自动 retry，Unknown 不自动伪装 Failed。Retry/Reconcile 是显式、可审计状态转移。
6. capability revoke 对 commit 后尚未 claim 的命令/effect 生效；in-flight 物理动作仍需 cancel/设备 fence。
7. lease 是逻辑控制权；只有当前 RequiredLease fence 的 effect 可 dispatch。TTL 不代表设备已物理停止。
8. Store 的 state + event commit 原子；message 只承诺 durable accepted，不承诺业务 consumer 已处理；event retention 可有限但 Snapshot 必须闭合当前状态。
9. 预算 reservation 能阻止 kernel 内并发超额；receipt 结算实际值。外部未申报消耗不在保证内。

---

## 7. 开放问题

1. 单 aggregate CAS 的真实性能上限；先 benchmark，再决定按 Run 分片或 delta store，不预暴露 shard API。
2. local-first 与独立 `Send + Sync` actor adapter 的真实需求量；目前 adapter 比污染 core 更合适。
3. capability bootstrap 与产品 IAM/签名的具体映射；本库仍假定可信宿主。
4. artifact locator 的前端可见性、加密和 retention；由部署威胁模型决定。
5. 长期不停机升级是否真的无法 pin/drain；出现两个实现后再定义 migrator contract。
6. 是否有两个以上实现需要跨 Run/Lease/Effect、并能安全引用新 ID 的 typed transaction；在此之前不加 generic batch。
7. Store 是否应提供审计归档流而非仅运行 retention；不影响 current Snapshot 闭合。
8. wire adapter 何时达到两个独立 consumer、版本偏差和长期兼容预算的门槛。

---

## 8. 最终取舍

v2 保留了 v1 的保守边界，但修正了一个关键自相矛盾：既然 kernel 声称拥有 outcome ledger、gap→snapshot 恢复和 lease/fence 强制点，公共观察面就必须能读回完整运行定义、等待请求、结果、receipt 与授权事实；effect 也必须能连续 retry/reconcile，并在执行时验证资源 fence。

同时，接口仍不冒充领域平台。高频世界模型、空间冲突、队列优化、SLA、审批规则、外部数据库 compare-and-use、跨区域共识和硬实时保护都留在正确层。最小接口面不是字段最少，而是**只保留跨至少两个领域反复出现、且不放进 kernel 就无法原子强制或恢复的事实**。

---

## 9. 修订记录：12 路走查逐条裁决

说明：`接受` 表示修改稳定接口；`组装` 表示现有/修订后原语足够，只补语义或惯用法；`边界外` 表示问题真实，但不属于本提案的单进程 kernel；`暂缓` 表示仅单领域或证据不足。12 路均未提出 overdesign 条目，因此没有因 overdesign 删除原 v1 项。

### 9.1 multi-agent-coding

1. 非 Send、多 worker 要经 owner 线程：**组装**。保持 local-first；文档明确专用 owner task/thread + channel facade，不为并发 worker 改 ABI。
2. 单资源单模式锁难表达 Cargo.lock/API 联合与读写：**接受**。新增 `Define + AcquireSet`、共享/独占、容量、绝对窗口，全有或全无。
3. 验证门只能拼 child/artifact/join：**组装**。这是通用 Run/Artifact/Join 的预期组合；补充 frozen manifest 与 stale invalidation 惯用法。
4. submit/open 无合法时间：**接受**。Create、submit、drive、Policy/Effects completion 都显式携时刻；Lease 使用绝对窗口。
5. Effect output 黑洞且 ack 无 EffectId：**接受**。Ack 加 EffectId，Snapshot/Event 加完整 EffectView、dispatch、receipt/output/detail。
6. gap 后无法列 roots、派发或读 result：**接受**。SnapshotQuery 可列 root；RunView 加 spec/outcome/message；Wait/Effect 完整投影。

### 9.2 deep-research

1. URL frontier 用 parent+StableKey 很绕：**组装**。明确 StableKey scope；canonical URL child 是可接受的小规模 keyed frontier，near-duplicate 索引仍属 app。
2. claim ledger 用整表 Artifact 冲突：**边界外**。细粒度 claim/index 是领域数据服务；kernel 不加通用 mutable blackboard，可用外部 CAS DB + Artifact checkpoint。
3. 价值优先级只能外队列：**部分接受**。Effect 加 `WorkClass/not_before` 保证 QoS 与定时；价值函数和候选队列仍属 app。
4. 历史报告冻结需 manifest：**组装**。immutable evidence manifest 是正确边界；不加历史 Snapshot。
5. Recover 看不到 RunSpec：**接受**。RunView/Event RunCreated 均含 spec，且可列 root。
6. Effect 业务结果不可读取：**接受**。完整 receipt 进入 Snapshot/Event。
7. Wait::Message 无 sender/key、消息/Artifact 无可见性：**部分接受**。加 sender/key match、Send authority；verified manifest + capability 隔离 Artifact，不加字段级 visibility label。
8. token/货币/任意预算无法硬执行：**接受税系项**。Budget 加通用 meter reservation/settlement；外部速率与未申报消耗仍由限流器/Resource。
9. 429/5xx 没 retry/reschedule/detail：**接受**。加 Retry(not_before/reason)、保留同 EffectId/attempt 链，并公开失败 detail；Unknown 用 Reconcile。

### 9.3 support-triage

1. SLA 暂停/恢复 timer 很绕：**组装**。取消旧 Wait、应用保存剩余营业时长、用新 key 建新 Until；营业日历不入核。
2. 只能取消整个 child 才能撤销单 Effect：**接受**。新增 `EffectOp::Cancel`。
3. 技能/容量/优先级/SLA 队列需外置：**边界外**。kernel 提供 WorkClass 与 Lease capacity，不提供 enqueue/dequeue 或领域匹配器。
4. 关闭工单重开需新 root：**组装**。Terminal 不可变是必要不变量；外部 ticket ID 串联历次 Run。
5. 审批 Wait 对前端不可表达：**接受**。WaitView/Event 保留 InputRequest、context、响应与 actor。
6. 领域 retry 没命令：**接受**。Effect Retry/Reconcile。
7. snapshot 无交接上下文：**接受**。Run spec/outcome/message/wait/effect 全量恢复。

### 9.4 warehouse-robots

1. 长动作拆成 ACK Effect + child Run：**接受通用闭环**。Effect 可 Pending，后续以当前 attempt/fence Record 最终 receipt，无需 drive 等物理完成。
2. 千车单 owner/整 aggregate CAS：**暂缓**。local actor facade 可用；规模/分片先 benchmark，跨车共识不是 v2 ABI。
3. PLC 急停需 kernel 外广播再补记：**边界外且认可**。硬实时安全链路必须外置；Control drive 只改善事实推进。
4. 非 Effect 命令无 capability：**接受**。Command authority + CommandClass/run/resource scope，原子 revoke 核验。
5. 时空预约、多格路线、容量、原子集合缺失：**接受**。AcquireSet + window/mode/units/capacity；几何规范化仍属 app。
6. 倒序机器人观测无法拒绝：**边界外**。versioned world model 属 app；关键决策输入以 Artifact manifest 固化。
7. Snapshot 无 Unknown effect/InputRequest：**接受**。完整 EffectView/WaitView。
8. 创建任务+pod+路线+派遣无法单事务：**暂缓**。先 AcquireSet，再 Propose；Spawn 用 StableKey 恢复对账。仅此类证据不足以加入可引用中间 ID 的 generic batch。

### 9.5 ride-dispatch

1. 完成订单快照无业务结果：**接受**。Run outcome 与 Effect receipt 进入 Snapshot。
2. 高频 GPS 不适合 durable kernel：**边界外且认可**。外部时序/分区流 + 版本 manifest。
3. offer 广播/首个有效接受需展开：**组装**。fan-out child/Send/Wait Any；地理 fan-out 和投递属 app，不加 broadcast。
4. 暂停模型/城市降级无 scope：**部分接受**。WorkClass 提供推进隔离；产品级 Pause/degradation config 仍属 app。
5. 订单+司机无法原子双锁：**接受**。AcquireSet 对 O7、D9 全有或全无。
6. 跨分片 ownership/fence：**边界外**。需要共识控制平面，不能由本地 Kernel API诚实保证。
7. 决策输入版本/freshness 前置条件：**边界外**。外部 entity CAS 由事务型 Effect；kernel 只强制自己拥有的 revision/artifact/lease。
8. 积压下取消无 QoS：**接受到库能力边界**。WorkClass + 可并存 control drive；不承诺 OS/网络硬实时。

### 9.6 quake-rescue

1. 安全广播/逐队确认需扇出：**组装**。child/Send/Wait；不加 group broadcast/ack DSL，紧急传输走外部通道。
2. IAP 版本切换无批量原子命令：**组装**。计划 Artifact + 激活 marker + 二阶段自停；不加 arbitrary batch。
3. 弱事实黑板需外部空间/新鲜度索引：**边界外且认可**。domain world model 不入核。
4. 慢 execute 阻断紧急 Cancel：**接受**。bounded dispatch/Pending + concurrent control drive。
5. 恢复看不到 RunSpec：**接受**。
6. OutcomeUnknown 看不到 intent/receipt：**接受**。
7. 非 Effect 命令缺原子授权：**接受**。Command capability/revoke。
8. InputRequest 断线不可恢复：**接受**。
9. 离线双重指挥历史不能合并：**边界外**。这是分布式冲突裁决/共识问题；Payload 只能留证，不能恢复双主授权。

### 9.7 traffic-twin

1. 实时观测伪装长期 Run：**边界外**。高频 observation/world model 属 app；只把关键摘要/manifest 入核。
2. 因果复盘需每次决策 Artifact：**组装**。这是正确 evidence snapshot；不加 as-of Snapshot。
3. 双人批准需宿主组装：**组装并加强审计**。多 child Wait + capability；Event 现记录 actor/authority/response。
4. 分区自治需多 Kernel saga：**边界外**。跨 Kernel correlation/outbox 属控制平面。
5. Recover 无 RunSpec：**接受**。
6. 设备 accepted/actual effective 无阶段账本：**接受**。Pending + external_ref，随后 Record 最终 receipt；完整视图可恢复。
7. Effect 没绑定 lease fence：**接受**。RequiredLease 与 attempt fence 分离。
8. 应急抢占无 QoS：**接受到软实时边界**。WorkClass/control drive；硬实时另路。

### 9.8 airspace-utm

1. 4D 去冲突/未来多资源预约只能外包：**部分接受**。通用未来 AcquireSet/容量加入；4D 几何冲突仍由外部 Effect。
2. 多 USS 联邦消息需 Effects 桥接：**边界外**。跨 Kernel durable inbox/outbox、签名与去重由 federation adapter。
3. 权威命令只能宿主前置授权：**接受**。所有 Command 带 atomically checked capability；IAM/验签仍在宿主。
4. 多复核/控制权交接需小 Run：**组装**。审批用 child；控制权可用一个 LeaseSet 原子取得，但跨组织确认仍是 saga。
5. Snapshot 不能恢复授权/receipt：**接受**。
6. 无应急 control QoS：**接受到库能力边界**。
7. submit 无 commit time：**接受**。显式时间贯穿 Create/submit/drive/callback。

### 9.9 grid-blackstart

1. 证据门无领域谓词：**组装**。verifier child + sender/key MessageMatch；拓扑/质量/龄阈值不入核。
2. 双人批准需要多个 Wait：**组装并加强审计**。不加 quorum DSL。
3. 补偿无一等关联：**组装**。causation + child Run + StableKey + Artifact/Join；补偿策略属 app。
4. 多资源/容量预留困难：**接受**。AcquireSet/units/capacity/window。
5. Snapshot 不闭合：**接受**。
6. Lease 与物理 Effect 不绑定：**接受**。RequiredLease。
7. 分区自治与防双主：**边界外**。需共识/权威控制平面。
8. actor/批准/Policy/receipt 审计丢失：**接受**。Event envelope actor/authority，Wait response、PolicyRecord、EffectReceipt 公共化。
9. 高优先级取消被慢 effect 阻塞：**接受到软实时边界**。Pending + control drive；保护继电器仍外置。

### 9.10 port-supplychain

1. 外部 EDI/ETA/遥测没有 ingress：**组装/边界外**。gateway Run + Send 适合低频事实；高频流外置，不加 Signal service。
2. 500 个同 schema Wait 无相关键：**接受**。MessageMatch 加 sender/key。
3. child result 不可见，被迫 Artifact：**接受**。RunOutcome 进入 RunView/Event；大结果仍推荐 Artifact。
4. Run deadline 不可更新：**暂缓**。取消+checkpoint+新 Run 或 app timer；动态 deadline 尚未跨领域证明需要修改不可变 RunSpec。
5. Effect ETA output 黑洞：**接受**。
6. 危险品批准者不可审计：**接受**。
7. QC/堆场/AGV 不能原子占用：**接受**。AcquireSet。
8. v1→v2 不停机升级无 migrator：**暂缓**。保持 NeedsMigration + pin/drain；单领域不足以冻结 migrator ABI。

### 9.11 auto-science-lab

1. 仪器排程要外置且无未来多资源/容量：**部分接受**。加入通用 AcquireSet/window/capacity；排程优化仍外置。
2. 物理样品稳定瓶号与版本谱系：**组装**。外部 sample manifest + immutable Artifact inputs；不把 mutable 实体塞进 ArtifactId。
3. 仪器事件需 gateway Run：**组装/边界外**。低频关键 event Send，高频 telemetry 外置。
4. 12 小时作业拆 submit/poll/stop 会失去身份：**接受**。Pending Effect 保留同一 EffectId，Record/Cancel 对应原作业。
5. Lease/Effect 无 fencing 绑定：**接受**。
6. XRD output 无读取路径：**接受**。
7. 长期恢复看不到 spec/result/message：**接受**。
8. 长 execute 阻断安全取消：**接受**。bounded dispatch + control drive。

### 9.12 chip-tapeout

1. 跨团队 DAG/反向依赖需 app 重建：**组装**。Artifact provenance + child/Join；不加通用 DAG/自动 invalidation。
2. EDA 长任务不能自然是 Effect：**接受**。Pending Effect + Record 保留逻辑 job 身份。
3. 资源向量/priority/排队原因：**部分接受**。Resource capacity/AcquireSet 与 WorkClass；复杂 scheduler queue/reason 仍属 LSF/Slurm adapter。
4. 单 aggregate 规模与按 block 拆分：**暂缓/边界外**。先 benchmark；跨 block transaction 不进 v2。
5. Recover 无 RunSpec：**接受**。
6. Effect/Run 结果不可恢复：**接受**。
7. license failure 无 retry：**接受**。Retry(not_before/reason) 保持同 EffectId。
8. freeze/compare-and-use 无原子强制：**部分接受**。Effect 在 claim 时核验 required leases/capability/artifact current state；外部 foundry DB 的 compare-and-use 仍应做单个 Effect。未加任意领域 entity precondition。
9. 审批与关键命令无 authority/audit：**接受**。Command capability、actor/authority/response/Policy/receipt 事件；双人规则仍用 child Wait 组装。

### 9.13 汇总

- **接受并修改的跨领域硬缺口：**显式时间、可列 root 的完整恢复投影、EffectId/receipt/result、Retry/Record/Reconcile/Cancel、sender/key message match、全命令 capability 与审计、原子带时间窗/容量 LeaseSet、RequiredLease fencing、bounded long-effect dispatch、control work lane、通用预算 meter。
- **用原语组装而不扩接口：**验证门、审批 quorum、广播确认、补偿、报告冻结、StableKey frontier、ticket reopen、plan version switch。
- **明确边界外：**高频世界模型与 freshness、领域 queue/optimizer/geometry、跨 Kernel/断网 fork/分片共识、硬实时急停、历史时光机、企业 IAM、wire protocol。
- **证据不足暂缓：**arbitrary multi-command transaction、Run deadline mutation、在线 state migrator、通用 mutable claim ledger。
