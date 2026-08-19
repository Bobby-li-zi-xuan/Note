# 面试问题整理

> 本文档基于项目实际代码整理，采用一问一答形式。

***

## 1. 数据库和Redis的数据一致性是怎么做的？

项目采用的是\*\*最终一致性（Eventual Consistency）\*\*方案，核心链路为：**主线程同步写（MySQL + Redis + MQ）→ 异步消费写MySQL**。

### 写入链路（以点赞为例）

对应代码：[EthnicCultureServiceImpl.like()](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/service/impl/EthnicCultureServiceImpl.java#L572)

1. **主线程**：先写MySQL点赞记录（`userFavoriteMapper.insert`），再用Redisson原子递增Redis计数器并设置过期时间，最后发送MQ消息
2. **消费线程**：`AsyncTaskServiceImpl`从RabbitMQ拉取消息 → 读Redis最新计数值 → 用条件更新（`.ne`）写回MySQL的`view_count/like_count/collect_count`字段

### 读取链路

对应代码：[EthnicCultureServiceImpl.getEthnicCultureById()](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/service/impl/EthnicCultureServiceImpl.java#L239)

1. 从Redis读取`CacheData<EthnicCulture>`包装类（包含数据和逻辑过期时间戳）
2. 调用`syncCountersToCulture()`用Redis实时计数值覆盖缓存对象中的计数字段
3. 若逻辑过期：返回旧数据的同时**异步刷新缓存**，用户无感知

### 一致性保障措施

| 策略        | 说明                                                                            |
| --------- | ----------------------------------------------------------------------------- |
| Redis原子操作 | 使用Redisson `RAtomicLong` 的 `incrementAndGet/decrementAndGet` 保证计数原子性          |
| 按需写DB     | `updateCounterIfChanged` 使用 `.ne(columnName, latestCount)` 条件，只在值变化时才执行UPDATE |
| 分布式锁      | 点赞/收藏使用 `lock:like:userId:cultureId` 防止并发重复操作                                 |
| 缓存计数器回写   | `syncCountersToCulture()` 每次读缓存时用Redis最新值覆盖，界面实时展示                            |

### 一致性程度说明

**不是强一致性**。数据库中的计数字段依赖MQ异步消费同步，存在秒级到分钟级延迟。但Redis中的计数始终实时，用户看到的展示数据是正确的。

***

## 2. 消息队列为什么选RabbitMQ？

### 选型原因

1. **路由灵活性**：项目需要将浏览、点赞、收藏三种操作精确分发到三个独立队列，RabbitMQ的`DirectExchange`天然支持按routingKey精确路由
2. **死信队列（DLX）开箱即用**：为每个业务队列绑定DLX，消费失败3次后自动转发到`.dlq`死信队列，运维可事后补偿
3. **手动ACK + 消息可靠性**：`basicGet(false)` + 手动ACK/NACK让消费端精确控制确认时机，配合`durable=true`持久化队列保证消息不丢失
4. **Spring AMQP无缝集成**：与Spring Boot的`spring-boot-starter-amqp`原生集成，`RabbitTemplate`一行代码即可发送消息
5. **消息体小**：消息体仅是一个`cultureId`字符串（如"123"），RabbitMQ在有消息确认和路由的场景下效率最优

### 对比RocketMQ/Kafka

| 维度     | RabbitMQ            | RocketMQ | Kafka         |
| ------ | ------------------- | -------- | ------------- |
| 路由能力   | 强（Exchange/Binding） | 中（Tag过滤） | 弱（仅partition） |
| DLX/死信 | 原生支持                | 需配置      | 无原生支持         |
| 事务消息   | 较弱                  | 强（分布式事务） | 较弱            |
| 适用场景   | 业务路由分发              | 交易核心链路   | 海量日志/流式       |
| 运维复杂度  | 低                   | 中        | 高             |

***

## 3. 消息队列的可靠性是怎么实现的？异常如何处理？如何保证消息不重复？

### 一、消息防丢失

**生产端**：`RabbitTemplate.convertAndSend` 基于AMQP协议，配合Spring的Confirm模式可确认消息到达Broker。

**Broker端**：所有队列和交换机均声明为持久化（[RabbitMQConfig.java](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/utils/MQ/RabbitMQConfig.java)）：

```java
new Queue(queueName, true, false, false, args);        // durable=true
new DirectExchange(EXCHANGE_DLX, true, false);          // durable=true
```

**消费端**：手动确认模式（[RabbitMessageQueue.java](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/utils/MQ/RabbitMessageQueue.java)）：

- `channel.basicGet(queueName, false)` —— `false`关闭自动ACK
- 成功 → `channel.basicAck(deliveryTag)` 确认消费
- 失败 → `channel.basicNack(deliveryTag, false, false)` 拒收且不重回队列

### 二、异常处理：有限重试（3次）→ 死信队列

对应代码：[MessageConsumeTask.java](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/utils/MQ/MessageConsumeTask.java)

```
消息处理失败
  ├─ 读取消息头 x-retry-count（默认0）
  ├─ retryCount < 3 → 重新发送原消息并递增retryCount → basicAck旧消息
  └─ retryCount >= 3 → 发送到死信交换机DLX → 进入.dlq队列 → basicAck旧消息
```

兜底：如果重试/死信转发本身也失败，执行`channel.basicNack(deliveryTag, false, false)`丢弃消息但保留日志追踪。

### 三、防重复

1. **业务幂等性**：点赞/收藏操作本身幂等——再次操作时数据库已有记录，直接抛`BusinessException("已经点赞过")`
2. **MessageConsumeTask防重复执行**：使用`AtomicBoolean started/completed`双重检查，防止同一Runnable被线程池重复调度
3. **计数器同步幂等**：`updateCounterIfChanged`使用`.ne(columnName, latestCount)`条件——如果值已是最新，UPDATE影响行数为0

***

## 4. 日志记录是怎么做的？AOP如何使用？

### AOP切面实现

对应代码：[OperationLogAspect.java](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/aspect/OperationLogAspect.java)

| 通知类型              | 切点                                                  | 职责                                       |
| ----------------- | --------------------------------------------------- | ---------------------------------------- |
| `@Before`         | `execution(* com.ethnicculture.controller.*.*(..))` | 记录请求开始时间（存入`ThreadLocal`）                |
| `@AfterReturning` | 同上                                                  | 计算耗时，记录成功日志（result=1），清理ThreadLocal      |
| `@AfterThrowing`  | 同上                                                  | 计算耗时，记录失败日志（result=0）+异常信息，清理ThreadLocal |

### AOP切面方法中提取的关键信息，用户信息也是通过Threadlocal中储存的用户上下文获得

```
saveOperationLog(joinPoint, duration, result, errorMsg)
  ├─ 用户身份 → UserContextHolder.currentOrNull() → userId/username
  ├─ 操作方法 → className + "." + methodName
  ├─ 请求参数 → 拼接为字符串（过滤HttpServletRequest等不可序列化参数）
  ├─ 执行耗时 → System.currentTimeMillis() - startTime
  ├─ 操作结果 → 1（成功）/ 0（失败）
  └─ 错误信息 → 仅失败时记录
```

### 事件解耦

AOP层不直接调用日志服务，而是发布`OperationLogEvent`事件，监听器监听到事件后调用异步线程池创建线程执行操作日志落库

```
Controller方法执行
  → @AfterReturning/@AfterThrowing
  → OperationLogEventPublisher.publish(event)
  → @EventListener(OperationLogEventListener)  // 异步监听
  → asyncTaskExecutor.execute(...)
  → OperationLogService.logOperation(...)      // 写入operation_log表
```

日志落库完全脱离HTTP主链路，用户响应不受影响。

### ThreadLocal防泄漏

`startTimeHolder`在`@AfterReturning`和`@AfterThrowing`中都执行了`remove()`——两个切点覆盖正常返回和异常退出，相当于finally的效果。

***

## 5. Redisson的实现与选型

### 一、为什么用Redisson？

> 对比直接用Redis原生SETNX实现分布式锁

- **WatchDog自动续期**：原生SETNX设置过期时间后，如果业务线程长耗时阻塞，锁会到期释放引发并发问题；Redisson内置"看门狗"机制，线程运行没结束会定时给锁续期
- **功能完备**：RLock直接实现`java.util.concurrent.locks.Lock`接口且为可重入锁，免去了SETNX复杂的代码缝合
- **额外能力**：支持红锁、读写锁、RAtomicLong（实现`java.util.concurrent.atomic.AtomicLong`接口）等

### 二、实现方式

项目配置`RedissonConfig`（[RedissonConfig.java](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/config/RedissonConfig.java)）装配单节点模式的`RedissonClient`，并封装两个工具类：

| 工具类                                                                                                                                                                | 底层API         | 功能                    |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------- | --------------------- |
| [RedissonDistributedLock](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/utils/RedissonDistributedLock.java) | `RLock`       | 可重入分布式锁，支持看门狗自动续期     |
| [RedissonCounter](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/utils/RedissonCounter.java)                 | `RAtomicLong` | 分布式原子计数器（递增/递减/设置/过期） |

### 三、分布式锁的具体使用

| 场景      | 锁键格式                                                 | 说明             |
| ------- | ---------------------------------------------------- | -------------- |
| 点赞/收藏操作 | `like:userId:cultureId`                              | 防止同一用户重复操作     |
| 缓存击穿保护  | `lock:culture:id`                                    | 只让一条线程重建缓存     |
| 列表缓存刷新  | `lock:culture:list:page:size:ethnicType:contentType` | 防止多线程同时刷新同条件列表 |

所有锁均使用`tryLock(waitTime, leaseTime)`模式：waitTime=3000ms等待3秒后放弃，leaseTime通过随机jitter（8-12秒）防止批量过期。

### 四、原子计数器的具体使用

```java
// 点赞 → Redis原子递增
redissonCounter.increment(LIKE_COUNT_KEY + cultureId);
// 取消点赞 → Redis原子递减
redissonCounter.decrement(LIKE_COUNT_KEY + cultureId);
// 回写MySQL前读取最新值
Long count = redissonCounter.get(VIEW_COUNT_KEY + cultureId);
```

使用`RAtomicLong`而非Redis原生`INCR`，因为它实现了Java标准接口，支持`exists()/expire()/set()`等附加操作，方便配合过期时间和缓存初始化。

***

## 6. 缓存问题是如何解决的？

项目针对缓存三大经典问题分别做了防护：

### 一、缓存穿透（查询不存在的数据）

**布隆过滤器**（[BloomFilterUtils.java](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/utils/BloomFilterUtils.java)）：

- 使用Guava的`BloomFilter<Long>`，**双过滤器交替**机制——`currentFilter`日常查询，`rebuildFilter`定时重建
- 每天凌晨2点（`@Scheduled(cron = "0 0 2 * * ?")`）从数据库`SELECT id WHERE status=1 AND deleted=0`重建过滤器
- 预估插入量10万条，误判率1%
- `mightContain(id)`返回false时直接返回null，不走Redis也不查MySQL

### 二、缓存击穿（热点key过期，大量请求冲击DB）

**Redisson分布式锁 + 逻辑过期**（[EthnicCultureServiceImpl.getEthnicCultureById()](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/service/impl/EthnicCultureServiceImpl.java#L239)）：

```
请求到达 → 读Redis缓存
  ├─ 缓存存在且未过期 → 直接返回
  ├─ 缓存存在但逻辑过期 → 返回旧数据 + asyncTaskExecutor异步刷新缓存
  └─ 缓存不存在 → Redisson.tryLock(lockKey, 3000ms)
       ├─ 获取锁成功 → 查DB → 写Redis → 释放锁 → 返回
       └─ 获取锁失败 → 读缓存（其他线程已重建）→ 返回
```

`CacheData`包装类携带逻辑过期时间戳（1小时），过期后不删除而是返回旧数据并异步刷新——确保缓存永不真正"空窗"。

### 三、缓存雪崩（大量key同时过期）

**过期时间随机打散**：

```java
// 计数器缓存过期：25-35分钟随机（基值30分钟 ± 5分钟jitter）
private long getCounterExpiration() {
    return COUNTER_EXPIRATION_BASE + (long)(Math.random() * (MAX - MIN));
}

// 分布式锁过期：8-12秒随机（基值10秒 ± 2秒jitter）
private long getLockExpiration() {
    return LOCK_EXPIRATION_BASE + (long)(Math.random() * (MAX - MIN));
}
```

每种缓存有独立随机范围，所有key的过期时间均匀分布，避免同一时刻大面积失效。

### 四、缓存预热

应用启动时通过`CacheWarmUpRunner`调用`warmUpCache()`，将访问量前3的热点数据预加载到缓存，避免冷启动时缓存为空导致大量请求直接打DB。

***

## 7. 权限校验是怎么实现的？为什么用JWT？

### 一、权限校验实现

对应代码：[AuthInterceptor.java](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/config/AuthInterceptor.java)

项目实现`AuthInterceptor`拦截器，基于`HandlerInterceptor.preHandle`拦截所有请求（白名单路径如`/auth/login`放行）。

**核心流程：**

```
请求到达 → AuthInterceptor.preHandle()
  ├─ 不是HandlerMethod → 直接放行
  ├─ resolveAndBindUserContext(request) → 统一解析Authorization头，构建UserContext
  │    ├─ 无Token → 构建匿名UserContext
  │    └─ 有Token → JwtTokenUtils.parseToken() → 解析userId/username/role
  ├─ 检查 @RequireLogin 注解
  │    └─ 需要登录但未认证 → throw BusinessException(401)
  └─ 检查 @RequireRole 注解
       └─ 角色不匹配 → throw BusinessException(403)
```

**关键设计：**

- 使用`@RequireLogin`和`@RequireRole`注解声明式鉴权，可标注在Controller类或方法上
- 统一入口解析构建`UserContext`，各层通过`UserContextHolder`获取，避免重复解析JWT
- 鉴权失败统一抛`BusinessException(401/403)`，由全局异常处理器输出`ResultDTO`格式

### 二、为什么用JWT不用Redis共享Session？

- **无状态化设计**：服务器不必在Redis留存所有用户的Session凭证。前端存储Token（LocalStorage/Vuex），每次通过Header带回自身，服务器只做密码学签名验签
- **节省内存**：无需为每个活跃用户维护服务端Session，显著降低内存开销
- **集群友好**：无状态架构天然支持水平扩展，新节点无需同步Session数据

***

## 8. 项目中线程池是如何设计的？

项目按职责分为**三个独立线程池**，形成"生产 → 队列缓冲 → 消费"的流水线架构。

对应代码：[ThreadPoolConfig.java](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/config/ThreadPoolConfig.java)

### 一、asyncTaskExecutor（生产端异步任务线程池）

| 参数    | 值                | 设计思路                             |
| ----- | ---------------- | -------------------------------- |
| 核心线程数 | CPU核心数 × 2       | IO密集型（Redis/MySQL/MQ），线程大部分时间等IO |
| 最大线程数 | CPU核心数 × 2       | 与核心相同，靠队列缓冲而非扩容，避免上下文切换          |
| 队列容量  | 最大线程数 × 10       | 有界队列防OOM，给突发流量缓冲                 |
| 拒绝策略  | CallerRunsPolicy | 任务不丢失，由调用方兜底执行                   |
| 线程名前缀 | async-task-      | 监控和Dump时识别                       |

**分工：** HTTP请求路径上的异步旁路任务——缓存异步刷新、浏览计数（Redis INCR + 发MQ）、浏览记录写入、操作日志写入。

### 二、consumerLoopExecutor（消费者轮询线程池）

| 参数    | 值              | 设计思路                    |
| ----- | -------------- | ----------------------- |
| 核心线程数 | 3（固定）          | 3个队列各占1个固定线程            |
| 最大线程数 | 3（固定）          | 不扩容，线程永不回收（keepAlive=0） |
| 队列容量  | 0              | 启动时一次性提交3个Runnable，无需缓冲 |
| 拒绝策略  | AbortPolicy    | 防御性编程，3线程对3任务永不触发       |
| 线程名前缀 | consumer-loop- | 标识为轮询消费线程               |

**分工：** 为每个RabbitMQ队列启动一个长生命周期轮询消费循环（while-loop）：basicGet拉取 → submitTask提交到messageConsumerExecutor。**与messageConsumerExecutor严格分离**——轮询线程只管拉取消息，不处理业务。

### 三、messageConsumerExecutor（消息处理线程池）

| 参数    | 值                     | 设计思路                                               |
| ----- | --------------------- | -------------------------------------------------- |
| 核心线程数 | Math.max(2, CPU核心数/2) | 下游消费不需要上游那么大并发                                     |
| 最大线程数 | 核心线程数 × 2             | 允许突发临时扩容                                           |
| 队列容量  | 核心线程数 × 10            | 有界队列防OOM                                           |
| 拒绝策略  | AbortPolicy           | 抛异常后由submitTaskWithRequeueSupport捕获，触发nack+requeue |
| 线程名前缀 | message-consumer-     | 标识为消息处理线程                                          |

**分工：** 执行MessageConsumeTask（读Redis计数器 → UPDATE MySQL数据库），包括浏览/点赞/收藏三种计数的持久化同步。

### 四、异常处理机制

- **线程异常**：三个线程池均配置自定义`ThreadFactory` + `UncaughtExceptionHandler`，线程因未捕获异常死亡时记录到日志而非静默丢弃
- **消费循环崩溃恢复**：每个循环外层`while(running) + try-catch(Throwable)`兜底，即使发生致命错误也5秒后自动重启
- **消息处理失败**：3次重试后转发到死信队列（DLX/DLQ），避免坏消息死循环
- **线程池拒绝**：messageConsumerExecutor被拒绝时，捕获`RejectedExecutionException`调用nack让RabbitMQ重新投递

***

## 9. 本轮重构优化（2026-04-23）做了什么？

### 9.1 建立统一请求上下文 UserContext

- 请求进入拦截器时统一解析`Authorization`头，一次性构建`UserContext`（含`userId/username/role/authenticated`）并绑定到请求上下文
- Controller和Aspect不再重复解析JWT，也不直接从`RequestContextHolder`手工读散落属性
- 统一入口解析避免"同一请求在不同层解析结果不一致"的问题

### 9.2 拦截器只做鉴权判定，失败抛标准异常

- 拦截器不再直接拼接JSON写响应
- 登录失效、权限不足统一抛`BusinessException(401/403)`，由全局异常处理器输出统一`ResultDTO`格式
- 好处：前后端错误协议稳定，异常处理逻辑集中

### 9.3 操作日志事件化

- Controller/AOP不再直接调用日志落库，改为发布`OperationLogEvent`
- 新增`OperationLogEventListener`异步消费事件并调用日志服务入库
- 主业务请求只负责发布事件，日志落库与主链路解耦，降低接口尾延迟风险

### 9.4 MQ失败处理接入死信队列

- 消费失败不再无限重试，改为3次有限重试后转发到DLX/DLQ
- 对应代码：[MessageConsumeTask.retryOrDeadLetter()](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/utils/MQ/MessageConsumeTask.java)

### 9.5 消费端Channel复用

- 从每次basicGet都创建/销毁Channel改为持久复用长连接Channel
- `ConcurrentHashMap<String, ChannelHolder>`缓存每个队列的Channel，失效时自动重建
- 对应代码：[RabbitMessageQueue.getOrCreateChannel()](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/utils/MQ/RabbitMessageQueue.java)

### 9.6 AOP ThreadLocal清理修正

- `startTimeHolder`清理逻辑覆盖`@AfterReturning`和`@AfterThrowing`两个切点，等价于finally块效果，确保ThreadLocal不泄漏

***

## 10. 并发写防护体系是怎样的？

项目采用\*\*"上游拦截、层层过滤"\*\*策略，能不在数据库层面解决的并发问题就在上游解决，最大程度降低数据库并发压力。

```
┌─────────────────────────────────────────────────────────────┐
│                    并发写库防护体系                           │
├─────────────────────────────────────────────────────────────┤
│  第一层：Redis原子计数器（RAtomicLong）                      │
│  → 高并发计数场景不直接写DB，避免数据库锁竞争                 │
│  → 详见问题1"数据一致性"和问题5"Redisson实现"                 │
├─────────────────────────────────────────────────────────────┤
│  第二层：RabbitMQ异步削峰                                    │
│  → 批量/延时写DB，降低数据库瞬时压力                          │
│  → 详见问题3"消息队列可靠性"                                 │
├─────────────────────────────────────────────────────────────┤
│  第三层：乐观锁（.ne条件更新）                                │
│  → 多消费者并发更新同一记录时，只有一个成功                    │
│  → 详见问题1".ne条件更新"                                    │
├─────────────────────────────────────────────────────────────┤
│  第四层：分布式锁（Redisson）                                 │
│  → 缓存重建、防重复操作等场景的互斥访问                        │
│  → 详见问题5"Redisson分布式锁"                               │
├─────────────────────────────────────────────────────────────┤
│  第五层：@Transactional事务                                  │
│  → 多表操作的原子性保证                                       │
├─────────────────────────────────────────────────────────────┤
│  第六层：线程池隔离                                          │
│  → 不同任务互不干扰，拒绝时优雅降级                           │
│  → 详见问题8"线程池设计"                                     │
└─────────────────────────────────────────────────────────────┘
```

### 各层关键代码位置

| 层级            | 关键代码                                                          | 文件                                                                                                                                                                                      |
| ------------- | ------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 第一层 Redis原子计数 | `redissonCounter.increment(countKey)`                         | [EthnicCultureServiceImpl.java](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/service/impl/EthnicCultureServiceImpl.java#L574)   |
| 第二层 MQ削峰      | `rabbitMessageQueue.sendMessage("likeCount", cultureId)`      | 同上 `#L600`                                                                                                                                                                              |
| 第三层 乐观锁       | `updateWrapper.ne(columnName, latestCount)`                   | [AsyncTaskServiceImpl.java](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/service/impl/AsyncTaskServiceImpl.java#L289)           |
| 第四层 分布式锁      | `distributedLock.tryLock(lockKey, 3000, getLockExpiration())` | [EthnicCultureServiceImpl.java](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/service/impl/EthnicCultureServiceImpl.java#L289)   |
| 第五层 事务        | `@Transactional`                                              | [EthnicCultureServiceImpl.like()](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/service/impl/EthnicCultureServiceImpl.java#L572) |
| 第六层 线程池隔离     | 三个独立线程池                                                       | [ThreadPoolConfig.java](file:///d:/Project/IdeaProject/ethnic-culture/ethnic-culture-backend/src/main/java/com/ethnicculture/config/ThreadPoolConfig.java)                              |

### 补充：幂等性设计

- **业务幂等**：点赞/收藏重复操作时DB已有记录，抛`BusinessException("已经点赞过")`
- **消息消费幂等**：`AtomicBoolean started/completed`双重检查，防止同一消息被重复处理
- **计数器同步幂等**：`.ne(columnName, latestCount)`条件更新，值已最新时UPDATE影响行数为0

