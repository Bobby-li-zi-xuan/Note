# 01 · Agent 对话引擎

## 一、架构定位

Agent 对话引擎是整个系统的"大脑皮层"——负责编排每一轮对话从接收到完成的全部流程。它不自己调模型、不自己查数据库，而是通过调度 ToolExecutor、ChatMemoryManager、MessagePublisher 完成工作。

### 核心设计原则

**模型负责决策，程序负责控制边界。** 这是整个 Agent 引擎的设计哲学。LLM 决定"要不要调工具""调哪个工具"，但真正的工具执行、失败恢复、消息持久化、缓存刷写全由代码控制。这样既不限制 LLM 的灵活性，又保住了系统的稳定性。

---

## 二、Think-Execute 双阶段编排

### 2.1 为什么不用 Spring AI 的自动工具执行

Spring AI 提供了 `internalToolExecutionEnabled` 开关，开启后 think 和 execute 在一次 `call()` 里完成。问题是这套"一步到位"的模式无法插入业务逻辑：

- 工具结果需要持久化到数据库
- 工具执行前后需要推送 SSE 状态
- 工具失败需要重试和兜底
- 对话变更需要触发缓存 checkpoint

所以当前实现关闭了内建工具执行，改成手动两步走：

```java
// MultiChatClientConfig 或 DefaultToolExecutor 中
DefaultToolCallingChatOptions.builder()
    .internalToolExecutionEnabled(false)
    .build();
```

### 2.2 think()：决策阶段

think() 的核心任务是：**把当前对话上下文发给 LLM，让 LLM 决定要不要调用工具。**

```java
private boolean think() {
    // 1. 从 L2 缓存加载结构化记忆，拼入 think prompt
    String cachedMemoryContext = buildCachedMemoryContext();

    // 2. 构建决策提示词，告知可用的知识库和缓存记忆
    String thinkPrompt = """
            你是智能决策模块，根据对话上下文决定下一步动作。
            【知识库信息】%s
            【轻量缓存记忆】%s
            """.formatted(availableKbs, cachedMemoryContext);

    // 3. 调用 LLM（先流式尝试，失败降级非流式）
    boolean needsExecution = toolExecutor.think(prompt, tools, thinkPrompt);

    // 4. 无工具调用 → 创建流式消息推送给前端
    //    有工具调用 → 保存消息，标记缓存脏
}
```

关键细节：think prompt 不只是让 LLM 决定是否调工具，还**注入当前知识库列表和缓存记忆**，让 LLM 知道"我有什么武器"和"对话进行到哪了"。没有这些上下文，LLM 就会盲目决策。

### 2.3 execute()：执行阶段

```java
private void execute() {
    // 1. 保存执行前的消息快照，用于后续对比是否变更
    List<Message> before = new ArrayList<>(memoryManager.getMessages(sessionId));

    // 2. 执行工具调用（含失败恢复闭环）
    List<Message> after = toolExecutor.execute(prompt, lastResponse);

    // 3. 整体替换记忆（工具执行可能产生多条新消息）
    memoryManager.replaceAll(sessionId, after);

    // 4. 如果对话有变化，标记缓存脏 + checkpoint
    if (isConversationChanged(before, after)) {
        cacheContext.markDirty();
    }
    checkpointSessionCache();
}
```

### 2.4 完整循环

```
AgentCore.run()
  └─ for step in 1..maxSteps (默认20):
       ├─ step()
       │   ├─ think()   → LLM 决策
       │   │   ├─ 有工具调用? → execute()
       │   │   └─ 无工具调用? → 流式输出最终答案 → FINISHED
       │   └─ checkpoint 缓存
       └─ 超过最大步数? → FINISHED + 警告日志
  └─ finally:
       flushSessionCache()   ← 无论如何保证刷盘
       publisher.sendStatus(AI_DONE)
```

---

## 三、Agent 状态机

```
IDLE → PLANNING → THINKING → {EXECUTING → THINKING} → FINISHED
                                                         ↓ (任何阶段异常)
                                                       ERROR
```

每个状态都会触发 SSE 推送，前端实时展示 Agent 当前在做什么。这不是花架子——在真实使用场景里，Agent 可能卡在某个工具调用上，前端只有知道当前状态才能给用户正确的等待提示。

---

## 四、多模型集成

### 4.1 设计

```java
// ChatClientRegistry 按模型名缓存 ChatClient
chatClientRegistry.get("deepseek-chat")  → DeepSeekChatModel
chatClientRegistry.get("glm-4.6")        → ZhiPuAiChatModel
```

每个 Agent 配置了 `model` 字段，Factory 创建时会按模型名选择对应的 ChatClient。这就是说你可以在系统里创建多个 Agent——一个用 DeepSeek 做知识检索、一个用智谱做日常聊天——互不冲突。

### 4.2 流式降级

DeepSeek 的流式 API 在携带 tools 参数时偶尔返回 400。当前实现做了**两层兼容**：

```java
try {
    return thinkStreaming(prompt, toolCallbacks, thinkPrompt); // 流式 + TTFT 测量
} catch (Exception e) {
    this.lastTtftMs = -1;
    return thinkNonStreaming(prompt, toolCallbacks, thinkPrompt);  // 降级非流式
}
```

流式调用的另一个价值是**测量到达前端并可显示的首 token 延迟（TTFT）**——这是衡量 LLM API 响应速度的关键指标：

```java
long startNanos = System.nanoTime();
flux.doOnNext(chunk -> {
    if (firstToken[0]) {
        ttftNanos[0] = System.nanoTime() - startNanos;
        this.lastTtftMs = ttftNanos[0] / 1_000_000;
        firstToken[0] = false;
    }
});
```

---

## 五、SSE 实时推送

Agent 执行的每一步都通过 SSE 推送到前端，而不是等全部执行完才一起返回。

### 推送的消息类型

| 类型 | 时机 | 含义 |
|------|------|------|
| AI_PLANNING | step() 开始 | 正在规划下一步 |
| AI_THINKING | think() 开始 | 正在调用 LLM 决策 |
| AI_EXECUTING | execute() 开始 | 正在执行工具调用 |
| AI_MESSAGE | 流式增量 | LLM 输出的文字片段 |
| AI_PENDING_MESSAGES | 工具结果持久化后 | 批量推送新消息 |
| AI_DONE | run() 结束 | 本轮对话完成 |

### 流式输出处理

当 LLM 直接回复（无工具调用）时，不是等全部文字生成完再推送，而是：

1. 先创建一条空内容的 `AssistantMessage` 入库，获得 `messageId`
2. LLM 每吐出一个 token，立即通过 SSE 推送增量块
3. 全部完成后，批量推送到前端展示

这样用户看到的是"逐字出现"的回复体验。

---

## 六、Agent 装配工厂

`RAGAgentFactory.create(agentId, sessionId)` 是唯一的 Agent 构造入口，遵循"装配"而非"创建"的原则：

```
1. loadAgent(agentId)           → Agent 实体（模型名、系统提示词、工具白名单）
2. toAgentConfig(agent)         → AgentDTO（解析 JSON 字段）
3. loadMemory(sessionId)        → 从冷存储恢复最近 N 条历史
4. resolveRuntimeKnowledgeBases → 按白名单过滤知识库
5. resolveRuntimeTools          → 固定工具 + 可选工具叠加
6. buildToolCallbacks           → 转成 Spring AI ToolCallback[]
7. buildAgentRuntime            → 组装 RAGAgent
```

关键决策点：
- 固定工具（FIXED）始终可用，可选工具按白名单叠加——既灵活又安全
- 冷存储加载消息时按 `messageLength` 限制条数，防止上下文爆炸
- 启动时自动引导 L2 缓存——如果冷存储有记忆则预热，如果都没有则从当前消息生成初始摘要

---

## 七、面试问答

### Q: 为什么你要自己实现 Agent 循环，而不是用 LangChain 或现成框架？

> 其实 Spring AI 本身就有自动工具执行的能力，我完全可以开着用。但我有两个考虑：一是控制力——我需要把工具执行分成 think 和 execute 两个阶段，在中间插入消息持久化和 SSE 推送；二是失败恢复——自动执行失败只会抛异常，我需要一个带预算控制的三层恢复机制。事实证明这个决策是对的，后面加失败恢复、加缓存 checkpoint，都是因为这两个阶段是分开的才能做。

> LangChain 我当时也考虑过，但它太重了。对我来说，Agent 循环的核心就是"发 prompt → 解析 toolCall → 执行工具 → 再发 prompt"，这个逻辑一两百行代码就写清楚了，没必要引入整个框架。

### Q: 你提到状态机，ERROR 状态下会发生什么？

> 如果 Agent 进入 ERROR 状态，最外层的 finally 块确保缓存和消息一定会刷盘。也就是说即使崩了，已经产生的对话记录不会丢。然后 SSE 会推送错误状态给前端。

> 但这里有个待改进的点——现在 ERROR 之后没有自动重试机制。如果是 LLM API 临时不可用导致的报错，用户需要手动重新发消息触发新一轮对话。更好的做法是在 ERROR 后让 AgentCore 自己去等待一下再恢复。

### Q: 流式和非流式你是怎么选的？

> 我优先用流式，有两个原因。一是用户体验——前端能看到逐字输出，不用傻等着。二是可以测 TTFT（首 token 延迟），这个指标能直接反映 LLM API 的响应速度——如果 TTFT 突然飙升，说明 API 那边可能有问题。

> 但 DeepSeek 的流式 API 偶尔在携带 tools 参数时返回 400，所以我做了降级——流式失败后自动切非流式。代价是流式失败的那一轮测不到 TTFT，但至少对话不会中断。

### Q: Factory 为什么要做这么复杂的装配，而不是用 Builder 模式？

> Factory 和 Builder 解决的问题不一样。Builder 适合"按参数构造"，适合参数多但创建逻辑简单的场景。Factory 要做的事情更多——查数据库、解析 JSON、过滤白名单、预热缓存——这些都是有依赖顺序的步骤，用 Builder 容易写出不完整的对象。

> 如果用 Builder，调用方需要自己保证"先设 chatClient 再设 tools 再设 memoryManager"，顺序乱了构造出来的 Agent 就跑不起来。Factory 保证每一步都按正确顺序执行，调用方只用传 agentId 和 sessionId。

### Q: think prompt 每次都要注入知识库列表和缓存记忆，会不会很浪费 token？

> 确实会消耗 token。这也是为什么我在缓存记忆里只取每种类型的**第一条**（最新的），而不是全部展开。知识库列表也是精简过的——只列名称和 ID，不展开元信息。

> 如果要做更激进的优化，可以对知识库列表做缓存——因为同一个 Agent 的知识库通常是不变的，不需要每轮都重复注入。

### Q: 如果 LLM 一直说"需要调工具"，会不会无限循环？

> 不会，有 maxSteps 兜底，默认 20 步。到了步数还没结束就会强制终止。另外 terminate 工具可以主动告诉 AgentCore"这一轮可以停了"。

> 还有一个隐式退出条件：如果 LLM 某一轮 think 返回了不带 toolCall 的文本，说明它认为可以直接回答了，这也会终止循环。

### Q: 多模型切换有什么坑？

> DeepSeek 和智谱的 API 行为不完全一样。比如 DeepSeek 的流式在带 tools 时会 400，智谱不会。还有一个是 tool_choice 参数——不同模型对 auto/none 的支持程度不同。这就是为什么 ChatClientRegistry 按模型名注册，每个模型都是独立的 ChatClient 实例，参数可以各自调优。

### Q: 你们是怎么处理 token 窗口超限的？

> 目前有两层防护。一是 Factory 加载历史时按 messageLength 限制条数，不会把整个会话历史全塞进去。二是 MessageWindowMemoryImpl 有 maxMessages 限制，超出就丢掉最早的。

> 但这其实是基于"消息条数"的估算，不是真正的 token 计数。如果用不同模型（tokenizer 不同），可能会估算不准。更好的做法是做一个 token 计数器——可以在每次 think 前算一下当前窗口的 token 量，超了就裁剪。

### Q: 如果面试官问"Agent 上下文太长怎么办"，怎么答？

> 我会分三层来讲。第一层是消息窗口（L1）——用 MessageWindow 截断，只保留最近 N 条。第二层是缓存记忆（L2）——把历史对话压缩成 summary/facts/constraints 结构化摘要，注入到 think prompt 里，不用每次都看原始消息。第三层是冷存储（L3）——完整历史存在数据库，需要时随时查。

> 这个设计的巧妙之处在于：LLM 看到的是"摘要 + 最近消息"，而不是全部历史。这样既保留了关键上下文，又不会因为历史太长撑爆 token 窗口。
