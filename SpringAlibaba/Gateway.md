处理流程：先处理断言，通过再用对应的路由规则路由到指定网关，中间使用过滤器过滤一些请求

依赖：
```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

```yml title:application.yml
spring:
	cloud:
		gateway:
			routes:
				- id: after_route 
				  uri:https://example.org:8020  #需要转发的地址
				  predicates:
					  - Path=/order-serv/**  #https://localhost:8088/order-serv/order/add  路由到 https://example.org:8020/order-serv/order/add
				  filters：
					  - StripPrefix=1
```

使用gateway的时候会报没有链接数据库，需要在`@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})`











