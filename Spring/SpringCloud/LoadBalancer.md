在config中新建一个类
```java title:LoadBalancerConfiguration.java
import org.springframework.cloud.client.ServiceInstance;  
import org.springframework.cloud.loadbalancer.core.RandomLoadBalancer;  
import org.springframework.cloud.loadbalancer.core.ReactorLoadBalancer;  
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;  
import org.springframework.cloud.loadbalancer.support.LoadBalancerClientFactory;  
import org.springframework.context.annotation.Bean;  
import org.springframework.core.env.Environment;

//@Configuration  //不用加Config注解，因为是直接指定调用的里面的类方法  
public class LoadBalancerConfiguration {  
    @Bean  
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(Environment environment, LoadBalancerClientFactory loadBalancerClientFactory){  
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);  
        return new RandomLoadBalancer(loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);  
    }  
}
```

在BeanConfiguration中配置
```java title:BeanConfiguration.java
@Configuration  
@LoadBalancerClient(value = "userservice",  //指定为userservice服务，只要是调用此服务都会使用我们指定的策略  
                    configuration = LoadBalancerConfiguration.class)  //指定我们刚定义好的配置类  
public class BeanConfiguration {  
    @Bean  
    @LoadBalanced    public RestTemplate restTemplate() {  
        return new RestTemplate();  
    }  
}
```

使用OpenFeign
依赖
```xml title:"borrow/pom.xml"
<dependency>  
    <groupId>org.springframework.cloud</groupId>  
    <artifactId>spring-cloud-starter-openfeign</artifactId>  
    <version>2.1.3.RELEASE</version>  
</dependency>
```

主类中添加`@EnableFeignClients`注解
在Service或其他地方创建文件夹`client`并新建几个接口
```java title:"client/UserClient.java"
@FeignClient("userservice") //声明为userservice服务的HTTP请求客户端  
public interface UserClient {  
    @RequestMapping("/user/{uid}")  
    User findUserById(@PathVariable("uid") int uid);  //把UserController的定义抄过来
}
```

这时调用的方法就变成
```java title:ServiceImpl.java
@Service  
public class BorrowServiceImpl implements BorrowService {  
    @Resource  
    BorrowMapper mapper;  
  
//    @Resource  
//    RestTemplate template;  
  
    @Resource  
    UserClient userClient;  
  
    @Resource  
    BookClient bookClient;  
  
    @Override  
    public UserBorrowDetail getUserBorrowDetailByUid(int uid) {  
        List<Borrow> borrows = mapper.getBorrowsByUid(uid);  
        //通过远程调用获取其他信息  
//        User user = template.getForObject("http://userservice/user/" + uid, User.class);  
//        List<Book> bookList = borrows  
//                .stream()  
//                .map(b -> template.getForObject("http://bookservice/book/" + b.getId(), Book.class))  
//                .collect(Collectors.toList());  
        User user = userClient.findUserById(uid);  
        List<Book> bookList = borrows  
                .stream()  
                .map(b -> bookClient.findBookById(b.getBid()))  
                .collect(Collectors.toList());  
        return new UserBorrowDetail(user, bookList);  
    }  
}
```