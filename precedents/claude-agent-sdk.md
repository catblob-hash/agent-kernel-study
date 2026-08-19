TL;DR：Claude Agent SDK 将「带工具/权限/上下文与持久会话的 Claude Code agent loop」作为可复用内核，以异步事件流和可更新的控制面给 CLI/IDE/自建应用；任务编排策略、审批 UI、凭据、部署隔离及业务状态则留给宿主。

## 概览

- 【文档事实】Agent SDK 是 Python 与 TypeScript 的库，运行在调用方进程中，并打包 native Claude Code binary。
- 【文档事实】它明确声称复用驱动 Claude Code 的同一 agent loop、built-in tools 与 context management。
- 【文档事实】SDK 与 CLI 的产品边界是：CLI 面向交互式终端；SDK 让应用嵌入 loop；Client SDK 则只给 API、loop 由使用者自己实现。
- 【推断】这是一种「一个执行内核，多宿主壳层」：官方 CLI 与 SDK 并非只共享模型 API，而是共享 tool protocol、session transcript、权限求值和 compaction 行为。
- 【文档事实】非 TS/Python 语言可用 `claude -p --output-format json` 驱动同一 loop，说明 process/JSON protocol 也是其跨语言接口。
- 【文档事实】SDK 的入口 `query()` 既可接受一次性 `string` prompt，也可接受 `AsyncIterable<SDKUserMessage>`；后者支持运行中的多轮输入。
- 【文档事实】`query()` 返回 `Query extends AsyncGenerator<SDKMessage, void>`，所以前端/宿主消费的是可顺序迭代的事件而不是同步的“最终答案”。
- 【推断】这比把“chat completion”作为主要接口更适合多前端：CLI 打印事件、IDE 渲染 tool timeline、Web 转发 SSE，都不必重写执行循环。
- 【文档事实】运行开始后是固定循环：prompt+system prompt+tools+history → assistant response/tool calls → SDK 执行工具并回填 results → 直至 assistant 无 tool calls。
- 【文档事实】一个 tool-use round trip 算一 turn；`maxTurns` 计的是这类 turn，默认不限。
- 【文档事实】`maxBudgetUsd` 限制估计花费；子 agent 的请求也计入父 query 的总成本。
- 【文档事实】最终 `ResultMessage` 即使在限制/错误时也带 `session_id`、`num_turns`、usage、cost，方便观测和恢复。
- 【推断】内核提供可计量且可终止的执行单元，而“一个工作是否值得继续、何时重试/升级模型”应由应用 policy 决定。

## 核心抽象与接口面（它暴露了哪些概念和 API）

### 1. Query / session / command control

- 【文档事实】TS 基本签名：
```ts
function query({ prompt, options }: {
  prompt: string | AsyncIterable<SDKUserMessage>;
  options?: Options;
}): Query
```
- 【文档事实】`startup({options?, initializeTimeoutMs?}) -> Promise<WarmQuery>` 可预热 subprocess，后续 `warm.query(prompt)` 避免首个请求初始化成本。
- 【文档事实】`Query` 是 async generator，同时是实时控制对象，而非只读流。
- 【文档事实】其控制方法包括 `interrupt()`、`close()`、`setPermissionMode(mode)`、`setModel(model?)`、`getContextUsage()`、`mcpServerStatus()`、`setMcpServers()`、`reconnectMcpServer()` 和 `stopTask(taskId)`。
- 【文档事实】`interrupt()` 与 `setPermissionMode()` 是 streaming-input mode 专用；控制面可在一个 session 尚未结束时改变后续请求的权限/模型。
- 【文档事实】还可 `rewindFiles(userMessageId, {dryRun?})`，但须显式 `enableFileCheckpointing`。
- 【文档事实】`Query.reinitialize()` 会重新派发 pending permission requests；文档要求 `canUseTool` 对同一 request ID 幂等。
- 【推断】“事件流 + 命令式 control handle”是值得保留的双向 kernel API；只给 event stream 会使交互前端无法取消、升级权限或恢复连接。

### 2. 事件协议：面向 UI、日志和恢复的稳定边界

- 【文档事实】核心消息种类为 `SystemMessage`、`AssistantMessage`、`UserMessage`、`StreamEvent`、`ResultMessage`。
- 【文档事实】`SystemMessage{subtype:"init"}` 提供 session metadata、可用 tools、MCP server 状态；setup/session hooks 的 lifecycle events 可先于它到达。
- 【文档事实】`SystemMessage{subtype:"compact_boundary"}` 在 automatic compaction 后发出。
- 【文档事实】其他 system subtype 有 `informational` 和 `worker_shutting_down`；后者表示 host exiting 或 Remote Control disconnected 后 loop 将于本 turn 结束。
- 【文档事实】每个 Claude response 都发 `AssistantMessage`，其中有 text blocks 与 tool-use blocks。
- 【文档事实】每次工具执行后发 `UserMessage`，承载会送回模型的 tool result；外部用户在运行期间送入的输入也以此类消息出现。
- 【文档事实】`includePartialMessages: true` 后还会产生 raw API `StreamEvent`，包含 text delta、tool input chunks 等。
- 【文档事实】`ResultMessage.subtype` 至少区分 `success`、`error_max_turns`、`error_max_budget_usd`、`error_during_execution`、`error_max_structured_output_retries`。
- 【文档事实】`result` 字段只在 `success` variant 可读；不要仅因拿到 Result 就停止迭代，因为之后可能还有 `prompt_suggestion` 等 system event。
- 【文档事实】TS 中 assistant/user 的原始 API content 在 `message.message.content`，不是 `message.content`。
- 【文档事实】可选 `forwardSubagentText` 将子 agent text/thinking（附 `parent_tool_use_id`）转发到外层；默认 false。
- 【文档事实】可选 `agentProgressSummaries` 让 `task_progress` 事件带 summary。
- 【推断】主 UI 不应解析模型自然语言来猜执行状态；应把这些 typed events 视为渲染、审计和可观测性的唯一可靠输入。
- 【推断】事件需有 `session_id`、agent/task parent correlation、turn/seq 与 terminal reason；Claude 的 `parent_tool_use_id` 是很好的最小关联键先例。

### 3. Tools：内核调度，不等于把业务能力硬编码

- 【文档事实】内置工具覆盖 files（Read/Edit/Write）、search（Glob/Grep）、execution（Bash）、web（WebSearch/WebFetch）和 orchestration（Agent/Skill/AskUserQuestion/TaskCreate/TaskUpdate）。
- 【文档事实】单一 assistant response 中的多个 read-only tool calls 可并行；Edit/Write/Bash 等有状态操作顺序执行以避免冲突。
- 【文档事实】custom tool 默认顺序执行；MCP `readOnlyHint` annotation 可使其获并行调度资格。
- 【文档事实】`tools` option 能限制本 session 的 tool set；`toolAliases` 可把 built-in 名称路由到 MCP 实现。
- 【文档事实】TS 可通过 `tool(name, description, zodSchema, handler, extras?)` 定义 tool，并用 `createSdkMcpServer({name, version?, instructions?, tools?, alwaysLoad?})` 在宿主进程内提供 MCP server。
- 【文档事实】tool annotations 包括 `readOnlyHint`、`destructiveHint`、`idempotentHint`、`openWorldHint`；它们是调度/风险提示而非安全授权本身。
- 【文档事实】structured output 可由 `outputFormat: {type: "json_schema", schema: JSONSchema}` 请求；无有效输出超过重试上限时终止为 `error_max_structured_output_retries`。
- 【推断】Rust kernel 的 tool trait 宜抽象为“schema + invoke + execution hints + result”，而具体 filesystem/shell/web/业务 tools 应在 adapter/应用层注册。

### 4. Subagent：隔离上下文的 Agent tool，而不是共享内存线程

- 【文档事实】programmatic subagent 用 `Options.agents: Record<string, AgentDefinition>` 定义；模型通过 `Agent` tool 选择/启动它们。
- 【文档事实】典型 AgentDefinition：
```ts
{
  description: string, prompt: string,
  tools?: string[], disallowedTools?: string[], model?: string,
  skills?: string[], memory?: "user"|"project"|"local",
  mcpServers?: (string|object)[], maxTurns?: number,
  background?: boolean, effort?: "low"|"medium"|"high"|"xhigh"|"max",
  permissionMode?: PermissionMode
}
```
- 【文档事实】`description` 供主模型决定何时委派，`prompt` 是子 agent system prompt，`tools` 是存在性限制而非仅审批提示。
- 【文档事实】省略 `tools` 会继承可用于 subagents 的工具；列出后，未列工具根本不进该 subagent context，模型也不会收到 permission prompt/error。
- 【文档事实】子 agent 有 fresh context window：收到自己的 system prompt、Agent tool prompt、project CLAUDE.md（取决 settingSources）、可用 tool definitions。
- 【文档事实】它不继承 parent conversation/tool results，也不继承 parent system prompt；parent 唯一显式传入内容是 Agent tool 的 prompt string。
- 【文档事实】因此子 agent 中间 tool trace 不污染 parent；默认只有 final message 回 parent，作为 Agent tool result。
- 【文档事实】子 agent 也可再生子 agent；默认最大嵌套深度为主 agent 之下 3 层，`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH=1` 禁止子 agent 再 spawn。
- 【文档事实】默认最多 20 个并发 subagents，`CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS` 超限时工具结果为 `Concurrent subagent limit reached`。
- 【文档事实】`maxBudgetUsd` 到限会拒绝新的 subagent、停止仍运行的 background subagents，并以 `error_max_budget_usd` 结束 query。
- 【文档事实】Claude Code v2.1.198 后未显式 `run_in_background` 的 Agent call 默认 background；模型若需结果才会请求 `run_in_background:false`。
- 【文档事实】可通过 assistant tool block 名称 `Agent`（兼容旧版 `Task`）检测 spawn；子 context 的 stream messages 携带 `parent_tool_use_id`。
- 【文档事实】subagent final result 含 `agentId` 时，后续可在同一 parent session `resume` 后让该 agent 继续；子 agent transcript 独立持久。
- 【文档事实】子 agent final message 传给 parent 前会扫描/中和 control-tag imitation 与 `Human:`/`Assistant:` turn-marker imitation，但不会删除或改写一般文本。
- 【推断】kernel 应内置“上下文隔离、层级 ID、并发/深度/总预算硬限、cancellation propagation”；“何时 fan-out、角色/提示词、结果 synthesis”属于应用 workflow policy。

### 5. Hooks：内核生命周期的可编程拦截点

- 【文档事实】hooks 可来自 `options.hooks` 的 app callback，也可来自 settingSources 开启后 settings files 中的 shell hooks。
- 【文档事实】配置形式：`hooks: { PreToolUse: [{matcher: "Write|Edit", hooks: [callback], timeout?}] }`。
- 【文档事实】matcher 对不同 event 匹配不同 target；tool hook 对 tool name，MCP tool 名式 `mcp__<server>__<action>`。
- 【文档事实】callback 签名概念为 `(input, toolUseId, {signal}) => Promise<HookOutput>`；公共输入含 `session_id`、`cwd`、`hook_event_name`，在子 agent 内带 agent_id/type。
- 【文档事实】所有 hook output 可带 `systemMessage` 与 `continue`（Python 叫 `continue_`）。
- 【文档事实】`PreToolUse` 可用 `hookSpecificOutput` 给 `permissionDecision: allow|deny|ask|defer`、reason、`updatedInput`。
- 【文档事实】`defer` 会结束当前 query，使宿主可稍后 resume；多个 hooks 合并时优先级为 deny > defer > ask > allow。
- 【文档事实】`PostToolUse` 可注入 `additionalContext`，或以 `updatedToolOutput` 替换任意 tool 的模型可见输出。
- 【文档事实】共用 hooks 有 PreToolUse/PostToolUse/PostToolUseFailure/UserPromptSubmit/Stop/SubagentStart/SubagentStop/PreCompact/PermissionRequest/Notification。
- 【文档事实】TS 还公开 PostToolBatch、UserPromptExpansion、MessageDisplay、StopFailure、PostCompact、PermissionDenied、SessionStart/End、Setup、TaskCreated/Completed、Elicitation、ConfigChange、InstructionsLoaded 等，Python 尚未全部支持。
- 【文档事实】hooks 在应用进程运行，不占模型 context；PreToolUse deny 使模型收到拒绝的 tool result，进而可改用其他办法。
- 【推断】hooks 应成为内核的事件/策略扩展 API（尤其 pre-action、post-action、pre-compaction），不要让前端靠截获文本间接实现审计或 guardrail。

### 6. Permission：固定优先级的 capability decision pipeline

- 【文档事实】每次 tool request 的规则顺序固定：hooks → deny rules → ask rules → permission mode → allow rules → `canUseTool` callback。
- 【文档事实】PreToolUse 的 allow 不会跳过 deny/ask；PreToolUse hooks 才是必须覆盖所有 tool call 的 guardrail 点。
- 【文档事实】`allowedTools` 是 auto-approve list，不是可见工具白名单：未列工具仍在模型 context，继续走 mode/callback。
- 【文档事实】`disallowedTools:["Bash"]` 会在 prompt 中移除 tool；`disallowedTools:["Bash(rm *)"]` 保留 Bash 但在每种 mode（含 bypass）拒绝匹配命令。
- 【文档事实】`allowedTools` 的 MCP wildcard 必须是具体 server 前缀，如 `mcp__github__*`；通配 `*` 或 `mcp__*` 会被忽略并 warning。
- 【文档事实】mode 为 `default`、`dontAsk`、`acceptEdits`、`plan`、`auto`、`bypassPermissions`。
- 【文档事实】`dontAsk` 将无预批的调用硬拒绝且不调 callback；与 `allowedTools` 组合才是 headless 受限 agent 的推荐方案。
- 【文档事实】`acceptEdits` 仅自动批准受 workspace/additionalDirectories 限制的 file edits 与常见 filesystem commands；其他 Bash 仍按普通路径。
- 【文档事实】`plan` 不自动批准编辑或写文件 shell command，即使 allow rule 匹配也会走 callback。
- 【文档事实】`bypassPermissions` 自动批准所有到该步骤的工具，不是只批准 allowedTools；TS 另需 `allowDangerouslySkipPermissions:true`。
- 【文档事实】deny/ask rules、hooks、requires-user-interaction MCP tools、组织强制 ask、critical-path 的 `rm/rmdir` 仍可挡住/绕过 bypass。
- 【文档事实】当 bare `allowedTools` 或 bypass 会使 `canUseTool` 被提前 shadow 时，TS 发 Node warning `CLAUDE_SDK_CAN_USE_TOOL_SHADOWED`。
- 【文档事实】subagent 继承 parent permission mode；parent 用 bypass/acceptEdits/auto 时，subagent 的 per-definition mode 不可覆写。
- 【文档事实】可在 streaming session 用 `setPermissionMode()` 立即改变后续 tool requests 的 mode。
- 【推断】Rust kernel 的最小安全面不能只是一枚 `approve(bool)` 回调；至少需要 declarative deny/ask/allow、强制优先级、每请求的 interactive decision、和不可被“方便模式”跳过的 pre-action policy hook。

### 7. MCP：可替换外部 capability 边界

- 【文档事实】`mcpServers` 可在 options 直接传，或由 project setting source 的 `.mcp.json` 装载；`strictMcpConfig` 可忽略用户/项目/plugin connector 配置。
- 【文档事实】支持 local stdio process、remote `type:"http"` / `"sse"`、以及 app-process 内 SDK MCP server 三类 transport。
- 【文档事实】模型侧名字标准化为 `mcp__<server-name>__<tool-name>`，并参与同一 permission pipeline。
- 【文档事实】官方建议精确 `allowedTools:["mcp__server__*"]`，而非用 bypassPermissions 来开放 MCP。
- 【文档事实】options 传入的 stdio 或未缓存 HTTP/SSE server 在 first turn 前等待连接，默认上限 30s；in-process server 不阻塞 first turn。
- 【文档事实】init event 含每 server 的 `pending|connected|failed|needs-auth|disabled`；`pending` 可能只是延后连接/缓存 schema，不能当作失败。
- 【文档事实】SDK 不执行 OAuth browser flow；服务端要求 auth 且无 token 时，该 server 不可用且报 `needs-auth`，OAuth/UI/token storage 是宿主职责。
- 【文档事实】tool search 默认开启，延迟加载 MCP tool schemas，以防几十/几百工具挤占每轮 context。
- 【文档事实】MCP output 超过默认 25,000 tokens 时完整结果落盘，模型只收到指向文件的提示；`MAX_MCP_OUTPUT_TOKENS` 可调。
- 【推断】kernel 应拥有统一的 external-tool registry、lifecycle/status、schema-lazy-loading 和 output bounding；credential acquisition、OAuth 页面、租户 policy、服务器部署均应留给使用者/前端。

### 8. Context / compaction / storage

- 【文档事实】每轮 context 包含 system prompt、tool definitions、conversation history、tool input/output；它不会在 turn 间重置。
- 【文档事实】稳定前缀（system prompt、tools、CLAUDE.md）会自动 prompt cache；大文件读入与 verbose command output 是 context 膨胀主因。
- 【文档事实】接近窗口限制时 automatic compaction 将旧 history 换为 summary，保留 recent exchanges/key decisions，并发 `compact_boundary` event。
- 【文档事实】早期 prompt 中的具体要求可能在 compaction 后丢失；官方建议把持久规则放在通过 settingSources 注入且每轮重放的 CLAUDE.md。
- 【文档事实】`PreCompact` hook 可归档完整 transcript，输入带 `trigger: manual|auto`；送 `/compact` 可手动触发。
- 【文档事实】subagent fresh context 和只回传最终消息，是官方建议的 context compression 手段。
- 【文档事实】tool search 延迟 MCP schemas；具体 agent 只给最小 tools，也是 context control 手段。
- 【文档事实】session 默认写本地 disk，保存完整 prompt/tool call/tool result/response conversation，而不保存/回滚 filesystem。
- 【文档事实】通过 Result/init message 获 session ID；`continue` 取 cwd 最新 session，`resume` 指定 ID，`forkSession` 复制 history 并产生新 ID。
- 【文档事实】fork 分叉的是 conversation 不是 filesystem；若需 revert 文件改动，另开 file checkpointing。
- 【文档事实】TS `persistSession:false` 可只在内存保存本 call；跨 host/serverless 要用户提供 `SessionStore` adapter 镜像 transcript。
- 【文档事实】SessionStore 是 dual-write：Claude Code subprocess 先本地落盘，再写外部 backend；开发者需理解重启/存储延迟下哪一份可恢复。
- 【推断】kernel 要区分三个资产：可压缩的 model context、不可丢的 audit/event log、可恢复的 session snapshot；不要用同一“历史字符串”同时承担三者。

## 它把什么放进核心、什么留给使用者

### 放进 core 的能力（文档事实）

- 自主 tool-use loop、tool execution sequencing、tool-result 回注与终止判定。
- session identity、transcript persistence、resume/continue/fork 和部分 file checkpoint control。
- built-in tool protocol、MCP transport/status、custom tool adapter 与 tool search。
- layered permission evaluation、interactive callback 接点、强制 deny/ask 规则、mode 和 pre/post hooks。
- subagent lifecycle、fresh-context isolation、parent correlation、background operation、depth/concurrency/budget enforcement。
- prompt/context accounting、automatic/manual compaction、prompt caching 和 compaction lifecycle event。
- message/event stream、partial output、final usage/cost/status 和 interrupt/control methods。

### 明确留给 SDK 使用者/前端的能力（文档事实）

- 宿主提供 prompt、systemPrompt、model/fallback、effort、tool/agent definitions、working directory、限额和 settings sources。
- 宿主用 `canUseTool` 把 approval request 接到自己的 UI；SDK 不替应用设计审批对话框。
- OAuth interactive flow 与 token 获得/保管由应用完成，SDK 只接收 headers/环境变量。
- 跨机器持久化由用户实现 SessionStore backend，或自行搬运 transcript；SDK 不提供托管会话服务。
- 业务工作流、何时委派、agent 角色、结果整合、重试规则、用户/租户身份和 RBAC 都由应用决定。
- MCP server 的部署、凭据、网络可达性和领域内真正的只读/写入约束由 server/宿主负责。
- CLI/IDE/Web 的显示样式、事件转发协议、终端渲染和产品 branding 属于前端壳层。

### 对“kernel + 多前端”的归纳（推断）

- 核心应对 UI 无感：它产出 typed lifecycle events，接收 typed commands/decisions；前端不应成为唯一能推进 loop 的状态机。
- 允许 UI 先只消费聚合 assistant/tool/result events，也可选择 partial deltas；流粒度是 presentation capability，不应改变执行语义。
- 权限 UI 是可替换适配器：kernel 发 `ApprovalRequested{request_id, tool, args, context}`，宿主回 `ApprovalDecision`；决策 ID 必须能在断线/重连后重派且幂等。
- session 列表、标题、tags、transcript view 可做在前端，但 session ownership/version/concurrency 规则应在 kernel/storage adapter。
- subagent 不是 frontend“开多个聊天页”就足够：隔离、配额、取消传播、父子追踪、结果回收必须由 kernel 管。

## 踩坑与教训

- 【官方文档】不要把永久约束只写在初始 prompt：auto-compaction 会替换旧 history；应每轮注入 durable instructions，且先在 PreCompact 存档。
- 【官方文档】`allowedTools` 是“预批准”而非“限制可见能力”；若 headless worker 要固定工具面，应组合 `allowedTools + dontAsk`，并对不可见工具用 `disallowedTools`/显式 tools 集合。
- 【官方文档】不要把必须审计/阻止的检查放在 `canUseTool`：任何更早 auto-approval 都能使它不被调用；应该用 PreToolUse。
- 【官方文档】`bypassPermissions + allowedTools` 并不构成 allowlist，反而可能给未列 Bash/Write/Edit 也放行；仅适合隔离 CI/container，且仍要 deny rules/hooks。
- 【官方文档】subagent 不带 parent history/system prompt；委派 prompt 必须显式带 paths、错误、约束/决策，否则 isolation 会成为信息缺失。
- 【官方文档】TS 的 `env` 替换而非合并 subprocess env，必须 `...process.env`，否则可能丢 PATH/凭据；Python 才是 merge。
- 【官方文档】MCP init 的 `pending` 不是失败；应持续查 status，并区别 `failed`/`needs-auth`。
- 【官方文档】大 MCP output 虽有落盘截断，工具选择/服务端 scope 仍需限制；大 schema 和 verbose output 会先耗尽 context。
- 【社区 issue，未获 maintainer 证实，2026-07 仍 open】#361 报告：background subagent 的 parent turn 结束后，共享 stdin/control channel 被关闭，后续 MCP permission request 失败为 `Error: Stream closed`，tool hooks 也不再发；报告者 workaround 是在 background task 未完成时阻止 parent Stop。这个报告不能当作稳定 API 事实，但提示 kernel 必须把“session 仍活着”与“parent turn 已完成”的 permission/event channel 生命周期分离。
- 【社区 issue，未获 maintainer 证实，2026-07 仍 open】#389 提供可复现报告：MCP `structuredContent` 可绕过 oversized-result 截断，导致 compaction 后 1--3 turn 内 context 重填、反复 compact，最终 `Autocompact is thrashing` abort；其临时 hook 截断可缓解。教训是 output quota 必须在最终送入 context 的完整 representation 上强制，不能只限一种 text field。
- 【社区 issue，未获 maintainer 证实，2026-05 仍 open】#316 声称 AgentDefinition 内声明的 MCP resources 对 subagent 不可用，迫使全部 server 提升到 orchestration 层；虽与当前文档中的 `mcpServers` 字段存在张力，仍说明必须对“工具定义可见”与“运行时 transport/资源已可用”分别测试，并提供 per-agent capability isolation。
- 【官方博客】复杂 workflow 的 loops/barriers/routing 放进确定性 JavaScript，避免其本身占模型 context；模型在该框架中按 isolated goal 调 worker。对数十/数百 agent，SDK 文档也建议 Workflow tool，而非单轮自然语言 delegation。
- 【推断】Rust kernel 的恢复模型需明确：terminal result 后仍有尾事件、单次 `query` error result 后会 raise、background completion 可晚于 parent text turn；不能用“收到一条最终文本”作为生命周期结束条件。

## 对我们 Rust agent kernel 的启示

### 推荐的最小稳定接口面

- `Kernel::start(StartRequest) -> RunHandle`：StartRequest 包含 immutable run config（model adapter、tools、agents、limits、policy refs、session selector），不混入 UI 对象。
- `RunHandle` 同时提供 `Stream<Item = KernelEvent>` 和命令方法/command channel：`submit_input`、`approve`、`deny`、`interrupt`、`cancel_task`、`set_mode`、`resume`。
- `KernelEvent` 的首批 variant 至少有 `RunStarted`、`AssistantTurn`、`ToolRequested`、`ToolFinished`、`ApprovalRequested`、`TaskSpawned/Progressed/Finished`、`ContextCompacted`、`RunFinished`、`Diagnostic`。
- 所有事件至少携带 `run_id`、monotonic `seq`、timestamp；task/tool 事件带 `task_id`、`parent_task_id`/`parent_call_id`、`correlation_id`，让任意前端可断线续流。
- `RunFinished { outcome, usage, costs, session_id }` 仅是正式 terminal marker；允许 drain trailing diagnostics 或用明确 `StreamClosed` 表示 transport 真正关闭。
- `Tool` 接口带 input/output schema、side-effect/read-only/idempotent hints、invoke cancel token；MCP/built-in/custom tools 全部落到该统一接口。
- `Policy::decide(ToolRequest)` 应得到 allow/deny/ask/defer/modify；其 precedence 和不可绕过的 pre-action policy 要由 kernel 保证，不让各 UI 自己拼安全逻辑。
- `AgentSpec` 应仅含 role prompt、tool capability subset、model/effort、turn limit、execution mode；把 application workflow/DAG 放在更高层。

### 责任边界建议

- kernel：执行状态机、tool/agent 调度、预算/并发/层级强限、cancellation、权限求值、context compaction、session/event identity、transcript persistence abstraction。
- 库使用者：业务 prompts、agent catalog、MCP config、凭据注入、context summary 要保留什么、业务 workflow/retry/escalation 与 storage backend。
- 前端：展示 event、让人做 approval/clarification、输入/取消按钮、session list 与日志视图；前端不得直接拥有唯一 session truth 或绕开 policy 执行 action。
- 基础设施 adapter：模型 provider、MCP transport、filesystem/shell sandbox、secret provider、SessionStore、telemetry sink；这些应可替换而非写死进 Rust kernel。

### 不要过早复制 Claude 专有层的部分

- 不应把 CLAUDE.md、slash command、Claude Code native binary、特定 prompt 字符串或 Anthropic cost estimator 作为 Rust kernel ABI；它们是上层 product/model adapter 行为。
- 不应让 kernel 自动决定“什么时候委派给哪个专家”；先给可靠的 `spawn/spec/limit/events`，由应用或 workflow engine 决策。
- 不应把文件系统回滚等同于 conversation fork；两个资源的 snapshot/transaction 有不同安全和并发语义。
- 不应把 MCP annotations 当安全边界；真正最小权限仍依赖 kernel policy、server-side enforcement、sandbox/network credentials scope。

## 来源

- Anthropic Claude Agent SDK overview（SDK/CLI/Client SDK 边界、same loop/tools/context）：https://code.claude.com/docs/en/agent-sdk/overview
- Anthropic “How the agent loop works”（turn、message、tool scheduling、limits、compaction、result/hook）：https://code.claude.com/docs/en/agent-sdk/agent-loop
- Anthropic TypeScript API reference（`query`、`Query` control surface、Options、custom tool、session APIs）：https://code.claude.com/docs/en/agent-sdk/typescript
- Anthropic “Subagents in the SDK”（AgentDefinition、inheritance、background、limits、correlation）：https://code.claude.com/docs/en/agent-sdk/subagents
- Anthropic “Intercept and control agent behavior with hooks”（hook lifecycle、callback/outputs/precedence）：https://code.claude.com/docs/en/agent-sdk/hooks
- Anthropic “Configure permissions”（six-step evaluation order、rules、modes、subagent inheritance）：https://code.claude.com/docs/en/agent-sdk/permissions
- Anthropic “Connect to external tools with MCP”（transport、status、OAuth ownership、tool search、output limit）：https://code.claude.com/docs/en/agent-sdk/mcp
- Anthropic “Work with sessions”（continue/resume/fork、local persistence、cross-host SessionStore）：https://code.claude.com/docs/en/agent-sdk/sessions
- Anthropic blog “A harness for every task: dynamic workflows in Claude Code”（确定性 workflow/agent orchestration 的动机；官方设计说明但非 SDK 契约）：https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code
- GitHub issue #361（社区 bug report，open，非已确认事实）：https://github.com/anthropics/claude-agent-sdk-typescript/issues/361
- GitHub issue #389（社区 bug report，open，含复现，非已确认事实）：https://github.com/anthropics/claude-agent-sdk-typescript/issues/389
- GitHub issue #316（社区 bug report，open，非已确认事实）：https://github.com/anthropics/claude-agent-sdk-typescript/issues/316
