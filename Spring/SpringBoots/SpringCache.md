基于注解的缓存策略
# 依赖
```xml
<!-- cache -->  
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-cache</artifactId>  
</dependency>

<!--假如使用redis做缓存-->  
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-data-redis</artifactId>  
</dependency>  

<!--假如使用caffeine做缓存-->  
<dependency>
   <groupId>com.github.ben-manes.caffeine</groupId>
   <artifactId>caffeine</artifactId>
   <version>3.1.8</version>  <!-- 3.x以上的版本需要使用java11以上环境 -->
</dependency>
```
配置类添加`@EnableCaching`


# 注解解析
## @CacheConfig
>@CacheConfig注解用于在类级别上配置缓存的公共属性，避免在每个缓存方法上重复配置相同的缓存属性。通过使用@CacheConfig注解，可以统一指定缓存的名称、键生成器等属性。

**属性：**
**cacheNames ：** 指定缓存的名称
**keyGenerator ：** 指定键生成器

*tips:在Spring框架中，缓存键生成器（KeyGenerator）负责为缓存中的每个数据项生成唯一的键，用于在检索时查找数据项。默认情况下，Spring使用SimpleKeyGenerator作为缓存键生成器，它使用方法的参数作为键*


## 使用Redis缓存
```yaml
spring:  
  data:  
    redis:  
      host: localhost  
      port: 6379  
      database: 0  
      timeout: 10s  # 连接超时时间
      lettuce:  
        pool:  
          enabled: true  
          max-active: 200  # 连接池最大连接数
          max-wait: -1ms  # 连接池最大阻塞等待时间（使用负值表示没有限制）
          max-idle: 10  # 连接池中的最大空闲连接
          min-idle: 0  # 连接池中的最小空闲连接
```

RedisConfig
```java
@Configuration  
public class RedisConfig {  
  
    @Resource  
    private ObjectMapper objectMapper;  
    /**  
     * 项目中手动使用Redis配置  
     */  
  
    /**     * 重新定义RedisTemplate，以string格式保存键、json格式保存值  
     * @param redisConnectionFactory 自动注册的redis🔗配置  
     * @return RedisTemplate<String, Object>  
     */  
    @Bean  
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory){  
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();  
        redisTemplate.setConnectionFactory(redisConnectionFactory);  
        Jackson2JsonRedisSerializer<Object> jsonSerializer = jsonSerializer();  
  
  
        //设置key序列化方式String  
        redisTemplate.setKeySerializer(new StringRedisSerializer());  
        //设置value的序列化方式json  
        redisTemplate.setValueSerializer(jsonSerializer);  
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());  
        redisTemplate.setHashValueSerializer(jsonSerializer);  
  
        redisTemplate.afterPropertiesSet();  
        return redisTemplate;  
    }  
  
    /**  
     * 定义json格式的序列化器  
     * @return Jackson2JsonRedisSerializer<Object>  
     */  
    @Bean  
    public Jackson2JsonRedisSerializer<Object> jsonSerializer() {  
  
        // ObjectMapper mapper = new ObjectMapper();  
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);  
        objectMapper.activateDefaultTyping(  
                LaissezFaireSubTypeValidator.instance,  
                ObjectMapper.DefaultTyping.NON_FINAL,  
                JsonTypeInfo.As.PROPERTY  
        );  
  
        return new Jackson2JsonRedisSerializer<>(objectMapper, Object.class);  
    }  
  
}
```

CacheConfig:
```java
@Configuration  
@EnableCaching  
public class CacheConfig {  
  
    @Bean  
    public RedisCacheManagerBuilderCustomizer myRedisCacheManagerBuilderCustomizer(Jackson2JsonRedisSerializer<Object> jsonRedisSerializer) {  
        return (builder) -> builder  
                .withCacheConfiguration(CacheName.CACHE_5M, RedisCacheConfiguration  
                        .defaultCacheConfig()  
                        .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(jsonRedisSerializer))  
                        // 5min过期  
                        .entryTtl(Duration.ofMinutes(5L))  
                )  
                // user缓存的数据是UserDetails对象，使用默认的jdk序列化可以较好的完成反序列化  
                .withCacheConfiguration(CacheName.CACHE_1H, RedisCacheConfiguration  
                        .defaultCacheConfig()  
                        .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(jsonRedisSerializer))  
                        .entryTtl(Duration.ofHours(1L))  
                );  
    }  
  
}
```

在所需方法上配置`Cacheable`
```java
@Override  
@Cacheable(cacheNames = CacheName.CACHE_5M, key = "#root.method.name + ':' + #id")  
public User getUser(Integer id) {  
    System.out.println(LocalDateTime.now());  
    return new User(id, "xiaoming");  
}
```