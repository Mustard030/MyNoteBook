[版本依赖说明](https://github.com/alibaba/spring-cloud-alibaba/wiki/%E7%89%88%E6%9C%AC%E8%AF%B4%E6%98%8E)

- 当Dubbo使用3.0.0及以上版本时，需要使用Nacos 2.0.0及以上版本
- Dubbo 3.1.8不支持SpringBoot3（Dubbo 3.2.0已支持），因此开发环境必须基于SpringBoot 2.x + JDK 17/8，对应的SpringCloud与SpringCloudAlibaba也要调整为2021.x

```xml
<properties>  
    <java.version>17</java.version>  
    <spring-cloud.version>2021.0.3</spring-cloud.version>  
    <spring-cloud-alibaba.version>2021.0.4.0</spring-cloud-alibaba.version>  
</properties>


<!-- 依赖管理 -->
<dependencyManagement>  
    <dependencies>  
        <dependency>  
            <groupId>org.springframework.cloud</groupId>  
            <artifactId>spring-cloud-dependencies</artifactId>  
            <version>${spring-cloud.version}</version>  
            <type>pom</type>  
        </dependency>  
    </dependencies>  
</dependencyManagement>
```

**生产者**环境下添加
```xml
<dependency>  
    <groupId>org.apache.dubbo</groupId>  
    <artifactId>dubbo-spring-boot-starter</artifactId>  
    <version>3.1.8</version>  
</dependency>
```

主类下添加`@EnableDubbo` 和 `@DubboComponentScan`

配置文件添加
```yml
dubbo:  
  application:  
    name: provider-service-dubbo  
  registry:  
    address: nacos://127.0.0.1:8848  
    username: nacos  
    password: nacos  
  protocol:  
    name: dubbo  
    port: 20880
```

在服务实现类上添加`@DubboService`表示这是一个对外暴露的接口
由于未来传输的对象用于传送给远程调用服务器，因此涉及到的实体类都要实现`Serializable`不写就会报错

> 如果Dubbo 3.1.8使用JDK17会产生一个专属于JDK17的一个BUG，必须增加一个VM选项：`--add-opens java.base/java.lang=ALL-UNNAMED`，在3.2.x中将解决这个bug



**消费者**环境下同样添加dubbo依赖（同上），并开启服务发现
```xml
<dependency>  
    <groupId>com.alibaba.cloud</groupId>  
    <artifactId>spring-cloud-alibaba-nacos-discovery</artifactId>  
</dependency>
```
同样的添加配置文件，`dubbo.application.name`改为consumer-service-dubbo（随意），但不需要添加`protocol`及其子项
只需在主类添加`@EnableDubbo` ，一般做法是把服务提供方的dto和dubbo的接口的包（不需要实现类）打成一个maven包，并在消费者方使用即可，直接复制粘贴也行，要和服务提供者保持一致，消费者要使用时直接`@DubboReference`打在要注入的接口Service上就行如：
```java
@DubboReference
private ProviderService providerService;
```