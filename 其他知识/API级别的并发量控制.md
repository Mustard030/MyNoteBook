# 舱壁模式
类似于船舱中一个个舱壁，即使有一个仓破损了（一个API占用了超多资源）也不会导致整个船进水（服务挂掉）
舱壁模式有两种实现方法：
## 线程池隔离
实现方式：使用线程池（如：`ThreadPoolExecutor`）为每个API定义独立的线程池，并设定其做大并发线程数。Hystrix、Resilience4j等库也提供线程池隔离实现。
这个实现更重量级一些，但是实现的功能更多。


## 信号量隔离
思路：利用信号量限制每个API的并发请求数。
实现：对每个API配置单独的信号量，并设定一个合理的许可数量。例如，某个API的信号量设置为3，在同一时刻只能有3个请求进入该API，超出部分会被拒绝或等待。

线程池隔离的方式请求的原始线程不是处理的线程，但是信号量的是原始线程、它只判断有没有足够的信号量执行目标API。
在轻量级的API中，用信号量方式即可，但是在复杂API中还是用线程池方式比较好。

## 示例
利用Resilience4j实现基于信号量隔离
```xml
<dependency>  
    <groupId>io.github.resilience4j</groupId> 
    <artifactId>resilience4j-spring-boot3</artifactId>  
    <version>2.2.0</version>  
</dependency>

<dependency>  
	<groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-aop</artifactId>  
</dependency>
```

配置：
```yaml
resilience4j:  
  bulkhead:  
    instances:  
      # 定义API舱壁数量  
      defaultApi:  # 配置舱壁实例名称
        max-concurrent-calls: 5   # 最大并发数  
        max-wait-duration: 10s  # 最大等待时间  
      test2:  
        max-concurrent-calls: 5  
        max-wait-duration: 100ms
```

使用时只需要在`Controller`上使用注解
```java
@GetMapping("/test")  
@Bulkhead(name = "test2")  
public TestEntity test() {}
```
`name`为配置中定义的舱壁实例名称，多个API指定了相同名称的舱壁的话，彼此共享这个舱壁的资源

`@Bulkhead`注解的属性解析：

| 属性名            | 作用                                                                                                          | 可配置项          |
| -------------- | ----------------------------------------------------------------------------------------------------------- | ------------- |
| name           | 指定舱壁实例的名称                                                                                                   | 实例名字符串        |
| fallbackMethod | 熔断降级执行方法名称，例如配置的max-wait-duration等待超时了，就会执行这个方法。但是有个要求：熔断方法返回值类型与目标方法返回值类型一致，并且熔断方法入参为BulkheadFullException | 方法名字符串        |
| type           | 实现类型，默认为Bulkhead.Type.SEMAPHORE，即信号量方式，也可以选择Bulkhead.Type.THREADPOOL线程池方式                                   | Bulkhead.Type |

