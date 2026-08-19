### 基于SpringBoot整合MP
1. 创建模块时只需选择MySQL
2. 添加mp起步依赖
3. 设置jdbc参数（application.yaml）
4. 制作实体类，创建表
>@Data：为当前实体在编译期设置对应的get/set方法、tostring方法、equals方法等
5. 定义数据接口
```java
@Mappper
public interface UserDao extends BaseMapper<User>{}
```
### 标准数据层开发
<img src="Pasted image 20260308191040.png" alt="alt text" width="800">

### 分页功能
1. 设置分页拦截器作为Spring管理的Bean
```java
@Configuration
public class MpConfig{
	@Bean
	public MybatisPlusInterceptor pageInterceptor(){
		MybatisPlusInterceptor icp = new MybatisPlusInterceptor();
		icp.addInnerInterceptor(new PainationInnerInterceptor());
		return interceptor;
	}
}
```
2. 执行分页查询
### DQL编程控制
