# 利用Redis实现分布式锁

依赖

```xml
<!--    redis Redisson    -->
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.16.2</version>
</dependency>
```





首先创建一个注解

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface RedisLock {
    //锁名称
    @AliasFor("name")
    String value() default "";
    
    @AliasFor("value")
    String name() default "";
    
    //锁等待时间
    long waitTime() default 5;
    
    //锁超时释放时间（默认-1：会触发自动续期）
    long leaseTime() default -1;
}
```



创建RedissonClient的Bean

```java
@Configuration
public class RedissonConfig {
    @Value("${spring.redis.host}")
    private String host;
    
    @Value("${spring.redis.port}")
    private String port;
    
    @Value("${spring.redis.password:}")	//冒号后面相当于空字符串，即若password不存在则用空字符串代替
    private String password;
    
    @Value("${spring.redis.database}")
    private int database;
    
    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://" + host + ":" + port)
                // .setPassword(password)
                .setDatabase(database);// 自行替换和注入
        return Redisson.create(config);
    }
}
```



切面类

```java
@Aspect
@Component
public class RedisLockAspect {
    // 处理EL表达式，使RedisLock的name支持使用EL表达式生成名称
    private static final ParameterNameDiscoverer parameterNameDiscoverer = new DefaultParameterNameDiscoverer();
    
    private static final ExpressionParser parser = new SpelExpressionParser();
    
    @Autowired
    private RedissonClient redissonClient;
    
    @Around("@annotation(redisLock)")
    public Object around(ProceedingJoinPoint pjp, RedisLock redisLock) throws Throwable {
        // 获取锁名
        String lockName = this.getLockName(pjp, redisLock);
        RLock lock = redissonClient.getLock(lockName);
        boolean isLocked = false;
        try {
            isLocked = lock.tryLock(redisLock.waitTime(), // 等待时间
                    redisLock.leaseTime(),  // 超时等待时间，-1会自动续期
                    TimeUnit.SECONDS);  // 如果返回false，意味着锁在被其他线程使用
            if (!isLocked) throw new RuntimeException("获取分布式锁失败");
            
            // 执行方法
            return pjp.proceed();
        } finally {
            // 只有当前线程持有锁时，才释放锁
            if (isLocked && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
    
    /**
     * 解析EL表达式，获取锁名
     */
    private String getLockName(ProceedingJoinPoint pjp, RedisLock redisLock) {
        MethodSignature signature = (MethodSignature) pjp.getSignature();
        Method method = resolveMethod(signature, pjp.getTarget());
        EvaluationContext context = new MethodBasedEvaluationContext(TypedValue.NULL, method, pjp.getArgs(), parameterNameDiscoverer);
        Expression expression = parser.parseExpression(redisLock.name());
        return expression.getValue(context, String.class);
    }
    
    /**
     * 解析方法签名，获取方法对象
     */
    private Method resolveMethod(MethodSignature signature, Object target) {
        Class<?> targetClass = target.getClass();
        try {
            return targetClass.getMethod(signature.getName(), signature.getMethod().getParameterTypes());
        } catch (NoSuchMethodException e) {
            throw new IllegalStateException("Cannot resolve target method" + signature.getName(), e);
        }
    }
}

```



使用：

```java
@RedisLock(name = "'xxBusinessLock-' + #user.account")
@PostMapping("/testLock")
@SneakyThrows
public TestDTO testLock(@RequestBody TestDTO user) {
    TimeUnit.SECONDS.sleep(30);
    return user;
}
```

