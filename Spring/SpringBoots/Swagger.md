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

# Springboot3
在Springboot3中更推荐使用knife4j
```xml
<dependency>
    <groupId>com.github.xiaoymin</groupId>
    <artifactId>knife4j-openapi3-jakarta-spring-boot-starter</artifactId>
    <version>${knife4j.version}</version>
</dependency>
```
配置文件
```yaml
# springdoc-openapi项目配置
springdoc:
  swagger-ui:
    path: /swagger-ui.html
    tags-sorter: alpha
    operations-sorter: alpha
  api-docs:
    path: /v3/api-docs
  group-configs:
    - group: 'default'
      paths-to-match: '/**'
      packages-to-scan: com.xiaominfo.knife4j.demo.web
# knife4j的增强配置，不需要增强可以不配
knife4j:
  enable: true
  setting:
    language: zh_cn
```

访问`http://ip:port/doc.html`

可以添加一个服务启动监听器
```java
@Component  
@Slf4j  
public class ServiceStartupListener implements ApplicationListener<ApplicationReadyEvent> {  
    private static final String URL_SUFFIX = "/doc.html";  
  
    /**  
     * 监听服务启动完成  
     *  
     * @param event 监听事件  
     */  
    @Override  
    public void onApplicationEvent(ApplicationReadyEvent event) {  
        Environment environment = event.getApplicationContext().getEnvironment();  
        logApplicationStartup(environment);  
    }  
          
private static void logApplicationStartup(Environment env) {  
        String protocol = "http";  
        if (env.getProperty("server.ssl.key-store") != null) {  
            protocol = "https";  
        }  
        String serverPort = env.getProperty("server.port");  
        String contextPath = env.getProperty("server.servlet.context-path");  
        if (CharSequenceUtil.isBlank(contextPath)) {  
            contextPath = URL_SUFFIX;  
        } else {  
            contextPath = contextPath + URL_SUFFIX;  
        }  
        String hostAddress = "localhost";  
        try {  
            hostAddress = InetAddress.getLocalHost().getHostAddress();  
        } catch (UnknownHostException e) {  
            log.warn("The host name could not be determined, using `localhost` as fallback");  
        }  
        log.info(String.format(  
                "\n----------------------------------------------------------" +  
                        "\n\t应用程序“%s”正在运行中......" +  
                        "\n\t接口文档访问 URL:" +  
                        "\n\t本地: \t%s://localhost:%s%s" +  
                        "\n\t外部: \t%s://%s:%s%s" +  
                        "\n\t配置文件: \t%s" +  
                        "\n----------------------------------------------------------",  
                env.getProperty("spring.application.name"),  
                protocol,  
                serverPort,  
                contextPath,  
                protocol,  
                hostAddress,  
                serverPort,  
                contextPath,  
                String.join(", ", env.getActiveProfiles())));  
    }  
}
```