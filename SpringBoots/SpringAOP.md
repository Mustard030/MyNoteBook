# 场景一：使用AOP与自定义注解实现方法入参出参上报日志  
**应用场景：**
- 收集上报指定关键方法的入参、执行时间、返回结果等关键信息，用作后期调优
- 关键方法在幂等性前置校验（基于本地消息表）
- 类似于Spring-Retry模块，提供方法多次调用重试机制
- 提供关键方法自定义的快速熔断、服务降级等职责
- 关键方法的共性入参校验
- 关键方法在执行后的扩展行为，例如记录日志、启动其他任务等

依赖：
```xml        
<!-- 直接用这个就行，aop包含了下面的aspectjweaver -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>

<!-- 可以不直接用这个 -->
<dependency>  
    <groupId>org.aspectj</groupId>  
    <artifactId>aspectjweaver</artifactId>  
    <version>1.9.21</version>  
</dependency>
```

定义自定义注解：
```java title:MethodExporter.java
@Target({ElementType.METHOD})  
@Retention(RetentionPolicy.RUNTIME)  
public @interface MethodExporter {  }
```

开发切面类：
```java
@Aspect //关键1：说明当前类是一个切面类  
@Component  //关键2：允许在Spring ioc对当前对象实例化并管理  
@Slf4j  
public class MethodExportAspect {  
    //关键3：说明切面的作用范围，任何增加@MethodExporter的目标方法都将在执行方法前先执行该切面方法  
    //@Around环绕通知，最强大的通知类型，可以控制方法入参，执行，返回结果等各方面细节  
    @Around("@annotation(org.example.springbootstudy.myinterface.MethodExporter)")  //如果某个方法上出现了这个注解，就执行下面的代码  
    public Object methodExporter(ProceedingJoinPoint joinPoint) throws Throwable {  
        long st = new Date().getTime();

        //获取参数
        Object[] args = joinPoint.getArgs();

        // 获取方法签名
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();

        // 获取方法上的注解实例
        Xxx xxx = signature.getMethod().getAnnotation(Xxx.class);
  
        //执行目标方法，获取方法返回值  
        Object proceed = joinPoint.proceed();  
  
        long et = new Date().getTime();  
  
        ObjectMapper mapper = new ObjectMapper();   
        //将入参Json序列化  
        String jsonParam = mapper.writeValueAsString(args);  
        //将返回结果Json序列化  
        String jsonResult = null;  
        if (proceed != null) {  
            jsonResult = mapper.writeValueAsString(proceed);  
        } else {  
            jsonResult = "null";  
        }  
        //模拟上报过程  
        log.debug("正在上报服务器调用过程:\ntarget:{}.{}()\nexecution:{}ms,\nparameter:{}\nresult:{}",  
                joinPoint.getTarget().getClass().getSimpleName(),  
                joinPoint.getSignature().getName(),  
                (et-st),  
                jsonParam,  
                jsonResult  
        );  
        return proceed;  
    }  
}
```






# 场景二：使用AOP实现内部调用而无需AopContext.currentProxy()                          
**应用场景：**
比如有个`@Service`类，里面有个A方法，调用了带有`@Transactional`注解的B方法，那么你在调用A方法，执行到B方法的时候是没有事务关联的。因为A调用B的时候，并不是通过代理类，而事务相关逻辑是放在代理类的。一般情况下，使用`AopContext.currentProxy()`方法来获取到当前类的代理对象，调用代理对象的B方法，也是可以完成事务的，但是有更简洁优雅的方式。


第一步：引入依赖（同上）
第二步：开启AspectJ支持 
```java                        
@Configuration
@EnableAsync(mode=AdviceMode.ASPECTJ)                                                                             
@EnableCaching(mode=AdviceMode.ASPECTJ)                             
@EnableTransactionManagement(mode=AdviceMode.ASPECTJ)
public class AppConfig {}
```
`@EnableAsync`、`@EnableCaching`、`@EnableTransactionManagement`都有一个mode参数，他的类型是`AdviceMode`，用来表示通过Spring AOP还是AspectJ来实现功能。我们的目的是使用AspectJ实现内部调用，因此选择`AdviceMode.ASPECTJ`，这三个Enable不是都要的，具体看项目里用到了哪些功能，开启之后要不要使用AspectJ，都是可以配置的。      

第三步：
将javaagent参数传递进去，比如在开发环境中，在Run/Debug Configurations中的Environment的VM options中添加`-javaagent:tools/aspectjweaver-1.9.4.jar`，并将下载来的`aspectjweaver-1.9.4.jar`放到项目根目录的tools目录中。如果在生产环境中，可以通过`java -jar app.jar -javaagent:path/to/weaver.jar`


这样就能实现目标功能了，只要方法B添加了`@Async`、`@Cacheable`、`@Transaction`注解，不管从什么地方调用B方法，都可以完成预想的功能，不存在内部调用切面失效的问题。

总的来说，这种方法使用成本很低，只需要在IDE加个配置就能开发了，再不行就写个监测AspectJ有没有启用，没有就直接结束程序提醒开发加上javaagent。在生产环境使用的成本也很低，比如用Docker部署，直接在Dockerfile里传递javaagent参数，一次改动即可。

## 小问题
终端输出了很多“[Xlint:cantFindType]”字样的日志，怎么去掉？

创建`demo/src/main/resource/META-INF/aop.xml`并将目标类限定在本项目包下
```xml               
<?xml version="1.0"?>
<aspectj>             
    <weaver options="-showWeaveInfo">
        <!-- 改为实际的包名 -->             
        <include within="com.xxx.demo..*"/>   
    </weaver>

    <!-- 或直接将Xlint去掉（不推荐） -->
    <weaver options="-showWeaveInfo -Xlint:ignore">     
    </weaver>    
</aspectj>
```



# AOP相关知识
AOP中的各注解执行顺序为：
1. @Around
2. @Brfore
3. 原方法
4. @After
5. @AfterThrowing

**注意**：使用切面本应该在主类上加上`@EnableAspectJAutoProxy`注解，但我们可以发现在在springboot环境下，由于存在`spring-boot-autoconfigure`依赖，默认会注入`AopAutoConfiguration`配置类，该类的作用等同于`@EnableAspectJAutoProxy`注解，所以在这种情况下可以不加`@EnableAspectJAutoProxy`注解，且AopAutoConfiguration可以通过spring.aop.auto属性控制。但是，为了保险起见，请一律加上`@EnableAspectJAutoProxy`避免出现一些没有启用自动配置导致AOP切面失效的情况。










