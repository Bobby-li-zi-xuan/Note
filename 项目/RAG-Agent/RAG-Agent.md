# RAG-Agent 项目说明

本文档根据当前仓库内容整理，用于快速了解项目结构、核心流程、技术栈和当前已经实现的关键能力。

---

## 1. 项目定位

RAG-Agent 是一个面向知识检索和工具增强对话的 Java Agent 项目。它把以下能力组合在一起：

- 多轮聊天会话
- RAG 知识库检索
- 工具调用与失败恢复
- SSE 实时推送
- L1/L2/L3 三层记忆
- 前后端分离的管理界面

项目的核心不是单轮问答，而是“有状态的 Agent 编排”。每一轮对话都要完成：模型决策、工具执行、会话持久化、缓存更新和前端推送。

---

## 2. 目录结构

```
RAG-Agent/
├── data/                         本地缓存与文档数据
├── docs/superpowers/             设计文档和实施计划
├── org/                          Spring AI 源码镜像与注释版参考
├── src/main/java/com/person/ragagent/
│   ├── agent/                    Agent 核心：编排、缓存、记忆、工具
│   ├── config/                   Spring 配置
│   ├── controller/               接口层
│   ├── event/                    事件与监听
│   ├── mapper/                  MyBatis Mapper
│   ├── model/                   DTO / Entity / VO / Request / Response
│   ├── service/                 业务服务
│   ├── typehandler/             pgvector 类型处理器
│   └── ...
├── src/main/resources/
│   ├── application.yaml          运行配置
│   └── Mapper/                   MyBatis XML
├── src/test/                     单元测试
└── ui/                           React + Vite 前端
```

`org/` 目录不是业务代码，而是 Spring AI 依赖源码镜像。它的作用是方便阅读和加中文注释，帮助理解底层消息、提示词和工具接口的真实行为。

---

## 3. 技术栈

### 后端

| 技术 | 说明 |
|------|------|
| Java 23 | 运行时语言 |
| Spring Boot 3.2.9 | 服务启动、依赖注入、Web 层 |
| Spring AI 1.1.0 | LLM 调用、ToolCallback、Prompt、ChatResponse |
| MyBatis | 数据访问层 |
| PostgreSQL + pgvector | 文档向量检索与持久化 |
| Lombok | 减少样板代码 |
| Caffeine | JVM 内存缓存 |
| SSE | 实时消息推送 |
| Flexmark | Markdown 解析 |
| Spring Mail | 邮件发送 |

### 前端

| 技术 | 说明 |
|------|------|
| React 19 | 前端框架 |
| TypeScript | 类型系统 |
| Vite | 构建工具 |
| Ant Design / Ant Design X | UI 与对话组件 |
| Tailwind CSS | 样式 |
| React Router | 路由 |

---

## 4. 核心运行链路

### 4.1 对话链路

```
用户发消息
  → Controller / Event
  → ChatEventListener @Async
  → RAGAgentFactory.create(agentId, sessionId)
  → AgentCore.run()
      → think(): 让模型决定是否需要工具
      → execute(): 执行模型请求的工具
      → 循环直到结束
  → SSE 推送状态和结果
  → finally 刷新缓存和冷存储
```

### 4.2 RAG 检索链路

```
Markdown 文件
  → MarkdownParserService
  → MarkdownChunkComposer
  → embedding
  → pgvector 存储
  → query embedding
  → similarity search
  → token budget 裁剪
  → 返回上下文片段
```

### 4.3 工具调用链路

```
think() 生成 toolCalls
  → ToolCallingManager 解析工具
  → executeToolCalls()
  → 工具成功：返回 ToolResponseMessage
  → 工具失败：进入恢复闭环
```

---

## 5. 工具失败恢复机制

这是当前项目刚补齐的重点能力。原来工具失败更接近“异常传播”，现在变成了“可恢复的编排流程”。

### 5.1 恢复流程

```
工具执行抛错
  → DefaultToolExecutor 先做快速重试
  → 仍失败则构造 ToolFailureContext
  → ToolRecoveryDecisionClient 让模型给出策略
  → 解析为 ToolRecoveryDecision
  → 根据动作决定继续重试、降级或停止
  → 最终通过 ToolResponseMessage 返回给上层
```

### 5.2 恢复相关类

| 类 | 职责 |
|----|------|
| `ToolFailureContext` | 失败上下文快照 |
| `ToolRecoveryDecision` | 模型返回的恢复决策 |
| `ToolFailureRecoveryService` | 纯逻辑服务：分类、解析、构造失败消息 |
| `ToolRecoveryDecisionClient` | 恢复决策入口抽象 |
| `ChatClientToolRecoveryDecisionClient` | 基于 ChatClient 的默认实现 |
| `DefaultToolExecutor` | 编排快速重试、模型决策和兜底 |

### 5.3 约束

- 快速重试有上限
- 模型咨询有上限
- 退避时间有最小值和最大值
- `RETRY_WITH_POLICY` 可以重置重试窗口
- 失败消息会保留完整 `baseHistory`
- 失败响应里会包含 `toolArguments`，便于模型和日志定位问题

### 5.4 支持的策略动作

- `RETRY`
- `RETRY_WITH_POLICY`
- `DEGRADE`
- `STOP`

### 5.5 面试时可以怎么说

可以概括为：**先做技术层快速恢复，再让模型做策略判断，最后用结构化失败消息安全兜底。**

---

## 6. Agent 关键模块

### 6.1 `AgentCore`

负责整个对话循环：

- 读取消息历史
- 调用 `think()`
- 调用 `execute()`
- 更新记忆和缓存
- 驱动状态流转和 SSE 推送
- 在 finally 中保证刷盘

### 6.2 `DefaultToolExecutor`

负责将模型决策和真实工具执行分开。它当前支持：

- 流式 think 和非流式降级
- 手动工具调用执行
- 工具失败恢复闭环
- 工具调用统计
- TTFT 统计

### 6.3 `MemoryCacheManager`

负责 L2 结构化缓存，和 L1 消息窗口、L3 冷存储协同工作。

### 6.4 `RAG` 相关服务

- Markdown 解析
- Chunk 切分
- Embedding 生成
- pgvector 检索
- token 预算裁剪

---

## 7. 数据模型与表

项目主要表包括：

- `Agent`
- `ChatSession`
- `ChatMessage`
- `KnowledgeBase`
- `Document`
- `Chunk`
- `memory_cache`

其中：

- `Chunk` 保存向量和原文
- `ChatMessage` 保存对话语义片段，包括工具调用信息
- `memory_cache` 保存结构化记忆快照

---

## 8. 重要配置

`src/main/resources/application.yaml` 中的几个关键配置方向是：

- 数据库连接
- LLM 模型配置
- Ollama embedding 配置
- 缓存大小和过期时间
- RAG token 预算
- SSE 和异步处理相关配置

项目里常见的默认值包括：

- 候选召回数默认 8
- 默认上下文预算 1200 tokens
- 工具恢复重试和退避都有上限

---

## 9. 当前实现特征

### 9.1 优点

- Agent 编排边界清晰
- 记忆层次分明
- 工具调用可观测
- 失败恢复是结构化闭环，不是简单抛异常
- RAG 和工具体系都保留了可扩展点

### 9.2 当前约束

- DeepSeek 流式 + 工具场景下可能需要回退到非流式
- 工具恢复依赖模型决策时，需要控制决策轮数和退避窗口
- `org/` 目录只是源码镜像，不应作为业务实现修改点

---

## 10. 推荐阅读顺序

如果想快速理解项目，建议按下面顺序看：

1. `src/main/java/com/person/ragagent/agent/core/AgentCore.java`
2. `src/main/java/com/person/ragagent/agent/tool/DefaultToolExecutor.java`
3. `src/main/java/com/person/ragagent/agent/tool/failure/ToolFailureRecoveryService.java`
4. `src/main/java/com/person/ragagent/service/imp/RagServiceImpl.java`
5. `src/main/java/com/person/ragagent/service/imp/DocumentFacadeServiceImpl.java`
6. `src/main/java/com/person/ragagent/agent/cache/MemoryCacheManager.java`
7. `src/main/java/com/person/ragagent/agent/memory/ChatMemoryManager.java`

---

## 11. 测试入口

当前和工具失败恢复相关的测试主要是：

- `src/test/java/com/person/ragagent/agent/tool/failure/ToolFailureRecoveryServiceTest.java`
- `src/test/java/com/person/ragagent/agent/tool/DefaultToolExecutorFailureHandlingTest.java`

这两组测试覆盖了：

- 超时是否可重试
- 失败消息是否包含 `toolArguments`
- `RETRY_WITH_POLICY` 是否能重置重试窗口
- 决策失败是否能安全兜底
- `DEGRADE` 和 `STOP` 是否有区别

---

## 12. 一句话总结

这个项目是一个把 **RAG、工具调用、实时推送、记忆缓存和失败恢复** 组合在一起的 Java Agent 系统，最近最重要的增强点是把工具失败从“抛异常”升级成了“可重试、可决策、可兜底”的编排闭环。
