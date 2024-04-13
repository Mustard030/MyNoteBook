```xml title:pom.xml
<dependency> 
	<groupId>org.springframework.cloud</groupId> 
	<artifactId>spring-cloud-starter-openfeign</artifactId> 
</dependency>
```

==启动类添加`@EnableFeignClients`注解==

调用其他服务的接口时只需要创建一个对应服务的接口类即可
```java title:service/client/UserClient.java
@FeignClient("userservice") //声明为userservice服务的HTTP请求客户端 
public interface UserClient {
	//路径保证和其他微服务提供的一致即可 
	@RequestMapping("/user/{uid}") 
	User getUserById(@PathVariable("uid") int uid); //参数和返回值也保持一致
}
```

使用时直接注入即可：
```java title:ServiceImpl.java
@Resource 
UserClient userClient; 

@Override 
public UserBorrowDetail getUserBorrowDetailByUid(int uid) { 
	List<Borrow> borrow = mapper.getBorrowsByUid(uid); 
	User user = userClient.getUserById(uid); //这里不用再写IP，直接写服务名称bookservice 
	List<Book> bookList = borrow 
							.stream() 
							.map(b -> template.getForObject("http://bookservice/book/"+b.getBid(), Book.class)) 
							.collect(Collectors.toList()); 
	return new UserBorrowDetail(user, bookList); 
}
```

Hystrix也可以配合Frign实现降级，创建一个实现类，对原有的接口方法进行替代方案实现：
```java title:service/client/impl/UserFallbackClient.java
@Component //注意，需要将其注册为Bean，Feign才能自动注入 
public class UserFallbackClient implements UserClient{ 
	@Override 
	public User getUserById(int uid) { //这里我们自行对其进行实现，并返回我们的替代方案 
	User user = new User(); 
	user.setName("我是替代方案"); 
	return user; 
	} 
}
```
实现完成后，只需要在原有的接口中指定失败替代实现即可：
```java title:service/client/UserClient.java
//fallback参数指定为我们刚刚编写的实现类 
@FeignClient(value = "userservice", fallback = UserFallbackClient.class) 
public interface UserClient { ... }
```

在配置文件中开启熔断支持
```yml title:application.yml
feign: 
	circuitbreaker: 
		enabled: true
```

### 监控页面部署

除了对服务的降级和熔断处理，我们也可以对其进行实时监控，只需要安装监控页面即可，这里我们创建一个新的项目，导入依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
    <version>2.2.10.RELEASE</version>
</dependency>
```

接着添加配置文件：

```yaml
server:
  port: 8900
hystrix:
  dashboard:
    # 将localhost添加到白名单，默认是不允许的
    proxy-stream-allow-list: "localhost"
```

接着创建主类，注意需要添加`@EnableHystrixDashboard`注解开启管理页面：

```java
@SpringBootApplication
@EnableHystrixDashboard
public class HystrixDashBoardApplication {
    public static void main(String[] args) {
        SpringApplication.run(HystrixDashBoardApplication.class, args);
    }
}
```

启动Hystrix管理页面服务，然后我们需要在要进行监控的服务中添加Actuator依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

> Actuator是SpringBoot程序的监控系统，可以实现健康检查，记录信息等。在使用之前需要引入spring-boot-starter-actuator，并做简单的配置即可。

添加此依赖后，我们可以在IDEA中查看运行情况：

![image-20230306230322560](https://image.itbaima.cn/markdown/2023/03/06/a8XmyswEbW1TrtA.png)

然后在配置文件中配置Actuator添加暴露：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: '*'
```

接着我们打开刚刚启动的管理页面，地址为：http://localhost:8900/hystrix/

![image-20230306230335020](https://image.itbaima.cn/markdown/2023/03/06/5rbCxLtR1e8DZAu.png)

在中间填写要监控的服务：比如借阅服务：http://localhost:8301/actuator/hystrix.stream，注意后面要添加`/actuator/hystrix.stream`，然后点击Monitor Stream即可进入监控页面：

![image-20230306230345666](https://image.itbaima.cn/markdown/2023/03/06/iRT6cva5jfd7JSK.png)

可以看到现在都是Loading状态，这是因为还没有开始统计，我们现在尝试调用几次我们的服务：

![image-20230306230355084](https://image.itbaima.cn/markdown/2023/03/06/evLyDl8gVxboiap.png)

可以看到，在调用之后，监控页面出现了信息：

![image-20230306230402987](https://image.itbaima.cn/markdown/2023/03/06/UtcOjEfdnMGQ7ge.png)

可以看到5次访问都是正常的，所以显示为绿色，接着我们来尝试将图书服务关闭，这样就会导致服务降级甚至熔断，然后再多次访问此服务看看监控会如何变化：

![image-20230306230414273](https://image.itbaima.cn/markdown/2023/03/06/97by1FDirquwtv2.png)

可以看到，错误率直接飙升到100%，并且一段时间内持续出现错误，中心的圆圈也变成了红色，我们继续进行访问：

![image-20230306230501543](https://image.itbaima.cn/markdown/2023/03/06/mrVf1qSsDXioIBc.png)

在出现大量错误的情况下保持持续访问，可以看到此时已经将服务熔断，`Circuit`更改为Open状态，并且图中的圆圈也变得更大，表示压力在持续上升。

OpenFeign