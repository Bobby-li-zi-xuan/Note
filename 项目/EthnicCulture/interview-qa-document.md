# 民族文化数字档案管理平台 — 面试问答文档

**定位**: 中级后端开发
**侧重**: 架构设计、技术选型、并发控制、故障处理、性能优化
**数据支撑**: 性能测试报告（2026-05-11），410k 条测试数据

---

## 一、项目整体与架构设计

---

### Q1: 先介绍一下这个档案管理平台的整体架构，核心模块和职责划分是怎样的？为什么选择 Spring Boot + Redis + RabbitMQ 这个技术栈？

**结论**：这是一个基于 Spring Boot 的民族文化数字档案管理平台，核心分为认证鉴权、内容管理、缓存加速、异步削峰、操作审计五大模块。选择 Redis + RabbitMQ 是为了解决高并发计数和异步持久化的问题。

**整体架构**：

```
Client → Nginx (未来) → Spring Boot (8080, /api)
                          ├── AuthInterceptor (JWT 解析 + 鉴权)
                          ├── Controller 层 (API 入口)
                          ├── Service 层 (业务逻辑)
                          │    ├── 缓存: Redis (Redisson 锁 + 计数器)
                          │    ├── 数据库: MySQL (MyBatis Plus)
                          │    ├── 消息: RabbitMQ (异步刷盘)
                          │    └── 过滤: Guava BloomFilter (本地内存)
                          └── AOP 横切
                               ├── OperationLogAspect (操作日志)
                               └── GlobalExceptionHandler (统一异常)
```

**核心模块和职责**：

| 模块 | 职责 | 关键实现 |
|------|------|---------|
| 认证鉴权 | JWT 解析、@RequireLogin/@RequireRole 注解驱动鉴权 | AuthInterceptor + UserContextHolder (ThreadLocal) |
| 内容管理 | 民族文化 CRUD、分页搜索、字段筛选 | EthnicCultureService + MyBatis Plus Page |
| 缓存加速 | 三级缓存体系防穿透/击穿/雪崩 | CacheData 逻辑过期 + Redisson 分布式锁 + Guava BloomFilter |
| 异步削峰 | 点赞/收藏/浏览计数异步落库 | RabbitMQ (viewCount/likeCount/collectCount 三个队列) |
| 操作审计 | Controller 层全量日志记录 | AOP + Spring Event + 异步线程池落库 |

**为什么选 Spring Boot + Redis + RabbitMQ**：

Spring Boot 2.7.15：成熟生态，自动配置减少样板代码。项目启动只需配置 MySQL/Redis/RabbitMQ 连接信息即可运行。Java 11 是当时 LTS 版本，稳定优先。

Redis（通过 Redisson）：不只是缓存。Redisson 提供了分布式锁（看门狗自动续期）、RAtomicLong（原子计数器）、可重入锁——这些是纯 RedisTemplate 需要自己封装的能力。项目中的"逻辑过期 + 分布式锁"防击穿方案直接依赖 Redisson 的 `RLock.tryLock(waitTime, leaseTime)` API。

RabbitMQ（通过 Spring AMQP）：对比 Kafka，RabbitMQ 更适合"业务操作触发的单条消息异步处理"场景——队列数量多（3 个计数队列 + 3 个死信队列）、消息量中等（不是日志流那种百万/秒级别）、需要灵活的路由和死信机制。Kafka 的优势在超高吞吐和日志流处理，对于计数异步刷盘来说过重。

**追问预测**：

- *追问：如果重新选技术栈，你会怎么选？* → 保持 Spring Boot + Redis 不变。MQ 可以换成 RocketMQ（阿里系更广泛，支持事务消息、延迟消息），但 RabbitMQ 的管理界面和社区生态对中小项目够用。考虑升级 Spring Boot 3.x + Java 17/21（虚拟线程可能简化线程池管理）。
- *追问：为什么不用微服务架构？* → 当前项目的业务模块数量有限（用户、内容、日志），单体架构足够清晰。模块间没有需要独立扩缩容、独立技术栈、独立团队维护的诉求。过早拆分微服务会增加网络延迟、分布式事务、运维复杂度——得不偿失。
- *追问：缓存和 MQ 会不会让系统更复杂、更容易出问题？* → 会。每引入一个中间件就多一个故障点。但这是"复杂性换性能"的权衡——如果只用数据库，高并发下点赞操作的行锁竞争会让 TPS 降到几百，而 Redis + MQ 方案可以支撑 20k QPS。中间件的故障处理方案见第四部分。

**避坑提示**：介绍架构时不要只说"用了什么技术"，要说出"因为什么业务问题所以选了它"——面试官关注的是"为什么"，不是"是什么"。

---

### Q2: 项目支撑高并发访问和分布式数据一致性，你是怎么定义这个"高并发"场景的？比如预估的 QPS、读写比例、热点数据特征是怎样的？

**结论**：这个项目的"高并发"不是双十一级别的全站流量，而是特定场景下的局部高并发——热门档案的点赞计数、浏览计数、缓存击穿。从测试数据反推，系统当前的并发能力约 100 线程级别的短时突发。

**场景定义**：

**QPS 预估**：项目的峰值并发不是首页或列表页（那些可以被 CDN/缓存覆盖），而是热点档案详情页的点赞/浏览操作。假设一个热点内容短时间内 100 个用户同时点赞——这就是项目的"高并发"场景。测试中模拟了 100 线程并发原子自增计数器，实际 QPS 约 20,040/s，远超当前业务需要的几百 QPS。

**读写比例**：重度读多写少。浏览:点赞:收藏的比例大约是 200:5:1（浏览是大头，点赞是情感表达，收藏是深度交互）。浏览操作只需 Redis 原子 +1 + 投递 MQ，极轻量；点赞/收藏需要加分布式锁 + 写 user_favorite 表，相对重。

**热点数据特征**：二八定律——80% 的访问量集中在 TOP 3 热门档案。测试中用 `ORDER BY view_count DESC LIMIT 3` 预热热点数据，就是用浏览量排名识别热点。热点数据的生命周期较长（几天到几周），不会突然冒出一大批新热点——这使得布隆过滤器定时重建（每小时）和缓存预热（启动时 TOP 3）的策略有效。

**追问预测**：

- *追问：如果热点特征变了呢？比如突然所有数据都变成热点了？* → 所有数据都变热点意味着缓存策略会退化——逻辑过期 + 异步刷新会让所有 key 都在竞争分布式锁。此时需要降级：关闭逻辑过期检查，改为纯物理 TTL，同时用本地 Caffeine 缓存做 L1 缓冲。但这种情况在档案管理场景中几乎不会发生（访问分布天然长尾）。
- *追问：你实际做过 QPS 压测吗？* → 性能测试覆盖了组件级别的基准测试（数据层、缓存层、消息层），但没有做端到端的 HTTP 压测（如 JMeter 多线程模拟真实请求）。组件测试的结果可以估算系统能力——缓存命中 1.3ms + 业务逻辑 2-5ms，预估单机 QPS 在 200-500 级别（取决于缓存命中率和数据库负载）。
- *追问：如果要做全链路压测，你会怎么设计？* → 用 JMeter 或 wrk 模拟 500 并发用户，按 200:5:1 的比例执行浏览/点赞/收藏操作。预热缓存后启动压测，监控接口平均响应时间、P99 延迟、数据库连接池使用率、Redis 连接数、MQ 队列深度。瓶颈大概率在分页查询（当前 1.5s），需要先修复。

**避坑提示**：不要凭空编造 QPS 数字。如果没做过端到端压测，就说"组件级别有基准测试数据"，不要让面试官追问后发现是编的。

---

### Q3: 系统的权限体系设计中，三级角色（USER/EDITOR/ADMIN）的权限控制是如何实现的？注解级别的权限隔离具体怎么落地？有没有遇到过权限越界的场景？

**结论**：通过自定义注解 `@RequireLogin` 和 `@RequireRole` 在拦截器中实现声明式鉴权，支持方法和类级别的注解继承，无权限时抛 BusinessException 统一返回 401/403。

**完整实现流程**：

**步骤 1——JWT 携带角色信息**：登录成功后，JWT Token 的 payload 中包含 userId、role（0/1/2）、username。JWT 签发时设置 24 小时有效期。前端存储在 localStorage，每次请求带 `Authorization: Bearer <token>`。

**步骤 2——拦截器统一解析**：`AuthInterceptor.preHandle()` 是所有请求的入口。核心逻辑：

```
从 request attribute 检查是否已有缓存的 UserContext (避免重复解析)
  → 有 → 直接使用
  → 无 → 从 Authorization 头提取 Bearer Token
    → Token 不存在或格式错误 → 构建匿名 UserContext (authenticated=false)
    → Token 存在 → jjwt parseClaimsJws() 解析
      → 解析失败 (过期/签名错误) → 匿名 UserContext
      → 解析成功 → 提取 userId/role/username → 构建认证 UserContext
→ 绑定到 UserContextHolder (ThreadLocal)
```

**步骤 3——注解鉴权**：

```java
// 检查 @RequireLogin
RequireLogin requireLogin = method.getAnnotation(RequireLogin.class);
if (requireLogin == null) {
    requireLogin = handlerMethod.getBeanType().getAnnotation(RequireLogin.class);
}
if (requireLogin != null && requireLogin.value()) {
    if (!userContext.isAuthenticated()) → throw BusinessException(401);
}

// 检查 @RequireRole
RequireRole requireRole = method.getAnnotation(RequireRole.class);
if (requireRole == null) {
    requireRole = handlerMethod.getBeanType().getAnnotation(RequireRole.class);
}
if (requireRole != null) {
    if (!checkRolePermission(userContext.getRole(), requireRole)) → throw BusinessException(403);
}
```

注解可以放在方法上（精确控制单个接口），也可以放在类上（整个 Controller 的默认权限）。方法上的注解优先于类上的。

**步骤 4——统一异常响应**：`GlobalExceptionHandler` 捕获 `BusinessException(401)` 和 `BusinessException(403)`，统一封装为 `ResultDTO(code, message, null)` 返回。

**角色定义**：
- 0 = ROLE_USER：浏览、搜索、点赞、收藏、查看个人记录
- 1 = ROLE_EDITOR：内容编辑（可修改档案内容但不能审核发布）
- 2 = ROLE_ADMIN：审核发布、删除、管理用户、查看操作日志

`@RequireRole({2})` → 仅管理员可访问；`@RequireRole({1, 2})` → 编辑者和管理员可访问

**权限越界防护**：当前通过注解在 Controller 层做了接口级别的权限控制。但缺少数据级别的权限控制——比如编辑者不应该编辑别人创建但未发布的档案。目前通过业务逻辑中的 userId 校验来防止（编辑操作会检查创建者），但没有统一的 AOP 切面。

**追问预测**：

- *追问：如果用户直接改 JWT payload 里的 role 字段呢？* → JWT 的签名机制保证 payload 不可篡改。jjwt 的 `parseClaimsJws()` 会验证签名（HMAC-SHA256），如果 payload 被修改，签名不匹配，解析直接失败，返回匿名上下文。
- *追问：权限校验为什么在拦截器而不是 AOP 切面？* → 拦截器在请求映射到 Controller 方法之前执行，可以在请求到达 Controller 前就拦截掉，避免业务代码被执行。AOP 是在方法执行前后织入的，如果放在 AOP 中，无权限请求还是会进入 Controller 方法（虽然被切面挡住），从安全角度看更靠前更好。
- *追问：注解只能做"有没有权限"的二元判断，如果要求"只能编辑自己创建的内容"这种细粒度权限呢？* → 注解做粗粒度（接口级别），细粒度（数据级别）在 Service 层做。比如 `updateEthnicCulture` 的 Service 方法中检查 `culture.getCreatorId().equals(currentUserId)`。更复杂的可以用 Spring Security 的 `@PreAuthorize("hasPermission(#id, 'edit')")` ACL 机制，但当前项目规模不需要。

**避坑提示**：权限设计要能说清"粗粒度（接口级）和细粒度（数据级）的区别和处理方式"。只做了接口级权限不要假装做全了——诚实说明边界，并表达知道如何扩展。

---

## 二、高性能缓存架构

---

### Q4: 你提到用"逻辑过期 + Redisson 分布式锁"解决缓存击穿，能具体讲讲实现逻辑吗？逻辑过期时间是怎么设置的？怎么避免锁竞争导致的性能瓶颈？

**结论**：用 `CacheData<T>` 包装数据 + 逻辑过期时间字段，过期后返回旧数据并异步刷新，用 Redisson 分布式锁保证只有一个线程去查库重建。锁的 waitTime 设 3s，leaseTime 设 25-35s 随机，拿到锁后必须双重检查缓存。

**完整实现逻辑（6 步）**：

```
请求 → 查 Redis 缓存
  ├─ 缓存不存在 → 抢分布式锁 → 双重检查 → 查库 → 写缓存(带逻辑过期时间) → 返回
  └─ 缓存存在 → 检查 CacheData.expireTime
       ├─ 未过期 → 合并 Redis 计数器最新值 → 返回
       └─ 已过期 → 提交异步刷新任务到线程池 → 合并 Redis 计数器最新值 → 返回旧数据
```

**核心数据结构**：

`CacheData<T>` 包含两个字段：
- `data`：实际缓存的数据
- `expireTime`：逻辑过期时间戳（毫秒），由 `System.currentTimeMillis() + 随机过期时长` 生成
- `isExpired()`：判断 `System.currentTimeMillis() > expireTime`

**锁竞争的精细化处理——refreshCache() 方法的三层防护**：

第一层——缓存不存在时的锁竞争：用 `distributedLock.tryLock(lockKey, 3000, getLockExpiration())` 等待 3 秒。Redisson 内部自动自旋等待，不需要自己写 while 循环。拿到锁后必须"双重检查"——因为等待锁期间前面的线程可能已重建缓存。

第二层——锁获取失败后的兜底：没拿到锁，先再查一次缓存（可能被其他线程重建了），还是没有就降级直接查库（此时返回数据比什么都不返回更重要）。

第三层——finally 中安全释放锁：用 `boolean locked` 标志位，只有真正拿到锁的线程才 unlock。加上 `isHeldByCurrentThread()` 检查防误释放。

**逻辑过期时间设置**：

```
基准: 1 小时 (3600 秒)
实际: 55-65 分钟随机 (Math.random() * 600s)
```

为什么是 1 小时？档案类数据更新频率低（一天更新几次），1 小时的数据延迟可以接受。为什么加随机偏移？防止同一时刻大量缓存同时逻辑过期触发并发重建——即使逻辑过期返回旧数据的体验下降较小，大量并发重建也会增加数据库的瞬时压力。

**如何避免锁竞争性能瓶颈**：
- **waitTime 设 3 秒**：足够等前面的线程完成查库建缓存（查库 ≈ 2ms），但不会等太久阻塞请求
- **leaseTime 设 25-35s 随机**：可以防止极端情况下所有锁同时过期；正常查库操作 2ms 就完成，25s 绰绰有余
- **锁粒度细**：每一条记录一把锁（`lock:culture:{id}`），不同记录之间不互相阻塞
- **获取锁失败直接降级查库**：不等锁，保证请求最终有结果

**追问预测**：

- *追问：逻辑过期返回旧数据，用户看到的数据不是最新的怎么办？* → 返回旧数据之前调用了 `syncCountersToCulture()`，会用 Redis 中的实时计数值覆盖对象的 viewCount/likeCount/collectCount 字段。所以用户看到的浏览/点赞/收藏数是最新的——只是档案的正文内容可能延迟最多 1 小时。对档案管理场景，正文内容很少变化（几乎不变）。
- *追问：如果缓存一直是旧数据，MQ 有没有任务去更新缓存？* → 没有。缓存刷新只发生在"有人访问"时——没人访问的冷数据，缓存过期就过期了，不需要主动刷新。有人访问时逻辑过期触发异步刷新。这是一个"惰性刷新"策略，避免浪费资源刷新没人看的数据。
- *追问：双重检查为什么要做两次？第一次检查不够吗？* → 因为"缓存不存在" → "抢锁" → "拿到锁"之间有时间差。在这段等待时间里，前面拿到锁的线程可能已经重建了缓存。如果拿到锁后不再次检查，就会重复查库重建——这就是双重检查防止的"多线程重复查库"问题。实测 20 并发下只触发了 1 次查库，证明有效。

**避坑提示**：提到双重检查时，一定要说清楚"第二次检查检查什么"——检查的是缓存是否已被其他线程重建（cacheData != null && !cacheData.isExpired()），而不是简单地检查是否为 null。

---

### Q5: 双缓存布隆过滤器防止缓存穿透，两个过滤器分别部署在哪里？过滤器的误判率是怎么控制的？数据更新时怎么同步更新布隆过滤器？

**结论**：两个过滤器都在 JVM 本地内存中（Guava BloomFilter），通过 `AtomicBoolean` 实现双缓冲——currentFilter 用于查询，rebuildFilter 后台重建后原子替换。误判率构造时指定为 1%（fpp=0.01），实测 1.08% 接近理论值。新增数据同时 put 到两个过滤器。

**双过滤器部署位置**：

两个 Guava BloomFilter 都部署在应用进程的堆内存中，不是 Redis 里。每个应用实例各自维护自己的布隆过滤器。

```
BloomFilterUtils (Spring Bean, 单例)
├── currentFilter: BloomFilter<Long> (日常查询用)
└── rebuildFilter: volatile BloomFilter<Long> (后台重建用)
```

为什么两个？单布隆过滤器不能删除元素，新增数据后只能 put 进去——但如果数据量涨了，过滤器的 bit 数组大小不变，误判率会上升。需要定期从数据库全量重建过滤器以控制误判率。双缓冲让重建过程不阻塞查询——重建时用 rebuildFilter 加载新数据，完成后 `this.currentFilter = newFilter` 原子替换。

**误判率控制**：

构造时指定两个参数：
- `expectedInsertions = 100000`：预期元素数量。Guava 根据此参数和 fpp 计算 bit 数组大小。
- `fpp = 0.01`：期望误判率 1%

Guava 内部根据公式 `m = -n * ln(p) / (ln(2)^2)` 计算 bit 数组长度 m，根据 `k = m/n * ln(2)` 计算哈希函数个数 k。实测用 10,000 个不存在的 ID 测试，108 个误判（108/10000 = 1.08%），接近理论值。

**数据更新时的同步策略**：

新增档案：调用 `bloomFilter.add(cultureId)`，同时 put 到 currentFilter 和 rebuildFilter。保证新 ID 在两个过滤器中都存在，即使正在重建也不会漏。

更新档案：不涉及 ID 变化，不需要更新布隆过滤器。

删除/下架档案（软删除 deleted=1）：布隆过滤器**不会移除历史 ID**。因为 Guava BloomFilter 不支持删除。结果：被软删除的 ID 依然会被布隆过滤器判定为"可能存在" → 查询缓存（miss） → 查询数据库（deleted=1 查不到） → 返回 null。多了一次缓存查询 + 一次数据库查询，可以接受。

定时重建（每小时 `@Scheduled(cron = "0 0 * * * ?")`）：从数据库全量加载 `SELECT id FROM ethnic_culture WHERE status=1 AND deleted=0`，重建一个新的 BloomFilter，替换 currentFilter 和 rebuildFilter。实现：
```java
BloomFilter<Long> newFilter = BloomFilter.create(Funnels.longFunnel(), EXPECTED_INSERTIONS, FPP);
loadCultureIdsToFilter(newFilter);
this.rebuildFilter = newFilter;
this.currentFilter = newFilter;
filterVersion.incrementAndGet();
```
`AtomicBoolean rebuilding` 防并发重建，`AtomicLong filterVersion` 追踪版本。

**追问预测**：

- *追问：一小时重建一次，那这一小时内新增的数据如果被误判怎么办？* → 新增数据调了 `add(cultureId)` 同时 put 两个过滤器，新增数据在重建之前就已经在过滤器里了——不会误判。重建只是把"已被软删除的数据从过滤器中清掉"。
- *追问：如果应用重启了，布隆过滤器没有了怎么办？* → `CacheWarmUpRunner`（ApplicationRunner）在应用启动时调用 `bloomFilter.init()`，从数据库全量加载重构。初始化耗时 742ms（100k ID），在应用接受外部请求前完成。
- *追问：为什么不把布隆过滤器放在 Redis 里，让多个实例共享一份？* → 核心原因是延迟。Redis 查询 1.3ms，本地内存 <0.01ms，差了 100 倍。每次查询都先走布隆过滤器，累积起来的延迟差异很大。次要原因是 Guava BloomFilter 100k ID 只占约 120KB 内存，每个实例各自维护完全可以接受。代价是多实例各自定时重建会各自查一次数据库——当前实例数少，可以接受。

**避坑提示**：要能说清"布隆过滤器不能删除元素"这个核心限制，以及双缓冲是如何绕过这个限制的——不是删除单个元素，而是全量重建后替换整个过滤器。

---

### Q6: 热点数据平滑更新的一致性保障是怎么做的？缓存和数据库的更新顺序、失败回滚策略是什么？怎么避免出现缓存和数据库的数据不一致问题？

**结论**：写操作采用 Cache Aside 模式——先更新数据库，再删除缓存。读操作在缓存未命中时查库并重建缓存。一致性通过"顺序设计 + MQ 异步刷盘兜底 + Redis 计数器实时覆盖"三层保障。

**更新顺序——为什么先更库再删缓存而不是反过来？**

假设先删缓存再更库：
```
线程A: 删缓存 → (窗口期) ← 线程B: 查缓存 miss → 查库(读到旧值) → 写缓存(旧值) → 线程A: 更库(新值)
```
结果：缓存是旧值，数据库是新值——缓存脏数据。

先更库再删缓存：
```
线程A: 更库(新值) → 删缓存 → 线程B: 查缓存 miss → 查库(读新值) → 写缓存(新值)
```
唯一风险：队列删除失败。如果更库成功但删缓存失败，缓存中是旧数据。但：
1. 逻辑过期（最长 65 分钟）后缓存被重新刷新，数据会自动恢复一致
2. 编辑操作频率极低（一天几次），删除失败的窗口极小

**失败回滚策略**：

更新数据库失败（抛异常） → 事务回滚 → 缓存不受影响 → 用户看到旧数据（正确行为）

更新数据库成功 → 删除缓存失败：下次读取时读到旧缓存。此时有两个兜底：
1. 逻辑过期时间（最长 65 分钟）后自动重建
2. 如果该档案的缓存不存在（之前未访问过），下次访问时直接查库读到最新值

没有 TCC 或分布式事务——对于档案管理场景，短暂的数据不一致（最长 1 小时）可以接受。

**读操作的一致性保障——syncCountersToCulture()**：

每次读取缓存后，不管缓存是否逻辑过期，都会调用此方法：
```java
// 用 Redis 中的实时计数值覆盖数据库返回的旧计数值
if (redissonCounter.exists(viewCountKey)) {
    culture.setViewCount(redissonCounter.get(viewCountKey));
} else {
    // 计数器不存在时用数据库值初始化
    redissonCounter.set(viewCountKey, culture.getViewCount(), ttl, TimeUnit.SECONDS);
}
```

这意味着用户永远能看到最新的浏览量/点赞数/收藏数——这三个计数字段由 Redis 计数器保证实时一致性。正文内容最多有 1 小时的延迟。

**最终一致性——MQ 异步刷盘兜底**：

Redis 计数器的增量最终通过 MQ 消费写入数据库。即使某次缓存刷新漏了某条记录，MQ 消费侧的 `UPDATE ethnic_culture SET view_count = ? WHERE id = ?` 会保证数据库最终与 Redis 一致。

**追问预测**：

- *追问：有没有考虑过用 Canal 订阅 binlog 来做最终一致？* → 考虑过但没有实现。Canal 可以监听数据库变更并刷新缓存，但增加运维复杂度（Canal Server + ZooKeeper）。当前 Cache Aside + 逻辑过期 + MQ 刷盘的组合对当前规模足够。
- *追问：如果数据库更新成功、删缓存成功、但此时有大量请求查了数据库并重建了缓存（旧值），怎么办？* → 这种情况不存在——因为数据库已经更新为新值了，重建缓存时读到的是新值。真正危险的是上面分析的"先删缓存再更库"的顺序——那才会导致读到旧值。
- *追问：如果缓存删了，但后面重建缓存时数据库连接超时读不到数据怎么办？* → 缓存在 refreshCache 方法中有三层降级：查缓存 → 抢锁查库 → 拿不到锁降级查库。如果数据库也读不到，会抛异常返回 500。没有"返回旧数据"的降级——因为缓存已经被删了，没有旧数据可用。可以考虑在删缓存时不直接 DELETE 而是标记为过期（保留旧值），但项目中未实现。

**避坑提示**：一致性讨论要坦诚。不要承诺"强一致性"——Cache Aside 本身就做不到强一致。要能清楚说出"不一致的窗口有多长、发生在什么条件下、后果是什么"，这比吹"我们做了一致"更能体现技术成熟度。

---

### Q7: 项目中 Redis 的部署方式是怎样的？单机、主从还是集群？缓存的过期策略、淘汰策略是怎么配置的？怎么处理大 Key、热 Key 问题？

**结论**：当前是单机 Redis 部署（开发和测试环境），缓存使用逻辑过期而非物理过期，因此未配置 Redis maxmemory-policy 淘汰策略。热 Key 通过 CacheData 逻辑过期 + 分布式锁处理，大 Key 未出现（缓存的是单条档案记录，大小可控）。

**部署方式**：

当前配置：`redis://172.22.96.1:6379`，单机模式，通过 Redisson `Config.useSingleServer()` 连接。密码认证 + database=0。这是开发和测试环境的配置，生产环境需要升级为 Redis Sentinel 或 Cluster。

单节点风险：Redis 挂了 → 缓存全 miss → 分布式锁不可用 → 计数器不可用。见第四部分故障场景。

**过期策略——逻辑过期替代物理过期**：

项目**故意不设 Redis 物理 TTL**。所有缓存 key 都持久存储在 Redis 中，过期判断在应用层通过 `CacheData.isExpired()` 完成。

为什么不用 Redis 的 TTL 机制？
- Redis TTL 到期后 key 被删除，缓存 miss 请求直接压库
- 逻辑过期后 key 还在，返回旧数据 + 异步刷新——永远有数据挡在数据库前面
- 计数器有 TTL（30min 随机）——因为计数器过期后可以重建（从数据库读取重置），不影响业务

**淘汰策略**：

因为不使用物理 TTL，Redis 的 maxmemory-policy 不是核心配置。默认 `noeviction`——内存满时拒绝写入。如果未来缓存数据量增长超过 Redis 内存，需要评估：
1. 限制缓存数量（只缓存 TOP N 热点数据，其余直接查库）
2. 使用 `allkeys-lru` 淘汰冷数据（但需改为物理 TTL，与逻辑过期方案冲突）
3. Redis Cluster 分片扩容

**热 Key 处理**：

热点数据的定义：浏览量 TOP 3 的档案，在启动时通过 `CacheWarmUpRunner` 预热加载到缓存。

热 Key 的读取压力：逻辑过期 + 异步刷新——任意数量的读请求都只命中 Redis 缓存（1.3ms），不会打数据库。热点数据更新时的竞争：分布式锁——只有第一个线程查库重建，其余返回旧数据。锁的粒度按 recordId 划分——不同热点记录之间不竞争。

**大 Key 处理**：

当前未出现大 Key 问题。缓存的 key（`culture:detail:{id}`）值是单个 EthnicCulture 实体序列化后的 JSON，包含 name/region/summary/content/images 等字段。images 字段存储图片路径而非图片本身，content 字段是文本内容（不会超过几十 KB）。如果 content 字段包含长文，可考虑不缓存 content 字段，只缓存摘要和元数据。

**追问预测**：

- *追问：生产级别的 Redis 部署你会怎么设计？* → Sentinel 模式（一主二从三哨兵），主节点负责读写，从节点做只读副本和故障转移。Sentinel 监控主节点健康状态，故障时自动选举新主。从节点可以分担部分读压力（比如缓存查询可以读从节点，计数器必须走主节点保证原子性）。
- *追问：逻辑过期不设 TTL，Redis 内存会不会被打爆？* → 理论上会——如果 never delete，缓存数据会越来越多。但实际上档案数据量有限（10 万级），每条缓存几 KB，总缓存内存不会超过几百 MB。如果数据量持续增长，可以加一个兜底——当缓存 key 数量超过阈值时，驱逐访问频率最低的 key。当前未实现但已知优化方向。
- *追问：你怎么判断一个 key 是热 Key？* → 当前靠业务经验（按 view_count 排序）。生产环境可以用 Redis 的 `redis-cli --hotkeys` 命令分析内存中的热 Key，或者在应用层对缓存访问加计数器，统计每个 key 的访问频率。

**避坑提示**：坦诚说明当前是单机 Redis 配置，并清楚说出生产化需要做什么（Sentinel/Cluster）。面试官更看重你"知道当前状态的局限性"而不是"已经完美了"。

---

## 三、分布式并发控制

---

### Q8: Redisson 看门狗自动续期的实现原理是什么？你在项目中有没有手动调整过看门狗的超时时间？怎么避免锁的误释放和死锁问题？

**结论**：看门狗是 Redisson 在 leaseTime=-1 时启用的后台定时任务（每 10 秒续期 30 秒），通过 Netty 的 Timer 实现，只有业务线程还活着才会续期。项目中显式传入 leaseTime（25-35s 随机），此时看门狗不启用，锁到期强制释放。

**看门狗实现原理**：

Redisson 的 `RLock.lock()` 方法（leaseTime=-1 时）：
1. 通过 Lua 脚本在 Redis 中设置一个 Hash key `lockKey → (threadId → 1)`
2. 同时启动一个后台定时任务（`nettyTimer.newTimeout`），每 `internalLockLeaseTime/3`（默认 10 秒）执行一次
3. 定时任务执行 Lua 脚本 `HEXISTS lockKey threadId` → 如果存在（锁还在且属于当前线程），则 `PEXPIRE lockKey internalLockLeaseTime`（续期为 30 秒）
4. 手动调 `unlock()` 时，调用 `cancelTimeout()` 取消定时任务，然后 Lua 脚本删除 key
5. 如果业务进程崩溃，Netty Timer 停止，30 秒后锁自动过期释放

**项目中手动指定 leaseTime 的考量**：

项目封装了 `RedissonDistributedLock.tryLock(lockKey, waitTime, leaseTime)`：
```
waitTime = 3000ms   (等待 3 秒拿不到锁就放弃)
leaseTime = 25000-35000ms 随机  (锁最多持有 25-35 秒)
```

为什么不用看门狗而显式指定 leaseTime？租赁时间设 25-35 秒包住了最坏情况（查库 2ms + 建缓存 1ms + GC 暂停），超时后锁强制释放防止死锁。看门狗的自动续期适合"业务处理时间不确定"的场景，缓存重建这种确定在几 ms 内完成的操作，给个上限更安全。

**防误释放**：

`unlock()` 方法中检查 `lock.isHeldByCurrentThread()`——只有当前线程持有的锁才释放。防止"线程 A 的锁过期被线程 B 拿到，线程 A 执行完去 unlock 却误删了线程 B 的锁"。

**防死锁**：

`tryLock()` 有超时时间（3 秒），拿不到就返回 false，不让线程无限等待。`finally` 块中保证一定会 unlock（用 boolean locked 标志位区分）。leaseTime 强制 25-35 秒后释放，即使代码走不到 finally（如进程 crash），锁也不会永久残留。

**追问预测**：

- *追问：看门狗续期期间 Redis 主节点挂了怎么办？* → Redisson 在 Redis 主从切换时会出现锁信息丢失的问题——锁数据在主节点上，主节点 crash 后从节点提升为新主，但锁数据可能还没同步过来，导致两个客户端同时持有同一把锁。这是 Redis 分布式锁的经典缺陷（Redlock 算法试图解决但争议很大）。对于当前的计数保护场景，即使出现双锁问题，最坏情况是多了一次数据库查询——不会造成数据错误（最终一致性兜底）。
- *追问：leaseTime 设 25-35 秒会不会太长了？* → 正常查库重建缓存只用 2-3ms，25 秒确实长。但这是"悲观的保护"——考虑极端 GC 停顿（Full GC 可能几秒）、网络抖动、数据库慢查询。设短了（比如 3 秒），一旦 GC 停顿超过 3 秒，锁就释放了，其他线程也能拿到锁，可能双重查库。既然正常操作 2ms 就能完成并主动 unlock，锁根本不会等到 25 秒的 leaseTime 才释放——leaseTime 只是一个保险。
- *追问：Redisson 的可重入锁是怎么实现的？* → 基于 Redis Hash 结构。`lockKey → { threadId: 重入次数 }`。同一个线程第一次 lock 时 `HSET lockKey threadId 1`，再次 lock 时 `HINCRBY lockKey threadId 1`，unlock 时 `HINCRBY lockKey threadId -1`，减到 0 时 `DEL lockKey`。可重入性在点赞/收藏操作中不是必需的（不会嵌套调用），但工具类封装了以备未来使用。

**避坑提示**：要能说清 lock() 和 tryLock() 的区别——lock() 拿不到锁会一直等待，可能导致请求堆积；tryLock(waitTime) 有超时保护。项目中用 tryLock 更安全。

---

### Q9: 原子计数器支撑高并发点赞/收藏计数，为什么不用 Redis 的 INCR 命令直接实现？原子计数器和 Redis 命令相比，在并发场景下的优势和劣势是什么？

**结论**：实际用的就是 Redis 的 INCR 命令——Redisson 的 RAtomicLong 底层调的是 Redis INCRBY。封装成 RAtomicLong 带来的优势是：统一 API、TTL 原子设置、连接池复用。

**澄清——RAtomicLong 就是 Redis INCR 的封装**：

`redissonCounter.increment(key)` 底层走的是 `RAtomicLong.incrementAndGet()`，对应 Redis 命令是 `INCRBY key 1`。不是"不用 Redis INCR"，而是"用 Redisson API 调用 Redis INCR"。封装的价值在于：

**优势**：

1. **原子性保证**：`INCRBY` 本身就是原子操作，Redis 单线程执行命令模型保证不会有并发累加问题。这点封装前后没区别。

2. **配套操作原子化**：`set(key, value, ttl, TimeUnit)` 方法同时做了 SET + EXPIRE 两个操作。相比用 RedisTemplate 需要调 `opsForValue().set(key, value)` + `expire(key, ttl, unit)` 两步（不原子——如果 SET 成功 EXPIRE 失败，key 会永久存在），Redisson 的 RAtomicLong 在 set 后立即 expire，减少两操作的时间窗口。

3. **连接池复用**：Redisson 底层 Netty 连接池，异步非阻塞通信。在 100 线程并发自增测试中 QPS 达到 20,040/s，接近单 Redis 节点的吞吐上限。

4. **API 统一**：`increment` / `decrement` / `get` / `exists` / `delete` 统一封装，业务代码不用直接操作 Redis 命令字符串。

**劣势（对比本地计数器）**：

| 对比维度 | Redis RAtomicLong | 本地 synchronized + AtomicLong |
|---------|-------------------|------|
| QPS (100 并发) | 20,040/s | 2,546,931/s |
| 延迟 | ~1.3ms（网络 IO） | ~0.001ms（内存操作） |
| 多实例一致性 | 天然一致 | 不一致（各自计数） |
| 持久性 | 取决于 Redis 持久化配置 | 进程重启丢失 |

核心劣势：Redis 网络 IO 的延迟比本地内存高 1000 倍。但这是分布式一致性的必要代价——多实例部署时必须用 Redis 保证计数器全局一致。

**追问预测**：

- *追问：有没有考虑过用 Redis Lua 脚本做更复杂的原子操作？* → 当前场景（自增 + 设 TTL）不需要 Lua。但如果需要"检查是否存在 → 不存在则初始化 → 自增"这种多步原子操作，就需要 Lua 了。项目中 `syncCountersToCulture` 用了先 `exists` 再 `get` 或 `set`——这两步不是原子的，但即使有竞争也是"谁最后 set 谁生效"，不影响正确性。
- *追问：如果计数器值溢出（超过 Long 最大值）怎么办？* → RAtomicLong 底层是 Redis 字符串存数字，Redis 的 INCRBY 对超过 64 位有符号整数范围会返回错误。但浏览量/点赞数/收藏数不可能达到 9.22 × 10^18——这是天文数字。不用考虑溢出。
- *追问：MQ 消费时如果计数器已被删除（TTL 过期），怎么处理？* → 消费者读到计数器不存在 → 跳过本次更新。计数器过期说明长时间没人访问这条数据，增量为 0，跳过是正确的。

**避坑提示**：不要说"原子计数器比 Redis 命令快"——本质就是 Redis 命令。要说清"封装带来的工程优势（TTL 原子设置、API 统一）"和"分布式 vs 本地的性能取舍"。

---

### Q10: 雪花算法生成全局唯一 ID，你是怎么解决时钟回拨问题的？分库分表场景下，雪花 ID 的分片键设计是怎样的？有没有做过 ID 的性能压测？

**结论**：时钟回拨 < 5 秒时等待追上，> 5 秒时抛异常拒绝服务。分库分表场景下使用雪花 ID 的时间戳部分做分片键（取模或 Range 分区）——时间天然递增，写入均匀分布。未做过 ID 的独立性能压测，但理论 QPS 为 4096/ms/节点。

**时钟回拨的完整解决方案**：

```java
public synchronized long nextId() {
    long timestamp = currentTimeMillis();
    
    if (timestamp < lastTimestamp) {  
        long offset = lastTimestamp - timestamp;
        if (offset <= 5000) {  // 回拨 5 秒以内
            Thread.sleep(offset);  // 等待时间追上
            timestamp = currentTimeMillis();
        } else {  // 回拨超过 5 秒
            throw new RuntimeException("Clock moved backwards. Refusing to generate id");
        }
    }
    // ... 正常生成逻辑
}
```

两层处理：
- 回拨 ≤ 5 秒：小回拨一般是 NTP 校时造成的，等待 offset 毫秒让时钟自然追上。
- 回拨 > 5 秒：大回拨大概率是人为修改系统时间或 NTP 严重偏差，此时等待时间太长，直接抛异常拒绝服务。宁可停止 ID 生成也不生成可能重复的 ID。

为什么不用其他方案？备选的有：
- 缓存历史序列号：回拨期间不依赖时间戳，只用序列号递增。But 需要记录最近 N 毫秒的序列号，回拨超过 N 毫秒无效。
- 借用未来时间：回拨期间继续用 lastTimestamp，但序列号递增更快可能耗尽。与"等待追上"思路类似但不够安全。

当前方案简单直接，适合中小项目。

**分库分表场景下的分片键设计**：

雪花 ID 的结构：`[1bit符号] [41bit时间戳] [10bit机器ID] [12bit序列号]`

分库分表策略选择：
1. **按时间戳部分取模**：`id >> 22` 提取时间戳 → `% 分表数`。优点：时间递增，写入均匀分散到各表。缺点：查询需要带时间范围条件。
2. **按 ID 整体取模**：`id % 分表数`。优点：实现简单。缺点：雪花 ID 时间戳在高位，低位的序列号近似随机，取模后分布均匀。
3. **Range 分区**：按时间戳范围分段，如 `2025-01` 的数据在一张表、`2025-02` 的在另一张。优点：查询优化简单（按时间查自然落到对应表）。当前推荐此方案——档案管理场景，时间天然是查询维度。

**ID 性能压测**：

未做过独立压测。理论值：雪花算法单机单毫秒 4096 个 ID（12 位序列号），即 QPS 约 4,096,000/s。实际瓶颈不在 ID 生成本身，而在 `synchronized` 关键字的锁竞争——毫秒内序列号用尽需要 `waitUntilNextMillis` 自旋等待。多线程高并发下，`synchronized nextId()` 成为瓶颈，但远在业务瓶颈（数据库查询 2ms+）之前。

**追问预测**：

- *追问：如果超过 1024 个服务节点怎么办？* → 可以压缩序列号位数扩展 workerId 位数。比如 workerId 从 10 位扩展到 12 位（支持 4096 节点），序列号从 12 位压缩到 10 位（每毫秒 1024 个）。或者引入数据中心 ID（datacenterId），标准的雪花算法就该有 5 bit datacenter + 5 bit worker 两层，当前实现简化了。
- *追问：有没有考虑过美团 Leaf 或百度 UidGenerator 这些现成的方案？* → 考虑过。Leaf 的 segment 模式解决"每次获取 ID 需要网络 IO"的问题（批量预取），适合超高并发场景。当前项目不需要——单机雪花算法够用，且零外部依赖。
- *追问：synchronized 关键字会不会成为瓶颈？* → 理论上 4096 个/ms，即 400 万/秒。实际 synchronized 竞争的 QPS 远高于此。profile 过才知道。如果真成瓶颈，可以每线程维护独立的 sequence 缓冲区（类似 Netty 的 object pool），但当前没必要。

**避坑提示**：时钟回拨是雪花算法的核心面试考点。要能说清"回拨多少秒等待、多少秒抛异常"这个阈值是怎么定的——5 秒是经验值，NTP 正常校时在毫秒级，超过 5 秒说明系统时间出了问题。

---

### Q11: 分布式场景下，多节点同时修改同一条档案数据时，除了分布式锁，还有哪些并发控制手段？比如乐观锁、版本号控制，你在项目中有没有使用？

**结论**：当前项目主要依赖 Redisson 分布式锁（悲观锁），没有使用乐观锁。但如果要补充，首选 MyBatis Plus 自带的 `@Version` 乐观锁注解，适合"读多写少、冲突概率低"的场景。核心区别：悲观锁在读取前加锁，乐观锁在更新时校验。

**各种并发控制手段对比**：

| 手段 | 原理 | 适用场景 | 优缺点 |
|------|------|---------|--------|
| 悲观锁（分布式锁） | 修改前先获取锁 | 写冲突高、单次操作重的场景 | 阻塞等待，有超时风险 |
| 乐观锁（版本号） | UPDATE SET x=x+1, version=version+1 WHERE version=? | 写冲突低、单次操作轻的场景 | 不阻塞，但冲突时需要重试 |
| 数据库悲观锁 | SELECT ... FOR UPDATE | 必须强一致、分布式锁不可用时 | 依赖数据库，连接持有时间长 |
| CAS（Compare And Swap） | Redis WATCH + MULTI/EXEC | 简单值修改（如余额扣减） | 乐观锁的 Redis 版本 |

**项目中哪些场景适合乐观锁**：

**点赞/收藏操作**：当前用分布式锁防止重复点赞（写 user_favorite 表）。改为乐观锁方案——在 user_favorite 表加 `version` 字段，插入时检查唯一约束（user_id + culture_id + type），如果重复插入 MySQL 报 DuplicateEntryException，捕获后提示"已点赞"。这样完全不用分布式锁——数据库唯一索引天然保证不会重复点赞。

**编辑档案**：当前用 Cache Aside（更库 → 删缓存），没有并发控制。如果两个管理员同时编辑同一条档案，后提交的会覆盖前面的。用 MyBatis Plus `@Version` 注解：
1. 在 ethnic_culture 表加 `version` 字段（Integer）
2. 实体类对应字段加 `@Version` 注解
3. MyBatis Plus 自动在 UPDATE 时的 SQL 中加 `WHERE version = ?`，并自动 version+1
4. 如果 version 不匹配（被其他请求先更新了），返回 0 行受影响 → 提示用户"数据已被修改，请刷新后重试"

**追问预测**：

- *追问：乐观锁和悲观锁你怎么选？* → 看两个维度：冲突概率 × 操作成本。点赞操作——冲突概率低（用户不会同时点两个赞）、操作成本低（插入一条记录）→ 用数据库唯一索引替代锁（轻量级乐观）。编辑操作——冲突概率中等、操作成本中等 → 用 `@Version` 乐观锁，冲突时提示用户。缓存重建——冲突概率高（缓存过期瞬间 20 个请求同时查库）、操作成本高（数据库查询）→ 用分布式悲观锁，保证只有一个查库。总结：高冲突 + 重操作 = 悲观锁，低冲突 + 轻操作 = 乐观锁。
- *追问：如果乐观锁冲突后自动重试呢？* → 可以。在 Service 层捕获 version 不匹配异常，自动重新读取最新数据、合并修改、重试提交。重试 3 次为止。但合并策略要小心——如果用户修改了字段 A，另一个用户修改了字段 B，两次修改不冲突，可以自动合并。如果两人都修改了字段 A，需要让后提交的用户决定怎么合并（人工介入）。
- *追问：分布式锁和数据库悲观锁什么时候选哪个？* → 分布式锁比数据库悲观锁快（Redis 1.3ms vs 数据库锁定等待时间不确定），且不占用数据库连接。只有当 Redis 不可用时才降级为数据库悲观锁。

**避坑提示**：当面试官问"你用了乐观锁吗"而你没用时，不要说"没想到"——要说"分析过，当前场景用分布式锁/唯一索引更合适，但加了 version 字段作为后续扩展的准备"。

---

## 四、异步削峰填谷设计

---

### Q12: RabbitMQ 异步落库高频操作，为什么选择 RabbitMQ 而不是 Kafka？消息的生产、消费模型是怎样的？

**结论**：RabbitMQ 更适合"多队列、低延迟、复杂路由、中小吞吐"的业务消息场景。Kafka 更适合"单主题、高吞吐、日志流"的大数据场景。项目中计数异步刷盘是典型的业务消息——队列多（3 个业务队列 + 3 个死信队列）、消息量中等（每操作一条消息）、需要灵活的死信路由。

**RabbitMQ vs Kafka 详细对比**：

| 维度 | RabbitMQ | Kafka | 项目场景 |
|------|---------|-------|---------|
| 消息模型 | 消息投递到队列，消费后删除 | 消息持久化到分区，消费者维护 offset | 计数刷盘后消息不需要保留 |
| 吞吐量 | 万级 msg/s（单队列） | 百万级 msg/s | 最多几千 msg/s |
| 延迟 | 微秒~毫秒级 | 毫秒级（批量推送） | 不敏感到微秒但需要低延迟 |
| 路由能力 | Exchange/Queue/Binding 灵活路由 | 简单（发到 Topic） | 需要 DLX 死信路由 |
| 消息优先级 | 支持 | 不支持 | 不需要 |
| 消费模式 | Push + Pull（basicGet） | Pull（poll） | 手动 Pull（手动 ACK 控制） |
| 运维复杂度 | 低（Erlang 自带管理界面） | 高（需 ZooKeeper/Kraft） | 开发环境友好 |
| 消息回溯 | 不支持 | 支持（基于 offset 重放） | 不需要 |

**选择 RabbitMQ 的核心理由**：
1. 死信队列 DLX：消费失败超过重试次数 → 自动路由到 DLX → DLQ。Kafka 的死信需要自己实现 consumer 逻辑。
2. 手动 ACK 精细控制：basicGet(autoAck=false) → 处理成功调 ack() → 失败调 retry 或 nack。Kafka 的 offset commit 是批量累积的，单条消息粒度的确认不如 RabbitMQ 直接。
3. 开发成本低：Spring AMQP 集成简单，管理界面 (15672) 直接可视化队列深度、消费速率、死信情况。

**消息生产模型**：

生产端采用的是推模式（Push），通过 `RabbitTemplate.convertAndSend()` 异步投递到指定队列。

```
业务操作 (点赞/收藏/浏览)
  → Redis 计数 +1
  → rabbitMessageQueue.sendMessage("viewCount", cultureId.toString())
    → rabbitTemplate.convertAndSend("viewCount", message)
      → RabbitMQ Broker 接收并存储到 viewCount 队列
```

投递时带有 UUID 生成的 CorrelationData，启用 Publisher Confirm 模式，Broker 收到消息后异步确认。

生产端消息为字符串格式，极简内容——只包含 cultureId（因为消费者可以从 Redis 读取最新计数值，不需要在消息中携带计数）。

**消息消费模型**：

采用的是拉模式（Pull），通过 `Channel.basicGet()` 主动拉取消息（非推送推送），这样可以保证一次只取一条，配合手动 ACK / 重试机制。

消费端：
```
MessageConsumeTask 拉取消息 (basicGet)
  → 解析消息中的 cultureId
  → 从 Redis 读取最新计数
  → UPDATE 数据库
  → 成功: basicAck(deliveryTag) → 消息从队列删除
  → 失败: retryOrDeadLetter()
    → 重试次数 < 3: 重新投递到原队列 (x-retry-count+1)
    → 重试次数 ≥ 3: 投递到 DLX → DLQ
```

消费者线程池 `messageConsumerExecutor` 专门处理消息消费，核心线程数 = max(3, CPU/2)，防止消费阻塞其他异步任务。

**追问预测**：

- *追问：Kafka 的高吞吐是怎么做到的？为什么不用？* → 顺序写磁盘（追加写，Page Cache）+ 零拷贝（sendfile）+ 批量压缩 + 分区并行。这些设计是为日志流、埋点流这类每秒百万条的场景优化的。项目中每秒最高几十条计数消息，Kafka 的优势根本用不上，反而引入 ZooKeeper 的运维成本。
- *追问：如果用 RabbitMQ 的话，消息积压了怎么办？* → 当前三个队列是直接（Direct）模式而非 Fanout，不会因为消费者离线导致消息"丢了"——消息一直在队列中持久化存储。如果消费速度跟不上生产速度，队列深度持续增长 → 监控告警 → 加大消费者线程池或增加消费者实例。
- *追问：消息的生产和消费是一一对应的吗？会不会一条消息被消费多次？* → 正常情况下一条消息被消费一次（ACK 后删除）。但异常情况下可能重复消费——消费者处理成功但 ACK 时网络故障，Broker 没收到 ACK，消息重新投递。消费者可能重复消费同一条数据。幂等性保证：UPDATE 操作本身就是幂等的（SET view_count = Redis 最新值），重复执行结果相同。

**避坑提示**：技术选型问题要说"场景匹配"，不要说"哪个更好"。RabbitMQ 和 Kafka 没有绝对的好坏——场景不同选择不同。能说出"在什么场景下会选 Kafka"比"我们的选择就是对的"更高级。

---

### Q13: 自定义线程池参数配合手动 ACK + 死信队列有限重试，你是怎么设置线程池的核心参数？怎么根据业务场景调整？

**结论**：两个线程池按业务特征分别配置——asyncTaskExecutor 是 IO 密集型（核心线程 CPU×2），messageConsumerExecutor 是稳定消费型（核心线程 max(3, CPU/2)）。有界队列防止 OOM，不同拒绝策略对应不同业务语义。

**asyncTaskExecutor（通用异步线程池）参数和设置依据**：

```
核心线程数: CPU核心数 × 2 (IO密集型: 线程多 = 并发IO等待多 = 吞吐高)
最大线程数: CPU核心数 × 2 (与核心相同，控制资源上限)
队列容量: MAX_POOL_SIZE × 10 (有界队列，防OOM。队列大小 = 缓冲能力)
空闲存活: 60s (超过核心线程数的线程空闲超时回收)
拒绝策略: CallerRunsPolicy (交给调用方线程执行，保证任务不丢失)
线程前缀: "async-task-"
```

为什么是 CPU×2？（IO 密集型的经验公式）IO 密集型任务的特征是线程大部分时间在等网络/磁盘响应（阻塞），CPU 利用率低。多开线程让更多 IO 请求同时进行，提高系统吞吐。公式：`CPU核心数 × (1 + IO等待时间/CPU计算时间)`。对于缓存刷新、数据库写入这类 IO 密集型操作，IO 等待时间远大于 CPU 计算时间，系数取 2-4 合理。项目取 2 是保守估计。

**messageConsumerExecutor（MQ 消费线程池）参数和设置依据**：

```
核心线程数: max(3, CPU核心数/2)
最大线程数: 核心线程数 × 2
队列容量: MAX_POOL_SIZE × 10
空闲存活: 60s
拒绝策略: AbortPolicy (抛异常，由调用方执行 nack+requeue)
线程前缀: "message-consumer-"
```

为什么比 asyncTaskExecutor 少？MQ 消费是稳定速率的工作流（生产者均匀投递），不需要大量线程应对突发。少量线程持续消费即可。队列中排队的消息由 RabbitMQ 本身的队列机制缓冲——不需要在 Java 层再建大型队列。

**AbortPolicy 的设计意图**：当线程池满、队列满时，AbortPolicy 抛出 `RejectedExecutionException`。`MessageConsumeTask.rejectAndRequeue()` 捕获此异常，将消息通过 retryOrDeadLetter 处理——这等于把"拒绝"当作"消费失败"来处理，消息不会丢失（要么重新投递，要么进 DLQ）。这是与 CallerRunsPolicy 的关键区别——MQ 消息不应该阻塞消费者主循环，拒绝后交给 RabbitMQ 负责重投。

**参数调整依据**：

| 场景 | 调整方式 |
|------|---------|
| 突发流量高（热点事件爆发大量点赞） | 增大 asyncTaskExecutor 的最大线程数 |
| MQ 消息积压严重 | 增大 messageConsumerExecutor 核心线程数或增加消费者实例 |
| 频繁 OOM | 减小队列容量或加大 JVM 堆内存 |
| 线程池拒绝频率高 | 增大队列容量或最大线程数（先分析是 CPU 瓶颈还是 IO 瓶颈） |

**追问预测**：

- *追问：怎么知道线程池设置得对不对？* → 监控回答：看线程池指标——活跃线程数 vs 核心线程数（接近核心说明需要扩容）、队列使用率（>80% 需要更大的队列或更多线程）、拒绝次数（>0 必须关注）、任务平均等待时间。没有监控的话，只能靠压测模拟各种负载，观察响应时间和错误率。
- *追问：如果用 Java 21 的虚拟线程，还需要这些线程池吗？* → 虚拟线程能简化 asyncTaskExecutor 的设计——每个任务一个虚拟线程，不需要操心核心/最大线程数、队列容量、拒绝策略。本质是用 JVM 的调度器替代手写线程池。但 messageConsumerExecutor 可能还需要保留——MQ 消费需要控制并发度（不能无限拉消息）。这是一个"降本增效"的演进方向。
- *追问：CallerRunsPolicy 会不会导致 Tomcat 线程被阻塞？* → 会。缓存刷新任务提交到线程池被拒绝时，由调用方（可能是 Tomcat 请求线程）自己执行。Tomcat 线程被阻塞意味着这个线程不能处理新请求——极端情况下所有 Tomcat 线程都被用来跑缓存刷新，新请求全部排队。缓解方案：限制缓存刷新的触发频率（用令牌桶），或降低刷新任务的重要性（用更低优先级的线程）。根本方案是保证线程池有足够容量——"缓存刷新"是低频事件，不应该频繁触发拒绝。

**避坑提示**：不要给一个"万能公式"说自己这样设线程池。要说"根据任务特征（IO 密集 vs CPU 密集、突发性、延迟敏感度）来选择参数"，并承认当前设置是经验值，需要压测验证。

---

### Q14: 死信队列的重试次数、重试间隔是怎么设置的？重试失败的消息怎么处理？怎么避免消息丢失和重复消费？

**结论**：最多重试 3 次（`MAX_RETRY_TIMES = 3`），无固定间隔（每次被消费就计为一次重试，间隔取决于消费循环的拉取频率），超过 3 次投递到死信队列 DLQ。防丢失由队列持久化 + 消息存储保证，防重复由 UPDATE 的幂等性保证。

**完整重试链路**：

```
消费者拉取消息 (basicGet with autoAck=false)
  ↓
处理业务 (从Redis读取计数 → UPDATE DB)
  ↓ 成功
ack(deliveryTag) → 消息从队列删除
  ↓ 失败 (异常)
retryOrDeadLetter()
  → 读取 x-retry-count 头 (默认 0)
  → retry < 3:
      构建新消息 (x-retry-count = retry + 1)
      投递到原队列 (重新排队等待消费)
      ack 原消息 (原消息从队列删除)
  → retry ≥ 3:
      投递到 DLX (死信交换机) → 路由到 DLQ (死信队列)
      ack 原消息 (原消息从队列删除)
  ↓ 重试/死信投递也失败
  fallback: basicNack(deliveryTag, false, false)
    (不 requeue，消息被丢弃——最后的兜底)
```

**为什么是 3 次**：
- 1 次重试：瞬时网络抖动、Redis 短暂的连接超时 → 大概率恢复
- 2 次重试：消费者重启、短暂资源竞争 → 可能恢复
- 3 次重试：小概率持久性故障 → 几乎不可能恢复
- 3 次以上：大概率是业务逻辑 BUG 或基础设施故障（如 Redis 长期 down，DB 连接满），继续重试只会堆积消息

**为什么没有固定重试间隔**：重试不是定时任务（不是"等 10 秒再试"），而是"重新投递到原队列"。消息重新进入队列尾后，消费者在拉取循环中自然遇到它——间隔取决于队列中还有多少条消息在排队。如果队列清空了，新投递的消息被消费者立即拉取（几乎无间隔）；如果队列积压了大量消息，重试消息要等前面的消息消费完。

**死信队列中的消息怎么处理**：进入 DLQ 的消息意味着"重试 3 次全部失败"——需要人工介入。通过 RabbitMQ 管理界面 (http://localhost:15672) 查看 DLQ 中的消息内容，分析根因（是业务逻辑 BUG？Redis 长时间不可用？消息格式错误？），修复后手动将消息从 DLQ 重新发布到原业务队列。

**防消息丢失**：
- RabbitMQ 持久化：队列声明为 `durable=true`，消息投递设置 `deliveryMode=2`（持久化）
- Publisher Confirm：投递时带 CorrelationData，Broker 确认接收后才算成功
- 拒绝时 nack + requeue：被 AbortPolicy 拒绝的消息重新入队，不丢失

**防重复消费**：
- 消息的幂等性：每条消息的内容只是 `cultureId` 字符串。消费者从 Redis 读取最新计数 → UPDATE 数据库。这个 UPDATE 是天然的幂等操作——执行 1 次和执行 100 次结果一样。
- 防重工具：`MessageWrapper` 用 `AtomicReference<AckState>` 防止同一条消息被 ack 或 nack 两次——`compareAndSet(PENDING, ACKED/NACKED)` 保证。

**追问预测**：

- *追问：如果消息进入 DLQ 了但没人发现怎么办？* → 这是当前的一个不足——没有 DLQ 的监控告警。面试时要坦诚并给出改进方案：用 `rabbitmqctl list_queues` 或 HTTP API 定期检查 DLQ 消息数，当 DLQ 消息 > 0 时通过钉钉/企业微信/邮件告警。或者用 Prometheus + RabbitMQ Exporter 暴露 DLQ 指标。
- *追问：为什么不用 Spring AMQP 的 `@RabbitListener` + `RetryTemplate` 的重试机制？* → @RabbitListener 的自动 ACK 模式下，消费失败会无限重试或按配置重试——但重试是在同一个消费者线程中阻塞重试，不释放连接，可能导致消费停顿。手动 basicGet + 重新投递的方式，重试消息进入队列尾，消费者继续处理下一条，不阻塞。这更贴近"异步削峰"的初衷。
- *追问：3 次重试都失败的消息，除了进死信，还有别的处理方式吗？* → 可以落库到一张"消息失败表"，由定时任务扫描重试。这种方式可以做更灵活的重试策略（如指数退避：第 1 次立即重试、第 2 次等 1 分钟、第 3 次等 10 分钟）。但引入额外的数据库表和定时任务，增加了复杂度。当前 DLQ 方案足够简单有效。

**避坑提示**：死信队列不是"万能药方"——要承认 DLQ 只是兜底，真正需要的是根因分析和监控告警。面试官更在意你"是否意识到 DLQ 消息被忽略"的风险。

---

### Q15: 主链路零阻塞是怎么保障的？消息发送失败时的降级策略是什么？有没有出现过消息堆积的情况？怎么排查和解决的？

**结论**：主链路（用户请求 → 返回响应）中只做 Redis 操作（1.3ms）+ MQ 投递（0.01ms），不等待 MQ 消费结果，也不等数据库写入。MQ 投递失败时有重试 + 降级策略。消息堆积未在生产中出现过，但准备了排查流程。

**主链路零阻塞的实现**：

用户点赞的完整路径，按阻塞/非阻塞划分：

```
阻塞（在主请求线程上，需要等待完成）:
  → 布隆过滤器检查 (本地内存, <0.01ms)
  → 获取分布式锁 (Redis, ~2-3ms)
  → 写 user_favorite 表 (数据库, ~2ms)
  → Redis 计数器 +1 (Redis, ~1ms)
  → 投递 MQ 消息 (RabbitMQ, ~0.01ms)  ← 最后一个同步操作
  → 释放分布式锁 (Redis, <1ms)
  → 返回成功响应
  
非阻塞（提交到线程池，主线程不等待）:
  → MQ 消费者处理 (UPDATE 数据库, ~2ms)
  → 缓存异步刷新 (查库 + 写缓存)
  → 操作日志异步落库
```

主链路中所有操作都是"同步但极轻量"——最大延迟是数据库写 user_favorite（2ms），总延迟约 6-8ms。MQ 投递只是将消息发给 Broker 并确认收到（不消费），延迟 0.01ms，对主链路几乎无影响。

真正的"重操作"（MQ 消费 UPDATE 数据库、缓存刷新查库、日志落库）都在线程池中异步执行，主线程返回后用户已经看到响应。

**MQ 投递失败时的降级策略**：

当前未实现自动降级——`rabbitTemplate.convertAndSend()` 失败会抛异常，当前只打日志。完整的降级链路应该是：
1. 第一次投递失败 → 重试（3 次，100ms 间隔）
2. 3 次重试全部失败 → 写入本地内存队列（LinkedBlockingQueue）
3. 定时任务每 30s 从本地队列批量重投到 RabbitMQ
4. 如果离线队列也满了 → 降级为同步写数据库（跳过 MQ）
5. 如果数据库也写失败 → 返回错误提示用户稍后重试（此时 Redis 计数器已更新，数据未丢失）

**消息堆积排查流程**（假设发生）：
1. **观察**：RabbitMQ 管理界面查看队列深度（viewCount 等队列），如果消息数量持续增长 → 堆积
2. **定位消费侧问题**：消费线程池是否耗尽？（活跃线程 = 最大线程，队列满，拒绝次数 > 0）；消费者是否异常挂死？（检查消费者日志，是否有大/量异常）
3. **定位生产侧问题**：生产速率是否突然激增？（是否有热点活动导致大量用户同时点赞）；是否是慢 SQL 导致消费端处理慢？（消费者 UPDATE 操作耗时，检查数据库慢查询日志）
4. **临时解决**：增加消费者实例；增大消息消费线程池核心线程数；临时降级——跳过部分消费（如浏览量直接丢弃不计）
5. **长期解决**：限流（前端按钮防抖 + 后端限流）+ 批量消费（积攒 10 条计数消息 → 批量 UPDATE）

**追问预测**：

- *追问：本地离线队列重启了不就丢了吗？* → 对，基于内存的离线队列重启会丢失。如果对可靠性要求高，可以先用本地 RocksDB/LevelDB 持久化重试队列。或者跳过本地队列这一步，直接让 RabbitMQ 的持久化消息承受——MQ 投递失败一般是网络问题，恢复后重试即可。
- *追问：主链路里写 user_favorite 表用了 2ms，如果数据库慢了怎么办？* → 数据库变慢会直接影响主链路延迟——这是没办法的，因为 user_favorite 的写入必须同步完成（要返回"点赞成功"给用户）。但可以优化：用异步写 + 前台乐观表示"已点赞"（如 Redis SET 一个临时标记），后台异步写数据库。矛盾在于——如果后台写入失败（如重复点赞），用户已经看到"点赞成功"了。当前选型是"同步保证强一致"，以毫秒级延迟换取状态确定性。
- *追问：如果 MQ 一直投递失败，队列一直积压，Redis 计数器过期了怎么办？* → 计数器 TTL 30 分钟通常足够让 MQ 恢复。如果 MQ 长时间不可用（>30 分钟），Redis 计数器过期丢失，这时增量数据会丢失。兜底：在 MQ 恢复后全量刷新所有活跃数据的计数（从 user_favorite 和 view_record 表统计）。

**避坑提示**：不要说"主链路零阻塞，完全没有瓶颈"。要诚实说明"哪些操作在链路上（数据库写入、Redis 操作），哪些在异步里（MQ 消费、缓存刷新、日志）。主链路延迟低是因为操作本身就轻量，不是用异步把同步操作藏起来了"。

---

## 五、安全与日志监控

---

### Q16: JWT 无状态认证 + Token 无感刷新，具体实现逻辑是什么？怎么处理 Token 泄露、过期、续期的安全问题？

**结论**：当前实现的是基础 JWT 认证——Token 有效期 24 小时，未实现无感刷新。过期后前端拦截 401，跳转登录页。Token 泄露的防护依赖 HTTPS + 短期有效期，续期可以通过 refreshToken 机制扩展。

**当前实现逻辑**：

```
登录流程:
  用户输入 username + password
    → 数据库验证 (BCrypt.matches)
    → 使用 jjwt 签发 Token
      Jwts.builder()
        .setSubject(username)
        .claim("userId", userId)
        .claim("role", role)
        .setIssuedAt(new Date())
        .setExpiration(new Date(System.currentTimeMillis() + 86400000)) // 24h
        .signWith(secretKey, SignatureAlgorithm.HS256)
        .compact()
    → 返回 Token 给前端 (不存储在后端)
    → 前端存储到 localStorage

鉴权流程 (每次请求):
  拦截器提取 Authorization 头
    → jjwt.parseClaimsJws(token) 验证签名 + 过期时间
    → 提取 userId/role/username
    → 写入 UserContext (ThreadLocal)
```

**JWT 的三个关键配置**：
- `jwt.secret`: HMAC-SHA256 签名密钥（至少 256 位），配置文件中注入
- `jwt.expiration`: 86400000ms（24 小时）
- `jwt.remember-expiration`: 604800000ms（7 天，预留但未实现"记住我"功能）

**当前未实现——Token 无感刷新**：

"无感刷新"（Silent Refresh）是指：用户在活跃使用系统时，Token 快过期自动续期，无需跳转登录页。标准方案：
- 双 Token 机制：短期 AccessToken（15 分钟）+ 长期 RefreshToken（7 天）
- AccessToken 过期 → 前端用 RefreshToken 换取新的 AccessToken → 后端验证 RefreshToken 是否在黑名单 → 签发新 AccessToken（旧的 AccessToken 自动过期）
- RefreshToken 也过期 → 跳转登录页

未实现的原因：对于档案浏览类应用，用户会话通常不会持续超过 24 小时。如果需要刷新，用户重新登录即可。对于需要长时间在后台工作的管理端，RefreshToken 机制才更有价值。

**Token 泄露的处理**：
1. 泄露后攻击者可以冒充用户访问 24 小时内的所有接口
2. 当前无主动失效机制——一旦签发，只有等 24 小时后自动过期
3. 改进方案：Redis 维护 Token 黑名单。需要"踢人"时把 Token 加入黑名单（设置 TTL = Token 剩余有效期），拦截器检查黑名单

**Token 续期的处理**：
当前是"过期重新登录"，不做续期。如果要做续期：
1. 短期 Token（AccessToken 15 分钟） + 长期 Token（RefreshToken 7 天）
2. RefreshToken 在服务端维护（存储在 sys_user 关联表或 Redis 中），可以撤销
3. 前端设置 Axios 拦截器——401 时自动用 RefreshToken 换新 AccessToken，用户无感
4. RefreshToken 也过期 → 跳转登录

**追问预测**：

- *追问：JWT 和 OAuth2.0 有什么区别？什么时候该用 OAuth？* → JWT 是 Token 格式，OAuth2.0 是授权协议。JWT 可以用于 OAuth2.0 中的 AccessToken。项目的场景是"自家应用 + 自家用户"，不需要第三方授权——JWT 足够。如果未来需要"微信扫码登录"或"允许第三方访问用户数据"，就需要 OAuth2.0。
- *追问：JWT 为什么不存数据库？如果有用户被禁用，Token 还在有效期内怎么办？* → JWT 自包含（payload 携带 userId），不依赖服务端存储。被禁用的用户不会主动失效——这是 JWT 的固有缺陷。解决方案是每次请求时检查用户状态（但损失了"无状态"的优势），或 Redis 黑名单（但引入了状态）。当前项目：用户禁用后在登录时被阻止（密码验证通过但检查 status != 1），已有 Token 不会立即失效，等 24 小时过期。这是一个已知权衡。
- *追问：前端存储 Token 用 localStorage 还是 cookie？* → 当前用 localStorage。localStorage 易受 XSS 攻击（恶意脚本读取），Cookie 设 httpOnly 可以防 XSS 但易受 CSRF。最佳方案：AccessToken 存内存（JS 闭包）+ RefreshToken 存 httpOnly Cookie + 服务端 CSRF Token。但这增加了前端复杂度。

**避坑提示**：JWT 的"无状态"既是优势也是劣势——要能说清什么场景适合（分布式、不需要主动失效），什么场景不适合（需要即时踢下线）。

---

### Q17: AOP 切面捕获接口日志、耗时与异常，日志的内容是怎么设计的？敏感数据是怎么脱敏的？

**结论**：AOP 切面在 Controller 层全量拦截，记录操作人、方法名、参数、耗时、结果、错误信息。目前未做敏感数据脱敏——日志中参数直接 toString，密码字段在 Controller 层已处理（LoginDTO 只传前端，不涉密码原文）。

**日志内容设计**：

AOP 记录的字段（对应 OperationLogEvent）：
- `userId` / `username`: 从 UserContextHolder 获取，匿名请求为空
- `operation`: `"类名:方法名"` 格式，如 `"EthnicCultureController:like"`
- `method`: 全限定方法名，如 `"com.ethnicculture.controller.EthnicCultureController.like"`
- `params`: 方法参数的非空字符串表示，过滤掉 HttpServletRequest 类型参数
- `result`: 1=成功，0=失败
- `errorMsg`: 异常消息（仅在失败时记录）
- `duration`: 方法执行耗时（毫秒），从 @Before 到 @AfterReturning/@AfterThrowing

**敏感数据脱敏现状**：

当前未做显式脱敏。但有几个天然的"避嫌"点：
1. 登录接口的 password 在 LoginDTO 中，AOP 记录参数时 `LoginDTO.toString()` 会输出密码。这是一个安全风险——日志中可能包含明文密码
2. 手机号（RegisterDTO.phone）也会出现在日志参数中
3. JWT Token 不出现在参数中（在 Header 中，AOP 不记录 Header）

**如果要加脱敏，方案**：
1. 在 DTO 的 `toString()` 方法中过滤敏感字段（不输出 password、phone 等）
2. 或者在 AOP 的 `getParams()` 方法中对参数值做正则脱敏（如手机号中间四位变 `****`）
3. 或者用 Jackson 的 `@JsonIgnore` 注解标记敏感字段（影响序列化，不推荐）

**追问预测**：

- *追问：日志中包含密码的风险怎么处理？* → LoginDTO 应该重写 toString() 方法不输出 password 字段。更彻底的是——Controller 的 login 方法接收 LoginDTO 后，AOP 在记录时应该识别这是一个"含敏感信息"的参数类型并自动脱敏。当前未做，已知风险。
- *追问：操作日志的保留周期是多久？日志量大了怎么办？* → 当前直接写 MySQL operation_log 表，没有归档策略。operation_log 表 50k 测试数据。线上每日可能上万条，几个月就到百万级。优化方向：按月分表（operation_log_202601），或迁移到时序数据库（ClickHouse / ElasticSearch）专门存日志。对历史数据做归档 + 压缩。
- *追问：AOP 在 @AfterThrowing 中记录的异常信息够详细吗？* → 记录的是 `e.getMessage()`，不是完整堆栈。对于生产排查，只有消息不够——需要堆栈追踪到具体出错的代码行。改进：在 errorMsg 中附加堆栈的前 5 层（`Arrays.stream(e.getStackTrace()).limit(5).collect(...)`）。

**避坑提示**：日志安全是面试官考察"安全意识"的常见切入点。要主动提及"密码可能出现在日志中"的风险，而不是等面试官追问。

---

### Q18: 事件驱动异步入库日志，日志的写入性能怎么保障？怎么避免日志写入阻塞主业务流程？日志的存储方案是什么？

**结论**：日志写入完全解耦于主链路——AOP 只做事件发布（微秒级），@EventListener 异步提交到 asyncTaskExecutor 线程池执行数据库 INSERT。主链路不被日志写入阻塞。当前存储方案是 MySQL operation_log 表，后续可迁移到 ES/ClickHouse。

**事件驱动架构的性能保障**：

```
Controller 方法执行 (主线程)
  → AOP @AfterReturning (主线程)
    → 组装 OperationLogEvent (堆内存, 微秒)
    → operationLogEventPublisher.publish(event) (Spring 同步事件发布, ~0.01ms)
    → 主线程返回响应

Spring ApplicationContext (事件分发线程)
  → @EventListener 收到事件
  → 提交到 asyncTaskExecutor 线程池 (异步)
  
asyncTaskExecutor 线程 (业务逻辑完成后)
  → OperationLogService.logOperation(...)
    → operationLogMapper.insert(operationLog) (数据库 INSERT, ~2ms)
```

关键设计：
- 事件发布是同步的但极轻量——Spring ApplicationEventPublisher 的事件分发只是把事件交给 @EventListener 标注的方法
- @EventListener 内部不直接执行数据库 INSERT，而是 `executorService.execute(() -> { 写库 })` 提交任务到线程池——这意味着事件分发线程也立即返回，不阻塞
- 真正的写库操作在 asyncTaskExecutor 线程池中完成，主业务链路完全不受影响

**数据库写入的性能**：
- 单条 INSERT 到 operation_log 表耗时约 2ms（测试数据）
- 线程池并发写入——如果短时间内有 100 个请求，就有 100 条日志 INSERT，在线程池中交错执行
- operation_log 表配置了合理的索引（user_id, create_time 等），查询日志时的分页性能可以接受

**日志存储方案**：

当前：MySQL operation_log 表

未来扩展方向：

| 方案 | 适用场景 | 优势 | 劣势 |
|------|---------|------|------|
| MySQL 分表 | 日志量中等 (日增<10万) | 无新技术成本 | 查询跨表麻烦 |
| ElasticSearch | 日志量大 + 需要全文检索 | 全文搜索、聚合分析 | 运维成本高 |
| ClickHouse | 日志量超大 (日增>百万) | 列式存储、压缩率高 | 不适合 OLTP |
| Kafka → 日志中心 | 完善的可观测性体系 | 统一采集、统一存储 | 架构重 |

当前项目规模（日均可能几百到几千次操作），MySQL 足够。但如果要查看"某用户过去一年的操作日志"且数据量以万条计，MySQL 分页查询会变慢，此时加 ES 做日志二级索引。

**追问预测**：

- *追问：如果线程池满了，日志任务被拒绝怎么办？* → asyncTaskExecutor 用的是 CallerRunsPolicy——被拒绝时由调用方线程执行。对于日志写入，异常不会影响主业务（用户已经收到响应），但日志记录失败时打 `log.error("操作日志事件消费失败", e)`。这是有意的取舍——日志丢失可接受，主业务永远优先。
- *追问：为什么不直接用 Logback/Log4j 写日志文件，而要写数据库？* → 文件日志适合给开发者排查问题（"系统出了什么异常"），数据库操作日志适合给产品/运营审计（"谁在什么时候做了什么操作"）。两者是不同维度的日志——文件日志记录系统行为，操作日志记录用户行为。操作日志写数据库方便按用户/时间/操作类型查询和统计。
- *追问：操作日志有没有性能影响？* → 对主链路的额外延迟约 0.01ms（事件发布），几乎可以忽略。数据库的压力——每条操作产生一条 INSERT，在 asyncTaskExecutor 线程池中异步完成。在高并发场景下（比如 500 QPS），每秒 500 条 INSERT 对 MySQL 有一定压力，但还在可控范围。真正的瓶颈可能是 operation_log 表的写热点——所有 INSERT 都往一张表写。考虑按日期分表缓解。

**避坑提示**：要区分"日志"的两种含义——开发者调试日志（Logback/Log4j 的文件日志）和业务操作审计日志（数据库 operation_log）。项目两者都有。

---

### Q19: 项目中有没有实现接口的限流、防刷策略？比如基于 IP、用户 ID 的限流，是怎么实现的？

**结论**：当前未实现系统级的限流防刷。点赞/收藏操作通过分布式锁的 waitTime（3 秒）间接起到了"同一用户对同一内容的并发控制"，但没有基于 IP/用户 ID 的全局频率限制。

**现状分析**：

已有的"防刷"保障：
1. **分布式锁（点赞/收藏）**：同一用户 + 同一内容的重复请求被锁串行化，其中一个拿到锁执行，另一个等锁 → 锁内检查是否已点赞 → 抛"已经点赞过"。这防止了重复点赞，但不防止"同一用户疯狂点赞不同内容"。
2. **布隆过滤器（查存在）**：不存在的 ID 被过滤器拦截，防止无效 ID 刷接口。
3. **@RequireLogin 注解**：未登录用户无法调用需要登录的接口——这是身份认证，不是频率控制。

**缺失的限流能力**：

| 需要防护的场景 | 当前状态 | 风险 |
|-------------|---------|------|
| 同一用户高频点赞（刷点赞数） | 未限制 | 可短时间内给大量内容点赞 |
| 同一 IP 暴力破解登录 | 未限制 | 可字典攻击（但 BCrypt 哈希增加了破解成本） |
| 同一用户高频搜索（消耗 DB） | 未限制 | 可制造慢查询对数据库施压 |
| 爬虫全量抓取档案 | 未限制 | 可遍历全部数据 |

**如果要实现限流，方案**：

1. **接口级别——Guava RateLimiter**：最轻量。在 AuthInterceptor 中加一个 RateLimiter，限制每个用户每秒最大请求数（如 10 QPS/用户）。超限返回 429 Too Many Requests。
   ```
   RateLimiter limiter = RateLimiter.create(10.0); // 10 permits/second
   if (!limiter.tryAcquire()) {
       throw new BusinessException(429, "请求过于频繁");
   }
   ```
   限制：本地限流，多实例各自计数，不精确。

2. **分布式级别——Redis 滑动窗口**：用 Redis Sorted Set 记录每个用户的请求时间戳，每次请求时统计窗口内的请求次数。
   ```
   ZADD user:ratelimit:{userId} {timestamp} {requestId}
   ZREMRANGEBYSCORE user:ratelimit:{userId} 0 {now - window}
   ZCARD user:ratelimit:{userId} → 超过阈值 → 限流
   ```
   优点：分布式精确。缺点：每个请求多一次 Redis 操作。

3. **成熟方案——Sentinel/Resilience4j**：阿里巴巴 Sentinel 提供了更丰富的限流/熔断/降级能力，支持控制台动态调整规则。适合中等规模以上项目。

**追问预测**：

- *追问：为什么没做限流？* → 业务规模小，没有遇到实际的刷接口问题。但这是一个已知的待改进点——面试时诚实承认比假装"不需要"更好。
- *追问：如果现在要加限流，你会优先加在哪些接口上？* → 优先级：登录接口（防暴力破解） > 搜索接口（防慢查询攻击） > 点赞/收藏接口（防刷数据） > 列表接口（防爬虫）。登录接口用 Redis 记录失败次数，超过 5 次锁定 15 分钟。
- *追问：限流对用户体验的影响怎么平衡？* → 阈值设置要宽松——正常用户的操作频率远低于恶意脚本。限流主要拦截的是异常高频（如 1 秒内 50 次请求）。对于真正的高频合法用户（如管理员批量操作），可以通过白名单放行。

**避坑提示**：限流是面试官考察"你有没有考虑过系统被滥用"的切入点。即使没实现，也要能说出"知道在哪些地方需要加、用什么方案加"。

---

## 六、性能压测与问题排查

---

### Q20: 这个项目的压测指标是怎样的？压测中遇到过哪些瓶颈？分别是怎么解决的？

**结论**：做了组件级别的基准测试（数据层、缓存层、消息层），未做端到端 HTTP 全链路压测。发现的最严重瓶颈是分页查询 1.5 秒（缺少 MyBatis Plus 分页插件 + 索引未命中），优化方向明确但尚未修复。

**压测范围**：

| 层级 | 测试 | 核心发现 |
|------|------|---------|
| 数据层 | 分页查询 / COUNT / FULLTEXT / INSERT / 索引验证 / 连接池压力 | 分页查询 1.5s 严重超标 ✅ 其余正常 |
| 缓存层 | Redis 读写延迟 / 布隆过滤器 / 分布式锁 / 计数器 / 击穿 / 雪崩 | 全部通过 ✅ |
| 消息层 | 投递延迟 / 消费者吞吐 / 死信队列 | 全部通过 ✅ |

**瓶颈 1——分页查询 1.5 秒（P0 紧急）**：

现象：`ORDER BY create_time DESC` 的分页查询在 100k 数据下耗时 1499ms，深分页（page=5000）耗时 1503ms。

根因分析（通过 EXPLAIN 验证）：MySQL 优化器在数据全部满足 `deleted=0 AND status=1` 时，判断 `idx_deleted_status` 索引无选择性（因为所有数据都满足），转而使用 `idx_create_time` 索引进行全索引扫描（type=index, Backward index scan）。此外，怀疑缺少 MyBatis Plus 的 `PaginationInterceptor`（分页插件），导致查询先查出全部数据再到内存分页。

修复方案：
1. 添加 MyBatis Plus 分页插件 `MybatisPlusInterceptor` + `PaginationInnerInterceptor`
2. 创建联合索引 `(deleted, status, create_time)` 替代分别建索引
3. 结合游标分页替代深分页（WHERE id > lastId LIMIT 10），避免扫描大量无关行

**瓶颈 2——Redisson 计数器相比本地计数器慢 100-300 倍（P2 可接受）**：

现象：100 线程并发自增，Redisson 20k QPS vs 本地 synchronized 2.5M QPS。

原因：Redis 网络 IO 的天然代价（每次 incrementAndGet 都是一次网络往返）。

应对：当前架构已通过"Redis 计数 → MQ 异步刷盘"将数据库写入延迟隐藏。20k QPS 已经远超业务需要的几百 QPS。这不是"需要修复的瓶颈"，而是"分布式一致性的必要代价"。

**瓶颈 3——索引未命中问题（P1 重要）**：

现象：`deleted + status + ORDER BY create_time` 查询没有使用预期的 `idx_deleted_status` 复合索引，走了 `idx_create_time`。

根因：当 `deleted=0 AND status=1` 条件几乎筛选全部数据时，MySQL 可能选择全索引扫描而非使用该索引。这是 MySQL 优化器的代价模型决策。修复：建 `(deleted, status, create_time)` 联合索引，让优化器在一个索引中完成筛选 + 排序。

**追问预测**：

- *追问：端到端压测为什么没做？* → 组件测试先验证各层级独立性能，找到最薄弱的环节（数据层 1.5s）。在全链路压测之前先把最慢的查询修好——否则全链路结果会被这个瓶颈主导，测不出其他问题。这是"先解决已知问题，再发现未知问题"的策略。
- *追问：如果让你主导一次完整的性能测试，你会怎么设计？* → 1) 造数据：用固定种子的数据生成脚本，保证可复现；2) 组件级基准测试：像现在一样逐层验证；3) 端到端压测：JMeter 模拟 100-500 并发用户按真实流量比例（200 浏览:5 点赞:1 收藏）施压，预热缓存后持续压测 10 分钟，记录 P50/P99 延迟、错误率、吞吐量；4) 瓶颈分析：通过 FlameGraph 或 Arthas profiling 找 CPU 热点，通过慢查询日志找 DB 瓶颈，通过 Redis SLOWLOG 找缓存瓶颈。
- *追问：JMeter 压测时怎么模拟真实用户行为？* → 用 CSV Data Set Config 从测试数据文件中读取真实的 userId 和 cultureId，用随机定时器模拟人类点击间隔（200ms-2s 随机），用 HTTP Header Manager 携带不同的 JWT Token。压测脚本已部分准备（`docs/performance-test-guide.md` 中有 JMeter 配套文件）。

**避坑提示**：不要虚报"做了全链路压测，QPS 5000+ 没问题"。诚实说明"做了组件级测试，发现了分页查询瓶颈，端到端压测在计划中"。数据真实比数据好看更重要。

---

### Q21: 分布式场景下，怎么排查缓存、数据库、MQ 之间的数据不一致问题？有没有典型的线上问题案例可以分享？

**结论**：按数据流向逐层排查——从用户看到的值 → Redis 计数器 → 数据库字段 → MQ 消息。每层对比数值是否一致，定位不一致的起点。暂无线上案例，但准备了排查方法论。

**数据流向和排查路径**：

数据在三层间的流转：
```
用户看到的值 (syncCountersToCulture 覆盖后的缓存对象)
  ↕
Redis 计数器 (view:count:{id}, like:count:{id}, collect:count:{id})
  ↕
MQ 消息 (viewCount/likeCount/collectCount 队列, 携带 cultureId)
  ↕
数据库 (ethnic_culture 表的 view_count, like_count, collect_count)
```

**排查步骤（以"点赞数不一致"为例）**：

步骤 1——查 Redis：`GET like:count:{cultureId}` → 拿到当前计数器的值
步骤 2——查数据库：`SELECT like_count FROM ethnic_culture WHERE id = {cultureId}` → 对比 Redis 和 DB 的差异
步骤 3——查 MQ 队列：`rabbitmqctl list_queues name messages | grep likeCount` → 查看队列深度。如果队列有积压（消息数 > 0），说明消费速度跟不上——Redis 的最新值还没同步到 DB
步骤 4——查死信队列：`rabbitmqctl list_queues name messages | grep likeCount.dlq` → 如果有消息，说明消费者处理失败超过 3 次，消息被移到了 DLQ。查看 DLQ 中的消息内容，分析消费失败原因
步骤 5——查消费者日志：搜索 `"消息处理失败"` 或 `"likeCount"` 关键字，查看是否有异常堆栈，找到消费失败的根本原因

**典型不一致场景和原因**：

| 现象 | 可能原因 | 排查方式 |
|------|---------|---------|
| Redis > DB | MQ 消费延迟或积压 | 查 MQ 队列深度 |
| Redis > DB (MQ 队列空) | 消费者异常退出或处理失败 | 查 DLQ + 消费者日志 |
| DB > Redis | 计数器 TTL 过期，被数据库值重置 | 查 Redis TTL，检查 syncCountersToCulture 逻辑 |
| 缓存存的旧值 | 数据库更新后删缓存失败 | 查缓存 key 是否存在 + 数据库该记录的 update_time |
| 用户点赞了但数据库没有 | 分布式锁没拿到 + MQ 投递失败 | 查 MQ 投递日志 + user_favorite 表 |

**追问预测**：

- *追问：你怎么确保数据最终一致？用什么工具验证？* → 没有自动化的数据一致性校验工具。一般靠监控异常（MQ 队列积压告警、DLQ 消息告警、数据库慢查询告警）来间接判断。如果要主动验证，可以写一个定时任务：遍历所有活跃的 Redis 计数器，对比数据库的值，不一致的上报告警。
- *追问：MQ 消息丢了你怎么发现？* → MQ 消息丢失很难主动发现——因为 RabbitMQ 告诉你"消息已确认"就认为投递成功了。但如果消费者一直没收到消息，队列深度会异常下降（消息被消费但没有 UPDATE 到数据库）。这种场景极少——更常见的是"消息没丢但消费失败"，会被 x-retry-count 和 DLQ 捕获。
- *追问：有没有考虑过做数据对账？* → 数据对账是一个好的工程实践——定期（如每天凌晨）全量统计 user_favorite 表中的点赞数，与 ethnic_culture 表中的 like_count 对比，不一致的补上差异并告警。相当于对 MQ 异步链路做最终一致性的"兜底校验"。目前未实现但方向清晰。

**避坑提示**：排查问题时不要急着说"可能是 XX 问题"，先说要"按数据流转方向逐层排查，从现象倒追源头"。方法论比具体答案更重要。

---

### Q22: 系统的监控告警体系是怎么搭建的？核心指标怎么采集和告警？

**结论**：当前未搭建系统化的监控告警体系。组件状态依赖日志输出和中间件自带的管理界面（RabbitMQ 15672、Redis CLI、MySQL 慢查询日志）。这是需要优先补齐的 P1 缺项。

**当前可用的观察手段**：

| 工具 | 可观察的内容 |
|------|-----------|
| Spring Boot 日志 | 错误堆栈、业务异常、线程池状态 |
| RabbitMQ 管理界面 (15672) | 队列深度、消费速率、DLQ 消息 |
| Redis CLI / MONITOR 命令 | 实时查看每一条 Redis 命令 |
| MySQL 慢查询日志 | SQL 耗时超过 long_query_time 的查询 |
| 线程池日志 | 启动时输出核心/最大线程数、队列容量 |

**缺失的监控能力**：

| 缺失项 | 影响 |
|--------|------|
| 无接口响应时间监控 | 无法感知系统变慢 |
| 无缓存命中率统计 | 不知道缓存策略效果如何 |
| 无 MQ 队列深度告警 | 消息积压可能持续数小时才被发现 |
| 无 DLQ 消息告警 | 消费失败的消息可能永远无人关注 |
| 无数据库连接池监控 | 连接耗尽前无预警 |
| 无 JVM 内存/GC 监控 | Full GC 频繁时无法主动发现 |

**推荐的监控体系搭建方案**：

第一层——指标暴露（Micrometer + Actuator）：
```
引入 spring-boot-starter-actuator + micrometer-registry-prometheus
暴露 endpoints: /actuator/metrics, /actuator/health
核心指标: jvm_memory_used, http_server_requests_seconds, 
          cache_gets_total, rabbitmq_queued_messages
```

第二层——指标采集（Prometheus）：
```
scrape_interval: 15s
targets: Spring Boot /actuator/prometheus 端点
```

第三层——可视化与告警（Grafana + AlertManager）：
- 面板 1：接口 QPS + P50/P99 延迟
- 面板 2：缓存命中率、布隆过滤器误判率
- 面板 3：MQ 队列深度 + 消费速率
- 面板 4：线程池活跃线程 + 队列使用率 + 拒绝次数
- 面板 5：JVM 堆内存使用 + GC 频率
- 告警规则：接口 P99 > 1s → 告警；MQ 队列深度 > 1000 → 告警；DLQ 消息数 > 0 → 告警；线程池拒绝次数 > 0 → 告警

**追问预测**：

- *追问：如果不用 Prometheus，还有其他方案吗？* → ELK Stack（Elasticsearch + Logstash + Kibana）可以从日志中提取指标做可视化。或者直接接阿里云/腾讯云的 APM 产品（ARMS/SkyWalking），免运维自动采集。当前项目规模最小成本方案是 Micrometer + Prometheus + Grafana，三者在 Spring Boot 生态中最成熟。
- *追问：日志监控有没有做？怎么排查线上问题？* → 当前通过 Spring Boot 的日志输出到控制台/文件，排查问题靠 `grep`。生产环境应该接入 ELK——Logstash 采集日志文件 → ElasticSearch 存储 → Kibana 全文搜索。这样排查问题时不用登服务器翻日志，直接在 Kibana 里按 traceId/userId/时间范围过滤。
- *追问：没做监控，那你认为最重要的三个监控指标是什么？* → 1) 接口 P99 延迟——这是用户感知的最终指标；2) MQ 队列深度——异步链路的风向标，积压意味着异步环节出了问题；3) 数据库连接池使用率——连接耗尽会导致整个系统不可用。

**避坑提示**：监控告警是系统可靠性的最后一道防线。面试官问这个问题是想知道"你有没有想过系统在凌晨 3 点出问题时怎么办"。能说出"现在没有，但我知道应该怎么做"比支吾不语强得多。

---

## 七、项目落地与扩展思考

---

### Q23: 项目中涉及到档案数据的分布式存储，数据的同步策略是怎样的？怎么保证多节点之间的数据一致性？

**结论**：当前是单机部署（一个 MySQL 实例 + 一个 Redis 实例），不存在多节点数据同步问题。如果多实例部署，缓存层面的一致性由 Redis 单节点保证，数据库由 MySQL 主从复制保证。

**现状说明**：

"分布式存储"在当前项目中特指：
- 数据在 MySQL（持久化存储）
- 缓存数据在 Redis（高速读写）
- 计数在 Redis RAtomicLong（实时计数）

三种存储各司其职，不是"同一份数据存多份相互同步"的分布式存储。所有数据以 MySQL 为最终真实来源（Source of Truth），Redis 是缓存层 + 计数层。

**多实例场景下的数据一致性**：

如果应用部署多个实例（2+ Tomcat + 同一个 MySQL + 同一个 Redis）：

1. **缓存一致性**：所有实例共享同一个 Redis，缓存的一致性由 Redis 单节点保证。各实例间不需要同步——读写都走同一个 Redis。

2. **计数器一致性**：Redisson RAtomicLong 基于 Redis INCRBY，所有实例的计数操作都打到同一个 Redis 上，天然全局一致。本地计数器（AtomicLong）不能用于多实例。

3. **分布式锁一致性**：Redisson RLock 基于 Redis SETNX，所有实例共享同一个 Redis，锁的互斥性由 Redis 单线程模型保证。

4. **数据库一致性**：只有一个 MySQL 主节点（或主从架构），写操作都打到主节点上，由 MySQL 的 ACID 事务保证一致性。

**主从架构下的数据同步**：

如果 MySQL 采用主从架构：
- 写操作 → 主库（Master）
- 读操作 → 从库（Slave）
- 同步策略：MySQL 异步复制（默认）或半同步复制

异步复制下可能出现"写入主库成功，但还没同步到从库"的窗口——如果此时有读操作命中了从库，可能读到旧数据。对应策略：
- 缓存层的数据优先从缓存读（数据已经在 Redis 中）
- 缓存 miss 时才查从库 → 此时可能在短暂窗口内看到旧数据 → 但这个窗口通常 < 1 秒（MySQL binlog 同步延迟极低）
- 如果对一致性要求极高，关键业务走主库读取（如编辑操作的前置查询）

**追问预测**：

- *追问：如果 Redis 用了 Cluster 模式，缓存数据分片存储，计数器的一致性怎么办？* → Redis Cluster 将 key 按 slot 分到不同节点。计数器 `like:count:{id}` 是单个 key，它只存在于一个 slot 的一个节点上，INCRBY 在这个节点上原子执行。多实例共享的 Redis Cluster 计数器仍然全局一致。但如果这个节点挂了，在故障转移期间该 key 不可写——受影响的只是那部分数据的计数器暂时无法更新。可以用 Redisson 的集群模式自动发现 slot 迁移。
- *追问：为什么不把缓存数据也同步到每个实例的本地内存？* → 本地缓存（Caffeine/Guava Cache）的延迟极低（<0.001ms），但多实例间数据不一致——实例 A 更新了数据并删了本地缓存，实例 B 的本地缓存还是旧值。方案：用 Redis Pub/Sub 广播缓存失效通知，各实例收到后删除对应本地缓存。这样本地缓存 + Redis 两级缓存，兼顾延迟和一致性。但当前项目缓存数据量不大、Redis 延迟已足够低（1.3ms），不需要加本地缓存的复杂度。
- *追问：如果数据库本身要做分库分表呢？* → 见 Q10 的分片键设计 + Q24 的扩展思考。

**避坑提示**：分布式数据一致性是个很广的话题——要从"当前单机架构 → 多实例 → 主从 → 集群分片"逐层展开，每个阶段的同步策略和一致性保证不同。不要从一开始就跳到最复杂的方案。

---

### Q24: 如果要给系统增加档案数据的全文检索功能，你会怎么设计？

**结论**：用 ElasticSearch（ES）做独立搜索引擎，通过 MySQL binlog 监听（Canal）或应用层双写保持 ES 索引与数据库同步。ES 专注于搜索和聚合，MySQL 专注于事务性数据存储。

**设计方案**：

```
           应用层
         (新增/编辑/删除档案)
              |
         同时写 MySQL + 发送 ES 索引消息
              |
         RabbitMQ (indexQueue)
              |
         索引消费者 (ESIndexConsumer)
              |
         ElasticSearch Cluster
              |
         搜索服务 (SearchService)
              |
         前端展示 (搜索结果 + 高亮 + 分页)
```

**分步实现**：

步骤 1——ES 索引设计：

```
索引: ethnic_culture
字段映射:
  - id: keyword (精确匹配)
  - name: text (ik_smart 中文分词) + keyword (排序)
  - region: keyword (聚合筛选)
  - ethnicType: keyword (聚合筛选)
  - contentType: keyword (聚合筛选)
  - era: keyword (聚合筛选)
  - summary: text (ik_smart 中文分词)
  - content: text (ik_smart 中文分词)
  - viewCount: long (排序)
  - likeCount: long (排序)
  - createTime: date (排序)
```

使用 ik_smart 中文分词器（ik_max_word 精度更高但索引更大），支持中文词组搜索。

步骤 2——数据同步：

方案 A（推荐）：Canal 监听 MySQL binlog → 解析 INSERT/UPDATE/DELETE → 同步到 ES。解耦应用层，不影响主业务流程。

方案 B（轻量）：应用层双写。新增/编辑档案时，同时调 `elasticsearchClient.index()`。简单直接但耦合度高。

推荐方案 A——监听 binlog 的异步同步，延迟通常在毫秒级别。

步骤 3——搜索接口设计：

```
GET /api/ethnic-culture/search
  ?keyword=藏族服饰&ethnicType=藏族&contentType=非遗技艺&page=1&size=10

SearchService:
  → ES bool query:
    - must: multi_match(keyword, [name^2, summary^1.5, content]) 
    - filter: term(ethnicType), term(contentType)
  → 结果 + 分面聚合（前 10 个 ethnicType、contentType 分类计数）
  → 高亮处理 (搜索结果中的关键词高亮)
```

步骤 4——MySQL FULLTEXT 保留吗？

保留作为降级方案——如果 ES 不可用，降级为 MySQL FULLTEXT + LIKE 混合搜索。CPU 开销较大但保证基本可用。

**追问预测**：

- *追问：ES 选型为什么不是 Solr？* → ES 生态更活跃（ELK Stack 统一技术栈）、分布式架构更成熟（自动分片、故障转移）、RESTful API 学习成本低。Solr 在特定场景（如电商分类检索）有优势，但社区活跃度和开发者认知度不如 ES。
- *追问：ES 索引占多少内存？中文分词性能如何？* → 100k 档案数据全量索引预计几百 MB 到 1GB。中文分词用 ik_smart 比标准分词器更适合短文本搜索，分词后的词语与用户搜索关键词匹配度高。
- *追问：ES 和 MySQL FULLTEXT 对比，什么时候选 MySQL FULLTEXT？* → MySQL FULLTEXT 适合小数据量（<100 万行）且只需要简单关键词搜索的场景。当前项目 100k 数据 MySQL FULLTEXT 性能很好（<1ms）。引入 ES 的时机是：a) 数据量 >100 万行；b) 需要复杂查询（同义词、拼音搜索、权重排序）；c) 需要聚合分析（如按民族类型统计数量分布）。

**避坑提示**：搜索方案选型不要只说"ES 好"，要说出"什么时候 MySQL FULLTEXT 足够，什么时候必须上 ES"——这是技术成本的评估能力。

---

### Q25: 你觉得这个项目目前还有哪些不足？如果让你重构，你会从哪些方面优化？

**结论**：分层梳理不足——P0 分页性能问题必须修复；P1 补充监控、动态 workerId、限流防刷；P2 升级技术栈、优化测试覆盖率、完善文档。重构采用渐进式改进而非重写。

**当前不足——分层评估**：

**🔴 P0——必须修复（影响可用性）**：

1. **分页查询 1.5 秒**：根因是缺少 MyBatis Plus 分页插件 + 索引优化器选择错误索引。修复方案明确：添加 PaginationInterceptor + 建联合索引 `(deleted, status, create_time)`。

**🟡 P1——尽快补齐（影响可靠性和安全）**：

2. **监控告警缺失**：无接口延迟监控、无 MQ 队列深度告警、无 DLQ 消息告警。用 Micrometer + Prometheus + Grafana 搭建基础监控面板，先覆盖最重要的 5 个指标。

3. **workerId 写死配置文件**：多机部署有重复 ID 风险。改为启动时 Redis INCR 动态获取。

4. **接口限流防刷缺失**：登录接口、搜索接口无频率限制。用 Redis 滑动窗口对关键接口加限流保护。

5. **Token 泄露无主动失效**：JWT 签发后 24 小时内无法撤销。引入 Redis Token 黑名单或 RefreshToken 机制。

6. **操作日志无脱敏 + 无归档**：密码可能出现在日志中；日志表无分表/归档策略。修复 DTO 的 toString() + 按月分表。

**🟢 P2——未来改进（提升质量和开发体验）**：

7. **升级 Java 17 LTS + Spring Boot 3.x**：利用虚拟线程简化线程池管理、利用 GraalVM Native Image 提速启动（适合容器化部署）。

8. **补充单元测试和集成测试**：当前只有性能基准测试，缺少业务逻辑的单元测试。建议用 JUnit 5 + Mockito + Testcontainers（MySQL/Redis/RabbitMQ 真实环境集成测试）。

9. **配置文件环境隔离**：当前 application.yml 中 MySQL/Redis/RabbitMQ 连接信息写死（IP + 密码）。改为 application-dev.yml / application-prod.yml + 环境变量注入（如 `SPRING_DATASOURCE_URL`）。

10. **布隆过滤器迁移到 Redis Bitmap**：当实例数 > 10 时，各自维护布隆过滤器会放大数据库定时查询压力。改为 Redis Bitmap 统一维护一份。

11. **接入 CI/CD**：GitHub Actions 或 Jenkins，自动编译 + 测试 + 部署，减少人工操作。

**重构建议——渐进式而非推倒重来**：

总体架构是合理的——Spring Boot 单体 + Redis + RabbitMQ 这个组合对当前业务规模足够。不是"推倒重做"而是"补齐短板"。

优先顺序：P0 → 本周内修复分页性能 → P1 → 本月内补齐监控和限流 → P2 → 下季度逐步升级。

**追问预测**：

- *追问：如果让你把项目从单体拆成微服务，你会怎么拆？* → 不会拆。当前业务量没必要。微服务的好处（独立扩缩容、独立技术栈、团队自治）在 1-2 人团队、中等业务量的档案管理系统中用不上。微服务的代价（网络延迟、分布式事务、运维复杂度、调试难度）会远超收益。保持单体 + 清晰的模块边界，为未来拆分留好接口即可。
- *追问：你认为这个项目的技术深度在哪里？* → 不在"用了什么框架"，而在于"面对缓存击穿/穿透/雪崩这三个经典问题，设计了一套完整的、可验证的解决方案"。逻辑过期 + 分布式锁 + 双重检查 + 布隆过滤器的组合，每个技术点都不难，但组合在一起形成了一个健壮的缓存架构——这是工程设计的深度。性能测试能回溯到代码行号的测试方法论，也是区别于"凭感觉优化"的竞争力。
- *追问：如果让你带一个新同事了解这个项目，你会怎么安排？* → 第一天：跑通环境（改数据库连接 → 启动 → 测试数据生成 → 跑通所有接口）。第二天：读核心链路（AuthInterceptor → EthnicCultureService.getById → 缓存架构图 → MQ 链路）。第三天：跑性能测试 + 理解性能报告。第四天：修一个 P1 级别的 Bug（如分页性能问题），从排查到修复到验证。

**避坑提示**："项目不足"是最容易暴露代码质量的环节——面试官想知道你有没有批判性思维。不要只列举缺点，要每个缺点跟一个具体改进方案，要有优先级排序（P0/P1/P2），表现你能区分轻重缓急。

---

*文档生成日期: 2026-05-13*
*基于 v2.0 分支代码 + 性能测试报告综合分析*
