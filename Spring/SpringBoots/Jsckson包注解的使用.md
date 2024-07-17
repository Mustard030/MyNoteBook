# @JsonCreator
正常反序列化时，默认会调用实体类的无参构造来实例化一个对象，然后使用setter来初始化属性值，`@JsonCreator`这个注解可以自定义反序列化方法，走有这个注解的方法进行反序列化，原先的过程无效
这个注解可以放在**构造方法**或者**静态的工厂方法**上

放在构造方法上：
```java
//@AllArgsConstructor
@Getter
public class Book {

    private String name;

    private Double price;

    @JsonCreator
    public Book(@JsonProperty("name") String name,@JsonProperty("price") Double price ){
        System.out.println("@JsonCreator生效");
        this.name = name;
        this.price = price+1;
    }
   
}
```
上面的@JsonProperty注解就是指定传参名和对象属性关系的，有点像MyBatis的@Param

放在静态工厂方法上：
```java
@AllArgsConstructor
@Getter
public class Book {

    private String name;

    private Double price;

    @JsonCreator
    public static Book unSerialize(){
        System.out.println("正在反序列化成对象");
        return new Book("111",1.00);
    }
}
```
**补充：**
`@ConstructorProperties`的作用和`@JsonCreator`是一样的，但是他只能加在构造方法上，作为反序列化函数。但是他用法更简洁更方便，不用加很多`@JsonProperty`
例如：
```java
@ConstructorProperties({"name", "age", "hobbies", "friends", "salary"})
public PlayerStar(String name, Integer age, String[] hobbies, List<String> friends, Map<String, BigDecimal> salary){
	this.name = name;
	...
}
```
效果也是一样的

# @JsonValue
这个注解可用在get方法上，属性字段上，**一个类只能用一个**，加上这个注解时，该类的对象序列化时就会只返回这个字段的值作为序列化结果
比如一个枚举类的get方法上加上该注解，那么在序列化这个枚举类的对象时，返回的就是枚举对象的这个属性，而不是这个枚举对象的序列化json串。继续改造上面的Book类：
```java
@AllArgsConstructor
@Getter
public class Book {

    //@JsonValue，加get方法或者这个属性上，都一样
    private String name;

    private Double price;

    @JsonCreator
    public static Book doSerialize(){
        System.out.println("正在反序列化成对象");
        return new Book("111",1.00);
    }

    /**
     * 序列化时，序列化成我return的值，即当前对象的name属性
     */
    @JsonValue
    public String getName(){
        return this.name;
    }
}

```

测试：
```java
public class TestSome1 {

    @Test
    void testJson() throws JsonProcessingException {
        ObjectMapper objectMapper = new ObjectMapper();
        Book book = objectMapper.readValue("{\n" +
                "    \"name\":\"Java\",\n" +
                "    \"price\":666.00\n" +
                "}", Book.class);
        System.out.println(book);
        System.out.println("=======");
        //序列化
	    System.out.println(objectMapper.writeValueAsString(book)); //Java
    }
}

```
**示例：**
例如可以用于枚举类中校验传参以及优化前后端数据交互
```java
@AllArgsConstructor
@Getter
public enum OrderFieldEnum {

    CREATE_TIME("createTime","create_time"),
    USER_NAME("userName","user_name");

    private final String value;
    private final String field;
    private static final Map<String,OrderFieldEnum> map = new HashMap<>(3);

    @JsonCreator
    public static OrderFieldEnum unSerializer(String value){
    	//把以value为key，以枚举对象为value，存进map
        if(map.isEmpty()){
            for (OrderFieldEnum fieldEnum : OrderFieldEnum.values()) {
                map.put(fieldEnum.value,fieldEnum);
            }
        }
        //map中找不到就是超出范围
        if(!map.containsKey(value)){
            throw new RuntimeException("超出范围");
        }
        return map.get(value);
    }

    @JsonValue
    public String getValue(){
        return this.value;
    }
}
```

此时，前端传个dto（json）过来，dto里有个参数的类型是枚举类型，反序列化json成dto对象时，枚举类型的属性也会反序列化，上面@JsonCreator定义的unSerializer方法执行，就会完成参数合法性校验。
比如需要返给前端枚举值展示下拉框，就直接Vo里就直接`List<Enum>` enumList，而不用定义个`List<String>` enumList，再反复getValue。因为返给前端时，Vo对象序列化，里面的一个个枚举对象也序列化，而JsonValue已经指定了序列化枚举对象时就把它的value返回就行。


# @JsonView

利用@JsonView注解返回不同的字段

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
     * 用户详情视图，extends代表包括了父类配置的的字段
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

