# 什么是SpringBoot starter机制
SpringBoot中的starter是一种非常重要的机制(自动化配置)，能够抛弃以前繁杂的配置，将其统一集成进starter，应用者只需要在maven中引入starter依赖，SpringBoot就能自动扫描到要加载的信息并启动相应的默认配置。

starter让我们摆脱了各种依赖库的处理，需要配置各种信息的困扰。SpringBoot会自动通过classpath路径下的类发现需要的Bean，并注册进IOC容器。SpringBoot提供了针对日常企业应用研发各种场景的spring-boot-starter依赖模块。

所有这些依赖模块都遵循着约定成俗的默认配置，并允许我们调整这些配置，即遵循“约定大于配置”的理念。
# starter加载原理
springboot通过一个@SpringBootApplication注解启动项目，springboot在项目启动的时候，会将项目中所有声明为Bean对象（注解、xml）的实例信息全部加载到ioc容器当中。 除此之外也会将所有依赖到的starter里的bean信息加载到ioc容器中，从而做到所谓的零配置，开箱即用。

# 什么时候需要创建自定义starter

在我们的日常开发工作中，可能会需要开发一个通用模块，以供其它工程复用。SpringBoot就为我们提供这样的功能机制，我们可以把我们的通用模块封装成一个个starter，这样其它工程复用的时候只需要在pom中引用依赖即可，由SpringBoot为我们完成自动装配。

**常见场景：**

1. 通用模块-短信发送模块
2. 基于AOP技术实现日志切面
3. 分布式雪花ID，Long-->string，解决精度问题
4. 微服务项目的数据库连接池配置
5. 微服务项目的每个模块都要访问redis数据库，每个模块都要配置redisTemplate，也可以通过starter解决

starter 组件开发，核心是自动注解类的注解顺序，即根据条件进行注解

![4ed955cb2ab50c100dbd2d2d1c93542c_watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bCP5b2t5LiN5Lya56eD5aS0,size_20,color_FFFFFF,t_70,g_se,x_16](D:\soft\360极速浏览器\360chrome\User Data\temp\4ed955cb2ab50c100dbd2d2d1c93542c_watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bCP5b2t5LiN5Lya56eD5aS0,size_20,color_FFFFFF,t_70,g_se,x_16.png)

# 自定义starter的开发流程
1. 创建Starter项目(spring-initl 2.1.14)
2. 定义Starter需要的配置类(Properties)
3. 编写Starter项目的业务功能
4. 编写自动配置类
5. 编写spring.factories文件加载自动配置类
6. 打包安装
7. 其它项目引用

## 案例：AOP方式统一服务日志

> 原先实现统一日志都是放到每个工程中以AOP方式实现，现在有了starter方式，就可以将公司的日志规范集中到这里来统一管理。

## 导入aop相关依赖

web依赖，因为编写切面需要

```xml
 <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
 </dependency>
```

## 编写相关属性类(XxxProperties)：WebLogProperties.java

```java
@ConfigurationProperties("weblog")
@Data
public class WebLogProperties{
 
    public Boolean enabled;  //Boolean封装类，默认为null

}
```

## 编写Starter项目的业务功能:输出日志

```java
@Aspect
@Component
@Slf4j
public class WebLogAspect {
    @Pointcut("execution(* *..*Controller.*(..))")
    public void webLog() {
    }

    @Before("webLog()")
    public void doBefore(JoinPoint joinPoint) {
        // 接收到请求，记录请求内容
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();

        // 记录下请求内容
        log.info("开始服务:{}", request.getRequestURL().toString());
        log.info("客户端IP :{}", request.getRemoteAddr());
        log.info("参数值 :{}", Arrays.toString(joinPoint.getArgs()));
    }

    @AfterReturning(returning = "ret", pointcut = "webLog()")
    public void doAfterReturning(Object ret) {
        // 处理完请求，返回内容
        log.info("返回值 : {}", ret);
    }
}
```

## 编写自动配置类AutoConfig 

```java
//表示这个类为配置类
@Configuration
//使 使用 @ConfigurationProperties 注解的类生效
@EnableConfigurationProperties({WebLogProperties.class})
//matchIfMissing属性：默认情况下matchIfMissing为false，也就是说如果未进行属性配置，则自动配置不生效。如果matchIfMissing为true，则表示如果没有对应的属性配置，则也生效
@ConditionalOnProperty(prefix = "weblog",value = "enabled",matchIfMissing = true)
public class WebLogAutoConfig {
 
    @Bean
    @ConditionalOnMissingBean
    public WebLogAspect webLogAspect(){
        return new WebLogAspect();
    }
 
}
```

## 编写spring.factories文件加载自动配置类

在**src/main/resources/META-INF/spring.factories**里编写

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.WebLogAutoConfig
```

## 在其他项目中引用

### 引入依赖

```xml
<dependency>
            <groupId>com</groupId>
            <artifactId>web-log-springboot-starter</artifactId>
            <version>0.0.1-SNAPSHOT</version>
</dependency>
```

### 在application.yml文件中添加配置

```yaml
weblog:
  enabled: true
```

### 控制层

```java
@RestController
public class DemoController {

    @Autowired
    private DemoService demoService;

    @GetMapping("demo")
    public ResponseEntity demo() {
        String service = demoService.getService();
        return new ResponseEntity(service, HttpStatus.OK);
    }

    public DemoController(DemoService demoService) {
        this.demoService = demoService;
    }
}
```

### 测试

请求 [http://localhost:8080/demo](http://localhost:8080/demo)

返回

```
2023-10-20 14:22:20.407  INFO 748 --- [nio-8080-exec-1] com.WebLogAspect                         : 开始服务:http://localhost:8080/demo
2023-10-20 14:22:20.452  INFO 748 --- [nio-8080-exec-1] com.WebLogAspect                         : 客户端IP :127.0.0.1
2023-10-20 14:22:20.453  INFO 748 --- [nio-8080-exec-1] com.WebLogAspect                         : 参数值 :[]
2023-10-20 14:22:20.458  INFO 748 --- [nio-8080-exec-1] com.WebLogAspect                         : 返回值 : <200 OK OK,aerozb,[]>
```