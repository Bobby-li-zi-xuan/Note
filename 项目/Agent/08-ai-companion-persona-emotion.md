# AI 伴侣角色与情绪系统

## 模块定位

persona 模块（`com.yourorg.biboagent.persona`）为 Agent 提供可自定义"人设"与"情绪"：用户可创建有特定性格、说话风格、背景故事和兴趣爱好的 AI 伴侣，Agent 在对话中持续追踪自身情绪状态，使交互具备情感连续性和人格一致性。

---

## 一、角色（Persona）系统

**PersonaDefinition**（15 个字段的 record）：`id`（`p_` + UUID）、`ownerId`、`name`、`avatarUrl`、`relationshipType`（FRIEND / PARTNER / MENTOR / CUSTOM）、`personality`、`speakingStyle`、`backstory`、`interests`（集合）、`boundaries`（行为边界约束）、`systemPrompt`（服务端拼合的完整 Prompt）、`fromTemplate`、`entertainmentPreferences`（娱乐偏好）、`createdAt` / `updatedAt`。

**PersonaTemplate**（预设模板）：`classpath:persona-templates/*.yml` 定义，内置三个：温柔姐姐（CUSTOM）、元气少女（FRIEND）、沉稳大叔（MENTOR）。用户创建角色可选模板快速填充。

**CRUD 与 Prompt 拼合**：`PersonaPort` 接口 → `PostgresPersonaRepository`（`personas` 表）。`PersonaService` 创建/更新时把性格、风格、背景、兴趣、边界按固定模板拼成 Markdown 格式的 `systemPrompt` 注入 LLM——"配置→Prompt 自动生成"，用户无需理解 Prompt Engineering。

**配置**：`PersonaProperties`（`agent.persona.*`）：`templatesLocation`（默认 classpath）、`maxPersonasPerUser`（默认 10）。

---

## 二、情绪（Emotion）系统

**Valence-Arousal 二维模型**（`EmotionState` record）：

- `valence`（愉悦度）-1.0~+1.0；`arousal`（唤醒度）-1.0~+1.0；`confidence` 0~1
- `dominantEmotion` 由 `inferDominantEmotion(v,a)` 映射为 7 种中文标签：开心（v>0.3,a>0.1）、兴奋（v>0.3,a>0.3）、平静（v>0.1,|a|<0.1）、担忧（v<-0.1,a<-0.1）、悲伤（v<-0.3,a<-0.1）、焦虑（v<-0.3,a>0.1）、中性（其他）

**三维加权更新**（`EmotionTracker`）：

```
新状态 = 衰减后状态 × inertiaWeight(0.4) + 用户影响 × influenceWeight(0.4)
```

- **时间衰减**：情绪绝对值按 `decayRate(0.1) × hours` 线性趋 0（10 小时消散）
- **用户影响**：`guessUserEmotion()` 关键词匹配估算用户情绪向量（未来可升级 LLM 分类）
- **AI 惯性**：inertiaWeight 保证 AI 不因单条消息剧变
- 存储于 `ConcurrentHashMap`（情绪是高频更新的短生命周期状态；生产可换 Redis TTL=1h，重启丢失可接受）

**情绪注入对话**：PromptAssembler 追加"你现在的情绪是「开心」。请在回复中自然地体现此情绪，但不要明确提及情绪标签或具体数值"——刻意不让 AI 生硬自报情绪。

**情绪影响记忆检索**：记忆写入携带当时 AI 的 `emotionTag`；pgvector 检索 SQL 中 `CASE WHEN emotion_tag = ? THEN 1.1 ELSE 1.0` 给匹配情绪的记忆 ×1.1 加成（"情绪一致性记忆"效应）。

---

## 三、完整数据流

```
前端选角色 → ChatRequest.personaId → AgentService.chatStream
  ├─ PromptAssembler: personaPort.findById → emotionPort.getCurrentEmotion → 拼合 Skill + L0 规则 + 记忆 + 角色 + 情绪
  ├─ TurnLoopCoordinator.run(ctx.personaId)
  ├─ emotionTracker.updateAfterUserMessage(personaId, userMessage)
  └─ MemoryExtractor(emotionTag) → MemoryRecord 携带 emotionTag → pgvector SQL 检索加权（匹配情绪 ×1.1）
```

---

## 四、API 端点

所有角色端点位于 `/api/v1/personas`：

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/v1/personas?userId=xxx` | GET | 列出用户角色 |
| `/api/v1/personas/{personaId}` | GET | 角色详情 |
| `/api/v1/personas` | POST | 创建角色（201） |
| `/api/v1/personas/{personaId}` | PUT | 更新角色 |
| `/api/v1/personas/{personaId}` | DELETE | 删除角色（204） |
| `/api/v1/personas/templates` | GET | 列出模板 |

---

## 五、面试题整理

### 1. "AI 情绪系统为什么用二维模型而不是直接打标签？"

三个原因：**连续性**（VA 是连续值，情绪过渡平滑，时间衰减有明确数学语义）；**组合性**（二维平面可表达混合情感，如"喜极而泣"高 valence + 高 arousal，"焦虑"低 valence + 高 arousal）；**可计算性**（VA 向量支持加权混合运算，一维标签无法加权）。

### 2. "情绪跟踪如何保证不因用户单条消息就剧变？"

惯性权重（inertiaWeight=0.4）：每个更新周期 AI 情绪只有 40% 受当前消息影响，60% 保持原有状态（含衰减后）。需要持续正向交互才能渐进改变，模拟真实人际关系的情绪惯性。

### 3. "角色系统和 Skill 机制有什么区别？"

Skill 是**任务级**约束（"当前应该怎么做"，可切换叠加、短期）；Persona 是**人格级**约束（"你是谁"，贯穿整个会话、保持底层行为一致性）。协同：Persona 决定语气态度，Skill 决定能力边界和领域知识。

### 4. "情绪如何影响记忆检索？"

每条长期记忆携带提取时的 AI 情绪标签；pgvector 检索 SQL 用 `CASE WHEN emotion_tag = ? THEN 1.1 ELSE 1.0` 给匹配情绪的记忆 ×1.1 加成优先返回——模拟人类"情绪一致性记忆"（开心时更容易想起积极经历）。

### 5. "角色系统的 Prompt 注入如何实现？有什么安全考量？"

创建/更新时按固定模板拼合 Markdown systemPrompt，对话时注入 System Prompt。安全：① 预设模板硬编码行为边界约束（6-9 条），自定义角色继承；② `buildSystemPrompt()` 强制追加"保持性格、语气和边界严格一致性，不要念出规则"兜底约束；③ 每个角色绑定 ownerId + belongsToUser 权限校验，用户不能改他人角色；④ maxPersonasPerUser=10 防资源滥用；⑤ security 模块 OWASP 清洗 Filter 前置过滤。

### 6. "为什么情绪状态存储在内存而非持久化到数据库？"

情绪是高频更新的短生命周期状态（每次对话更新、每小时衰减），写 PostgreSQL 会产生大量写 IO。内存存储兼顾性能；重启丢失合理（衰减算法保证长期不活动的情绪已回到中性）。生产可替换 Redis（TTL=1h）。

### 7. "如果没有选择角色，系统如何表现？"

personaId 为 null 时使用默认 `AGENT_ROLE`（"你叫 BiBo，用中文回复，语言有激情，像阳光的中二少年"）；情绪注入跳过；记忆提取 emotionTag 为 null。向后兼容——不选角色完全感知不到角色/情绪系统存在。

### 8. "如何防止用户通过极端人设描述让 LLM 输出不安全内容？"

三道防线：预设模板硬编码行为边界约束；`buildSystemPrompt()` 强制追加兜底约束；security 清洗 Filter 前置过滤恶意输入。三重防护确保人设描述中的不当内容无法绕过安全护栏。

### 9. "情绪标签和记忆的情绪加权在实际查询中如何体现？"

pgvector 语义检索 SQL：

```
final_score = (1 - (embedding <=> query_vector)) × importance × CASE WHEN emotion_tag = '开心' THEN 1.1 ELSE 1.0 END
```

一条 SQL 完成向量相似度 × 重要性 × 情绪加权排序；情绪标签匹配 ×1.1 优先返回；`emotionTag = null` 时 CASE 恒为 1.0，退化为纯相似度 × importance，无额外开销。
