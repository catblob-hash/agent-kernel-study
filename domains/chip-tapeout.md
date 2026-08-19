TL;DR：芯片 tape-out 不是让 agent 自治地“找答案”，而是在严格版本、权限、签核阈值与截止期下，把可重放的 EDA 计算和可审计的人类工程判断收敛到一份可交付给 foundry 的冻结制造数据包。

# 领域解剖：先进芯片完整 tape-out

## 范围与建模

- 本文把“agent”宽泛地包括：人类职能团队、CI/flow 控制器、EDA 工具批处理、回归管理器、farm 调度器和签核审查服务。
- 它们不是同质 LLM 群体：多数 EDA job 是确定性或受随机 seed 影响的专用程序；人类负责规格和风险决策。
- 一个设计的最终交付物通常是冻结的 GDSII/OASIS 版图数据库及其 tape-out manifest、签核报告和 foundry 所要求的附属表单。
- “signoff”不是一个单一绿灯；它是一组有版本绑定的证据：STA、功耗/IR/EM、DRC、LVS、天线、可靠性、DFM 等。
- 因工艺节点、foundry、IP 和公司流程而异，以下描述的是可迁移的协作机制，不假定每家公司使用同一 EDA 厂商或门限。

## 1. 参与者 / 角色

- 系统架构/RTL 团队持有功能规格、微架构和 RTL；它们发出时钟、功耗域、接口和性能目标的上游约束。
- 逻辑综合团队把 RTL、标准单元库和 SDC 约束综合成网表，并把时序/面积/功耗估计反馈给 RTL。
- 前端验证团队运行仿真、形式验证、UVM 回归、lint、CDC/RDC 和功耗意图检查；失败可阻断网表冻结。
- DFT 团队插入 scan、压缩、MBIST/JTAG 等逻辑，并验证可测性、故障覆盖率、pattern 以及与时钟/电源架构的兼容性。
- 物理实现（P&R）团队执行 floorplan、power grid、placement、CTS、route 和 parasitic extraction，拥有“能否实现”的关键物理事实。
- 模拟/定制电路团队以 SPICE、PEX、Monte Carlo 和版图检查处理 PLL、SerDes、SRAM/IO 等模拟宏；其时间尺度和验证方法与数字 flow 显著不同。
- signoff/物理验证团队运行 Calibre/同类 DRC、LVS、ERC、DFM、antenna；它们对 PDK rule deck 的解释权高于局部实现便利性。
- 时序、SI、功耗完整性和可靠性专家分别处理多模式多角（MMMC）STA、串扰、IR drop、EM、热与老化；这些签核维度会互相牵制。
- 封装/PI 团队处理 bump、RDL、chiplet/3D 连接和 package-board co-design；先进封装时它是 tape-out 的平行关键路径。
- IP 供应方给出 hard macro、LEF/Liberty、模型、使用约束和版本；这些是有许可证、黑盒和兼容性边界的外部参与者。
- PDK/foundry 接口人交付受控 PDK、rule deck、DRM 和提交流程，解释 waiver、例外及 mask deadline；foundry 是最终外部权威。
- CAD/方法学团队维护工具版本、runset、脚本、容器、许可证和回归基线；它把“某次结果”变成可复跑结果。
- 计算农场服务（如 LSF/Slurm 类调度器）不是设计决策 agent，却决定大规模回归和签核 job 何时、在哪台机器上完成。
- 项目经理/技术负责人（TL）维护 milestone、风险表、freeze 规则和升级路径；他们是跨组织冲突仲裁者。
- 高端 SoC 往往有数十到数百名工程师、数十个 IP/子系统和同时排队的数千至数万 CPU-core 任务；实际数量依设计、节点与组织而变。

## 2. 通信

- RTL→综合的正式语言是 RTL、约束 SDC、UPF/CPF 功耗意图、Liberty/LEF/工艺库和版本 manifest，而非自然语言口头承诺。
- 综合/P&R→STA 的核心交换物是 gate-level netlist、DEF、SPEF、SDF、时钟/例外约束、RC corner 和 MMMC 配置。
- P&R→PV 的交换物是 GDSII/OASIS、层映射、cell hierarchy、抽取/验证 runset 与 rule deck；LVS 还需 source netlist。
- DFT 与验证交换 scan-inserted netlist、test protocol、pattern、coverage report 和等价验证结果；版本不匹配会产生“假失败”。
- 模拟宏不是只发一个版图：会同时交付 abstract、GDS、schematic/netlist、Liberty/时序模型、noise/ESD/可靠性限制和集成说明。
- 团队日常用 issue tracker、代码评审、聊天和例会点对点协商；只有冻结、blocker、关键 ECO 与 signoff dashboard 才会广播到项目层。
- 工程真相通常通过配置仓库、设计数据管理系统和对象存储中的不可变 run artifact 传播；邮件附件不是可信发布通道。
- 每个 batch job 通过命令行、环境变量、run directory、配置文件和 scheduler API 与工具/农场通信；kernel 应保留这些机器可读协议，而不试图“翻译”EDA 语义。
- 适合 kernel 的模式是按 block、check、corner 和 ECO 分区的黑板/事件流：结果写入带 provenance 的状态记录，订阅者据此再启动下游检查。
- 不适合 kernel 的模式是让所有 agent 广播原始 log 或 GDS；大对象只传 URI、digest、权限引用及摘要。

## 3. 协调

- 任务分配本质是层级指挥加依赖图，不是市场竞价：chip TL 定义 tape-out date，block owner 对交付负责，CAD/flow owner 决定合格 run 的方法。
- WBS 通常以子系统、IP、签核类别和里程碑分解；“网表冻结→布局冻结→route 冻结→tape-out release”形成门禁。
- RTL freeze 后，功能修改必须走 ECO；physical ECO 要重新验证等价性、时序、功耗和受影响的物理检查，不能只看局部修补。
- ECO 以 change request 表达：问题、影响层级、owner、输入版本、补丁、风险、需要重跑的矩阵、批准人和关闭证据缺一不可。
- signoff 收敛是约束求解/反复迭代：修一条 setup violation 可能恶化 hold、拥塞、IR 或 DRC，因此没有一个独立 agent 能宣布全局完成。
- 多维冲突用预先声明的 signoff matrix、严重度门限、waiver policy 和 designated signoff owner 仲裁，而不是多数投票。
- “waiver”是有到期日、理由、风险接受者和可追溯 rule/version 的显式例外；它绝不能等价于在 dashboard 把失败隐藏。
- 遇到 PPA trade-off，架构/TL 以 product target、风险与时间作决定；工具只产生测量与候选解。
- 临近 tape-out 使用每日或更高频的 bug/ECO triage：按 blocker、影响面、预计 TTR 和 mask deadline 排序，明确单一 DRI。
- kernel 应提供依赖 DAG、状态机、门禁策略求值、人工批准和带作用域的 exception；工艺规则的具体接受阈值必须留在应用策略中。

## 4. 资源竞争

- 最稀缺的资源首先是日历：foundry mask release、MPW shuttle、package/assembly 槽位和外部 IP 交付日期不可由内部 scheduler 延长。
- 其次是 compute：全芯片 DRC/LVS、PEX、IR/EM、MMMC STA、形式验证和海量仿真均可能占用大量 core、内存、fast scratch 与许可证 token。
- EDA license 是独立资源维度；有空 CPU 不表示 job 可运行，工具 feature/token 不够时必须排队或降级。
- 高峰期多个项目会竞争同一 Calibre/STA/仿真 license pool、farm 高内存节点和存储 I/O，调度器需要同时看资源请求与项目优先级。
- 大数据库传输/复制也会竞争 NFS/对象存储带宽；在 tape-out 周期直接复制多份设计可能使本来可用的农场饿死。
- floorplan、routing tracks、power grid 和 bump map 是“空间资源”：IP 宏、block 和电源网络相互挤占，不能由 farm 调度解决。
- 模拟 corner、数字 MMMC corner 与封装 PI 的组合会使检查矩阵指数增长；团队需区分早期筛选角和最终必须签核角。
- 实践中通过 queue/priority、fair-share、reservation、preemption policy、resource class、高内存节点标签和 license-aware wrapper 调度 batch job。
- 最后的 signoff 常设冻结窗口：非必要 job 被限流，关键路径 run 取得较高优先级，避免试验性探索挤掉 release 证据。
- kernel 应有可取消/超时/优先级/配额/租约的 job 抽象，并能为 job 记录所用资源和等待原因；它不应内置某公司农场的 fair-share 公式。

## 5. 失败与恢复

- 常见计算失败包括 scheduler 节点失联、preemption、磁盘满、NFS timeout、许可证 checkout 失败、工具崩溃、内存耗尽和 job 被 walltime 杀死。
- 常见“成功但无效”包括 PDK/rule deck 版本漂移、错误的层映射、输入网表不一致、过期 SPEF、错误 corner 配置、seed 变化和脚本未固定的环境变量。
- 功能失败包括 LEC 不等价、仿真回归、CDC/RDC 违规、scan coverage 下降；物理失败包括 DRC/LVS/antenna、setup/hold、IR/EM 和拥塞。
- 检测不能只依进程 exit code：kernel/adapter 应解析结构化完成标志、artifact digest、输入 manifest 和结果状态；应用层解析工具报表并判定签核语义。
- 瞬态 farm/网络/许可失败可在幂等前提下采用有限次数、退避的重试；重试必须保留原 run 和失败原因，不能覆写证据。
- 对固定输入的确定性 EDA run，失败恢复宜复用 content-addressed cache 或从 checkpoint 恢复；对随机 seed run 必须显式保存 seed。
- DRC/LVS/STA 的真实失败不是可由 kernel 自动“修复”的错误，而是创建 defect/ECO、标注受影响检查、重新排队下游 DAG。
- 当一个 block owner 不可用，TL 指定备援 owner；权限与知识交接必须包括 runbook、当前 manifest、waiver 和未关闭 issue，而不是只转交聊天记录。
- 工具升级或 PDK 更新可使旧结果整体失效；系统应将相应签核状态标记 stale，要求重新签核，而非继续显示 green。
- 极端时间不足时可在经批准的风险流程下减少非最终探索、扩大机器资源或接受正式 waiver；不能跳过 foundry 强制格式/检查。
- kernel 应提供 attempt 记录、幂等键、checkpoint/artifact 引用、重试分类、失活传播与人工接管；ECO 如何改设计是应用/人类层。

## 6. 状态与真相

- 全局真相不是一个 mutable “项目完成”布尔值，而是 design release manifest：每个 IP、RTL/netlist、约束、PDK、tool/container、rule deck 和关键结果的版本/digest。
- source control 管脚本、RTL、约束和配置；大型二进制 layout/数据库通常由设计数据管理或受控存储管理，二者需在 manifest 汇合。
- 每次 signoff 结果必须绑定输入版本、命令/runset、工具版本、license feature（必要时）、host 环境、时间、日志和输出 artifact URI。
- 合格状态的最小粒度应是 `{check, scope/block, mode, corner, input-manifest-digest}`，不能只记录“STA green”。
- 聚合 dashboard 是派生视图，不应是真相源；它要能反查到每项证明和仍未知的 coverage。
- 一致性要求分层：提交/冻结/批准需要强一致且不可抵赖；运行中的进度、队列位置和预计完成时间可最终一致。
- 一个新 ECO 提交后，依赖旧网表/版图的结果应立即失效或标为 stale；旧结果可保留审计但不可计入 release。
- 结果新鲜度具有业务语义：同样是“24 小时前完成”，若其输入早于最新 ECO 就是零价值；kernel 要以 dependency/version 判新鲜，而非仅 wall-clock。
- foundry release 前的签核清单应是冻结快照，任何提交包变动都改变 release identity 并触发重新审批或规定的 delta signoff。
- kernel 应支持版本化实体、不可变 artifact、lineage/provenance、状态失效和原子 freeze snapshot；具体 manifest schema 由芯片流程定义。

## 7. 时间性

- EDA signoff 大多是异步长任务：分钟到天级；没有控制飞行器那样的毫秒闭环实时性，但 deadline 的后果极高。
- RTL/验证与物理实现可并行推进，前提是通过 interface contract、预估宏/约束和阶段性网表解耦；最终仍在冻结点汇合。
- 早期阶段可按 nightly regression 和每周 PPA review 节拍运行；临近签核常转为每次 ECO 自动触发和每天多轮 triage。
- STA/IR/DRC 的收敛常有迭代依赖：route/RC 更新→分析→ECO→重新实现→再分析，kernel 需避免把旧结果误作新闭环的终点。
- 对 foundry deadline 的倒排要显式留出全芯片 signoff、rerun、数据 prep、内部审查、foundry precheck 和提交传输缓冲，而不是只排“最后一次 GDS”。
- 任务 SLA 依优先级不同：开发性探索可延迟；release blocker 的 job 需要有启动/完成预算和升级规则。
- kernel 适宜表达 deadline、not-before、超时、cron/event trigger、critical path 与升级通知；“何时该 risk-tapeout”必须由负责人决定。

## 8. 人在环

- 人类制定规格、SDC/功耗意图的业务语义、PPA target、签核门限和风险接受标准；这些无法由通用编排器从 log 推导。
- block owner 审阅工具报告、定位根因、写 ECO，并确认修复未引入相邻 corner 退化。
- CAD/flow owner 审核 tool/PDK/rule deck 升级及其相关性，防止“工具更新后数字好看但不可比”。
- signoff owner 对各领域最终状态签名，TL/项目负责人对 release manifest、waiver 和提交流程作责任确认。
- foundry 问题、rule 解释、提交格式 exception 需要 designated contact 与正式工单/邮件记录，不能让任意 agent 直接承诺。
- 人类介入点应是高影响不可逆操作：freeze/unfreeze、批准 waiver、接受违例风险、外发 foundry、取消关键 run、变更资源保留。
- 交接必须生成当前状态快照、未决 blocker、责任人、下次动作、到期时间和关联 artifact；kernel 可强制这些字段后才允许 reassignment。
- kernel 应提供 approval queue、双人复核/职责分离、签名/理由、通知和 handoff checklist；人员组织结构及 escalation 名单属应用层。

## 9. 观测与追责

- 每个 run 要可回答：谁/什么启动、为什么启动、使用哪个输入 manifest、在哪个队列/主机、消耗何种资源、产生什么结论、后来是否失效。
- 关键指标包括 per-check pass/fail/unknown/stale、violation count 与严重度趋势、coverage、ECO turnaround、queue wait、run walltime、license wait 和关键路径 slack。
- 日志、命令、配置、tool version、seed、stdout/stderr 和产物 checksum 是再现与事故归因的最低证据。
- dashboard 应显示“最后一次成功”之外的 input age、受覆盖 corner、豁免数和 blocker ownership，否则 green status 容易误导。
- tape-out 后复盘通常关联 escape/返工与当时的 waiver、检查覆盖、rule deck/tool 版本、ECO 历史和审批决策，形成下一项目的流程改进。
- 审计日志必须 append-only 或有防篡改控制；修改结果标签、删 artifact、批准 waiver 与外发数据都应有 actor、时间和原因。
- kernel 应提供结构化 event/audit ledger、trace/correlation ID、artifact lineage 和可查询状态历史；具体 violation parser、PPA 图表和根因分类为应用层插件。

## 10. 信任与权限

- PDK、standard-cell library、foundry rule deck、第三方 IP 和最终 GDS 往往受 NDA/出口/许可证约束；并非所有 agent 能读取或复制它们。
- 需要按项目、block、数据域和动作授予最小权限：验证 job 可读指定输入和写自己的结果，不应可替换 golden PDK 或发布 GDS。
- 外部 IP 可只以 black-box model 或加密视图提供；kernel 的 artifact reference 需要支持可见性/访问控制，而非假设全局共享文件系统。
- 生产农场 submit 权限、优先级提升、license pool、删除大数据、冻结状态改变和 foundry upload 是不同的 capability，不能只用“管理员/普通用户”二分。
- ECO 合并应通过 code review/审批及受保护分支；release manifest 和 waiver 要由指定 role 签署，职责分离可降低单点误操作或恶意变更。
- 凭据（foundry portal、license、存储）应由短期 token/secret manager 注入 job，日志与 artifact 不得泄露凭据。
- 人员离开项目或供应商访问到期时，必须能立即撤销其数据、队列和外发权限；历史审计仍要保留 actor identity。
- kernel 应提供身份、RBAC/ABAC 或 capability、短期凭证委托、审批策略、审计和撤销；NDA 条款、IP 许可解释与 foundry 接口责任留给应用治理。

## 对通用 agent kernel 的结论

### 开发者最头疼的三件事

1. 多维签核的“假绿”：不同团队拿着不同 netlist/PDK/约束/角落的成功报告，无法证明它们共同对应同一个可制造 release。
2. ECO 与截止期的级联：一次局部修改会使哪些签核失效、应重跑哪一矩阵、谁有权接受剩余风险，靠表格和聊天极易漏项。
3. 稀缺 farm、license、存储与人类专家同时成为瓶颈：长任务排队且会失败，项目仍需在不可移动的 foundry deadline 前做可解释的取舍。

### 通用 kernel 必须扛住的能力

- 把工作表示为带输入/输出契约的 DAG，支持 fan-out/fan-in、条件门和从失败/变更向下游的失效传播。
- 为每次执行生成不可变 attempt、幂等键、状态机、超时/取消/重试策略和人工接管点。
- 以 digest 和 manifest 绑定 artifact、配置、工具环境与结果，提供可查询 provenance 和 release snapshot。
- 对资源声明提供队列、优先级、配额、租约、deadline、backpressure 和外部 scheduler adapter，而不假装拥有农场。
- 提供 policy-enforced gate、approval/waiver 对象、职责分离、身份/能力权限及可撤销的凭据委托。
- 提供事件流、审计账本、状态订阅、告警与 correlation ID，让前端能够显示 freshness、阻塞关系和责任人。
- 定义 plugin/adapter 边界，让 EDA 工具解析器、runset、farm、存储、issue tracker 和 foundry portal 可替换接入。

### 注定属于应用层 / 前端的事

- STA/DRC/LVS/IR/EM 的物理含义、rule deck 解释、corner 矩阵、通过门限、waiver 风险模型和 ECO 修复策略。
- 组织层的项目排期、PPA 商业取舍、人员 DRI、升级制度、NDA/IP 合同及对 foundry 的工程承诺。
- 每种 EDA 工具的命令、日志解析、许可证语义和 PDK/设计数据 schema；kernel 只承载其标准化契约。
- 前端应将 kernel 的版本、状态、lineage、队列与批准决策可视化，提供面向角色的 triage/审批体验；它不应成为另一份可被手工改写的真相源。

## 来源

- OpenROAD 项目，端到端 RTL-to-GDSII 流程及各阶段文档：https://openroad-flow-scripts.readthedocs.io/en/latest/
- OpenLane 2 文档，flow 配置、步骤与可复现 run 的开源实例：https://openlane2.readthedocs.io/en/latest/
- Siemens EDA，Calibre 物理验证产品页（DRC、LVS、DFM 等 signoff 类别）：https://eda.sw.siemens.com/en-US/ic/calibre-design/physical-verification/
- Synopsys，IC Validator physical verification（signoff DRC/LVS 与 rule deck 执行的产业实例）：https://www.synopsys.com/implementation-and-signoff/signoff/ic-validator.html
- Synopsys，PrimeTime static timing analysis（多模式多角时序签核的产业实例）：https://www.synopsys.com/implementation-and-signoff/signoff/primetime.html
- Cadence，Voltus IC power integrity（IR drop/EM 功耗完整性签核实例）：https://www.cadence.com/en_US/home/tools/digital-design-and-signoff/silicon-signoff/voltus-ic-power-integrity-solution.html
- Accellera，IEEE 1801 UPF 资源（低功耗意图交换标准）：https://www.accellera.org/downloads/standards/upf
- IEEE Standards Association，IEEE 1801 UPF 标准页：https://standards.ieee.org/ieee/1801/6865/
- IBM Spectrum LSF 文档（batch scheduler 的队列、资源、优先级等机制实例）：https://www.ibm.com/docs/en/spectrum-lsf
- Silicon Integration Initiative (Si2)，OpenAccess/EDA 数据互操作标准与生态：https://si2.org/standards/
