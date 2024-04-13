首先创建一个TraceIdUtil类
```java title:util/TraceIdUtil.java  
import org.slf4j.MDC;  
import org.apache.commons.lang3.StringUtils;  
import java.util.UUID;  
  
public class TraceIdUtil {  
    public static String TRACE_ID_KEY = "TraceId";  
  
    /**  
     * 生成TraceId  
     * @return  
     * */  
    public static String generateTraceId() {  
        String traceId = UUID.randomUUID().toString().toUpperCase();  
        MDC.put(TRACE_ID_KEY, traceId);  
        return traceId;  
    }  
  
    public static String generateTraceId(String traceId) {  
        if (StringUtils.isBlank(traceId)){  
            return generateTraceId();  
        }  
        MDC.put(TRACE_ID_KEY, traceId);  
        return traceId;  
    }  
  
    /**  
     * 获取TraceId  
     */    public static String getTraceId() {  
        return MDC.get(TRACE_ID_KEY);  
    }  
  
    /**  
     * 移除TraceId  
     */    public static void removeTraceId() {  
        MDC.remove(TRACE_ID_KEY);  
    }  
}
```

修改logback.xml以显示TraceId
```xml title:logback.xml
...
<!-- 彩色日志格式 -->
<property name="CONSOLE_LOG_PATTERN" value="%green(%d{yyyy-MM-dd HH:mm:ss.SSS}) %highlight(%-5level) %boldMegenta([%thread]) [%X{TraceId}] %cyan(%logger{15})"> </property>
<!-- 主要是这个[%X{TraceId}]，里面要对应TraceIdUtil中的TRACE_ID_KEY -->
<!-- 控制台日志：输出全部日志到控制台 -->
<appender name="STDOUT" class="ch.qos.logback.core.encoder.ConsoleAppender">
	<encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
		<Pattern>${CONSOLE_LOG_PATTERN}</Pattern>
	</encoder>
</appender>
...
```

创建一个Filter用于过滤所有的请求
```java title:TraceFilter.java
@Slf4j  
public class TraceFilter implements Filter {  
  
    @Override  
    public void init(FilterConfig filterConfig) throws ServletException {  
        Filter.super.init(filterConfig);  
    }  
  
    @Override  
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain) throws IOException, ServletException {  
        try {  
            HttpServletRequest servletRequest = (HttpServletRequest) request;  
            String gateWayTraceId = ((HttpServletRequest) request).getHeader(TraceIdUtil.TRACE_ID_KEY);  
            TraceIdUtil.generateTraceId(gateWayTraceId);  
            //请求放行  
            filterChain.doFilter(request, response);  
        } catch (Exception e) {  
            e.printStackTrace();  
        } finally {  
            //最后移除，否则可能内存泄露  
            TraceIdUtil.removeTraceId();  
        }  
    }  
  
    @Override  
    public void destroy() {  
        Filter.super.destroy();  
    }  
}
```

并在Configuration中启用
```java title:WebConfiguration.java
@Bean  
@ConditionalOnMissingBean({TraceFilter.class})  
@Order(Ordered.HIGHEST_PRECEDENCE + 101)
public FilterRegistrationBean<TraceFilter> traceFilterBean() {  
    FilterRegistrationBean<TraceFilter> registration = new FilterRegistrationBean<>();  
    registration.setFilter(new TraceFilter());  
    registration.addUrlPatterns("/*");  
    registration.setOrder(1);  
    return registration;  
}
```

并在ResultVO中加入一个字段
`public String traceId = TraceIdUtil.getTraceId();`


# Async实现链路追踪
首先在主方法上加上`@EnableAsync`，并创建Executor，如：
```java
//AsyncConfiguration.java

@Configuration
@EnableAsync  //开启异步执行
public class AsyncConfiguration {
    //自定义线程池
    @Bean("mailThreadPool")  //一定要具名，用于指定Async("mailThreadPool")
    public Executor mailThreadPool(){
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        //核心线程数
        executor.setCorePoolSize(10);
        //最大线程数
        executor.setMaxPoolSize(20);
        //缓冲队列
        executor.setQueueCapacity(500);
        //允许线程的空闲时间60秒
        executor.setKeepAliveSeconds(60);
        //线程池名称前缀
        executor.setThreadNamePrefix("Async----");
        //缓冲队列满后拒绝策略: 由调用线程处理
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy());

		//核心！！：线程装饰：针对线程池中每个线程，在开始和结束进行增强
		executor.setTaskDecorator(new MdcTaskDecorator());

        executor.initialize();
        return executor;
    }
}

```

其中，创建类`MdcTsakDecorator`
```java
public class MdcTaskDecorator implements TaskDecorator {  
  
    @Override  
    public Runnable decorate(Runnable runnable) {  
        Map<String, String> contextMap = MDC.getCopyOfContextMap();//从主线程中取到所有MDC上下文  
        return ()->{  
            try{  
                MDC.setContextMap(contextMap);  //从主线程中获得的Map设置到子线程中  
                runnable.run();  
            }catch (Exception e){  
                e.printStackTrace();  
            }finally {  
                MDC.clear();  
            }  
        };  
    }  
}
```


# RabbitMQ集成TraceId
发送消息之前的前置增强
```java
template.setBeforePublishPostProcessors(message -> {  
    message.getMessageProperties().setHeader(TraceIdUtil.TRACE_ID_KEY, TraceIdUtil.getTraceId());  
    return message;  
});

//也可以单独抽出一个类实现MessagePostProcessor接口并实现postProcessMessage方法
```

消息接收之前的增强，利用AOP增强rabbitListener注解
```java
@Slf4j  
@Aspect  
@Component  
public class MqTraceIdAspect {  
  
    @Around("@annotation(rabbitListener)")  
    public void doBefore(ProceedingJoinPoint joinPoint, RabbitListener rabbitListener) throws Exception {  
        Object[] args = joinPoint.getArgs();  
        for (Object arg : args) {  
            if (arg instanceof Message) {  
                Message message = (Message) arg;  
                Map<String, Object> header = message.getMessageProperties().getHeaders();  
                String traceId = (String) header.get(TraceIdUtil.TRACE_ID_KEY);  
                MDC.put(TraceIdUtil.TRACE_ID_KEY, traceId);  
            }  
        }  
        try {  
            joinPoint.proceed();  
        } catch (Throwable throwable) {  
            throwable.printStackTrace();  
        } finally {  
            TraceIdUtil.removeTraceId();  
        }  
    }  
}
```