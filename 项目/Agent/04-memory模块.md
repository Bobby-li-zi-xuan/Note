# memory 模块

## 模块定位

memory 模块为 Agent 提供长期记忆（`com.yourorg.biboagent.memory`）：记忆事实存入 PostgreSQL + pgvector，后续对话自动召回并注入 Prompt，让 Agent 记住用户偏好、延续讨论、从历史中学习。

## 分层记忆体系

### L0：静态规则

外部规则目录（默认 `rules/`，可热重载、可经 API/前端在线编辑）优先，classpath 兜底（`resources/rules/*.md`，jar 内置只读）。每用户一个规则文件（如 `rules/u1.md`），自然语言描述固定偏好。每次请求由 PromptAssembler 读入拼到 System Prompt。

### L1：Caffeine 本地热点缓存

进程内缓存语义检索结果和 embedding 向量：`memoryQueryCache`（查询结果，默认 1000 条、5 分钟 TTL）和 `queryEmbeddingCache`（查询向量，默认 500 条、5 分钟 TTL）。连续多条消息复用热点记忆，把毫秒级查询降到微秒级；写入新记忆后驱逐该用户缓存保证强一致。

### L2：PostgreSQL + pgvector（语义检索 + 全量存储）

所有记忆以 `MemoryRecord` 存于 `memories` 表，`embedding` 为 1024 维 float32（Ollama bge-m3 生成），HNSW 索引（m=16, ef_construction=128）加速余弦距离（`<=>`）Top-K 检索。**检索排序公式**：`final_score = (1 - cosine_distance) × importance × (emotion_tag 匹配 ? 1.1 : 1.0)`。它是全系统唯一权威数据源（Single Source of Truth），一条 SQL 同时完成语义匹配 + 重要性排序 + 情绪加权。

**与旧架构的对比**：旧架构用 Redis ZSet + Hash 做索引层、MongoDB 存全量文档；新架构合并为 PostgreSQL + pgvector 单库，省去一种中间件，消除多级缓存的数据一致性问题。

**完整查询路径**：L0 静态规则注入 → L1 命中直接返回 → L1 未命中回源 L2 pgvector 检索并回填 L1 → 组装进 System Prompt。链路任何环节失败都不抛阻断异常：查询失败降级返回空列表（或 importance 排序），Agent 最多暂时"失忆"，不会导致主流程崩溃。

## 记忆提取

### 提取内容（MemoryRecord）

- `memoryId`（`m_` 前缀 + UUID）、`userId`；`type`：USER / FEEDBACK / PROJECT / REFERENCE
- `tags`：LLM 生成的关键词列表。两个用途：**矛盾检测前置过滤**（tags 无交集不可能矛盾，剪枝避免全量 cosine）+ **相似合并标签聚合**（合并后取并集）
- `content`（完整事实）+ `summary`（LLM 压缩 ≤128 字，Prompt 注入只用 summary 控 token；LLM 主动 search_memory 时返回 content 细节）
- `importance`（0~1，初始 = 提取 confidence）；**去重**：embedding cosine > 0.85 时收集**所有**超阈值已有记忆，合并全部 tags、取最大 importance，其余冗余物理删除（避免旧版"只合并一条、冗余残留"）

### 三级触发模型（if → else if → else if 互斥链）

```
Turn Loop 结束
├─ P0：用户明确要求存储（"记下来""帮我记"等关键词，yml 可配置）
│   → 同步执行，专用 Prompt 不做价值判断（只做合规检查，confidence >= 0.3 就提取）+ 即时反馈"已加入记忆"
├─ P1：step >= 2（发生工具调用/自纠正，有摘要上下文）
│   → 异步深度提取，注入 SystemMessage 中的结构化摘要；LLM 自主判断，confidence >= 0.5 才提取
├─ P2：周期到了（step 增量 >= 4 且距上次 >= 1 分钟）且信息密度足够（用户消息总量 >= 200 字符 且 实质消息数 >= 2）
│   → 异步浅度提取，仅注入原始 user/assistant 消息；LLM 自主判断，confidence >= 0.5
└─ 都不满足 → 不提取
```

设计原则：**提取由 LLM 自主判断，不由规则触发**（不确定输出 `[]`）。P0 关键词只决定"要不要即时反馈"，不影响提取判断。深度/浅度唯一区别是输入信息量——深度注入系统摘要（含 ongoingTasks、keyConstraints 等维度），浅度仅原始消息；UserMessage/AiMessage 均完整注入（AiMessage 截断至 800 字符），深度模式额外注入 SystemMessage 中含 `【系统注入】`/`压缩的历史上下文摘要` 标记的段落（截断至 1000 字符），浅度不注入 SystemMessage；Prompt 模板相同；当轮工具结果以 `tool(<工具名>): <内容>` 形式注入（单条截断至 800 字符、最多最近 10 条），供 LLM 提炼跨会话可复用信息。

**关键词触发为什么可以接受**：误触发（如"我记得上次"）多做一次同步 LLM 调用，LLM 判定信息不完整返回 `[]`，不产生错误记忆；漏触发由 P1/P2 异步提取兜底，yml 补关键词即可。

## 记忆整理与遗忘

### 相似合并（MemoryConsolidator，每小时 @Scheduled）

`SELECT DISTINCT user_id FROM memories` 找活跃用户 → 单用户包裹 try-catch（个别用户脏数据不影响同批次其他用户）→ 同类型记忆两两比较 embedding cosine，> `similarityThreshold`（0.85）则合并（保留重要性高者、删除冗余并建立 `EVOLVED_FROM` 关系）。写入阶段已做"新 vs 旧"去重，定时整理补"旧 vs 旧"的窄场景。

### 矛盾检测（已迁移至写入阶段）

`MemoryExtractor.writeMemory()` 每次写入时，利用已有 cosine 遍历结果做两层筛选：cosine 在 [0.4, 0.85) 且 tags 有交集的列为候选（最多 3 条）→ 调用轻量 LLM 判断是否矛盾。确认矛盾：旧记忆 `importance × 0.7` 衰减并建立 `CONTRADICTS` 关系。相比旧版定时全量扫描 + 否定词表，LLM 语义判断更精准（"改用 Vue 了"不含否定词也能识别）、实时、零额外 embedding 开销，且异步执行不阻塞响应。

### 遗忘调度（ForgetScheduler，每天凌晨 3 点 @Scheduled）

- **importance 衰减**：`importance = importance × max(minImportance, 1 - daysSinceLastAccess × dailyDecayRate)`（`dailyDecayRate` 默认 0.02，`minImportance` 默认 0.1）
- **状态降级**：低于 `dormantThreshold`（0.3）且超过 30 天未访问 → DORMANT（不注入 Prompt）；DORMANT 后低于 `archivedThreshold`（0.1）且超过 90 天 → ARCHIVED（仅 REST API 可查）
- **容量控制**：每用户 ACTIVE 记忆超 `maxActivePerUser`（2000）时删除重要度最低的超出部分

## 重要度机制

贯穿提取 → 检索 → 整理 → 遗忘全生命周期：

- **初始值** = 提取时 LLM 给的 confidence
- **随时间衰减**：未访问记忆每日按 `dailyDecayRate`（0.02）衰减，约 35 天减半；最近访问过的不衰减
- **被检索时递增**：检索命中按排名递增 importance（第 1 名 +0.08，第 2 名 +0.065，第 3 名 +0.05，上限 1.0），形成"越常被检索越重要"的正反馈
- **矛盾时衰减**：确认矛盾后 `× 0.7`
- **排序权重**：有 query 时 `cosine × importance × emotion_boost`；无 query 降级按 `importance DESC, updated_at DESC`

## 记忆关系体系

`RelationType`：`EVOLVED_FROM`（合并：被删记忆 → 保留记忆）、`CONTRADICTS`（矛盾：新记忆 → 旧记忆）、`SUPPORTS` / `RELATED`（预留）。`MemoryRelation` record 存 `memory_relations` 表（source/target/type/weight/createdAt），写入用 `ON CONFLICT ... DO UPDATE` 幂等 upsert（合并 weight 0.8，矛盾 0.9）。

**检索时**：① 过滤被推翻记忆——批量查询结果中是 CONTRADICTS 关系 target 的旧记忆从结果集剔除；② 关系补充——对 `importance > 0.8` 的记忆查询关联关系补充进结果集（上限 `limit + 2`，用 seenIds 去重）。

## 面试题整理

### 1. "为什么需要分层记忆体系？直接全部放 PostgreSQL 不行吗？"

性能与成本权衡。全部走 pgvector 语义检索，热用户在短时间内连续多条消息也要每次都做向量计算；L1 Caffeine 把热点查询从毫秒降到微秒。L0 静态规则用"外部目录优先 + classpath 兜底"兼顾可编辑性与默认安全。三层不是冗余，是对不同访问模式和延迟要求的精确匹配。（架构演进：旧架构另有 Redis 索引层 + MongoDB 存储层，新架构合并为 PostgreSQL + pgvector 单库，见正文。）

### 2. "embedding cosine 对比旧版 Jaccard 有什么优势？"

语义层面感知："喜欢 Java"与"热爱 Java"字符差异大但语义等价，cosine 能捕捉；长文本判断也更稳定（Jaccard 长文本并集远大于交集导致低分）。Jaccard 优势是零模型依赖、计算便宜；embedding 需要 Ollama bge-m3 生成，有延迟和资源开销，但准确性显著提升。

### 3. "矛盾检测是怎么工作的？"

已从定时整理迁移至写入阶段：写入时利用已有 cosine 结果做两层筛选——cosine 在 [0.4, 0.85) 且 tags 有交集列为候选（最多 3 条），调用轻量 LLM 判断是否矛盾。确认矛盾 → 旧记忆 importance × 0.7 衰减 + 建立 CONTRADICTS 关系。优势：不需要维护否定词表（"改用 Vue 了"也能识别）、利用写入时已算好的 cosine 零额外开销、异步执行不阻塞响应。

### 4. "记忆提取的三级触发模型是怎么设计的？"

P0→P1→P2 互斥链。P0：用户明确要求存储 → 同步提取 + 即时反馈，LLM 不做价值判断只做合规检查。P1：step ≥ 2 → 异步深度提取（注入结构化摘要）。P2：周期到了 + 信息密度足够 → 异步浅度提取（仅原始消息）。核心原则：提取由 LLM 自主判断（不确定输出 `[]`），P0 关键词只决定反馈方式；深度/浅度仅区别是否注入系统摘要，Prompt 模板相同。

### 5. "重要度机制如何帮记忆系统做淘汰决策？"

importance（0~1）代表保留价值，初始 = 提取 confidence。淘汰三层次：矛盾被否定 ×0.7；每日遗忘调度按 dailyDecayRate 衰减未访问记忆，低于阈值降级 DORMANT（0.3/30 天）→ ARCHIVED（0.1/90 天）；容量控制（每用户 ACTIVE ≤ 2000）。软删除策略——不立刻抹掉，给"缓刑期"逐渐降权自然淘汰。

### 6. "当多轮对话历史超长（超出大模型上下文窗口）时，系统是如何处理的？"

采用**动态截断与即时摘要**机制：每次请求发起前（`HistoryTruncator.truncate`，由 AgentService 调用），系统用 jtokkit 精确估算历史消息 token 量（纯文本 + 消息结构开销），（消息 + SystemPrompt + 工具 schema 固定开销）占比 ≥70% 时对旧消息执行结构化摘要（<70% 不处理）。

1. **保留近期脉络**：按 token 预算自适应保留最近消息（`recent-message-budget-ratio` 默认 30% 上下文，`min-keep-rounds=2`），工具调用与结果成对保留，避免截断出孤儿工具。
2. **压缩远期上下文（结构化 JSON 摘要）**：按 6 维 JSON 提取：核心诉求/临时规则/任务进度/约束条件/待办事项/工具结果，单条 ≤30 字。Prompt 注入已存储记忆做去重（"绝对不要重复提取"），并要求"新覆盖老"、标注"(已更新)"处理冲突。
3. **复合上下文组装**：摘要注入 SystemPrompt 末尾（SystemPrompt 与摘要解耦，不再被误压缩）。截断后若"摘要 + 消息"占比仍 ≥70%，递归再截断（最多 2 层，递归重检计入摘要 token）；到达深度上限后执行最后一轮结构化摘要，不再递归。LLM 摘要彻底失败时降级为关键词兜底（"禁止、必须、架构"等词）。

**总结**：该机制在避免"爆窗"与控制成本的前提下保留长线语境连贯性。摘要只服务当前回合上下文传递，与长期记忆落地（写入 PostgreSQL）是两套解耦机制——截断摘要不写入长期记忆库（Phase 4 异步预压缩会短暂缓存到 Redis 供下一轮复用）。
