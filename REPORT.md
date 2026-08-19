# Agent Kernel 抽库研究报告

> 目标:把 agent kernel(多 agent 协作运行时核心)从代码中抽成 Rust 依赖库,只暴露有限接口,供一个前端使用。
> 研究问题:开发者构建 agent kernel / 多 agent 协作系统时真正关心什么?据此决定 kernel 的职责边界与最小接口面。
> 日期:2026-08-19。全部支撑材料在 `agent-kernel-study/` 目录(文件地图见附录)。

---

## 0. 一页结论

**Agent kernel 的窄腰不是 `Agent/Crew/Graph/Tool/UI` 这些流行概念,而是一个"单 owner、可持久恢复的运行事实机"**:宿主用带显式时间与权限的幂等命令驱动它,用显式 effect 回调连接外界,用"足以恢复的 snapshot + 有序 event"观察它。

接口面收敛为:**5 个宿主操作**(`open / submit / drive / snapshot / subscribe`)+ **3 个宿主端口**(`Store / Policy / Effects`)+ **10 个核心名词**(Kernel / Run / Command / Observation / Effect / Wait / Capability / Budget / Lease / Artifact)。

三条最重要的判定:

1. **kernel 管"事实",不管"智能"。** 26 份独立分析一致把 prompt、角色、planner、评分、成功标准判为应用层——它们跨领域即变,冻结进 ABI 是历史上反复失败的过度设计(FIPA ontology、CrewAI role DSL、BDI plan selection)。kernel 管的是:谁在跑、谁生了谁、什么外部动作以什么授权发生了、结果是什么、钱花了多少、谁在等什么、崩溃后如何恢复——这些是所有领域共享的**运行事实**。
2. **单前端场景下,先做进程内 Rust 库,不做进程分离/RPC。** xi-editor 作者 postmortem 明确否定 core/frontend 进程分离(序列化税、双 cache、race/tearing,"把滚动做对花了数月");Zed 转向单进程。协议化(类 Jupyter/LSP)是未来出现"独立发布的多前端/跨语言/不可信隔离"时才加的 adapter,不是现在的边界。
3. **对抗验证不可省。** 接口提案 v1 被 12 路领域走查全部打出硬缺口,v2 复核又被 5 路全部打出新的横切缺陷,v3 才闭合主路径。每一轮暴露的都是"文档自称能做、公开类型做不到"的自相矛盾——不做逐领域场景走查,这些缺陷会全部漏进实现。

---

## 1. 研究方法与规模

两路并行,极端扇出,长文落盘、只回流压缩摘要:

- **先例路(14 个集群)**:Python 主流框架(LangGraph/AutoGen/CrewAI/OpenAI SDK)、Claude Code 与 Agent SDK、Rust agent 生态、持久执行(Temporal/Restate/DBOS/Inngest)、Actor 系统(OTP/Akka/Orleans/actix)、经典 MAS(FIPA/JADE/合同网/黑板/BDI)、ROS2、核心-前端协议分离(Jupyter/LSP/DAP/CDP)、xi-editor postmortem 与 Zed 对照、嵌入式核心库纪律(SQLite/libgit2/tree-sitter/wasmtime)、微内核与 capability(seL4/WASI)、K8s reconciliation、DAG 编排引擎、Rust API 设计模式(sans-io/UniFFI/Tauri)。每个集群回答同一问题:**它把什么放进核心、什么留给使用者、踩了什么坑**。
- **领域路(12 个系统)**:从简单到极难——多 agent 编码团队、深度调研蜂群、客服分诊、仓储机器人集群、实时派单;地震城市救援(按 ICS 指挥体系)、城市交通数字孪生、全国空域+空中出租车(按 FAA/NASA UTM)、电网黑启动(按真实规程)、自动化港口+全球供应链(按 TOS/EDI)、自主科学实验室(按 A-Lab)、先进芯片 tape-out(按 EDA signoff 流程)。每个按 10 条固定框架解剖(通信/协调/资源竞争/失败恢复/状态真相/时间性/人在环/审计/权限),回答:**开发者最头疼什么,通用 kernel 能扛掉哪些**。
- **收敛与对抗**:关注点聚类(18 簇税系)→ 反模式提炼(30 条带真实代价证据)→ 接口提案 → 12 路领域对抗走查 → 修订(v2)→ 5 路复核 + 税系增补 + 完整性批判 → 修订(v3)→ 缺陷闭合核验。
- 共约 55 个子代理、~200 万 token、650+ 次工具调用,三轮对抗循环。

---

## 2. 开发者关心什么:19 簇关注点税系

横跨两路(先例 P + 领域 D)的簇远比单路可信。边界判定:`kernel` = 内核必须做;`lib-user` = 库使用者用 kernel 原语组装;`frontend` = 前端职责;`app` = 纯应用逻辑;`contested` = 证据冲突。

| # | 关注点簇 | 边界 | 证据 |
|---|---|---|---|
| 1 | Run/Task 生命周期与终态 | **kernel** | 24/24,14P+10D |
| 2 | 稳定身份、父子血缘与因果关联 | **kernel** | 24/24 |
| 3 | 命令—事件—快照一致观察协议 | **kernel** | 24/24 |
| 4 | 持久状态、恢复、重放与版本兼容 | **kernel** | 22/24 |
| 5 | 外部 Effect/Tool 调用边界与事实账本 | **kernel** | 21/24 |
| 6 | 副作用幂等、重试、补偿与 exactly-once | contested | 22/24 |
| 7 | 暂停、Signal、人工输入与审批握手 | **kernel** | 20/24 |
| 8 | Capability、委托、撤销与执行前强制 | **kernel** | 22/24 |
| 9 | 预算、配额、资源预留、并发、背压与 admission | **kernel** | 22/24 |
| 10 | Deadline、Timer、Lease、Heartbeat 与 Fencing | **kernel** | 18/24 |
| 11 | 多 Agent 消息、Spawn/Join、监督与失败传播 | **kernel** | 19/24 |
| 12 | Workflow/DAG/协议 FSM/组织编排 | contested | 20/24 |
| 13 | 版本化 Workspace、Artifact 与 Provenance | **kernel** | 20/24 |
| 14 | 观测权威、质量/新鲜度与领域有效性判定 | app | 14/24,领域偏强 |
| 15 | Agent 认知与业务决策语义(prompt/角色/评分) | app | 24/24 含排除证据 |
| 16 | Store/Executor/Provider/Transport/Sandbox 实现 | lib-user | 24/24 |
| 17 | 展示、交互、视图状态与产品工作台 | frontend | 24/24 |
| 18 | Rust 公共 API、版本化 DTO 与 Sans-I/O 纪律 | **kernel** | 18/24,先例偏强 |
| 19 | 外部现实对账、Unknown outcome 与控制面收敛 | contested(增补) | 港口/实验室补齐后新增 |

两个 contested 簇的裁决方式一致:**kernel 收"机制",外置"语义"**。#6:kernel 记 effect key、attempt、receipt、硬上限,但不承诺 exactly-once(下游幂等与补偿只有应用知道);#12:kernel 只固化 Spawn/Join/Wait 原语,Graph/Crew/Loop 等编排 DSL 外置(先例中它们差异巨大且都在漂移)。#19 是补齐港口/实验室两域后新增的:物理世界的"命令发出后结果不明"(合闸了吗?炉子停了吗?)要求 kernel 有分阶段 outcome(delivered/accepted/observed/confirmed/unknown)与证据追加机制,而"派人去现场看"这类探测动作属于外层。

**最有分量的反向发现**:#15(agent 认知)拿到 24/24 的一致**排除**证据——所有先例和所有领域都表明,把"agent 怎么想"放进 kernel 是最常见也代价最高的边界错误。

---

## 3. 边界怎么切:四层责任 + 库 vs 协议

**kernel 必须管**:命令/状态/事件的原子线性化与幂等;完整恢复投影(崩溃+事件过期后,仅凭 snapshot 就能重新派发、渲染审批、对账 Unknown effect);effect 的 propose→authorize→claim→dispatch→receipt 闭环;capability 的原子核验与撤销;资源集合的全有或全无授予与 effect-lease fencing;硬预算与推进隔离(Control/Interactive/Background)。

**库使用者管**:handler 注册与 `RunSpec.kind` 分发;领域状态、世界模型、新鲜度、空间冲突、优化器、队列、SLA;Store/Policy/Effects 三个端口的实现(数据库、模型、MCP、sandbox);retry 分类、补偿、审批 quorum、跨 kernel saga;多线程 facade 与分片。

**前端只管**:从 Snapshot/Event 投影界面;按 InputRequest 渲染表单;经宿主提交响应。**前端不得成为运行事实源**——删掉前端,kernel 必须仍能正确推进和判终态(这是反模式清单的第一条否决题)。

**明确在库边界之外**(需要时引入共识型控制平面,而不是把它们偷渡进 core):跨 Kernel 顺序/跨分片 fencing、离线 fork 合并/CRDT、硬实时急停(PLC/保护继电器必须走独立安全链路,再向 kernel 补记事实)、历史时光机(`snapshot(as_of)`)、企业 IAM、wire 协议。

**库 vs 协议的裁决标准**:当前"同进程、同版本、单前端"没有 N×M 问题,协议化的税(序列化、双 cache、版本偏差、背压、重连)没有对价。出现以下信号才值得加 `kernel-protocol` adapter:独立发布节奏的多 consumer、跨语言、跨机器、不可信插件隔离。届时借 Jupyter 的"按交互时程分通道"(控制/输入/请求/广播不能退化成一个 send)和 LSP 的 capability 协商 + additive evolution,而不是照搬其 socket 结构。

---

## 4. 最小接口面(v3)与它的演化史

### 4.1 形态

```text
宿主操作(5):  open(Create{now,bootstrap} | Recover) → submit(now, Command) → drive(DriveRequest)
              → snapshot(SnapshotQuery) → subscribe(Subscription)
宿主端口(3):  Store(load/commit/read_events,CAS)  Policy(authorize→Allow/Deny/Ask)
              Effects(execute→Completed|Pending, cancel)
命令族(5):    Run(Start/Spawn/SpawnBound/Finish/Cancel)  Coordinate(Send/Wait/Respond)
              Effect(Propose/Retry/Cancel/Record/对账)  Authority(Grant/Revoke)
              Resource(Define/Retire/Lease/Artifact/Archive)
```

关键设计(每项都能溯源到 ≥2 个簇/领域):显式时间贯穿(kernel 不读系统时钟,`now` 由宿主传入——审计与恢复的前提);Snapshot 是**完整恢复投影**(含 RunSpec、消息、Wait 条件与响应、effect intent/receipt、顶层 capability——事件可过期,snapshot 必须闭合);`AcquireSet` 原子多资源租约(时间窗/共享独占/容量,全有或全无——仓储/派单/港口/电网四域共同要求);`RequiredLease` 把 effect 绑到资源 fence(防"旧控制者的排队命令在失权后执行"——交通/电网/实验室);Capability 带 grantee、可缩权、可撤销、覆盖全部命令类;真 Sans-I/O `KernelCore::apply` + 薄 async `Kernel` facade 双层公开(确定性测试、WASM/FFI 友好);Control/Interactive/Background 工作类(紧急取消不被长物理作业阻塞)。

### 4.2 三轮对抗的演化史(这是本研究最有价值的过程证据)

- **v1 → 12/12 领域全 gaps**。四个自相矛盾被反复击中:声称有 outcome ledger 却没有公共读取路径(effect 结果黑洞);声称 gap→snapshot 恢复,snapshot 却不含 RunSpec/receipt/Wait 条件(恢复投影不闭合);要求生成时间事实却禁止读时钟又没有时间输入;声称 lease 防双控却没有 lease-effect 绑定。
- **v2 修订**接受了跨领域缺口(11 项大改),明确驳回单领域诉求(跨分片共识、世界模型、quorum DSL 等 9 项)——**"只有 ≥2 个领域共同确认才改接口"的纪律防住了接口面失控**。
- **v2 复核 → 5/5 仍 gaps**,但缺陷量级收窄为横切字段缺失:`Grant` 缺 grantee 身份(5 域全中——多主体授权完全无法表达)、取消终态不闭合(4 域)、迟到回执的 attempt 归属(2 域)、Rust 层字面不可用(ID 无 Clone/Copy)等 9 项。
- **v3 修订 + 独立核验**:D1–D8 闭合,D9(长寿命 kernel 的状态回收)部分闭合;剩余 5 个**实现分歧级**开口(见 4.3)。同时接受了批判的改判:公开真 sans-io core、定义监督/失败传播语义、加入受限的 `SpawnBound` 原子原语、做删减检查。

趋势:根本矛盾 → 横切缺失 → 规范歧义。收敛明确,但也要如实报告:公共项计数 v2→v3 净增 39(109→148),删减检查仅删 2 项——**"评审只加不减"的偏置是真实的**,实现阶段应引入 API 预算与负对照(一个"短命、低风险、不需要 durability"的最小应用)做删减压力测试。

### 4.3 已知开口(留给 v4 / 实现阶段)

1. 同一 effect 的 attempt 历史模型:旧 attempt 的迟到 completion 在单一 `completion` 字段模型下无处无歧义落账(应改为 per-attempt 账本)。
2. `PolicyExchange` 未绑定 attempt/WaitId,Retry 后旧审批是否随行不可判定。
3. `SpawnBound` 的 lease 自动绑定规则未定义完全(claims 与首 effect resource 的关系)。
4. 监督关系(ChildSupervision)有 snapshot 投影但无增量事件,旧 cursor 消费者无法重建。
5. 归档(Archive)与全局 CommandId 幂等账本、顶层 capability 生命周期互相冲突——有界状态承诺尚未真正闭合(D9)。

---

## 5. 反模式:30 条"千万不要做"(附真实代价)

全文 `synthesis/anti-patterns.md`,每条都挂着可核验代价(作者 postmortem / 官方重设计 / 规范约束 / 社区 issue,标注证据等级)。五大类与最锋利的几条:

- **不要让 kernel 变成 UI 的另一半**:不保存前端展示 cache 的增量变换、不让正确性等待反向 UI RPC、不为假想多前端过早协议化(xi 三连教训);不把发现期的产品交互冻结成 wire contract;不默认 CRDT/自动合并。
- **不要把编排策略和"agent 智能"固化进 kernel**:不原生固定 Role/Crew/GroupChat/BDI 选择策略;不用 performative 词表冒充可执行契约(模型输出 `Action: X` 不是执行事实——CrewAI 有工具未执行但 trace 完整的真实 issue);不提供无 owner 无 revision 的全局黑板(ROS1→ROS2 参数系统重设计);不用一个 send 承载所有交互时程(ROS Action 的三 service + 两 topic 是付过学费的答案);首版不承诺通用可变 agent graph。
- **不要伪造 durability**:checkpoint/retry ≠ exactly-once(LangGraph/DBOS 官方明示);外部 I/O 不得藏进 replay 代码;step 身份不得随机/位置漂移,版本迁移要显式 `NeedsMigration`(Temporal/DBOS/Inngest 三家共同结论);恢复失败不得静默降级成新 run;关键状态不得只写在 cleanup 回调(Orleans/OTP 都不保证其执行)。
- **不要把流和并发当免费**:无界 fan-out/mailbox/collect 必禁(Airflow 默认 1024 上限的由来);live event stream 不是永久真相(K8s watch 的 410 Gone + relist 模式);provider raw token 不是稳定协议,"最后一个 token"不是终态;HITL 必须是带 request ID/schema/deadline 的一等状态;取消必须走高优先级 control path 且原请求必须闭合(LSP/Jupyter 双证据)。
- **不要把安全押在 prompt 和名字上**:自然语言/tool 名/前端事件不是授权凭据;预批准列表 ≠ 强制白名单(Claude Agent SDK 权限文档的细节教训);child 不得拿 master capability,revoke 不是时间机器(seL4/WASI);hook 不得递归 drive 同一 run(SQLite callback 纪律);durable 约束不得只放在会被 compaction 吞掉的 prompt 里。

文末还有一张"旧概念换新名字"对照表(handoff=合同网、workspace=黑板、durable agent=Temporal step……)和 **15 道 v1 否决题**——任何一题答"是"就不该进稳定面。建议把这 15 题直接做成你们抽库的 review checklist。

---

## 6. 领域解剖的横向发现

**所有领域反复出现的(→kernel)**:恢复后重新派发(崩溃+事件过期是常态而非异常);"命令已发出、结果不明"的对账(从合闸到退款到 EDA license);多主体授权与撤销后拒绝(指挥权移交、海关放行、双人复核);原子多资源占用(订单+司机、岸桥+堆场+AGV、机械臂+炉+XRD);预算与递归 spawn 的硬上限;人工介入点的可恢复表达。

**每个领域都不同的(→app,kernel 碰了就死)**:什么算"新鲜"(GPS 2 秒 vs 论文 2 年)、什么算"成功"(测试通过 vs 相角合格 vs DRC clean)、怎么排优先级(SLA vs 生命安全 vs 流片截止期)、空间/几何冲突(路权格子 vs 4D 航路 vs 电网拓扑)、以及一切领域 schema。

**最难的三个领域教会我们的**:电网黑启动——lease 过期不等于物理停止,fencing 必须绑到 effect 上;空域 UTM——战略去冲突是"外部事务型 effect + 回灌",kernel 不内置几何;芯片 tape-out——长任务的正确模型是"受理即返回 Pending + 事后 Record",不是挂住一个 future 十二小时。

---

## 7. 研究局限(如实呈报)

完整性批判代理给出的意见,其中三条必须认真对待:

1. **语料偏差**:证据全部来自架构文档、规范、postmortem 与研究者构造的领域推演,没有真实开发者访谈、issue 使用频率挖掘、性能 benchmark。税系里"24/24"是同一分析框架对选定语料的编码计数,不等于 24 个独立开发者的声音。**建议**:抽库开工前,用你们自己 kernel 的真实前端需求做一次"第 13 域"走查,比任何外部证据都硬。
2. **样本缺口**:先例缺 A2A/ACP 等 agent 互操作协议、structured concurrency、事务 outbox/saga、Cedar/OPA 政策引擎;领域缺支付/金融账本、医疗隐私、多租户 SaaS,以及**负对照**(短命低风险场景)。当前 12 域偏重安全关键+资源密集,会系统性把 Lease/Capability/Artifact 推成必选——如果你们的前端场景轻量,这三件应设计成可裁剪的 feature。
3. **单源决定**:三类 WorkClass 主要来自 Jupyter 类比;拒绝进程协议主要靠 xi 一个 postmortem;`NeedsMigration + pin/drain` 的充分性缺长期运行实证。这些决定方向大概率正确,但强度不如多源结论,实现时留改判余地。

---

## 8. 下一步建议

1. **先把 15 道否决题过一遍你们现有的 kernel 代码**——这比任何新设计都快地暴露当前边界错在哪。
2. **抽库顺序**:先抽 `kernel-core`(sans-io 状态机:类型 + apply/poll,零依赖,可确定性测试),再包薄 async facade(Store/Policy/Effects 三端口),前端最后接 Snapshot/Event。不要从协议开始。
3. **用 v3 当蓝本而不是圣经**:D1–D8 的修复方案可直接采用;4.3 节的 5 个开口在写代码时裁决(尤其 attempt 账本模型);对照你们真实前端做删减——证据显示评审只会加不会减,减法只能靠真实实现逼出来。
4. **验证方法可复用**:每次接口变更,挑 3–5 个最狠的领域场景(多主体授权撤销、崩溃+事件过期恢复、Unknown 对账、原子多资源、紧急取消)做走查,比通用 review 有效一个量级。

---

## 附录:文件地图

```text
agent-kernel-study/
├── precedents/          14 份先例集群笔记(含来源链接)
├── domains/             12 份领域解剖(10 条固定框架)
└── synthesis/
    ├── taxonomy.md            18 簇税系全文(逐簇证据与判定论证)
    ├── taxonomy-addendum.md   增补核对(第 19 簇提案)
    ├── anti-patterns.md       30 条反模式 + 对照表 + 15 道否决题
    ├── interface-proposal.md  接口 v1
    ├── interface-v2.md        v2(含 12 路走查逐条裁决)
    └── interface-v3.md        v3(含第二轮裁决 + 批判回应,当前最新)
```
