处理流程：先处理断言，通过再用对应的路由规则路由到指定网关，中间使用过滤器过滤一些请求

依赖：
```xml
<!--响应式网关-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<!--负载均衡-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>

<!--服务发现-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```



使用gateway的时候会报没有链接数据库，需要在`@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})`


## 路由
一般来说路径匹配是从上到下的，但是可以配置order定义匹配顺序
配置路由规则有两种方式：

一、使用配置文件

```yml title:application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: order_route
          uri: lb://service-order  #需要转发的地址，在nacos注册了就是服务名，lb代表负载均衡
          predicates:
            - Path=/api/order/**  #https://localhost:8088/api/order/add  路由到 https://example.org:8020/api/order/add
          order: 0  # 数字越小，优先级越高
        - id: product_route
          uri: lb://product-order
          predicates:  #断言
            - Path=/api/product/**
          filters: 
            - RewritePath=/api/order/?(?<segment>.*), /$\{segment}  # /api/order/a -> /a
            - AddResponseHeader=X-Response-Abc, 123  # 添加响应头
```



二、使用编码方式



## 断言
断言的作用就是只有符合条件的才会被路由



## 过滤器
gateway在路由的时候会原封不动的将请求路径重新定向到某个服务中，如果需要改写路径，则需要配置路径重写过滤器
默认filter：`spring.cloud.gateway.default-filters=[]`

### 全局过滤器
例如编写全局运行耗时过滤器
```java
@Component
@Slf4j  
public class RtGlobalFilter implements GlobalFilter, Ordered {  
    @Override  
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {  
        ServerHttpRequest request = exchange.getRequest();  
        ServerHttpResponse response = exchange.getResponse();  
          
        String uri = request.getURI().toString();  
        long start = System.currentTimeMillis();  
        log.info("请求【{}】开始：时间为{}", uri, start);  
        // ==================前置逻辑===============  
        return chain.filter(exchange)  
                // ==================后置逻辑===============  
                .doFinally((result) -> {  
                    long end = System.currentTimeMillis();  
                    log.info("请求【{}】结束：时间为{}，耗时为{}ms", uri, end, end - start);  
                });  
    }  
    @Override  
    public int getOrder() {  
        return 0;  
    }  
}
```

### 自定义过滤器


## 全局跨域
```yaml
spring:  
  cloud:  
    gateway:  
      globalcors:  
        cors-configurations:  
          '[/**]:':  
            allowedOrigins: "https://your.domain.com"  
            allowedMethods:  
              - GET  
              - POST  
              - PUT  
              - DELETE  
              - OPTIONS
```
也支持在某一个路由规则下的跨域配置
在`spring.cloud.gateway.routes`的某一个规则下添加`metadata.cors`属性，添加`allowedOrigins`、`allowedMethods`等配置项即可


默认微服务之间的调用不经过gateway，如果要经过网关，只需将FeignClient中对应服务名改成网关的服务名，并且url要随之改变即可，但是实践中不建议过网关，网关的作用是对前端的