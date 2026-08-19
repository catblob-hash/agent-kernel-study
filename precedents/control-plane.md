TL;DR：Kubernetes 将「声明的期望状态 + 可恢复的版本化状态流 + 幂等收敛器」置于控制平面核心，而把具体领域决策留给可插拔 controller/operator；对 agent kernel，最小可靠内核应拥有权威状态、订阅/重放和协调语义，而不应内置 agent 的业务策略或执行实现。

## 概览

- 研究对象：Kubernetes control plane、controller reconciliation、watch/informer 与 Operator/CRD 扩展模式。
- 本文优先引用 Kubernetes 官方文档和 client-go API 文档；末尾单列可核对 URL。
- 「事实」是来源明确陈述或 API 参考；「推断」是对 Rust agent kernel 的设计类比，不是 Kubernetes 官方承诺。
- Kubernetes cluster 由 control plane 和一个或多个 worker node 构成。【事实，Components】
- control plane 的职责是管理集群整体状态。【事实，Components】
- `kube-apiserver` 是暴露 Kubernetes HTTP API 的核心服务端组件。【事实，Components】
- `etcd` 是存放所有 API server data 的一致性、高可用 key-value store。【事实，Components】
- `kube-controller-manager` 运行 controllers 以实现 Kubernetes API behavior。【事实，Components】
- 这形成一个清楚的分层：API 对象为共享可见的状态；controller 读取它并发出变更；底层执行者再报告观察到的状态。【事实，Controllers】
- Kubernetes 并不要求全局状态静止；工作和自动修复可使集群持续变化。【事实，Controllers】
- 系统的成功条件是可运行的控制器持续进行有用变更，而非一次性达到永远不动的终态。【事实，Controllers】

### 控制循环（reconciliation）

- control loop 是一个调节系统状态的非终止循环。【事实，Controllers】
- controller 会 watch cluster state，并在需要处做出或请求变更。【事实，Controllers】
- 每个 controller 试图使 current state 更接近 desired state。【事实，Controllers】
- Kubernetes object 的 `spec` 通常表达 desired state。【事实，Controllers】
- 以 Job controller 为例：它不直接运行 Pod/container，而是要求 API server 创建或移除 Pod。【事实，Controllers】
- Job 完成时，该 controller 将 Job object 标记为 Finished。【事实，Controllers】
- 外部系统 controller 从 API server 取 desired state，直接作用于外部系统后把 current state 报回 API server。【事实，Controllers】
- 因而「命令」不是跨组件的隐藏 RPC 事实，而最好被转成 API server 可见的对象变更／status 回报。【推断】
- Kubernetes 倾向多个各管一个方面的小 controller，而非相互纠缠的单体控制回路。【事实，Controllers】
- 原因之一是 controller 可能失败；小而独立的 controller 容错边界更清晰。【事实，Controllers】
- 多个 controller 可创建同类资源，例如 Deployment 和 Job 都会创建 Pod。【事实，Controllers】
- labels 等归属信息让 controller 能分辨哪些对象由自己负责，避免互相删除资源。【事实，Controllers】

### 声明式与命令式的实际取舍

- 使用者写入 object `spec`，表达「希望是什么」；controller 决定「如何收敛」。【事实归纳，Controllers】
- controller 可以向 API server 提交创建、更新、删除等动作；API 仍是共享边界。【事实，Controllers】
- 这不是没有命令式行为：controller 内部仍需选择步骤、重试、外部调用和补偿。【推断】
- Kubernetes 的关键取舍是：把命令式过程封装在 controller 实现里，而把跨角色协作暴露成声明式、持久的资源状态。【推断】
- 适合声明式接口的目标：可长期维持、可在故障后重算、可由不同 actor 观察的状态。
- 适合命令式接口的目标：一次性、不应重放、需要立即返回过程性结果、调用者必须严格指定步骤的动作。【推断】
- 不应把「所有东西都建成 desired state」当成教条：不可幂等副作用和不可逆外部操作仍须独立命令/操作协议。【推断】

## 核心抽象与接口面（它暴露了哪些概念和 API，尽量给出真实签名或消息格式）

### 资源、读写和版本

- 资源是 API 的中心抽象；内置及 Custom Resource 都可被 client 以标准 API 查询和修改。【事实，Operator】
- 核心组资源路径形如 `/api/v1/...`；非核心组资源形如 `/apis/GROUP/VERSION/...`。【事实，API Concepts】
- 每个对象的 `metadata.resourceVersion` 是 opaque string；客户端不得解析为数字或比较大小。【事实，API Concepts】
- object 修改时 resourceVersion 会变化；它用于乐观并发、list/watch 一致性和 watch 起点。【事实，API Concepts】
- 写入携带过时版本可导致 `409 Conflict`（或一些前提操作的 `412 Precondition Failed`）。【事实，API Concepts】
- 集合 list 响应的 `metadata.resourceVersion` 是启动后续 watch 的正确版本点。【事实，API Concepts】
- 未指定 resourceVersion 的 list 是强一致（quorum）读取，足以作为同步缓存的可靠快照起点。【事实，API Concepts】
- `resourceVersion="0"` 允许从缓存返回 Any 状态，可能是陈旧的。【事实，API Concepts】
- 这一接口不承诺客户端能从版本字符串推导全局时间或排序。【事实，API Concepts】

### watch：状态与事件流的连接点

- 标准 watch 在集合 URI 添加 `watch=1`，并携带已有的 `resourceVersion`。
- 真实请求示例：【事实，API Concepts】
```http
GET /api/v1/namespaces/test/pods?watch=1&resourceVersion=10245
```
- 成功后服务器以流式 JSON `WatchEvent` 返回变化。【事实，API Concepts】
```json
{"type":"ADDED","object":{"kind":"Pod","apiVersion":"v1","metadata":{"resourceVersion":"10596"}}}
{"type":"MODIFIED","object":{"kind":"Pod","apiVersion":"v1","metadata":{"resourceVersion":"11020"}}}
```
- 标准 event type 有 `ADDED`、`MODIFIED`、`DELETED`、`BOOKMARK`、`ERROR`。【事实，API Concepts】
- `DELETED` 的 object 是删除前最后状态。【事实，API Concepts】
- `BOOKMARK` 只携带最新 resourceVersion，不表示资源有变化。【事实，API Concepts】
- 客户端可请求 `allowWatchBookmarks=true`，但服务端不保证 bookmark 的发送频率。【事实，API Concepts】
- `ERROR` 事件通常携带 `Status`，例如历史版本已过期的 410。【事实，API Concepts】
- watch 的价值不是充当独立消息总线，而是从一个可验证快照版本延续状态变化。【推断】
- 断线后客户端以最后看到的 event（可包括 BOOKMARK）的 resourceVersion 重新连接，可避免正常断线造成的事件缺口。【事实，API Concepts】
- API server/etcd 只保留有限变更历史；过旧版本通常会产生 `410 Gone` / `Expired`。【事实，API Concepts】
- 收到 410 的正确恢复是重新 list 全量状态，再从新 list 的版本重新 watch（relist + rewatch）。【事实，API Concepts】
- 因此消费者必须把 event stream 当作可中断的 cache-invalidation / state-sync 协议，而非永不丢失的业务事件日志。【推断】
- 新的 streaming list 可用 `watch=1&sendInitialEvents=true&resourceVersionMatch=NotOlderThan&allowWatchBookmarks=true` 先流式送合成 `ADDED`，再以带 `k8s.io/initial-events-end=true` 注解的 BOOKMARK 划分初始快照结束。【事实，API Concepts】
- streaming list 的意图是减少大型一次性 LIST 的服务端内存压力，同时保有一致性起点。【事实，API Concepts】

### client-go informer：客户端派生视图而非第二真相源

- `client-go/tools/cache` 是用于减少 server call 的 client-side caching mechanism。【事实，client-go cache】
- `Reflector` watch server 并更新 Store。【事实，client-go cache】
- 典型数据流为 Reflector（ListAndWatch）→ DeltaFIFO/queue → process function → Store/Indexer 和 event handlers。【事实归纳，client-go cache】
- 真实构造与运行接口：【事实，client-go cache】
```go
func NewReflector(lw ListerWatcher, expectedType interface{}, store ReflectorStore, resyncPeriod time.Duration) *Reflector
func (r *Reflector) Run(stopCh <-chan struct{})
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error
func (r *Reflector) LastSyncResourceVersion() string
```
- 真实 SharedInformer 接口：【事实，client-go cache】
```go
func NewSharedInformer(lw ListerWatcher, exampleObject runtime.Object, defaultEventHandlerResyncPeriod time.Duration) SharedInformer
func NewSharedIndexInformer(lw ListerWatcher, exampleObject runtime.Object, defaultEventHandlerResyncPeriod time.Duration, indexers Indexers) SharedIndexInformer
```
- 事件回调形状如下；`OnAdd` 的 `isInInitialList` 能区分初始同步与后来新增。【事实，client-go cache】
```go
type ResourceEventHandlerFuncs struct { /* AddFunc / UpdateFunc / DeleteFunc */ }
func (r ResourceEventHandlerFuncs) OnAdd(obj interface{}, isInInitialList bool)
func (r ResourceEventHandlerFuncs) OnUpdate(oldObj, newObj interface{})
func (r ResourceEventHandlerFuncs) OnDelete(obj interface{})
```
- 官方警告：返回的 informer Store 只应用于 Get/List；直接 Add/Modify/Delete 会令 event notification 错误。【事实，client-go cache】
- `WaitForCacheSync(stopCh, cacheSyncs...) bool` 让 controller 在本地 view 完成初次同步后才开始依赖它。【事实，client-go cache】
- resync 会带来 synthetic update/Sync，即使资源没有真实修改；它让处理逻辑有机会重新评估当前状态。【事实，client-go cache】
- `DeletedFinalStateUnknown` 明确表达断线期间可能遗漏删除事件，所带 object 可能陈旧。【事实，client-go cache】
- relist 的 `Replace` 会把本地已有、但新 list 中不存在的 key 合成为删除语义。【事实，client-go cache】
- informer local cache 是可重建的派生物；权威性不应转移到它上面。【推断】

### Operator、CRD 和第三方扩展

- Operator 是一种用 Custom Resource 管理应用及组件的软件扩展。【事实，Operator】
- 官方术语：Operator 是 “a specialized controller used to manage a custom resource”。【事实，Operator】
- 最常见部署方式是向集群添加 CRD 及相关 controller。【事实，Operator】
- controller 通常在 control plane 之外运行，例如作为 Deployment 的 Pod。【事实，Operator】
- Operator 作为 Kubernetes API client，从 control plane 查询自定义资源并通过 API 收敛。【事实，Operator】
- Operator 不需要修改 Kubernetes 核心代码，即可因一个或多个 custom resource 扩展集群行为。【事实，Operator】
- SampleDB 的 controller 可据 custom resource 创建 PVC、StatefulSet、Job；删除时先快照再清理；升级时创建 Job。【事实，Operator】
- 用户可通过标准命令访问 custom resource：`kubectl get SampleDB`、`kubectl edit SampleDB/example-database`。【事实，Operator】
- 官方列出的另一条 API 扩展路径是 API Aggregation Layer / `APIService`，适用于 extension API server。【事实，Operator 相关官方扩展导航】
- CRD + controller 将扩展分成两个可独立演化部分：声明/验证的资源 schema 与解释/执行该资源的 controller。【推断】
- 此分裂是第三方扩展控制面的关键：核心只需承认资源类型和协议，不必知道领域工作流。【推断】

## 它把什么放进核心、什么留给使用者

### 放入 Kubernetes control-plane 核心的部分

- HTTP API、对象资源模型及其版本化读写语义。【事实，Components；API Concepts】
- API server data 的一致、高可用持久存储（etcd）。【事实，Components】
- 以 list snapshot + resourceVersion + watch 实现的状态同步协议。【事实，API Concepts】
- API object 的通用生命周期表达、包括 `spec` 期望状态和可观察的状态更新模式。【事实归纳，Controllers】
- 内建资源的 controller 实现及高可用接管机制。【事实，Controllers】
- 控制器可通过 API server 间接产生副作用，而不是直接互相耦合。【事实，Controllers】

### 留给 controller / Operator 作者的部分

- custom resource 的领域含义、该资源的 desired/current state 映射与具体 reconcile logic。【事实+归纳，Operator】
- 外部系统 API 的调用、凭据使用、超时、重试、补偿和副作用幂等性设计。【事实+推断，Controllers】
- 对领域资源的状态机、健康标准、升级/备份策略与失败处置顺序。【事实+推断，Operator】
- 何时创建哪些底层对象（如 SampleDB 的 PVC / StatefulSet / Job）。【事实，Operator】
- 对哪些对象拥有管理权；labels/owner-like 元数据的归属规范。【事实+推断，Controllers】

### 留给终端用户 / 平台运营者的部分

- 他们希望达到的资源配置，即 `spec` 中的目标。【事实，Controllers】
- 部署哪些 Operator、赋予哪些权限、选择哪些 CRD / API version。【事实+推断，Operator】
- 以 `kubectl` 或其他 API client 查询、编辑 custom resource。【事实，Operator】
- 事件处理的业务消费、告警、UI 展示和人类审批策略。【推断】

## 踩坑与教训

- 不能假定全局稳定：reconcile 的设计应以持续变化和局部失败为常态。【事实，Controllers】
- controller 不应直接管理无法确认归属的同类对象；Kubernetes 以 labels 区分创建者。【事实，Controllers】
- 不要将 `resourceVersion` 当整数或时间戳；它是 opaque string。【事实，API Concepts】
- 不能将 watch 当成永久可靠队列：历史压缩会导致 410，必须支持 relist + rewatch。【事实，API Concepts】
- bookmark 只是降低版本过期概率；没有频率保证，不能当 heartbeat SLA。【事实，API Concepts】
- informer cache 不是可随意写的状态库，写 Store 会破坏通知正确性。【事实，client-go cache】
- 消失的删除事件不总能恢复对象最终细节；`DeletedFinalStateUnknown` 的 object 可能陈旧。【事实，client-go cache】
- 多线程并行 `DeltaFIFO.Pop()` 可能处理同一对象的略不同版本；并发模型必须明确。【事实，client-go cache】
- process function 在 queue 锁下运行的实现约束意味着不应在其中做昂贵 I/O。【事实，client-go cache】
- 周期性 resync 会生成没有真实变更的处理机会，因此 reconcile 必须幂等且不能把每次 update 都解释为业务变化。【事实+推断，client-go cache】
- 过度把命令细节塞入共享状态，会使每个 controller 都要解释编排过程，削弱可替换性；应优先共享意图和已观察事实。【推断】

## 对我们 Rust agent kernel 的启示

### 建议的 kernel 责任边界

- kernel 应拥有一个版本化、可持久化的权威 `AgentRun` / `Task` / `Artifact` 状态模型；前端的内存 store 仅是派生 cache。【推断，类比 API server/etcd 与 informer】
- kernel 应提供“读取一致快照 + 从 revision 订阅后续变化”的单一同步协议，而不是将 snapshot API 与无版本 websocket 事件割裂。【推断，类比 list-watch】
- kernel 应定义 revision 为 opaque token；使用者只能回传/相等比较，不能从中推断顺序或时间。【推断，类比 resourceVersion】
- kernel 应规定版本过期或订阅缺口的标准错误（如 `RevisionExpired`），并规定客户端必须重新 snapshot；不要承诺无限事件留存。【推断，类比 410 relist】
- kernel 应在所有写入上做 optimistic concurrency（`expected_revision`）或明确的 merge 规则，避免多个 agent/UI 盲写覆盖。【推断，类比 resourceVersion】
- kernel 应把每个 reconcile worker / agent loop 设计为可重复执行、可停止重启、幂等地趋向 `desired` 的小控制器。【推断，类比 controllers】
- kernel 应记录由 worker 观察到的事实和执行结果（status），使其他 worker 和前端可从权威状态而非私有内存继续工作。【推断】
- kernel 不应内置 LLM prompt、模型选择、具体工具实现、领域任务分解或 UX 工作流；这些对应 Operator/controller 的可替换领域逻辑。【推断】
- kernel 需要 ownership/lease 或至少 scope 元数据，防止多个 worker 无意接管同一 task 的副作用；这一点类比 labels 归属与 controller 边界。【推断】

### 可考虑的最小 Rust 接口面（设计提案，非 Kubernetes API）

```rust
pub trait StateStore {
    async fn get(&self, key: &ResourceKey) -> Result<Option<Versioned<Resource>>, KernelError>;
    async fn list(&self, query: ListQuery) -> Result<Snapshot<Resource>, KernelError>;
    async fn apply(&self, change: DesiredChange, expected: Option<Revision>) -> Result<Versioned<Resource>, KernelError>;
    async fn watch(&self, from: Revision, filter: WatchFilter) -> Result<EventStream, KernelError>;
}

pub struct Snapshot<T> { pub items: Vec<T>, pub revision: Revision }
pub enum WatchEvent<T> { Added(T), Modified(T), Deleted(T), Bookmark(Revision), Error(WatchError) }
pub enum WatchError { RevisionExpired { requested: Revision }, Transient(/* ... */) }
```

- `list()` 返回 snapshot revision；`watch(snapshot.revision, ...)` 是唯一无缺口的正常接续路径。【推断】
- `apply(..., expected)` 承载并发前提；冲突应是显式可处理结果，而不是静默 last-write-wins。【推断】
- `WatchEvent` 只传播对权威状态的变化；非持久的 token-by-token agent 输出可另设 best-effort stream，不能伪装成可恢复状态日志。【推断】
- kernel 需暴露 cache 同步完成的信号，令 controller/前端知道何时可安全根据本地派生 view 作决策。【推断，类比 `WaitForCacheSync`】
- kernel 还需允许领域插件注册资源类型、schema/validation、reconciler，而不是为每种 agent workflow 增加核心 enum 分支。【推断，类比 CRD + Operator】
- 若初期不实现动态 schema，也至少让 plugin 在 typed resource namespace 中声明其 owned kind 与 reconciliation handler。【推断】

### 前端边界

- 前端负责将用户意图编辑为 desired state / command request、展示 snapshot 和 watch 投影、处理 revision-expired 后的全量刷新。【推断】
- 前端不应成为任务状态的权威所有者，也不应自行综合多 agent 私有事件来猜测最终状态。【推断】
- UI 可以提供即时流式输出，但重连后必须能从 kernel 的持久 status/artifact 重建可信画面。【推断】
- 前端的 optimistic UI 必须以 kernel 的 accepted revision 或 conflict 为准，而不是假定本地写入必定成功。【推断】

## 来源

- Kubernetes, “Controllers”。https://kubernetes.io/docs/concepts/architecture/controller/
- Kubernetes, “Kubernetes Components”。https://kubernetes.io/docs/concepts/overview/components/
- Kubernetes, “The Kubernetes API”。https://kubernetes.io/docs/concepts/overview/kubernetes-api/
- Kubernetes, “API concepts”（resource versions、watch、bookmarks、streaming lists）。https://kubernetes.io/docs/reference/using-api/api-concepts/
- Kubernetes, “Operator pattern”。https://kubernetes.io/docs/concepts/extend-kubernetes/operator/
- Kubernetes, “Extending Kubernetes”（extension patterns/CRD/API aggregation 入口）。https://kubernetes.io/docs/concepts/extend-kubernetes/
- Kubernetes client-go, `tools/cache` package API（Reflector、informer、DeltaFIFO、resync）。https://pkg.go.dev/k8s.io/client-go/tools/cache
- Kubernetes API reference, `CustomResourceDefinition` v1。https://kubernetes.io/docs/reference/kubernetes-api/extend-resources/custom-resource-definition-v1/
- Kubernetes API reference, `APIService` v1。https://kubernetes.io/docs/reference/kubernetes-api/extend-resources/api-service-v1/

### 来源局限

- 本轮未找到或采用可核对的 Kubernetes 作者 postmortem / 单一高热度 GitHub issue 作为主证据；故没有把推测性事故因果写成事实。
- `client-go` 页面版本随模块版本变化；文中 Go signature 以检索时 `v0.36.3` 的 pkg.go.dev 页面为准，使用库时须锁定目标版本核验。
- “API server 是唯一真相源”是便于本研究的架构概括：严格文档事实是 etcd 存储所有 API server data，controller 通过 API server 共享/改变对象状态；外部世界的实际状态仍需 controller 观测并回报。
