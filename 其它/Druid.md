### Druid

java实现的数据库连接池有很多，c3p0,dbcp等，还有号称速度最快的HikariCP，并且springboot2.0.2版本默认使用的就是HikariCP。
为什么选用Druid呢？
- 性能够好，比c3p0,dbcp强一些
- 经过考验，毕竟是阿里开源出来的项目
- 最关键的是带一个建议的数据库监控

### 能监控哪些数据呢?

1. 数据源 

2. SQL监控 对执行的MySQL语句进行记录，并记录执行时间、事务次数等 

3. SQL防火墙 对SQL进行预编译，并统计该条SQL的数据指标 

4. Web应用 对发布的服务进行监控，统计访问次数，并发数等全局信息 

5. URI监控 对访问的URI进行统计，记录次数，并发数，执行jdbc数等 

6. Session监控 对用户请求后保存在服务器端的session进行记录，识别出每个用户访问了多少次数据库等 

7. Spring监控 （按需配置）利用aop对各个内容接口的执行时间、jdbc数进行记录

### Druid的工程结构

![img](https://gitee.com/wang-xiangtai11/typora-figure/raw/master/img/202112191444026.png)

统计相关的内容都在stat包中，比较核心的就是DruidStatService，很多服务通过这个类暴露出来，例如统计数据获取等。元数据还可以通过DruidStatManagerFacade获取。

统计页面主要是通过StatViewServlet来注册的，这个类的集成结构是 StatViewServlet -> ResourceServlet -> HttpServlet,就是这个Servlet来映射相关的统计界面的，关于监控页面的源html,可查看support包下的http->resourses

Filter插件，stat功能（监控）、wall功能（sql防火墙）、log4j2功能（监控日志输出），都是以插件的形式配置的

### 如何配置？

想达到的目标效果，监控sql，监控sql防火墙，监控url，监控session，监控spring 
先说明一下，监控sql、监控url、基础信息，几乎不怎么需要配置，集成好druid，配置好监控页面，就可以显示

配置大概分为3部分，基础连接池配置，基础监控配置，定制化监控配置

#### 1. 完全使用 .properties配置文件

maven的pom文件引入依赖

```java
只引入这个starter即可
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.1.9</version>
</dependency>
 
为什么不引入源druid依赖
用IDEA去看 druid-spring-boot-starter项目的pom文件中已经引入了源工程，请勿多此一举，会导致依赖冲突
```

基础连接池配置，主要是配置数据库的账户密码，还有连接池的参数

```java
spring.datasource.url=jdbc:mysql://数据库的IP:3306/数据库名?characterEncoding=utf-8&useSSL=false&useUnicode=true
spring.datasource.username=账户
spring.datasource.password=密码
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
# 连接池指定 springboot2.02版本默认使用HikariCP 此处要替换成Druid
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
 
## 初始化连接池的连接数量 大小，最小，最大
spring.datasource.druid.initialSize=5
spring.datasource.druid.minIdle=5
spring.datasource.druid.maxActive=20
## 配置获取连接等待超时的时间
spring.datasource.druid.maxWait=60000
# 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒
spring.datasource.druid.timeBetweenEvictionRunsMillis=60000
# 配置一个连接在池中最小生存的时间，单位是毫秒
spring.datasource.druid.minEvictableIdleTimeMillis=300000
spring.datasource.druid.validationQuery=SELECT 1 FROM DUAL
spring.datasource.druid.testWhileIdle=true
spring.datasource.druid.testOnBorrow=false
spring.datasource.druid.testOnReturn=false
# 是否缓存preparedStatement，也就是PSCache  官方建议MySQL下建议关闭   个人建议如果想用SQL防火墙 建议打开
spring.datasource.druid.poolPreparedStatements=true
spring.datasource.druid.maxPoolPreparedStatementPerConnectionSize=20
# 配置监控统计拦截的filters，去掉后监控界面sql无法统计，'wall'用于防火墙
spring.datasource.druid.filters=stat,wall,log4j
# 通过connectProperties属性来打开mergeSql功能；慢SQL记录
spring.datasource.druid.connectionProperties=druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000
# ！！！请勿配置timeBetweenLogStatsMillis 会定时输出日志 并导致统计的sql清零
#spring.datasource.druid.timeBetweenLogStatsMillis=20000
```

基础监控配置（主要是配置监控的身份验证信息，毕竟系统运行状态也是个小秘密）

```java
# WebStatFilter配置，说明请参考Druid Wiki，配置_配置WebStatFilter
#是否启用StatFilter默认值true
spring.datasource.druid.web-stat-filter.enabled=true
##spring.datasource.druid.web-stat-filter.url-pattern=
spring.datasource.druid.web-stat-filter.exclusions=*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*
 
 
 
# StatViewServlet配置，说明请参考Druid Wiki，配置_StatViewServlet配置
#是否启用StatViewServlet默认值true
spring.datasource.druid.stat-view-servlet.enabled=true
spring.datasource.druid.stat-view-servlet.url-pattern=/druid/*
spring.datasource.druid.stat-view-servlet.reset-enable=false
spring.datasource.druid.stat-view-servlet.login-username=admin
spring.datasource.druid.stat-view-servlet.login-password=123456
```

其他进阶配置： 

1. Spring监控，对内部各接口调用的监控

```java
spring.datasource.druid.aop-patterns=com.company.project.service.*,com.company.project.dao.*,com.company.project.controller.*,com.company.project.mapper.*
```

禁止手动重置监控数据

```java
spring.datasource.druid.stat-view-servlet.reset-enable=false
```

设置监控页面的登录名和密码

```java
spring.datasource.druid.stat-view-servlet.login-username=admin
spring.datasource.druid.stat-view-servlet.login-password=123456
```

设置不统计哪些URL

```java
spring.datasource.druid.web-stat-filter.exclusions=*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*
```

设置使用哪些插件 
stat是统计，wall是SQL防火墙，防SQL注入的，log4j是用来输出统计数据的（我觉得这个没啥用，输出日志我又重写了个工具类）

```java
spring.datasource.druid.filters=stat,wall,log4j
```

# Druid连接池的配置信息

# 连接池指定 springboot2.02版本默认使用HikariCP 此处要替换成Druid

spring.datasource.type=com.alibaba.druid.pool.DruidDataSource

## 初始化大小，最小，最大

spring.datasource.druid.initialSize=5 
spring.datasource.druid.minIdle=5 
spring.datasource.druid.maxActive=20

## 配置获取连接等待超时的时间

spring.datasource.druid.maxWait=60000

# 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒

spring.datasource.druid.timeBetweenEvictionRunsMillis=60000

# 配置一个连接在池中最小生存的时间，单位是毫秒

spring.datasource.druid.minEvictableIdleTimeMillis=300000 
spring.datasource.druid.validationQuery=SELECT 1 FROM DUAL 
spring.datasource.druid.testWhileIdle=true 
spring.datasource.druid.testOnBorrow=false 
spring.datasource.druid.testOnReturn=false

# 是否缓存preparedStatement，也就是PSCache MySQL下建议关闭

spring.datasource.druid.poolPreparedStatements=true 
spring.datasource.druid.maxPoolPreparedStatementPerConnectionSize=20

# 配置监控统计拦截的filters，去掉后监控界面sql无法统计，’wall’用于防火墙

spring.datasource.druid.filters=stat,wall,log4j

# 通过connectProperties属性来打开mergeSql功能；慢SQL记录

spring.datasource.druid.connectionProperties=druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000

# ！！！请勿配置timeBetweenLogStatsMillis 会定时输出日志 并导致统计的sql清零

# spring.datasource.druid.timeBetweenLogStatsMillis=20000

# 配置Druid的日志输出

spring.datasource.druid.filter.slf4j.enabled=true 
spring.datasource.druid.filter.slf4j.statement-create-after-log-enabled=false 
spring.datasource.druid.filter.slf4j.statement-close-after-log-enabled=false 
spring.datasource.druid.filter.slf4j.result-set-open-after-log-enabled=false 
spring.datasource.druid.filter.slf4j.result-set-close-after-log-enabled=false

# WebStatFilter配置，说明请参考Druid Wiki，配置_配置WebStatFilter

# 是否启用StatFilter默认值true

spring.datasource.druid.web-stat-filter.enabled=true

## spring.datasource.druid.web-stat-filter.url-pattern=

spring.datasource.druid.web-stat-filter.exclusions=.js,.gif,.jpg,.png,.css,.ico,/druid/* 
spring.datasource.druid.web-stat-filter.session-stat-enable=true 
spring.datasource.druid.web-stat-filter.session-stat-max-count=100

# 身份标识 从session中获取

# spring.datasource.druid.web-stat-filter.principal-session-name=

# 身份标识 从cookie中获取 例如cookie中存gk=xiaoming 设置属性为gk即可

# user信息保存在cookie中，你可以配置principalCookieName，使得druid知道当前的user是谁

# spring.datasource.druid.web-stat-filter.principal-cookie-name=

# 配置profileEnable能够监控单个url调用的sql列表。

spring.datasource.druid.web-stat-filter.profile-enable=true

# StatViewServlet配置，说明请参考Druid Wiki，配置_StatViewServlet配置

# 是否启用StatViewServlet默认值true

spring.datasource.druid.stat-view-servlet.enabled=true 
spring.datasource.druid.stat-view-servlet.url-pattern=/druid/* 
spring.datasource.druid.stat-view-servlet.reset-enable=false 
spring.datasource.druid.stat-view-servlet.login-username=admin 
spring.datasource.druid.stat-view-servlet.login-password=123456

# spring.datasource.druid.stat-view-servlet.allow=

# spring.datasource.druid.stat-view-servlet.deny=

# Spring监控配置，说明请参考Druid Github Wiki，配置_Druid和Spring关联监控配置

# Spring监控AOP切入点，如x.y.z.service.*,配置多个英文逗号分隔

spring.datasource.druid.aop-patterns=com.company.project.service.,com.company.project.dao.,com.company.project.controller.,com.company.project.mapper.