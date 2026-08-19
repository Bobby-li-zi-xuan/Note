RAG-Agent
├── data/                                                                                                          **// 本地数据与缓存目录**
│   └── cache/                                                                                                  **// 会话缓存存储路径**
│       └── memory-cache.json                                                                        **// FileBasedMemoryColdStorageGateway 落盘的JSON文件**
├── docs/                                                                                                          **// 项目架构设计与规划文档**
│   └── superpowers/                                                                                         **// 功能模块设计（包括轻量缓存方案、重构设计等）**
├── pom.xml                                                                                            **// 项目Maven依赖配置**
├── src/main/java/com/person/ragagent
│ ├── RagAgentApplication.java                                                                  **// 项目启动入口**
│ ├── agent/                                                                                                  **// 【核心领域】Agent智能体核心模块（大脑）**
│ │ ├── core/                                                                                                 **// Agent核心编排器与运行时上下文**
│ │ ├── cache/                                                                                                **// 轻量缓存系统（L2 Cache），持久记忆存储**
│ │ ├── memory/                                                                                             **// 对话记忆管理器（滑动窗口）**
│ │ ├── messaging/                                                                                        **// 消息发布器，对接SSE实时推送**
│ │ ├── tool/                                                                                                   **// 工具调用执行器（Think-Execute编排）**
│ │ └── tools/                                                                                                 **// 【核心领域】Agent可调用的工具集**
│ │     └── test/                                                                                             **// 测试/演示用工具**
│ ├── config/                                                                                                     **// 【基础设施】Spring配置类（多模型、异步、CORS）**
│ ├── controller/                                                                                             **// 【接口层】RESTful接口，前端交互入口**
│ ├── converter/                                                                                             **// 【转换层】对象转换器：Entity↔DTO↔VO**
│ ├── event/                                                                                                     **// 【事件驱动】Spring事件，解耦消息发送与Agent执行**
│ │ └── listener/                                                                                            **// 事件监听器**
│ ├── exception/                                                                                             **// 【异常处理】自定义业务异常+全局异常捕获**
│ ├── mapper/                                                                                                 **// 【数据访问层】MyBatis Mapper接口，数据库操作**
│ ├── message/                                                                                               **// 【消息模型】SSE消息结构体定义**
│ ├── model/                                                                                                   **// 【数据模型】全项目数据模型定义**
│ │ ├── common/                                                                                             **// 通用模型：统一API返回体ApiResponse**
│ │ ├── dto/                                                                                                     **// 数据传输对象：Service层内部传输**
│ │ ├── entity/                                                                                                 **// 数据库实体：与表结构一一对应**
│ │ ├── request/                                                                                               **// 接口入参：前端POST/PUT请求体**
│ │ ├── response/                                                                                             **// 接口出参：接口返回给前端的结构体**
│ │ └── vo/                                                                                                       **// 视图对象：返回给前端的展示对象**
│ ├── service/                                                                                                   **// 【业务逻辑层】业务逻辑实现，分接口+实现类**
│ │ └── imp/                                                                                                     **// 业务实现类**
│ └── typehandler/                                                                                           **// 【数据类型转换】MyBatis自定义类型处理器（pgvector）**
├── src/main/resources
│ ├── application.yaml                                                                                 **// 应用配置文件：数据库、AI模型、邮件、文档存储等**
│ └── Mapper/                                                                                                 **// MyBatis映射文件：SQL语句定义**
├── src/test/                                                                                                     **// 单元测试**
└── ui/                                                                                                              **// 前端项目（React + Vite + TypeScript）**
    └── src/
        ├── api/                                                                                                  **// API请求封装**
        ├── components/                                                                                   **// React组件**
        ├── contexts/                                                                                        **// React Context状态管理**
        ├── hooks/                                                                                              **// 自定义Hook**
        ├── layout/                                                                                             **// 布局组件**
        ├── types/                                                                                               **// TypeScript类型定义**
        └── utils/                                                                                               **// 工具函数**

---

## 一、项目概述与技术栈

### 后端技术栈
| 技术 | 版本/说明 |
|------|-----------|
| Java | 23 |
| Spring Boot | 3.2.9 |
| Spring AI | 1.1.0（DeepSeek + Zhipu AI） |
| MyBatis | 3.0.3（mybatis-spring-boot-starter） |
| PostgreSQL | 含 pgvector 向量扩展 |
| Lombok | 1.18.32 |
| SSE | Spring MVC SseEmitter（服务端推送） |
| Flexmark | 0.64.8（Markdown解析） |
| Spring Mail | QQ邮箱异步发送 |
| Caffeine | 高性能JVM缓存库（替代ConcurrentHashMap） |

### 前端技术栈
| 技术 | 版本/说明 |
|------|-----------|
| React | 19.2.0 |
| TypeScript | 5.9.3 |
| Vite | rolldown-vite 7.2.5 |
| Ant Design | 6.0.0 |
| Ant Design X | 2.0.0（AI对话组件） |
| Tailwind CSS | 4.1.17 |
| React Router | 7.9.6 |

---

## 二、数据库设计

数据库实体类位于 [model/entity/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/model/entity)

| 实体类 | 说明 |
|--------|------|
| Agent | 智能体配置，描述它是谁、行为基调以及能做什么 |
| ChatMessage | 对话消息记录，存储每轮对话的语义片段（含工具调用信息） |
| ChatSession | 聊天会话，维护完整对话的标题、关联Agent等基本信息 |
| Chunk | 向量块表，存储文档分块后的嵌入向量与原文内容 |
| Document | 原始文档管理，记录来源、解析方式等元信息 |
| KnowledgeBase | 知识库，管理文档集合并配置检索参数 |

### 数据层

#### Mapper接口
>提供访问数据库的增删改查操作，同时将查询结果映射到Java实体类

[mapper/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/mapper)
├── KnowledgeBaseMapper.java
├── DocumentMapper.java
├── ChunkMapper.java
├── AgentMapper.java
├── ChatSessionMapper.java
└── ChatMessageMapper.java

#### Mapper XML文件
>在XML文件中配置SQL语句

[resources/Mapper/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/resources/Mapper)
├── KnowledgeBaseMapper.xml
├── DocumentMapper.xml
├── ChunkMapper.xml
├── AgentMapper.xml
├── ChatSessionMapper.xml
└── ChatMessageMapper.xml

#### 类型处理器
[typehandler/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/typehandler)
└── PgVectorTypeHandler.java

>`PgVectorTypeHandler` 在Java的 `float[]` 类型与PostgreSQL数据库中的 `vector` 类型之间进行转换，是实现向量检索（RAG）的关键组件。

---

## 三、业务层逻辑

### 1. DTO类
>用于Service层内部数据传输，隔离数据库实体与业务逻辑

[model/dto/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/model/dto)
├── KnowledgeBaseDTO.java
├── DocumentDTO.java
├── ChunkDTO.java
├── AgentDTO.java
├── ChatSessionDTO.java
└── ChatMessageDTO.java

### 2. 转换器
>实现不同数据模型间的相互转换，采用单向依赖原则（Entity ↔ DTO ↔ VO）

[converter/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/converter)
├── KnowledgeBaseConverter.java
├── DocumentConverter.java
├── ChunkConverter.java
├── AgentConverter.java
├── ChatSessionConverter.java
└── ChatMessageConverter.java

### 3. 服务接口
>封装各个模块的业务，采用门面模式（Facade Pattern）

[service/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/service)

**基础服务：**
- `DocumentStorageService` — 文档存储服务接口
- `MarkdownParserService` — 解析Markdown文件，提取标题和对应的内容
- `RagService` — RAG（检索增强生成）服务，包括文本嵌入向量生成、语义相似性搜索、距离过滤、Jaccard去重和上下文组装
- `EmailService` — 邮件异步发送服务
- `SseService` — SSE实时推送服务，用于Agent边思考边输出

**门面服务：**
- `KnowledgeBaseFacadeService` — 知识库的创建、查询、更新、删除
- `DocumentFacadeService` — 文档获取、创建、上传、删除、更新
- `AgentFacadeService` — Agent增删改查
- `ToolFacadeService` — 统一工具访问接口，分固定工具和可选工具
- `ChatSessionFacadeService` — 聊天会话增删改查
- `ChatMessageFacadeService` — 聊天消息增删改查，含事件发布触发Agent执行
- `SseService` — SSE连接管理与消息推送

### 4. 服务接口实现

[service/imp/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/service/imp)
├── KnowledgeBaseFacadeServiceImpl.java
├── DocumentFacadeServiceImpl.java
├── DocumentStorageServiceImpl.java
├── MarkdownParserServiceImpl.java
├── **RagServiceImpl.java** — RAG核心实现
├── EmailServiceImpl.java
├── AgentFacadeServiceImpl.java
├── ChatSessionFacadeServiceImpl.java
├── ChatMessageFacadeServiceImpl.java
├── SseServiceImpl.java
└── ToolFacadeServiceImpl.java

#### RAG服务实现详解 ([RagServiceImpl](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/service/imp/RagServiceImpl.java))

**嵌入向量生成：**
- 通过本地部署的 **BGE-M3** 模型（`http://localhost:11434`）生成1024维嵌入向量
- `RagServiceImpl.doEmbed()` 通过 WebClient 调用 Ollama `/api/embeddings` 接口
- 支持文本 → `float[]` 向量的双向转换

**写入流程（文档入库）：**
- 文档处理：`DocumentFacadeServiceImpl.processMarkdownDocument()` 调用 `MarkdownParserService` 解析 Markdown 文件为 `List<MarkdownSection>`（title + content）
- 语义分块：`MarkdownChunkComposer.compose(title, content, maxTokensPerChunk)` 将每个章节拆分为 1..N 个语义 chunk
  - 每个 chunk 以 `# 标题\n\n` 开头，确保嵌入向量包含层级语义
  - 按段落边界切分，fenced code block 和表格视为原子块不分割
  - 单段超长时按句子边界或固定长度切分
  - Token 估算采用保守策略：ASCII 4字符≈1 token，CJK 1字符≈2 token
- 向量生成：对每个 chunk 文本调用 `ragService.embed(chunkText)` 生成嵌入向量
- 存储：通过 `ChunkMapper.insert()` 写入 `chunk_bge_m3` 表，`PgVectorTypeHandler` 自动处理向量类型转换
- Metadata 记录：`{"sectionTitle":"...","chunkIndex":0,"chunkCount":3}`

**读取流程（相似性检索）：**
- 查询文本 → `doEmbed(query)` → 生成查询向量 → `toPgVector()` 转为字符串格式
- SQL 检索：`ChunkMapper.similaritySearch(kbId, vectorLiteral, candidateLimit)`
  - 使用 pgvector `<->` 操作符（L2 欧几里得距离）计算相似度
  - 按 `relevance_score` 升序排列，距离越小越相似
  - `candidateLimit`（默认8）控制候选召回数
- Token 预算控制：
  - 从知识库元数据读取 `maxContextTokens`（默认1200）
  - `estimateTokens()` 按字符级估算 token 数（ASCII 4字符≈1 token，非ASCII 1字符≈1 token）
  - 从后向前丢弃低相关度块，直到满足预算
- 返回：`List<String>`（每个 chunk 的 content 字段）

**关键常量：**
- `RagServiceImpl.DEFAULT_CANDIDATE_LIMIT = 8` — SQL 返回的最大候选数
- `KnowledgeBase.DEFAULT_MAX_CONTEXT_TOKENS = 1200` — 默认 token 预算，可通过知识库 metadata 覆盖

**RAG 检索管线优化方向（参考 `docs/superpowers/specs/2026-05-07-rag-relevance-optimization-design.md`）：**
- **SQL 层**：切换 `<=>` 余弦距离操作符，增加 `maxDistance` WHERE 过滤
- **过滤层**：`RagSearchFilter` 实现距离过滤 + 两段式 Jaccard 去重（标题+正文）
- **组装层**：`RagContextAssembler` 实现稳定排序 → greedy packing → 相邻合并 → 编号输出
- **入库层**：`MarkdownChunkComposer` 替代原有标题 embedding，改为 chunk 文本（标题+正文）embedding
- **配置化**：所有阈值通过 `application.yaml` 中的 `rag.search.*` 和 `rag.chunk.*` 配置

### 5. VO类
>封装前端展示的数据

[model/vo/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/model/vo)
├── KnowledgeBaseVO.java
├── DocumentVO.java
├── AgentVO.java
├── ChatSessionVO.java
└── ChatMessageVO.java

---

## 四、控制层

### 1. 请求和响应类
- 封装前端提交的数据并进行校验
- 转换为DTO进行业务处理

[model/request/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/model/request)
├── CreateKnowledgeBaseRequest.java
├── UpdateKnowledgeBaseRequest.java
├── CreateDocumentRequest.java
├── UpdateDocumentRequest.java
├── CreateAgentRequest.java
├── UpdateAgentRequest.java
├── CreateChatSessionRequest.java
├── UpdateChatSessionRequest.java
├── CreateChatMessageRequest.java
└── UpdateChatMessageRequest.java

[model/response/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/model/response)
├── CreateKnowledgeBaseResponse.java
├── GetKnowledgeBasesResponse.java
├── CreateDocumentResponse.java
├── GetDocumentsResponse.java
├── CreateAgentResponse.java
├── GetAgentsResponse.java
├── CreateChatSessionResponse.java
├── GetChatSessionResponse.java
├── GetChatSessionsResponse.java
├── CreateChatMessageResponse.java
└── GetChatMessagesResponse.java

### 2. 控制器
>RESTful API入口，调用Service层处理请求

[controller/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/controller)
├── HomeController.java — 首页/健康检查
├── KnowledgeBaseController.java — 知识库CRUD
├── DocumentController.java — 文档上传与管理
├── AgentController.java — Agent配置CRUD
├── ToolController.java — 工具列表查询
├── ChatSessionController.java — 会话管理
├── ChatMessageController.java — 消息管理（发送消息触发Agent）
└── SseController.java — SSE连接建立

---

## 五、核心功能实现 — Agent系统

### 整体架构

Agent系统采用分层解耦架构：

```
RAGAgent (门面) → AgentCore (核心编排器)
                    ├── ChatMemoryManager (对话记忆管理)
                    ├── ToolExecutor (Think-Execute工具编排)
                    ├── MessagePublisher (SSE消息推送)
                    ├── MemoryCacheManager (L2轻量缓存 - 基于Caffeine)
                    ├── MemorySnapshot (会话级聚合快照)
                    ├── IncrementalMemorySummarizer (增量摘要器)
                    ├── MemoryMergeService (记忆合并与淘汰)
                    ├── SessionCacheContext (缓存进度跟踪)
                    └── AsyncMemoryWriteService (异步写回服务)
```

### 模块详解

#### agent/ 目录结构

[agent/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/agent)
├── RAGAgent.java — 门面类，对外保留入口和元信息，内部委托 AgentCore
├── RAGAgentFactory.java — 工厂类，负责从数据库加载Agent配置并组装依赖
├── AgentState.java — Agent状态枚举
└── core/
    ├── AgentContext.java — 运行时上下文（会话ID、模型客户端、工具列表、知识库列表等）
    └── AgentCore.java — 核心编排器，控制 Think-Execute 循环

#### AgentState 状态机

[AgentState.java](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/agent/AgentState.java)
```
IDLE → PLANNING → THINKING → EXECUTING → THINKING → ... → FINISHED
                                    ↓
                                 ERROR
```
- `IDLE` — 空闲，等待启动
- `PLANNING` — 规划处理步骤
- `THINKING` — 思考下一步操作（调用LLM决策）
- `EXECUTING` — 执行工具调用
- `FINISHED` — 正常结束
- `ERROR` — 异常结束

#### AgentCore 核心循环 ([AgentCore.java](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/agent/core/AgentCore.java))

```java
run() {
    for (int step = 0; step < maxSteps; step++) {
        step() {
            think() {
                // 1. 组装轻量缓存记忆上下文（从MemoryCacheManager获取快照）
                // 2. 向LLM投喂历史消息 + 决策提示词（含知识库信息、缓存记忆）
                // 3. LLM返回：直接回答 OR 工具调用请求
                // 4. 无工具调用 → 创建流式消息，逐字推送前端
                // 5. 有工具调用 → 持久化消息，进入 execute()
            }
            execute() {
                // 1. 执行LLM请求的工具调用
                // 2. 将工具响应追加到对话历史
                // 3. 更新SessionCacheContext（标记dirty、推进checkpoint）
                // 4. 整体替换记忆 → 回到 think() 继续循环
            }
        }
    }
    // 运行结束后：refreshSessionCache() → 增量摘要 → 合并 → 异步写回冷存储
}
```

#### RAGAgentFactory 装配流程 ([RAGAgentFactory.java](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/agent/RAGAgentFactory.java))

1. 从数据库加载Agent配置（名称、描述、系统提示词、关联模型等）
2. 解析Agent配置的JSON选项（ChatOptions）
3. 根据配置的模型名称从 `ChatClientRegistry` 获取对应 `ChatClient`
4. 解析运行时可用工具：固定工具全部加载 + 根据配置叠加可选工具
5. 将业务层 `Tool` 对象转换为 Spring AI 的 `ToolCallback`
6. 解析运行时知识库列表
7. 从冷存储加载历史对话消息，恢复到 `ChatMemory` 中作为会话上下文
8. 从冷存储加载L2缓存快照到 `MemoryCacheManager`
9. 组装 `AgentContext`，创建 `RAGAgent` 实例（注入增量摘要器、合并服务等缓存组件）

### 对话记忆管理 — Memory

[agent/memory/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/agent/memory)
├── ChatMemoryManager.java — 记忆管理器接口
└── MessageWindowMemoryImpl.java — 基于 Spring AI MessageWindowChatMemory 的实现

- 采用**滑动窗口策略**，只保留最近N条消息（由Agent配置的 `messageLength` 控制）
- **完整性保障**：`ensureToolCallResponsePairs()` 方法确保 tool_calls 和 tool_response 成对保留，避免截断时破坏工具调用完整性
- 每个会话创建独立的记忆实例，避免跨会话污染

### L2 轻量缓存系统 — Cache

[agent/cache/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/agent/cache)
├── MemoryEntry.java — 内存记忆条目（key/value/scope/type/version/expire/source/confidence）
├── MemoryType.java — 记忆类型枚举（SUMMARY/FACT/CONSTRAINT/TOOL_HINT/PROMPT_FRAGMENT）
├── MemoryScope.java — 记忆作用域枚举（SESSION/USER/AGENT）
├── MemorySnapshot.java — 会话级聚合快照（按类型分组的记忆条目集合）
├── MemoryCacheManager.java — 缓存管理门面接口
├── DefaultMemoryCacheManager.java — 基于Caffeine的JVM级缓存实现（含快照管理、合并、统计）
├── MemoryColdStorageGateway.java — 冷存储读写门面接口
├── FileBasedMemoryColdStorageGateway.java — 基于JSON文件的冷存储实现
├── MemorySummarizer.java — 记忆摘要器接口
├── IncrementalMemorySummarizer.java — 增量摘要器（基于新增消息生成摘要/事实/约束/工具提示）
├── DefaultMemorySummarizer.java — 全量兜底摘要器
├── MemoryMergeService.java — 记忆合并与淘汰服务（两层去重+相似度截断）
└── AsyncMemoryWriteService.java — 异步Write-Behind服务

**设计要点（参考 Claude Code 三层缓存设计思想）：**
- **分层存储（三层记忆）**：热数据在JVM内存（L1 / 遵循对话轮次的 TurnCache 或 MessageWindow）→ 温数据在L2缓存（MemoryCacheManager，基于Caffeine的KV结构，支持快照管理）→ 冷数据在文件或数据库（L3，落盘于 `data/cache/memory-cache.json` 中，确保极低依赖）
- **Write-Behind模式**：Agent运行结束后异步将L2缓存写入冷存储，不阻塞主流程
- **增量摘要与KV化表达**：`IncrementalMemorySummarizer` 将对话历史提炼为摘要（summary）、事实（facts）、约束（constraints）和工具提示（tool-hints），以结构化JSON格式跨回话存储，注入到后续对话的决策提示词中，避免提示词膨胀。
- **记忆合并与淘汰**：`MemoryMergeService` 实现两层去重（Key去重 + Jaccard相似度去重），按置信度和更新时间排序截断，防止记忆膨胀
- **会话级快照**：`MemorySnapshot` 按类型分组聚合记忆条目，支持版本控制和增量更新
- **Caffeine缓存**：替代原有ConcurrentHashMap实现，提供过期策略、最大容量限制、命中率统计等企业级特性
- **过期机制**：每个MemoryEntry带过期时间，默认24小时TTL，不污染正常交互。
- **版本控制**：通过 version 字段防止旧版本覆盖新版本
- **置信度评分**：每个记忆条目带confidence字段（0.0-1.0），用于排序和淘汰决策

### 消息推送 — Messaging

[agent/messaging/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/agent/messaging)
├── MessagePublisher.java — 消息发布器接口
└── SseMessagePublisher.java — 基于SSE的实现

- `sendStatus()` — 推送Agent运行状态（规划中/思考中/执行中/完成）
- `sendStreamingMessage()` — 流式推送AI生成内容（打字机效果，每12字符分块，间隔30ms）
- `sendPendingMessages()` — 批量补发持久化后的消息
- `sendCompleteMessage()` — 发送完整消息标记

### 工具执行器 — Tool

[agent/tool/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/agent/tool)
├── ToolExecutor.java — 工具执行器接口
└── DefaultToolExecutor.java — 默认实现

**核心设计：手动控制工具调用流程**
- 关闭 Spring AI 的 `internalToolExecutionEnabled`（自动执行），改为手动编排
- `think()` 阶段：调用LLM判断是否需要工具，返回 `ChatResponse`
- `execute()` 阶段：根据 think 的输出手动构建 `ToolCallingManager` 执行工具
- 两层分离使 AgentCore 可以在中间插入持久化、状态管理、消息推送等逻辑

### 工具集

[agent/tools/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/agent/tools)
├── Tool.java — 工具接口（getName/getDescription/getType）
├── ToolType.java — 工具类型枚举（FIXED/OPTIONAL）
└── ToolRegistry.java — 工具注册表，按名称或类型检索

#### 固定工具（FIXED — 始终可用）

| 工具 | 类名 | 说明 |
|------|------|------|
| KnowledgeTool | KnowledgeTools | 从知识库执行语义检索（RAG），参数：kbsId、query |
| terminate | TerminalTools | 终止Agent Loop，当所有任务完成时调用 |
| getDate | DateTool (test) | 获取当前日期 |
| getCity | CityTool (test) | 获取当前城市（演示用，固定返回"深圳"） |
| weather | WeatherTool (test) | 查询指定城市和日期的天气（模拟） |

#### 可选工具（OPTIONAL — 按Agent配置加载）

| 工具 | 类名 | 说明 |
|------|------|------|
| dataBaseTool | DataBaseTools | 安全数据库查询（仅支持SELECT），结果以表格形式输出 |
| fileSystemTool | FileSystemTools | 文件系统操作（读取/写入/列出目录），**默认禁用** |
| emailTool | EmailTools | 通过QQ邮箱异步发送邮件 |
| DocumentSearchTool | DocumentSearchTool | 在知识库中检索内容片段，支持limit参数 |

---

## 六、事件驱动与流程

### 事件系统

[event/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/event)
├── ChatEvent.java — 聊天事件（agentId + sessionId + userInput）
└── listener/
    └── ChatEventListener.java — 异步监听器

### 完整对话流程

```
1. 用户在前端发送消息
     ↓
2. ChatMessageController 接收请求，调用 ChatMessageFacadeService
     ↓
3. Service 将消息存入 chat_message 表，然后发布 ChatEvent 事件
     ↓
4. ChatEventListener（@Async 异步）监听到事件
     ↓
5. 调用 RAGAgentFactory.create(agentId, sessionId)
   ├── 从数据库加载 Agent 配置
   ├── 从 ChatClientRegistry 获取对应模型的 ChatClient
   ├── 解析可用工具（固定 + 可选）
   ├── 解析知识库列表
   ├── 从冷存储加载历史消息 → 初始化 ChatMemory
   ├── 从冷存储加载L2缓存快照 → 初始化 MemoryCacheManager
   └── 组装 AgentContext + 创建 RAGAgent（注入增量摘要器、合并服务等）
     ↓
6. agent.run() → AgentCore.run() 启动 Think-Execute 循环
     ↓
7. think() 阶段：
   ├── 读取 L2 轻量缓存记忆（从MemoryCacheManager获取快照）
   ├── 组装决策提示词（知识库信息 + 缓存记忆：摘要/事实/约束）
   ├── 将历史消息 + 决策提示词发送给 LLM
   └── LLM 返回：直接回答 OR 工具调用请求
     ↓
8. 若为直接回答 → SSE 逐字流式推送到前端
     ↓
9. 若需要工具调用 → execute() 阶段：
   ├── 执行 LLM 请求的工具
   ├── 将工具结果追加到对话历史
   ├── 更新 SessionCacheContext（标记dirty、推进checkpoint）
   └── 回第7步继续 think()
     ↓
10. 循环 think() ↔ execute()，直到：
    ├── LLM 直接给出回答（无工具调用）
    ├── 调用 terminate 工具明确结束
    └── 达到最大步数限制（默认20步）
     ↓
11. refreshSessionCache() → L2缓存刷新：
    ├── IncrementalMemorySummarizer 对新增消息做增量摘要
    ├── MemoryMergeService 执行合并（Key去重 → 相似度去重 → 排序截断）
    ├── 更新 MemoryCacheManager 中的快照
    └── AsyncMemoryWriteService 异步写回冷存储
     ↓
12. SSE 推送 AI_DONE 状态，通知前端对话完成
```

---

## 七、辅助功能

### 1. 配置类

[config/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/config)
├── MultiChatClientConfig.java — 多模型ChatClient Bean定义（deepseek-chat、glm-4.6）
├── ChatClientRegistry.java — ChatClient注册表，按模型名称检索
├── AsyncConfig.java — 异步配置（支持@Async事件监听）
├── CorsConfig.java — 跨域配置
└── BobbyConfigLoader.java — .bobby配置文件加载器（缓存参数管理）

### 2. 异常处理

[exception/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/exception)
├── BizException.java — 自定义业务异常
└── GlobalExceptionHandler.java — 全局异常处理（@RestControllerAdvice）

### 3. 消息模型

[message/](file:///d:/Project/IdeaProject/RAG-Agent/src/main/java/com/person/ragagent/message)
└── SseMessage.java — SSE消息结构体

```
SseMessage {
    type: AI_GENERATED_CONTENT | AI_PLANNING | AI_THINKING | AI_EXECUTING | AI_DONE
    payload: {
        message: ChatMessageVO,    // 聊天消息（流式场景为当前累计内容）
        statusText: String,        // 状态文本（如"规划处理步骤"）
        done: boolean              // 是否完成
    }
    metadata: {
        chatMessageId: String      // 关联的消息ID
    }
}
```

### 4. 应用配置

[application.yaml](file:///d:/Project/IdeaProject/RAG-Agent/src/main/resources/application.yaml)
- 数据库：PostgreSQL `jdbc:postgresql://localhost:5432/jchatmind`
- AI模型：DeepSeek（deepseek-chat）+ Zhipu AI（GLM-4.6）
- 邮件：QQ邮箱SMTP（异步发送，端口587）
- 文档存储：`./data/documents`
- MyBatis：自动扫描实体类、Mapper XML、TypeHandler
- 服务端口：`8080`

---

## 八、前端集成

前端项目位于 [ui/](file:///d:/Project/IdeaProject/RAG-Agent/ui)，使用 React 19 + TypeScript + Vite 构建。

### 技术栈
- **UI框架**：Ant Design 6.0.0 + Ant Design X 2.0.0（AI对话组件）
- **样式**：Tailwind CSS 4.1.17
- **路由**：React Router 7.9.6
- **图标**：@ant-design/icons 6.1.0
- **Emoji**：emoji-picker-element

### 目录结构
```
ui/src/
├── api/
│   ├── api.ts — 业务API封装
│   └── http.ts — HTTP请求基础配置
├── components/
│   ├── modals/
│   │   ├── AddAgentModal.tsx — 添加Agent模态框
│   │   └── AddKnowledgeBaseModal.tsx — 添加知识库模态框
│   ├── tabs/
│   │   ├── AgentTabContent.tsx — Agent管理标签页
│   │   ├── ChatTabContent.tsx — 聊天标签页
│   │   └── KnowledgeBaseTabContent.tsx — 知识库管理标签页
│   ├── views/
│   │   ├── agentChatView/ — Agent聊天视图组件
│   │   │   ├── AgentChatHistory.tsx — 聊天历史
│   │   │   ├── AgentChatInput.tsx — 聊天输入框
│   │   │   └── EmptyAgentChatView.tsx — 空状态
│   │   ├── AgentChatView.tsx — Agent聊天主视图
│   │   └── KnowledgeBaseView.tsx — 知识库视图
│   ├── JChatMindLayout.tsx — 主布局组件
│   └── SideMenu.tsx — 侧边菜单
├── contexts/
│   └── ChatSessionsContext.tsx — 聊天会话状态管理
├── hooks/
│   ├── useAgents.ts — Agent数据Hook
│   ├── useChatSessions.ts — 聊天会话Hook
│   ├── useDocuments.ts — 文档数据Hook
│   └── useKnowledgeBases.ts — 知识库数据Hook
├── layout/
│   ├── Content.tsx — 内容区布局
│   ├── Layout.tsx — 整体布局
│   └── Sidebar.tsx — 侧边栏
├── types/
│   └── index.ts — TypeScript类型定义
├── utils/
│   └── index.ts — 工具函数
├── App.tsx — 应用入口组件
├── index.css — 全局样式
└── main.tsx — 应用启动入口
```

### SSE 前端接收流程

1. 前端通过 `EventSource` 连接到 `/sse/connect/{chatSessionId}`
2. 收到 `init` 事件确认连接成功
3. 监听 `message` 事件，解析JSON中的 `type` 字段：
   - `AI_PLANNING` / `AI_THINKING` / `AI_EXECUTING` — 更新状态指示器
   - `AI_GENERATED_CONTENT` — 实时追加/更新AI回复内容（打字机效果）
   - `AI_DONE` — 标记本轮对话完成

---

## 九、项目难点与解决方案

### 1. Think-Execute 循环实现
>需要理解Agent的工作机制，设计合理的循环流程

- 将 think（决策）和 execute（执行）分离为两个独立阶段
- 关闭 Spring AI 自动工具执行，通过 `ToolExecutor` 手动编排
- 最大步数限制（默认20步）防止无限循环
- 支持 `terminate` 工具让Agent主动结束任务

### 2. 向量类型处理
>PostgreSQL的vector类型需要自定义TypeHandler来处理向量序列化和反序列化

- `PgVectorTypeHandler` 实现 `float[]` ↔ PostgreSQL `vector` 类型的双向转换
- `toPgVector()` 将浮点数组转为 `[0.1,0.2,...]` 格式字符串
- `parse()` 将数据库返回的 vector 字符串解析回 `float[]` 数组
- 通过 MyBatis `type-handlers-package` 自动扫描注册

### 3. 手动控制工具调用流程
>禁用Spring AI自动执行，使用ToolCallingManager手动管理

- 在 `DefaultToolExecutor` 中设置 `internalToolExecutionEnabled(false)`
- think 阶段只做决策，execute 阶段才实际调用工具
- 两层间可插入持久化、消息推送、日志记录等横切逻辑

### 4. 多模型支持
>使用注册表模式统一管理多个ChatClient

- `MultiChatClientConfig` 定义不同模型的 `@Bean`（deepseek-chat、glm-4.6）
- `ChatClientRegistry` 自动注入所有 `ChatClient` Bean 到 Map 中
- 使用时根据Agent配置的模型名称动态选择
- 添加新模型只需新增Bean + 配置yaml，无需修改其他代码

### 5. 消息窗口完整性保障
>Tool calls 和 Tool responses 必须成对保留

- `ensureToolCallResponsePairs()` 从后向前遍历消息列表
- 若 AssistantMessage 的 tool_calls 没有对应的 ToolResponseMessage，则移除该消息
- 避免因滑动窗口截断导致 LLM API 报错

### 6. 上下文Token预算控制
>相似性检索结果可能超出模型上下文限制

- 知识库元数据中配置 `maxContextTokens`
- `estimateTokens()` 按字符级估算 token 数（ASCII 4字符≈1 token）
- 从后向前丢弃低相关度块，直到满足预算

### 7. Write-Behind 异步写回
>L2缓存更新不阻塞Agent主流程

- Agent运行结束后触发 `refreshSessionCache()`
- 增量摘要器对对话历史做摘要、事实、约束提取
- `MemoryMergeService` 执行两层去重（Key去重 + Jaccard相似度去重）
- `AsyncMemoryWriteService` 异步将摘要结果写回文件冷存储

### 8. 增量记忆管理
>Phase 1缓存优化核心特性

- `MemorySnapshot` 按类型分组聚合记忆条目，支持版本控制
- `IncrementalMemorySummarizer` 基于新增消息生成增量摘要（而非全量重建）
- `MemoryMergeService` 实现记忆合并：Key去重 → 相似度去重 → 置信度排序 → 截断淘汰
- `SessionCacheContext` 跟踪缓存进度（committedIndex/checkpointedIndex/dirty标记）
- Jaccard相似度算法：将文本分词后计算交集/并集比值，阈值0.75判定为相似
- 每种记忆类型限制最大条目数（默认3条），防止缓存膨胀

### 9. Caffeine缓存升级
>替代原有ConcurrentHashMap实现

- 提供自动过期策略（expireAfterAccess，默认30分钟）
- 最大容量限制（maximumSize，默认256个会话）
- 内置命中率统计（CacheStats），便于监控和调优
- 通过 `.bobby` YAML配置文件管理参数，支持默认值和用户自定义

### 10. RAG 检索管线优化
>从入库端和检索端同时收紧 RAG 管线，提升检索结果的相关性和可读性

**入库链路优化（`MarkdownChunkComposer`）：**
- **旧版问题**：仅对章节标题做 embedding，正文不参与语义向量，搜索"怎么配置超时"命中不了正文写满超时配置但标题是"高级设置"的章节
- **新版方案**：`MarkdownChunkComposer.compose(title, content, maxTokensPerChunk)` 将章节拆分为 1..N 个语义 chunk
  - 每个 chunk 以 `# 标题\n\n` 开头，确保嵌入向量包含层级语义
  - 按段落边界切分，fenced code block 和表格视为原子块
  - 单段超长时按句子边界或固定长度切分
- **Token 估算**：`TokenEstimator` 保守估算，ASCII 4字符≈1 token，CJK 1字符≈2 token
- **Metadata 记录**：`{"sectionTitle":"...","chunkIndex":0,"chunkCount":3}`

**检索链路优化（RagSearchFilter + RagContextAssembler）：**
- **SQL 层**：使用 pgvector `<->` 操作符（L2 距离），`candidateLimit`（默认8）控制候选召回数
- **距离过滤**：丢弃 `relevanceScore > maxDistance` 的 chunk
- **Jaccard 去重**：
  - 将 chunk 拆分为标题部分（# 开头的行）和正文部分
  - 分别计算标题和正文的 2-gram Jaccard 相似度
  - 仅当标题 AND 正文同时 > jaccardThreshold（默认0.8）才去重
  - 标题相同但正文不同 → 同章节分片，保留
  - 标题不同但正文相似 → 保守保留，避免误删
- **上下文组装**：
  - 稳定排序：relevanceScore 升序 → 正文长度降序 → updatedAt 降序
  - Greedy Packing：按排序顺序累计 token，超过总预算停止
  - 相邻合并：同一 docId 且同一 sectionTitle 的相邻 chunk，在预算内合并
  - 编号输出：【1】...内容...、【2】...内容...

**配置化参数（`application.yaml`）：**
```yaml
rag:
  search:
    candidateLimit: 12          # SQL 候选召回上限
    minSimilarity: 0.75         # 最小余弦相似度
    maxChunksPerKb: 4           # 每个知识库最多保留的 chunk 数
    defaultToolOutputLimit: 4   # KnowledgeTools 默认输出块数
    jaccardThreshold: 0.8       # Jaccard 去重阈值
  chunk:
    maxTokensPerChunk: 800      # 单个 chunk token 上限
```

---

## 十、常见问题

### 什么是Agent
>Agent是一种能够自主感知、做出决策并执行的智能系统，通过观察、思考、决策和行动，通过主动规划和工具调用等来完成复杂任务。

### Think-Execute 循环
- **Think 阶段**：分析当前任务和上下文 → 规划执行步骤 → 选择合适的工具 → 决定是否需要调用工具
- **Execute 阶段**：调用选定的工具 → 执行思考出的操作 → 获取执行结果
- 循环条件：任务是否完成 → 未完成则返回Think阶段 → 完成则返回最终结果
- 项目给循环设定了最大值（默认20步），避免无限循环。Agent也可调用 terminate 工具明确结束任务

### RAG（检索增强生成）
>一种结合信息检索和文本生成的技术

工作流程：
1. 文档处理：将文档分块，生成 Embedding 向量
2. 将向量存储到向量数据库（PostgreSQL pgvector）
3. 将用户查询转换为向量，进行相似度搜索（余弦相似度）
4. 将检索到的相关文档作为上下文注入到 LLM
5. LLM 基于上下文生成回答

与微调的区别：
1. 微调在预训练模型基础上，使用特定数据对AI进行训练
2. 在**模型权重层面**修改大模型本身，让模型**永久内化**领域知识
3. 微调对算力和训练成本要求高

核心痛点及优化方案：
1. **召回精度低** → 向量化模型微调、多路召回（向量+关键词混合检索）、标题+正文联合 embedding
2. **上下文丢失** → 合理切片分块，增加块间关联，chunk 带标题前缀
3. **幻觉严重** → 严格要求LLM只基于检索原文回答
4. **时效性差** → 定时更新向量库、对接实时知识库
5. **上下文窗口有限** → 摘要压缩、过滤无用文档、Token预算控制
6. **重复内容浪费** → RagSearchFilter 距离过滤 + 两段式 Jaccard 去重
7. **检索结果无结构** → RagContextAssembler 编号化输出（【1】...【2】...）

**RAG 检索详细流程：**

入库链路：
```
Markdown 文件
  → MarkdownParserService.parseMarkdown() → List<MarkdownSection> (title + content)
  → MarkdownChunkComposer.compose(title, content, maxTokensPerChunk) → List<String> (标题前缀 + 正文片段)
  → ragService.embed(chunkText) × N → Ollama bge-m3 → float[1024]
  → ChunkMapper.insert(chunk) → chunk_bge_m3 表
```

检索链路：
```
query 文本
  → ragService.embed(query) → float[1024] query vector
  → ChunkMapper.similaritySearch(kbId, vectorLiteral, candidateLimit=8)
  → 按 relevance_score 升序返回候选 chunk 列表
  → Token 预算控制：从后向前丢弃低相关度块，直到满足 maxContextTokens
  → List<String> 返回给调用方
```

检索工具：
- `KnowledgeTool`（固定工具）：始终可用，参数 `kbsId` + `query`，返回与查询最相关的内容片段
- `DocumentSearchTool`（可选工具）：按 Agent 配置加载，额外支持 `limit` 参数控制返回片段数

### 向量数据库的作用
1. 高效存储向量数据
2. 利用相似度算法快速找到相似向量

项目实现：
- 使用 `vector(1024)` 类型存储 BGE-M3 的 Embedding（1024维）
- 使用余弦相似度算法（`<=>` 操作符）计算向量相似度
- 支持 Top K 检索，返回最相似的文档片段
- 本地部署 BGE-M3 模型（通过 Ollama API，`http://localhost:11434`）

> [!NOTE] Embedding
> 嵌入向量，将文本、图像等数据转换为数值向量的技术。本项目使用 BGE-M3 模型生成1024维向量。

### Function Calling / Tool Calling
>函数调用和工具调用，是让LLM能够调用外部函数/工具的能力。不借助Agent只能单次调用。

工具调用流程：
- 定义工具的功能、参数、返回值（通过 `@Tool` 注解）
- 注册工具信息传递给 LLM
- LLM 分析任务，决策是否调用工具
- 执行工具调用，获取结果
- 将工具结果返回给 LLM 继续推理

项目实现：
1. 工具实现 `Tool` 接口（getName/getDescription/getType）
2. 工具方法使用 `@Tool` 注解声明名称、描述、参数
3. `ToolRegistry` 统一管理所有工具，按类型（FIXED/OPTIONAL）分类
4. `RAGAgentFactory` 将业务 Tool 转换为 Spring AI 的 `ToolCallback`
5. `DefaultToolExecutor` 在执行阶段基于 `ToolCallingManager` 调用工具

### 提示词工程
- 作用：影响模型输出质量、控制模型行为、减少无效调用
- 技巧：明确任务和目标、提供示例、使用结构化格式
- 本项目在 think 阶段使用专门的决策提示词，包含知识库信息和缓存记忆上下文

### SSE 实时推送
1. **连接建立**：客户端访问 `/sse/connect/{chatSessionId}` → 服务器创建 `SseEmitter`（30分钟超时）→ 发送 `init` 事件确认连接
2. **连接管理**：`ConcurrentHashMap` 存储所有客户端连接，保证线程安全；`onCompletion/onTimeout/onError` 自动清理
3. **消息结构**：`SseMessage` 包含 Type/Payload/Metadata 三部分
4. **消息发送**：`ObjectMapper` 序列化JSON → `SseEmitter.event("message")` 推送
5. **流式内容**：每12字符分块，间隔30ms推送，实现打字机效果

> 为什么不用 WebSocket：SSE更适合服务端单向推送的场景，WebSocket适合双向通信。

### 消息持久化策略
1. 用户消息立即持久化到 `chat_message` 表
2. AI回复消息（含工具调用信息）在 think 阶段生成后持久化
3. 工具调用结果在 execute 阶段执行后持久化
4. 创建Agent时，从 `chat_message` 表加载历史消息恢复到 `ChatMemory` 中作为会话上下文
5. `metadata` JSON字段存储工具调用信息（toolCalls/toolResponse），支持完整恢复对话状态

### 轻量缓存（L2 Cache）设计
1. L1（热）— `ChatMemory` 滑动窗口，保存最近N条原始消息
2. L2（温）— `MemoryCacheManager` JVM缓存（基于Caffeine），存储规则摘要（summary/facts/constraints/tool-hints），支持快照管理和增量更新
3. 冷存储 — `MemoryColdStorageGateway` JSON文件持久化
4. Write-Behind — Agent结束后异步写回，不阻塞主流程
5. 过期机制 — 默认24小时TTL（MemoryEntry级别），Caffeine缓存30分钟无访问过期
6. 版本控制 — 通过 version 字段防止旧版本覆盖新版本
7. 置信度评分 — 每个记忆条目带confidence字段，用于排序和淘汰
8. 记忆合并 — `MemoryMergeService` 实现Key去重 + Jaccard相似度去重 + 截断淘汰
9. 配置化 — `.bobby` 文件管理缓存参数（expireAfterAccessMinutes/maximumSize/maxPerType）

---

## 十一、量化指标测试体系

### 测试框架

基于 JUnit 5 + `@SpringBootTest` 集成测试，使用自研 `MetricsCollector`（计时与百分位统计）和 `MetricsReport`（Markdown 报告生成器），覆盖缓存、RAG 管线、Agent 对话、记忆系统 4 个维度共 28 项指标。

### 测试数据来源

| 数据来源 | 说明 |
|----------|------|
| PostgreSQL `chunk_bge_m3` 表 | 103 条真实文档分块数据（2 个知识库） |
| PostgreSQL `memory_cache` 表 | 真实冷存储会话记忆数据 |
| Ollama bge-m3 API | 本地 1024 维 Embedding 模型 |
| DeepSeek API | 真实 LLM 调用（think/execute/TTFT） |
| Caffeine 缓存 | L1 缓存统计（命中率/驱逐数） |

### 指标分类与实测数据

#### 缓存指标（10 项）

| 指标 | 实测值 | 基线 | 数据来源 |
|------|--------|------|----------|
| 真实缓存命中率 | 100% | 70% | `memory_cache` 真实会话数据 |
| 人工构造命中率 | 55.1% | 50% | 手工构造 MemoryEntry |
| 缓存驱逐数 | 745 条 | 1 条 | Caffeine CacheStats |
| 冷存储读延迟 (avg) | 1 ms | 200 ms | `DbMemoryColdStorageGateway` |
| 冷存储写延迟 (avg) | 5 ms | 500 ms | `DbMemoryColdStorageGateway` |

**基线依据**：冷存储延迟基于本地 SSD + PostgreSQL 的合理预期（<200ms 读、<500ms 写）；命中率基于缓存预热后的正常表现（>50%~70%）；驱逐数仅验证驱逐机制生效（>1 即可）。

#### RAG 管线指标（10 项）

| 指标 | 实测值 | 基线 | 数据来源 |
|------|--------|------|----------|
| precision@4 | 79.3% | 50% | 103 条真实 chunk，29 次查询 |
| Chunk Token P50 | 261 | 200-600 | `TokenEstimator` 对全部 chunk 估算 |
| Chunk Token P95 | 783 | 800 | 同上 |
| Chunk Token P99 | 791 | 1200 | 同上 |
| 超大 chunk 占比 | 0.0% | 5% | maxTokensPerChunk=800 阈值 |
| Jaccard 去重率 | 0.0% | 记录 | 真实文档间内容不重复 |
| Embedding 延迟 P95 (500字) | 137 ms | 1500 ms | Ollama bge-m3 API |
| Embedding 并发 P95 (4线程) | 547 ms | 记录 | 4 线程 × 3 次 |

**基线依据**：precision@4 >= 50% 是信息检索领域的经验基准；Token P50 200-600 基于 `maxTokensPerChunk=800` 的配置约束反推的理想分布；Embedding 延迟 <1500ms 基于 bge-m3 模型在本地 GPU 上的典型推理耗时。

#### Agent 对话指标（6 项）

| 指标 | 实测值 | 基线 | 数据来源 |
|------|--------|------|----------|
| think() 决策耗时 | 700 ms | 5000 ms | DeepSeek API |
| TTFT（首 token 延迟） | 950 ms | 2000 ms | DeepSeek API 流式响应 |
| execute() 工具执行耗时 | <1 ms | 10000 ms | 本地工具调用 |
| 工具调用总次数 | 3 次 | ≤10 次 | 3 轮对话 |
| 工具调用失败次数 | 0 次 | ≤1 次 | 同上 |
| 工具调用成功率 | 100% | 95% | 同上 |

**基线依据**：think() <5000ms 和 TTFT <2000ms 基于 DeepSeek API 的 P95 响应延迟 + 网络抖动裕量；execute() <10000ms 给工具执行留足余量（实际工具执行 <1ms）；工具调用成功率 95% 是生产系统的基本可靠性要求。

#### 记忆系统指标（2 项）

| 指标 | 实测值 | 基线 | 数据来源 |
|------|--------|------|----------|
| SUMMARY 一致性 | 51.3% | 50% | 2 轮模拟对话，Jaccard 相似度 |
| FACT 一致性 | 64.4% | 40% | 同上 |

**基线依据**：增量摘要只看到最后一轮（4 条消息），全量摘要覆盖全部（8 条消息），二者自然存在差异。SUMMARY 50%/FACT 40% 反映"方向一致即可"的宽松判定——只要求增量摘要与全量摘要存在语义重叠，不要求精确匹配。

### 评测体系设计原则

1. **真实数据优先**：除 Agent 对话和 Memory 测试需外部 API 外，其余测试优先使用 `chunk_bge_m3` 和 `memory_cache` 中的真实数据
2. **量化指标可复现**：每个指标通过 `MetricsReport.register()` 登记，测试结果写入 `target/metrics-report.md`，支持版本间对比
3. **基线分级**：
   - **硬基线**（必须满足）：如工具调用成功率 ≥95%、precision@4 ≥50%
   - **软基线**（仅记录）：如 Jaccard 去重率、并发 Embedding P95
4. **外部依赖容错**：Ollama 不可达时测试 SKIP 而非 FAIL；DeepSeek API 不稳定性通过流式→非流式自动降级缓解

### 运行测试

```bash
# 运行全部量化指标测试
mvn test -Dtest="CacheMetricsTest,EmbeddingMetricsTest,RagAccuracyTest,AgentTimingTest,MemoryQualityTest"

# 运行单个测试类
mvn test -Dtest=CacheMetricsTest

# 查看报告
cat target/metrics-report.md
```
