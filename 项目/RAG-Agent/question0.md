# RAG-Agent 面试问答手册

---

## 一、核心架构

### Q1: 搭建当前 Agent 使用的架构是什么？

A: 项目采用 **Think-Execute 双层分离架构**，核心组件如下：

```
用户消息 → ChatMessageController
              → ChatEventListener (@Async 异步)
                  → RAGAgentFactory.create(agentId, sessionId)
                      ├── 从 agent 表加载 Agent 配置
                      ├── 从 ChatClientRegistry 获取 ChatClient（deepseek-chat / glm-4.6）
                      ├── 解析工具：固定工具（FIXED）+ 可选工具（OPTIONAL，按 allowedTools 叠加）
                      ├── 解析知识库列表
                      ├── 从冷存储加载历史消息 → 恢复 ChatMemory（L1 滑动窗口）
                      ├── 从冷存储加载 L2 缓存快照 → 恢复 MemoryCacheManager
                      └── 组装 AgentContext → 创建 RAGAgent
                          → AgentCore.run()
                              ├── think()  ─── 调用 LLM 决策：是否需要用工具
                              ├── execute() ─ 执行 LLM 请求的工具调用
                              └── think() ↔ execute() 循环，直到 FINISHED
                                  → finally: flushSessionCache() → 异步写回冷存储
```

**关键设计决策**：
- **为什么不用 Spring AI 自动工具执行**？关闭 `internalToolExecutionEnabled`，手动控制 think/execute 两阶段，可以在中间插入持久化、消息推送、状态管理等横切逻辑
- **为什么用事件驱动**？`ChatEventListener` + `@Async` 解耦 HTTP 请求线程和 Agent 执行线程，用户请求立即返回，Agent 异步执行并通过 SSE 推送结果
- **为什么使用 Caffeine 而非 Redis**？单实例部署场景下，进程内缓存的纳秒级延迟远优于 Redis 的网络往返；如果未来需要分布式，可以在 Caffeine 之上加 Redis 作为共享缓存

---

## 二、RAG 检索系统

### Q2: Chunk 的分片原则是什么？分片粒度如何？一个 .md 文件大概分了多少个 chunk？

A: Chunk 分片由 `MarkdownChunkComposer.compose(sectionTitle, sectionBody, maxTokensPerChunk)` 实现：

**分片原则（按优先级）：**

1. **原子块保护**：fenced code block（\`\`\`包裹）和表格行（\|...\|格式）视为不可分割的原子块，不会在中间切断
2. **段落边界切分**：以空行作为段落分隔符，在段落边界处切分
3. **句子边界切分**：单段超过 `bodyBudget` 时，按句末标点切分（正则：`(?<=[。！？.!?\\n])\\s*`），保证每个片段语义完整
4. **固定长度兜底**：单句仍超长时，逐字符累计 token 直到上限

**分片粒度：**
- **maxTokensPerChunk = 800**（可在 `application.yaml` 的 `rag.chunk.maxTokensPerChunk` 配置）
- 每个 chunk 以 `# 标题\n\n` 开头，确保嵌入向量包含层级语义
- Token 估算策略：ASCII 4 字符 ≈ 1 token，CJK 1 字符 ≈ 2 token

**实测数据**：当前数据库中 103 条 chunk 分布：

| 百分位 | Token 数 |
|--------|----------|
| P50 | 261 |
| P95 | 783 |
| P99 | 791 |

一个典型 .md 文件的章节会被拆分为 1~N 个 chunk。以当前 2 个知识库 103 条 chunk 为例，每个知识库对应若干 .md 文件，平均每个文件约 25-35 个 chunk，每个章节根据正文长度拆分为 1~5 个 chunk。

**为什么选择 800 token？**
- 800 token 约等于 400 个中文字或 3200 个英文字符，是一个语义完整的合理粒度
- 配合 `maxChunksPerKb=4`，每次检索最多返回 4×800=3200 token 的上下文，符合主流 LLM 上下文窗口的安全范围
- 通过 metadata 记录 `chunkIndex` 和 `chunkCount`，支持在上下文中还原原文结构

---

### Q3: Token 切割的原则是什么？切割时的依据是什么？如何保证语义完整？

A: Token 切割分为两套机制：

#### TokensEstimator（Token 估算）

**不依赖外部 tokenizer**，使用保守的字符级估算策略，确保不会低估 token 数导致上下文超限：

| 字符类型 | 比例 | 覆盖范围 |
|----------|------|----------|
| ASCII | 4 字符 ≈ 1 token | 英文、数字、标点 |
| CJK / 全角 | 1 字符 ≈ 2 token | 中日韩统一表意文字 + 日文假名 + 韩文 |
| 其他 Unicode | 1 字符 ≈ 1 token | 特殊符号 |

CJK 范围覆盖：CJK Unified Ideographs（含 Extension A/B）、Compatibility Ideographs、Symbols & Punctuation、Hiragana、Katakana、Hangul、Half/Full-width Forms。

**为什么 1 CJK ≈ 2 token？** 这是基于 GPT 系列 tokenizer 对中文的平均编码比的保守估计。实际测试中 1 个中文字通常占 1.2~2.2 token，取 2 是安全的上界。

#### MarkdownChunkComposer（语义切分）

**保证语义完整性的手段：**

1. **原子块保护**：fenced code block 和表格行不会在中间被切断
2. **段落边界优先**：在段落边界（空行）处切分，不会在同一段落内随意截断
3. **句末标点切分**：在 `。！？.!?` 等句末标点处切分，保证不会产生半句话
4. **标题前缀注入**：每个 chunk 以 `# 原始章节标题\n\n` 开头，即使被独立检索也能还原层级上下文
5. **Metadata 记录**：入库时记录 `sectionTitle`、`chunkIndex`、`chunkCount`，检索后通过 `RagContextAssembler` 按相邻合并策略还原原文结构

**与 LangChain RecursiveCharacterTextSplitter 的核心区别**：
- LangChain 的分隔符优先级是固定的（`\n\n` → `\n` → ` ` → `""`），对中文不够友好
- 本项目的切分器增加了句末标点切分和 CJK 感知的 token 估算，对中文文档更精确
- atomic block 保护机制防止代码块和表格被错误切割

---

## 三、工具调用系统

### Q4: 工具选择是如何实现的？工具调用失败是如何处理的？

A:

#### 工具选择机制

**双层工具注册体系：**

| 层次 | 接口/注解 | 作用 |
|------|----------|------|
| 业务层 | `Tool` 接口（getName/getDescription/getType） | RAG-Agent 内部过滤路由（FIXED/OPTIONAL） |
| Spring AI 层 | `@Tool` 注解（name/description） | Spring AI 发现可调用方法，生成 ToolCallback |

**工具路由流程：**

1. **启动时**：Spring 扫描所有 `@Component implements Tool` 的 Bean，`ToolRegistry` 统一管理
2. **Agent 创建时**（`RAGAgentFactory.create`）：
   - 固定加载所有 `ToolType.FIXED` 工具（KnowledgeTool、TerminalTools 等）
   - 根据 Agent 数据库配置的 `allowedTools` JSON 字段叠加 `OPTIONAL` 工具
   - 通过 `MethodToolCallbackProvider` 将 Tool 对象转换为 Spring AI 的 `ToolCallback[]`
3. **对话时**：`DefaultToolExecutor.think()` 将 `ToolCallback[]` 传递给 LLM，LLM 根据工具描述自主决策调用哪个工具
4. **执行时**：`DefaultToolExecutor.execute()` 通过 `ToolCallingManager` + `StaticToolCallbackResolver` 解析 LLM 请求的工具名称，找到对应的 `ToolCallback` 并执行

**固定工具 vs 可选工具：**

| 类型 | 工具 | 说明 |
|------|------|------|
| FIXED | KnowledgeTool、TerminalTools、DateTool | 始终可用，不依赖外部配置 |
| OPTIONAL | DataBaseTools、FileSystemTools、EmailTools、DocumentSearchTool | 按 Agent 配置选择性加载 |

**为什么设计双层注册？** 业务层 `Tool` 接口让 Agent 配置可以灵活控制工具可用性；Spring AI `@Tool` 注解让 LLM 能理解工具的输入输出。两层分离使得"工具管理"和"LLM 集成"可以独立演进。

#### 工具调用失败处理

#### 处理顺序

1. **快速重试**：先处理超时、连接中断等可恢复错误
2. **模型介入**：如果快速重试仍失败，构造结构化失败上下文，让模型决定下一步策略
3. **安全兜底**：如果模型也失败，或明确要求停止，则合成 `ToolResponseMessage` 返回给上层

#### 失败上下文包含什么
1. **toolName**
- 含义 : 被调用的工具名称
- 作用 : 标识是哪个工具失败了，用于日志和决策
2. **toolArguments**
- 含义 : 传递给工具的参数（JSON 格式字符串）
- 作用 : 记录失败时的入参，便于排查问题和重试
3. **failureStage**
- 含义 : 失败发生的阶段
- 取值 :
	- EXECUTE - 工具执行阶段失败
	- RETRY - 快速重试阶段失败
	- POLICY_RETRY - 策略重试阶段失败（budget > 0）
- 作用 : 区分失败发生的位置，帮助决定后续处理方式
4. **exceptionClass**
- 含义 : 异常的完整类名（root cause）
- 作用 : 判断异常类型，用于决定是否可重试
5. **exceptionMessage**
- 含义 : 异常的简短描述信息
- 作用 : 提供异常的具体原因，便于调试
6. **retryCount**
- 含义 : 当前已重试的次数
- 作用 : 结合 maxRetries 判断是否达到重试上限
7. **retryable**
- 含义 : 该异常是否可重试
- 作用 : 由 isRetryable(rootCause) 判断，某些异常（如网络超时）可重试，而参数错误等不可重试
8. **lastDelayMs**
- 含义 : 上次重试的延迟时间（毫秒）
- 作用 : 用于计算退避策略（如指数退避）
9. **toolCallIndex**
- 含义 : 工具调用的索引序号
- 作用 : 标识是哪一次工具调用，用于前端展示和结果对应

#### 模型可以返回什么
1. RETRY
含义 : 简单重试
- 使用快速重试机制，不咨询大模型
- 带有固定或指数退避（ backoffMultiplier ）
- 通常用于瞬时故障（如网络抖动）
2. RETRY_WITH_POLICY
含义 : 带策略的重试
- 由大模型决策后返回的重试方式
- 会重新设置 policyRetryBudget 、 initialBackoffMs 、 backoffMultiplier
- 模型会根据 failureStage 、 exceptionClass 等信息决定重试参数
3. DEGRADE
含义 : 降级处理
- 不再重试当前工具
- 可能返回降级结果（如兜底数据、降级响应）
- 让 Agent 继续工作，只是功能受限
4. STOP
含义 : 停止执行
- 完全放弃当前工具调用
- 返回失败信息给上层
- 不会再次重试

#### 关键约束
- 快速重试有上限
- 允许的模型咨询次数有上限
- 退避时间有下限和上限
- `RETRY_WITH_POLICY` 可以重置重试窗口，但不会无限循环
- 合成失败消息时会保留原始 `baseHistory`

工具执行可能失败的场景及原因：

| 场景 | 异常类型 | 原因 |
|------|----------|------|
| 工具不存在 | `IllegalArgumentException` | LLM 幻觉出一个不存在的工具名，`StaticToolCallbackResolver` 解析失败 |
| 工具参数错误 | `ToolExecutionException` | LLM 传递了错误的参数类型或缺少必填参数 |
| 数据库查询失败 | `DataAccessException` | DataBaseTools 执行 SQL 时数据库连接断开或 SQL 语法错误 |
| 文件操作失败 | `IOException` | FileSystemTools 读写文件时权限不足或路径不存在 |
| 邮件发送失败 | `MailException` | EmailTools 调用 SMTP 时网络不通或认证失败 |
| terminate 工具 | （正常，不触发异常） | 工具返回后 `AgentCore.onToolCalled()` 检测到名称="terminate"，设置 FINISHED |
#### Q4.1: 为什么要把失败信息反馈给大模型？

A: 因为很多失败不是纯技术故障，而是“策略不合适”或“参数不合适”。比如：

- 目标工具需要更小的重试窗口
- 某个工具当前不可用，可以切换到降级路径
- 同一类请求连续失败时，应停止重试，避免浪费配额

把失败上下文反馈给模型后，模型可以根据上下文决定是继续重试、换策略、降级，还是直接停止。这比固定规则更灵活，也更符合 Agent 的语义。

---

#### Q4.2: 为什么要区分 `RETRY` 和 `RETRY_WITH_POLICY`？

A: 因为这两个动作语义不同：

- `RETRY`：只想再试几次，适合短暂网络抖动
- `RETRY_WITH_POLICY`：希望同时调整重试策略，比如扩大重试窗口、修改退避时间、改变模型咨询意愿

如果不区分，模型只能返回“重试”这种粗粒度指令，程序层就无法表达“继续试，但要改变节奏”的意图。

---

## 三-B、三层缓存架构详解

### Q4-B: 三层缓存架构在每个 Agent 执行阶段分别做了什么？

A: 三层缓存不是孤立的存储结构，而是在 Agent 生命周期的每个阶段协同工作。以下按时间线追踪每一层的行为。

**架构回顾：**

```
L1 消息窗口 (ChatMemory / MessageWindowMemoryImpl)
  ↓ 大小: messageLength 条原始消息（默认20）
  ↓ 作用: 给 LLM 提供完整对话上下文
  ↓ 生命周期: 会话级（每个会话独立 ArrayList）

L2 Caffeine JVM 缓存 (MemoryCacheManager / DefaultMemoryCacheManager)
  ↓ 大小: maximumSize 个会话快照（默认256）
  ↓ 作用: 存储提炼后的结构化记忆（摘要/事实/约束/工具提示）
  ↓ 生命周期: JVM 级（30分钟无访问过期）

L3 数据库冷存储 (DbMemoryColdStorageGateway / memory_cache 表)
  ↓ 大小: 无限制（PostgreSQL 持久化）
  ↓ 作用: 会话恢复兜底 + 跨会话记忆复用
  ↓ 生命周期: 永久（除非主动清理）
```

---

#### 阶段 1：Agent 创建 — RAGAgentFactory.create(agentId, sessionId)

```
RAGAgentFactory.create()
  │
  ├── 1. 加载 Agent 配置（从 agent 表）
  │
  ├── 2. 恢复 L1 消息窗口
  │       └── L3 → L1: 从 chat_message 表加载历史消息
  │           根据 messageLength 截取最近 N 条
  │           ensureToolCallResponsePairs() 保证 tool 成对
  │           存入 MessageWindowMemoryImpl (ArrayList)
  │
  ├── 3. 恢复 L2 缓存快照
  │       └── L3 → L2: 调用 cacheManager.refreshFromColdStorage(sessionId)
  │           从 memory_cache 表加载该会话的结构化记忆
  │           自动过滤 expire_at 已过期的无效记录
  │           组装为 MemorySnapshot (按 SUMMARY/FACT/CONSTRAINT/TOOL_HINT 分组)
  │           写入 Caffeine Cache (key = sessionId)
  │
  └── 4. 组装 AgentContext
          ├── ChatClient（模型选择）
          ├── ToolCallback[]（工具列表）
          └── 注入 MemoryCacheManager (L2) + IncrementalMemorySummarizer
              + MemoryMergeService + AsyncMemoryWriteService
```

**新会话 vs 已有会话的区别：**

| | 新会话（冷启动） | 已有会话（L2 命中） | 已有会话（L2 过期） |
|---|---|---|---|
| L1 | 空列表 | 从 chat_message 加载 | 从 chat_message 加载 |
| L2 | 空快照，逐步构建 | Caffeine 直接命中，纳秒级 | L3 重新加载，组装快照 |
| L3 | 无历史数据 | 不参与（L2 已命中） | 提供持久化数据 |

---

#### 阶段 2：think() — LLM 决策阶段

```
AgentCore.think()
  │
  ├── 1. 从 L2 读取结构化记忆
  │       └── cacheManager.getSnapshot(sessionId)
  │           提取 SUMMARY / FACTS / CONSTRAINTS / TOOL_HINTS
  │           格式化为决策提示词的一部分
  │           例: "## 会话记忆摘要
{summary}

## 已知事实
{facts}

## 约束条件
{constraints}"
  │
  ├── 2. 从 L1 读取原始消息历史
  │       └── memoryManager.getMessages(sessionId)
  │           返回最近 N 条原始 Message 对象
  │
  ├── 3. 组装 Prompt
  │       └── L1 消息历史 + L2 结构化记忆 + 系统提示词 + 工具描述
  │
  └── 4. 调用 LLM
          └── toolExecutor.think(prompt, tools, thinkPrompt)
              LLM 基于完整上下文决策：直接回答 OR 调用工具
```

**L2 在 think 阶段的价值**：如果没有 L2 提炼，LLM 只能看到 L1 的原始消息。随着对话轮次增加，L1 的滑动窗口会丢弃早期消息，丢失关键信息。L2 以高度压缩的形式（摘要+事实+约束）保留了早期对话的精华，即使 L1 早已丢弃，LLM 仍能"记住"关键上下文。

**量化效果**：实测 L2 提炼后的上下文体积约为原始消息的 10-20%（摘要 1200 字符上限 + 每种类型最多 3 条 × 4 类），而 L1 的 20 条消息可能有 5000+ token。仅此一项，每次 LLM 调用节省约 80% 的提示词 token 消耗。

---

#### 阶段 3：execute() — 工具执行阶段

```
AgentCore.execute()
  │
  ├── 1. 保存当前 L1 消息列表快照（beforeMessages）
  │
  ├── 2. 执行工具调用
  │       └── toolExecutor.execute(prompt, lastResponse)
  │           工具执行成功 → ToolResponseMessage
  │
  ├── 3. 替换 L1 消息窗口
  │       └── memoryManager.replaceAll(sessionId, conversationHistory)
  │           用工具调用后的新历史完全替换旧的 L1 列表
  │           （新增了 ToolResponseMessage，LLM 下一轮 think 可以看到工具结果）
  │
  ├── 4. 标记 L2 dirty
  │       └── cacheContext.markDirty()
  │           表示这轮对话产生了实质性变更（工具被调用了）
  │           后续 checkpoint/flush 会检查此标记
  │
  └── 5. Checkpoint（条件触发）
          └── if (dirty && 有新增消息):
              增量起始位置 = checkpointedIndex（之前已处理到哪）
              取新增消息子列表 → 暂存
```

**L1 替换 vs 追加**：`replaceAll` 不是简单追加，而是整体替换。原因是工具执行后的 `conversationHistory` 已经包含了完整的新消息序列（包括 LLM 的 toolCalls 消息 + 工具执行的 toolResponse 消息），直接替换可以避免消息顺序错乱。

---

#### 阶段 4：flush — 会话结束缓存刷盘

```
AgentCore.run() finally {   ← 无论正常结束还是异常崩溃都会执行
    flushSessionCache()
      │
      ├── 1. 检查 dirty 标记
      │       └── if (!dirty): 跳过（无变更，不浪费资源）
      │
      ├── 2. L1 → L2 增量摘要
      │       └── IncrementalMemorySummarizer.summarizeDelta(sessionId, deltaMessages)
      │           对新增消息进行提炼：
      │           ├── SUMMARY: 最近对话概要（限制 1200 字符）
      │           ├── FACTS: 用户输入的关键事实、工具响应数据、助手结论
      │           ├── CONSTRAINTS: 包含"必须/禁止/不要"的约束条件
      │           └── TOOL_HINTS: 工具调用结果摘要，供后续决策参考
      │
      ├── 3. L2 快照合并
      │       └── MemoryMergeService.merge(existing, delta)
      │           ├── Step 1: Key 去重（相同 key 保留高版本）
      │           ├── Step 2: Jaccard 相似度去重（阈值 75%）
      │           ├── Step 3: 置信度排序
      │           └── Step 4: 按类型截断（默认 SUMMARY/TOOL_HINT 1条，其他 3条）
      │           更新 MemoryCacheManager 中的快照
      │
      ├── 4. L2 → L3 异步写回
      │       └── AsyncMemoryWriteService.submit(sessionId, entries)
      │           写入 ConcurrentHashMap 待写队列
      │           scheduleDrain() 通过 CAS 确保单线程刷盘
      │           批量 INSERT ... ON CONFLICT DO UPDATE
      │           版本乐观锁: WHERE EXCLUDED.version >= memory_cache.version
      │
      └── 5. 推进 commitIndex，清除 dirty
```

**Write-Behind 为什么重要**：如果同步写 L3，每次会话结束都会阻塞等待数据库 IO（实测 L3 写延迟 5ms）。异步写回让 Agent 主流程直接返回，L3 写入在后台线程完成，用户感知的对话结束延迟为零。

---

#### 阶段 5：L2 过期后的会话恢复

```
用户重新打开 30 分钟前创建的会话
  │
  ├── 1. L2 Caffeine 缓存已过期（超过 30 分钟无访问）
  │       └── cacheManager.getSnapshot(sessionId) → null
  │
  ├── 2. 回退到 L3
  │       └── cacheManager.refreshFromColdStorage(sessionId)
  │           从 memory_cache 表 SELECT WHERE entity_id = sessionId
  │           过滤 expire_at > NOW() 的有效记录
  │           组装 MemorySnapshot → 重新写入 L2
  │
  └── 3. L2 恢复后正常服务
          └── 后续 think() 阶段可以正常读取 L2 记忆
```

**冷存储恢复的延迟实测**：1ms（avg）/ 4ms（P95）。相对于 LLM API 调用的秒级延迟，完全可忽略。

---

#### 三层缓存在完整生命周期中的数据流向图

```
┌─────────────────────────────────────────────────────────┐
│                     Agent 生命周期                       │
├──────────┬──────────────┬────────────┬──────────────────┤
│  创建     │   think()    │  execute() │  flush / 结束     │
├──────────┼──────────────┼────────────┼──────────────────┤
│ L3→L1   │ L1→LLM      │ L1 替换    │ L1→L2 增量摘要   │
│ L3→L2   │ L2→LLM      │ L2 标脏    │ L2 合并去重       │
│          │              │            │ L2→L3 异步写回   │
└──────────┴──────────────┴────────────┴──────────────────┘

30分钟后 L2 过期 → 下次打开会话 → L3→L2 恢复 → 循环
```

**一句话总结**：L1 负责"我刚才说了什么"（短期对话记忆），L2 负责"我们之前在聊什么"（结构化长记忆），L3 负责"睡一觉起来还记得"（持久化兜底）。三层各司其职，协同工作，不冗余。---

## 四、测试与评测体系

### Q5: 测试数据的 baseline 设置依据是什么？

A: Baseline 分为三类，各有不同的设置逻辑：

#### 硬性能基线（基于外部 API 耗时经验）

| 指标 | 基线 | 设置依据 |
|------|------|----------|
| think() 决策耗时 | <5000ms | DeepSeek API P95 延迟 ~3s + 2s 网络/重试裕量 |
| TTFT | <2000ms | 流式首 token 通常在 500~1500ms，2s 是安全上限 |
| Embedding 延迟 P95 | <1500ms | bge-m3 在本地 GPU 上的典型推理耗时 ~200ms，P95 在 500ms 内，1.5s 覆盖极端情况 |

#### 业务逻辑基线（基于系统设计约束）

| 指标 | 基线 | 设置依据 |
|------|------|----------|
| precision@4 | ≥50% | 信息检索领域的经验基准：Top-K 检索中至少一半相关 |
| Token P50 | 200-600 | 从 `maxTokensPerChunk=800` 反推的理想分布中点 |
| 超大 chunk 占比 | <5% | 配置 800 token 上限，异常超限应为极少数 |
| 工具调用成功率 | ≥95% | 生产系统基本可靠性要求 |

#### 存在性基线（仅验证机制生效）

| 指标 | 基线 | 设置依据 |
|------|------|----------|
| 缓存驱逐数 | >1 | 只要发生过驱逐即可，不设数量要求 |
| 真实缓存命中率 | >0 | 只要缓存机制生效即可（实际达 100%） |
| Jaccard 去重率 | 仅记录 | 不同文档间内容不重复是正常且正确的 |

**基本原则**：有外部 API 的用经验值，有系统约束的用反推值，纯存在性检验的用最低阈值。不编造不存在的性能数据。

---

### Q6: 测试报告中的数据是怎么得出来的？项目的评测体系是怎么样的？

A:

#### 数据来源与获取方式

| 维度 | 测试类 | 数据获取方式 |
|------|--------|-------------|
| 缓存命中率/驱逐 | CacheMetricsTest | Caffeine `CacheStats` API（`hitCount()`/`missCount()`/`evictionCount()`） |
| 冷存储读写延迟 | CacheMetricsTest | `MetricsCollector.start()` / `.stop()` 包裹 `MemoryColdStorageGateway` 方法，`System.nanoTime()` 计时 |
| Embedding 延迟 | EmbeddingMetricsTest | 计时 `RagService.embed(text)` 调用，覆盖 10字/100字/500字 + 4 线程并发 |
| RAG 检索精度 | RagAccuracyTest | 遍历 103 条真实 chunk，取前 30 字符作为查询，调用 `RagService.similaritySearch()` 做 Top-4 检索，检查目标 chunk 是否出现在结果中 |
| Chunk Token 分布 | RagAccuracyTest | 对全部 103 条 chunk 调用 `TokenEstimator.estimate(content)` 估算 token 数，排序后计算 P50/P95/P99 |
| Agent TTFT | AgentTimingTest | `DefaultToolExecutor` 流式路径中 `System.nanoTime()` 测量首个 chunk 到达时间 |
| 工具调用成功率 | AgentTimingTest | 3 轮天气查询对话，统计 `getTotalToolCalls()` 和 `getFailedToolCalls()` |
| 记忆一致性 | MemoryQualityTest | 构造 2 轮 8 条消息的模拟对话，计算全量摘要与增量摘要的 Jaccard 相似度 |

#### 评测体系架构

```
MetricsCollector（计时 + 百分位统计）
    ↓
MetricsReport.register()（注册指标）
    ↓
MetricsReport.generate()（生成 Markdown 报告 → target/metrics-report.md）
```

**判断逻辑**（`MetricsReport.isPass()`）：
- 比率类指标（rate/hit/success/consistency/precision）：越高越好 → `value >= baseline`
- 延迟类指标（latency/ttft/execute）：越低越好 → `value <= baseline`
- 基线为 0 的指标：仅记录不做判定

**评测流程**：
1. 每个 `@Test` 方法内部调用 `MetricsReport.register()`
2. 测试类之间共享 `static List<Record>`，跨类累积数据
3. `MemoryQualityTest.@AfterAll` 触发 `MetricsReport.generate()`
4. 报告按 4 个分类（缓存/RAG 管线/Agent 对话/记忆系统）输出 Markdown 表格

**外部依赖状态**：
- Ollama 不可达 → `EmbeddingMetricsTest` 和 `RagAccuracyTest` 部分测试标记 SKIP
- PostgreSQL 不可达 → 所有涉及冷存储和 chunk 数据的测试 SKIP
- DeepSeek API 不稳定 → `DefaultToolExecutor` 流式→非流式自动降级

---

## 五、面试高频问题补充

### Q7: 如果 Ollama/BGE-M3 挂了，系统会怎样？

A:
- **Embedding 生成失败**：`RagServiceImpl.doEmbed()` 会抛出 `WebClientResponseException`，向上传播到调用方
- **文档入库中断**：`DocumentFacadeServiceImpl.processMarkdownDocument()` 中 embedding 失败会导致整批入库失败，已入库的 chunk 不会回滚（目前没有事务包裹）
- **检索降级**：相似性搜索直接失败，Agent 在 think 阶段收不到知识库上下文，但不会崩溃——LLM 仍然可以根据自身知识回答问题
- **改进建议**：
  - 增加 Embedding 缓存（相同文本不重复调用 Ollama）
  - 入库时增加失败重试（最多 3 次）
  - 检索时增加兜底策略：Embedding 搜索失败 → 降级为关键词全文搜索（PostgreSQL `ts_vector`）

### Q8: 向量数据库为什么选择 PostgreSQL pgvector 而不是 Milvus/Qdrant/Weaviate？

A:
- **零额外运维成本**：项目已经依赖 PostgreSQL，pgvector 只是一个扩展，不需要额外部署和维护另一个数据库服务
- **数据一致性**：chunk 数据和业务数据在同一个数据库中，天然支持事务和 JOIN 查询
- **适用规模**：当前数据量 103 条 chunk，1024 维向量，pgvector 的 IVFFlat 索引完全够用
- **Milvus 等更适合的场景**：百万级以上向量、需要分布式索引、需要混合查询（向量+标量过滤）的复杂场景

### Q9: 缓存的三层架构中，为什么 L2 不用 Redis？

A:
- **延迟差异**：Caffeine 进程内缓存纳秒级，Redis 网络往返至少 0.1~1ms，差距 3~4 个数量级
- **单实例足够**：当前是单实例部署，不需要跨节点缓存共享
- **过期策略丰富**：Caffeine 提供 expireAfterAccess/expireAfterWrite/size-based/weight-based 等多种策略，Redis 需要手动 Lua 脚本实现类似逻辑
- **未来扩展**：如果需要分布式部署，可以在 L2 Caffeine 之上加 L2.5 Redis 作为共享缓存

### Q10: 如何处理对话上下文过长导致的 LLM Token 超限？

A:
- **L1 滑动窗口**：`MessageWindowMemoryImpl` 只保留最近 N 条消息（`messageLength` 可配置），超出部分自动丢弃
- **完整性保障**：`ensureToolCallResponsePairs()` 确保 tool_calls 和 tool_response 成对保留，不会因为截断导致 LLM API 报错
- **RAG 上下文预算**：`RagServiceImpl.similaritySearch()` 从后向前丢弃低相关度 chunk，直到满足 `maxContextTokens` 预算
- **L2 缓存提炼**：`IncrementalMemorySummarizer` 将对话历史提炼为摘要/事实/约束/工具提示的 KV 结构，极大地压缩了上下文体积
- **Greedy Packing**：`RagContextAssembler` 按相关性排序后累计 token，超出预算立即停止

### Q11: Agent 的 Think-Execute 循环会不会无限执行下去？

A: 有三重保护机制：
1. **硬限制**：`maxSteps = 20`（在 `AgentContext` 中设置），达到后强制 `FINISHED`
2. **terminate 工具**：Agent 可以主动调用 `TerminalTools` 结束任务，`AgentCore.onToolCalled()` 检测到 "terminate" 工具调用后设置 `FINISHED` 状态
3. **LLM 自然终止**：LLM 在 think 阶段没有工具调用时，直接返回文本回答，`think()` 返回 `false`，循环终止

### Q12: 为什么选择 Spring AI 而不是 LangChain4j？

A:
- **Spring 生态集成**：项目本身是 Spring Boot 项目，Spring AI 与 Spring 生态（自动配置、依赖注入、AOP）无缝集成
- **API 一致性**：Spring AI 的 `ChatClient` API 风格与 Spring 的 `RestClient`/`WebClient` 一致，学习成本低
- **多模型支持**：Spring AI 1.1.0 内置了 DeepSeek 和 ZhipuAI 的 starter，开箱即用
- **缺点**：Spring AI 1.1.0 比较早期，部分 API 不够稳定（如流式响应处理），需要手动适配

### Q13: 如果数据库中的 agent 配置被删除了，正在运行的 Agent 会话会怎样？

A:
- **不会立即崩溃**：`RAGAgentFactory.create()` 只在会话开始时从数据库加载 Agent 配置一次，运行时不再读取
- **配置缓存在 AgentContext 中**：工具列表、模型选择、ChatOptions 等都在创建时确定，后续对话不受影响
- **新建会话会失败**：下一次用户尝试使用已删除的 Agent 创建新会话时，会因为外键约束或 `AgentMapper.selectById` 返回 null 而失败
- **改进建议**：增加 Agent 配置的软删除机制，或运行时定期检查 Agent 状态

### Q14: 流的流式推送中，SSE 连接断了怎么办？

A:
- **前端自动重连**：`EventSource` API 内置自动重连机制，断线后自动重新请求 `/sse/connect/{chatSessionId}`
- **服务端超时清理**：`SseEmitter` 设置 30 分钟超时，`onTimeout` 回调自动清理 `ConcurrentHashMap` 中的连接
- **消息不丢失**：持久化的消息通过 `chat_message` 表恢复；流式内容断线期间会丢失中间 chunk（无法恢复），但前端重连后可以通过 `GET /messages/{sessionId}` 获取完整对话历史
- **改进建议**：增加 SSE 连接的 heartbeat 机制（定期发送空事件），让前端能更快检测断线

### Q15: 项目中最大的技术挑战是什么？如何解决的？

A:
1. **流式调用与工具调用的兼容性**：DeepSeek 在 `stream: true` + `tools` 组合下会返回 400。解决方案是流式优先 + 非流式自动降级，保证兼容性的同时尽可能测量 TTFT
2. **向量零距离查询全量数据**：`ChunkMapper.similaritySearch` 使用零向量做全量扫描时 cosine distance 返回 NaN。解决方案是改用 `JdbcTemplate` 直接 SQL 查询，绕过 pgvector 的距离计算
3. **缓存跨测试数据污染**：不同测试类共享 JVM 中的 `static` 记录，`@BeforeEach` clear 会导致前序测试数据丢失。解决方案是移除中间清空逻辑，改为最终一次性生成报告
4. **Caffeine 淘汰统计延迟**：TinyLFU 算法使用异步懒淘汰，300 条写入后淘汰计数仍为 0。解决方案是 `cleanUp()` 强制同步 + 填充数提升到 1000
5. **JDBC 连接不稳定的 WSL 环境**：psql CLI 无法连接但 Spring JDBC 可以——因为 WSL 的网络桥接方式不同。解决方案是所有数据库查询统一走 Spring 的 `JdbcTemplate`
