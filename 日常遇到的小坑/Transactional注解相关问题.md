# @Transactional和Synchronized同时使用导致的Synchronized不生效问题
## 场景
当涉及一个单数据库内多表读写时，使用了事务注解，又需要保证同一个用户短时间内多次请求只能串行执行。在压力测试时发现了问题Synchronized没有生效
```java
@Transactional
@Synchronized //或是lombok的Synchronized注解
public void demo() {
    ...一些操作...
    synchronized(lock) {
        ...数据库读写操作...
    }
}

```
## 原因分析
`@Transactional`注解是通过AOP的方式实现的，本质上是个代理方法，因此，在进入获得锁之前，事务就已经开始了，此时如果有多个线程请求该方法，会导致事务多次开启但没有锁保护，因此会出现脏数据。

## 解决办法
1. 将锁扩散到事务范围以外，保证先获取锁，再开始事务
2. 查询数据库时加锁`select ... for update`

例如：
```java
//Controller
@RestController
public class TestController {
    @Resource 
    private TestService testService;
    
    @PostMapping("/test")
    public String testInterface() {
            ...一些操作...
        synchronized(lock) {
            testService.databaseOption(); // 调用数据库操作方法
        }
    }
}

//Service
@Service("testService")
public class TestServiceImpl implements TestService {
    
    @Transactional
    public void databaseOption() {
        ...数据库读写操作...
    }
}
```

# @Transactional和@Async的循环依赖问题
>尽管可以同时打上这两个注解，但是仍然不建议这样做，还是分开到两个类中，使用注入的方式调用可以确保不会出问题

在`transation`方法上面加上了`@Transactional`和`@Async`两个注解，然后在`TransationServiceImpl` 类中自己把自己的实例注入到`transationService`属性中，存在循环依赖，理论上单例的循环依赖是允许的，~~但是实际上会报错~~，原因在于三级缓存创建Bean时可能会导致提前暴露了两个版本的Bean，导致SpringIOC容器不知道该注入哪一个
```java
@Service("transationServiceImpl")
public class TransationServiceImpl implements TransationService {

    @Autowired
    TransationService transationService;
	
    @Transactional
    @Async
    @Override
    public void transation() { }
}
```
因此需要确保这两个注解标记的方法位于不同的类中，并且使用`@Async`注解时，必须确保异步方法不会直接调用同一个类中的另一个异步方法，因为这可能会导致死锁。对于事务方法，尽量保持方法的简单性，避免在事务方法内部直接调用异步方法或者由异步方法调用。如果确实需要在事务方法中调用异步方法，可以通过在异步方法执行的内容中显式地开始一个新的事务。

# 回归原始的编程式事务
为什么要回归原始的编程式事务？

答：这样可以避免其他操作对事务造成影响，尽可能使事务范围更小


```java
@Resource  
private TransactionTemplate transactionTemplate;  
  
public void test() {  
    // do something  
  
    transactionTemplate.execute((transactionStatus) -> {  
        try {  
            updateTable();  // do something  
        } catch (Exception e) {  
            transactionStatus.setRollbackOnly();  
            log.error("[service] update error!", e);  
        }  
        return transactionStatus;  
    });  
  
    return;  
}
```
