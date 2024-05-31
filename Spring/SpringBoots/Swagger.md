```xml
<dependency>  
    <groupId>org.springdoc</groupId>  
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>  
    <version>2.1.0</version>  
</dependency>
```

启动项目后访问[http://localhost:8080/swagger-ui/index.html](http://localhost:8080/swagger-ui/index.html)

也可以创建Swagger配置类
```java
// config/SwaggerConfiguration.java

@Configuration  
public class SwaggerConfiguration {  
    @Bean  
    public OpenAPI springDocOpenAPI(){  
        return new OpenAPI().info(new Info()  
                .title("XXXXX管理系统")  
                .description("后端API文档，欢迎查阅")  
                .version("2.0")  
                .license(new License()  
                        .name("我的个人主页")  
                        .url("https://github.com/Mustard030")  
                )  
        );  
    }  
}

```

在Controller类上添加`@Tag`
在方法上添加`@Operation`注解可以自定义信息
`@ApiResponses` `@ApiResponse` 注解自定义返回状态码描述
`@Hidden` 可以隐藏该接口
```java
@RestController
@Tag(name="异步接口", description = "异步请求发送邮件")  
public class AsyncController {
	@GetMapping("/sendMails")  
	@Operation(summary = "异步发送邮件", description = "非阻塞接口")  
	@ApiResponses({  
        @ApiResponse(responseCode = "200", description = "请求成功"),  
        @ApiResponse(responseCode = "500", description = "请求失败")  
	})
	public String sendMails(@RequestParam String[] emails) {
		//do something
	}
}
```

对于实体类
```java
@Data 
@Schema(description = "用户信息实体类") 
public class User { 
	@Schema(description = "用户编号") int id; 
	@Schema(description = "用户名称") String name; 
	@Schema(description = "用户邮箱") String email; 
	@Schema(description = "用户密码") String password; 
}
```

生产环境下需要关闭文档生成
```yml
springdoc: 
	api-docs: 
		enabled: false
```

