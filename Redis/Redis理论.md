### Redis(Remote Dictionary Server)远程词典服务器，是一个基于内存的键值型NoSQL数据库

## 特征：
- 键值型
- 单线程，每个命令具备原子性
- 低延迟，速度快（***基于内存***、IO多路复用、良好的编码）
- 支持数据持久化
- 支持主从集群、分片集群
- 支持多语言客户端


> [!NOTE] Redis是单线程但是快的原因
> 1. **IO多路复用**：利用一个线程同时监听多个文件的socket，只处理就绪事件，避免阻塞和轮询
> 2. 纯内存操作：所有的读写都在内存，内存读写的速度本身就快
> 3. 单线程避免了锁和上下文切换的开销
> 4. 高效的数据结构：ziplist、skiplist、hashtable等
> 5. 大多数命令的时间复杂度都短，单条执行速度快
> 6. Redis6+支持多线程IO（读写时多线程，命令执行仍然是单线程）

## 基本类型
#### String
 - 使用场景
	- 普通缓存、用户信息、配置常量
	- 计数器（点赞、阅读量、库存）
	- 分布式锁、全局唯一 ID
#### Hash
存储结构化对象：用户信息、商品信息
#### List
- 消息队列、简单延时队列
- 时间线信息流、朋友圈列表
- 排行榜简单时序列表、日志顺序存储
#### Set
- 去重场景：点赞用户、已读用户
- 交集 / 并集 / 差集：共同好友、兴趣标签匹配
- 随机抽奖、黑名单 / 白名单
#### SortedSet（ZSet）
- 各种排行榜：销量、积分、战力、热度榜
- 延时任务队列（score 存时间戳）
- 带权重的排序业务
## 数据结构
#### SkipList
>传统链表只有指向前后元素的指针，而跳表内部包含跨度不同的多级指针，可以跳跃查找链表中的元素
#### SortedSet（ZSet）
>是一个有序集合。存储的每个数据都包含 `element` 和 `score`两个值，根据每个 `element`的 `score`值排序
- **元素唯一性**
    集合内的元素（element）是唯一的，不能重复；但 `score` 可以重复。
- **排序特性**
    - 元素默认按 `score` **升序排列**，支持按 `score` 范围、排名范围查询。
    - 排序是**天然的、持久化的**，新增元素时自动插入到对应位置，无需手动排序。
- **底层结构**
    - 小数据量时：底层用 **ziplist**（压缩列表）存储，节省内存。
    - 大数据量时：自动切换为 **skiplist（跳表） + dict（字典）** 组合结构：
        - `dict`：存储 element → score` 的映射，实现 O (1) 时间复杂度的元素查询。
        - `skiplist`：存储元素的排序结构，实现 O (log n) 时间复杂度的范围查询和插入。

## 基于Redis实现共享session
- 分布式共享能力：Session 仅存储在单台服务器内存，多服务器部署时登录态 / 验证码无法共享；Redis 作为独立存储，所有服务器均可访问
- 性能与资源优化：Session 占用应用服务器内存，用户量大会导致内存溢出；Redis 是独立内存数据库，读写速度（微秒级）远快于服务器内存操作，且不占用应用服务器资源，支撑高并发
- 精细化过期控制：Session 过期时间全局统一，无法单独设置验证码（2 分钟）、Token（30 分钟）的有效期；Redis 可针对不同数据精准设置过期时间，且自动清理过期数据，无需手动维护
- 安全性与灵活性：Token 续期、注销等操作在 Redis 中可通过原子命令快速完成，且 Token 更难伪造
- 开发与维护成本低：Redis 原生支持过期、原子操作，无需额外写定时任务清理无效数据

## Spring提供的StringRedisTemplate类来操作Redis
- 对象由spring管理，自动注入，直接通过注解使用
```java
 @Resource  
private StringRedisTemplate stringRedisTemplate;
```
- 类和对象是自己手动新创建，用构造器注入
```java
private StringRedisTemplate stringRedisTemplate;  
  
//这个类的对象是手动创建的，没有spring帮助做依赖注入，不能使用@Resource或者@Autowired，必须手动创建构造函数注入  
public RefreshTokenInterceptor(StringRedisTemplate stringRedisTemplate) {  
    this.stringRedisTemplate = stringRedisTemplate;  
}
```
- 常用方法：
	- opsForValue ()：操作字符串
	- opsForList ()：操作列表
	- opsForSet ()：操作集合
	- opsForHash ()：操作哈希
	- opsForZSet ()：操作有序集合
```java
// 存值 
stringRedisTemplate.opsForValue().set("key", "value"); 
// 取值
String val = stringRedisTemplate.opsForValue().get("key"); 
// 设置过期时间 
stringRedisTemplate.opsForValue().set("key", "value", 30L, TimeUnit.MINUTES); 
// 删除 
stringRedisTemplate.delete("key");
```

## 缓存
> 数据交换的缓冲区，临时储存数据的地方，读写性能高
- 作用：
	- 降低后端负载
	- 提高读写效率，降低响应时间
- 成本
	- 数据一致性成本
	- 代码维护成本
	- 运维成本

#### 缓存更新策略
- 内存淘汰：不用自己维护，当内存不足时自动淘汰部分数据，下次查询时更新缓存。内存充足时数据无法及时更新，数据一致性差
- 超时剔除：给缓存数据添加TTL时间，到期后自动删除缓存。下次查询时更新缓存。可控制数据更新的时间，数据一致性一般，有一定的维护成本
- 主动更新：编写业务逻辑，每当修改数据库的同时，更新缓存，一致性好，维护成本较高
	- cache Aside Pattern：由缓存的调用者在更新数据库的同时更新缓存
	- Read/Write Through Pattern：缓存与数据库整合为一个服务，由服务来维护一致性。调用者调用该服务，无需关心缓存一致性的问题
	- Write Behind Caching Pattern：调用者只操作缓存，由其他线程异步的将缓存数据持久化到数据库，保证最终一致
- **最佳方案**
	- 低一致性需求：内存淘汰
	- 高一致性需求：主动更新
		- 读操作
			- 缓存命中则直接返回
			- 未命中则查询数据库，并写入缓存，设定超时时间
		- 写操作
			- 先写数据库，然后再删除缓存
			- 确保数据库与缓存操作的原子性
#### 缓存穿透
>指客户端请求的数据在缓存中和数据库中都不存在，这样缓存永远都不生效，这些请求都会传到数据库，高并发环境下可能导致数据库需要同时处理大量请求，使得服务反应超时
##### 常见解决方法
###### 缓存空对象
>数据库查询无结果时往缓存中存入空值，过期时间内有相同请求再次发起查询直接返回空值，精准拦截**同一无效 key**的重复请求
- 优点：实现简单，维护方便
- 缺点：有额外的内存消耗且可能造成缓存短期内不一致
- 应用场景：适合中小数据量、查询频率适中的场景
代码示例
```java
/**  
 * 方法3  
 * 根据指定的key拆查询缓存，并反序列化为指定类型，利用换缓存空对象解决缓存穿透问题  
 * @param keyPrefix 缓存的键的前缀  
 * @param id 缓存的业务实体的标识符  
 * @param type 返回值的类型  
 * @param dbFallback 数据库查询方法  
 * @param time 缓存的过期时间  
 * @param unit 时间单位  
 */  
public <R, ID> R queryWithPassThrough(String keyPrefix, ID id, Class<R> type, Function<ID, R> dbFallback, Long time, TimeUnit unit) {  
    //从redis查询商铺缓存  
    String key = keyPrefix + id;  
    //获取key为id对应的value值  
    String json = stringRedisTemplate.opsForValue().get(key);  
    //判断是否存在  
    if(StrUtil.isNotBlank(json)){  
        //存在则直接返回  
        return JSONUtil.toBean(json, type);  
    }  
    // 命中空值则直接返回空值
    if(json != null){  
        return null;  
    }  
  
    //不存在，根据id查询数据库  
    R r = dbFallback.apply(id);  
  
    //数据库中也不存在  
    if(r == null){  
        //将空值写入redis  
        stringRedisTemplate.opsForValue().set(key, "", CACHE_NULL_TTL, TimeUnit.MINUTES);  
        //返回错误信息  
        return null;  
    }  
    //存在，写入redis  
    this.set(key,r,CACHE_SHOP_TTL,TimeUnit.MINUTES);  
    //返回  
    return r;  
}
}
```

###### 布隆过滤器
>过滤掉一定不存在的数据请求，放进来可能存在的数据请求。底层是一个 **固定长度的二进制数组（位数组）**，初始状态下所有位都为 `0`。同时，它会关联 **k 个独立的哈希函数**
- 工作原理
	- 元素添加流程：当向布隆过滤器中添加一个元素 `X` 时：
		1. 用 `k` 个哈希函数分别对 `X` 进行哈希计算，得到 `k` 个不同的**哈希值**。
		2. 将这 `k` 个哈希值映射到位数组的 `k` 个对应下标位置。
		3. 将这 `k` 个下标位置的二进制位 **从 0 置为 1**。
	- 元素查询流程：当判断元素 `Y` 是否存在于集合中时：
		1. 用同样的 `k` 个哈希函数对 `Y` 计算，得到 `k` 个下标位置。
		2. 检查这 `k` 个下标对应的二进制位 **是否全部为 1**：
		    - **如果有任意一位为 0** → 元素 `Y` **一定不存在**于集合中（绝对准确）。
		    - **如果全部为 1** → 元素 `Y` **可能存在**于集合中（存在误判）。

> [!NOTE] 误判原因
> 误判的本质是 **哈希碰撞**：不同的元素经过哈希函数计算后，可能得到相同的下标组合，导致它们在位数组上的标记位完全重叠。此时查询未添加过的元素，会被误判为 “存在”。

- 优点：内存占用少，查询速度块
- 缺点：实现复杂，有误判的可能，依然有可能出现缓存穿透，只能储存哈希标记，不能存储具体的元素值
- **优化方案**：
	1. 增大位数组长度
	2. 根据位数组长度和预期数据量设置合理的哈希函数个数
	3. 预期数据量要有冗余，防止位数组占用率过高
	4. 定期重建布隆过滤器
- 应用场景：数据量大、对内存敏感、误判可接受
###### 其他解决办法
- 增强id的复杂度，避免被猜测规律
- 做好数据的基础格式校验
- 加强用户权限校验
- 做好热点参数的限流
#### 缓存雪崩
>同一时段内大量的缓存key同时失效或者redis服务宕机，导致大量请求直接到达数据库，使得数据压力剧增
##### 解决方案
- 给不同的key的***TTL***（Time To Live存活时间）添加随机值
- 利用**redis集群**提高服务的可用性
- 给缓存业务添加**降级限流**策略
- 给业务添加**多级缓存**
#### 缓存击穿
>也叫热点key问题，就是一个**被高并发访问**并且缓存**重建业务较复杂**的key突然失效（过期了）了，大量的请求访问会在瞬间给数据库带来巨大冲击
##### 常见解决方案
###### 互斥锁
>一次只让一个线程做缓存查询
- 优点
	- 内存占用小
	- 保证数据一致性
	- 实现简单
- 缺点
	- 线程需要等待，性能受到影响
	- 可能出现死锁
- 定义锁
```java
//获取锁  
private boolean tryLock(String key){  
// 使用Redis的setIfAbsent方法尝试获取锁，如果key不存在则设置key的值为"1"，并设置过期时间  
// 参数说明：  
// 1. key: 锁的键值  
// 2. "1": 锁的值，可以是任意值，这里简单使用"1"  
// 3. LOCK_SHOP_TTL: 锁的过期时间  
// 4. TimeUnit.SECONDS: 时间单位为秒  
    Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(key, "1", LOCK_SHOP_TTL, TimeUnit.SECONDS);  
// 使用BooleanUtil.isTrue确保返回值是boolean类型，避免可能的null值  
    return BooleanUtil.isTrue(flag);  
}  
  
//释放锁  
private void unLock(String key){  
    stringRedisTemplate.delete(key);  
}
```
- 业务逻辑
```java
public Shop queryWithMutex(Long id){  
    //从redis查询商铺缓存  
    String key = CACHE_SHOP_KEY + id;  
    //获取key为id对应的value值  
    String shopJson = stringRedisTemplate.opsForValue().get(CACHE_SHOP_KEY + id);  
    //判断是否存在  
    if(StrUtil.isNotBlank(shopJson)){  
        //存在则直接返回  
        Shop shop = JSONUtil.toBean(shopJson, Shop.class);;  
        return null;  
    }  
    // 命中空值则直接返回  
    if(shopJson != null){  
        return null;  
    }  
    //实现缓存重建  
    //获取互斥锁  
    String lockKey = LOCK_SHOP_KEY + id;  
    Shop shop = null;  
    try {  
        boolean islock = tryLock(lockKey);  
        //判断是否获取成功  
        if (!islock) {  
            //失败，则休眠一段时间后并重试，避免高并发
            Thread.sleep(50);  
            return queryWithMutex(id);  
        }  
        //成功，根据id查询数据库  
        shop = getById(id);  
        //模拟重建的延迟  
        Thread.sleep(200);  
        //数据库中也不存在  
        if (shop == null) {  
            //将空值写入redis  
            stringRedisTemplate.opsForValue().set(key, "", CACHE_NULL_TTL, TimeUnit.MINUTES);  
            //返回错误信息  
            return null;  
        }  
        //存在，写入redis  
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(shop), CACHE_SHOP_TTL, TimeUnit.MINUTES);  
  
    }catch(InterruptedException e){  
        throw new RuntimeException(e);  
    }  
    //释放互斥锁  
    unLock(lockKey);  
    //返回  
    return shop;  
}
```
###### 逻辑过期
>不设置 Redis key 的物理过期时间（TTL），而是在缓存的 value 中嵌入「过期时间字段」，业务代码判断该字段是否过期，过期则异步更新缓存（缓存过期时，仅一个线程更新缓存，其他线程返回旧数据，无请求阻塞），但需要允许数据短暂不一致
- 优点
	- 线程无需等待，性能好
- 缺点
	- 不保证数据一致
	- 有额外内存消耗
	- 实现复杂
- 业务逻辑
	- 在RedisData类中封装逻辑过期属性
```java
public class RedisData {  
    private LocalDateTime expireTime;       //逻辑过期时间  
    private Object data;  
}
```
- 创建线程池减少创建销毁线程的开销  
```java
private static final ExecutorService CACHE_REBUILD_EXECUTOR = Executors.newFixedThreadPool(10);
```
- 实现
```java
/**  
     * 方法4  
     * 根据指定的key拆查询缓存，并反序列化为指定类型，利用逻辑过期解决缓存击穿问题  
     * @param keyPrefix 缓存的键的前缀  
     * @param id 缓存的业务实体的标识符  
     * @param type 返回值的类型  
     * @param dbFallback 数据库查询方法  
     * @param time 缓存的过期时间  
     * @param unit 时间单位  
     */  
    //封装逻辑过期解决缓存击穿的方法  
    public <R, ID> R queryWithLogicalExpire(String keyPrefix,ID id, Class<R> type, Function<ID, R> dbFallback,Long time,TimeUnit unit){  
        //从redis查询指定业务实体的缓存  
        String key = keyPrefix + id;  
        //获取key为id对应的value值  
        String json = stringRedisTemplate.opsForValue().get(key);  
        //判断是否存在  
        if(StrUtil.isBlank(json)){  
            //不存在则直接返回空  
            return null;  
        }  
        //命中，将json反序列化为对象  
        RedisData redisData = JSONUtil.toBean(json, RedisData.class);  
        JSONObject data = (JSONObject) redisData.getData();  
        //从返回的JSON信息中取出实体信息  
        R r = JSONUtil.toBean(data, type);  
        LocalDateTime expireTime = redisData.getExpireTime();  
        //判断是否过期  
        if(expireTime.isAfter(LocalDateTime.now())){  
            //未过期直接返回  
            return r;  
        }  
        //过期，尝试获取互斥锁  
        String lockKey = LOCK_SHOP_KEY + id;  
        boolean isLock = tryLock(lockKey);  
        //判断是否获取锁成功，成功后应该再次检测redis缓存是否过期，DoubleCheck，如果存在则无需重建缓存  
        if(isLock) {  
            // --- Double Check 开始 ---//        
            //例如有100个请求，如果这 100 个请求是串行到达的（例如，锁的释放时间很短，或者请求之间的间隔很短），那么每个请求都可能成功获取到锁（一个请求重建完缓存后释放锁，另外一个请求马上又获取到锁）。  
			//结果：100 个请求都会去查询数据库，并尝试重建缓存。这完全违背了使用互斥锁来防止缓存击穿的初衷，会对数据库造成巨大的压力。  
            // 再次从 Redis 获取数据，检查缓存是否已被其他线程重建  
            String shopJsonDoubleCheck = stringRedisTemplate.opsForValue().get(CACHE_SHOP_KEY + id);  
            // 如果 Redis 中已有数据，说明其他线程已经完成了缓存重建，当前线程无需再重建  
            if (StrUtil.isNotBlank(shopJsonDoubleCheck)) {  
                // 释放锁  
                unLock(lockKey);  
                // 直接返回旧数据（因为逻辑过期策略下，旧数据也是可用的）  
                return r;  
            }  
            //获取锁成功，开启独立线程执行缓存重建过程  
            CACHE_REBUILD_EXECUTOR.submit(() -> {  
                try{  
                    //查询数据库  
                    R r1 = dbFallback.apply(id);  
                    //将数据写入redis  
                    this.setWithLogicalExpire(key, r1, time, unit);  
  
                }catch(Exception e){  
                    throw new RuntimeException(e);  
                }finally{  
                    //释放锁  
                    unLock(lockKey);  
                }  
            });  
        }  
        //返回信息  
        return r;  
    }
```

## 全局唯一ID
#### 全局ID生成器
>在分布式系统下用来生成全局唯一ID的工具
- 特性：唯一性、高可用、高性能、递增性

## 主从
#### 主从集群结构
![[Pasted image 20260312214244.png]]
如图所示，集群中有一个master主节点、两个slave从节点（现在叫replica）。当我们通过Redis的Java客户端访问主从集群时，应该做好路由：
- 如果是写操作，应该访问master节点，master会自动将数据同步给两个slave节点
- 如果是读操作，建议访问各个slave节点，从而分担并发压力
#### 主从同步原理
###### 全量同步
>主从第一次建立连接时，会执行全量同步，将master结点的所有数据都拷贝给slave结点
- 全量同步流程
	- `slave`请求数据同步
	- `master`判断是否是第一次数据同步
	- 是第一次则返回 `master`的数据版本信息
	- `master`执行bgsave，生成RDB文件，同时记录RDB期间的所有命令到复制积压缓冲区
	- `master`向 `slave`发送RDB文件
	- `slave`清空本地数据，加载RDB文件
	- `master`在slave加载完RDB文件后，向`slave`发送repl_backlog中的命令
	- `slave`执行收到的命令

> [!NOTE]
> - **RDB(redis database)**：是Redis的持久化机制之一，核心是生成内存数据的快照文件，将某一时刻的全量数据以二进制形式保存到磁盘
> - **bgsave（background save，即后台保存）**：生成RDB快照的命令
> - **repl_backlog**：复制积压缓冲区，在RDB生成、传输、加载的过程中，主节点把所有新写的命令都缓存到repl_backlog内

- `master`判断`slave`是否是第一次同步的流程
	 - `slave`节点请求增量同步
	- `master`**节点判断`replid`**，发现不一致，拒绝增量同步
	- `master`将完整内存数据生成`RDB`，发送`RDB`到`slave`
	- `slave`清空本地数据，加载`master`的`RDB`
	- `master`将`RDB`期间的命令记录在`repl_baklog`，并持续将log中的命令发送给`slave`
	- `slave`执行接收到的命令，保持与`master`之间的同步

> [!Replication Id]
> 简称`replid`，是数据集的标记，replid一致则是同一数据集。每个`master`都有唯一的`replid`，`slave`则会继承`master`节点的`replid`

###### 增量同步
>只更新 `slave`和 `master`存在差异部分的数据， 除了第一次建立连接时建立全量同步，其他时候大部分 `slave`与 `master`都是做增量同步
- 流程：
	- `slave`携带replid和offset向`master`请求同步
	- `master`判断replid是否一致，再判断`slave`的offset是否在有效范围内
	- 不是第一次请求同步，`master`回复+continue
	- `master`去repl_baklog中获取offset中的增量数据
	- `master`发送offset后的命令
	- `slave`执行增量命令，追上主节点

> [!offset]
> - **`offset`**：偏移量（从节点单独记录），随着记录在`repl_backlog`中的数据增多而逐渐增大。`slave`完成同步时也会记录当前同步的`offset`。如果`slave`的`offset`在`master`在repl_backlog中记录的范围，说明`slave`数据落后于`master`，需要更新。

- `master`通过`repl_backlog`来获取自己与 `slave`数据的差异：主从正常同步时，主节点除了把写命令发给从节点，还会同时写入 `repl_backlog`；若从节点短暂断线（如网络抖动），重连后不会直接触发全量同步，而是先向主节点发送自己**已同步到的偏移量（offset）**：
	- 如果该偏移量在 `repl_backlog` 的覆盖范围内 → 主节点从该偏移量开始，把缓冲区里的命令发给从节点（增量同步）；
	- 如果该偏移量已经失效（`master`在 `slave`断线期间持续写入新数据，**覆盖了**从节点断线时的偏移量） → 才会触发全量同步
	
> [! repl_backlog]
> （复制积压缓冲区）：是 Redis 主节点上的一块**固定大小的环形内存缓冲区**，核心作用是存储主节点近期的写命令，为「增量同步」和「断线重连」提供数据支撑，避免主从节点断线后重新全量同步
> 
> - 特性：
> 	- 固定大小：由配置项 `repl-backlog-size` 设定（默认 1MB，生产建议根据业务调整，如 100MB~1GB）；
> 	- 循环覆盖：缓冲区写满后，新命令会覆盖最旧的命令；
> 	- 记录偏移量：每个写入的命令都有唯一的偏移量（offset），主节点记录缓冲区的「起始偏移量」和「结束偏移量」，从节点记录自己已同步到的偏移量。

###### 主从同步优化
- 在master中配置`repl-diskless-sync:yes`启用无磁盘复制，避免全量同步时的磁盘IO。
- Redis单节点上的内存占用不要太大，减少RDB导致的过多磁盘IO
- 适当提高`repl_backlog`的大小，发现slave宕机时尽快实现故障恢复，尽可能避免全量同步
- 限制一个master上的slave节点数量，如果实在是太多slave，则可以采用`主-从-从`链式结构，减少master压力
### 哨兵
>哨兵（Sentinel）机制来监控主从集群监控状态，确保集群的高可用性
- 作用
	- 状态监控：不断检查 `master`和 `slave`是否按预期工作
	- 故障恢复（`failover`）：当 `master`故障时，哨兵会将一个 `slave`提升为 `master`。当故障恢复后再退回为 `slave`
		- 选择依据：
			- 首先会判断slave节点与master节点断开时间长短，如果超过`down-after-milliseconds * 10`则会排除该slave节点
			- 然后判断slave节点的`slave-priority`值，越小优先级越高，如果是0则永不参与选举（默认都是1）。
			- 如果`slave-prority`一样，则判断slave节点的`offset`值，越大说明数据越新，优先级越高
			- 最后是判断slave节点的`run_id`（即replid）大小，越小优先级越高（`通过info server可以查看run_id`）。
		- 进行流程
			- 在多个`sentinel`中选举一个`leader`
			- 由`leader`执行`failover`
				- **选举leader**：
					- 要成为`leader`要满足两个条件：
						- 最先获得超过半数的投票
						- 获得的投票数不小于`quorum`值
					- 而sentinel投票的原则有两条：
						- 优先投票给目前得票最多的 
						- 如果目前没有任何节点的票，就投给自己
					- 故而第一个发现 `master`客观下线的会立刻发起投票，先投票的一定会成为leader
	- 状态通知：当集群发生故障恢复时，会将最新的集群消息推送给redis的客户端
#### 状态同步
>哨兵基于心跳机制监测服务状态，每隔一秒向集群的每个节点发送ping命令，并通过实例的响应结果来做出判断
- 主观下线：某个哨兵发现某个Redis节点在规定时间未响应
- 客观下线：超过指定数量（通过 `quorum`设置）的哨兵都认为该节点主观下线，则该节点客观下线。`quorum`的数量最好超过哨兵节点数量的一半，哨兵节点的数量至少三台
### Redis分片集群(Cluster)
![[Pasted image 20260313122207.png]]
- 特征：
	- 集群中有多个 `master`，每个 `master`保存不同分片数据 ，解决海量数据存储问题
	- 每个 `master` 都可以有多个 `slave` 节点 ，确保高可用
	- `master` 之间通过ping监测彼此健康状态 ，类似哨兵作用
	- 客户端请求可以访问集群任意节点，最终都会被转发到数据所在节点
- 核心原理：哈希槽（Hash Slot）分片机制
>集群预分配了16384个哈希槽（2KB空间），每个主节点管理一部分哈希槽，对任意key执行 `key%16384`取余，计算出该key归属的哈希槽。客户端连接任意节点即可获取全集群槽位映射表，后续请求直接根据计算结果路由到目标节点。
## Redis内存回收
#### 缓存删除
>数据库中的数据发送更新或删除操作时，必须主动删除对应的缓存，防止缓存中数据与数据库中不一致
- 两种主流方案，优劣和适用场景不同：

| 方案             | 优点                     | 缺点                                                           | 适用场景                           |
| -------------- | ---------------------- | ------------------------------------------------------------ | ------------------------------ |
| **先删缓存，再更数据库** | 实现简单，不会出现**永久脏数据**     | 存在**短暂缓存缺失**：并发场景下，步骤 1 后、步骤 2 完成前，有请求读取数据会直接查库并回写缓存，可能写入旧数据 | 并发量较低的业务场景（可通过延迟双删解决并发脏数据问题）   |
| **先更数据库，再删缓存** | 避免先删缓存的并发脏数据问题，工业界主流方案 | 存在**极小概率脏数据**：若步骤 2 删除缓存失败，会导致缓存长期脏数据                        | 高并发核心业务场景（需配合**重试机制**解决删除失败问题） |

> [!NOTE]
> 如何解决先更库再删缓存的删除失败问题？
> 1. **重试机制**：删除缓存失败时，将操作写入消息队列（如 RabbitMQ），由消费者重试删除，直到成功。
> 2. **定时任务兜底**：定期全量比对缓存与数据库数据，删除不一致的脏缓存。

- 针对长期不访问的**冷数据**或业务下线后的**冗余缓存**，需要手动删除
	- 常用命令：
		- DEL：删除单个key（原子操作）
		- UNLINK ：异步删除
		- FLUSHDB：清空当前数据库
		- FLUSHALL：清空所有数据库

> [!NOTE] 
> 删除大key为什么要用UNLINK？
> DEL是同步命令，UNLINK是异步命令，该命令会将大key放入后台任务队列，由子线程删除，不会阻塞主线程

#### 内存过期
>存入redis中的数据可以设置过期时间
###### 过期策略
>redis中有两个dict，也就是hashtable，其中一个记录键值对，另一个记录key和过期时间。要判断一个key是否过期只需要到记录过期时间的dict中根据key值查询
###### 删除策略
- **惰性删除**：不主动扫描和删除过期键，只在客户端访问该键时才检查是否过期，过期则立即删除并返回空值
	- **性能开销低**：无需主动遍历所有键，避免了大量的 CPU 消耗，对 Redis 主线程的影响极小，适合高并发场景。
	- **空间精准回收**：只有在键被访问时才触发删除，不会误删未过期的键，也不会遗漏对当前访问键的回收。
	- **内存浪费风险**：大量过期键长期不被访问时，会一直占用内存，导致内存利用率降低，甚至可能触发内存溢出（OOM）。
- **定期删除**：周期性地抽样部分过期的键然后删除（Redis默认策略）
	- slow模式：设置一个定时任务 `serverCron()`,按照 `server.hz`的频率来执行清理
	- fast模式：每个事件循环前执行过期key清理
- **定时删除**：key过期后立马删除
	- 内存最干净
	- 大量键过期时会占用大量CPU，压力大
#### 内存淘汰
>设置内存告警阈值，当内存使用达到阈值时就会主动挑选部分key删除以释放内存，每次执行任何命令时都会判断内存是否达到阈值
###### 内存淘汰策略
- `noeviction`： 不淘汰任何key，但是内存满时不允许写入新数据，默认就是这种策略。
- `volatile``-ttl`： 对设置了TTL的key，比较key的剩余TTL值，TTL越小越先被淘汰
- `allkeys``-random`：对全体key ，随机进行淘汰。也就是直接从db->dict中随机挑选
- `volatile-random`：对设置了TTL的key ，随机进行淘汰。也就是从db->expires中随机挑选。
- `allkeys-lru`： 对全体key，基于LRU算法进行淘汰
- `volatile-lru`： 对设置了TTL的key，基于LRU算法进行淘汰
- `allkeys-lfu`： 对全体key，基于LFU算法进行淘汰
- `volatile-lfu`： 对设置了TTL的key，基于LFU算法进行淘汰

> [!LRU和LFU]
>  - **LRU**（**`L`**`east` **`R`**`ecently` **`U`**`sed`），最近最久未使用。用当前时间减去最后一次访问时间，这个值越大则淘汰优先级越高。
> 	 - ，Redis会在Key的头信息中使用24个bit记录每个key的最近一次使用的时间`lru`。每次需要内存淘汰时，就会**抽样**一部分KEY，找出其中空闲时间最长的，也就是`now - lru`结果最大的，然后将其删除。如果内存依然不足，就重复这个过程。
>  -  **LFU**（**`L`**`east` **`F`**`requently` **`U`**`sed`），最少频率使用。会统计每个key的访问频率，值越小淘汰优先级越高。
> 	 - Redis会在key的头信息中使用24bit记录最近一次使用时间和逻辑访问频率。其中高16位是以分钟为单位的最近访问时间，后8位是逻辑访问次数。与LFU类似，每次需要内存淘汰时，就会**抽样**一部分KEY，找出其中逻辑访问次数最小的，将其淘汰。
> - **逻辑访问次数的计算**：会根据当前的访问次数做计算，结果要么是次数`+1`，要么是次数不变。但随着当前访问次数越大，`+1`的概率也会越低，并且最大值不超过255，逻辑访问次数还有一个**衰减周期**，默认为1分钟，即每隔1分钟逻辑访问次数会`-1`。这样逻辑访问次数就能基本反映出一个`key`的访问热度了
## Redis持久化策略
>将内存中的数据持久化到磁盘的机制，核心目标是防止数据丢失
- 作用：
	- redis重启后，加载持久化文件恢复数据
	- 持久化文件可以复制到其他机器，用于数据迁移或灾备
	- 支撑主从同步的全量同步阶段
#### RDB
- （Redis Database）快照持久化：在指定时间间隔内，检测到指定次数的写操作后，自动生成内存数据的**二进制快照文件**
- 优点：
	- 二进制文件体积小，数据恢复速度快（比 AOF 快 10 倍以上）
	- 适合全量数据备份、灾备、主从同步
	- 对主线程性能影响小（异步执行）
- 缺点
	- 数据安全性低：两次快照间隔内，若 Redis 宕机，期间的写数据会丢失
	- `fork` 子进程消耗 CPU 资源，数据量越大，`fork` 耗时越长，可能导致主线程短暂卡顿
	- 无法实现实时持久化（两次快照期间又有新数据更新），不适合对数据一致性要求高的场景
#### AOF
>（Append Only File）日志持久化，以**文本协议格式**记录redis执行的所有写命令，重启时通过**重放日志**恢复数据
- **AOF重写**：AOF 记录所有写命令，长期运行后文件会越来越大（如多次修改同一个 key，会记录多条命令），导致恢复速度变慢。
	- 解决方式：Redis 会 `fork` 子进程，遍历内存数据，生成**重建当前数据的最小命令集**（如将 100 次 `INCR key` 合并为 `SET key 100`），替换旧 AOF 文件。
- 优点：
	- 数据安全性高：可配置 `everysec` 刷盘，最多丢失 1 秒数据
	- 日志文件是文本格式，可读性强，可手动修复错误命令
	- 支持重写，避免文件无限膨胀
- 缺点：
	- AOF 文件体积比 RDB 大，数据恢复速度慢
	- 写操作会产生额外 IO 开销，性能略低于 RDB
	- 重写时 `fork` 子进程同样会消耗 CPU 资源

> [!fork()操作]
> Redis 是**单线程模型**，主线程负责处理所有客户端的读写请求。如果直接由主线程执行持久化（如 `save` 命令），会阻塞所有请求，严重影响性能。
> `fork` 的作用就是**创建一个与主线程共享内存数据的子进程**，由子进程专门负责生成 RDB 文件或重写 AOF 文件，主线程则继续处理客户端请求，实现**异步持久化**

#### 混合持久化
>(Redis4.0+新特性）上述两种持久化方案的结合，解决了RDB数据丢失多，AOF恢复慢的问题
- 开启混合持久化后，**AOF 重写时**，子进程会：
    1. 生成当前数据的 RDB 格式数据，写入新 AOF 文件开头。
    2. 将重写过程中产生的新写命令，以 AOF 格式追加到新 AOF 文件末尾。
- 最终 AOF 文件结构：`RDB 数据 + AOF 增量命令`。
- 数据恢复流程
	1. 若开启 AOF，优先加载 AOF 文件恢复数据（AOF 数据完整性更高）。
	2. 若未开启 AOF，加载 RDB 文件恢复数据。
	3. 若两者都未开启，启动后为空白数据库。


> [!NOTE]
> 面试追问：如果 RDB 和 AOF 文件都存在，为什么优先加载 AOF？
> 答：因为 AOF 记录的是实时写命令，数据完整性比 RDB 高，丢失的数据更少。

#### 持久化方式的选择
- **追求极致性能 + 可容忍少量数据丢失**：仅开启 RDB，适合缓存场景（如会话缓存）。
- **数据安全性优先 + 可接受略低性能**：开启 AOF，刷盘策略配置 `everysec`，适合核心业务数据（如订单、库存）。
- **平衡性能与安全性（推荐）**：开启 **混合持久化**，兼顾 RDB 恢复速度快和 AOF 数据完整性高的优点。
- **禁用持久化**：仅适用于纯缓存场景（如临时计算数据），Redis 重启后数据可从数据库重新加载。
## Redis实现分布式锁
>分布式系统并发控制的核心方案，用于解决多进程竞争同一资源的问题
- 核心特性：
	- 互斥性：同一时刻只能有一个客户端持有锁，避免并发冲突
	- 安全性：锁只能被持有它的客户端释放
	- 死锁避免：持有锁的客户端崩溃锁也能自动释放
	- 高可用：redis节点故障时，锁服务依然可用（需要集群）
#### 基础实现：单机redis锁
- **核心命令 SET NX PX**（SET if not exists)
>由于redis是单线程的，用了这个命令之后，只能有一个客户端对某一个key设置值。在没有过期或删除key的时候，其他客户端是不能设置这个key的。
```
# 加锁：key=锁名，value=唯一标识（如 UUID+线程ID），NX=不存在才设置，PX=过期时间（毫秒） SET lock_key unique_value NX PX 30000
```

- `NX`：保证互斥性，只有 key 不存在时才会设置成功，返回 `OK`；否则返回 `nil`（加锁失败）。
- `PX 30000`：设置锁的自动过期时间（30 秒），避免客户端崩溃导致死锁。
- **`unique_value`**：—— 必须用客户端唯一标识，不能用固定值！目的是防止**误删**其他客户端的锁。解锁的时候，要先判断锁的 `unique_value` 是否为加锁客户端，是才将 `lock_key` 键删除。
<br>
- Lua解锁（原子操作）
>redis会将整个Lua脚本作为一个整体执行，不允许中间插入其他命令，避免**误删锁**问题
- 实现原理：redis使用单线程执行命令，在执行Lua脚本时，redis会阻塞其他所有命令，防止命令被打断

> [!NOTE] 
>  为什么不用 `SETNX + EXPIRE` 实现？
核心原因是 **两步操作非原子**：
>- 客户端执行 `SETNX lock_key unique_value` 成功。
>- 执行 `EXPIRE lock_key 30` 前，客户端崩溃，锁没有过期时间，导致死锁。
>- 而 `SET NX PX` 是原子命令，一步完成设置和过期时间，避免了这个问题。

#### Redisson解决锁超时问题
>看门狗机制：客户端加锁成功后，启动一个后台线程，定期检查锁是否还持有，若持有则将过期时间重置为初始值
- 注意事项：
	- 后台线程必须与业务逻辑同生命周期，业务执行完后要停止续约，避免无效续约。
	- 续约频率必须小于锁过期时间的 1/3（如 30 秒过期，续约频率 10 秒），防止续约不及时导致锁过期。
	- 为了避免死锁的产生，Redisson实现的分布式锁是可重入的
#### Redlock解决单点故障（主从一致性问题）
>要求 **部署N个独立的Redis节点**，客户端向所有节点发起加锁请求，只有当 **超过半数**节点加锁成功，且总耗时 **不超过锁过期时间的一半**，才算加锁成功
- 优点：
	- **不依赖主从复制**，避免主从切换导致锁丢失，解决了主从一致性的问题
	- - 解决普通单机 Redis 锁问题：**单节点挂了直接丢锁、锁失效**。
	- 红锁通过**多数派机制**，降低单点故障带来的锁失效风险，提高可靠性。
- 缺点
	- 性能开销大
	- 对时钟同步敏感，节点时钟漂移可能导致锁过期时间不一致
#### 与zookeeper分布式锁对比

|特性|Redis 分布式锁|Zookeeper 分布式锁|
|---|---|---|
|实现原理|基于 `SET NX PX` 和 Lua 脚本|基于临时有序节点，利用 Watch 机制监听节点变化|
|锁失效机制|依赖过期时间，可能出现锁超时|客户端宕机后临时节点自动删除，无死锁风险|
|性能|高，Redis 是内存数据库，命令执行快|较低，Zookeeper 需维护节点状态和 Watch 机制|
|高可用|需 Redlock 或集群，实现复杂|天然支持集群，基于 Zab 协议，高可用易实现|
|适用场景|高并发、短持有时间的场景（如库存扣减）|低并发、长持有时间的场景（如分布式任务调度）|

## Redis和数据库的数据一致性
1. 高并发下先更新数据库再删缓存，通过重试机制防止缓存脏数据
2. 低并发可以先删缓存再更新数据库，通过延迟双删解决并发读把旧数据写回缓存的问题
3. 强一致性：读写时加上分布式锁，牺牲性能来彻底杜绝并发不一致
4. 设置缓存过期时间，同时设置定时任务对比数据库与redis