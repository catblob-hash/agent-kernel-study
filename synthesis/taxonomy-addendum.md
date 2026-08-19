# 税系增补检查：港口供应链与自主科学实验室

> 检查对象：在既有 14 个先例 + 10 个领域形成的 18 簇税系上，加入 [D11 港口供应链](../domains/port-supplychain.md) 与 [D12 自主科学实验室](../domains/auto-science-lab.md)。  
> 判定日期：2026-08-18。  
> 方法：沿用主税系的“不变量而非高频需求”“机制与政策分离”“不承诺无法端到端兑现的语义”口径。新领域文件中的“kernel 必须”只视为领域需求证据，不直接视为边界结论。

## 1. 结论摘要

**结论：18 个既有簇没有任何一个需要翻转边界标签；但税系需要结构性增补，不能简单写成“全部 unchanged”。**

需要调整的地方有三项：

1. **新增一个暂定 contested 簇：`外部现实对账、Unknown outcome 与控制面收敛`。** 两个新领域都独立指出：软件恢复成功、命令超时或 lease 到期，并不说明箱子/设备/样品已经停止或处于内核推断的位置。现有 #4 恢复、#5 effect 账本、#6 幂等补偿分别只覆盖了内部状态恢复、调用事实和重复副作用风险，没有完整命名“控制面状态与外部现实发生分歧后如何进入 Unknown、收证、冻结风险动作并收敛”这一义务。
2. **重写 #14 的簇名和边界陈述，但保留 `app` 裁决。** 当前簇把“时间/来源/版本等可验证元数据机制”和“何者权威、多少秒算 stale、confidence 是否足以行动”等领域判定混在一起。D11/D12 对前者要求很强，但对后者也都明确留给领域。因此不应把 #14 整体改成 kernel 或 contested；应把通用元数据搬回 #3/#10/#13/#18，把 #14 收窄为 `观测权威、质量/新鲜度与领域有效性判定`（app）。
3. **扩展 #9 的名称与最小机制范围。** 改为 `预算、配额、资源声明/预留、并发、背压与 Admission`。kernel 可以原子维护由宿主声明的资源 claim/capacity/reservation 以及“任务绑定 + 资源授予”；设备运动学、化学相容性、潮窗、污染矩阵和优化目标仍是 app/lib-user。资源的持续占有、过期与 fencing 仍归 #10，避免把 #9 变成领域调度器。

此外，D11/D12 要求将 #5 的 outcome 从单次 `Result` 加强为可表达 `delivered/accepted/started/observed/confirmed/unknown` 的可演进事实；这是接口细化，不是边界翻转。

按原计数口径，将 D11/D12 加入后，所有明确被二者支持的旧簇均在分母 26 上各增加 2 条领域证据；这只是覆盖更新，不是投票。新增簇的全语料计数不能仅凭两份新文件诚实给出，需回查旧 24 个来源。

## 2. 对三个重点争议的对抗性结论

### 2.1 #6 副作用幂等、重试、补偿与 exactly-once：仍为 contested

新证据没有把它推向 kernel，反而更清楚地证明原来的跨边界裁决正确：

- D11 指出 EDI 可按消息 ID/业务版本重发，但不能通过“再吊一次箱”获得幂等；箱已吊离、落位或设备进入禁区后，补偿首先是领域定义的安全稳定步骤。参见 [D11 失败与恢复](../domains/port-supplychain.md#5-失败与恢复) 及 [SMDG EDIFACT messages](https://www.smdg.org/index.php/edifact-messages/)。
- D12 明确采用“至少一次 + 去重键”，拒绝端到端 exactly-once；已进炉或已消耗试剂的步骤只能先核验现场，补偿可能是隔离、清洁或重做新样。参见 [D12 通信](../domains/auto-science-lab.md#2-通信协议语言点对点还是广播黑板)、[D12 失败与恢复](../domains/auto-science-lab.md#5-失败与恢复)。

因此原拆分不变：kernel 维护稳定 effect key、attempt/claim/outcome、去重 completion、硬重试预算和恢复回灌；lib-user/app 提供下游幂等、retry classifier、补偿操作与人工核验。新增的 Unknown/reconciliation 机制不等于 exactly-once。

### 2.2 #12 Workflow/DAG/协议 FSM/组织编排：仍为 contested

两份领域稿都用强措辞要求 DAG、前置条件、动态分支和 fan-in/fan-out，表面上像是在推动 `kernel`：

- 港口将一个 container move 拆成岸桥、水平运输、场桥的依赖链，并要求滚动 supersede/replan。
- 实验室将实验拆成称量、混料、热处理、冷却、XRD、分析和决策更新的 DAG。

但同两份稿也明确把领域 graph 的内容和求解政策留在外层：港口的船舶稳性、危品隔离、泊位/堆场/AGV 优化，以及实验室的炉温兼容、污染矩阵、批处理收益、BO/acquisition policy 都不能由通用内核推断。D12 暴露草案甚至是 `submit_workflow(spec, ...)`，说明 spec 是输入 artifact，而不是证明某个 Rust DAG DSL 应固化进 ABI。

因此新证据只加强现有两层拆分：

- kernel：稳定 step/task identity、依赖满足的原子事实、Spawn/Join/Wait/Complete、取消、恢复、动态派生的最小机制；
- lib-user/app：DAG/FSM artifact、解释器、条件语言、优先级与重排策略、图迁移政策。

领域需求不能消除 P1/P2/P7/P10/P14 所揭示的 DSL 分裂与 ABI 债，故不翻转为 kernel。

### 2.3 #14 “强事实/弱观测”：app 裁决保留，但必须收窄

D11/D12 是对此簇最强的边界压力：

- D11 要求 owner/source、`observed-at/effective-at/received-at`、版本/因果 token、confidence、expiry，并建议 freshness gate；同时又明确“哪个来源优先”取决于对象类型、海关法定权力和应用规则。
- D12 要求 source timestamp、received time、sequence、freshness SLA、confidence；同时明确 controller、LIMS、原始数据仓和分析数据库各自只对特定事实权威，科学质量门和相鉴定属于应用。

如果维持当前宽泛簇名并只标 `app`，读者可能误解为 kernel 连时间戳、provenance、sequence 和声明式 gate 都不管；如果改为 `kernel`，又会误称 kernel 能判断 OCR 是否可信、海关 release 是否仍有效、XRD 质量是否足够训练 BO。两种都错。

建议精确拆位：

- **归 kernel 机制（并入既有簇）**：
  - #3：source/event/receive time、sequence、乱序/gap、版本化 envelope；
  - #10：deadline/TTL 与宿主声明的 freshness condition 的确定性执行；
  - #13：source/owner/provenance、输入与决策版本冻结；
  - #18：这些字段的版本化 DTO、unknown/conflict 的机器可读表示。
- **#14 保持 app**：谁对哪个字段有权威、staleness 阈值、confidence/质量码含义、冲突优先级、是否足以执行物理/法定动作。

建议把 #14 改名为 **`观测权威、质量/新鲜度与领域有效性判定`**，从而让 `app` 标签与簇内容严格一致。

## 3. 逐簇核对

| # | 既有裁决 | D11/D12 对抗检查 | 结论与建议 | 更新后覆盖* |
|---|---|---|---|---|
| 1. Run/Task 生命周期与终态 | kernel | 两域都要求 accepted、started、observed、confirmed 分离；timeout/失联不能压成 failed，存在 Unknown、Reconcile、Quarantine、不可逆阶段。 | **加强，不翻转。** kernel 维护小型通用控制状态和 certainty；“箱已落位”“科学发现成功”等领域 phase/成功判据仍为 app。避免把所有领域词塞进核心 enum。 | 26/26 |
| 2. 稳定身份、父子血缘与因果关联 | kernel | 港口需跨 BAPLIE/COPRAR、move、ECS、闸口与回报传播 correlation；实验室需 workflow/task/attempt/sample/recipe 关联及样品谱系。 | **加强。** 增补“多方关联/关系边”，但组织、箱号、样品本体仍是外部 ID；不要把单 owner 或单棵父子树当完整领域模型。 | 26/26 |
| 3. 命令—事件—快照一致观察协议 | kernel | 两域都出现异步、乱序、重复、补发、持久订阅、cursor、backpressure；计划、执行和业务确认不可从文本或最后消息推断。 | **强加强。** 将 source/event/receive time 与 outcome stage 纳入版本化协议；大遥测和 UI 流仍可弱化，不能冒充事实流。 | 26/26 |
| 4. 持久状态、恢复、重放与版本兼容 | kernel | 港口要滚动 plan epoch、恢复后对账；实验室要冻结 attempt 输入、recipe/固件/校准/分析版本并支持 schema 演进。 | **加强。** 内核持久化协调语义与 revision；外部设备/LIMS/TOS 的真实状态不能因 journal replay 被“恢复”。这也是新增 reconciliation 簇的切口。 | 24/26 |
| 5. 外部 Effect/Tool 调用边界与事实账本 | kernel | 物理动作展示 `command delivered ≠ accepted ≠ started ≠ observed complete ≠ business/scientific confirmed`；adapter 可能只能返回送达/执行未知。 | **边界不变，接口需加宽。** 单一 `resolve_effect(Result)` 过窄；应允许阶段事件、evidence ref、unknown certainty 和后续 reconcile resolution。executor/设备协议仍外置。 | 23/26 |
| 6. 副作用幂等、重试、补偿与 exactly-once | contested | 两域均拒绝从软件重试推导物理 exactly-once，并要求幂等键、outbox/inbox、attempt budget、补偿 hook 与人工核验。 | **显著加强 contested 原判。** 不翻转；详见 §2.1。 | 24/26 |
| 7. 暂停、Signal、人工输入与审批握手 | kernel | 港口有人为 override、区域暂停、远控接管、双人复核；实验室有 EHS、多签、值班轮换、手动状态修正。 | **加强。** kernel 管可恢复握手、request identity、幂等决定、expiry；谁能批准、N-of-M 条件、值班路由政策和 UI 仍在外层。独立 E-stop/联锁不经过通用 kernel。 | 22/26 |
| 8. Capability、委托、撤销与执行前强制 | kernel | 两域都要求 scope 绑定组织/对象/设备/样品/recipe/version/TTL，并强调已开始物理动作不能被软件撤权撤回；adapter 还需验签/范围。 | **加强，且强化保证边界。** kernel 保证自身 effect 路径上的强制与未来调用 fencing；下游 adapter 的二次验证是 lib-user 义务，物理停止由现场安全层完成。无需改 contested。 | 24/26 |
| 9. 预算、配额、并发、背压与 Admission | kernel | 新证据的主压力是异构命名资源、容量、预约、不可抢占阶段，以及“task binding + reservation”原子提交，不只是 token/queue 上限。 | **范围调整但边界不翻转。** 建议簇名加入“资源声明/预留”。kernel 只原子维护宿主声明的 claim/capacity；可行性、运动学、化学和目标函数外置。 | 24/26 |
| 10. Deadline、Timer、Lease、Heartbeat 与 Fencing | kernel | 泊位/潮窗/截关/设备授权与炉/XRD 长预约都需要 TTL、heartbeat、time window、fencing；两个领域都强调 lease/timeout 到期不等于物理资源已安全释放。 | **强加强。** 增加 `expiry -> suspect/reconciling` 而非自动 free 的语义；港口日历、潮汐、设备安全确认是 app/lib-user。 | 20/26 |
| 11. 多 Agent 消息、Spawn/Join、监督与失败传播 | kernel | agent 范围扩展到组织、设备 adapter、人工队列；某船公司接口或设备 fleet 故障需隔离，子任务依赖失效需传播。 | **加强。** worker/agent 不应等同进程；kernel 管本地身份、mailbox/信封、树与失败事件，不承诺跨组织位置透明或自动选 supervisor。 | 21/26 |
| 12. Workflow/DAG/协议 FSM/组织编排 | contested | 两域强要 DAG/动态分支，同时强烈依赖领域约束与滚动策略。 | **加强 contested，不翻转。** 只固定依赖事实和执行原语，不固定领域 graph DSL；详见 §2.2。 | 22/26 |
| 13. 版本化 Workspace、Artifact 与 Provenance | kernel | 港口需要计划 supersede、消息/回执证据与多权威来源；实验室需要 sample lineage、attempt input snapshot、raw data/analysis/firmware/calibration 版本冻结。 | **强加强。** kernel 保存引用、digest、producer、lineage、decision-time pin；TOS/LIMS schema、谱图、EDI 文件和 blob store 外置。 | 22/26 |
| 14. 强事实/弱观测、新鲜度、置信度 | app | 两域既要求通用 metadata，又明确权威、有效性和门限依赖海关/设备/科学语义。 | **标签保留 app，但簇必须收窄/拆位。** 改名并迁移机制字段，详见 §2.3。 | 16/26 |
| 15. Agent 认知与业务决策语义 | app | 港口优化/稳性/危品/商业优先级与实验 BO/配方/科学 QC/风险判定均高度领域化。 | **最强加强之一。** 即使领域稿说 kernel 要“可解释调度”，score、候选生成、硬约束内容和解释语义仍不能进入 kernel ABI。 | 26/26 |
| 16. Store/Executor/Provider/Transport/Sandbox 实现 | lib-user | EDIFACT/API/PCS/PLC/ECS 与 SiLA/厂商 SDK/OPC UA/LIMS 完全异构；安全 PLC 和硬实时 watchdog 还必须独立运行。 | **强加强。** kernel 定义 adapter 契约、签名/回执/健康 DTO；协议、驱动、数据库、对象存储、OT 安全路径由 lib-user/现场系统实现。 | 26/26 |
| 17. 展示、交互、视图状态与产品工作台 | frontend | 港口地图/堆场图/视频/异常台与实验资源日历/样品谱系/告警均需复杂 UI，但两域都禁止前端保存权威状态。 | **加强。** 前端提交 command/signal/evidence，不直接改 task、lease、release、样品位置或 effect outcome。 | 26/26 |
| 18. Rust 公共 API、版本化 DTO 与 Sans-I/O | kernel | 两域明确要求 schema/version envelope、adapter 边界、cursor、typed errors、外部 artifact reference；异步物理设备进一步反对 core 拥有 socket/thread。 | **加强。** DTO 需表示 unknown、late、superseded、evidence ref 与 staged outcome；不要把 EDIFACT、SiLA、TOS/LIMS 或 PLC 类型泄漏进 ABI。 | 20/26 |

\*“更新后覆盖”只是将 D11/D12 对旧计数各加一；没有重新评价旧来源。#14 为原 14/24 + 两域 = 16/26，#18 为原 18/24 + 两域 = 20/26，其余依此类推。

## 4. 建议新增的第 19 簇

### 外部现实对账、Unknown outcome 与控制面收敛

**暂定裁决：contested。**  
**直接新增证据：D11、D12；既有语料中至少 P12 Kubernetes reconciliation 具有潜在支持，但本次未重审全部 P/D，暂不伪造总计数。**

#### 为什么现有簇容纳不完整

- #4 关心 journal/checkpoint 恢复后内核自身状态是否一致；即使它完全正确，设备或跨公司系统仍可能在断线期间继续动作。
- #5 记录 effect 请求与 outcome；但新领域的核心问题恰是 outcome 一时不可知或来源互相冲突。
- #6 关心能否安全重试/补偿；reconciliation 可能在不重试、不补偿的情况下仅通过取证确认原动作已经完成。
- #14 关心观测是否新鲜、可信和足以行动；reconciliation 还需要一套运行时 case 生命周期、冲突冻结和幂等收敛过程。

#### kernel 最小机制

1. `UnknownOutcome/Reconciling/Quarantined` 等通用非终态 certainty，不把 timeout、heartbeat 丢失或 lease expiry 自动解释为停止/失败/可重用。
2. 可关联原 effect/task/resource 的 reconciliation case ID、expected revision、deadline、责任主体与证据引用。
3. 接受 adapter/人工提交的幂等 observation/evidence，保留冲突而非 last-write-wins，并以版本化 resolution 闭合 case。
4. 在结果不明时阻止冲突性新派工或把资源置为 suspect/quarantined；具体阻断 scope 由宿主声明。
5. 将最终更正写成追加事件，不重写历史；恢复后能够继续未决对账。

#### lib-user/app/frontend 义务

- lib-user 实现查询 TOS/ECS/设备 controller/LIMS/仪器/人工现场的 probe adapter；
- app 判定哪个来源权威、需要哪些证据、怎样安全停靠、补录、隔离、重做或发布更正；
- frontend 展示冲突、证据和裁决表单，但不自行决定真实状态。

#### 为什么标 contested 而不是 kernel

`Unknown` case 的身份、状态、阻断和闭合一致性是 kernel 不变量；但“现实是什么”与“如何纠正”只能由领域系统决定。若把整个 concern 标 kernel，会暗示库能核验集装箱位置、炉状态或样品谱图；标 app 又会使每个 adapter 各自处理未知状态，破坏恢复和并发安全。它与 #6 一样是诚实的跨边界义务。

## 5. 对最小接口面的直接影响

不建议因两个领域就加入港口/实验室专用 API；只需修正窄腰的表达能力：

```text
report_effect_event(call_id, stage, certainty, evidence_ref, idempotency_key)
open_reconciliation(subject_ref, expected_revision, reason, deadline)
submit_observation(case_id, source, observed_at, evidence_ref, idempotency_key)
resolve_reconciliation(case_id, expected_revision, resolution, evidence_refs)
claim_resources(task_id, declared_claims, window, expected_revision)
```

这些是语义草案，不是要求逐字成为 v1 API。尤其：

- `stage` 必须可扩展，不能把港口的四阶段或实验室 phase 固化为 enum；
- `certainty` 至少能表达 confirmed/unknown/conflicted，但领域“科学成功”“法定放行”仍是 typed payload/policy 结果；
- `declared_claims` 是宿主已经判定过的逻辑资源需求，kernel 不运行泊位、路径、化学或 batch optimizer；
- reconciliation 不应成为硬实时 safety loop，PLC/E-stop/设备联锁仍独立于 kernel。

## 6. 新证据只是加强、没有造成新增簇的内容

以下看似新颖，但无需另开簇：

- **安全关键/硬实时控制面**：是 #5/#8/#10/#16 的重要排除项，而非 kernel 功能簇。D11 的 E-stop 和 D12 的 PLC/watchdog 都明确必须独立、经认证并能在网络故障下工作。
- **多方 owner/组织租户**：扩展 #2 的关联和 #8 的 capability scope；不应把完整企业/港口组织模型写入 kernel。
- **决策解释与候选方案记录**：可由 #13 保存版本化 evidence/provenance，由 #15 的 app 产生其领域含义；kernel 不生成解释。
- **资源健康、维护、校准和隔离**：通用状态/lease/fencing 属 #9/#10，健康判定和校准语义属 app/lib-user。
- **死信/隔离队列**：是 #3/#6/#9 的可靠投递与背压机制，不足以单独成簇。

## 7. 最终裁决

- **边界标签翻转：0 个。**
- **既有 contested 保留：#6、#12。** 两个新领域都同时要求 kernel 机制和领域语义，故它们没有被“解决”为单边归属。
- **#14 保持 app，但必须收窄簇定义。** 通用时间/来源/provenance/声明式 gate 机制不能被 `app` 标签误删；真正的权威、质量、staleness 与行动有效性判定仍是 app。
- **新增簇：建议 1 个，暂定 contested。** `外部现实对账、Unknown outcome 与控制面收敛`。
- **范围扩展：#9 加入声明式资源 claim/reservation；#5 outcome 改为多阶段且可不确定。**
- **总体窄腰不变。** 核心仍是身份、生命周期、命令/事件/快照、effect/signal/capability、预算/资源 claim、lease/fencing、版本/provenance 和 adapter 中立 DTO；领域优化、事实权威政策、物理补偿、设备驱动、硬安全和 UI 留在外层。

## 8. 新增来源索引

- **D11** [港口供应链](../domains/port-supplychain.md)：[SMDG EDIFACT messages](https://www.smdg.org/index.php/edifact-messages/)、[DCSA Standards](https://dcsa.org/standards/)、[IMO FAL Convention](https://www.imo.org/en/ourwork/facilitation/pages/falconvention.aspx)、[NIST Cybersecurity Framework 2.0](https://www.nist.gov/cyberframework)。
- **D12** [自主科学实验室](../domains/auto-science-lab.md)：[A-Lab, Nature 2023](https://doi.org/10.1038/s41586-023-06734-w)、[SiLA 2 Standard](https://sila-standard.com/)、[Data infrastructure for materials acceleration platforms](https://doi.org/10.1038/s41578-023-00523-2)、[Self-driving laboratories perspective](https://doi.org/10.1038/s41586-023-06469-5)。
