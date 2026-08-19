TL;DR：网络深度调研蜂群的通用 kernel 应是「受政策约束的证据生产流水线」：可靠地分派、限额抓取、去重、保全可复现出处、隔离不可信文本并在失败时收敛；研究问题分解、事实判断标准和报告写作仍必须由应用与人负责。

# 网络深度调研蜂群：领域解剖

## 0. 分析边界与真实参照

- 此处的「蜂群」不是让几十个 LLM 自由互聊，而是把检索、页面获取、阅读、取证、反证、编辑拆成受控工作单元。
- 真实产品的深度研究通常会执行多步网页检索、阅读、综合，并输出带来源的报告；这使它更接近小型 ETL/证据审计系统，而非普通聊天。
- 可运行规模是几十个短命 worker 并行，外加少数长期 coordinator、索引器和审稿器；并行度应由域名配额、预算和问题分支数限制，而不能固定为「越多越好」。
- 异构性来自能力，不必来自模型品牌：搜索 API agent、浏览/下载 agent、HTML/PDF/OCR 抽取 agent、主张抽取 agent、检索反证 agent、证据评审 agent、引用编排 agent、报告 editor。
- 外部 Web 是敌对且会变的输入面：robots.txt、验证码、登录墙、动态渲染、付费墙、恶意指令、内容农场、转载与删除都会直接改变协作策略。
- 本文将「可信」拆成可核查的过程属性：主张能定位到版本化原文、支持关系可复查、冲突被显式记录、时效与来源类型符合该问题；不把它误作 kernel 能自动判断的客观真理。

## 1. 参与者与角色

- 任务发起 agent 将用户问题转为研究契约：范围、截止时间、语言、允许来源、预算、交付物和风险等级。
- planner 把契约拆为互斥但可覆盖的问题格，例如「原始公告」「监管文件」「同行评议研究」「反方证据」「时间线」，而不是重复地让每个 agent 搜同一句 query。
- query agent 为格子生成查询式、同义词、语言变体和时间范围；它没有任意发布最终结论的权力。
- search agent 调用一个或多个检索提供方，保存查询、提供方、检索时间、排名和结果 URL，而不只保存标题片段。
- fetch agent 负责 HTTP 获取、重定向链、内容类型、robots/TOS 检查、限速、快照哈希和缓存。
- extraction agent 将网页、表格、PDF、扫描件转换为有页码、段落锚点或 DOM selector 的文本片段。
- claim agent 只产出原子主张，例如「机构 X 于日期 Y 发布 Z」，并附原文片段和限定条件；它不应以完整页面摘要替代证据。
- verifier agent 针对主张寻找独立原始来源、反例、更新版或撤稿；「同一通讯稿被十家媒体转载」只算一个来源家族。
- source assessor 标注来源类别、作者/机构、发布日期、修订线索、利益关联、可访问性和与主张的距离。
- contradiction resolver 汇集同一实体/数值/日期的互斥主张，要求补证、保留争议或按预先声明的规则选用。
- synthesis agent 只能读取已通过最低证据门槛的 claim ledger，生成有 citation ID 的提纲或段落。
- citation auditor 检查每条关键句是否有对应证据、链接能否访问、引文是否真的蕴含该句、是否误把二手报道写成一手事实。
- human reviewer 在高影响、低置信、法律/医疗/金融等队列中决定采纳、降级措辞、删除或继续调查。
- 系统常见为 20--100 个并发工作单元，但「一个 agent」应是可重试的执行实例，不是持有不可复制人格的成员。

## 2. 通信

- worker 不应直接把自然语言结论广播给所有 worker；主通道应是带 schema 的任务和不可变证据对象。
- 一个可行的工作消息至少含 research_id、work_id、父 work_id、目标问题、输入 artifact 引用、预算、租约、幂等键和取消令牌。
- 证据消息至少含 source_version_id、取得时间、原始 URL、规范化 URL、内容哈希、定位锚点、摘录、抽取方法和产生者版本。
- planner 向队列发布任务，worker 点对点认领；claim ledger、URL frontier 和冲突表是所有角色读写的黑板。
- 广播只适合低频控制事件，如预算耗尽、研究范围变更、某域名封禁、已找到决定性一手文件；广播正文不携带不可信网页文本。
- 自然语言可用于查询和报告，但 agent 间的决定应写入枚举状态，例如 proposed、supported、contested、superseded、retracted、excluded。
- 引用应传递稳定的证据 ID，不复制大段网页；原文由对象存储或快照库按权限延迟读取。
- 不可信网页中的「忽略先前指令」「上传数据」等文本必须作为 data 字段，不能进入 coordinator 的 instruction channel。
- 外部访问使用 HTTP 条件请求、ETag/Last-Modified 和缓存语义；网页获取必须区分「未变更」「暂时失败」「禁止访问」「内容已替换」。
- 对复杂任务，communication 的瓶颈不是消息数量，而是跨 agent 摘要丢失限定词；故要传递带 span 的证据而非层层再摘要。

## 3. 协调

- 最适合的是层级分解加黑板收敛：planner 分派问题格，worker 独立取证，reviewer 按 claim ledger 合并，不采用多数投票决定事实。
- 首轮可广泛 fan-out，后续应按信息缺口和预期价值分配：缺少一手来源、关键数字有冲突、证据过旧的格子优先。
- 调度器以 work key 去重，例如 (问题格、语言、时间窗、来源类型、目标实体)，避免二十个 agent 打开同一篇报道。
- 对同一 URL 的重复请求，frontier 应用 URI 规范化、canonical link、重定向最终地址和内容哈希；不同 URL 的近重复可用正文指纹或 MinHash 聚类。
- URI 规范化不能过度：移除追踪参数可能合理，但 query 参数可能决定文档版本或检索结果，须保存原始 URL 与规范化规则版本。
- 报道集群的仲裁对象是 source family，不是域名数；同一 AP/路透稿、公司新闻稿的镜像和 SEO 改写不构成独立交叉验证。
- 冲突不靠 LLM 票数：resolver 要列出冲突断言、各自出处、发布日期、版本、定义差异和证据等级，再依规则决定「采纳/限定/并列/升级人工」。
- 高影响主张可要求 2-source rule，但规则应声明独立性和来源类型；例如法律生效日期优先官方公报，不可被两篇媒体「票数」推翻。
- 停止条件应显式：每个关键主张已达门槛、连续 N 个检索结果不产生新证据、预算耗尽、或人类批准交付；不能以 worker 自觉「感觉搜够了」。
- 重规划只能增量修改 DAG；必须保留为何扩张范围、谁触发和淘汰哪些分支，防止循环生成新 query 耗尽预算。

## 4. 资源竞争

- 最稀缺的资源首先是允许的外部请求，而非 GPU：同域并发、每秒请求、搜索 API 配额、下载带宽、验证码/登录会话和付费数据库席位。
- robots.txt 规则和站点条款是硬约束；scheduler 应在请求前按 host policy、crawl-delay（若声明）、合规身份和授权凭据判定，而非失败后猛重试。
- 第二类资源是 token/context：原页面、PDF、OCR、摘要、工具结果和对话历史都消耗窗口与费用。
- kernel 应为研究、分支、work item、工具调用和模型调用分别设硬上限，并在账本中记录 token、字节、wall time 和货币成本。
- context broker 先向 reader 提供证据卡片（摘录、锚点、元数据、哈希），只有确需语境时才升级读取局部原文或完整文档。
- 大型 PDF/OCR 应分块且保留页码；禁止把整个文档反复塞给多个 agent 造成成本放大和版本漂移。
- 任务租约防止慢 worker 或网络分区时的双重执行；过期工作可由另一 worker 接管，提交端用幂等键去重。
- 优先级队列应区分 deadline、关键性、预计信息增益、来源易失性和人类等待时间；不能只按先到先服务。
- 当某一域名限流时，调度器应切换到其它来源类型、延迟队列或请求人工授权，而非将并发洪峰转移成封禁风险。
- 资源调度政策本身属于应用可配置策略，kernel 提供配额、账本、lease、取消和优先级机制。

## 5. 失败与恢复

- 搜索失败的具体形态包括 API 429、超时、DNS/TLS 错误、结果漂移、地区差异和提供方索引缺失。
- 获取失败包括 403/401、robots 禁止、验证码、JavaScript 依赖、PDF 损坏、链接腐烂、重定向循环和内容类型伪装。
- 阅读失败包括 OCR 数字误读、表格列错位、引用范围漂移、模型漏掉否定词或把作者观点改写成事实。
- 协作失败包括重复执行、worker 崩溃、队列消息至少一次投递、摘要覆盖原证据、过时 worker 在取消后写回、恶意页面提示注入。
- kernel 应把每次外部调用分类为 transient、rate-limited、policy-denied、permanent-not-found、parse-failed、unsafe-input，分别退避、换源、终止或升级。
- 429/5xx 可使用受上限的指数退避和 jitter；403/robots/TOS 不应自动重试，而应成为可审计的不可获取证据缺口。
- 每一个 artifact 先写入内容寻址存储，再原子更新 claim/任务状态；这样 coordinator 崩溃后可从日志和快照重放。
- 取消须为协作式：scheduler 标记取消，worker 在工具调用边界检查；迟到结果可以存为 orphan artifact，但不能自动进入已批准报告。
- 对 reader 输出应做结构验证：锚点能否定位、quote 是否是 source text 的子串、日期/单位能否解析；失败即退回原文或交人工。
- 当验证 agent 不可用时，系统可降级为「仅收集带原文出处的候选材料」，不得标成已核实报告。
- 恢复目标不是保证每个 URL 都成功，而是诚实报告覆盖范围、不可访问来源与尚未解决的冲突。

## 6. 状态与真相

- 全局控制状态应在 durable store：研究契约、工作 DAG、队列状态、预算、租约、访问策略、审计事件和报告版本。
- 证据真相应采用 append-only provenance graph：SourceVersion --supports/refutes/mentions--> Claim --used_by--> ReportSentence。
- SourceVersion 必须把「URL」和「当时获取的字节版本」分开；同一 URL 后来改文不应静默改写旧报告的依据。
- 内容哈希、抓取时间、HTTP 响应元数据与可选网页快照是复现的最低要求；访问受限内容至少记录不可共享的书目元数据与许可状态。
- URL frontier 的已访问集合、任务租约、预算扣减要求强一致或原子 compare-and-set，否则并发会重复花钱和越额。
- 原始 artifact 可最终一致，但 claim 状态转为 supported/published 必须经过版本检查，避免旧 reader 覆盖新版抽取。
- 「最新」不是一个布尔值：每条证据要有出版时间、事件时间、抓取时间、最后验证时间和适用的 freshness SLA。
- 对新闻快报可设小时级刷新，对历史研究可设月级或任务期内冻结；具体 SLA 是应用风险选择，kernel 只执行和暴露它。
- 报告生成固定在 evidence snapshot 上；交付后刷新形成新 report revision，并显示哪些结论因来源更新而变化。
- 置信度不能只取模型自评分；至少分开保存来源可靠性、直接性、独立性、时效、抽取质量、冲突未解程度和人工状态。

## 7. 时间性

- 这是分钟到小时级的异步批处理，而不是飞控式毫秒实时控制；用户可等待，外部网站响应却高度不确定。
- 系统应设研究 deadline、每个请求 timeout、租约期限、缓存 TTL 和报告截止点，所有时间采用可比较的 UTC 时间戳。
- fan-out/fan-in 有节拍：计划完成后并发搜集，搜集达到阈值才启动定向反证，验证栅栏通过才允许综合。
- 不必全局同步等待最慢 URL；关键路径达到证据阈值可继续，迟到低优先级结果在下一报告版评估。
- 但报告最终化需要快照栅栏：此版本中哪些 work 已完成、失败、取消、未启动必须确定，以免正文与引用集合不一致。
- 流式向前端发布时，先发布「正在研究/候选来源/待验证」，不能把草稿断言伪装成最终答案。
- 对突发事件，kernel 需支持取消旧问题格、提升新鲜来源优先级和基于版本的重新验证，而非清空全部历史。

## 8. 人在环

- 人类首先定义问题边界：何谓「可信」、哪些司法辖区/语言/时间、是否允许付费库、哪些结论不可自动发布。
- 人类可在研究计划阶段删改问题格，避免系统因语义歧义进行错误而昂贵的搜索。
- 人类 reviewer 应处理无法由通用规则裁决的事：来源是否独立、统计口径是否可比、措辞是否足以表达不确定性、是否存在伤害或诽谤风险。
- 人类负责受控凭据和付费资源的授权；agent 不应从网页、日志或其他 agent 文本中取得新权限。
- 高风险报告的交接物应是审阅队列，而非仅一篇 prose：关键 claim、证据卡、反证、冲突、未覆盖项、成本和报告差异。
- 人类可以批准、拒绝、标注条件性采纳或要求重查；决定要成为带身份和理由的不可变事件。
- 若无人值守，产品必须明确标为自动汇编，并降低措辞或限制交付范围；kernel 可强制「没有人工签署不能发布」的 gate。

## 9. 观测与追责

- 一次研究的 trace 应可从报告句反查 claim、证据 span、来源版本、抓取请求、使用模型/提示模板/工具版本和审批事件。
- 审计日志应记录谁在何时发起任务、分派、访问哪个受限域名、花费多少、改变了什么状态、何种策略批准或拒绝。
- 关键指标包括 URL 去重率、来源家族去重率、每个被采纳 claim 的独立证据数、未解决冲突数、引用可达率、锚点验证率、重试率、预算耗尽率和人工推翻率。
- 复盘要能回答「错误来自检索盲区、读取/OCR、主张抽取、来源家族误判、推理过度还是编辑错引」，因此每步都不可只留自然语言总结。
- 使用 W3C PROV 类实体、活动、代理及其关系能使 provenance 可移植；存储格式可自定，但语义需明确。
- 日志与原文可能包含查询隐私、版权材料和凭据痕迹，所以审计可追责不等于所有参与者可任意读取。
- 前端应提供从句子到证据的 drill-down、冲突面板、失败解释与版本 diff；这依赖 kernel 暴露稳定事件和 artifact 引用。

## 10. 信任与权限

- 任何来自 Web 的文本、搜索 snippet、PDF metadata、网页链接、其他低权限 agent 摘要均为不可信输入，不能改变系统指令、策略或权限。
- fetcher 的网络访问权限应按域名、方法、字节上限、是否可上传、是否可登录分层；通常只需要 GET，不应拥有写入外站的权力。
- 凭据应由独立 secret broker 按任务发短期、最小范围 token；worker 只拿能力令牌，日志不得记录秘密。
- searcher 不必获得支付数据库、report editor 不必获得任意网络、reviewer 不必读取用户的全部历史研究；按角色和研究 ID 隔离。
- 提交 artifact 时要验证 producer identity、工作租约、schema、内容哈希和政策标签；不能因为「另一个 agent 说已验证」而提升状态。
- 撤权应能立即取消未开始 work、拒绝新 fetch、使短期 token 失效并将进行中的写入标为未批准；已获取数据按保留策略隔离或删除。
- 对跨租户系统，URL/文本缓存、向量索引、模型上下文和可观察日志均是数据泄漏边界，缓存 key 必须含租户与访问政策。

## 开发者最头疼的三件事

1. 证据质量不是检索数量：转载、过期页面、错误 OCR 和模型合理化会形成「很多链接支持同一错话」的幻觉，而独立性、蕴含关系和限定条件难以自动判断。
2. 并发一放大就失控：重复 query、同域限流、上下文复制、无止境重规划、队列重试和慢任务会把几十个 agent 变为费用与合规事故放大器。
3. 可信交付难以复现：网页会变、来源会消失、结论会被后续 agent 覆盖；若没有版本化来源、claim-level provenance 和审计轨迹，就无法答复「这句话为什么可信」。

## 通用 kernel 必须承担的职责

- 提供持久工作 DAG、队列、lease、幂等提交、取消、超时、重试分类和可恢复的 fan-out/fan-in。
- 提供层级预算与并发/域名/字节/token 配额，记录成本并在阈值处停止或降级。
- 提供受策略控制的工具执行边界：域名允许列表、robots/TOS gate、凭据能力、网络/上传隔离和撤权。
- 提供不可变 artifact、内容哈希、版本化 SourceVersion、稳定引用 ID 和 provenance graph，而非只保存 agent 对话。
- 提供共享黑板的 schema、claim 状态机、冲突对象、原子版本检查与 source-family 去重挂钩点。
- 提供不可信内容隔离和结构验证 hook，防止网页注入进入控制面，防止无锚点输出升级为证据。
- 提供事件流、审计日志、可查询 trace、指标和报告快照，使前端和审阅流程能解释与复盘。
- 提供 policy/strategy 插件点，让调用方定义来源规则、停止条件、优先级和人工审批 gate，而非把这些实体判断硬编码进库。

## 注定属于应用层或前端的职责

- 研究问题的拆解、行业术语、哪些来源权威、统计口径、证据阈值、结论措辞及风险判定，是领域认识论和产品责任，不能由通用 kernel 决定。
- query 模板、模型选择、特定搜索/付费数据库适配、OCR 质量模型、来源信誉模型和 near-duplicate 算法参数应作为可替换应用策略。
- 报告的叙事结构、受众语气、可视化、引文格式和交互审阅体验属于前端/应用；kernel 仅提供可溯源的结构数据和事件。
- 人工审批者身份、组织权限矩阵、数据保留地域、法律合规解释和发布责任属于部署者；kernel 只能执行已声明政策并留下证据。

## 最小接口面建议

- `create_research(contract, policy_ref, budget) -> research_id`：建立版本化研究契约和资源上限。
- `enqueue_work(research_id, work_spec, idempotency_key)` 与 `claim_work(capabilities)`：让外部角色以受控 schema 参与，不暴露调度内部实现。
- `submit_artifact(work_lease, artifact_manifest)`：原子登记内容寻址证据及 provenance，不接受无来源自由文本作为已验证事实。
- `transition_claim(claim_id, expected_version, decision, evidence_refs)`：以乐观并发控制管理支持、反驳、争议和审批状态。
- `read_snapshot(research_id, selector)`：向综合 agent/前端提供冻结的证据、冲突、预算和完成度视图。
- `subscribe_events(research_id, cursor)`：为进度、审阅和审计 UI 提供可重放事件。
- `cancel_or_revoke(research_id, scope, reason)`：统一停止任务、撤销能力和标记迟到结果。
- `finalize_snapshot(research_id, gate)`：在 policy/human gate 通过后产出可引用的报告输入快照。

## 来源

- OpenAI，Introducing deep research（多步在线研究、带引文报告的产品参照）：https://openai.com/index/introducing-deep-research/
- Google，Deep Research in Gemini（代理式研究与来源报告的产品参照）：https://blog.google/products/gemini/google-gemini-deep-research/
- RFC 9309，Robots Exclusion Protocol（robots.txt 的标准语义与访问约束）：https://www.rfc-editor.org/rfc/rfc9309
- RFC 9111，HTTP Caching（缓存验证、陈旧性和条件请求）：https://www.rfc-editor.org/rfc/rfc9111
- RFC 3986，Uniform Resource Identifier: Generic Syntax（URL/URI 解析与规范化边界）：https://www.rfc-editor.org/rfc/rfc3986
- W3C PROV-DM（实体、活动、代理与溯源关系模型）：https://www.w3.org/TR/prov-dm/
- Crossref REST API（可用于 DOI 元数据核对与书目信息交叉验证）：https://api.crossref.org/
- NIST SP 800-53 Rev. 5，AU Audit and Accountability（审计事件与可追责控制的参照）：https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final
