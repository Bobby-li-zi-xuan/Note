# API / Security / ModelRegistry / Skill / Observability

---

## 一、api 模块

### 模块定位

对外接口层（`com.yourorg.biboagent.api`）：REST 用于会话管理与元数据操作，SSE 用于对话流式交互。

### REST 接口

会话增删改查、按用户分页列出、查看历史消息、删除会话（同时清理 PostgreSQL 消息与 Redis 快照，不影响用户级记忆——记忆与具体会话解耦）。会话列表自动生成标题：取会话第一条用户消息前 15 字符。

### SSE 流式接口

**双流异步机制**：先 `POST /api/v1/chat:stream` 提交 → 后端生成全局唯一 `requestId`（`req_` 前缀），在虚拟线程中异步执行 AgentService，立即返回 200 + requestId → 前端再 `GET /api/v1/chat/stream/events?requestId=...` 订阅。SseEmitter 与 Canceller 存入 Map，断连/超时时清理连接池并触发 `canceller.cancel()`（穿透到 Turn Loop 与并行子任务）。心跳每 30s 发空注释（`SseEmitter.event().comment("")`）防止代理因空闲超时断连。

**事件格式**（`text/event-stream`）：

| 事件 | 用途 | Data |
|------|------|------|
| TOKEN_DELTA | 推送 LLM 逐 token 输出 | 文本碎片 |
| STAGE | 阶段进度（LLM_CALL / TOOL_CALL / TOOL_RESULT / STEP_RETRY / FINAL） | JSON（stage、step、tools 等） |
| ERROR | 链路异常 | `{code, message}` |

工具调用时推送 `STAGE`（stage=TOOL_CALL，tools=工具名列表），执行完成后再推 TOOL_RESULT。

**断连重连**：所有事件同时写入 Redis（带单调递增 ID）和 SseEmitter；重连带 `Last-Event-Id` 时从该 ID 之后重放；重放流已含 FINAL 则直接完成连接（对话已结束）。无 Last-Event-Id 则从头重放。

### 全局异常处理

`AgentException` 返回对应 HTTP 状态码与错误码，未知异常统一 500；格式 `{error:{code,message}}`。

---

## 二、security 模块

四道防线：JWT 鉴权、OWASP 输入清洗、敏感数据脱敏、开发模式绕过（`com.yourorg.biboagent.security`）。

Filter 执行顺序（@Order）：TraceFilter(0) → DevModeFilter(1) → SanitizerFilter(2) → JwtAuthFilter（未声明 @Order，按默认最低优先级最后执行）。

- **JwtAuthFilter**：解析 `Authorization` Bearer JWT（HS256），验证签名+过期，从 claims 提取用户 ID/角色设置 SecurityContext；未配置密钥时跳过所有请求（方便开发）；无效/过期返回 401。
- **DevModeFilter**：开发环境通过 `X-Dev-UserId` 请求头指定身份，绕过完整 JWT 验证。
- **SanitizerFilter**：过滤 OWASP Top 10 常见恶意输入（XSS、SQL 注入尝试）。
- **SensitiveDataRedactor**：日志与审计中隐藏密码、Token、密钥等敏感信息。

---

## 三、modelregistry 模块

模型信息全部在 `application.yml` 的 `agent.models` 下配置（modelId、defaultModel、provider、展示名、能力描述、默认参数、状态），由 `@ConfigurationProperties` 绑定到 `YamlModelRegistry`。

选择逻辑：请求指定模型 ID → 必须存在且 enabled（否则返回 null → MODEL_DISABLED）；未指定 → 选 `defaultModel=true` 的 enabled 模型。canary 灰度发布为未来方向。

---

## 四、observability 模块

- **全链路追踪**：TraceFilter（order=0）生成/提取 trace ID（优先取 `X-Request-Id` 头），注入 MDC；记录请求入口/出口日志（方法、路径、状态码、耗时）。
- **指标**：Micrometer → Prometheus（`/actuator/prometheus`）：LLM 调用耗时（按模型/供应商标记）、工具重试/超时次数、幂等缓存命中率。
- **审计日志**：关键业务事件（对话开始/结束、LLM 成功/失败、工具执行结果）持久化 PostgreSQL，details 用 JSONB。

---

## 五、Skill 机制

Skill = "特定角色的 Prompt 片段 + Markdown 指令知识 + 关联工具列表"，文件存于 `bibo-agent/skills/*.md`：YAML Front Matter 定义 name/description/aliases/prompts/tools，Markdown 主体为操作规范（解析为 bodyContent）。`SkillFileLoader` 解析为 `SkillDefinition` → `SkillSpec` 注册到 `InMemorySkillRegistry`。

调度流程：

1. 用户在消息里用 Slash 命令（如 `/code`）触发 → `SlashCommandParser` 截获并识别 Skill ID，同时从发给 LLM 的文本中抹去指令标记。
2. `PromptAssembler.mergeSkillIds` 合并会话级 + 请求级 skill（去重）。
3. `PromptAssembler` 把对应 Prompt 片段 + Markdown 主体注入 System Prompt，框定"角色边界与动作规范"。
4. 工具层面是"软约束"：所有 @Tool 定义均下发，通过 Prompt 引导 LLM 使用对应领域工具；如需硬性白名单，可在 LlmInvoker/ToolPolicyService 层按 `SkillSpec.tools()` 做权限校验（未来方向）。
5. 热重载：`POST /api/v1/skills:reload` 触发 `SkillFileLoader` 重新扫描文件系统。

**Tool vs Skill**：Tool 是能力的"手臂"（细粒度、单一 Java 方法操作）；Skill 是能力的"业务大脑"（面向用户场景组合 Prompt + Tool，改 Markdown 即可扩展，无需写 Java）。用户前端交互的是 Skill，后台支撑的是 Tool。
