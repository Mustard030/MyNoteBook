首先创建一个TraceIdUtils类
```java title:util/TraceIdUtils.java  
import org.slf4j.MDC;  
import cn.hutool.core.util.StrUtil;
import java.util.UUID;  
  
public class TraceIdUtils {  
    public static final String TRACE_ID_KEY = "TraceId";  
  
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
        if (StrUtil.isBlank(traceId)){  
            return generateTraceId();  
        }  
        MDC.put(TRACE_ID_KEY, traceId);  
        return traceId;  
    }  
  
    /**  
     * 获取TraceId  
     */    
    public static String getTraceId() {  
        return MDC.get(TRACE_ID_KEY);  
    }  
  
    /**  
     * 移除TraceId  
     */    
    public static void removeTraceId() {  
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
这里给一个logback的模板
```xml
<?xml version="1.0" encoding="UTF-8"?>  
<configuration>  
    <!-- 日志存放路径 -->  
    <property name="log.path" value="./logs" />  
    <!-- 日志输出格式 -->  
    <!-- 举例：2025-01-23 15:42:47.609 [http-nio-8080-exec-1] INFO  c.v.a.s.t.i.TransformerServiceImpl - [getTransformerTree,53] - 对象转换 -->  
    <!-- <property name="log.pattern" value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{20} - [%method,%line] - %msg%n" /> -->    <!-- 举例：2025-01-24 09:51:41.610 [http-nio-8080-exec-2] [TraceId:ff94035e917c4c76ada38446751b44c4] INFO  - [getTransformerTree,53] - 对象转换 -->  
    <property name="log.pattern" value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] [TraceId:%X{TraceId}] %-5level - [%method,%line] - %msg%n" />  
  
    <!-- 控制台输出 -->  
    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">  
        <encoder>  
            <pattern>${log.pattern}</pattern>  
        </encoder>  
    </appender>  
  
    <!-- 系统日志输出 -->  
    <appender name="file_info" class="ch.qos.logback.core.rolling.RollingFileAppender">  
        <file>${log.path}/vcmy-info.log</file>  
        <!-- 循环政策：基于时间创建日志文件 -->  
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">  
            <!-- 日志文件名格式 -->  
            <fileNamePattern>${log.path}/vcmy-info.%d{yyyy-MM-dd}.log</fileNamePattern>  
            <!-- 日志最大的历史 60天 -->  
            <maxHistory>60</maxHistory>  
            <cleanHistoryOnStart>true</cleanHistoryOnStart>  
        </rollingPolicy>  
        <encoder>  
            <pattern>${log.pattern}</pattern>  
        </encoder>  
        <filter class="ch.qos.logback.classic.filter.LevelFilter">  
            <!-- 过滤的级别 -->  
            <level>INFO</level>  
            <!-- 匹配时的操作：接收（记录） -->  
            <onMatch>ACCEPT</onMatch>  
            <!-- 不匹配时的操作：拒绝（不记录） -->  
            <onMismatch>DENY</onMismatch>  
        </filter>  
    </appender>  
  
    <appender name="file_error" class="ch.qos.logback.core.rolling.RollingFileAppender">  
        <file>${log.path}/vcmy-error.log</file>  
        <!-- 循环政策：基于时间创建日志文件 -->  
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">  
            <!-- 日志文件名格式 -->  
            <fileNamePattern>${log.path}/vcmy-error.%d{yyyy-MM-dd}.log</fileNamePattern>  
            <!-- 日志最大的历史 60天 -->  
            <maxHistory>60</maxHistory>  
            <cleanHistoryOnStart>true</cleanHistoryOnStart>  
        </rollingPolicy>  
        <encoder>  
            <pattern>${log.pattern}</pattern>  
        </encoder>  
        <filter class="ch.qos.logback.classic.filter.LevelFilter">  
            <!-- 过滤的级别 -->  
            <level>ERROR</level>  
            <!-- 匹配时的操作：接收（记录） -->  
            <onMatch>ACCEPT</onMatch>  
            <!-- 不匹配时的操作：拒绝（不记录） -->  
            <onMismatch>DENY</onMismatch>  
        </filter>  
    </appender>  
  
    <!-- 用户访问日志输出  -->  
    <appender name="vcmy-user" class="ch.qos.logback.core.rolling.RollingFileAppender">  
        <file>${log.path}/vcmy-user.log</file>  
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">  
            <!-- 按天回滚 daily -->            <fileNamePattern>${log.path}/vcmy-user.%d{yyyy-MM-dd}.log</fileNamePattern>  
            <!-- 日志最大的历史 60天 -->  
            <maxHistory>60</maxHistory>  
            <cleanHistoryOnStart>true</cleanHistoryOnStart>  
        </rollingPolicy>  
        <encoder>  
            <pattern>${log.pattern}</pattern>  
        </encoder>  
        <filter class="ch.qos.logback.classic.filter.LevelFilter">  
            <!-- 过滤的级别 -->  
            <level>DEBUG</level>  
            <!-- 匹配时的操作：接收（记录） -->  
            <onMatch>ACCEPT</onMatch>  
            <!-- 不匹配时的操作：拒绝（不记录） -->  
            <onMismatch>DENY</onMismatch>  
        </filter>  
    </appender>  
  
    <!-- 系统模块日志级别控制  -->  
    <logger name="com.vcmy" level="debug">  
        <!--系统用户操作日志-->  
        <appender-ref ref="vcmy-user"/>  
    </logger>  
    <!-- Spring日志级别控制  -->  
    <logger name="org.springframework" level="warn" />  
  
    <root level="debug">  
        <appender-ref ref="console" />  
    </root>  
  
    <!--系统操作日志-->  
    <root level="info">  
        <appender-ref ref="file_info" />  
        <appender-ref ref="file_error" />  
    </root>  
  
</configuration>
```

创建一个Filter用于过滤所有的请求
```java title:TraceFilter.java
import lombok.extern.slf4j.Slf4j;  
  
import javax.servlet.*;  
import javax.servlet.http.HttpServletRequest;  
import java.io.IOException;

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



# Async集成TraceID

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
                TraceIdUtils.removeTraceId();  
            }  
        };  
    }  
}
```



# OpenFeign集成TraceID

添加拦截器

```java
import feign.RequestInterceptor;
import feign.RequestTemplate;

@Component
public class TraceRequestInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        // 从ThreadLocal中获取traceId，放到feign请求头中，传给调用者
        String traceId = TraceIdUtils.getTraceId();
        template.header(TraceIdUtils.TRACE_ID_KEY, TraceIdUtils.generateTraceId(traceId););
    }
}
```

Spring 会自动将实现了 `RequestInterceptor` 接口的类注入到 OpenFeign 中，因此只需添加 `@Component` 注解即可。如果没有启用 Spring 自动装配，也可以手动配置：

```java
@Configuration
public class FeignConfig {

    @Bean
    public RequestInterceptor traceIdFeignInterceptor() {
        return new TraceRequestInterceptor();
    }
}
```





# RestTemplate集成TraceID

编写拦截器

```java
public class TraceRestTemplateInterceptor  implements ClientHttpRequestInterceptor {
    
    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {
        // 在请求头中添加或更新Trace ID
        String traceId = TraceIdUtils.getTraceId();
        request.getHeaders().add(TraceIdUtils.TRACE_ID_KEY, TraceIdUtils.generateTraceId(traceId););
        
        // 执行请求
        return execution.execute(request, body);
    }
}
```

添加拦截器

```java
@Configuration
public class RestTemplateConfig {

    /*
    @Bean
    public RestTemplate restTemplate() {
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.getInterceptors().add(new TraceRestTemplateInterceptor());
        return restTemplate;
    }
    */
    
    // 或者使用RestTemplateBuilder（推荐）
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
                .additionalInterceptors(new TraceRestTemplateInterceptor())
                .build();
    }
}
```



# RabbitMQ集成TraceID

## 生产者端

**方式一：使用自定义 `RabbitTemplate` 的 `BeforePublishPostProcessors`**

`RabbitTemplate` 提供了 `BeforePublishPostProcessors`，可以在消息发送前统一处理消息，比如添加 `TraceId` 到消息头。

```java
@Configuration
public class RabbitMqConfig {

    @Bean
    public RabbitTemplate rabbitTemplate(RabbitTemplate rabbitTemplate) {
        rabbitTemplate.addBeforePublishPostProcessors(traceIdPostProcessor());
        return rabbitTemplate;
    }

    @Bean
    public MessagePostProcessor traceIdPostProcessor() {
        return message -> {
            // 从 MDC 获取 TraceId 并添加到消息头
            String traceId = TraceIdUtils.getTraceId();
            
            message.getMessageProperties().getHeaders().put(TraceIdUtils.TRACE_ID_KEY, TraceIdUtils.generateTraceId(traceId));
            
            return message;
        };
    }
}
```



**方式二：使用 `CustomMessageConverter`**

编写消息转换器

```java
public class TraceIdMessageConverter implements MessageConverter {
    private final MessageConverter delegate = new SimpleMessageConverter();

    @Override
    public Message toMessage(Object object, MessageProperties messageProperties) {
        // 从 MDC 获取 TraceId 并添加到消息头
        String traceId = TraceIdUtils.getTraceId();
        messageProperties.setHeader(TraceIdUtils.TRACE_ID_KEY, TraceIdUtils.generateTraceId(traceId));
        
        return delegate.toMessage(object, messageProperties);
    }

    @Override
    public Object fromMessage(Message message) {
        return delegate.fromMessage(message);
    }
}
```

配置`MessageConverter`

```java
@Configuration
public class RabbitMqConfig {

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
        rabbitTemplate.setMessageConverter(new TraceIdMessageConverter());
        return rabbitTemplate;
    }
}
```

**对比两种方法**

| 方法                          | 优点                                                         | 适用场景                                           |
| ----------------------------- | ------------------------------------------------------------ | -------------------------------------------------- |
| `BeforePublishPostProcessors` | 简单易用，支持多个消息处理器链，适合在现有项目中快速集成     | 希望在发送消息前添加其他统一逻辑（如加密、压缩等） |
| 自定义 `MessageConverter`     | 更灵活，可以直接控制消息的序列化和反序列化，适合需要统一定制消息格式的场景 | 需要对消息内容和头进行深度控制的场景               |



## 消费者端

**方式一：使用自定义 `MessageListener`**

自定义 `MessageListenerAdapter`

```java
public class TraceIdMessageListenerAdapter extends MessageListenerAdapter {
    @Override
    public void onMessage(Message message) {
        String traceId = (String) message.getMessageProperties().getHeaders().get(TraceIdUtils.TRACE_ID_KEY);

        try {
            // 将 TraceId 放入 MDC
            TraceIdUtils.generateTraceId(traceId)

            // 调用原来的消息处理逻辑
            super.onMessage(message);
        } finally {
            // 清理 MDC
            TraceIdUtils.removeTraceId();
        }
    }
}
```

配置 `SimpleMessageListenerContainer`

```java
@Configuration
public class RabbitMqConsumerConfig {

    @Bean
    public SimpleMessageListenerContainer messageListenerContainer(ConnectionFactory connectionFactory) {
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.setQueueNames("your-queue-name"); // 替换为你的队列名

        // 使用自定义的 MessageListenerAdapter
        container.setMessageListener(new TraceIdMessageListenerAdapter());
        return container;
    }
}
```



**方式二：使用自定义 `Advice`**

Spring AMQP 提供了一个 `Advice` 机制，允许消息处理之前或之后执行额外的逻辑。

自定义`MethodInterceptor`

```java
public class TraceIdInterceptor implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Message message = (Message) invocation.getArguments()[1]; // 获取 Message
        String traceId = (String) message.getMessageProperties().getHeaders().get(TraceIdUtils.TRACE_ID_KEY);

        try {
            // 将 TraceId 放入 MDC
            TraceIdUtils.generateTraceId(traceId);

            // 执行实际的消息处理逻辑
            return invocation.proceed();
        } finally {
            // 清理 MDC
            TraceIdUtils.removeTraceId();
        }
    }
}
```

配置 `SimpleMessageListenerContainer` 与 `Advice`

```java
@Configuration
public class RabbitMqConsumerConfig {

    @Bean
    public SimpleMessageListenerContainer messageListenerContainer(ConnectionFactory connectionFactory) {
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.setQueueNames("your-queue-name"); // 替换为你的队列名

        // 设置 Advice
        container.setAdviceChain(traceIdAdvice());
        return container;
    }

    @Bean
    public TraceIdInterceptor traceIdAdvice() {
        return new TraceIdInterceptor();
    }
}
```

**方式三：使用消息转换器处理 `TraceId`**

如果你的消费者已经使用了自定义的 `MessageConverter` 来处理消息体的反序列化，也可以在消息转换时提取并设置 `TraceId`。

自定义 `MessageConverter`

```java
public class TraceIdAwareMessageConverter implements MessageConverter {
    private final MessageConverter delegate = new SimpleMessageConverter();

    @Override
    public Message toMessage(Object object, MessageProperties messageProperties) {
        return delegate.toMessage(object, messageProperties);
    }

    @Override
    public Object fromMessage(Message message) {
        String traceId = (String) message.getMessageProperties().getHeaders().get(TraceIdUtils.TRACE_ID_KEY);

        // 将 TraceId 放入 MDC
        TraceIdUtils.generateTraceId(traceId);

        try {
            return delegate.fromMessage(message);
        } finally {
            // 清理 MDC
            TraceIdUtils.removeTraceId();
        }
    }
}
```

配置自定义 `MessageConverter`

```java
@Configuration
public class RabbitMqConsumerConfig {

    @Bean
    public SimpleMessageListenerContainer messageListenerContainer(ConnectionFactory connectionFactory) {
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.setQueueNames("your-queue-name"); // 替换为你的队列名

        MessageListenerAdapter listenerAdapter = new MessageListenerAdapter();
        listenerAdapter.setMessageConverter(new TraceIdAwareMessageConverter());
        container.setMessageListener(listenerAdapter);

        return container;
    }
}
```



**方式四：直接用AOP对rabbitListener注解增强**

依赖：

```xml
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
                TraceIdUtils.generateTraceId(traceId);  
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







**总结**

| 方法                            | 优点                                                    | 使用场景                                        |
| ------------------------------- | ------------------------------------------------------- | ----------------------------------------------- |
| 自定义 `MessageListenerAdapter` | 简单直接，适合需要对所有消费者统一处理 `TraceId` 的场景 | 仅需要为消费者添加额外逻辑的场景                |
| 使用 `Advice`                   | 灵活可扩展，可以加入更多的切面逻辑                      | 需要对消息消费前后的流程进行更复杂的控制的场景  |
| 自定义 `MessageConverter`       | 集成性强，适合同时对消息体和 `TraceId` 进行统一处理     | 需要定制消息体格式，同时对 `TraceId` 处理的场景 |

------

**推荐方案：**

- 如果只是简单地添加 `TraceId` 处理逻辑，建议使用 **自定义 `MessageListenerAdapter`**。
- 如果需要对消息体进行深度定制，或者有复杂的消息消费逻辑，建议使用 **自定义 `MessageConverter`** 或 **Advice**。



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