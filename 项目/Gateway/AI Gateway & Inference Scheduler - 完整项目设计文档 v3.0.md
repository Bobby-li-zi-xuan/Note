# AI Gateway & Inference Scheduler — 完整项目设计文档 v3.0

> 本文档在 v2.0 基础上，系统吸收 LiteLLM、New API（One API）、Portkey、Higress、Kong、APISIX、Envoy AI Gateway 等主流开源 AI 网关的设计优点，重构并完善整体设计。
> **本文档为唯一权威设计文档**，v2.0 及以下版本的设计描述以本文档为准。

---

## 1. 项目定位

> 面向大模型推理场景的**轻量化、企业级、智能路由调度中间件**（Mini AI Inference Platform）。

### 1.1 核心价值主张

- **统一接入**：一套 OpenAI 兼容 API 接入多家模型供应商，调用方零感知。
- **智能调度**：基于实时状态与多策略（延迟 / 成本 / 质量）自适应路由，而不是简单的轮询或随机。
- **企业治理**：渠道、令牌、配额、限流、成本预算形成闭环。
- **高可用**：重试、分层超时、链式降级、熔断、流式代理。
- **可观测**：全链路指标与状态驱动的决策闭环，路由决策可解释、可复盘。

### 1.2 与开源项目的差异化

| 维度 | LiteLLM | New API | 本项目 |
|---|---|---|---|
| 定位 | LLM 代理 / 网关（Python+Rust） | 渠道分发与计费平台（Go） | 状态驱动的智能调度网关（Java 21 / Spring MVC + 虚拟线程） |
| 路由方式 | 负载均衡 + 路由插件信号 | 渠道加权随机 + 失败重试 | 多目标打分（延迟/成本/质量/健康）+ 策略引擎 + 插件信号 |
| 治理重点 | 预算、Guardrail、日志 | 令牌、额度、计费、控制台 | 渠道/令牌/配额/成本闭环 + 智能调度 |
| 独特能力 | Provider 适配最广 | 计费生态最完整 | 决策与执行分离、状态闭环驱动的推理调度 |

---

## 2. 开源项目借鉴清单

| 开源项目 | 核心优点 | 本项目吸收点 |
|---|---|---|
| **LiteLLM** | ① 统一 OpenAI 兼容接口 + 100+ Provider 适配；② 路由插件流水线（RoutingContext + signals，插件按序执行、可收窄候选池）；③ 成本追踪、预算、Guardrail；④ 配置即代码（config.yaml） | ① Connector SPI 抽象；② **路由信号（Signals）与候选收窄机制**；③ 成本计量模型与预算；④ 声明式配置 Schema |
| **New API / One API** | ① 渠道管理（权重、优先级、自动重试）；② 令牌管理（配额、过期、模型范围、IP 白名单）；③ 额度计费、兑换码、数据看板；④ OpenAI/Claude/Gemini 协议互转 | ① **Channel（渠道）领域模型**；② **ApiToken 与配额模型**；③ 用量/成本看板（V2）；④ 协议适配层预留互转能力 |
| **Portkey** | ① 条件路由、Fallback、熔断、多 Key 负载均衡；② 简单缓存 + 语义缓存；③ 预算与多维度限流；④ 金丝雀发布；⑤ Agent 网关（MCP/A2A）方向 | ① 条件路由规则；② 语义缓存插件（V2）；③ 预算/限流维度设计；④ **金丝雀路由策略**；⑤ MCP 支持列入远期 |
| **Higress** | ① 多模型统一路由 + 协议转换；② 语义缓存；③ Token 级限流；④ Prompt 注入检测等安全能力；⑤ 热插拔插件；⑥ **成本配额触达自动降级到低成本模型** | ① 插件生命周期与热加载；② 安全护栏插件（V2）；③ **成本驱动的模型降级策略**；④ Token 级限流 |
| **Kong** | ① 成熟的插件化架构（全局/路由级作用域）；② AI Proxy、按 Token 限流、语义 Prompt Guard、PII 脱敏等 AI 专项插件 | ① **阶段化插件链与作用域模型**；② AI 专项插件目录设计 |
| **APISIX** | ① 云原生网关 + 插件生态；② 配置热更新、Schema 校验 | ① 插件配置 Schema 校验；② 配置版本化与原子生效 |
| **Envoy AI Gateway** | ① **速率限流与配额预算分离**（velocity vs budget）；② 按用户/模型维度 Token 计量与预算；③ 上游鉴权 | ① 双层治理模型（限流 + 配额）；② 按 Token 的用量计量与按身份预算 |

---

## 3. 总体架构

### 3.1 全局架构

```
Client（OpenAI 兼容 SDK / HTTP）
                │
                ▼
┌─────────────────────────────────────────────────┐
│              接入层 API Gateway Layer            │
│   /v1/chat/completions   /v1/models  Admin API  │
└─────────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────┐
│            数据面 Data Plane（请求处理）          │
│                                                 │
│   请求流水线（插件链：鉴权→限流→护栏→缓存→识别）   │
│                    │                           │
│                    ▼                           │
│   Decision Engine 决策引擎                       │
│   Filter Chain（候选收窄）→ Scorer（多目标打分）  │
│   → Scheduler（排序/加权/生成降级链）             │
│                    │                           │
│                    ▼                           │
│   Routing Execution Layer 执行层                │
│   Connector / Retry / Timeout / Fallback       │
│   / CircuitBreaker / Stream Proxy              │
└─────────────────────────────────────────────────┘
                │
                ▼
        Model Service（真实/模拟推理服务）
                │
                ▼
┌─────────────────────────────────────────────────┐
│   可观测层：Metrics Pipeline → Logs → Traces     │
└─────────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────┐
│   状态系统 State System（健康/负载/延迟/熔断态）   │
└─────────────────────────────────────────────────┘
                │
                ▼
        决策引擎驱动下一轮决策（状态闭环）

┌─────────────────────────────────────────────────┐
│  控制面 Control Plane（低频管理）                 │
│  ModelRegistry / ChannelManager / TokenManager  │
│  PolicyManager / Quota&Cost / ConfigService     │
│  Admin API → 管理后台（V2）                      │
└─────────────────────────────────────────────────┘
```

### 3.2 核心设计原则

1. **决策与执行完全分离**：决策层只负责“选策略、选模型、出方案”，执行层只负责“按方案落地”，通过 `RoutingDecision` 纯数据结构解耦，可独立测试、独立扩展。
2. **控制面与数据面分离**：控制面低频执行、可复杂计算；数据面超高并发、极低延迟、纯流量治理。
3. **状态驱动闭环**：请求 + 模型实时状态 + 场景策略 → 动态决策 → 流量执行产生真实运行指标 → 更新全局模型状态 → 驱动下一次决策。
4. **插件化请求流水线**（新增，借鉴 Kong/Higress/LiteLLM）：所有横切能力（鉴权、限流、护栏、缓存、审计）以插件形式挂在生命周期上，核心路由保持稳定。
5. **配置即代码 + 热更新**（新增，借鉴 LiteLLM/APISIX）：模型、渠道、策略、插件全部声明式配置，支持版本化热更新。
6. **OpenAI 兼容协议优先**：对外协议以 OpenAI 格式为唯一契约，内部协议适配通过 Connector 隔离。

---

## 4. 请求生命周期与插件流水线

### 4.1 请求生命周期

```
1 鉴权（ApiToken 校验：配额/模型范围/IP）
2 限流（QPS/Token 速率，滑动窗口）
3 安全护栏（Prompt 注入检测/敏感内容，V2）
4 语义缓存查询（V2，命中则直接返回）
5 任务识别（复杂度/语言/领域分类，写入 Signals）
6 决策引擎
   6.1 Filter Chain 候选收窄（能力/健康/预算/策略/插件信号）
   6.2 Scorer 多目标打分（延迟/成本/质量/健康）
   6.3 Scheduler 排序 + 加权随机 + 生成降级链
7 执行层（Connector → Retry → Timeout → Fallback → CircuitBreaker → StreamProxy）
8 响应转换（协议归一、错误格式统一）
9 用量计量与成本核算（UsageRecord → Quota/Budget 扣减）
10 可观测事件（Metrics/Logs/Trace）→ 状态系统更新
```

### 4.2 插件契约（Plugin SPI）

```java
public interface GatewayPlugin {
    String name();
    PluginScope scope();          // GLOBAL / ROUTE / MODEL
    int order();                  // 同阶段内的执行顺序

    // 生命周期钩子
    void beforeDecision(PluginContext ctx);
    void afterDecision(PluginContext ctx);
    void beforeExecution(PluginContext ctx);
    void afterExecution(PluginContext ctx);
    void onError(PluginContext ctx, Throwable t);
}
```

- `PluginContext` 携带：原始请求、元数据（租户/调用方）、`signals`（路由信号）、候选集合 `candidates`、`decision`（可写）。
- 插件按 `order` 顺序执行，后置插件可以看到前置插件写入的信号与收窄后的候选池（借鉴 LiteLLM Router Plugins）。
- **Fail-closed 语义**：插件可以把候选池收窄，若某插件把候选清空，请求直接拒绝而不是静默回退全量候选，确保策略链不可被绕过（借鉴 LiteLLM）。
- 内置插件目录（P1 起）：`auth`、`rate-limit`、`quota`、`prompt-guard`（V2）、`semantic-cache`（V2）、`cost-degrade`、`canary`、`audit-log`。

### 4.3 路由信号（Signals）机制

借鉴 LiteLLM 的 `RoutingContext.signals`：各阶段/插件向上下文写入结构化信号，供决策引擎消费。

```json
{
  "language": "zh",
  "task_complexity": "SIMPLE",
  "tenant": "default",
  "budget_remaining_ratio": 0.72,
  "cache_hit": false,
  "canary_group": "stable"
}
```

Scorer 可基于信号调整权重（例如：预算不足时提高成本权重、复杂任务强制走质量优先策略），实现**信号驱动的自适应调度**。

---

## 5. 核心领域模型

| 实体 | 关键字段 | 说明 |
|---|---|---|
| `Channel` | id、provider、baseUrl、credentialsRef、models、priority、weight、status、quota | 上游渠道（借鉴 New API）：一个供应商账号/端点 + 一组模型 |
| `ModelInstance` | id、model、channelId、capabilities、priceIn/priceOut、latencyProfile | 具体模型实例，v2.0 模型演进 |
| `ModelAlias` | alias、strategy、candidateSelectors | 对外模型名 → 候选实例集合 + 路由策略 |
| `ApiToken` | id、tokenHash、quota、modelScope、ipWhitelist、expiresAt | 调用方令牌（借鉴 New API） |
| `Policy` | id、type、params、weightVector | 策略模板：latency-first / cost-first / quality-first / balanced / conditional |
| `PluginConfig` | name、scope、enabled、order、config | 插件实例化配置，带 Schema 校验 |
| `RoutingContext` | request、metadata、signals、candidates | 决策上下文（v2.0 的 RequestContext 演进） |
| `RoutingDecision` | primary、fallbackChain、scoringDetails | 标准调度方案（v2.0 保留，新增 fallbackChain 与可解释打分明细） |
| `UsageRecord` | requestId、tokenIn、tokenOut、cost、channelId、model、latency、result | 单次调用计量 |
| `Budget` | scope、limitType（TOKEN/COST）、period、limit、used、alerts | 配额预算（借鉴 Envoy AI Gateway / Portkey） |
| `ModelState` | instanceId、healthy、circuitOpen、ewmaLatency、errorRate、load | 实时状态，决策引擎输入（v2.0 保留） |

---

## 6. 决策引擎设计

### 6.1 Filter Chain（候选收窄）

按顺序执行，任一阶段可淘汰候选：

1. **能力过滤**：请求所需能力（streaming、vision、tools 等）不在实例能力集内的淘汰。
2. **可用性过滤**：健康检查失败、熔断开启、权重为 0 的实例淘汰。
3. **策略过滤**：租户/令牌的模型范围、模型分组限制。
4. **预算过滤**：超出渠道/模型预算的候选降权或淘汰（可配置）。
5. **插件信号过滤**：LiteLLM 式插件收窄（语言、复杂度、金丝雀分组等）。

### 6.2 Scorer（多目标打分）

```
score(instance) = w1 * norm(latency) + w2 * norm(cost) + w3 * norm(quality) + w4 * norm(health)
```

- `latency` 取 `ModelState.ewmaLatency`（指数移动平均实时延迟）。
- `cost` 由单价表（input/output per 1K tokens）与预估 Token 计算。
- `quality` 默认静态配置，可由插件动态写入（如评测分、反馈分）。
- `health` 由错误率、熔断状态、负载计算。
- 权重向量 `w` 由 `Policy` 提供，可在控制面热更新；支持基于 Signals 的动态调权（如预算紧张时放大成本权重）。

### 6.3 策略类型

| 策略 | 说明 | 借鉴来源 |
|---|---|---|
| `MULTI_OBJECTIVE` | 默认：多目标加权打分 | 本项目核心 |
| `WEIGHTED_RANDOM` | 同分候选按渠道权重随机 | New API |
| `CONDITIONAL` | 显式规则路由：`if task=SIMPLE → [mini 模型]` | Portkey |
| `FALLBACK_CHAIN` | 主选 + 顺序降级链 | Portkey / Higress |
| `CANARY` | 按流量比例切新模型/新渠道，灰度验证 | Portkey |

### 6.4 Scheduler

- 对打分结果排序，产出 `RoutingDecision`（主选 + 降级链 + 打分明细，支持面试可讲的“可解释路由”）。
- 可选**会话亲和**（同一会话尽量命中同一实例，关闭时保证策略变更即时生效，借鉴 LiteLLM 的 trade-off）。

---

## 7. 执行层设计

### 7.1 Connector SPI（协议适配）

```java
public interface Connector {
    void stream(ChatRequest request, Channel channel, Consumer<ChatCompletionChunk> consumer);
    ChatCompletion complete(ChatRequest request, Channel channel);
}
```

- P1 实现 `OpenAICompatibleConnector`（覆盖 OpenAI、DeepSeek、通义、moonshot、vLLM 等兼容服务）。
- V2 增加协议转换 Connector（Anthropic、Gemini），对外仍统一 OpenAI 格式（借鉴 New API 协议互转）。

### 7.2 容错治理

| 能力 | 设计要点 | 借鉴来源 |
|---|---|---|
| Retry | 可配置次数、指数退避 + 抖动、按状态码触发（网络错误/429/5xx）、流式中断重试 | LiteLLM / Portkey |
| Timeout | 分层超时：连接、响应、首字节、总时长；流式空闲超时 | Portkey |
| Fallback | 沿 `RoutingDecision.fallbackChain` 顺序降级；**成本配额触顶自动降级到低成本模型** | Portkey / Higress |
| CircuitBreaker | 按 `channel+model` 维度；滑动窗口错误率/慢调用阈值；熔断 → 半开探测；状态写入 `ModelState` | Portkey / 通用网关 |
| Stream Proxy | SSE 透传、增量 Token 计量、背压、客户端断开时取消上游调用 | v2.0 保留强化 |

---

## 8. 限流与配额（双层治理）

借鉴 Envoy AI Gateway 的 **velocity（速率）与 budget（预算）分离**设计：

| 维度 | 限流（Rate Limit） | 配额（Quota/Budget） |
|---|---|---|
| 度量 | QPS / RPM / Token 速率 | Token 总量 / 成本金额 |
| 窗口 | 秒/分钟级滑动窗口 | 小时/天/月周期 |
| 作用域 | 全局、令牌、用户、模型、渠道 | 令牌、租户、模型、渠道 |
| 超额行为 | 拒绝（429） | 拒绝 / 预警 / **自动降级到低成本模型** |

实现：P1 单实例内存实现（滑动窗口 + 计数器），P3 可选 Redis 分布式实现。

---

## 9. 成本与计费模型

1. **单价表**：每个 `ModelInstance` 配置 `priceIn / priceOut`（每 1K tokens）。
2. **计量**：非流式在响应后一次性计量；流式在 `Stream Proxy` 中按 chunk 增量累计 Token，结束时生成 `UsageRecord`。
3. **成本核算**：`cost = tokenIn * priceIn + tokenOut * priceOut`。
4. **预算扣减**：`Budget` 按 scope 扣减，支持预警阈值（70%/90%）与超额策略。
5. **查询**：Admin API 按令牌/模型/渠道汇总用量与成本（V2 提供看板）。

> V2 可选：兑换码、充值、多级用户体系（借鉴 New API）。若定位企业内部工具，可裁剪。

---

## 10. 管理面（Control Plane）

### 10.1 配置即代码

单一 `gateway.yaml`（可拆分为多文件），Schema 校验后加载：

```yaml
models:
  - alias: gpt-4o-mini
    strategy: balanced
    candidates:
      - channel: openai-main
        model: gpt-4o-mini
channels:
  - id: openai-main
    provider: openai-compatible
    baseUrl: https://api.openai.com/v1
    credentialsRef: env:OPENAI_API_KEY
    weight: 10
policies:
  cost-first:
    type: MULTI_OBJECTIVE
    weights: { latency: 0.2, cost: 0.6, quality: 0.1, health: 0.1 }
plugins:
  - name: auth
    order: 1
  - name: rate-limit
    order: 2
    config: { qps: 100 }
limits:
  budget:
    - scope: token:my-token
      limit: { type: COST, period: MONTHLY, max: 100 }
```

### 10.2 Admin API（P1）

- 渠道 CRUD、启停、权重调整
- 令牌创建/吊销、配额调整
- 策略与插件配置热更新（版本化、校验后原子生效）
- 用量/成本/日志查询

---

## 11. 可观测性

### 11.1 指标（Prometheus 格式）

| 指标 | 说明 |
|---|---|
| `gateway_requests_total{model,channel,result}` | 请求量 |
| `gateway_latency_seconds{model,stage}` | 延迟分布（P50/P95/P99） |
| `gateway_error_rate{model,channel}` | 错误率 |
| `gateway_token_usage{model,type}` | Token 用量（in/out） |
| `gateway_cost_total{model,channel}` | 成本 |
| `gateway_circuit_breaker_events{channel,model}` | 熔断事件 |
| `gateway_fallback_events{from,to}` | 降级事件 |
| `gateway_cache_hit_total` | 语义缓存命中（V2） |
| `gateway_decision_distribution{strategy}` | 路由策略分布 |

### 11.2 日志与链路

- 结构化日志：请求 ID 贯穿全链路，记录决策打分明细（面试亮点：**可解释路由**）。
- 可选 OpenTelemetry Trace（P2）。

### 11.3 状态闭环

指标流水线 → 聚合为 `ModelState`（EWMA 延迟、错误率、熔断态）→ 写入状态系统 → 决策引擎下一轮消费。**这就是“AI Gateway + Inference Scheduler”区别于普通网关的核心卖点。**

---

## 12. 安全设计

- **密钥托管**：上游密钥只通过配置引用（`env:XXX`），不落库、不落日志（借鉴 LiteLLM）。
- **令牌鉴权**：ApiToken 校验 + 模型范围 + IP 白名单 + 过期时间（借鉴 New API）。
- **Prompt 安全**（V2）：Prompt 注入检测、敏感内容识别、PII 脱敏插件（借鉴 Higress / Kong）。
- **审计日志**（V2）：管理面操作审计。

---

## 13. 非功能需求

- **性能**：非流式请求网关额外开销 P99 < 10ms；流式首字节低延迟；单实例支撑 1k+ QPS。
- **可靠性**：单进程可完整运行；状态可重建（内存态可从指标重放）。
- **可测试性**：`mock-model-server` 模拟不同延迟、故障、流式行为；决策引擎纯函数化可单测；支持故障注入测试容错链路。
- **部署**：Maven 多模块、Spring Boot 单 Jar、Docker 一键启动（gateway + mock）。

---

## 14. 非目标（Non-Goals）

- 不做多租户 SaaS 级用户体系（V2 的可选兑换码/充值默认裁剪）。
- 不做 Wasm/独立插件市场（插件以 Java SPI 实现，保持轻量）。
- 不追求 K8s 原生（P3 可提供 Helm 包，但架构上不绑定）。
- 不实现 MCP/A2A Agent 网关（列为远期方向，Portkey Agent Gateway 思路）。

---

## 15. 路线图

| 阶段 | 范围 | 状态 |
|---|---|---|
| **P0 MVP** | 统一 OpenAI API、模型注册、健康检查、简单打分路由、mock 模型服务 | 现有代码已覆盖 |
| **P1 核心完善** | 插件流水线框架、渠道/令牌/配额、双层限流、重试/超时/降级/熔断、Token 计量与成本、Admin API、配置热更新 | 待开发 |
| **P2 能力增强** | 语义缓存、安全护栏、管理后台看板、协议互转、金丝雀、会话亲和、OTel | 待开发 |
| **P3 分布式可选** | Redis 分布式限流/配额、K8s 部署、MCP/A2A 支持 | 远期 |

---

## 16. 与 v2.0 的差异摘要

| 改进点 | 借鉴来源 | 说明 |
|---|---|---|
| 插件化请求流水线 | Kong / Higress / LiteLLM | 横切能力解耦为插件，核心路由稳定 |
| 路由信号 Signals | LiteLLM Router Plugins | 插件间共享上下文，Scorer 可动态调权 |
| Channel 渠道模型 | New API | 渠道权重/优先级/启停/配额 |
| ApiToken 令牌与配额 | New API | 模型范围、IP 白名单、额度 |
| 双层治理（限流+配额） | Envoy AI Gateway | velocity 与 budget 分离 |
| 成本计量与预算闭环 | LiteLLM / Portkey / Higress | 单价表、流式增量计量、超额降级 |
| 降级链与成本降级 | Portkey / Higress | RoutingDecision 携带 fallbackChain |
| 条件路由/金丝雀 | Portkey | 策略类型扩展 |
| 可解释路由 | 自研亮点 | 决策打分明细可查询、可复盘 |
