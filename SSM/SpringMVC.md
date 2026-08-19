### 概念：一种基于Java实现MVC模型的亲两级Web框架

### SpringMVC配置步骤 
1. <img src="image.png" alt="alt text" width="400">
2. 创建SpringMVC控制器类（等同于Servlet功能）
~~~java
@Controller
public class UserController{
    @RequestMapping("/save")            //页面调用需要的访问路径
    @ResponseBode
    //功能1实现
    public String save(){
        system.out.println("user save ...");
        return "{'info' : 'springmvc'}";
    }

    @RequestMapping("/delete")            //页面调用需要的访问路径
    @ResponseBode
    //功能2实现
    public String save(){
        system.out.println("user delete ...");
        return "{'info' : 'springmvc'}";
    }

    ...
}
~~~

3. 初始化SpringMVC环境，设定SpringMVC加载对于的bean
```java
@Configuration
@ComponentScan("com..")         //要扫描的包的路径
public class SpringMvcConfig{
}
```

4. 初始化**Servlet容器**，加载SpringMVC环境，并设置SpringMVC技术处理的请求
```java
public class ServletContainersInitConfig extends AbstractDispatcherServletInitializer{
    
    //告诉Tomcat容器要加载sring配置
    protected WebApplicationContext createServletApplicationContext(){
        AnnotationConfigWebApplicationContext ctx = new AnnotationConfigWebApplicationContext();
        ctx.register(SpringMvcConfig.class);
        return ctx;
    }

    //设置哪些请求给SpringMVC去处理
    protected String[] getServletMappings(){
        return new String[]{"/"};
    }
    //加载spring容器配置
    protected WebApplicationContext createRootApplicationContext(){
        AnnotationConfigWebApplicationContext ctx = new AnnotationConfigWebApplicationContext();
        ctx.register(SpringConfig.class);
        return ctx;'
    }
}
```
简便写法
```java
public class ServletContainersInitConfig extends AbstractAnnotationConfigDispatcherServletInitializer{
    protected Class<?>[] getRootConfigClasses(){
        return new Class[]{SpringConfig.class};
    }

    protected Class<?>[] getServletConfigClasses(){
        return new Class[]{SpringMvcConfig.class};
    }

    protected Class<?>[] getServletMappings(){
        return new String[]{"/"};
    }
}
```

### 启动服务器初始化过程
1. 服务器启动，执行ServletContainersInitConfig类，初始化Web容器
2. 执行createServletApplicationContext方法，创建了WebApplicationContext对象
3. 注册容器加载SpringMvcConfig
4. 执行@ComponentScan加载对应的bean（配置文件中）
5. 加载UserController，每个@RequestMapping的名称对应一个具体的方法
6. 执行getServletMapping方法，定义所有的请求都通过SpringMVC

### 单次请求过程
1. 发送请求localhost/save
2. web容器发现所有请求都经过SpringMVC，将请求交给SpirngMVC处理
3. 解析请求路径/save
4. 由/save匹配执行对应的方法save()
5. 执行save()
6. 检测到由@ResponseBode直接将save()方法的返回值作为响应请求体返回给请求方

### Controller加载控制与业务bean加载机制
因为***功能不同***，如何避免Spring错误地加载到SpringMVC控制的bean
- 方法一：Spring加载的bean设定扫描范围时排除掉SpringMVC控制的bean
```java
@Configuration
@ComponentScan(value="com..",
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.ANNOTATION，            //按注解过滤
        classes = Controller.class
    ))         
public class SpringConfig{
}
```  

- 方法二：Spring加载的bean设定扫描范围为精准范围
```java
@Configuration
@ComponentScan({"com.."，"com.."})         //数组形式定义要扫描的多个包的路径
public class SpringConfig{
}
```

## 请求
### 1.请求映射路径@RequestMapping
作用：设置当前控制器方法请求访问路径，如果设置在类上方统一设置当前控制器方法请求访问路径前缀

### 2.Get请求传参
- 普通参数：url传地址，地址参数名与形参变量名相同，定义形参即可接收参数
```java
@RequestMapping("/commonParam")
@ResponseBody
public String commonParam(String name,int age){
    return "{'module':'common param'}";
}
```
- 普通参数：请求参数名与形参变量名不同，使用@RequestParam绑定参数关系
```java
@RequestMapping("/commonParam")
@ResponseBody
public String commonParam(@RequestParam("name")String name,int age){
    return "{'module':'common param'}";
}
```
- POJO参数：请求参数名与形参对象属性名相同（例中就是User实体类中的属性:name和age）
```java
@RequestMapping("/pojoParam")
@ResponseBody
public String commonParam(User user){
    return "{'module':'pojo param'}";
}
```
- 数组参数：形参定义为数组类型
```java
@RequestMapping("/arrayParam")
@ResponseBody
public String commonParam(String[] likes){
    return "{'module':'array param'}";
}
```

- 集合保存普通参数
```java
@RequestMapping("/commonParam")
@ResponseBody
public String commonParam(@RequestParam List<String likes>){
    return "{'module':'list param'}";
}
```

### 3.Post请求传参
- 普通参数：form表单post请求传参，表单参数名与形参变量名相同，定义形参即可接收参数

### 4.乱码处理(定义在配置类ServletContainersInitConfig中)
添加过滤器并指定字符集
```java
@Override
protected Filter[] getServletFilters(){
    ChrarcterEncodingFilter filter = new CharacterEncodingFilter();
    filter,setEncoding("UTF-8");
    return new Filter[]{filter}
}
```

### 5.json数据参数传递
1. pom.xml文件中加入com.fasterxml.jackson.core坐标
2. 接口测试软件中发送json数据
3. 在SpringMVC配置文件中开启自动转换json数据的注解@EnableWevMvc(可以根据类型匹配对应的类型转换器)
4. 设置接收json数据，参数前加@requestBody（原来@RequestParam的位置）
#### @ResponseBody作用：设置当前控制器方法的返回值作为响应体


### 6.日期型参数传递
接收形参时根据不同的日期格式设置不同的接收方式
```java
@RequestMapping("/dataParam")
@ResponseBody
public String dataParam(Date date,
                    @DateTimeFormat(pattern = "yyyy-MM-dd") Date date1,
                    @DateTimeFormat(pattern = "yyyy/MM/dd HH:mm:ss) Date date2
){
return "{'module': 'date param'}";
}
```

## 响应
- 响应json数据
```java
@RequestMapping("/url")
@ResponseBody
public User toJsonPOJO(){
    User user = new User();
    user.setName("XXX");
    user.setAge(40);
    return user;
}
```
## REST风格
### 概念：Representation State Transfer，表现形式状态转换
### 优点：
- 隐藏资源的访问行为，无法通过地址得知对资源是什么操作
- 书写简化

### 步骤
1. 设定http请求动作
@RequestMapping(value = "/模块名", method = RequestMethod.POST) 描述模块的名称通常用复数
- GET 查询
- POST 新增、保存
- PUT 修改、更新
- DELETE 删除

2. 设置请求参数（路径变量:查询或删除单个时需提供，传入id）
```java
@RequestMapping(value = "/模块名/{参数名}", method = RequestMethod.GET)
@ResponseBody
public String delete(@PathVariable T 参数名){
    return "";
}
```
#### @PathVariable:绑定路径参数与控制器方法形参间的关系，要求路径参数名与形参名一一对应
### REST快速开发
- 所有控制器方法前的@ResponseBody提取到控制类前
- @RestController等同于@Controller加上@ResponseBody
- @RequestMapping(value = "/模块名/{参数名}", method = RequestMethod.GET)简化为@GetMapping("/{id}")

- 设置对静态页面的访问放行([SpringMVC配置步骤](#springmvc配置步骤)第四步中`return new String[]{"/"};`拦截了所有请求给SpringMVC）<br>
新定义一个配置类
```java
@Configuration
public class SpringMvcSupport extends WebMvcConfigurationSupport{
    @Override
    protected void addResourceHandlers(ResourceHandlerRegistry registry){
        //当访问/page/????的时候，走/page目录下的内容
        register.addResourceHandlers("/pages/**").addResourceLocations("/pages/");
    }
}
```

## 拦截器
### 概念：动态拦截方法调用的机制，在SpringMVC中动态拦截控制器方法的执行
### 作用：
- 在指定的方法调用前后执行预先设定的代码
- 组织原始方法的执行

与过滤器的区别
- 归属不同：Filter属于Servlet技术，Interceptor属于SpringMVC技术
- 拦截内容不同：Filter对所有访问进行增强，Interceptor仅针对SpringMVC的访问进行增强

### 步骤
1. 配置拦截器功能类(在controller包下)
```java
@Component
public class ProjectInterceptor implements HandlerInterceptor{
    //拦截原始操作之前执行的代码，返回false即终止原始操作的运行
    @Override 
    public boolean preHandle(HttpServletRequest request,HttpServletResponse response, Object handler) throws Exception{
        return true;
    }
    //拦截原始操作之后执行的代码
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse reponse, Object handler, ModelAndView modelAndview){

    }
    //在postHandle之后执行的代码
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception{

    }

}
```
2. 设置拦截器的执行位置（需在SpringMVC配置类中@ComponentScan中加上config包路径）<br>
定义在config包中
```java
@Configuration
public class SpringMvcSupport extends WebMvcConfigurationSupport{
    //拦截器对象在容器中，需要自动装配
    @Autowired
    private ProjectInterceptor projectInterceptor1;//拦截器1
    @Autowired
    private ProjectInterceptor projectInterceptor2;//拦截器2

    //放行静态资源
    @Override
    protected void addResourceHandlers(ResourceHandlerRegistry registry){
        //当访问/page/????的时候，走/page目录下的内容
        register.addResourceHandlers("/pages/**").addResourceLocations("/pages/");
    }

    //拦截器
    @Override
    protected void addInterceptors(InterceptorRegistry registry){
        registry.addInterceptor(projectInterceptor1).addPathPatterns("/path");        //要拦截的路径,可以设置多个
        registry.addInterceptor(projectInterceptor1).addPathPatterns("/path");
    }
}
```

### 拦截器参数
- handler:被调用的处理器对象，本质上是一个方法对象，对反射技术中的Method对象进行了再包装

### 拦截器链
1. 运行顺序参照拦截器添加顺序
2. 当拦截器中出现对原始操作的拦截，后面的拦截器均终止运行
3. 当拦截器中断，仅运行配置在前面的拦截器的afterCompletion操作