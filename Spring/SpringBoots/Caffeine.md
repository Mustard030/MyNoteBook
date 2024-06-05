一个高性能本地缓存库，与ConcurrentMap很像，支持并发，支持O(1)的存取，区别在于：
- ConcurrentMap将存储所有存入的数据，直到显式地移除
- Caffeine通过给定的配置，自动移除不常用的数据，保持内存的合理占用
因此，另一种理解是：Caffeine是一种带有存储和移除策略的Map

依赖：
```xml

<dependency>
   <groupId>com.github.ben-manes.caffeine</groupId>
   <artifactId>caffeine</artifactId>
   <version>3.1.8</version>
</dependency>
```

#### Cache：
在获取缓存值时，如果想要在缓存值不存在时，原子地将值写入缓存，可以调用`get(key, k -> value)`方法，该方法将避免写入竞争。
调用`invalidate()`方法将手动移除缓存。
多线程情况下，使用`get(key, k -> value)`时，如果另一线程同时调用本方法进行竞争，则后一线程会被阻塞，直到前一进程更新完成；而若另一线程调用`getIfPresent`方法，则会立即返回null，不会被阻塞。
使用范例：
```java
Cache<String, String> cache = Caffeine.newBuilder().build();

cache.getIfPresent("123"); //null
cache.get("123", k -> "456"); //456
cache.getIfPresent("123"); //456
cache.set("123", "789");
cache.getIfPresent("123"); //789
```
#### LoadingCache：
LoadingCache是一种自动加载的缓存，和普通的缓存不同的地方在于，当缓存不存在/已过期时，若调用`get()`方法则会自动调用`CacheLoader.load()`方法重新加载最新值。调用`getAll()`方法将遍历所有的key调用`get()`，除非实现了`CacheLoader.loadAll()`方法。
使用LoadingCache时，需要指定CacheLoader，并实现其中的`load()`方法供缓存缺失时自动加载。
多线程情况下，使用`get(key, k -> value)`时，如果另一线程同时调用本方法进行竞争，则后一线程会被阻塞，直到前一进程更新完成。
```java
LoadingCache<String, String> cache = Caffeine.newBuilder()  
        .build(new CacheLoader<String, String>() {  
            @Override  
            public String load(String key) throws Exception {  
                return "value1";  
            }  
              
            @Override  
            public Map<String, String> loadAll(Iterable<? extends String> keys) throws Exception {  
                return null;  
            }  
        });  
  
cache.getIfPresent("abc");  
cache.get("abc");
```

#### AsyncCache：
AsyncCache是Cache的一个变体，其响应结果均为CompletableFuture，通过这种方式，AsyncCache对异步编程模式进行了适配。默认情况下，缓存计算使用`ForkJoinPool.commonPool()`作为线程池，如果想要指定线程池，可以覆盖并实现`Caffeine.executor(Executor)`方法。
多线程情况下，当两个线程同时调用`get(key, k -> value)`，会**返回同一个CompletableFuture对象**。由于返回结果本身不进行阻塞，可以根据业务设计自行选择阻塞等待或者非阻塞。
```java
AsyncCache<String, String> cache = Caffeine.newBuilder().buildAsync();  
CompletableFuture<String> future = cache.get(key, k -> "abc");  
future.get();//阻塞，直到缓存更新完成
```

#### AsyncLoadingCache
以上两者的结合

### 驱逐策略/刷新机制
```java
Caffeine.newBuilder()
		.maximumSize(1000) //最大容量为1000
		.expireAfterWrite(10, TimeUnit.HOURS)//写入10小时后过期
		.expireAfterAccess(1, TimeUnit.HOURS)//访问1h后过期
		.refreshAfterWrite(10, TimeUnit.MINUTES)//写入10分钟后的第一次访问会刷新
```

## 在Springboot中使用Caffeine
依赖：
```xml
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-cache</artifactId>  
</dependency>
```
使用：
1. 注册`cacheManager`的bean
```java
@Bean  
public CacheManager cacheManager(){  
    SimpleCacheManager cacheManager = new SimpleCacheManager();  
    ArrayList<CaffeineCache> caches = new ArrayList<>();  
    caches.add(new CaffeineCache("test", Caffeine.newBuilder().build()));  
    cacheManager.setCaches(caches);  
    return cacheManager;  
}
```
2. 在使用的方法上加上`@Cacheable`注解

和`@Cacheable`相关的常用注解包括：
- `@Cacheable`：表示该方法支持缓存。当调用被注解的方法时，如果对应的键已存在缓存，则不再执行方法体，直接从缓存中返回。当方法返回为null时，不进行缓存操作
- `@CachePut`：表示执行该方法后，其值将作为最新结果更新到缓存中，**每次都会执行该方法**。
- `@CacheEvict`：表示执行该方法后，将触发缓存清除操作。
- `@Caching`：用于组合前三个注解，例如：
```java
@Caching(cacheable = {@Cacheable(value = "test", key = "#root.methodName")},  
        evict = {@CacheEvict(value = "test", key = "#root.methodName")})  
public Student find(Integer id){  
    return null;  
}
```

`@Cacheable`常用的注解属性如下：
- cacheNames/value：缓存组件的名字，即cacheManager中缓存的名称
- key：缓存数据时使用的key。默认使用方法参数值，也可以使用SpEL表达式编写
- keyGenerator：和key二选一使用
- cacheManager：指定使用的缓存管理器
- condition：在方法执行开始前检查，符合condition才进行缓存
- unless：方法执行完成后检查，符合unless才进行缓存
- sync：是否使用同步模式，若使用同步，在多个线程同时对一个key进行load时，其他线程将被阻塞


# 实践：基于注解+Caffeine的通用IP黑名单
依赖：
```xml
<!--SpringBoot-->  
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-aop</artifactId>  
</dependency>  
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-cache</artifactId>  
</dependency>  
 
  
<!--maven-->  
<dependency>  
    <groupId>org.aspectj</groupId>  
    <artifactId>aspectjweaver</artifactId>  
    <version>1.9.7</version>  
</dependency>
<dependency>  
    <groupId>com.github.ben-manes.caffeine</groupId>  
    <artifactId>caffeine</artifactId>  
    <version>3.1.8</version>  
</dependency> 
```

application.yml：
```yaml
blacklist:
  limit: 3
  per: seconds
  counter:
    duration: 3
    unit: minutes
    initial: 100
    max: 10000
  block:
    duration: 60
    unit: minutes
    initial: 100
    max: 10000
```

BlackListConfig：
```java
@Data  
@Configuration  
@ConfigurationProperties(prefix = "blacklist")  
public class BlacklistConfig {  
    private Integer limit;  //每单位多少次的请求上限  
    private TimeUnit per;   //单位（TimeUnit类型）  
    private CounterConfig counter;  //统计名单  
    private CounterConfig block;    //封禁名单  
    @Data  
    public static class CounterConfig {  
        private int duration;   //保存在缓存中多久  
        private TimeUnit unit;  //时间单位  
        private int initial;    //缓存池初始容量  
        private int max;        //缓存池最大容量  
    }  
}
```

CacheConfig：
```java
@Configuration  
public class CacheConfig {  

	@Resource  //使用自动注入获取配置文件中的值动态设置下面的
	private BlacklistConfig blacklistConfig;
      
    @Bean
    public Cache<String, CountObject> counterCache() {
        return Caffeine.newBuilder()
                //设置最后一次写入或访问后经过固定时间过期
                .expireAfterWrite(blacklistConfig.getCounter().getDuration(), blacklistConfig.getCounter().getUnit())
                //初始的缓存空间大小
                .initialCapacity(blacklistConfig.getCounter().getInitial())
                //缓存的最大条数
                .maximumSize(blacklistConfig.getCounter().getMax())
                .build();
    }
    
    @Bean
    public Cache<String, CountObject> blackListCache() {
        return Caffeine.newBuilder()
                //设置最后一次写入或访问后经过固定时间过期
                //黑名单有效期60分钟
                .expireAfterWrite(blacklistConfig.getBlock().getDuration(), blacklistConfig.getBlock().getUnit())
                //初始的缓存空间大小
                .initialCapacity(blacklistConfig.getBlock().getInitial())
                //缓存的最大条数
                .maximumSize(blacklistConfig.getBlock().getMax())
                .build();
    }
}
```

BlackList注解：
```java
@Target(ElementType.METHOD)  
@Retention(RetentionPolicy.RUNTIME)  
public @interface BlackList { }
```

CountObject：
```java
@Data
@AllArgsConstructor
public class CountObject {
    private String method;  //请求的方法名
    private String ip;      //ip地址
    private Integer current; //本日内第几个单位时间：如本日第n秒、第n分
    private Integer count;  //在该秒请求的次数
    private String key;
    private String value;
    
    public CountObject(String method, String ip, Integer current, Integer count){
        this.method = method;
        this.ip = ip;
        this.current = current;
        this.count = count;
    }
    
    /**
     * 用方法名和ip作为key
     */
    public String setKey(String method, String ip){
        this.method = method;
        this.ip = ip;
        this.key = method + "-" + ip;
        return this.key;
    }
    
    public String getKey(){
        return method + "-" + ip;
    }
    
    /**
     * 用当前秒和访问次数作为value
     */
    public String setValue(Integer current, Integer count){
        this.current = current;
        this.count = count;
        this.value = current + "-" + count;
        return this.value;
    }
    
    public String getValue(){
        return current + "-" + count;
    }
    
    public synchronized Integer incr(){
        this.count = this.count + 1;
        return this.count;
    }
}
```

核心：BlackListHandler：
```java
@Aspect  
@Component  
@Slf4j  
public class BlackListHandler {  
    @Resource
    BlacklistConfig blacklistConfig;
    
    @Resource(name = "blackListCache")  
    Cache<String, CountObject> blackListCache;  
      
    @Resource(name = "counterCache")  
    Cache<String, CountObject> counterCache;  
      
    @Around("@annotation(org.example.springbootdemo.annotation.BlackList)")  
    public synchronized Object methodExporter(ProceedingJoinPoint joinPoint) throws Throwable {  
        String method = joinPoint.getTarget().getClass().getName() + "." + joinPoint.getSignature().getName();  
        String ip = null;  
        
        /////////////////////////////////////////////////////////////////
        Object[] args = joinPoint.getArgs();  
        for (Object arg : args) {  
            if (arg instanceof HttpServletRequest) {  
                HttpServletRequest request = (HttpServletRequest) arg;  
                ip = request.getRemoteAddr();  
            }  
        }  
        // ↑需要在controller参数中传入HttpServletRequest，↓（推荐）直接通过RequestContextHolder获取，二选一使用//
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();  
        if (attributes != null) {  
            HttpServletRequest request = attributes.getRequest();  
            // 在这里使用 request 对象  
            //System.out.println("Request Address: " + request.getRemoteAddr());  
            ip = request.getRemoteAddr();  
        }
        /////////////////////////////////////////////////////////////////
        
        String key = method + "-" + ip;  
        CountObject blocked = blackListCache.getIfPresent(key); //是否已被拉黑  
        if(blocked != null){  
            log.error("", new IllegalAccessException("Block by blackList : " + blocked.toString()));  
            return R.error("Block by blackList");  
        }  
        
        //如果还没被拉黑
        
        LocalDateTime now = LocalDateTime.now();    //当前时间
        TimeUnit per = blacklistConfig.getPer();    //设定每多少单位时间内
        int current;
        if (per == TimeUnit.MINUTES){
            current = now.get(ChronoField.MINUTE_OF_DAY);
        } else if (per == TimeUnit.HOURS){
            current = now.getHour();
        } else {    //默认使用秒做单位
            current = now.get(ChronoField.SECOND_OF_DAY);
        }
        
        
        
        CountObject countObject = counterCache.getIfPresent(key);
        
        try{
            //第一次请求或者不在同一单位时间内重新计数，创建一条记录即可
            if (countObject == null || Math.abs(countObject.getCurrent() - current) >= 1) {
                countObject = new CountObject(method, ip, current, 1);
                log.info("{}", countObject);
            } else {    //同一单位时间内则校验访问数量
                Integer count = countObject.incr();
                log.info("{}", countObject);
                if (count > blacklistConfig.getLimit()) {   //当前时间单位同一个IP调用超过设定阈值个请求，加入黑名单缓存
                    blackListCache.put(countObject.getKey(), countObject);
                    log.warn("{}已被加入黑名单", countObject);
                    throw new IllegalAccessException("Blocked by blackList");
                }
            }
        } finally {
            counterCache.put(countObject.getKey(), countObject);
        }
        
        return joinPoint.proceed();
    }  
}
```

使用：
```java
@RestController
public class ViewController {
	@GetMapping("/blackList")
	@BlackList
	public R blackList test(HttpServletRequest request){
		return new R();
	}
}
```
> 注意：这里的`HttpServletRequest`是`jakarta.servlet.http.HttpServletRequest`，切面类里的也是

改进点：
1. HttpServletRequest request参数可以通过WebApplicationContext传入
2. 拉黑时间和次数可以通过配置文件注入的形式设置