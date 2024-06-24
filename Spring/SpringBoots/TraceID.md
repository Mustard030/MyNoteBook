首先创建一个TraceIdUtils类
```java title:util/TraceIdUtils.java  
import org.slf4j.MDC;  
import org.apache.commons.lang3.StringUtils;  
import java.util.UUID;  
  
public class TraceIdUtils {  
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
<!-- 控制台日志：输出全部日志到控制台 -->  
<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">  
    <layout class="ch.qos.logback.classic.PatternLayout">  
        <!-- 主要是这个[%X{TraceId}]，里面要对应TraceIdUtils中的TRACE_ID_KEY -->  
        <pattern>%green(%d{yyyy-MM-dd HH:mm:ss.SSS}) %highlight(%-5level) %boldMagenta([%thread]) [%X{TraceId}] %cyan(%logger{15}) %msg%n</pattern>  
    </layout>  
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
            String gateWayTraceId = servletRequest.getHeader(TraceIdUtils.TRACE_ID_KEY);  
            TraceIdUtils.generateTraceId(gateWayTraceId);  
            //请求放行  
            filterChain.doFilter(request, response);  
        } catch (Exception e) {  
            e.printStackTrace();  
        } finally {  
            //最后移除，否则可能内存泄露  
            TraceIdUtils.removeTraceId();  
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
`public String traceId = TraceIdUtils.getTraceId();`


# Async实现链路追踪
首先在主方法上加上`@EnableAsync`，并创建Executor，如：
```java
//AsyncConfiguration.java

@Configuration
@EnableAsync  //开启异步执行
public class AsyncConfiguration {
    //自定义线程池
    @Bean("xxx")  //一定要具名，用于指定Async("xxx")
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


# RabbitMQ集成TraceID
发送消息之前的前置增强
```java
template.setBeforePublishPostProcessors(message -> {  
    message.getMessageProperties().setHeader(TraceIdUtils.TRACE_ID_KEY, TraceIdUtils.getTraceId());  
    return message;  
});

//也可以单独抽出一个类实现MessagePostProcessor接口并实现postProcessMessage方法
```

消息接收之前的增强，利用AOP增强rabbitListener注解
依赖：
```java
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```
切面类：
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
                String traceId = (String) header.get(TraceIdUtils.TRACE_ID_KEY);  
                MDC.put(TraceIdUtils.TRACE_ID_KEY, traceId);  
            }  
        }  
        try {  
            joinPoint.proceed();  
        } catch (Throwable throwable) {  
            throwable.printStackTrace();  
        } finally {  
            TraceIdUtils.removeTraceId();  
        }  
    }  
}
```


# Kafka集成TraceID
编写拦截器：
```java
public class TraceWebInterceptor extends HandlerInterceptorAdapter {
 
    private static final Logger LOGGER = LoggerFactory.getLogger(TraceWebInterceptor.class);
 
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        request.setAttribute("startTime", System.currentTimeMillis());
        //traceOrigin、traceCaller、traceId
        String traceOrigin = request.getHeader(TraceConstants.LOG_TRACE_ORIGIN);
        String traceCaller = request.getHeader(TraceConstants.LOG_TRACE_CALLER);
        String traceId = request.getHeader(TraceConstants.LOG_TRACE_ID);
 
        //如果不存在traceId需要生成
        if (StringUtils.isBlank(traceId)) {
            boolean generate = TraceUtil.loadTraceInfo();
            if(generate) {
                LOGGER.debug("[生成追踪信息]" + TraceUtil.getTraceInfoString());
            }
        }else {
            //设置MDC
            MDC.put(TraceConstants.LOG_TRACE_ORIGIN, traceOrigin);
            MDC.put(TraceConstants.LOG_TRACE_CALLER, traceCaller);
            MDC.put(TraceConstants.LOG_TRACE_ID, traceId);
        }
 
        //IP
        String traceIp = IpUtil.getIp(request);
        MDC.put(TraceConstants.LOG_TRACE_IP, traceIp);
 
        //响应返回
        response.setHeader(TraceConstants.LOG_TRACE_ID, TraceUtil.getTraceId());
 
        return super.preHandle(request, response, handler);
    }
 
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws IOException {
        if (LOGGER.isInfoEnabled()) {
            long upmsStartTime = (long) request.getAttribute("startTime");
            long upmsEndTime = System.currentTimeMillis();
            long upmsIntervalTime = upmsEndTime - upmsStartTime;
            LOGGER.info("{} {}接口耗时{}毫秒", request.getRequestURL(), request.getMethod(), upmsIntervalTime);
        }
        MDC.clear();
    }
```

编写Config类：
```java
@Configuration
@ConditionalOnClass({HandlerInterceptorAdapter.class, MDC.class, HttpServletRequest.class})
public class TraceWebAutoConfiguration implements WebMvcConfigurer {
 
    private static List<String> EXCLUDE_PATHS = new ArrayList<>();
 
    @Value("${" + TraceConstants.CONFIG_TRACE_EXCLUDE_PATHS + ":}")
    private String excludePaths;
 
    @Bean
    public TraceWebInterceptor traceWebInterceptor() {
        return new TraceWebInterceptor();
    }
 
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        EXCLUDE_PATHS.add("/error");
        EXCLUDE_PATHS.add("/actuator/**");
 
        if (StringUtils.isNotBlank(excludePaths)) {
            if (excludePaths.contains(",")) {
                String[] split = excludePaths.split(",");
                EXCLUDE_PATHS.addAll(Arrays.asList(split));
            } else {
                EXCLUDE_PATHS.add(excludePaths);
            }
        }
 
        //该方式不能过全部过滤掉
        registry.addInterceptor(traceWebInterceptor()).order(-100).excludePathPatterns(EXCLUDE_PATHS);
    }
}
```