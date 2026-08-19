# llm 模块

## 模块定位

llm 模块是系统与 LLM 之间的适配层（`com.yourorg.biboagent.llm`）：把内部消息/工具定义翻译成 LLM 认识的格式，把流式响应解析为 Java 对象，让 runtime 只面向统一的 `StreamingChatModel` 编程，不关心厂商差异。

### 关于 langchain4j

项目只用了 langchain4j 的最小功能子集：`StreamingChatModel` 接口、`ChatRequest/ChatResponse` 协议对象、`@Tool`/`@P` 注解与 JSON Schema 生成、`StreamingChatResponseHandler` 回调。Agent 编排（Turn Loop）、记忆系统、子任务并行全部自研。取舍原则：**框架做基础设施（HTTP 流式、SSE 解析、Schema 生成），业务逻辑自研**——多步条件路由、记忆精细过滤、并发容错套用框架高级抽象会失去灵活性。

## 模块职责与实现

### 请求构建

把消息列表（SystemMessage/UserMessage/AiMessage/ToolExecutionResultMessage）+ 工具定义转换为 `ChatRequest`。每次调用前 `LlmInvoker` 都会 `new ArrayList<>(messages)` 创建副本，避免 LLM 流式回调在异步线程修改共享列表带来的并发风险。

### 模型适配

- `LlmClientFactory`：按 `application.yml` 的 `agent.models` 配置创建 `StreamingChatModel` 实例。
- `YamlModelRegistry`：管理模型生命周期，支持 enabled / disabled 状态 + `defaultModel` 标记——指定模型 ID 时要求存在且 enabled（否则返回 null，调用方映射为 MODEL_DISABLED）；未指定时选 `defaultModel=true` 的 enabled 模型。canary 灰度发布是未来方向。

### 工具调用解析

双重处理兼容不同厂商：`onCompleteToolCall` 回调中实时收集工具调用，`onCompleteResponse` 中兜底检查——若回调未收集到但最终响应里有，则从最终响应提取。

## 面试题整理

1. **为什么每次构建 ChatRequest 时创建新的 ArrayList 副本？** 并发安全。Turn Loop 多 step 共享消息列表，LLM 流式回调在不同线程异步修改，直接传同一引用有并发风险；每次复制避免隐式共享状态。
2. **如何适配不同 LLM 提供商？** 两层抽象：`StreamingChatModel` 统一 OpenAI 兼容协议；`LlmClientFactory` 按 YAML 配置创建实例，厂商参数差异通过 defaultParams 透传。
3. **工具调用解析如何兼容不同提供商的行为差异？** "实时收集 + 最终兜底"：onCompleteToolCall 实时收集，onCompleteResponse 检查遗漏并补提。
4. **模型注册表如何做模型管理与切换？** enabled / disabled + defaultModel：指定 ID 时校验 enabled；未指定时选默认模型；改 YAML 即可切换/回滚；canary 灰度为未来方向。
