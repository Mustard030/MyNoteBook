```xml title:pom.xml
<!--远程调用-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<!--负载均衡也要一起引入-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

==启动类添加`@EnableFeignClients(basePackages="com.example")`注解==
一般情况下，我们在启动类上面添加了@EnableFeignClients注解就是表明当前应用服务（我们称之为服务A）中有的地方想要引用其它应用服务（我们称之为服务B）中的接口。

如果服务B可以单独启动起来并且注册到注册中心，则我们仅仅在服务A的启动类中添加@EnableFeignClients注解即可；

如果服务B没有单独启动起来，而是以Jar包的形式被引入到服务A中，则服务A在启动的时候是不会主动去加载服务B中标注了@FeignClient注解的interface而去自动生成bean对象。这个时候就需要使用basePackages属性字段去指明应用程序A在启动的时候需要扫描服务B中的标注了@FeignClient注解的接口的包路径。


调用其他服务的接口时只需要创建一个对应服务的接口类即可
```java title:service/client/UserClientService.java
@FeignClient("user-client-service") //声明为userservice服务的HTTP请求客户端，填写生产服务的application.name 
public interface UserClientService {
	//路径保证和其他微服务提供的一致即可 
	@RequestMapping("/user/{uid}") 
	User getUserById(@PathVariable("uid") int uid); //参数和返回值也保持一致，如果是实体类入参也一定要加上@RequestBody，否则报405
}
```

消费者使用时直接注入接口即可：
```java title:ServiceImpl.java
@Resource 
UserClientService userClientService; 

@Override 
public UserBorrowDetail getUserBorrowDetailByUid(int uid) { 
	List<Borrow> borrow = mapper.getBorrowsByUid(uid); 
	User user = userClientService.getUserById(uid); //这里不用再写IP，直接写服务名称userClientService 
	List<Book> bookList = borrow 
							.stream() 
							.map(b -> template.getForObject("http://bookservice/book/"+b.getBid(), Book.class)) 
							.collect(Collectors.toList()); 
	return new UserBorrowDetail(user, bookList); 
}
```



另外，feign也可以对非注册中心内的微服务发起直接的URL请求调用，只需使用url属性`@FeignClient(name="随意指定", url="需要访问的http地址")`即可完成



## 超时机制

对于超时，feign有两种超时时间，一个是连接超时（默认10秒），指TCP连接超时；另一个是读取超时（默认60秒），指发送请求后对方处理时间过长导致超时。这个超时时间可以通过配置文件修改，路径为`spring.cloud.openfeign.client.<feignName 用你注解里的name>.connectTimeout`和同路径下的`readTimeout`设置，单位为毫秒。或者feignName的部分输入default后面层级不变就是默认配置，对所有FeignClient生效



## 重试机制

远程调用超时失败后，还可以进行多次尝试，如果某次成功返回ok，都失败则结束调用返回错误。默认不重试。

配置重试器

```java
@Configuration
public class FeignConfig {
    @Bean
    Retryer retryer() {
        return new Retryer.Default(
            1000, //超时后等待时间
            1000, //每次重试前会将总等待时间x1.5算出下次等待时间，与这个值取最小值作为每次最大等待时间
            3  // 重试次数
        );
    }
}
```



## 拦截器

### 请求拦截器

在请求真正发送之前做拦截，可以做定制化的修改，比如统一添加X-Token

```java
@Component  //注册到容器中，所有的FeignClient共用
public class TokenRequestInterceptor implements RequestInterceptor {
    /**
     * 请求拦截器
     * @param template 本次请求的请求信息
     */
    @Override
    public void apply(RequestTemplate template) {
        template.header("X-Token", UUID.randomUUID().toString());
    }
}
```

如果不想在全局生效，就不要使用`@Component`注册到容器，而是直接在想要使用的FeignClient中设置configuration属性`@FeignClient(name = "service-product", configuration = {TokenRequestInterceptor.class})`



### 响应拦截器

响应预处理



### Fallback降级

导入Sentinel依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

配置文件中开启熔断

```yaml
feign:
  sentinel:
    enabled: true
```



#### Sentinel方案

实现FiegnClient中的方法，并注册为@Component

```java title:service/client/impl/UserFallbackClient.java
@Component //注意，需要将其注册为Bean，Feign才能自动注入 
public class UserFallbackClient implements UserClientService{ 
	@Override 
	public User getUserById(int uid) { //这里我们自行对其进行实现，并返回我们的替代方案 
        User user = new User(); 
        user.setName("我是替代方案"); 
        return user; 
	} 
}
```
实现完成后，只需要在原有的FeignClient中指定fallback属性为刚刚的实现类即可：
```java title:service/client/UserClient.java
//fallback参数指定为我们刚刚编写的实现类 
@FeignClient(value = "user-client-service", fallback = UserFallbackClient.class) 
public interface UserClientService { ... }
```



#### Hystrix方案

Hystrix也可以配合Frign实现降级，创建一个实现类，对原有的接口方法进行替代方案实现：

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