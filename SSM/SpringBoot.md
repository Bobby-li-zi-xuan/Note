### 快速启动
1. 对springboot项目打包
2. 执行启动指令 
`java - jar jar包名.jar`
	- jar支持命令行启动需要maven插件支持，打包时需要提前配置插件

### 配置文件
（加载优先级从上到下）
- application.properties
- application.yml
- application.yaml
##### yaml
- 一种数据序列化格式
- 优点
	- 容易阅读
	- 容易与脚本语言交互
	- 以数据为核心，重数据轻格式
- 语法规则
	- 大小写敏感
	- ’#‘表示注释
	- **属性值前要添加空格**
###### 数据读取方式
1. 使用@Value读取单个数据
<img src="Pasted image 20260307105338.png" alt="alt text" width="800">
2. 封装全部数据到Environment对象
<img src="Pasted image 20260307105522.png" alt="alt text" width="800">
3. 使用自定义实体类封装数据
<img src="Pasted image 20260307105837.png" alt="alt text" width="800">
警告解决
<img src="Pasted image 20260307110244.png" alt="alt text" width="800">
### 多环境启动
- 设置启用环境
```
spring:
	profiles:
		active: test

spring:
	#设置环境名称
	profiles: 

spring:
	profiles:
```
- 命令启动格式
	- 带参数启动
`javal -jar springboot.jar --spring.profiles.active=test`

### 整合第三方技术
###### 整合JUnit
@SpringBootTest
- 测试类注解，定义在测试类上方
`@SpringBootTests(classes= )
###### 整合Mybatis
- 创建新模块时选择mybatis和mysql的数据集
- 设置数据源参数
```
spring:
	datasource:
		typy:
		driver-class-name:
		url:
		username:
		password:
```