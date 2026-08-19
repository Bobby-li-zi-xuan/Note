# runtime 模块

## 模块定位

runtime 是 Agent 的核心编排层（`com.yourorg.biboagent.runtime`），不直接接触 LLM 或数据库，而是协调 llm、tools、memory 等模块完成一次完整对话。内部按 `app`（核心业务）、`domain`（领域模型）、`infra`（基础设施适配）、`ports`（接口定义）分层。

---

## AgentService：对话总入口

一次请求的处理流程：

1. **消息预处理**：生成内部消息 ID（`m_` 前缀）、配置 MDC（sessionId/requestId/userId）、校验会话、合并会话级与请求级 skill。
2. **System Prompt 组装**：`PromptAssembler` 拼接角色人设（含当前情绪）、Skill 片段、L0 静态规则、记忆、娱乐上下文。
3. **消息列表构建**：检查 Redis 快照——有则恢复步数与消息（断点续连），无则从 PostgreSQL 加载历史。
4. **历史截断**：`HistoryTruncator` 用 jtokkit 精确估算 token（纯文本 + 消息结构开销），（消息 + SystemPrompt + 工具 schema 固定开销）占比 ≥85%（`truncation-trigger-ratio`，可配置）时对旧消息执行结构化摘要（6 维 JSON），<85% 不处理。保留最近消息轮数按 token 预算自适应（`recent-message-budget-ratio` 默认 30% 上下文）。SystemPrompt 与摘要解耦，摘要注入 SystemPrompt 末尾；LLM 摘要失败回退关键词提取。异步预压缩缓存命中时直接用预计算结果，跳过同步截断（见"上下文压缩机制 §5"）。
5. **Turn Loop**：`TurnLoopCoordinator` 多轮循环，见下节。
6. **收尾**：AI 伴侣情绪更新（非阻塞，异常不阻断主流程）；步数超限/总超时/LLM 失败时生成终止摘要（经 SSE 推给用户）；持久化 UserMessage + AiMessage；更新会话活跃时间。
7. **记忆提取**：三级触发（P0 用户意图词 → 同步提取 + 即时反馈；P1 step ≥ 2 → 异步深度；P2 周期 + 信息密度足够 → 异步浅度），详见 memory 模块。
8. **推送 FINAL 事件**（持久化成功后才推送），随后清理快照。

## TurnLoopCoordinator：多轮交互循环

每轮迭代：

1. **取消检查（优先）**：SSE 断连置位 Canceller → 立即终止（CANCELLED，保留快照供续连）。
2. **总超时检查**：超过 `totalTimeoutMs`（默认 120s）→ TOTAL_TIMEOUT。
3. **LLM 调用**：`LlmInvoker.call()` 返回 Mono，虚拟线程阻塞等待；成功返回后保存快照（Redis，TTL 30min）。
4. **工具调用**：`toolCallsCount++`（finish 不计入），超过 `maxToolCalls`（默认 4）→ STEP_LIMIT；否则 AiMessage 加入消息列表 → `PlanExecutor` 执行工具 → 每个 tool_call 注入专属 `ToolExecutionResultMessage` → 保存快照 → 下一轮。**结果注入查找顺序逐级回退**：LLM 工具调用 ID 精确匹配 → 位置下标 key（`tool_`+索引，ID 丢失时 PlanExecutor 以该 key 落结果，同轮唯一防同名覆盖）→ 工具名匹配（create_plan 等单调用路径）→ 聚合摘要兜底。执行与注入统一使用同一份工具调用列表（`aiMessage.toolExecutionRequests()` 优先），消除流式收集与 aiMessage 两份列表 id 错位导致的"串味"。
5. **纯文本**：空文本触发一次自我修正（注入"请重新思考是否需要调用工具"，仅一次）；有实质内容则接受为最终回复。
6. **finish 工具**：LLM 主动声明任务结束 → COMPLETED，立即退出循环。

终止原因：`COMPLETED` / `STEP_LIMIT` / `TOTAL_TIMEOUT` / `LLM_FAILURE` / `SELF_CORRECT_FALLBACK` / `CANCELLED`。

设计要点：三个终止条件互补——步数限制防死循环、工具调用次数防工具链无限延伸、总超时兜底；快照"执行中增量保存、完成后清理"，服务崩溃最多丢失一个 step 的进度。

## LlmInvoker：LLM 调用封装

- 用 `Mono.create()` 把 LangChain4j 流式回调桥接为响应式流；`Mono.delay` 在独立调度器上发超时信号，`AtomicBoolean` CAS 裁决"超时先到还是回调先到"，不阻塞调用线程。
- **首 token 前超时**：`sink.error` 触发声明式重试链 `.retryWhen(Retry.backoff(2, 200ms).maxBackoff(2s).filter(isRetryable))`；仅 IOException / HTTP 429 / 含 rate、timeout 的错误可重试，其他错误（如 400）不重试。
- **首 token 后超时**：不重试（LLM 已计费，重试会产生新旧内容杂糅），返回空结果由下一轮 step 兜底。

## PromptAssembler：System Prompt 组装

按顺序拼装：① 角色人设（含当前情绪状态）→ ② Skill Prompt 片段 → ③ 用户静态规则 → ④ 记忆（按 `memory-injection-max-tokens` 预算注入，默认 800，候选最多 20 条）→ ⑤ 娱乐上下文（角色偏好 + 播放器状态）。

## StreamingResponseHandler：SSE 事件桥接

既是 LLM 流式回调处理器，也是阶段事件通知器：`TOKEN_DELTA`（逐 token 内容）、`STAGE`（LLM_CALL / TOOL_CALL / TOOL_RESULT / STEP_RETRY / FINAL）。事件同时写入 Redis（持久化供断连重放）和 SseEmitter（实时推送），带单调递增 ID，客户端用 `Last-Event-Id` 断点重连。

---

## 上下文压缩机制

> 核心组件是 `HistoryTruncator`。SystemPrompt 不进入截断器——摘要通过 `TruncationResult.summary()` 返回，由 AgentService 注入 SystemPrompt 末尾（P1 结构性修复，避免旧实现把 SystemPrompt 当"已有摘要"误压缩）。

### 1. Token 估算

`estimateMessageTokens()` = `MessageSerializer.extractContent()` 纯文本 token（jtokkit CL100K_BASE）+ 消息结构开销：ToolExecutionResultMessage +80、AiMessage（含 tool calls）+150、AiMessage（纯文本）+30、其他 +20——补正 `extractContent` 只取纯文本导致的系统性低估。

固定开销（SystemPrompt + 工具 schema）由 AgentService 估算后作为 `overheadTokens` 传入截断器，统一计入占比判断与保留预算（`recent-message-budget-ratio` 作用于扣除固定开销后的剩余预算）。异步预压缩的触发判断用简化估算（纯文本 + 每条 +40 结构开销），任务内由 `truncate()` 精确处理（两处口径允许粗/精差异）。

### 2. 截断触发与结构化摘要

| 阈值 | 行为 |
|------|------|
| < 85% | 不处理 |
| ≥85%（消息 + 固定开销） | 结构化摘要：6 维 JSON（字段见下，单条 ≤30 字）；Prompt 注入已存储记忆做"绝对不要重复提取"去重，"新覆盖老"处理冲突并标注"(已更新)" |


**六维 JSON 字段**（结构化摘要输出，每字段为 List<String>，单条 ≤30 字）：

| 字段 | 含义 |
|------|------|
| `coreRequests` | 核心诉求、明确下达的指令与目标 |
| `sessionRules` | 会话内临时设定/变更的规则与偏好 |
| `ongoingTasks` | 进行中任务、进度与卡点 |
| `keyConstraints` | 关键约束、数值参数、范围限制、前提与最新结论（含作废标注） |
| `upcomingActions` | 待办事项、后续需执行动作 |
| `toolResults` | 工具调用的关键返回信息摘要 |

- **递归兜底**：截断后"摘要 + 消息 + 固定开销"占比仍 ≥85%（`truncation-trigger-ratio`）→ 递归再截断（≤ MAX_TRUNCATION_DEPTH=2，重检计入摘要与固定开销 token）；到达深度上限后执行最后一轮结构化摘要，不再递归。
- **LLM 摘要失败**：降级 `buildKeywordFallback()`（按"禁止、必须、架构"等关键词提取 + 消息列表压缩）。

### 3. 自适应保留轮数

替代固定 8 轮：最近消息预算 = `contextLength × recent-message-budget-ratio`（默认 0.30），从后往前累计、超预算停止；下限 `min-keep-rounds=2`。工具调用与其 `ToolExecutionResultMessage` **成对保留**，避免截断出"孤儿工具"。

### 4. 工具结果源头限长

工具结果经 `ToolInvoker` 的 `toString()` 出生后会原样进入回合内上下文，链路无上限。`SingleToolExecutor` 在 `normalize()` 后用 `ToolResultTrimmer.trim()` 按 `tool-result-max-tokens` 限长（可配置），超限时结构采样并追加截断标记，让 LLM 知道结果不完整。采样逻辑仅服务于源头限长本身。

### 4.1 工具结果归档（Redis TTL，供追问恢复）

回合结束后，`AgentService` 把当轮 `ToolExecutionResultMessage`（源头限长后的内容）写入 Redis List（key `tool:results:<sessionId>`，TTL 72h，每会话保留最近 50 条，按 sessionId 隔离）。归档不进入下一轮上下文窗口。

**`recall_tool_result` 工具**：用户追问"上次/刚才查到的内容细节"时，LLM 调用它取回归档内容。主路径为候选召回：`toolName`（可空，空=发现模式）+ `keyword` + `count`（默认 5），返回最近 N 条候选（含相对时间与内容），由模型按用户指称匹配（同一工具可能对应多次不同任务）；`toolCallId` 精确匹配保留为次要路径（归档键 = 结果 JSON 中模型可见的 tcId，提取失败回退 provider id）。sessionId 由 `ToolExecutionContextHolder` 沿执行链透传（PlanExecutor → StageScheduler → SingleToolExecutor），无匹配时返回"未找到（可能已过期）"。

### 5. 异步预压缩

**两个阈值的分工**（最容易混淆的点——60% 与 85% 不是重复配置）：

| 阈值 | 默认值 | 性质 | 决定什么 |
|------|--------|------|---------|
| `async-precompress-threshold` | 0.60 | **触发"计算"** | 回合结束后要不要派异步任务去算截断结果（只决定"何时提前算"） |
| `truncation-trigger-ratio` | 0.85 | **触发"压缩"** | 是否真正执行结构化摘要（`truncate()` 内部判断，异步任务内原样执行） |

**执行流程**：

1. 回合结束（persistTurn 之后），简化估算（纯文本 + 固定开销 + 每条 40 token 结构开销）占比 ≥ 0.60 → 提交虚拟线程任务（含 DB 查询 + 最长 20s 同步 LLM 摘要的阻塞 IO）
2. 任务内精确估算（与同步路径同一套 `truncate()` 逻辑）：
   - **< 85%** → `summary() == null` → **不写缓存**（白算一次；此区间不调 LLM，仅 token 估算，开销可忽略）
   - **≥ 85%** → 结构化摘要 + 消息裁剪 → 写入 `CompressionCache`（Redis，TTL 300s），版本号 = 最后一条消息 id，CAS（版本一致才覆盖）防止旧任务覆盖新结果
3. 下一轮 `SessionLifecycleService.validateAndLoad` 校验缓存版本 == 历史最后一条 id → 命中则 `SessionLoadResult` 携带 `precomputedSummary`/`precomputedMessages`，AgentService 跳过同步截断直接把摘要注入 SystemPrompt——**消除压缩导致的用户感知首字延迟**（压缩动作提前到上一轮末尾执行）

**缓存命中时跳过的只是"重复计算"**：压缩动作本身已在异步任务内用同一套 truncate 逻辑完成，结果与同步执行等价。

**同步 85% 截断仍是不可替代的兜底**——以下场景缓存必然未命中，同步截断是唯一执行者：

- 异步任务尚未完成：用户紧接着发下一条消息时，任务（DB 查询 + 最长 20s 的 LLM 摘要）还在执行中，缓存未写入
- 异步任务异常（`exceptionally` 吞掉）或版本过期（CAS 失败 / 缓存 version 与历史最后一条 id 不符）
- 上下文从 <60% 直接跳跃到 ≥85%：上一轮结束时占比 <60%（未排异步任务），本轮用户发超长消息直接冲过 85% —— 同步是唯一执行者
- 快照恢复路径：`validateAndLoad` 快照恢复直接返回（`precomputedSummary = null`），不读缓存，走同步截断

**增量模式的特例**：`truncateIncremental()` 内部没有 85% 门槛——只要保留窗口覆盖不了全部消息就做增量摘要（游标定位，见 §6）。开启 `incremental-summary-enabled` 后异步任务在 ≥60% 即产出摘要写缓存，同步全量 85% 截断退化为纯兜底角色。

### 6. 增量滚动摘要

开关 `incremental-summary-enabled`（默认 false，依赖异步预压缩）；状态 `RollingSummaryState`（summaryText / lastTruncatedMessageId / compressionCount / summaryTokens / selfCompressCount）随缓存条目存 Redis。核心思想：只对"上次截断之后新滚出保留窗口"的消息做 LLM 摘要，已有摘要作为"已知背景"注入 Prompt 防重复提取。

**执行流程**（异步预压缩任务内 `truncateIncremental(history, prevState, ...)`，`prevState` 从缓存条目取出）：

1. **状态检查**：`prevState` 为 null（首次 / 缓存丢失）→ 降级全量截断（`doTruncate`），不设游标
2. **保留窗口判断**：`calculateKeepCount`（与 Phase 3 同一套自适应预算）→ 保留窗口覆盖全部消息 → 无旧消息可增量，复用已有摘要（仅检查自压缩），流程结束
3. **游标定位**：按 messageId 找增量起点（游标 = 上次被摘要的最后一条消息 id，**用 messageId 而非索引**——`loadHistory` 有 LIMIT 500 上限，窗口前移时索引会漂移）：
   - 游标失效（找不到：被 LIMIT 截断 / 快照恢复）→ 降级全量截断并**更新游标**（避免后续回合永远重复全量）
   - 防倒挂：keepCount 变大时游标落入新保留窗口，clamp 后走"无增量"分支（这些消息仍在新窗口内，下次窗口前移时会被增量覆盖，不丢失）
4. **增量摘要**：只对增量部分（游标之后 → 保留窗口起点）调 LLM，Prompt 注入已有摘要作"已知背景"（**绝对不要重复提取**）；LLM 失败 → 降级全量重摘要并更新游标（比拿残缺增量合并强）
5. **合并**：`mergeSummaries()` 用 LLM 语义合并（去重 + 冲突裁决 + 控体积——硬拼接必然膨胀、必然触发自压缩），失败回退硬拼接，由步骤 6 的自压缩兜底
6. **自压缩检查**：合并后 token > `summary-token-budget`（2000）→ `compressSummarySelf()` 压到预算的 50%（LLM 失败回退规则取前 30 行）——单次截断产生的新摘要不会触发，只有增量累积膨胀才触发
7. **更新滚动状态**：游标 = 本次被摘要的最后一条消息（保留窗口起点前一条，实现"不重不漏"）、compressionCount + 1 等 → 随缓存条目 CAS 写回，供下一轮复用

> **与长期记忆的区别**：截断摘要是回合边界的临时压缩，不写入 PostgreSQL 长期记忆库（Phase 4 会短暂缓存到 Redis 供下一轮复用）；长期记忆由 memory 模块的 P0/P1/P2 提取负责，两者解耦。



## 面试题整理

### 1. "Turn Loop 的终止条件有哪些？"

Turn Loop 的终止条件分为三个层级，从外到内依次生效：

**第一层：循环入口检查（每次 while 迭代开始时）**

| 条件 | 字段/来源 | 默认值 | 终止原因 | 作用 |
|------|----------|--------|---------|------|
| 步数耗尽 | `agent.runtime.maxSteps` | 8 | `STEP_LIMIT` | while 循环条件 `step < maxSteps`，成功调用 LLM 的步数达到上限后自然退出。防止 LLM 反复调用工具永不结束 |
| 用户取消 | `Canceller`（AtomicBoolean） | — | `CANCELLED` | SSE 连接断开或用户刷新页面时置位，检查优先级高于超时。取消后保留快照供断点重连 |
| 总超时 | `agent.runtime.totalTimeoutMs` | 120000ms | `TOTAL_TIMEOUT` | 整个 Turn Loop 从开始到当前的 wall-clock 时间超过阈值，作为所有其他条件的最终兜底 |

**第二层：LLM 调用结果判断（每次 `LlmInvoker.call()` 返回后）**

| 条件 | 字段/来源 | 默认值 | 终止原因 | 作用 |
|------|----------|--------|---------|------|
| LLM 调用失败 | `LlmInvoker` 内部重试耗尽 | 2 次重试 | `LLM_FAILURE` | `streamResult == null` 表示 LLM 在所有重试后仍失败。首 step 失败时保存初始快照供恢复 |
| 单步 LLM 超时 | `agent.runtime.stepTimeoutMs` | 30000ms | 返回 null → `LLM_FAILURE` | 单次 LLM 调用超时。首 token 前超时可重试（最多 2 次），首 token 后超时直接返回 null |
| LLM 调用 `finish` 工具 | LLM 行为 | — | `COMPLETED` | 正常完成——LLM 主动声明任务结束，循环立即退出 |
| 工具调用次数超限 | `agent.runtime.maxToolCalls` | 4 | `STEP_LIMIT` | 累计工具调用次数超过上限，防止工具链无限延伸 |
| 纯文本（无工具调用） | 自检逻辑 | 最多 1 次 | `SELF_CORRECT_FALLBACK` | LLM 返回纯文本时触发自我修正一次（不计入 step），提示 LLM 重新思考是否需要调用工具。自检后仍为纯文本则接受为最终回复 |

**第三层：循环退出后的覆盖检查**

| 条件 | 触发场景 | 结果 |
|------|---------|------|
| step 达上限但非 finish | 循环自然退出且最后一步未调用 `finish` | `reason` 从 `COMPLETED` 覆盖为 `STEP_LIMIT` |
| `SELF_CORRECT_FALLBACK` | 自检后接受纯文本 | 不覆盖，视为"软完成" |

**StageScheduler 内部的终止（Plan 执行期间）**

| 条件 | 状态 | 作用 |
|------|------|------|
| `canceller.isCancelled()` | CANCELLED | Plan 执行中检测到取消信号，立即中断所有未完成任务 |
| `remainingMs <= 0` | TIMEOUT | 使用 Turn Loop 传入的剩余 time budget 做二次检查 |
| Stage FAIL_FAST 触发 | FAILED | Stage 配置了快速失败策略且任一任务执行失败 |
| interactive Stage 完成 | PAUSED | 暂停等待 LLM 确认（非终止，是将控制权交回 Turn Loop） |

### 2. "AgentService 是如何保证崩溃恢复的？"

每轮 LLM 成功返回后和每次工具执行后，系统都会在 Redis 中保存一个 TurnSnapshot（含当前步数、消息列表快照、请求 ID 等）。服务崩溃重启后，AgentService 在处理下一个请求时检查该会话是否有活跃快照——有则还原步数和消息列表，从中断的步数继续执行；对话正常完成后清除快照。这种"执行中增量保存、完成后清理"的策略在保证恢复能力的同时避免快照无限堆积。

### 3. "Prompt 组装包含哪些内容？怎么保证不超长？"

Prompt 组装包含角色人设（含情绪）、技能 Prompt 片段、用户偏好规则、记忆（token 预算内注入）和娱乐上下文。为了控制上下文长度，系统在每次请求前用 token 计数精确估算消息占比，（消息 + SystemPrompt + 工具 schema）占比 ≥85%（`truncation-trigger-ratio`）时对旧消息执行结构化摘要（6 维 JSON），<85% 不处理；保留最近消息轮数按 token 预算自适应，摘要注入 SystemPrompt 末尾；异步预压缩缓存命中时跳过同步截断直接用预计算结果。这种"先估算、达标再摘要"的策略在保证信息完整性和控制 token 消耗之间做了平衡。

### 4. "为什么 LLM 调用失败后的重试策略要区分首 token 前后？"

这是一个基于成本收益分析的设计。首 token 之前的失败意味着请求尚未到达 LLM 或 LLM 还没开始生成，通常是网络抖动、限流、服务暂时不可用——可安全重试且代价低：通过 Reactor Mono 的 sink.error 发射异常，自动触发声明式 retryWhen 退避重试链（最多 2 次，200ms 起步递增）。首 token 之后的失败则不同：LLM 已消耗上下文并开始生成，重试会浪费已消耗的 token 配额，且可能生成不同内容导致不一致——因此不重试，直接返回空结果交给下一轮 step 兜底（下一轮重新发送完整上下文，本质是"间隔性重试"）。
