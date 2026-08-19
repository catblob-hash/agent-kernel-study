TL;DR：经典 MAS 已经给出了 message envelope、有限协商协议、共享状态和 deliberation loop 四类可复用原语，但 FIPA 的全套 speech-act/ontology 语义与 BDI 的固定推理策略不宜进入通用 Rust kernel；应保留可观测、可取消、带 provenance 的最小运行时骨架，并把任务评分、领域语义、状态治理和策略留在应用/前端。

## 概览

### 研究范围与证据等级

- 本笔记考察 FIPA ACL / JADE、FIPA Contract Net Protocol (CNP)、blackboard architecture、BDI/AgentSpeak（以 Jason 为可执行先例）。
- `【文档事实】` 表示可由本笔记“来源”中的规范、API 或官方教程直接支持。
- `【推断】` 是结合这些先例，对 LLM agent runtime/Rust library 的设计判断；不是历史文献的原话。
- 研究目标不是复刻 MAS platform，而是识别今天 agent kernel 正在重新需要的稳定机制与应避免固化的策略。
- FIPA 的旧规范站点目前对自动抓取返回 403；字段和 protocol 名称按其公开规范链接及 JADE/Jason 互操作资料交叉核对。对不能从在线正文复核的细节明确标注为“规范记载”。

### 四个先例解决的不同问题

- FIPA ACL：把“消息是什么、谁发的、何时关联、采用什么语义/内容语言”变成标准 envelope。
- CNP：把一次 market-style task delegation 变成有 deadline 和结果收敛的 interaction protocol。
- Blackboard：把多个专长模块的部分结论汇聚到同一 problem state，并用 control shell 决定下一步。
- BDI/AgentSpeak：把单个自主体的 reactive execution 表示成 belief、goal/event、plan 和 intention。
- 【推断】LLM agent 系统的 task inbox、handoff、shared artifact store、planner/executor loop、tool result、run trace，分别是上述四类机制的现代名称。

### 不应混为一谈的层次

- ACL envelope 解决 transport-adjacent metadata 与 conversation correlation，不决定任务如何评估。
- CNP 决定一轮“征询—出价—授予—完成/失败”的状态机，不定义候选人能力如何产生或比较。
- Blackboard 决定共享工作区的协作形态，不自动提供并发一致性、访问控制或正确的调度。
- BDI 规定 reasoning-cycle 中 event/plan/intention 的组织方式；它既是解释器语言语义，也是很强的 runtime policy。
- 【推断】Kernel 若把四层同时当作“一套 agent intelligence”，将难以让前端/库使用者替换任一策略。

## 核心抽象与接口面

### 1. FIPA ACL：结构化的 conversation message

- 【文档事实；FIPA ACL Message Structure】ACL 的抽象 message 以一个 communicative act（performative）加一组命名参数构成。
- 【规范记载】常见 SL/字符串表示如下，字段可省略：

```lisp
(inform
  :sender (agent-identifier :name seller@platform)
  :receiver (set (agent-identifier :name buyer@platform))
  :content "(price book 42)"
  :language fipa-sl0
  :ontology bookstore
  :protocol fipa-request
  :conversation-id c-123
  :reply-with q-17
  :in-reply-to q-16
  :reply-by 2026-08-18T12:00:00Z)
```

- 【规范记载】message parameters 包括 `sender`、`receiver`、`reply-to`、`content`、`language`、`encoding`、`ontology`、`protocol`、`conversation-id`、`reply-with`、`in-reply-to`、`reply-by`。
- 【文档事实；JADE API】JADE 的 `jade.lang.acl.ACLMessage` 是 ACL message 的 Java 表示；官方 API 为各字段提供 getter/setter，并把 performative 表示为常量/整数与 `getPerformative()`。
- 【文档事实；Jason performatives】Jason 的 `.send(Receiver, Performative, Content)` 同样把“内容”与“发送意图（performative verb）”分开；这说明 API 层的三元组足够承载常用 communication primitive。
- 【文档事实；Jason performatives】`askOne` 带第 4 个 reply variable 时会 suspend 当前 intention，直到 reply 到达并与该变量 unification。
- 【推断】Rust kernel 可把 send API 设计为 `send(Message) -> MessageId`，把 request/reply waiting 表示为可取消 future，而不暴露 Java/ACL 的 mutable bean。

### 2. FIPA performatives：把消息意图显式化

- 【规范记载】FIPA ACL 定义大量 communicative acts，例如 `accept-proposal`、`agree`、`cancel`、`cfp`、`confirm`、`disconfirm`、`failure`、`inform`、`inform-if`、`inform-ref`、`not-understood`、`propose`、`query-if`、`query-ref`、`refuse`、`reject-proposal`、`request`、`request-when`、`request-whenever`、`subscribe`、`proxy`、`propagate`。
- 【规范记载】FIPA 用 Semantic Language (SL) 风格的 `feasibility precondition (FP)` 与 `rational effect (RE)` 为 performative 给出心智状态层面的形式语义；这不是网络送达/执行成功的保证。
- 【文档事实；Jason performatives】Jason 实现的是 KQML-style 实用子集：`tell`、`untell`、`achieve`、`unachieve`、`askOne`、`askAll`、`askHow`、`tellHow`、`untellHow`、`signal`。
- 【文档事实；Jason performatives】`tell` 给接收者 belief base 加上 `source(sender)` annotation；`untell` 只撤回同一 source 的对应 belief。
- 【文档事实；Jason performatives】`achieve` 使接收者采用一个带 source annotation 的 goal；`tellHow`/`askHow` 可传递或索要 plan，而不只是 data。
- 【文档事实；Jason performatives】`signal` 直接产生 event；相同 signal 可产生重复 event，这不同于 belief set 的去重直觉。
- 【推断】今天的 LLM systems 正重新发明的不是 20 余个 verbs，而是其中少数可验证的 intent：`notify/inform`、`request`、`offer/accept/reject`、`result`、`failure/cancel`。
- 【推断】将 `tell` 的 provenance 重新命名为 artifact/message attribution 很有价值；将 `tellHow`（远程注入可执行 plan）作为 kernel 默认能力则会扩大 trust boundary。

### 3. CNP：有界的任务委托状态机

- 【文档事实；FIPA Contract Net spec】CNP 有 initiator（manager）与若干 participant（contractor）角色；它规定 protocol messages 的合法次序，而不是选择算法。
- 【规范记载】基础一轮通常是：initiator 广播/多播 `cfp`（call for proposal）→ participant 在 deadline 前 `propose` 或 `refuse` → initiator 对 proposals 发送 `accept-proposal` 或 `reject-proposal` → 获选 participant 执行并以 `inform` 或 `failure` 收束。
- 【规范记载】`reply-by` 是 proposal 的时间界限；各 message 通过 `protocol` 和 `conversation-id` 关联。
- 【规范记载】FIPA 另有 Iterated Contract Net，允许 initiator 在 proposal 阶段进行多轮 CFP/修订，而不应把它误当作基础 CNP 的必需路径。
- 【文档事实；JADE protocol API 文档】JADE 将 interaction protocol 编码为 `Behaviour`，并提供 `ContractNetInitiator`/`ContractNetResponder`；这表明框架把 protocol state machine 放在 runtime library，而把 callback 中的业务决策交给应用。
- 【推断】“委派任务并等待多个候选结果”的 agent orchestration 正是 CNP 的重现；现代差别是 candidate proposal 往往是 tool/cost/context window/模型路由的估计，而非人类式承诺。
- 【推断】最小 kernel primitive 应提供 `broadcast request`、correlation、deadline、每 participant 的 terminal outcome 和 cancellation；“最优 bidder”排序不应固化在核心。

### 4. Blackboard：共享工作区与机会主义调度

- 【文档事实；Nii/Hayes-Roth 综述】blackboard system 的三要素是 blackboard（共享、结构化 problem-solving data）、knowledge sources (KS，独立专长模块) 和 control component/shell。
- 【文档事实；Hearsay-II/blackboard 综述】KS 从 blackboard 上的条件被触发，写入 hypotheses、partial solutions 或其他贡献；这些写入可进一步激活别的 KS。
- 【文档事实；Hayes-Roth BB1 综述】blackboard 强调 opportunistic problem solving：控制器根据当前状态和机会选择下一项工作，而非固定 pipeline。
- 【文档事实；blackboard 文献】控制本身可成为 meta-level knowledge source：根据剩余时间、目标、停滞等信息切换策略。
- 【文档事实；blackboard literature】适用问题常是复杂、信息不完整、不同专长的中间结论可逐步组合的场景；Hearsay-II speech understanding 是经典例子。
- 【推断】共享 artifact/workspace、plan board、event log，以及“新文件出现触发 reviewer”是 LLM multi-agent 的 blackboard 复现。
- 【推断】将 blackboard 简化成无版本的 mutable JSON dict 会丢失它最重要的设计问题：谁能写、冲突如何处理、哪些变更会触发谁、controller 怎样避免 thrashing。

### 5. BDI / AgentSpeak：目标到计划的解释执行

- 【文档事实；Jason 官方教程】belief 来自 perception、communication 或初始 facts；Jason belief 可以带 source annotation，如 `happy(bob)[source(alice)]`。
- 【文档事实；Jason 官方教程】achievement goal 用 `!` 写，例如 `!say(hello)`；intention 是正在执行、以达成 goal 为目的的 instantiated plan。
- 【文档事实；Jason 官方教程】plan 的核心语法是 `+trigger : context <- body.`。
- 【文档事实；Jason 官方教程】`+!g(X)` 可触发 achievement-goal plan；`+belief`/`-belief` 可触发 belief addition/removal plan。
- 【文档事实；Jason 官方教程】context 是针对当前 belief base 求值的逻辑公式；body 为 actions/internal actions/subgoals 的序列。
- 【文档事实；Jason 官方教程】reasoning cycle 将 initial goals 放入 event queue、plans 放入 plan library；每轮选择 event、以 unification 找 matching plan、检查 context、建立 intention 并执行 action。
- 【文档事实；Jason 官方教程】该 tutorial 的默认 plan selection 是第一个 context 成立的 plan；对每个 event 只选一个 plan。没有 applicable plan 时该 intention 以 failure 结束。
- 【文档事实；Jason 官方教程】subgoal 让 parent intention 保持存活；递归 subgoal 会导致不终止 intention。
- 【文档事实；Jason 官方教程】多条 intention 可并发保存，系统仍对新的 event 保持 reactive。
- 【文档事实；Jason TransitionSystem API】`TransitionSystem(Agent, Circumstance, Settings, AgArch)` 聚合 agent、运行时 circumstance、settings、architecture；公开 `sense()`、`deliberate()`、`act()`、`reasoningCycle()`。
- 【文档事实；Jason TransitionSystem API】`runAtBeginOfNextCycle(RunnableSerializable)` 用于将 mutation 安排在下个 cycle 开始，文档说明主要是为了由 TransitionSystem thread 改变 circumstance。
- 【文档事实；Jason TransitionSystem API】GoalListener 可以订阅 goal creation、suspension 等事件；`generateGoalDeletion` 将 intention failure 生成为 failure event。
- 【推断】LLM planner 的 `goal → candidate plan → execute steps → tool observation → replan/cancel` 是 BDI 的操作性重述；belief base 则常被 vector store、workspace 和 run state 拆散。

## 它把什么放进核心、什么留给使用者

### FIPA / JADE 的边界

- 【文档事实】核心：message object、agent identity/address、message transport、conversation metadata、performative vocabulary、standard interaction-protocol state machines。
- 【文档事实】核心：JADE 的 protocol Behaviour 管理消息收发与 transition；应用通过继承/覆写 callback 提供具体 proposal、接受和执行行为。
- 【推断】这条边界是可取的：kernel 实现 protocol mechanics，使用者实现 domain policy。
- 【推断】不可照搬之处：FIPA 还标准化 content language/ontology/mental-state semantics；LLM system 的 schema、tool contract、trust model 演进太快，不宜都放 Rust crate 的稳定 ABI。

### Blackboard 的边界

- 【文档事实】核心：共享数据结构、KS activation 的基本形式、control shell。
- 【文档事实】留给系统设计者：blackboard 的 levels/panel schema、KS 的知识与触发条件、control strategy、solution acceptance criterion。
- 【推断】kernel 应提供 append-only event/artifact ledger 或 versioned workspace abstraction；artifact schema、merge/review rules、retention/visibility policy 属于库使用者/前端。

### Jason BDI 的边界

- 【文档事实】核心：belief base、event queue、plan library、intentions、reasoning cycle、internal actions 和 message semantics。
- 【文档事实】留给 programmer：plan library、context condition、action implementation、环境架构 (AgArch)、selection function 的定制。
- 【推断】目标状态、step lifecycle、observation/event、cancellation 和 a scheduler hook 属于可泛化 kernel；“belief 逻辑”“第一个匹配计划”“如何 replan”“prompt/template”应作为应用 policy。

### 面向前端的边界

- 【推断】前端需要读取而非拥有 runtime truth：agents、tasks/goals、message/correlation、workspace artifact revision、run/step status、failure/cancel reason、trace event。
- 【推断】前端可提交 command（spawn、send、delegate、cancel、approve、write artifact），由 kernel 校验状态转换；不要让 UI 直接修改 intention/event queue。
- 【推断】前端的 layout、chat rendering、human approval UX、token/cost dashboard 和领域表单不是 kernel API。

## 踩坑与教训

### 1. Performative 词汇不能代替可执行契约

- 【文档事实】FIPA 的 `inform`/`request` 等语义以 FP/RE 描述 agent 的信念、意图和效果；它们不意味着接收者实际完成 tool invocation 或结果真实。
- 【文档事实】Jason 选择较小的 KQML practical performative 集，并让 `tell`、`achieve` 等直接映射到 belief/goal/event 的可执行变更。
- 【推断】这是历史上的“语义过宽”教训：kernel 不应只收一个自由字符串 `performative` 就假定互操作。每个关键 command 要有 machine-checkable payload schema、terminal status 和 error/cancel 语义。

### 2. Ontology/content-language 标准化成本常超过收益

- 【文档事实】ACL 分离 `content`、`language`、`ontology`、`encoding`；标准的目标是让接收者知道怎样解释内容。
- 【文档事实】Jason 的常用 API 却以 `.send(receiver, performative, content)` 为主，且其 messaging document 面向实际 effects（belief/goal/plan）解释。
- 【推断】跨团队预先统一 ontology 在开放 LLM app 中通常过度设计；先用 versioned typed JSON/CBOR schema 加 capability/version negotiation。只有稳定跨进程协议才升级为正式 ontology registry。

### 3. CNP 的流程完整性不等于分派质量

- 【文档事实】CNP 定义 CFP、proposal、accept/reject、inform/failure 的交互次序，未规定 proposal utility 函数或 initiator 选择策略。
- 【推断】“多 agent bidding”若没有可靠 capability、成本、排队长度、权限和质量信号，只会把不确定性包装成 `propose` messages。
- 【推断】kernel 要保存可解释 selection evidence；candidate scoring、预算约束、retry/fan-out 为可插拔 policy。不要让 CNP API 暗示竞价已经优化。

### 4. Blackboard 的中心化并不免费

- 【文档事实】blackboard 的 control shell 负责防止 KS 相互干扰，并决定机会主义工作流；控制问题本来就是该架构的一部分。
- 【文档事实】早期 blackboard 系统的效率与 heuristic control 是文献反复讨论的主题；GBB 等系统试图以更高效 pattern matching/多维 blackboard 改善性能。
- 【推断】共享 workspace 会带来 write conflict、stale observation、notification storm、hot key 和 sensitive context leakage。把全局 store 直接放 kernel 但无 revision/ACL/backpressure，会重演 central scheduler bottleneck。

### 5. BDI 的“理性循环”也会卡死或僵化

- 【文档事实】Jason tutorial 明示 recursive subgoal 可造成不终止 intention；无适用 plan 时以 failure 终止。
- 【文档事实】Jason 默认选择第一个满足 context 的 plan，这是一项具体 selection policy，而不是普适最优性结论。
- 【推断】LLM runtime 需让 max steps/time/budget、cancellation、failure propagation、idempotency 和 replan boundary 成为 kernel 级功能；不要把固定 priority 或 LLM planner selection 写死。

### 6. 远程代码/计划迁移扩大安全边界

- 【文档事实】Jason 的 `askHow`、`tellHow`、`untellHow` 支持请求、加入和移除 plan library 中的 plans。
- 【推断】这相当于现代 agent 间共享 executable workflow/prompt/tool recipe；若默认允许，将产生 supply-chain、prompt injection、权限提升和不可重放问题。
- 【推断】kernel 默认传 data references 与 declared capabilities；安装/运行外来 plan 必须经 explicit approval、签名/来源记录和 sandbox policy。

## 对我们 Rust agent kernel 的启示

### 建议纳入核心的最小、稳定接口

- `AgentId`、`RunId`、`TaskId/GoalId`、`MessageId`、`ConversationId`、`ArtifactId`、`RevisionId`：不可变、可序列化 identity；不要以 UI label 作主键。
- `Envelope { id, from, to, correlation_id, kind, payload_ref, deadline, causation_id, trace_context }`：借 ACL 的 correlation/deadline/provenance，去掉 mental-state/ontology 强制语义。
- `MessageKind` 保持窄且可扩展，例如 `Request`、`Offer`、`Decision`、`Progress`、`Result`、`Failure`、`Cancel`、`Event`；unknown kind 必须能安全保存/显示。
- `TaskState`/`RunState` 的显式 terminal transition：至少 `Pending → Running → Succeeded|Failed|Cancelled|TimedOut`，以及有原因的 retry/requeue；CNP/Jason 都说明 failure 必须是一等事件。
- `Protocol` trait 或 data-driven finite-state machine：接收输入、验证当前 state、输出 commands/effects；内建一个轻量 delegation/CNP adapter，而不是硬编码 market。
- `Workspace` trait：append/read/list 与 version/CAS 或 immutable revision；给 write、subscribe、permission、retention 留 extension points，避免“裸 HashMap blackboard”。
- `Scheduler`/`SelectionPolicy` trait：决定哪个 runnable task/event/intention 运行；提供公平、deadline-aware 等实现，但任何“第一个匹配”都不成为语义要求。
- `Event`/trace subscription：让 UI 读到 goal/task/message/artifact lifecycle，且 `command` 与 `event` 分离以支持 replay、audit、debug。

### 建议留给库使用者

- Agent capability 描述、模型/工具选择、CNP proposal 构造与排名、fan-out 数量、预算/quality policy。
- Payload schema 与领域 ontology；kernel 只注册 schema id/version 或保留 opaque bytes/JSON。
- Prompt、plan/workflow 的具体表示；若要共享，使用者定义 trust/approval/sandbox。
- Blackboard artifact 的领域 merge、冲突解决、搜索/embedding、敏感数据分级。
- 失败后的 retry/backoff/escalation、human approval、任务成功判定。

### 建议留给前端

- 对 message/task/artifact trace 的投影、排序、筛选、可视化。
- 用户发起的 command 表单、approval/cancel/override 操作、chat 与文件预览。
- 领域 artifact editor、dashboard、成本展示；前端不能绕过 kernel transition validation。

### 一个足够小的 Rust 轮廓（推断性草案）

```rust
pub trait Kernel {
    fn dispatch(&mut self, command: Command) -> Result<Vec<Event>, KernelError>;
    fn poll(&mut self, now: Timestamp) -> Vec<Event>;
    fn subscribe(&self, after: EventCursor) -> EventStream;
}

pub struct Envelope<P> {
    pub id: MessageId,
    pub from: AgentId,
    pub to: Recipient,
    pub correlation_id: ConversationId,
    pub kind: MessageKind,
    pub payload: P,
    pub deadline: Option<Timestamp>,
    pub causation_id: Option<EventId>,
}

pub trait Workspace {
    fn read(&self, key: &ArtifactKey) -> Result<Revisioned<Artifact>, WorkspaceError>;
    fn compare_and_append(&mut self, write: ArtifactWrite) -> Result<RevisionId, Conflict>;
}
```

- 【推断】这里的 `dispatch` 适合承接前端 command；Event stream 是 BDI listener/blackboard activation 的可观测替代；`correlation_id` 和 deadline 来自 ACL/CNP 的可复用部分。
- 【推断】不要将 `BeliefBase`、`PlanLibrary`、`Ontology`、`performative 的 FP/RE theorem prover` 放进 v1 public API；它们是可选 upper-layer packages。

## 来源

1. FIPA, *FIPA ACL Message Structure Specification*, SC00061（规范，ACL message fields、communicative act 结构与编码）。
   - http://www.fipa.org/specs/fipa00061/SC00061G.html
   - 检索注记：旧 FIPA 站点对本次自动抓取返回 HTTP 403；上文以“规范记载”标出依赖该规范的字段/列表，未把其当作已在线逐段验证的引文。
2. FIPA, *FIPA Contract Net Interaction Protocol Specification*, SC00029（规范，CNP/Iterated CNP roles 与消息序列）。
   - http://www.fipa.org/specs/fipa00029/SC00029H.html
   - 检索注记：旧站点同样返回 HTTP 403；CNP sequence 以“规范记载”标示。
3. JADE documentation, *ACLMessage* API（官方 API，Java message model/performative interface）。
   - https://jade.tilab.com/doc/api/jade/lang/acl/ACLMessage.html
   - 检索注记：本次 TLS certificate verification 失败；不据此声称未直接见到的 overload/实现细节。
4. JADE documentation, *ContractNetInitiator* API（官方 API，FIPA protocol behaviour 的 library 形态）。
   - https://jade.tilab.com/doc/api/jade/proto/ContractNetInitiator.html
5. Jason project, *Jason documentation index*（官方）。
   - https://jason-lang.github.io/jason/
6. Jason project, *Hello BDI tutorial*（官方教程；belief/goal/intention、plan syntax、reasoning-cycle、失败/并发示例）。
   - https://jason-lang.github.io/jason/tutorials/hello-bdi/readme.html
7. Jason project, *List of implemented communication performatives*（官方技术文档；`.send`、KQML performatives 的 operational effects）。
   - https://jason-lang.github.io/jason/tech/performatives.html
8. Jason 3.3.0 API, `jason.asSemantics.TransitionSystem`（官方 API；cycle phases、circumstance mutation、goal listeners/failure）。
   - https://jason-lang.github.io/api/jason/asSemantics/TransitionSystem.html
9. H. P. Nii, *Blackboard Systems*, Stanford Computer Science Department Technical Report STAN-CS-86-1123, 1986（综述性原始技术报告）。
   - https://apps.dtic.mil/sti/pdfs/ADA163137.pdf
   - 检索注记：DTIC 对本次自动抓取返回 HTTP 403；结构性事实以公开综述交叉核对。
10. B. Hayes-Roth, “A Blackboard Architecture for Control,” *Artificial Intelligence* 26(3), 1985, pp. 251–321（BB1/control 的原始论文）。
   - https://doi.org/10.1016/0004-3702(85)90044-4
11. L. D. Erman, F. Hayes-Roth, V. R. Lesser, D. R. Reddy, “The Hearsay-II Speech-Understanding System,” *ACM Computing Surveys* 12(2), 1980（经典应用先例）。
   - https://doi.org/10.1145/356810.356816
12. Blackboard system overview（二级综述；仅用于定位上述原始文献与交叉核对三要素，不将其替代原始论文）。
   - https://en.wikipedia.org/wiki/Blackboard_system
