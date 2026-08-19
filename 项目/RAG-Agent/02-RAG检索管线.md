# 02 · RAG 检索管线

## 一、管线全景

RAG 检索管线是项目中最核心的技术模块，完整流程覆盖文档入库和查询检索两个方向：

```
【文档入库】
Markdown 文件
  → MarkdownParserService.parseMarkdown()
  → List<MarkdownSection> (title + content)
  → MarkdownChunkComposer.compose(title, body, maxTokensPerChunk)
  → doEmbed(chunk content) → Ollama bge-m3 → float[1024]
  → PgVectorTypeHandler → PostgreSQL chunk_bge_m3 表

【查询检索】
用户 query
  → doEmbed(query) → float[1024]
  → toPgVector() → "[0.12,0.34,...]"
  → chunkMapper.similaritySearch(kbId, vector, maxDistance, candidateLimit)
  → RagSearchFilter.filter(candidates, maxDistance, jaccardThreshold)
  → RagContextAssembler.assemble(filtered, maxTokensPerChunk, maxChunksPerKb)
  → 编号化输出: "【1】chunk内容\n【2】chunk内容"
```

---

## 二、Embedding 嵌入层

### 2.1 为什么选 bge-m3

bge-m3 是 BAAI 开源的多语言 embedding 模型，1024 维向量。选择的理由：

- **本地部署**：通过 Ollama 运行，不依赖外部 API，零调用成本
- **中文友好**：bge-m3 对中文语义理解能力强，比 OpenAI text-embedding-ada-002 在中文场景表现更好
- **维度适中**：1024 维在精度和存储成本之间取得了平衡

### 2.2 WSL 环境适配

Ollama 运行在 Windows 宿主机，开发环境在 WSL2 里。当前实现通过 `ip route` 自动检测 WSL 网关 IP：

```java
// OllamaConfig
private String detectWslGateway() {
    Process process = new ProcessBuilder("ip", "route").start();
    // 解析 "default via 172.30.112.1 dev eth0"
    return reader.lines()
        .filter(line -> line.startsWith("default "))
        .map(line -> line.split("\\s+")[2])
        .findFirst().orElse(null);
}
```

非 WSL 环境（如纯 Linux 服务器）则使用 localhost:11434。

### 2.3 Embedding 调用

```java
private float[] doEmbed(String text) {
    EmbeddingResponse resp = webClient.post()
        .uri("/api/embeddings")
        .bodyValue(Map.of("model", "bge-m3", "prompt", text))
        .retrieve()
        .bodyToMono(EmbeddingResponse.class)
        .block();
    return resp.getEmbedding();
}
```

使用 WebClient 而非 RestTemplate，支持响应式调用，可在并发场景下复用连接池。

---

## 三、pgvector 向量检索

### 3.1 为什么用 pgvector 而不是专用向量数据库

这是一个关键的架构决策。选择 pgvector 的理由：

| 维度 | pgvector | Milvus / Pinecone |
|------|----------|-------------------|
| 部署复杂度 | 零——已在用 PostgreSQL | 需要额外服务 |
| 数据一致性 | 天然与业务数据同一事务 | 需要自己维护一致性 |
| 运维成本 | 无额外组件 | 需要独立监控和运维 |
| 扩展性 | 十万级向量没问题 | 百万级以上更优 |
| 混合查询 | SQL JOIN 自然支持 | 需要跨系统拼接 |

对于当前数据规模（知识库通常几百到几千条 chunk），pgvector 完全够用。如果未来数据量上到百万级，再考虑引入 Milvus 做向量检索层。

### 3.2 PgVectorTypeHandler

MyBatis 不原生支持 pgvector 类型，需要自定义类型处理器：

```java
// 写入：float[] → "[0.1,0.2,...]"
public void setNonNullParameter(PreparedStatement ps, int i,
        float[] parameter, JdbcType jdbcType) {
    ps.setObject(i, toPgVectorString(parameter), Types.OTHER);
}

// 读取：PGobject → float[]
public float[] getNullableResult(ResultSet rs, String columnName) {
    PGobject obj = (PGobject) rs.getObject(columnName);
    return parseVector(obj.getValue()); // "[0.1,...]" → float[]
}
```

### 3.3 相似度检索 SQL

```sql
SELECT c.*, 1 - (c.embedding <-> #{queryEmbedding}::vector) AS relevance_score
FROM chunk_bge_m3 c
WHERE c.kb_id = #{kbId}
  AND 1 - (c.embedding <-> #{queryEmbedding}::vector) >= #{minSimilarity}
ORDER BY c.embedding <-> #{queryEmbedding}::vector
LIMIT #{candidateLimit}
```

关键点：
- `<->` 是 pgvector 的 L2 欧几里得距离操作符
- `1 - distance` 转换为余弦相似度（因为 bge-m3 输出的是归一化向量，L2 距离与余弦距离等价）
- `minSimilarity` 默认 0.5，过滤掉过于不相关的结果

---

## 四、RagSearchFilter 过滤层

从 pgvector 拿回候选 chunk 后，还需要做两步过滤：

### 4.1 距离过滤

```java
for (Chunk c : candidates) {
    if (c.getRelevanceScore() <= maxDistance) {
        byDistance.add(c);
    }
}
```

其中 `maxDistance = 1.0 - minSimilarity`，把配置的相似度阈值转换为距离阈值。

### 4.2 两段式 Jaccard 去重

这是检索质量优化的关键。chunk 之间的内容可能高度重叠，直接全塞给 LLM 会浪费 token 预算。去重策略是：

```java
// 把每个 chunk 拆成标题部分和正文部分
SplitResult a = splitTitleBody(kept.getContent());
SplitResult b = splitTitleBody(current.getContent());

double titleJaccard = jaccard(a.title, b.title);
double bodyJaccard = jaccard(a.body, b.body);

// 仅当标题 AND 正文都高度相似时才视为重复
if (titleJaccard > jaccardThreshold && bodyJaccard > jaccardThreshold) {
    isDuplicate = true;
}
```

两段式去重的好处：
- 标题相同但正文不同 → 保留（不同段落讲不同内容）
- 正文相似但标题不同 → 保留（不同章节可能有类似表述但不同主题）
- 标题且正文都相似 → 去重（基本是同一内容）

Jaccard 相似度用字符 2-gram 计算，不需要分词器，对中英文都适用：

```java
private double jaccard(String a, String b) {
    Set<String> gramsA = buildBigramSet(a);  // {"ab","bc","cd",...}
    Set<String> gramsB = buildBigramSet(b);
    // Jaccard = |A ∩ B| / |A ∪ B|
}
```

---

## 五、RagContextAssembler 上下文拼装

### 5.1 设计动机

pgvector 返回的 chunk 是一串无序列表。直接拼接会丢失文档的层级结构。RagContextAssembler 的目标是：**在 token 预算约束下，产出最紧凑、最有结构感的上下文。**

### 5.2 三步组装

**Step 1：稳定排序**

```java
List<Chunk> sorted = chunks.stream()
    .sorted(Comparator
        .comparingDouble(Chunk::getRelevanceScore)  // 先按相关性
        .thenComparingInt(c -> -contentLength(c))     // 再按内容长度降序
        .thenComparing(c -> c.getUpdatedAt(), nullsLast(reverseOrder())))  // 再按更新时间
    .collect(toList());
```

**Step 2：贪心 packing**

在总 token 预算（`maxTokensPerChunk × maxChunksPerKb`）内，按排序逐个装入。

**Step 3：邻接合并**

如果两个相邻 chunk 满足条件——同文档、同节标题、chunk 索引连续、合并后不超预算——就把它们合并成一个块。这恢复了被切块打断的文档结构。

### 5.3 Token 估算

```java
public class TokenEstimator {
    public static int estimate(String text) {
        // 中文: 字符数 ≈ token 数（粗略但快速）
        // 英文: 字符数 / 4 ≈ token 数
    }
}
```

这是快速估算，不是精确 tokenize，但对于上下文预算控制来说够用了。

---

## 六、配置调优

```yaml
rag:
  search:
    candidateLimit: 12      # SQL 召回上限
    minSimilarity: 0.5      # 最低相似度阈值（太低会有噪声，太高会漏掉相关内容）
    maxChunksPerKb: 4       # 单个知识库最大返回块数
    defaultToolOutputLimit: 4  # 工具默认输出块数
    jaccardThreshold: 0.8   # Jaccard 去重阈值（越高越保守，>0.8才去重）
  chunk:
    maxTokensPerChunk: 800  # 单个 chunk 的 token 预算上限
```

调参经验：
- `minSimilarity: 0.5` 是比较宽松的阈值。实际测试中发现，设太高（比如 0.75）会让很多语义相关但用词不同的内容被过滤掉
- `jaccardThreshold: 0.8` 是保守去重——宁少去不能多去，因为丢了相关内容的代价比多花 token 大
- `candidateLimit: 12` 从默认的 8 提升到 12，给过滤和拼装层更多选择空间

---

## 七、MarkdownChunkComposer 语义切块

切块不是暴力按字符数切割，而是保留文档的结构语义：

```
优先级从高到低：
1. Fenced code block（```...```）→ 原子块，不切割
2. 表格行 → 原子块
3. 段落边界（空行）→ 优先切分点
4. 句末标点（。！？\n）→ 次优切分点
5. 固定长度切分 → 最后手段
```

这样保证了 LLM 拿到的每个 chunk 都是语义完整的片段，而不是被拦腰截断的半句话。

---

## 八、面试问答

### Q: 你们是用的什么 embedding 模型？为什么？

> 用的是 bge-m3，通过 Ollama 本地部署。选它主要三个原因：一是中文效果好，BAAI 专门为中文做的优化；二是本地跑零成本，不会像调 OpenAI 的 embedding API 那样按 token 计费；三是 1024 维，精度和存储都够用。我的 RAG 项目不是面向百万级文档的企业搜索，几百到几千条 chunk 的规模，1024 维加上 pgvector 索引完全撑得住。



### Q: 你们怎么解决 embedding 调用的性能问题（并发瓶颈怎么解决）？

> 目前有两个优化。一是连接池——WebClient 本身就是异步非阻塞的，高并发下不会因为线程阻塞浪费资源。二是 embedding 调用不在 Agent 循环的主路径上——只在用户查询时调一次，工具结果不再重复 embedding。

> 更大的优化空间在批量化——如果文档入库时一次性处理几百个 chunk，现在是一个一个调 embedding，改成批量调用能快很多。但 Ollama 的 batch embedding 支持还需要验证。

### Q: Jaccard 去重为什么要分标题和正文两段？

> 因为"两段都相似才算重复"这个逻辑更符合语义。举个例子：一本书的不同章节可能有类似的写作风格，正文的 Jaccard 相似度会偏高，但那不是重复内容。只有标题也高度一致时，才说明这两段在讲同一件事。

> 如果不用两段式，而是直接对整个 chunk 算 Jaccard，就会过度去重——把不同章节但用词类似的内容误判为重复。

### Q: chunk 切块大小怎么定？

> 当前是 800 tokens。这个数字不是拍脑袋的——太小的 chunk 缺少上下文，LLM 看到"机器学习是..."这种半句话没法用；太大的 chunk 会把多个主题挤在一起，检索精度下降。800 tokens 大概是 500-600 个中文字，能装下一个完整段落加几个辅助句子，又不会过于分散。

> 而且我的切块策略会优先在语义边界切——代码块不动、表格不拆、段落结尾优先断。所以 800 是最多 800，很多时候会比这个短。

### Q: 如果检索结果全是"不相关"的，你们怎么处理？

> 当前的做法是交给 LLM 自行判断。检索没有结果时 KnowledgeTool 会返回"未从知识库中检索到相关内容"，LLM 看到这个之后应该告诉用户"你的问题在知识库里没找到"而不是强行编造答案。

> 这个设计是故意的——我不想在 RAG 层替 LLM 做"是否相关"的判断。因为相关性是一个主观概念，一个 chunk 对问题 A 不相关，对问题 B 可能高度相关。把判断权交给 LLM，利用它的语义理解能力比我自己设一个硬阈值要好。

### Q: 为什么使用 L2 距离而不是余弦距离？

> 实际上 bge-m3 输出的是归一化向量，归一化之后 L2 距离和余弦距离是等价的——有一个数学关系：对单位向量，`L2² = 2(1-cosθ)`。所以配置里写 `1 - distance` 就是在计算余弦相似度。

> 但严格来说用 `<->` 操作符算 L2 距离再转换多了一步。如果追求极致性能，可以自己把向量归一化后用 `<#>` 内积操作符，少做一次转换。这是个小优化点。

### Q: 你们评估过检索质量吗？

> 我做了几项量化测试。Embedding 延迟平均在几百毫秒，受 Ollama 本地推理速度影响。Jaccard 去重的效果也验证过——在测试集上能过滤掉 10-20% 的高度重复 chunk，同时对相关 chunk 的保留率接近 100%。

> RAG 准确率测试目前依赖真实的块数据——需要有标记的"查询→正确答案块"对。这部分数据还在积累，目前主要靠人工抽样验证。如果要更严谨地评估检索质量，可以用 Ragas 这类框架做自动化评测。

### Q: 如果面试官问"怎么做一个好的 chunk 策略"，你怎么答？

> 好问题，我会分三个层面回答。第一是**怎么切**——优先在语义边界切，不要在句子中间砍断，代码块和表格要保持完整。第二是**切多大**——取决于 embedding 模型的上下文窗口和检索精度需求，我用的 800 tokens 对 bge-m3 来说适中。第三是**切完之后怎么办**——每个 chunk 要带元信息，比如文档标题、章节名、chunk 序号，这样拼装的时候能把被打散的结构恢复回来。
