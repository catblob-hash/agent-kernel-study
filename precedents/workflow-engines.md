TL;DR：Airflow、Prefect、Dagster 都把「可恢复的、可观察的有界工作单元」及其调度放入核心；它们虽以 runtime mapping、Python control flow、pause/suspend 放松静态 DAG，却仍不提供 agent 所需的开放式循环、逐步决策状态、工具副作用去重与交互协议，因而 Rust kernel 应以 durable event/state-machine + lease/scheduler 为核，把 agent policy、LLM loop、UI 和业务补偿留在外层。

## 概览

- 研究对象：Apache Airflow 3.3.x、Prefect 3、Dagster 1.13.x 的官方文档与 Airflow AIP-42 设计提案。
- 这里的「DAG 编排」指：一次运行中的工作单元及依赖关系可表示为一个有向无环图，并由中心服务决定何时把已满足依赖的节点交给执行层。
- 「不适合 agent」不是这些项目的官方统一结论；下文把官方事实与由事实导出的 agent 适配性判断明确分开。
- 三者并非等同的静态 DAG 产品：Airflow 的基准模型最静态，Dagster 支持受限的 runtime fan-out/fan-in，Prefect 直接执行普通 Python flow，图可在运行时长出来。
- 共同点是把一次运行视为可命名、可持久化、可调度和可重试的对象，而不是把一个长期自主对话/推理循环视为一等内核对象。
- Airflow 的核心单位是 DAG、DagRun、TaskInstance；Prefect 是 flow run、task run、state；Dagster 是 job/run/step、op、asset（本笔记聚焦 op/job）。
- 三者都将「编排决定」与「实际算力/进程」分离，只是边界名词不同。
- 这使它们很适合 ETL、批处理、分区历史重跑、可并行 fan-out；也使其持久化模型天然偏向一次性工作单元的终态，而不是 agent 每一步的认知状态。

### 静态 DAG、动态任务图、agent loop 的区别

- 静态 DAG：作者部署前就写定 node、edge、依赖和大致重试边界；调度器只在既有图上找 ready node。
- runtime mapping：上游输出确定后才复制某个已定义 node 或局部子图；它是「展开已有模板」，不是任意创建新的控制流语义。
- 动态 Python flow：运行到 `if`、`for`、函数调用时才产生 task/child-flow run；Prefect 在这点最接近 agent 的控制流。
- agent loop：模型输出会在运行时决定下一工具、参数、是否继续、是否要求人确认、是否创建子 agent；循环次数与图形通常不可预先上界，且决策理由/消息历史本身是业务状态。
- 【推断】前三种可以有动态节点，却不自动等价于第四种；真正差别是「谁拥有 continuation 与决策状态」，以及副作用在重试/恢复时如何去重。

### Airflow 的定位与静态假设

- 【事实】Airflow scheduler 持续监控 DAG 和 task；依赖满足后触发 task instance，并通过 configured Executor 运行 ready task。
- 【事实】scheduler 定期收集 DAG parsing 结果；HA scheduling 使用 serialized DAG 作决定。
- 【事实】调度循环创建所需 DagRun，检查一批 DagRun 的可调度 TaskInstance，受 Pool 和其他 concurrency limit 约束后 enqueue。
- 【事实】因此普通 Airflow DAG 的 authoring interface 是部署时 Python 定义，scheduler 处理的是持久化的 DagRun/TaskInstance，而非运行中 worker 任意改写整个 DAG。
- 【事实，AIP-42 问题陈述】在 Dynamic Task Mapping 出现前，Airflow 可在写 DAG 时定义任意并行度，但不能根据上游 task 的运行时状态改变任务数。
- 【事实】AIP-42 所列旧 workaround 是静态 worker sharding、一个 PythonOperator 监控所启动的 DAG，或不建议的 top-level DAG code；后者会拖慢 parsing。
- 【事实】Airflow 的一般调度语义还以 timetable/data interval 为中心：cron/timedelta 任务在 interval 结束后才触发。
- 【推断】这种 logical-date/partition 优先的模型非常适合补数，不天然表达「同一用户会话下一步由模型决定」的因果时间线。

### Airflow 如何松动：Dynamic Task Mapping

- 【事实】TaskFlow/API 以 `task.partial(...).expand(keyword=value)` 为主；`expand()`/`partial()` 只收 keyword arguments。
- 【事实】`expand` 将 task 创建延后到 runtime；scheduler 在 mapped task 执行前按输入创建 n 个 TaskInstance。
- 【事实】可 mapping 的输入目前是 dict、list，或存放在 XCom 中的这两类上游结果；普通 string 不可直接 mapping。
- 【事实】当前默认 `[core] max_map_length=1024`，超过则 source task 失败；这是明确的展开熔断器。
- 【事实】零长度输入不创建 task，mapped task 记为 `SKIPPED`。
- 【事实】`max_active_tis_per_dag` 限制同时运行的 copies，而且跨该 DAG 的所有 active DagRun 生效。
- 【事实】下游 reduce 接到的是 lazy proxy（`LazySelectSequence`），强制 `list()` 可能昂贵；这避免 scheduler/worker 无意将大扇出结果全部装入内存。
- 【事实】不能嵌套 mapping 到 mapped task group；文档明确说这是有意不支持，理由是 UI complexity。
- 【事实】task-generated mapping 不允许 `TriggerRule.ALWAYS`，parse 时即报错。
- 【事实】AIP-42 要求 mapped 输入是有限 iterable，单项须能放入一个 worker 内存；提案还专门设计最大 map size 防失控/组合爆炸。
- 【事实】AIP-42 的 scheduler-side 实现用 `TaskMapping` 元数据使 scheduler 不必读取大 XCom，再扩展 TaskInstance；说明动态性仍需被数据库可索引地物化。
- 【推断】Airflow mapping 是严格受限的「数据驱动并行化」，不是 agent 可在每一步创建任意节点、任意回边或自定义 continuation 的图编辑 API。
- 【推断】把 LLM 输出直接作为 `expand()` 的输入会把模型不稳定的长度、格式和扇出转成调度器负荷；1024 上限只是一道资源闸，不是 agent policy。

### Airflow retry、backfill、人工输入

- 【事实】TaskInstance 生命周期含 `scheduled`、`queued`、`running`、`success`、`failed`、`up_for_retry`、`deferred`、`awaiting_input` 等状态。
- 【事实】`up_for_retry` 的定义是失败但尚有 retry attempts，将被重新调度。
- 【事实】默认重试按固定次数和 delay，且不区分 error type。
- 【事实】`retry_policy=` 可传 `ExceptionRetryPolicy(rules=[RetryRule(exception=..., action=..., retry_delay=..., reason=...)])`。
- 【事实】retry policy 在 task worker process 评估、不会在 scheduler 评估；首个命中 rule 胜出。
- 【事实】policy 可 `RETRY`、`FAIL` 或 `DEFAULT`；它可以提前失败，但不能超过 `retries` 配置的最大次数。
- 【事实】`AirflowFailException` 优先于 retry policy；`AirflowSkipException` 标记 skipped。
- 【事实】Airflow backfill 通过 CLI/REST 为过去日期建 run；无 time-based schedule 的 DAG 不适用。
- 【事实】典型命令为 `airflow backfill create --dag-id tutorial --from-date ... --to-date ... --reprocess-behavior failed --max-active-runs 3`。
- 【事实】`--reprocess-behavior` 可为 `none`、`failed`、`completed`；运行中或 queued 的该 date 不会另建 run，无论选择哪种。
- 【事实】backfill 的 `--max-active-runs` 独立于 DAG-level `max_active_runs`。
- 【事实】`awaiting_input` 被定义为 human-in-the-loop state，既不占 worker slot 也不占 triggerer。
- 【推断】Airflow 已能把「等待人工」作为一个调度状态，但 agent 所需的 approval payload schema、审计、超时升级、拒绝后的策略跳转仍需 task/operator 或外部系统自行定义。
- 【推断】把 backfill 语义套到 agent 会话十分危险：同一 logical interval 的「再跑」不表示工具调用可安全重放；该差异要求 agent kernel 有 effect/idempotency 模型。

### Prefect：以普通 Python 运行时松动 DAG

- 【事实】Prefect flow 是 `@flow` 装饰的 Python function；直接调用该函数即可建 flow run。
- 【事实】官方明确 flow 不必调用 Prefect task/flow，但可以调用；故普通 Python 条件、循环、库调用都可在 flow body 中执行。
- 【事实】调用 task 或 nested flow 时运行时创建对应 run；nested flow 的 parent-child link 分别记录在 task run/child flow run 的 `state_details` 中。
- 【事实】这允许 conditional business logic：某些 task group 仅在条件成立时运行，图由运行时决定。
- 【事实】tasks 被描述为 quick、cacheable、retryable、有 transactional semantics 且易并发；flows 偏 composition、deployment、server-side interaction。
- 【事实】常见并发接口为 `task.submit()`，future 可 `.result(raise_on_failure=False)`，并可 `task.submit(wait_for=[x])`。
- 【事实】每个 nested flow 创建自己的 task runner，nested flow 完成时 runner 关闭；不同 nested flow 可选不同 runner。
- 【事实】同步 nested flow 阻塞 parent；async nested flow 可由 AnyIO task group 或 `asyncio.gather` 并发。
- 【事实】flow 支持 configurable retry limits/delays 和 timeout。
- 【推断】Prefect 是三者中最能承载「一个 agent loop 写在 Python 中」的引擎，但 loop 的 checkpoint、记忆压缩、工具幂等键和模型决策协议仍不是 flow/task 的通用一等类型。

### Prefect 执行面与暂停

- 【事实】work pool 是 Prefect orchestration layer 与实际 flow infrastructure 的 bridge。
- 【事实】hybrid pool 由用户 worker poll 并提交运行；push/managed pool 不需用户 worker（self-hosted server 支持的是 hybrid/worker 类型）。
- 【事实】部署创建 Deployment 并引用 remote flow code，worker poll pool 的 scheduled run，拉取 code 后在目标基础设施 launch flow run。
- 【事实】work queue 有 priority 与 concurrency；优先级为 waterfall，较高优先 queue 先被 drain。
- 【事实】交互 API 有 `pause_flow_run(wait_for_input=...)`、`suspend_flow_run(wait_for_input=..., timeout=...)`、`resume_flow_run(flow_run_id, run_input={...})`。
- 【事实】`wait_for_input` 可为 builtin type、Pydantic `BaseModel` 或 `RunInput`；builtin type 会包成字段名固定为 `value` 的模型。
- 【事实】`suspend_flow_run` 会退出 flow 并拆除 infrastructure；恢复时继续执行设计依赖于该 runtime 的持久化 state，而非保留原进程。
- 【事实】UI 先做 JSON schema validation，但 custom Python validator 要到 resume 后才运行；未捕获 `ValidationError` 会令 run failure，文档建议 catch 后循环再次 pause。
- 【事实】`receive_input(...)`/`send_input(...)` 允许无需 pause 的 typed message exchange；发送早于监听也会在类型匹配的 receiver 出现时交付。
- 【事实】为避免 suspend 后重复处理输入，官方例子将 seen input keys 存入以 flow-run id 命名的 Prefect Variable，并通过 `exclude_keys` 恢复。
- 【推断】这暴露出 durable agent 的关键要求：恢复不是保存 Python call stack；每个可恢复边界要有可重建状态、input cursor 和明确重复投递策略。
- 【推断】kernel 应提供 typed signal / command、ack/cursor 与 pause reason；不要承诺序列化任意语言调用栈。

### Dagster：受约束的 runtime graph 与 executor

- 【事实】Dagster dynamic graphs 用 `DynamicOut` 声明 op 的动态输出，并 yield `DynamicOutput(value, mapping_key=...)`。
- 【事实】典型构图为 `pieces = load_pieces()`、`results = pieces.map(compute_piece)`、`merge_and_analyze(results.collect())`。
- 【事实】`map` 在 runtime 将 downstream ops 按每个 dynamic output clone；`collect` 建立所有 copies 的 fan-in，给下游 list。
- 【事实】在 `@job` composition function 中，dynamic output 不能直接传给普通 op，必须先 `map` 或 `collect`。
- 【事实】`mapping_key` 是每个 DynamicOutput 的唯一标识，适合把展开实例同数据片段关联。
- 【事实】文档以「单块 op 某部分失败需整块重启」为动机，主张切分为 dynamic outputs 来缩小失败重做范围。
- 【事实】每个 job 有 executor；默认 `multi_or_in_process_executor`，默认每 step 独立 process，也可配置 single process。
- 【事实】executor 只负责一个 job run 内的 steps；deployment layer 先分配并启动 run worker，之后 executor 接手 step execution。
- 【事实】可在 `@dg.job(executor_def=dg.multiprocess_executor)` 或 `Definitions(executor=...)` 设定；job 显式设置优先于 code-location default。
- 【事实】内建 executor 包含 in-process、multiprocess，以及 Dask、Celery、Docker/K8s、ECS 等 integration。
- 【事实】官方警告 custom executor 的 internal APIs 仍在变动且文档不足。
- 【推断】Dagster 的 dynamic graph 与 Airflow mapping 同样是 clone/map/collect 模型；它放松的是扇出规模，不是允许 runtime 形成循环依赖或将 agent 的整个思考树作为可变图。

## 核心抽象与接口面（它暴露了哪些概念和 API）

| 系统 | 定义/图 API | 运行与并发 API | 持久化控制面 | 人工/恢复接口 |
| --- | --- | --- | --- | --- |
| Airflow | DAG、Task、TaskInstance；`.partial().expand()` | scheduler → Executor；Pool、TI concurrency | DagRun、XCom、serialized DAG、TaskMapping | `awaiting_input` state；backfill CLI |
| Prefect | `@flow`、`@task`、普通 Python call | `.submit()` / future / task runner；work pool/queue | flow/task run、state、deployment | `pause_flow_run` / `suspend_flow_run` / `resume_flow_run`；RunInput |
| Dagster | `@op`、`@job`、`DynamicOut`/`DynamicOutput`、`.map/.collect` | run worker → Executor；`executor_def` | job run、dynamic mapping key | 本次深读来源未验证特定 HITL API |

- 【接口比较的推断】三者最稳定的公共边界不是「把任意函数调度出去」，而是 `definition + run identity + dependency/state + execution adapter + persisted event/result`。
- 【接口比较的推断】对 Rust library 而言，可学这种分层，但不应把 Python decorator、DAG serialization 或 infrastructure provisioning 当成核心 ABI。

## 它把什么放进核心、什么留给使用者

### 放进核心的共同职责

- run identity、状态转移、ready 判定、并发/优先级/资源限制。
- failure 后重新排队、延迟重试、超时/僵尸 run 检测或恢复边界。
- 调度控制面与执行 substrate 的适配边界。
- 运行历史、日志/状态可观察性，以及按定义重建/重跑的最小元数据。
- 有界数据依赖的传递与 fan-out/fan-in 的实例身份。

### 明确留给使用者/部署者的职责

- DAG/flow/job 的业务定义，任务代码和业务数据 schema。
- worker/Executor 实际运行的位置、凭据、网络与云资源选择。
- task 的幂等性、外部系统的事务/补偿，以及重试是否安全。
- Prefect 的 approval 表单之外的授权流程、人工角色、审批审计与业务语义。
- Dagster custom executor 的稳定性保障；官方已提示该扩展 API 仍变动。
- 【推断】agent 的 prompt、模型路由、memory policy、工具 allowlist、子 agent 委派/合并策略，均应属于 library consumer 或可插拔 policy，而非通用 DAG scheduler 内核。

## 踩坑与教训

- 【AIP-42 设计共识】不要伪装 Python 控制流：AIP-42 拒绝看似 `for` 但语义并非 Python for 的设计，选择显式 mapping API；agent kernel 也应让「spawn、wait、retry、signal」在 API 上可见。
- 【官方文档事实】mapping cardinality、每项内存和 concurrency 必须受限；否则动态 fan-out 可压垮 scheduler、metadata DB 或 worker。
- 【官方文档事实】Airflow lazy reduce 防止一次性 load 全部 mapped outputs；agent kernel 的 message/result collect 也要流式、分页或配额化。
- 【官方文档事实】Airflow mapping 有 UI-complexity 导致的 nested mapping 缺口；图可视化不是免费附属品，任意 agent spawn tree 需要独立的树/事件视图，而非强塞 DAG view。
- 【官方文档事实】Airflow retry policy 虽能按 exception 分流，却在 worker 决定且不能突破 max retries；「调度重试」不能替代对外部 effect 的 exactly-once。
- 【官方文档事实】Prefect suspend 后要自行保存 seen input key 避免重放；任何 input/mailbox 模型都需 cursor、ack 与 idempotency key。
- 【官方文档事实】Airflow backfill 不会为仍 running/queued 的 logical date 再建 run；历史重跑是具有并发和去重规则的操作，不是简单 loop。
- 【推断】agent 把 tool call 重放成两次发邮件/扣费的风险高于典型 read/transform task；kernel 至少应记录 effect intent、attempt、idempotency key 与 outcome，而不能只保存 `Failed`。
- 【推断】「DAG 不适合 agent」准确说法应是：若只提供静态依赖、batch retry 和时间分区 replay，则不足；若使用 Prefect 式动态 flow，仍需另补 durable agent semantics。

## 对我们 Rust agent kernel 的启示

### 建议最小核心（kernel owning）

- `RunId` / `AgentId` / `StepId` / `AttemptId`：稳定身份，所有 event、log、effect 和 UI 命令可关联。
- 持久化 state machine：至少 `Ready`、`Leased/Running`、`WaitingSignal`、`RetryAt`、`Succeeded`、`Failed`、`Cancelled`；状态应以 append-only event 或带版本的 CAS 转移保存。
- ready queue + lease：scheduler 仅选取已满足条件且资源允许的 work；executor/worker 只领取 lease 并回报结果，二者可替换。
- `Spawn`/`Join` 作为显式、可配额的运行时图增长原语；只保证 parent-child lineage 和 join policy，不把任意循环硬编码为 DAG edge。
- typed durable `Signal`：request id、schema/version、payload、deadline、sender/role；支持 UI/外部系统投递及 ack/cursor。
- retry record：failure class、attempt、not-before、retry decision；由 policy hook 决定下一步，但固定上限、预算和取消由 kernel 强制。
- effect journal：至少 `EffectIntent{idempotency_key,...}` 与 outcome，供 host 在调用工具前后原子式记录/恢复；无法实现原子性时明确 at-least-once。
- resource/accounting hook：并发、token/费用、工具调用数、spawn depth/width 和 payload 大小的 hard limit。

### 建议有限 Rust 接口面（方向性签名，非已实现 API）

```rust
pub trait Store {
    fn append(&self, run: RunId, expected_version: u64, events: Vec<Event>) -> Result<Version, Conflict>;
    fn lease_ready(&self, worker: WorkerId, limit: u32, ttl: Duration) -> Result<Vec<Lease>, StoreError>;
}

pub enum Command {
    Spawn { parent: StepId, spec: ChildSpec, idempotency_key: String },
    Complete { step: StepId, output: Value },
    Fail { step: StepId, class: FailureClass, detail: Value },
    WaitForSignal { step: StepId, request: SignalRequest },
    RecordEffect { step: StepId, effect: EffectIntent },
}

pub trait Policy {
    fn on_failure(&self, ctx: FailureContext<'_>) -> RetryDecision;
    fn admit_spawn(&self, ctx: SpawnContext<'_>) -> Result<(), Denied>;
}
```

- 【推断】`Store` 的 CAS/version 比「只给 callback」更关键：多 scheduler、worker timeout、UI resume 与重复投递会并发竞争同一个 run。
- 【推断】`Executor`/`Worker` trait 应接收可序列化 step specification 和 lease，不应强制 kernel 知道 Tokio task、container、HTTP server 或前端框架。
- 【推断】前端应是 command/signal producer 与 event projection consumer：展示 timeline、待审批卡、agent tree、retry/effect 状态；它不拥有调度真相。
- 【推断】LLM agent adapter 在 kernel 外实现循环：读取 event/state → 选下一 action → 提交 `Command`。这样可替换模型、prompt 与 agent framework，不污染 Rust core。
- 【推断】先支持 parent-child tree + signal wait，不急于声称通用 mutable graph；DAG engines 的 dynamic mapping 已显示图增长、可视化和重放会快速复杂化。

### 不应放进第一版 kernel

- prompt 模板、provider SDK、模型选择/temperature、memory summarization 算法。
- 业务工具的 OAuth、HTTP schema、补偿语义和领域审批规则。
- Airflow 式 cron/backfill/timetable；可作为 consumer/controller，避免把时间分区模型耦合进 conversation runtime。
- Kubernetes/Docker/云队列的具体部署实现；只做 worker/executor adapter trait。
- 所有 agent graph DSL、可视化画布、Python/JS bindings；这些应建立在稳定 command/event ABI 之上。

## 来源

- [Airflow Dynamic Task Mapping（官方，3.3.1）](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/dynamic-task-mapping.html)
- [Airflow Scheduler（官方，3.3.1）](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/scheduler.html)
- [Airflow Tasks / states / retry policy（官方，3.3.1）](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html)
- [Airflow Backfill（官方，3.3.1）](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/backfill.html)
- [AIP-42 Dynamic Task Mapping（Apache 设计提案与讨论）](https://cwiki.apache.org/confluence/display/AIRFLOW/AIP-42+Dynamic+Task+Mapping)
- [Prefect Flows（官方，v3）](https://docs.prefect.io/v3/concepts/flows)
- [Prefect Work Pools（官方，v3）](https://docs.prefect.io/v3/concepts/work-pools)
- [Prefect Interactive Workflows（官方，v3）](https://docs.prefect.io/v3/advanced/interactive.md)
- [Dagster Dynamic Graphs（官方，1.13）](https://docs.dagster.io/guides/build/ops/dynamic-graphs)
- [Dagster Run Executors（官方，1.13）](https://docs.dagster.io/guides/operate/run-executors)

### 来源性质与置信度

- Airflow/Pefect/Dagster 的 API、状态与限制均来自对应官方文档；AIP-42 是设计提案，文中已将其与现行 `.expand()` API 区分。
- 「DAG 编排不适合 agent」及 Rust 边界建议为跨来源推断，不应误读为任何一家产品方的官方立场。
- 未找到并深读足以代表「业界普遍认为」的作者 postmortem 或高热度 issue；因此没有把该社会性断言作为事实使用。
