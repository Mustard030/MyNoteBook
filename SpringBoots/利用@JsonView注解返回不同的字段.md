> JsonView注解来自jackson库中的com.fasterxml.jackson.annotation.JsonView
> SpingMVC也支持这个注解
> 此注解的主要用途就是当你返回实体类时去除敏感信息。比如：有个user表里面有个pwd字段，查询出user后，不想返回pwd字段，可以使用此注解去除pwd字段。

使用：
```java title:Controller
@GetMapping("/user")
@JsonView({User.UserSimpleView.class})
public List<User> query(){...}

@GetMapping("/user/{userName}")
@JsonView(User.UserDetailView.class)
public User get(@PathVariable String userName){...}
```


```java title:User
@Data
@NoArgsConstructor  
@AllArgsConstructor
public class User {
    /**
     * 用户简单视图
     */
    public interface UserSimpleView{};
    /**
     * 用户详情视图
     */
    public interface UserDetailView extends UserSimpleView{};
    
	@JsonView(UserSimpleView.class)
    private String username;
    
    @JsonView(UserDetailView.class)
    private String password;
    
    @JsonView(UserSimpleView.class)
    private Integer age;
	
    //@JsonView放在get\set方法上也可以
}
```

此时如果有字段没有加上`@JsonView`则在所有地方都不显示，这个注解也可以放在类上，他代表的含义是：没有加注解的字段默认用类上那个设置，除非特别设定

