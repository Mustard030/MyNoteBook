# SpringBoot多线程应用

## 以异步发送邮件为例

**准备一个AsyncConfiguration用于配置发送邮件的线程池**

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
        executor.setThreadNamePrefix("mail-thread-pool");
        //缓冲队列满后拒绝策略: 由调用线程处理
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy());

        executor.initialize();
        return executor;
    }
}

```



**配置Service**

使用`asyncService.sendMail()`方法不等待发送完毕直接输出SUCCESS

使用`asyncService.blockSendMail()`方法等待发送完毕返回信息后再输出

```java
//AsyncService.java

@Service
@Slf4j
public class AsyncService {
    @Async("mailThreadPool")  //指定执行的线程池
    @SneakyThrows
    public void sendMail(String email, String message) {
        Thread.sleep(new Random().nextInt(3000));
        log.info("The email to {} has been successfully sent. The message is {}",email, message);

    }

    @Async("mailThreadPool")  //指定执行的线程池
    @SneakyThrows
    public CompletableFuture<String> blockSendMail(String email, String message) {
        Thread.sleep(new Random().nextInt(3000));
        log.info("The email to {} has been successfully sent. The message is {}",email, message);
        return CompletableFuture.completedFuture("The email to "+email+" has been successfully sent. The message is "+message);
    }
}

```



**配置Controller**

```java
//AsyncController.java

@Slf4j
@RestController
public class AsyncController {
    @Resource
    private AsyncService asyncService;

    @GetMapping("/sendMails")
    @SneakyThrows
    public String sendMails() {
        //CompletableFuture<String>[] futures = new CompletableFuture[5];

        String[] emails = new String[]{"a1@163.com", "a2@163.com", "a3@163.com", "a4@163.com", "a5@163.com",};
        for (String email : emails) {
            asyncService.sendMail(email, "Sample text");
        }

        return "SUCCESS";
    }

    @GetMapping("/blockSendMails")
    @SneakyThrows
    public String blockSendMails() {
        CompletableFuture<String>[] futures = new CompletableFuture[5];

        String[] emails = new String[]{"a1@163.com", "a2@163.com", "a3@163.com", "a4@163.com", "a5@163.com",};
        for (int i = 0; i < emails.length; i++) {
            futures[i] = asyncService.blockSendMail(emails[i], "Sample text");
        }
        
        //阻塞等待所有CompletableFuture响应结果
        CompletableFuture.allOf(futures);

        StringBuffer out = new StringBuffer();
        for (CompletableFuture<String> future : futures) {
            try {
                out.append(future.get() + "\n");
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }

        return out.toString();
    }

}
```



## 异步任务也可以返回结果
**Service：**
```java
@Service
public class AsyncService {
	@Async
	public Future<String> executeAsyncTask() {
		try {
			// 模拟长时间运行的任务
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		return new AsyncResult<>("任务执行完成");
	}
}
```

**调用：**
```java
@Autowired
private AsyncService asyncService;

@GetMapping("/async")
public String startAsyncTask() throws ExecutionException, InterruptedException {
	Future<String> future = asyncService.executeAsyncTask();
	return future.get(); // 这里会阻塞并等待异步方法执行完成
}
```