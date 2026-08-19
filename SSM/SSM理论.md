## Spring
#### Bean
>Spring 应用的**核心组件**，本质是被 **IoC（Inversion of Control） 容器**管理的 Java 对象，核心作用是通过容器接管对象的生命周期和依赖关系，实现 **解耦、复用、可扩展**，通过IoC容器创建、装配、管理
###### 具体执行逻辑
1. Spring 启动时会通过 `@ComponentScan`（`@SpringBootApplication` 已包含）扫描指定包下的类，当扫描到标注 `@Component` 的类时，会将其识别为**组件类**。容器会为这个类生成对应的 `BeanDefinition`（存储类的全限定名、作用域、依赖等元数据），存入注册表。
2. 容器根据 `BeanDefinition` 的信息，通过**反射调用类的构造器**([Java反射机制](JAVA理论.md))，创建该类的对象实例（默认是单例）
3. 将创建好的实例（Bean）放入容器的缓存中，后续通过 `@Autowired`或`@Resource` 就能从容器中取出这个实例进行注入。

> [!NOTE] `@Autowired`和`@Resource`的区别
> 
>|特性|@Autowired|@Resource|
>|---|---|---|
>|所属规范|Spring 原生注解|JDK 标准注解（`javax.annotation`）|
>|默认匹配规则|**按类型** → 按名称（需 `@Qualifier`）|**按名称** → 按类型|
>|解决多实例冲突|搭配 `@Qualifier("beanName")`|直接设置 `name = "beanName"`|
>|支持的注入位置|字段、构造器、Setter|字段、Setter（不支持构造器）|
>|非必须注入配置|`@Autowired(required = false)`|`@Resource(lookupOnStartup = false)`|

###### 生命周期

| 阶段             | 具体步骤                                                                                                                                                                                                                                   | 核心说明                                                                        |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| **1. 实例化**     | Spring 容器通过**反射调用无参构造器**，创建 Bean 实例（仅分配内存，属性为默认值）                                                                                                                                                                                      | 这是 Bean 生命周期的第一步，和 Java 对象 `new` 类似，但由容器触发。                                 |
| **2. 属性赋值**    | 把容器中创建好的Bean实例赋值给对应的变量（如 `@Autowired` 自动装配、XML `property` 配置）                                                                                                                                                                          | 完成 **DI（依赖注入）**，解决 Bean 之间的依赖关系。                                            |
| **3. 初始化前**    | 1. 执行 `BeanNameAware.setBeanName()`：注入 Bean 在容器中的名称<br><br>2. 执行 `BeanClassLoaderAware.setBeanClassLoader()`：注入类加载器<br><br>3. 执行 `BeanFactoryAware.setBeanFactory()`：注入当前 BeanFactory<br><br>4. 执行 `ApplicationContextAware` 相关方法（若实现） | 只有实现了 `*Aware` 接口的 Bean 才会执行，用于获取容器的核心组件。                                   |
| **4. 初始化前置处理** | 执行 `BeanPostProcessor.postProcessBeforeInitialization()`                                                                                                                                                                               | **全局扩展点**，对所有 Bean 生效，可修改 Bean 属性、替换 Bean 实例。                               |
| **5. 初始化**     | 1. 执行 注解了`@PostConstruct` 的方法（JSR-250 标准，优先级最高）<br><br>2. 如果实现 `InitializingBean`接口，调用`afterPropertiesSet()`方法<br><br>3. 执行自定义初始化方法（如 XML `init-method` 属性）                                                                            | 这是 Bean 初始化的核心步骤，用于执行业务初始化逻辑（如加载资源、连接数据库）。                                  |
| **6. 初始化后置处理** | 执行 `BeanPostProcessor.postProcessAfterInitialization()`                                                                                                                                                                                | **AOP 动态代理的核心生成位置**，Spring 在此为 Bean 创建代理对象（如事务、日志切面）。目标类有接口用 JDK，无接口用 CGLIB |
| **7. 销毁**      | 1. 容器关闭时，执行 `@PreDestroy` 注解方法<br><br>2. 执行 `DisposableBean.destroy()` 接口方法<br><br>3. 执行自定义销毁方法（如 XML `destroy-method` 属性）                                                                                                             | 用于释放资源（如关闭连接、清理缓存），仅**单例 Bean** 会执行（原型 Bean 由用户手动销毁）。                       |
###### 作用域
>决定Bean实例的数量和存活范围，默认为singleton

| 作用域             | 说明                                                   | 适用场景                              | 线程安全                             |
| --------------- | ---------------------------------------------------- | --------------------------------- | -------------------------------- |
| **singleton**   | 单例，容器中**仅一个实例**，全局共享。容器启动时创建（懒加载除外）。                 | 无状态 Bean（如工具类、Service）            | **不安全**，多线程并发修改其**成员变量**会导致数据不一致 |
| **prototype**   | 原型，每次获取 Bean（`getBean()`）都创建**新实例**，容器仅负责创建，不管理销毁。   | 有状态 Bean（如 Request 相关对象）          | 安全（每次新实例）                        |
| **request**     | 每个 HTTP 请求创建一个实例，仅在 Web 环境生效。                        | Web 层请求级对象（如 Controller 内的请求数据载体） | 安全                               |
| **session**     | 每个 HTTP Session 创建一个实例，仅 Web 环境生效。                   | 会话级对象（如用户登录信息）                    | 安全（单会话内单例）                       |
| **application** | 整个 Web 应用一个实例，与 `singleton` 类似，但作用域是 ServletContext。 | 全局应用级对象                           | 不安全                              |
| **websocket**   | 每个 WebSocket 连接一个实例，仅 WebSocket 环境生效。                | 实时通信场景                            | 安全                               |

> [!NOTE]
> **`singleton` 懒加载如何配置？**
> 通过 `@Lazy(true)` 注解（默认 `false`），容器启动时不创建 Bean，第一次 `getBean()` 时才实例化。
###### 依赖注入（Dependency Injection）方式
- 构造器注入
- Setter注入
- 自动装配
#### Spring核心思想
###### AOP
>即面向切面编程，在Spring中用于将那些**与业务无关**但**对多个对象产生影响**的**公共行为和逻辑**抽取出来，实现公共模块复用，**降低耦合**。常见的应用场景包括公共日志保存和事务处理。
- **核心概念**
1. 横切关注点：多个核心业务中重复出现的通用逻辑
2. 切面(aspect) ：封装横切关注点的类 / 模块
3. 连接点(joinpoint)：程序执行过程中能被 AOP 拦截的**所有可能位置**
4. 切入点(pointcut)：从所有连接点中**筛选出需要执行切面逻辑的具体位置**
5. 通知(advice)：指的就是指拦截到连接点之后要执行的代码，通知分为前置（Before)、后置(After)、异常(AfterThrowing)、最终(AfterReturning)、环绕(Around)通知五类

- **作用**
1. 在目标方法执行的**任意时机**（前置 / 后置 / 环绕 / 异常）添加自定义逻辑，实现功能扩展
2. 通过切面拦截方法调用，根据规则**控制方法是否执行**，实现权限校验、流量控制等
3. 在不侵入业务代码的前提下，监控方法的执行状态，辅助调试和问题排查

- **执行流程**
>Spring AOP 是**运行时织入**，所有逻辑都在项目运行时完成，和 Bean 生命周期深度绑定
1. **扫描切面，解析元数据**
    - Spring 启动时，扫描标注 `@Aspect` 的切面类，解析其中的 `@Pointcut` 切入点表达式、`@Before`/`@Around` 等通知注解。
    - 将切入点和通知封装为 `Advisor`（切面执行器），存入 IoC 容器。
2. **动态生成代理对象（核心步骤）**
    - 当 IoC 容器创建目标 Bean（如 `UserService`）时，会在 **Bean 初始化后置处理阶段**（`BeanPostProcessor.postProcessAfterInitialization`）判断：当前 Bean 的方法是否匹配切入点。
    - 若匹配，Spring 会通过 **JDK 动态代理或 CGLIB 动态代理** 生成代理对象：
        - 目标类**实现接口** → 用 JDK 动态代理
        - 目标类**无接口** → 用 CGLIB 动态代理
    - 最终 IoC 容器中存储的是**代理对象**，而非原始目标对象。
3. **方法调用时，织入增强逻辑**
    - 客户端调用目标方法，实际调用的是**代理对象的方法**。
    - 代理对象先执行切面的增强逻辑（如前置日志），再调用目标对象的原始方法，最后执行后续增强逻辑（如后置事务）。
###### IoC
>Inversion of Control，即控制反转，是一种设计思想。在Java开发中，使用IoC后，我们不需要自己去创建某个类的实例，而由**IoC容器**去创建，当我们需要使用某个对象时，直接到容器中去获取就可以了(依赖注入或者实现 ApplicationContextAware 接口，手动从IoC容器中获取Bean)
#### 循环依赖
>循环依赖发生在两个或两个以上的bean互相持有对方，**形成闭环**
- 属性循环依赖解决方式：**三级缓存**
	1. 实例化A对象，并创建`ObjectFactory`存入三级缓存。
	2. A在初始化时需要B对象，开始B的创建逻辑。
	3. B实例化完成，也创建`ObjectFactory`存入三级缓存。
		1. B需要注入A，通过三级缓存获取`ObjectFactory`得到提前暴露的A引用（AOP过程则是获取A的代理对象），存入二级缓存。
	4. B通过二级缓存获得A对象后，B创建成功，存入一级缓存。
	5. A对象初始化时，由于B已创建完成，可以直接注入B，A创建成功存入一级缓存。
	6. 清除二级缓存中的临时对象A。
- 构造器的循环依赖无法解决

#### 事务
>Spring实现事务的本质是利用AOP完成的。它对方法前后进行拦截，在执行方法前开启事务，在执行完目标方法后根据执行情况提交或回滚事务。(@Transactional启用)
- 事务失效场景
1. 如果方法**内部捕获并处理了异常，没有将异常抛出**，会导致事务失效。因此，处理异常后应该确保异常能够被抛出。
2. 如果方法抛出检查型异常（checked exception），并且没有在`@Transactional`注解上**配置`rollbackFor`属性为`Exception`**，那么异常发生时事务可能不会回滚。
3. 如果**事务注解的方法不是公开（public）修饰的**，也可能导致事务失效。因为Spring AOP动态代理默认只拦截public方法
4. 本类内部调用 `this.事务方法（）`
5. 当前类没有被spring管理

#### Spring中的设计模式
- 工厂模式 : Spring使用工厂模式通过 BeanFactory、ApplicationContext 创建 bean 对象。
- 代理模式 : Spring AOP 功能的实现。
- 单例模式 : Spring 中的 Bean 默认都是单例的。
- 模板方法模式 : Spring 中 jdbcTemplate、hibernateTemplate 等以 Template 结尾的对数据库操 作的类，它们就使用到了模板模式。
- 装饰器模式 : 我们的项目需要连接多个数据库，而且不同的客户在每次访问中根据需要会去访 问不同的数据库。这种模式让我们可以根据客户的需求能够动态切换不同的数据源。
- 观察者模式: Spring 事件驱动模型就是观察者模式很经典的一个应用。
- 适配器模式 :Spring AOP 的增强或通知(Advice)使用到了适配器模式、spring MVC 中也是用到了 适配器模式适配Controller。
## SpringMVC
#### 常用注解
```
`@RequestMapping`：映射请求路径。
    
`@RequestBody`：接收HTTP请求的JSON数据。
    
`@RequestParam`：指定请求参数名称。
    
`@PathVariable`：从请求路径中获取参数。
    
`@ResponseBody`：将Controller方法返回的对象转化为JSON。
    
`@RequestHeader`：获取请求头数据。
    
`@PostMapping`、`@GetMapping`等。
```

## SpringBoot

> [!NOTE] 与Spring的区别
> Spring一套包含 IOC 容器、AOP、SpringMVC 等多个模块的完整底层开发框架，配置时需要xml文件或者Java config注解。使用时需要开发者手动管理全部依赖版本，自行编写配置文件或者配置类开启组件扫描。项目中有大量重复样板的配置。而SpringBoot是基于Spring打造一个帮助快速开发的脚手架。只是通过例如Starter启动器统一管理版本、依据自动装配机制根据项目引入的依赖自动配置Bean、自带web容器等。依托 @SpringBootApplication 复合注解自动完成包扫描与配置加载，无需开发者手动编写大量配置代码，消除原生 Spring 繁琐的配置与环境搭建工作，提升开发效率

#### 常用注解
```
1. @SpringBootApplication：
用于标识主应用程序类，通常位于项目的顶级包中。这个注解包含了@Configuration、@EnableAutoConfiguration和@ComponentScan。

2. @Controller：
用于标识类作为Spring MVC的Controller。

3. @RestController：
类似于@Controller，但它是专门用于RESTful Web服务的。它包含了@Controller和@ResponseBody。

4. @RequestMapping：
用于将HTTP请求映射到Controller的处理方法。可以用在类级别和方法级别。

5. @Autowired：
用于自动注入Spring容器中的Bean，可以用在构造方法、字段、Setter方法上。

6. @Service：
用于标识类作为服务层的Bean。

7. @Repository：
用于标识类作为数据访问层的Bean，通常用于与数据库交互。

8. @Component：
通用的组件注解，用于标识任何Spring托管的Bean。

9. @Configuration：
用于定义配置类，类中可能包含一些@Bean注解用于定义Bean。

10. @EnableAutoConfiguration：
用于启用Spring Boot的自动配置机制，根据项目的依赖和配置自动配置Spring应用程序。

11. @Value：
用于从属性文件或配置中读取值，将值注入到成员变量中。

12.@Qualifier：
与@Autowired一起使用，指定注入时使用的Bean名称。

13. @ConfigurationProperties：
用于将配置文件中的属性映射到Java Bean。

14. @Profile：
用于定义不同环境下的配置，可以标识在类或方法上。

15. @Async：
用于将方法标记为异步执行。

16.@AllArgsConstructor
自动为类生成包含所有成员变量的全参构造器

17.@slf4j
自动生成日志对象
```

#### 开箱即用的原因
1. **起步依赖**：与框架相关的依赖都打包成了一个starter包，由SpringBoot内置的仲裁机制统一管理，不用手动管理依赖版本号
2. **自动装配**： 根据项目引入的依赖，**自动装配框架的核心组件和参数**，无需手动编写 XML 或 Java 配置类。
3. **默认配置**：Spring Boot 为所有框架提供 **合理的默认参数**，无需手动配置即可运行
4. **内置嵌入式容器**：内置 Tomcat、Jetty 等 Servlet 容器，项目可直接通过 `java -jar` 启动，无需手动部署到外部容器

#### 自动装配流程
- **启动触发**：SpringBoot 启动类的 `@SpringBootApplication` 注解，核心依赖 `@EnableAutoConfiguration` 通过 `@Import(AutoConfigurationImportSelector.class)` 导入选择器
- **加载候选类**：`AutoConfigurationImportSelector` 会调用 `SpringFactoriesLoader.loadFactoryNames()`，从 **所有 Jar 包** 的 `META-INF/spring.factories` 文件中，读取 `EnableAutoConfiguration` 键对应的全类名列表
- **条件过滤**：加载的候选类很多，通过 `@ConditionalOnClass`/`@ConditionalOnBean` 等条件注解，筛选出符合当前项目依赖和环境的有效配置类。
- **注入容器**：有效配置类被解析，其 `@Bean` 方法生成组件并注入 Spring 容器。
- **用户覆盖**：遵循「用户配置优先」原则，自定义 Bean / 配置会覆盖自动配置的默认值。
## Mybatis
#### 执行流程
- **步骤1：每次启动程序时执行**
1. 读取MyBatis配置文件`mybatis-config.xml`。
2. 构造会话工厂`SqlSessionFactory`,封装数据源和配置信息。
- **步骤2：每次操作数据库时执行**
1. 会话工厂创建`SqlSession`对象。
2. 通过动态代理创建 Mapper 代理对象（底层用 JDK 动态代理），代理对象会关联 `SqlSession`，负责将接口方法映射到具体的 SQL。
3. MyBatis 解析 Mapper 接口或 XML 文件中的 SQL 配置（如 `select`/`insert` 标签），将接口方法与 SQL 语句绑定，同时处理参数映射（#{} 占位符）
4. `SqlSession` 内部创建 `Executor`（执行器，MyBatis 的核心执行引擎），负责 SQL 的执行
5. `Executor` 通过 `StatementHandler` 构建 `PreparedStatement`，将参数填充到 SQL 中，发送给数据库执行。
6. 数据库返回结果后，MyBatis 通过 `ResultSetHandler` 将结果集映射为 Java 对象（实体类），最终通过 Mapper 代理对象返回给调用方
#### 延迟加载
>需要用到数据时才加载。可以通过配置文件中的`lazyLoadingEnabled`配置启用或禁用延迟加载
- 原理：
	 1. 使用CGLIB创建目标对象的代理对象。
	2. 调用目标方法时，如果发现是null值，则执行SQL查询。
	3. 获取数据后，设置属性值并继续查询目标方法。
#### 一二级缓存
###### 一级缓存
- **作用域**：`SqlSession` 级别，同一个 `SqlSession` 内共享缓存，不同 `SqlSession` 相互隔离。（为SqlSession私有属性）
- **工作机制**
    1. 同一个 `SqlSession` 中，首次查询数据时，MyBatis 会将查询结果存入一级缓存。
    2. 后续相同的 SQL 查询（参数、SQL 语句、分页等完全一致），直接从缓存获取，不查数据库。
    3. 当执行 **增删改操作** 或调用 `sqlSession.clearCache()` 时，一级缓存会被**清空**，避免脏数据。
- **特点**：基于 `PerpetualCache` 实现（本质是 `HashMap`），轻量高效，无需手动配置。
###### 二级缓存
- **作用域**：`Mapper` 级别（`namespace` 级别），多个 `SqlSession` 共享同一个 `namespace` 的缓存。
- **开启步骤**
    1. 全局配置开启：`mybatis.configuration.cache-enabled=true`（默认 true，可省略）。
    2. Mapper 接口 / XML 中添加 `@CacheNamespace` 注解 或 `<cache/>` 标签。
    3. 实体类必须实现 `Serializable` 接口（缓存数据可能序列化存储）。
- **工作机制**
    1. 当 `SqlSession` 执行查询并**提交（`commit()`）** 后，查询结果会存入二级缓存。
    2. 其他 `SqlSession` 查询相同 `namespace` 的相同 SQL，优先从二级缓存获取。
    3. 同一 `namespace` 内执行增删改操作，二级缓存会被清空。
- **特点**
    - 支持自定义缓存实现（如整合 Redis、Ehcache），默认是 `PerpetualCache`。
    - 可通过 `eviction`（回收策略）、`flushInterval`（刷新间隔）、`size`（缓存大小）等参数配置
###### 数据库查询顺序
二级缓存 -> 一级缓存 ->数据库
#### 分页查询
###### 手动拼接limit语句
###### MybatisPlus实现
- 配置分页拦截器
```java
@Configuration 
public class MyBatisPlusConfig { 
@Bean 
	public MybatisPlusInterceptor mybatisPlusInterceptor() {MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor(); 
// 添加分页拦截器（MySQL 方言） 
	interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL)); 
	return interceptor; 
	} 
}
```
- 业务层代码
```java
// 1. 构建分页对象：页码、每页条数 
Page<User> page = new Page<>(1, 10); 
// 2. 执行分页查询（MP 自动拼接 LIMIT 和 COUNT） 
IPage<User> userPage = userMapper.selectPage(page, null); // 3. 获取结果 
List<User> userList = userPage.getRecords(); 
long total = userPage.getTotal();
```
###### 性能优化
- 避免SELECT *，只查询需要的字段
- 大量数据分页可以通过主键游标分页优化