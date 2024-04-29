**应用场景：**
- 收集上报指定关键方法的入参、执行时间、返回结果等关键信息，用作后期调优
- 关键方法在幂等性前置校验（基于本地消息表）
- 类似于Spring-Retry模块，提供方法多次调用重试机制
- 提供关键方法自定义的快速熔断、服务降级等职责
- 关键方法的共性入参校验
- 关键方法在执行后的扩展行为，例如记录日志、启动其他任务等

依赖：
```xml
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
