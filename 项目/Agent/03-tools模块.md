# tools 模块

## 模块定位

tools 模块是 Agent 与外部世界交互的通道（`com.yourorg.biboagent.tools`）。每个被 `@Tool` 注解标记的 Java 方法代表一项能力；LLM 决定调用哪个工具、传什么参数，模块负责接收、校验、执行，并把结果归一化后回传给 LLM。

## 工具注册与 JSON Schema 生成

**两层注册**：① 启动时扫描器遍历 Spring 容器所有 Bean 的方法，识别 `@Tool` 注解（方法名/描述/参数元信息），连同方法所属 Bean 实例按工具名存入 Map；同时读取自定义 `@ToolPolicy`（风险等级、超时、是否可重试）。② `ToolMetadataRegistry` 在 @PostConstruct 阶段从扫描器复制全部 ToolSpecification 与策略配置，对外提供只读查询。扫描只做一次，注册中心只负责缓存，职责分离。

**LangChain4j 生成 Schema**：`ToolSpecifications.toolSpecificationFrom(Method)` 反射读取——`@Tool` 的 name/value 映射工具名与描述；`@P` 的 name/value 映射参数名与描述；Java 类型映射为 JSON Schema 类型（String→string、int/long→integer、double/float→number、boolean→boolean、List/数组→array、POJO/record→object 递归解析）；排除 `@ToolMemoryId` 特殊参数（用于注入 sessionId 等上下文，不暴露给 LLM），其余参数默认 required。Schema 设置到 `OpenAiChatRequestParameters.toolSpecifications`。

每轮请求构建工具列表时，会把注册中心的全部 ToolSpecification 追加一个内置 `finish` 工具（虚拟终止信号，见面试题 5）。

## 工具执行流程

### 阶段 1 调用检测（TurnLoopCoordinator）

- 检测 `toolCalls` 非空；**先检测 finish**（→ COMPLETED，不计入计数），再 `toolCallsCount++`（超 `maxToolCalls=4` → STEP_LIMIT）
- AiMessage 加入消息历史；计算剩余 time budget `remainingMs = max(5000, totalTimeoutMs - elapsed)`
- 推送 STAGE 事件 onToolCall(step, toolNames)；保存快照（含最新 AiMessage）

### 阶段 2 路由（PlanExecutor 统一入口）

所有工具调用统一走 Plan 执行模型，三条路径：

- **A 显式计划**：LLM 调用 `create_plan`（参数为 planJson，含 summary + stages）。`PlanParser.parse()` 反序列化 → `StageConstraintValidator.validate()` 静态校验（循环依赖、占位符、任务 ID 唯一）→ 创建/恢复 `PlanExecutionContext` → 按 Stage 调度执行。
- **B Interactive 恢复**：Plan 含 `interactive: true` 的 Stage 完成时返回 PAUSED 等 LLM 决策；下一轮 LLM 返回普通工具调用时，从快照恢复 PlanExecutionContext，过滤已完成 Stage 继续执行；全部完成后销毁上下文。
- **C 隐式计划（最常见）**：普通多工具调用自动生成单 Stage Plan（`on_failure=CONTINUE`），每个 ToolCall 转为 `PlannedTask("tool_"+i, toolName, args, null)`，同 Stage 内并行执行；结果映射为 `toolCallResults`（toolCallId → 结果字符串，同轮同名工具不串结果）。

### 阶段 3 Stage 调度（StageScheduler）

跳过已完成 Stage → 取消/超时检查 → `PlaceholderResolver` 解析 `{{stage_x.task_y.result.id}}` 占位符 → `toSubtasks()`（超时均分）→ `subagentRunner.run()` 并行执行（详见 subagent 模块）→ 记录结果（`stageId → Map<taskId, content>`）并 `markStageCompleted()` → 错误策略检查（`FAIL_FAST` 立即终止 / `CONTINUE` 继续并让 LLM 感知失败）→ interactive Stage 返回 PAUSED。

`SubtaskResult` 内容为归一化 JSON：`{"success":true/false,"toolCallId":"...","toolName":"...","durationMs":...,"data":{...},"summary":"..."}`，失败时含 `error{code,message}`。

### 阶段 4 单工具管线（SingleToolExecutor）

取消检查 → `toolPolicy.decide()` 鉴权（拒绝返回 FORBIDDEN，不抛异常，包装成正常结果交 LLM 决策）→ 实际超时 `min(policy.timeoutMs(), remainingBudget)` → `toolExecutor.execute()` → `toolNormalizer.normalize()` 统一结果格式 → `metrics.recordTool()`。工具结果在此处经 `ToolResultTrimmer` 做源头限长（token 上限，超限带截断标记，防单条大结果撑爆回合内上下文）。

### 阶段 5 重试装饰器（RetryableToolExecutor）

幂等缓存检查（命中直接返回）→ 工具策略非 retryable 则直接执行 → 解析重试配置（工具级覆盖全局默认）→ 重试循环：每次执行前查取消信号；`executeWithTimeout()` 用 `CompletableFuture.supplyAsync().get(timeout)` 隔离执行，超时抛 TimeoutException → 包装为 TIMEOUT（可重试）；`RetryableException` → RETRYABLE；其他异常 → TOOL_EXEC_ERROR（不可重试）。失败且可重试则线性退避 `baseBackoffMs × attempt` 后重试；成功缓存最终结果。

### 阶段 6 实际执行（DefaultToolExecutor → ToolInvoker）

注册表查找 @Tool 方法 → `ToolParamValidator` JSON Schema 校验 → 参数反序列化（双保险映射：`@P` 注解名优先，回退 Java 编译参数名）→ `method.invoke(bean, args)` 反射调用 → 返回 `result.toString()` 或错误前缀。

### 阶段 7 结果回注（TurnLoopCoordinator）

按工具调用 ID 取专属结果注入 `ToolExecutionResultMessage`（无 ID 时回退工具名；解决旧代码"所有工具共享同一聚合字符串"的问题）

### 阶段 8 工具结果归档与追问恢复

回合结束后 `AgentService` 将当轮工具结果（源头限长后）写入 Redis List（`tool:results:<sessionId>`，TTL 72h，每会话保留最近 50 条，写入端按模型可见 tcId 幂等去重）。内置工具 `recall_tool_result(toolName?, keyword?, count?)` 在用户追问历史结果细节时按需取回——主路径返回最近 N 条候选（含相对时间与内容，由模型按用户指称匹配），`toolCallId` 精确匹配保留为次要路径；sessionId 经 `ToolExecutionContextHolder` 透传；归档不进入下一轮上下文窗口。

## Schema 校验与失败处理

**两层校验**：第一层 `ToolParamValidator` 用 networknt JsonSchemaFactory（JSON Schema 2020-12）对 argsJson 做结构性校验（参数名/类型/必填）；第二层参数反序列化后走 JSR-380 `@Valid` Bean 校验。

**错误分类**：

| 状态码 | 触发条件 | 可重试 |
|--------|---------|--------|
| SUCCESS | 正常返回 | — |
| SCHEMA_VALIDATION_FAILED | Schema 校验不通过 | ❌ |
| TOOL_EXEC_ERROR | 反射调用抛非重试异常 | ❌ |
| RETRYABLE | 工具代码抛 `RetryableException` | ✅ |
| TIMEOUT | CompletableFuture 超时 | ✅ |
| FORBIDDEN | 风险策略拒绝 | ❌ |
| UNKNOWN_TOOL | 注册表未找到 | ❌ |
| CANCELLED | Canceller 已置位 | ❌ |

错误不会静默吞掉：归一化为 `{"success":false,"error":{"code":...,"message":...},"summary":...}` 注入消息历史，LLM 下一轮自行决定修正参数/换工具/调 finish。

**自修正与终止兜底**：SELF_CORRECT 只处理"LLM 返回空文本 + 无工具调用"（注入一次"请重新思考是否需要调用工具"，仅一次）；工具失败不做专门修正提示——LLM 看到 error 信息后通常自行调整，连续犯错由 step/工具调用上限兜底；可重试网络错误在 RetryableToolExecutor 层自动消化，不透传 LLM；`fallbackTool` 支持主工具失败后降级一次。

## 幂等缓存

解决重试副作用问题（如重复发邮件）。**Redis 主存储 + 本地 ConcurrentHashMap 备用**的双写模式：正常读写走 Redis（TTL=120s，`agent.tools.idempotency.ttlSeconds`）；Redis 不可用时捕获异常自动回退进程内缓存，对调用方完全透明（区别：本地缓存不跨进程共享、不自动过期，Redis 恢复后自动回到 Redis 路径）。key 是请求创建时生成的幂等键，同一幂等键的重复调用直接返回缓存结果。

## 关键设计点

- **进程内反射调用 vs 远程调用**：当前所有工具都是进程内 Java 方法，简单低延迟，但扩展需重新部署；未来可在 ToolExecutor 接口层扩展 gRPC/HTTP 实现，对上层透明。
- **幂等键生命周期**：UUID 幂等键贯穿缓存检查、重试循环、结果存储。
- **超时与重试协同**：每次尝试独立超时（默认 8s），超时返回 TIMEOUT 触发重试；工具链还有兜底超时（`agent.tools.execution.defaultTimeoutMs=10000ms`）防止卡死。

## 风险策略

每个工具注册时带风险等级（LOW/MEDIUM/HIGH），执行前由 `ToolPolicyService.decide()` 决策。V1 策略：**HIGH 默认拒绝，LOW/MEDIUM 放行**；拒绝返回明确错误码和原因，LLM 收到后可向用户请求授权或换工具。接口预留扩展：用户级判断、ABAC、人工审批等。

## 面试题整理

### 1. "工具注册的 @Tool 注解扫描机制有什么坑？如何解决的？"

Spring 的 `getBeanNamesForAnnotation` 只对类级别注解生效，扫不到方法级 `@Tool`。解决：先尝试用 `getBeanNamesForAnnotation` 扫描，返回空（Spring 未检测到方法注解）则遍历所有 Bean 定义，对每个 Bean 反射扫描其方法。双保险确保所有 @Tool 工具都被正确注册。

### 2. "工具的重试机制是怎么设计的？"

装饰器模式——`RetryableToolExecutor` 装饰 `DefaultToolExecutor`，对调用方透明。重试三条件同时满足：工具标记可重试 + 错误属可重试类型（超时、网络错误）+ 未达最大尝试次数。每次重试有递进等待（线性退避），过程中 Micrometer 记录重试/超时次数。

### 3. "如果 Redis 挂了，幂等缓存还能正常工作吗？"

能。幂等缓存"Redis 主存 + 本地内存备用"双写：Redis 不可用时捕获异常自动回退进程内 ConcurrentHashMap，对调用方透明；唯一区别是本地缓存不跨进程共享、不自动过期；Redis 恢复后新请求自动走回 Redis 路径。

### 4. "工具的风险策略是怎么设计的？未来可以怎么扩展？"

当前 V1：HIGH 默认拒绝、LOW/MEDIUM 放行。`ToolPolicyService` 是接口，预留扩展点：基于用户角色的权限判断、基于上下文的动态风险评估、人工审批流程。接口接收用户上下文、工具名、风险等级和参数，返回允许/拒绝及原因。

### 5. "为什么 finish 工具要单独传 Schema、且单独移除工具结果？"

- finish 不是业务工具，而是 TurnLoop 的**终止信号**（`FinishToolSchema` 注释明确）
- 它不在 toolInvoker 注册表中、没有业务逻辑实现；单独添加避免污染业务工具注册表
- 检测到 finish 直接 break，不进入工具提交流程
- 从 finish 的 argsJson 解析 content 作为最终返回结果；COMPLETED 时从 finalToolCalls 中过滤掉 finish，避免前端收到 `[tool call: finish]` 而非实际回答

## 补充：Skill 机制（完整版见 06 文档）

Skill = "Prompt 片段 + Markdown 指令知识 + 关联工具列表"，定义于 `bibo-agent/skills/*.md`（YAML Front Matter + Markdown 主体）。用户以 `/code` 等 Slash 命令触发 → `SlashCommandParser` 截获并抹去指令标记 → `PromptAssembler.mergeSkillIds` 合并会话级/请求级 skill → Prompt 片段 + Markdown 主体注入 System Prompt，语义上引导 LLM 使用对应领域工具。**Tool 是能力的"手臂"（Java 方法），Skill 是能力的"业务大脑"（面向场景组合 Prompt + Tool，改 Markdown 即可扩展）**。
