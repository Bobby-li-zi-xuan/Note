# 民族文化数字档案管理平台

> 面向民族文化保护、展示、收藏、分享的后台管理 + 前端展示系统，支持高并发访问、分布式控制、数据安全与高效缓存

---

## 一、项目概述

### 1.1 项目背景

民族文化数字档案管理平台是一个面向民族文化保护的综合性系统，旨在实现民族文化资源的数字化存储、展示、收藏和分享。系统采用前后端分离架构，后端基于 Spring Boot 构建，提供 RESTful API 服务。

### 1.2 核心功能

| 功能模块 | 功能描述 |
|----------|----------|
| **民族文化管理** | 民族文化内容的增删改查、审核管理 |
| **用户认证** | 用户注册、登录、Token刷新、权限控制 |
| **互动功能** | 点赞、收藏、浏览记录 |
| **搜索功能** | 关键词搜索民族文化内容 |
| **操作日志** | 用户操作行为记录与审计 |
| **缓存系统** | 多级缓存、防穿透/击穿/雪崩 |

### 1.3 技术选型概览

| 层次 | 技术栈 | 版本 |
|------|--------|------|
| 后端框架 | Spring Boot | 2.7.15 |
| 持久层 | MyBatis-Plus | 3.5.3.1 |
| 数据库 | MySQL | 8.0.33 |
| 缓存 | Redis + Redisson | 3.24.3 |
| 消息队列 | RabbitMQ | - |
| 认证 | JWT (JJWT) | 0.11.5 |
| 工具库 | Lombok, Guava | - |

---

## 二、技术栈详解与选型理由

### 2.1 核心技术框架

#### 2.1.1 Spring Boot 2.7.15

| 维度 | 说明 |
|------|------|
| **技术作用** | 企业级Java应用开发框架，提供自动配置、内嵌容器、生产就绪特性 |
| **选型理由** | 成熟稳定的LTS版本，经过大量企业验证，生态完善，开发效率高 |
| **项目中的应用** | 作为项目基础框架，整合所有组件，提供自动配置能力 |
| **核心特性** | 自动装配、Starter依赖管理、Actuator监控、内嵌Tomcat |

#### 2.1.2 Spring MVC

| 维度 | 说明 |
|------|------|
| **技术作用** | 基于Servlet的MVC框架，处理HTTP请求与响应 |
| **选型理由** | RESTful API设计简洁，与Spring Boot无缝集成，注解驱动开发效率高 |
| **项目中的应用** | 实现所有Controller层接口，处理请求映射、参数绑定、响应序列化 |
| **核心特性** | @RestController、@RequestMapping、@RequestBody/@ResponseBody |

#### 2.1.3 Spring AOP

| 维度 | 说明 |
|------|------|
| **技术作用** | 面向切面编程，实现横切关注点的模块化 |
| **选型理由** | 无侵入式实现权限控制、操作日志、全局异常处理，代码解耦 |
| **项目中的应用** | OperationLogAspect（操作日志切面）、GlobalExceptionHandler（全局异常处理） |
| **核心特性** | @Aspect、@Before/@AfterReturning/@AfterThrowing、切点表达式 |

#### 2.1.4 MyBatis-Plus 3.5.3.1

| 维度 | 说明 |
|------|------|
| **技术作用** | MyBatis增强工具，提供通用CRUD、代码生成、条件构造器 |
| **选型理由** | 大幅减少样板代码，内置分页插件，Lambda表达式构建查询条件更优雅，支持逻辑删除 |
| **项目中的应用** | 所有Mapper层继承BaseMapper，使用QueryWrapper构建查询条件 |
| **核心特性** | BaseMapper通用接口、QueryWrapper条件构造器、PaginationInnerInterceptor分页插件 |

#### 2.1.5 MySQL 8.0.33

| 维度 | 说明 |
|------|------|
| **技术作用** | 开源关系型数据库，支持事务、索引、主从复制 |
| **选型理由** | 成熟稳定的关系型数据库，适合存储结构化的民族文化数据，社区活跃，运维成本低 |
| **项目中的应用** | 存储用户信息、民族文化内容、收藏记录、浏览记录、操作日志等核心业务数据 |
| **核心特性** | InnoDB存储引擎、行级锁、事务支持、索引优化 |

#### 2.1.6 HikariCP

| 维度 | 说明 |
|------|------|
| **技术作用** | JDBC连接池，Spring Boot 2.x默认连接池 |
| **选型理由** | 性能最优的连接池，低延迟、高吞吐量，字节码级别优化 |
| **项目中的应用** | 管理数据库连接，提供连接复用、超时控制、连接检测 |
| **核心特性** | 连接池化管理、空闲连接检测、连接泄漏检测 |

---

### 2.2 分布式与缓存技术

#### 2.2.1 Redis

| 维度         | 说明                                                  |
| ---------- | --------------------------------------------------- |
| **技术作用**   | 内存数据库，支持多种数据结构、持久化、集群                               |
| **选型理由**   | 高性能缓存，支持String/Hash/List/Set/ZSet等丰富数据结构，单线程模型保证原子性 |
| **项目中的应用** | 缓存民族文化详情/列表、计数器存储、分布式锁实现、布隆过滤器数据存储                  |
| **核心特性**   | 单线程模型、IO多路复用、RDB/AOF持久化、主从复制                        |

**Redis在本项目中的具体应用场景：**

| 场景 | 数据结构 | 键格式 | 说明 |
|------|----------|--------|------|
| 详情缓存 | String（JSON序列化） | `culture:detail:{id}` | 存储民族文化详情，逻辑过期 |
| 列表缓存 | String（JSON序列化） | `culture:list:{page}:{size}:{type}` | 存储分页列表数据 |
| 浏览计数 | RAtomicLong | `view:count:{cultureId}` | 原子计数器，线程安全 |
| 点赞计数 | RAtomicLong | `like:count:{cultureId}` | 原子计数器，支持增减 |
| 收藏计数 | RAtomicLong | `collect:count:{cultureId}` | 原子计数器，支持增减 |
| 分布式锁 | RLock | `lock:culture:{id}` | 防止缓存击穿 |

#### 2.2.2 Redisson 3.24.3

| 维度 | 说明 |
|------|------|
| **技术作用** | Redis Java客户端，提供分布式锁、原子计数器等高级特性 |
| **选型理由** | 生产级Redis客户端，可重入锁、看门狗机制、原子操作，比Jedis/Lettuce功能更丰富 |
| **项目中的应用** | RedissonDistributedLock（分布式锁）、RedissonCounter（原子计数器）、布隆过滤器 |
| **核心特性** | 可重入锁RLock、看门狗自动续期、RAtomicLong原子计数器、RBloomFilter布隆过滤器 |

**Redisson核心组件详解：**

| 组件 | 作用 | 项目中的应用 |
|------|------|--------------|
| **RLock** | 可重入分布式锁 | 防止缓存击穿，保护热点数据重建 |
| **RAtomicLong** | 分布式原子计数器 | 浏览量/点赞数/收藏数的原子增减 |
| **RBloomFilter** | 分布式布隆过滤器 | 快速判断文化ID是否存在，防止缓存穿透 |

**Redisson分布式锁优势：**

```
┌─────────────────────────────────────────────────────────────┐
│                    Redisson 分布式锁特性                      │
├─────────────────────────────────────────────────────────────┤
│  1. 可重入性：同一线程可多次获取同一把锁，避免死锁             │
│  2. 看门狗机制：leaseTime=-1时自动续期，防止业务未执行完锁过期 │
│  3. waitTime自动等待：内部自动重试获取锁，简化代码            │
│  4. 公平锁支持：支持公平锁和非公平锁                          │
│  5. 锁重入计数：支持锁的重入次数统计                          │
└─────────────────────────────────────────────────────────────┘
```

#### 2.2.3 RabbitMQ

| 维度 | 说明 |
|------|------|
| **技术作用** | 开源消息队列，支持多种消息模式、持久化、ACK确认 |
| **选型理由** | 成熟稳定的消息队列，实现异步解耦、削峰填谷，支持手动确认保证消息可靠 |
| **项目中的应用** | 异步同步浏览量/点赞数/收藏数到数据库，实现计数器与数据库的最终一致性 |
| **核心特性** | 消息持久化、手动ACK确认、死信队列、消息重试 |

**RabbitMQ在本项目中的应用架构：**

```
┌─────────────────────────────────────────────────────────────────┐
│                    RabbitMQ 消息队列架构                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   生产者                    队列                     消费者       │
│      │                       │                        │         │
│      │   sendMessage()       │                        │         │
│      ├──────────────────────►│  viewCount 队列         │         │
│      │                       ├───────────────────────►│         │
│      │                       │                        │ 更新浏览量│
│      │                       │                        │ 到数据库  │
│      │                       │                        │          │
│      │                       │  likeCount 队列        │          │
│      ├──────────────────────►├───────────────────────►│          │
│      │                       │                        │ 更新点赞数│
│      │                       │                        │ 到数据库  │
│      │                       │                        │          │
│      │                       │  collectCount 队列     │           │
│      ├──────────────────────►├───────────────────────►│          │
│      │                       │                        │ 更新收藏数│
│      │                       │                        │ 到数据库 │
│      ▼                       ▼                        ▼         │
└─────────────────────────────────────────────────────────────────┘
```

---

### 2.3 安全与认证技术

#### 2.3.1 Spring Security Crypto

| 维度 | 说明 |
|------|------|
| **技术作用** | 密码加密组件，提供BCrypt加密算法 |
| **选型理由** | BCrypt加密算法安全性高，自带盐值，每次加密结果不同，防止彩虹表攻击 |
| **项目中的应用** | 用户注册时加密密码，用户登录时验证密码 |
| **核心特性** | 自动生成盐值、可配置加密强度、单向加密不可逆 |

**BCrypt加密流程：**

```
用户注册：
明文密码 → BCryptPasswordEncoder.encode() → 加密密码（含盐值）→ 存入数据库

用户登录：
明文密码 + 数据库加密密码 → BCryptPasswordEncoder.matches() → 验证结果
```

#### 2.3.2 JJWT 0.11.5

| 维度 | 说明 |
|------|------|
| **技术作用** | JWT（JSON Web Token）Java实现，用于无状态认证 |
| **选型理由** | 无状态认证，服务端无需存储会话，支持Token刷新，适合分布式系统 |
| **项目中的应用** | 用户登录生成Token、请求拦截验证Token、Token刷新机制 |
| **核心特性** | 自定义Claims、过期时间控制、签名验证、Token刷新 |

**JWT认证流程：**

```
┌─────────────────────────────────────────────────────────────────┐
│                       JWT 认证流程                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   客户端                        服务端                           │
│      │                           │                              │
│      │  1. 登录请求              │                              │
│      │  (username/password)      │                              │
│      ├──────────────────────────►│                              │
│      │                           │ 验证用户名密码                 │
│      │                           │ 生成JWT Token                 │
│      │  2. 返回Token             │                              │
│      │◄──────────────────────────┤                              │
│      │                           │                              │
│      │  3. 请求资源              │                              │
│      │  (Header: Authorization: Bearer xxx)                     │
│      ├──────────────────────────►│                              │
│      │                           │ 解析Token                     │
│      │                           │ 验证签名和过期时间              │
│      │                           │ 提取用户信息                   │
│      │  4. 返回资源              │                              │
│      │◄──────────────────────────┤                              │
│      ▼                           ▼                              │
└─────────────────────────────────────────────────────────────────┘
```

---

### 2.4 工具库

#### 2.4.1 Lombok 1.18.36

| 维度 | 说明 |
|------|------|
| **技术作用** | 编译时代码生成工具，通过注解自动生成样板代码 |
| **选型理由** | 减少样板代码，提高开发效率，代码更简洁易读 |
| **项目中的应用** | @Data（实体类）、@Slf4j（日志）、@Builder（构建者模式） |
| **核心特性** | @Data/@Getter/@Setter、@Slf4j、@Builder、@AllArgsConstructor |

#### 2.4.2 Guava 31.1-jre

| 维度 | 说明 |
|------|------|
| **技术作用** | Google核心Java库，提供丰富的工具类 |
| **选型理由** | 布隆过滤器实现成熟稳定，丰富的工具类库，经过大规模生产验证 |
| **项目中的应用** | BloomFilter（布隆过滤器），用于快速判断文化ID是否存在 |
| **核心特性** | BloomFilter布隆过滤器、Cache本地缓存、RateLimiter限流器 |

---

### 2.5 线程池技术

#### 2.5.1 ThreadPoolExecutor

| 维度 | 说明 |
|------|------|
| **技术作用** | Java原生线程池，管理线程生命周期，复用线程资源 |
| **选型理由** | 避免频繁创建销毁线程的开销，控制并发数量，防止资源耗尽 |
| **项目中的应用** | asyncTaskExecutor（异步任务线程池）、messageConsumerExecutor（消息消费线程池） |
| **核心特性** | 核心线程数、最大线程数、队列容量、拒绝策略 |

---

## 三、项目架构

### 3.1 项目包结构

```
com.ethnicculture
├── config/                          # 配置类
│   ├── RedissonConfig.java          # Redisson客户端配置
│   ├── RedisConfig.java             # Redis配置
│   ├── RabbitMQConfig.java          # RabbitMQ队列配置
│   ├── ThreadPoolConfig.java        # 线程池配置
│   ├── SecurityConfig.java          # Spring Security安全配置
│   ├── WebMvcConfig.java            # Web MVC配置
│   ├── AuthInterceptor.java         # 认证拦截器
│   └── MyBatisPlusMetaObjectHandler.java  # MyBatis-Plus自动填充
├── controller/                      # 控制器层
│   ├── AuthController.java          # 认证控制器
│   ├── EthnicCultureController.java # 民族文化控制器
│   ├── AdminEthnicCultureController.java  # 后台管理控制器
│   ├── OperationLogController.java  # 操作日志控制器
│   └── ViewRecordController.java    # 浏览记录控制器
├── service/                         # 服务层
│   ├── EthnicCultureService.java
│   ├── AsyncTaskService.java
│   ├── UserService.java
│   ├── OperationLogService.java
│   ├── ViewRecordService.java
│   └── impl/
│       ├── EthnicCultureServiceImpl.java
│       ├── AsyncTaskServiceImpl.java
│       ├── UserServiceImpl.java
│       ├── OperationLogServiceImpl.java
│       └── ViewRecordServiceImpl.java
├── mapper/                          # 数据访问层
│   ├── EthnicCultureMapper.java
│   ├── UserFavoriteMapper.java
│   ├── ViewRecordMapper.java
│   ├── OperationLogMapper.java
│   └── SysUserMapper.java
├── model/
│   ├── entity/                      # 实体类
│   ├── dto/                         # 数据传输对象
│   └── MessageConsumeTask.java      # 消息消费任务
├── utils/                           # 工具类
│   ├── RedissonDistributedLock.java # Redisson分布式锁
│   ├── RedissonCounter.java         # Redisson原子计数器
│   ├── BloomFilterUtils.java        # 布隆过滤器
│   ├── JwtTokenUtils.java           # JWT工具
│   ├── RabbitMessageQueue.java      # RabbitMQ封装
│   ├── SnowflakeIdGenerator.java    # 雪花算法ID生成器
│   └── CacheWarmUpRunner.java       # 缓存预热
├── aspect/                          # 切面
│   ├── GlobalExceptionHandler.java  # 全局异常处理
│   └── OperationLogAspect.java      # 操作日志切面
├── exception/                       # 异常类
│   └── BusinessException.java
└── constant/                        # 常量类
    └── RoleConstant.java
```

### 3.2 架构分层说明

| 层次 | 职责 | 关键类 |
|------|------|--------|
| **Controller层** | 接收HTTP请求，参数校验，调用Service，返回响应 | AuthController, EthnicCultureController |
| **Service层** | 业务逻辑处理，事务管理，缓存操作 | EthnicCultureServiceImpl, UserServiceImpl |
| **Mapper层** | 数据库访问，SQL执行 | EthnicCultureMapper, SysUserMapper |
| **Utils层** | 通用工具类，分布式组件封装 | RedissonDistributedLock, JwtTokenUtils |

---

## 四、核心业务流程

### 4.1 用户注册流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           用户注册流程                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   客户端              Controller              Service              数据库    │
│      │                    │                     │                    │       │
│      │  1.提交注册信息     │                     │                    │       │
│      ├───────────────────►│                     │                    │       │
│      │  (username/password│                     │                    │       │
│      │   /phone)          │                     │                    │       │
│      │                    │  2.调用注册服务     │                    │       │
│      │                    ├────────────────────►│                    │       │
│      │                    │                     │  3.参数校验         │       │
│      │                    │                     │ (用户名/密码/手机号) │       │
│      │                    │                     │                    │       │
│      │                    │                     │  4.检查用户名是否存在│       │
│      │                    │                     ├───────────────────►│       │
│      │                    │                     │◄───────────────────┤       │
│      │                    │                     │   查询结果          │       │
│      │                    │                     │                    │       │
│      │                    │                     │  5.检查手机号是否存在│       │
│      │                    │                     ├───────────────────►│       │
│      │                    │                     │◄───────────────────┤       │
│      │                    │                     │   查询结果          │       │
│      │                    │                     │                    │       │
│      │                    │                     │  6.生成用户ID       │       │
│      │                    │                     │ (雪花算法)          │       │
│      │                    │                     │                    │       │
│      │                    │                     │  7.BCrypt加密密码   │       │
│      │                    │                     │                    │       │
│      │                    │                     │  8.创建用户对象     │       │
│      │                    │                     │                    │       │
│      │                    │                     │  9.插入数据库       │       │
│      │                    │                     ├───────────────────►│       │
│      │                    │                     │                    │ INSERT│
│      │                    │                     │◄───────────────────┤       │
│      │                    │                     │   插入成功          │       │
│      │                    │                     │                    │       │
│      │                    │  10.返回用户信息    │                    │       │
│      │                    │◄────────────────────┤                    │       │
│      │  11.返回注册结果   │                     │                    │       │
│      │◄───────────────────┤                     │                    │       │
│      ▼                    ▼                     ▼                    ▼       │
└─────────────────────────────────────────────────────────────────────────────┘
```

**关键代码逻辑：**

```java
@Transactional
public SysUser register(RegisterDTO registerDTO) {
    // 1. 参数校验
    if (registerDTO.getUsername() == null || registerDTO.getUsername().trim().isEmpty()) {
        throw new BusinessException(400, "用户名不能为空");
    }
    
    // 2. 检查用户名是否已存在
    QueryWrapper<SysUser> userQueryWrapper = new QueryWrapper<>();
    userQueryWrapper.eq("username", registerDTO.getUsername());
    if (sysUserMapper.selectOne(userQueryWrapper) != null) {
        throw new BusinessException(400, "用户名已存在");
    }
    
    // 3. 检查手机号是否已存在
    QueryWrapper<SysUser> phoneQueryWrapper = new QueryWrapper<>();
    phoneQueryWrapper.eq("phone", registerDTO.getPhone());
    if (sysUserMapper.selectOne(phoneQueryWrapper) != null) {
        throw new BusinessException(400, "手机号已存在");
    }
    
    // 4. 生成用户ID（雪花算法）
    Long userId = idGenerator.generateId("user");
    
    // 5. BCrypt加密密码
    String encodedPassword = passwordEncoder.encode(registerDTO.getPassword());
    
    // 6. 创建用户对象并插入数据库
    SysUser user = new SysUser();
    user.setId(userId);
    user.setUsername(registerDTO.getUsername());
    user.setPassword(encodedPassword);
    user.setPhone(registerDTO.getPhone());
    user.setRole(0);  // 普通用户
    user.setStatus(1); // 启用
    
    sysUserMapper.insert(user);
    return user;
}
```

---

### 4.2 用户登录流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           用户登录流程                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   客户端              Controller              Service              数据库    │
│      │                    │                     │                    │       │
│      │  1.提交登录信息     │                     │                    │       │
│      ├───────────────────►│                     │                    │       │
│      │  (username/password│                     │                    │       │
│      │   /rememberMe)     │                     │                    │       │
│      │                    │  2.调用登录服务     │                    │       │
│      │                    ├────────────────────►│                    │       │
│      │                    │                     │  3.参数校验         │       │
│      │                    │                     │                    │       │
│      │                    │                     │  4.查询用户         │       │
│      │                    │                     ├───────────────────►│       │
│      │                    │                     │◄───────────────────┤       │
│      │                    │                     │   用户信息          │       │
│      │                    │                     │                    │       │
│      │                    │                     │  5.验证密码         │       │
│      │                    │                     │ (BCrypt.matches)   │       │
│      │                    │                     │                    │       │
│      │                    │                     │  6.验证账号状态     │       │
│      │                    │                     │ (status == 1)      │       │
│      │                    │                     │                    │       │
│      │                    │  7.返回用户信息     │                    │       │
│      │                    │◄────────────────────┤                    │       │
│      │                    │                     │                    │       │
│      │                    │  8.生成JWT Token    │                    │       │
│      │                    │ (JwtTokenUtils)     │                    │       │
│      │                    │                     │                    │       │
│      │  9.返回Token和用户信息                   │                    │       │
│      │◄───────────────────┤                     │                    │       │
│      ▼                    ▼                     ▼                    ▼       │
└─────────────────────────────────────────────────────────────────────────────┘
```

**关键代码逻辑：**

```java
public SysUser login(LoginDTO loginDTO) {
    // 1. 参数校验
    if (loginDTO.getUsername() == null || loginDTO.getUsername().trim().isEmpty()) {
        throw new BusinessException(400, "用户名不能为空");
    }
    
    // 2. 查询用户
    QueryWrapper<SysUser> queryWrapper = new QueryWrapper<>();
    queryWrapper.eq("username", loginDTO.getUsername());
    SysUser user = sysUserMapper.selectOne(queryWrapper);
    
    if (user == null) {
        throw new BusinessException(401, "用户名或密码错误");
    }
    
    // 3. 验证密码（BCrypt）
    if (!passwordEncoder.matches(loginDTO.getPassword(), user.getPassword())) {
        throw new BusinessException(401, "用户名或密码错误");
    }
    
    // 4. 验证账号状态
    if (user.getStatus() != 1) {
        throw new BusinessException(403, "账号已被禁用");
    }
    
    return user;
}
```

---

### 4.3 民族文化详情查询流程（核心缓存流程）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    民族文化详情查询流程（缓存穿透/击穿防护）                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   请求                  布隆过滤器           Redis缓存         数据库          │
│     │                      │                   │                │           │
│     │  1.查询详情(id)      │                   │                │            │
│     ├─────────────────────►│                   │                │           │
│     │                      │                   │                │           │
│     │                      │  2.判断ID是否存在  │                │           │
│     │                      │ (mightContain)    │                │           │
│     │                      │                   │                │           │
│     │         ┌────────────┴────────────┐      │                │           │
│     │         │                         │      │                │           │
│     │    不存在（穿透防护）          存在      │                │              │
│     │         │                         │      │                │           │
│     │         ▼                         ▼      │                │           │
│     │    直接返回null            3.查询缓存    │                │             │
│     │                               ├─────────►│                │           │
│     │                               │          │                │           │
│     │                    ┌──────────┴─────────┐│                │           │
│      │                    │                    ││                │          │
│      │               缓存存在              缓存不存在              │          │
│      │                    │                    ││                │          │
│      │                    ▼                    ▼│                │          │
│      │              检查逻辑过期           4.获取分布式锁           │          │
│      │                    │             (Redisson)               │          │
│      │         ┌──────────┴──────────┐         ││                │          │
│      │         │                     │         ││                │          │
│      │    未过期                  已过期        ││                │          │
│      │         │                     │         ││                │          │
│      │         ▼                     ▼         ││                │          │
│      │    返回缓存数据         异步刷新缓存        ││                │          │
│      │    更新浏览量           返回旧数据         ││                │          │
│      │                              │          ▼│                │          │
│      │                              │     5.查询数据库            │          │
│      │                              │          ├───────────────►│          │
│      │                              │          │◄───────────────┤          │
│      │                              │          │   数据          │          │
│      │                              │          │                │          │
│      │                              │     6.写入缓存            │          │
│      │                              │     (逻辑过期)            │          │
│      │                              │          │                │          │
│      │                              │     7.释放锁              │          │
│      │                              │          │                │          │
│      ▼                              ▼          ▼                ▼          │
└─────────────────────────────────────────────────────────────────────────────┘
```

**关键代码逻辑：**

```java
@Override
public EthnicCulture getEthnicCultureById(Long id, Long userId) {
    // 1. 布隆过滤器快速判断ID是否存在（防止缓存穿透）
    if (!bloomFilter.mightContain(id)) {
        return null;
    }
    
    // 2. 构建缓存键
    String cacheKey = CULTURE_DETAIL_KEY + id;
    
    // 3. 尝试从缓存获取
    CacheData<EthnicCulture> cacheData = (CacheData<EthnicCulture>) 
        redisTemplate.opsForValue().get(cacheKey);
    
    if (cacheData != null) {
        // 异步更新浏览量
        executorService.execute(() -> updateViewCount(id));
        
        // 检查是否逻辑过期
        if (!cacheData.isExpired()) {
            // 未过期，直接返回数据
            return cacheData.getData();
        } else {
            // 逻辑过期，返回旧数据，同时异步更新缓存
            executorService.execute(() -> refreshCache(id, cacheKey));
            return cacheData.getData();
        }
    }
    
    // 4. 缓存不存在，从数据库查询并更新缓存
    EthnicCulture culture = refreshCache(id, cacheKey);
    
    return culture;
}

private EthnicCulture refreshCache(Long id, String cacheKey) {
    String lockKey = "lock:culture:" + id;
    boolean locked = false;
    
    try {
        // 获取分布式锁（防止缓存击穿）
        if (distributedLock.tryLock(lockKey, 3000, getLockExpiration())) {
            locked = true;
            
            // 双检：再次检查缓存
            CacheData<EthnicCulture> cacheData = (CacheData<EthnicCulture>) 
                redisTemplate.opsForValue().get(cacheKey);
            if (cacheData != null && !cacheData.isExpired()) {
                return cacheData.getData();
            }
            
            // 查询数据库
            EthnicCulture culture = ethnicCultureMapper.selectById(id);
            
            // 存入缓存（逻辑过期）
            CacheData<EthnicCulture> newCacheData = new CacheData<>(
                culture, 
                System.currentTimeMillis() + getLogicExpiration()
            );
            redisTemplate.opsForValue().set(cacheKey, newCacheData);
            
            return culture;
        }
    } finally {
        if (locked) {
            distributedLock.unlock(lockKey);
        }
    }
    
    // 获取锁失败，从缓存获取数据
    // ...
}
```

---

### 4.4 点赞/收藏流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        点赞/收藏流程                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   用户操作              Service层              Redis              数据库    │
│      │                     │                    │                   │       │
│      │  1.点击点赞/收藏    │                    │                   │       │
│      ├────────────────────►│                    │                   │       │
│      │                     │                    │                   │       │
│      │                     │  2.获取分布式锁    │                   │       │
│      │                     │ (防重复操作)       │                   │       │
│      │                     ├───────────────────►│                   │       │
│      │                     │◄───────────────────┤                   │       │
│      │                     │   获取锁成功       │                   │       │
│      │                     │                    │                   │       │
│      │                     │  3.查询是否已操作  │                   │       │
│      │                     ├──────────────────────────────────────►│       │
│      │                     │◄──────────────────────────────────────┤       │
│      │                     │   操作记录          │                   │       │
│      │                     │                    │                   │       │
│      │            ┌────────┴────────┐           │                   │       │
│      │            │                 │           │                   │       │
│      │       未操作             已操作           │                   │       │
│      │            │                 │           │                   │       │
│      │            ▼                 ▼           │                   │       │
│      │       新增操作记录      删除操作记录      │                   │       │
│      │            │                 │           │                   │       │
│      │            │                 │           │                   │       │
│      │       Redis计数器+1     Redis计数器-1    │                   │       │
│      │            ├───────────────────────────►│                   │       │
│      │            │                 │          │                   │       │
│      │       发送消息到队列   发送消息到队列    │                   │       │
│      │            ├───────────────────────────►│                   │       │
│      │            │                 │          │                   │       │
│      │            │                 │          │  异步消费者        │       │
│      │            │                 │          │      │            │       │
│      │            │                 │          │ 读取Redis计数     │       │
│      │            │                 │          │      │            │       │
│      │            │                 │          │ 更新数据库        │       │
│      │            │                 │          │      │            │       │
│      │  4.返回操作结果           │          │      │            │       │
│      │◄───────────────────────────┤          │      │            │       │
│      ▼                     ▼                    ▼      ▼            ▼       │
└─────────────────────────────────────────────────────────────────────────────┘
```

**关键代码逻辑：**

```java
@Override
@Transactional
public boolean toggleLike(Long userId, Long cultureId, boolean isLike) {
    String lockKey = "lock:like:" + userId + ":" + cultureId;
    
    // 1. 获取分布式锁（防止重复操作）
    if (distributedLock.tryLock(lockKey, 3000, 30000)) {
        try {
            // 2. 查询是否已点赞
            QueryWrapper<UserFavorite> queryWrapper = new QueryWrapper<>();
            queryWrapper.eq("user_id", userId);
            queryWrapper.eq("culture_id", cultureId);
            queryWrapper.eq("type", 1); // 1-点赞
            UserFavorite existingFavorite = userFavoriteMapper.selectOne(queryWrapper);
            
            if (isLike) {
                // 点赞操作
                if (existingFavorite == null) {
                    // 新增点赞记录
                    UserFavorite favorite = new UserFavorite();
                    favorite.setId(idGenerator.generateId("like"));
                    favorite.setUserId(userId);
                    favorite.setCultureId(cultureId);
                    favorite.setType(1);
                    userFavoriteMapper.insert(favorite);
                    
                    // 3. Redis原子计数器+1
                    String countKey = LIKE_COUNT_KEY + cultureId;
                    redissonCounter.increment(countKey);
                    redissonCounter.expire(countKey, getCounterExpiration(), TimeUnit.SECONDS);
                    
                    // 4. 发送消息到队列（异步同步到数据库）
                    rabbitMessageQueue.sendMessage("likeCount", cultureId.toString());
                    
                    return true;
                }
                return false;
            } else {
                // 取消点赞操作
                if (existingFavorite != null) {
                    userFavoriteMapper.deleteById(existingFavorite.getId());
                    
                    // Redis原子计数器-1
                    String countKey = LIKE_COUNT_KEY + cultureId;
                    redissonCounter.decrement(countKey);
                    redissonCounter.expire(countKey, getCounterExpiration(), TimeUnit.SECONDS);
                    
                    // 发送消息到队列
                    rabbitMessageQueue.sendMessage("likeCount", cultureId.toString());
                    
                    return true;
                }
                return false;
            }
        } finally {
            distributedLock.unlock(lockKey);
        }
    }
    return false;
}
```

---

### 4.5 浏览量计数器同步流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    浏览量计数器同步流程                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   用户访问              Service层              Redis            数据库      │
│      │                     │                    │                 │         │
│      │  1.访问详情页       │                    │                 │         │
│      ├────────────────────►│                    │                 │         │
│      │                     │                    │                 │         │
│      │                     │  2.布隆过滤器判断  │                 │         │
│      │                     │ (ID是否存在)       │                 │         │
│      │                     │                    │                 │         │
│      │                     │  3.Redis计数器+1   │                 │         │
│      │                     ├───────────────────►│                 │         │
│      │                     │   increment()      │                 │         │
│      │                     │                    │                 │         │
│      │                     │  4.设置过期时间    │                 │         │
│      │                     ├───────────────────►│                 │         │
│      │                     │   expire()         │                 │         │
│      │                     │                    │                 │         │
│      │                     │  5.发送消息到队列  │                 │         │
│      │                     ├───────────────────►│                 │         │
│      │                     │  "viewCount"队列   │                 │         │
│      │                     │                    │                 │         │
│      │  6.返回页面内容     │                    │                 │         │
│      │◄────────────────────┤                    │                 │         │
│      ▼                     ▼                    ▼                 ▼         │
│                                                                             │
│   ════════════════════════════════════════════════════════════════════════ │
│                              异步消费者线程                                   │
│   ════════════════════════════════════════════════════════════════════════ │
│                                                                             │
│   消费者                RabbitMQ              Redis            数据库      │
│      │                     │                    │                 │         │
│      │  7.消费消息         │                    │                 │         │
│      │◄────────────────────┤                    │                 │         │
│      │   (手动确认模式)    │                    │                 │         │
│      │                     │                    │                 │         │
│      │  8.读取Redis计数    │                    │                 │         │
│      │ ├───────────────────────────────────────►│                 │         │
│      │ │◄──────────────────────────────────────┤                 │         │
│      │ │  当前浏览量         │                 │         │
│      │ │                    │                 │         │
│      │  9.更新数据库        │                    │                 │         │
│      │ ├─────────────────────────────────────────────────────────►│         │
│      │ │                    │                 │  UPDATE          │         │
│      │ │                    │                 │  view_count      │         │
│      │ │                    │                 │         │         │
│      │  10.确认消息(ACK)    │                    │                 │         │
│      │ ├───────────────────►│                    │                 │         │
│      │ │  basicAck()        │                    │                 │         │
│      ▼ ▼                    ▼                    ▼                 ▼         │
└─────────────────────────────────────────────────────────────────────────────┘
```

**关键代码逻辑：**

```java
// 浏览量更新（EthnicCultureServiceImpl.java）
@Override
public void updateViewCount(Long cultureId) {
    // 布隆过滤器快速判断
    if (!bloomFilter.mightContain(cultureId)) {
        return;
    }
    
    // Redis原子计数器+1
    String countKey = VIEW_COUNT_KEY + cultureId;
    redissonCounter.increment(countKey);
    redissonCounter.expire(countKey, getCounterExpiration(), TimeUnit.SECONDS);
    
    // 发送消息到队列
    rabbitMessageQueue.sendMessage("viewCount", cultureId.toString());
}

// 异步消费（AsyncTaskServiceImpl.java）
@Override
public void consumeViewCountMessages() {
    while (running.get()) {
        // 手动确认模式获取消息
        MessageWrapper wrapper = rabbitMessageQueue.receiveMessageManual("viewCount");
        if (wrapper != null) {
            submitTaskWithRequeueSupport(wrapper, cultureId -> {
                updateViewCountInDb(Long.parseLong(cultureId));
            });
        }
    }
}

@Override
public void updateViewCountInDb(Long cultureId) {
    // 从Redis读取最新计数值
    Long viewCount = redissonCounter.get(VIEW_COUNT_KEY + cultureId);
    if (viewCount != null) {
        EthnicCulture culture = new EthnicCulture();
        culture.setId(cultureId);
        culture.setViewCount(viewCount);
        ethnicCultureMapper.updateById(culture);  // 更新数据库
    }
}
```

---

### 4.6 操作日志记录流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        操作日志记录流程（AOP切面）                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   请求                  Controller              AOP切面            数据库    │
│      │                      │                     │                  │       │
│      │  1.HTTP请求          │                     │                  │       │
│      ├─────────────────────►│                     │                  │       │
│      │                      │                     │                  │       │
│      │                      │  2.@Before切面      │                  │       │
│      │                      ├────────────────────►│                  │       │
│      │                      │                     │ 记录开始时间      │       │
│      │                      │                     │ (ThreadLocal)    │       │
│      │                      │                     │                  │       │
│      │                      │  3.执行业务方法     │                  │       │
│      │                      │                     │                  │       │
│      │                      │  4.@AfterReturning  │                  │       │
│      │                      ├────────────────────►│                  │       │
│      │                      │                     │ 计算执行时长      │       │
│      │                      │                     │                  │       │
│      │                      │                     │ 提取请求信息：    │       │
│      │                      │                     │ - 用户ID(从JWT)   │       │
│      │                      │                     │ - 操作描述        │       │
│      │                      │                     │ - 方法名          │       │
│      │                      │                     │ - 请求参数        │       │
│      │                      │                     │ - IP地址          │       │
│      │                      │                     │                  │       │
│      │                      │                     │ 5.异步保存日志    │       │
│      │                      │                     ├─────────────────►│       │
│      │                      │                     │                  │ INSERT│
│      │  6.返回响应          │                     │                  │       │
│      │◄─────────────────────┤                     │                  │       │
│      ▼                      ▼                     ▼                  ▼       │
│                                                                             │
│   ────────────────────────────────────────────────────────────────────────  │
│                          异常情况处理                                        │
│   ────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│      │                      │  4.@AfterThrowing   │                  │       │
│      │                      ├────────────────────►│                  │       │
│      │                      │                     │ 记录异常信息      │       │
│      │                      │                     │ result=0(失败)   │       │
│      │                      │                     │ error_msg        │       │
│      │                      │                     ├─────────────────►│       │
│      │  5.返回错误响应      │                     │                  │       │
│      │◄─────────────────────┤                     │                  │       │
│      ▼                      ▼                     ▼                  ▼       │
└─────────────────────────────────────────────────────────────────────────────┘
```

- 完整过程：
1. 请求到达Controller时被拦截，开启一个ThreadLocal记录开始时间
2. 在成功提交或者抛出异常后记录下操作完成时间，并计算出整个方法的耗时，关闭ThreadLocal实例，防止内存泄漏
3. 从捕获的上下文信息中提取用户ID、方法名、参数等，线程提交到线程池异步保存日志
**关键代码逻辑：**

```java
@Aspect
@Component
public class OperationLogAspect {
    
    private ThreadLocal<Long> startTimeHolder = new ThreadLocal<>();
    
    // 前置通知：记录开始时间
    @Before("execution(* com.ethnicculture.controller.*.*(..))")
    public void before(JoinPoint joinPoint) {
        startTimeHolder.set(System.currentTimeMillis());
    }
    
    // 返回通知：记录成功日志
    @AfterReturning(pointcut = "execution(* com.ethnicculture.controller.*.*(..))", returning = "result")
    public void afterReturning(JoinPoint joinPoint, Object result) {
        long duration = System.currentTimeMillis() - startTimeHolder.get();
        startTimeHolder.remove();
        saveOperationLog(joinPoint, duration, 1, null);  // result=1 成功
    }
    
    // 异常通知：记录失败日志
    @AfterThrowing(value = "execution(* com.ethnicculture.controller.*.*(..))", throwing = "e")
    public void afterThrowing(JoinPoint joinPoint, Throwable e) {
        long duration = System.currentTimeMillis() - startTimeHolder.get();
        startTimeHolder.remove();
        saveOperationLog(joinPoint, duration, 0, e.getMessage());  // result=0 失败
    }
    
    private void saveOperationLog(JoinPoint joinPoint, long duration, int result, String errorMsg) {
        HttpServletRequest request = getRequest();
        
        // 从JWT Token提取用户ID
        Long userId = null;
        String authHeader = request.getHeader("Authorization");
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);
            userId = jwtTokenUtils.validateToken(token);
        }
        
        // 提取操作信息
        String operation = getOperationName(joinPoint);
        String methodName = joinPoint.getSignature().getName();
        String params = getParams(joinPoint);
        
        // 异步保存日志
        operationLogService.logOperationAsync(userId, null, operation, methodName, params, result, errorMsg, duration);
    }
}
```

---

## 五、缓存策略详解

### 5.1 缓存架构

项目采用**多级缓存 + 逻辑过期 + Redisson优化**的缓存架构，全面解决缓存击穿、雪崩、穿透问题。

```
请求 → 布隆过滤器拦截 → Redis缓存 → 分布式锁保护 → 数据库
         ↓                     ↓           ↓
      快速判断              逻辑过期     可重入锁+看门狗
```

### 5.2 缓存键设计

| 缓存类型 | 键格式 | 说明 |
|----------|--------|------|
| 详情缓存 | `culture:detail:{id}` | 单个民族文化详情 |
| 列表缓存 | `culture:list:{page}:{size}:{ethnicType}:{contentType}` | 分页列表 |
| 搜索缓存 | `culture:search:{keyword}:{page}:{size}` | 搜索结果 |
| 浏览数缓存 | `view:count:{cultureId}` | Redisson RAtomicLong |
| 点赞数缓存 | `like:count:{cultureId}` | Redisson RAtomicLong |
| 收藏数缓存 | `collect:count:{cultureId}` | Redisson RAtomicLong |

### 5.3 缓存过期策略

```java
// 计数器缓存：30分钟（上下5分钟随机jitter防止雪崩）
private static final long COUNTER_EXPIRATION_BASE = 1800;  // 30分钟
private static final long COUNTER_EXPIRATION_MIN = 1500;    // 25分钟
private static final long COUNTER_EXPIRATION_MAX = 2100;    // 35分钟

// 逻辑过期时间：1小时（上下5分钟随机jitter）
private static final long LOGIC_EXPIRATION_BASE = 3600 * 1000;  // 1小时
private static final long LOGIC_EXPIRATION_MIN = 3300 * 1000;   // 55分钟
private static final long LOGIC_EXPIRATION_MAX = 3900 * 1000;   // 65分钟

// 分布式锁过期时间：25-35秒随机（防止雪崩）
private static final long LOCK_EXPIRATION_BASE = 30000;  // 30秒
private static final long LOCK_EXPIRATION_MIN = 25000;   // 25秒
private static final long LOCK_EXPIRATION_MAX = 35000;   // 35秒
```

### 5.4 缓存问题解决方案

#### 5.4.1 缓存穿透

**问题**：大量请求查询不存在的数据，绕过缓存直达数据库。

**解决方案**：双布隆过滤器

```java
// BloomFilterUtils.java - 双布隆过滤器
public class BloomFilterUtils {
    private BloomFilter<Long> currentFilter;   // 当前使用
    private BloomFilter<Long> rebuildFilter;   // 重建中

    // 定时重建（每小时）
    @Scheduled(cron = "0 0 * * * ?")
    public void rebuildFilter() {
        // 从数据库加载所有有效ID
        // 交换currentFilter和rebuildFilter
    }

    public boolean mightContain(Long cultureId) {
        // 同时检查两个过滤器
        return currentFilter.mightContain(cultureId) 
            || rebuildFilter.mightContain(cultureId);
    }
}
```

#### 5.4.2 缓存击穿

**问题**：热点Key过期瞬间，大量并发请求击穿到数据库。

**解决方案**：Redisson可重入分布式锁 + 逻辑过期

```java
// 逻辑过期缓存数据结构
public class CacheData<T> {
    private T data;
    private long expireTime;  // 逻辑过期时间

    public boolean isExpired() {
        return System.currentTimeMillis() > expireTime;
    }
}

// 使用Redisson分布式锁
if (distributedLock.tryLock(lockKey, 3000, getLockExpiration())) {
    // 双检：再次查询缓存
    // 加载数据库并写入缓存
}
```

**Redisson分布式锁优势**：
- **可重入性**：同一线程可多次获取同一把锁
- **看门狗机制**：leaseTime=-1时自动续期
- **waitTime自动等待**：内部自动重试获取锁

#### 5.4.3 缓存雪崩

**问题**：大量缓存同时过期，导致数据库压力骤增。

**解决方案**：
1. **随机Jitter**：缓存过期时间±5分钟随机
2. **逻辑过期**：缓存数据永不过期，只在逻辑上判断
3. **异步刷新**：检测到逻辑过期时，返回旧数据，异步刷新

---

## 六、异步任务处理

### 6.1 架构设计

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         异步任务处理架构                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Controller                  RabbitMQ                  消费者线程池         │
│       │                          │                           │              │
│       │  1.发送消息              │                           │              │
│       ├─────────────────────────►│                           │              │
│       │  (cultureId)             │                           │              │
│       │                          │  2.消息入队                │              │
│       │                          │                           │              │
│       │                          │  3.消费者拉取消息          │              │
│       │                          ├──────────────────────────►│              │
│       │                          │  (手动确认模式)            │              │
│       │                          │                           │              │
│       │                          │                           │  4.执行任务  │
│       │                          │                           │  - 读取Redis │
│       │                          │                           │  - 更新DB    │
│       │                          │                           │              │
│       │                          │  5.确认消息(ACK)          │              │
│       │                          │◄──────────────────────────┤              │
│       │                          │                           │              │
│       ▼                          ▼                           ▼              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 完整的消息消费流程
（点赞为例）

#### 阶段一
1. 用户点击点赞，`EthnicCultureController.like()`获取到用户id和文化id传入service
2. `EthnicCultureServiceImpl.like()`
	1. redis计数器+1 
	` redissonCounter.increment(LIKE_COUNT_KEY + cultureId)`
	2. 发送点赞消息到MQ
	 `rabbitMessageQueue.sendMessage("likeCount", cultureId)`
3. 响应'点赞成功'
#### 阶段二：消费者线程接收消息
1. 在线程池中创建一个点赞消费者线程
```java
//AsyncTaskServiceImpl.java

//注解告诉Spring在Bean初始化完成后调用init方法，用于启动异步任务消费者
@PostConstruct
@Override
public void init(){
messageConsumerExecutor.execute(this::consumeLikeCountMessages)
}
```
2. 消费者线程持续运行
```java
//AsyncTaskServiceImpl.java

@Override
    public void consumeLikeCountMessages() {
        while (running.get()) {
            try {
            //从消息队列拉取消息
                MessageWrapper wrapper = rabbitMessageQueue.receiveMessageManual(RabbitMessageQueue.QUEUE_LIKE_COUNT);
                if (wrapper != null) {
                //将消息提交到线程池
               submitTaskWithRequeueSupport(wrapper, cultureId -> updateLikeCountInDb(Long.parseLong(cultureId)));
                }
            } catch (Exception e) {
                if (running.get()) {
                    log.error("消费点赞数消息异常: {}", e.getMessage());
                }
            }
        }
        log.info("点赞数消费者已停止");
    }


//AsyncTaskServiceImpl.java

private void submitTaskWithRequeueSupport(MessageWrapper wrapper, java.util.function.Consumer<String> handler) {

        //根据从消息队列提取到的消息，创建一个任务对象
        MessageConsumeTask task = new MessageConsumeTask(wrapper, handler);
        try {
            //将任务交给线程池去执行
            messageConsumerExecutor.execute(task);
        } catch (RejectedExecutionException e) {
        //线程池已满，拒绝当前任务，将任务重新加入消息队列
        //重新入队后当前task对象会被丢弃，没有引用后被GC
            task.rejectAndRequeue();
        }
    }
    
//MessageConsumeTask.java
/**
     * 对外暴露一个拒绝任务方法，用于拒绝任务并重新入队，保护了MessageWrapper的内部细节
     * <p>
     * 当线程池拒绝此任务时调用，通知 RabbitMQ 重新投递消息。
     * </p>
     */
    public void rejectAndRequeue() {
    //消息没有被处理或确认才重新入队，避免重复入队
        if (!handled && !messageWrapper.isAcknowledged()) {
            handled = true;
            log.warn("【线程池拒绝】消息被拒绝，准备nack并requeue，queue: {}, deliveryTag: {}",
            messageWrapper.getQueueName(), messageWrapper.getDeliveryTag());
            messageWrapper.nackAndRequeue();
        }
    }
    
//RabbitMessageQueue.java    
	   
	    //拒绝消息并重新入队，等待下一次在consumeLikeCountMessages()中被拉取
	  public void nackAndRequeue() {
            if (!acknowledged) {
                acknowledged = true;
                try {
                    //消息处理失败了
                    //basicNack(deliveryTag, false, true) 中的 false 表示 手动确认模式
                    //true 表示 重新入队
                    //false 表示 不重新入队
                    channel.basicNack(deliveryTag, false, true);
                    log.warn("消息已nack并重新入队，queue: {}, deliveryTag: {}", queueName, deliveryTag);
                } catch (IOException e) {
                    log.error("nack消息失败，deliveryTag: {}, 错误: {}", deliveryTag, e.getMessage());
                }
            }
        }
```
#### 阶段三：线程池执行任务
1. `messageConsumerExecutor.execute(task)`被调用后，线程池分配一个线程来执行`MessageConsumeTask.run()`
```java
@Override
    public void run() {
        //若任务已经处理过则直接跳过
        if (handled) {
            return;
        }
        try {
            //根据消息包装中的消息内容，调用消息处理逻辑，处理消息内容
            messageHandler.accept(messageWrapper.getMessage());
            //通知RabbitMQ消息处理成功，确认消息。
            // RabbitMQ收到后将调用channel.basicAck(deliveryTag, false)将消息从队列中移除
            messageWrapper.ack();
            //通知当前线程，确认消息处理成功，防止当前任务被其他线程的run方法重复执行
            handled = true;
            log.debug("消息处理成功，queue: {}, deliveryTag: {}",
                messageWrapper.getQueueName(), messageWrapper.getDeliveryTag());
        } catch (Exception e) {
            log.error("消息处理失败，准备nack并requeue，queue: {}, deliveryTag: {}, 错误: {}",
                messageWrapper.getQueueName(), messageWrapper.getDeliveryTag(), e.getMessage());
             //若线程消费该消息时出现异常，则将消息重新入队
            messageWrapper.nackAndRequeue();
            handled = true;
        }
    }
```
#### 阶段四：数据库更新
执行updateLikeCountInDb(Long cultureId)方法更新数据库
### 6.3 线程池拒绝策略

```java
// 线程池繁忙时，拒绝任务并让 RabbitMQ 重试
private void submitTaskWithRequeueSupport(MessageWrapper wrapper, Consumer<String> handler) {
    MessageConsumeTask task = new MessageConsumeTask(wrapper, handler);
    
    try {
        messageConsumerExecutor.execute(task);
    } catch (RejectedExecutionException e) {
        // 线程池拒绝 → nack + requeue，让 MQ 重新投递
        log.warn("【线程池繁忙】任务被拒绝，执行nack+requeue");
        task.rejectAndRequeue();
    }
}
```

**拒绝策略设计原则**：
- **不阻塞**：拒绝时立即返回，不等待
- **不sleep**：不使用Thread.sleep阻塞线程
- **不丢失任务**：通过nack + requeue让RabbitMQ重新投递

---

## 七、线程池配置

### 7.1 线程池参数设计

| 线程池名称                   | 核心线程数                           | 最大线程数 | 队列容量 | 拒绝策略             | 用途     |
| ----------------------- | ------------------------------- | ----- | ---- | ---------------- | ------ |
| asyncTaskExecutor       | CPU×2                           | CPU×4 | 200  | CallerRunsPolicy | 异步任务执行 |
| messageConsumerExecutor | Math.max(3, CORE_POOL_SIZE / 2) | CPU×2 | 100  | AbortPolicy      | 消息队列消费 |

### 7.2 参数设计理由

**asyncTaskExecutor（异步任务线程池）**：
- **核心线程数 = CPU×2**：异步任务多为IO密集型（Redis操作、数据库操作），需要更多线程
- **最大线程数 = CPU×4**：应对突发流量
- **队列容量 = 200**：较大的队列缓冲能力
- **拒绝策略 = CallerRunsPolicy**：调用者线程执行，保证任务不丢失

**messageConsumerExecutor（消息消费线程池）**：
- **核心线程数 =Math.max(3, CORE_POOL_SIZE / 2)：消息消费需要稳定可控,项目中定义了三个消费者进程，所以核心线程数至少为3
- **最大线程数 = CPU×2**：适度扩展
- **队列容量 = 100**：适中容量
- **拒绝策略 = AbortPolicy**：抛出异常，触发nack + requeue

---

## 八、分布式组件

### 8.1 Redisson分布式锁

```java
@Component
public class RedissonDistributedLock {
    
    @Autowired
    private RedissonClient redissonClient;
    
    /**
     * 尝试获取分布式锁
     * @param lockKey 锁键
     * @param waitTime 等待时间（毫秒）
     * @param leaseTime 持有时间（毫秒）
     */
    public boolean tryLock(String lockKey, long waitTime, long leaseTime) {
        RLock lock = redissonClient.getLock(lockKey);
        try {
            return lock.tryLock(waitTime, leaseTime, TimeUnit.MILLISECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        }
    }
    
    public void unlock(String lockKey) {
        RLock lock = redissonClient.getLock(lockKey);
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
}
```

### 8.2 Redisson原子计数器

```java
@Component
public class RedissonCounter {
    
    @Autowired
    private RedissonClient redissonClient;
    
    public long increment(String key) {
        RAtomicLong atomicLong = redissonClient.getAtomicLong(key);
        return atomicLong.incrementAndGet();
    }
    
    public long decrement(String key) {
        RAtomicLong atomicLong = redissonClient.getAtomicLong(key);
        return atomicLong.decrementAndGet();
    }
    
    public long get(String key) {
        RAtomicLong atomicLong = redissonClient.getAtomicLong(key);
        return atomicLong.get();
    }
}
```

### 8.3 雪花算法ID生成器

```java
@Component
public class SnowflakeIdGenerator implements IdGenerator {
    
    private final long workerId;
    private final long datacenterId;
    private long sequence = 0;
    private long lastTimestamp = -1L;
    
    // 生成唯一ID
    public synchronized long nextId() {
        long timestamp = System.currentTimeMillis();
        
        if (timestamp < lastTimestamp) {
            throw new RuntimeException("时钟回拨");
        }
        
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & 0xFFF;
            if (sequence == 0) {
                timestamp = tilNextMillis(lastTimestamp);
            }
        } else {
            sequence = 0;
        }
        
        lastTimestamp = timestamp;
        
        return ((timestamp - 1288834974657L) << 22)
             | (datacenterId << 17)
             | (workerId << 12)
             | sequence;
    }
}
```

---

## 九、安全设计

### 9.1 JWT认证机制

| 特性 | 说明 |
|------|------|
| **Token生成** | 登录成功后生成JWT Token，包含用户ID、用户名、过期时间 |
| **Token验证** | 请求拦截器验证Token签名和过期时间 |
| **Token刷新** | 支持Token刷新机制，延长会话时间 |
| **无状态** | 服务端不存储会话，适合分布式部署 |

### 9.2 密码安全

| 特性 | 说明 |
|------|------|
| **加密算法** | BCrypt，自带盐值 |
| **加密强度** | 默认10轮加密 |
| **存储方式** | 只存储加密后的密码，不存储明文 |

### 9.3 权限控制

| 角色 | 权限 |
|------|------|
| 普通用户(0) | 浏览、点赞、收藏、评论 |
| 编辑(1) | 内容管理、审核 |
| 管理员(2) | 用户管理、系统管理 |

---

## 十、全局异常处理

### 10.1 异常处理架构

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    // 业务异常
    @ExceptionHandler(BusinessException.class)
    public ResultDTO<Void> handleBusinessException(BusinessException e) {
        return ResultDTO.error(e.getCode(), e.getMessage());
    }
    
    // 参数校验异常
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResultDTO<Void> handleValidationException(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldError().getDefaultMessage();
        return ResultDTO.error(400, message);
    }
    
    // 其他异常
    @ExceptionHandler(Exception.class)
    public ResultDTO<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return ResultDTO.error(500, "系统异常");
    }
}
```


## 十一、常见问题：

####  数据库和redis的数据一致性是怎么做的
1. Cache Aside策略
	- 读操作
		- 若缓存命中，则直接返回数据
		- 若缓存未命中，走「查库 → 写缓存 → 返回数据」的流程，通过分布式锁来控制查库时的并发
	- 写操作
		- 先更数据库，再删缓存
2. 逻辑过期
	- 读操作时数据过期时先返回旧数据，同时异步更新缓存
	- 对于删除失败的缓存在过期后会自动shixiao
3. 通过redisson原子计数器来保证高并发下浏览、点赞等操作的计数正确

#### 消息队列为什么选RabbitMQ

#### 消息队列的可靠性是怎么实现的
1. 开启消息持久化
2. 通过手动确认模式防止线程池拒绝或消息业务执行失败后消息丢失，消息未执行完成会重新入队
3. 通过handler标签防止消息被线程**重复处理**
4. 同过acknowledged标签防止消费被重复确认导致重复调用basicAck/basicNack

#### 日志记录是怎么做的，AOP如何使用的

#### 项目中的Redisson是怎么实现的

#### 为什么用redisson而不是直接用stnx

#### 为什么用redis不用本地缓存

#### 缓存问题是如何解决的

#### 权限校验是怎么实现的

#### 为什么用JWT不用 Redis共享Session








