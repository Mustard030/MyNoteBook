# @JsonSerialize 和 @JsonDeserialize

JsonSerialize：指定自定义的序列化器，控制如何将对象转换为JSON字符串，包括自定义字段值的格式、类型转换等。
JsonDeserialize：指定自定义的反序列化器，控制如何将JSON字符串转换为对象，包括处理特殊格式、类型转换等。
使用时在类上或者在字段上`@JsonSerialize(using = CustomSerializer.class)`或者`@JsonDeserialize(using = CustomDeserializer.class)`

# @JsonIgnore
用在属性上，序列化和反序列化时忽略这个属性。

# @JsonIgnoreProperties
用在类上，序列化和反序列化时忽略指定的属性。
```java
@JsonIgnoreProperties({"property1", "property2"})  
public class MyClass {  
    private String property1;  
    private String property2;  
    private String property3;  
    // Getter and Setter methods  
}
```
# @JsonFormat
序列化或反序列化时，对日期、时间等特殊类型的字段进行格式化的方式。它的作用是控制日期、时间等特殊类型字段的序列化和反序列化格式。
```java
public class Event {  
    @JsonFormat(pattern = "yyyy-MM-dd")  
    private Date eventDate;  
  
    // 省略构造函数和getter/setter方法  
}
```
@JsonFormat 注解还支持如timezone、shape 等，用于更精细地控制字段的序列化和反序列化行为。

# @JsonUnwrapped
在序列化过程中，@JsonUnwrapped 注解告诉Jackson库将嵌套对象的属性合并到外层对象中，从而在生成的JSON数据中直接包含嵌套对象的属性。这样可以减少生成的JSON结构中的层级，使其更加扁平化。
在反序列化过程中，@JsonUnwrapped 注解告诉 Jackson 库将指定的属性值从JSON数据中提取出来，并填充到外层对象的对应属性中。这样可以让 JSON 数据中的扁平结构直接映射到外层对象的属性上，简化了处理嵌套结构的代码逻辑。
```java
public class Employee {  
    private String name;  
      
    @JsonUnwrapped  
    private Address address;  
 
}  
  
public class Address {  
    private String city;  
    private String street;  
}
```
此时一个`Employee`对象会被序列化为：
```java
Employee employee = new Employee("John Doe", new Address("New York", "123 Main St"));

{  
  "name": "John Doe",  
  "city": "New York",  
  "street": "123 Main St"  
}
```
而在反序列化时使用上面的json将会被转换为一个包含相应属性的`Employee`对象。
除了基本用法，@JsonUnwrapped 注解还支持一些属性，如 prefix 和 suffix，用于控制展开的属性在合并到外层对象时是否添加前缀或后缀。
```java
public class Contact {  
    @JsonUnwrapped(prefix = "home_")  
    private Address homeAddress;  
  
    @JsonUnwrapped(prefix = "work_")  
    private Address workAddress;  
}
```
使用@JsonUnwrapped 注解后，嵌套对象的属性被直接合并到外层对象中，使得JSON数据与Java对象之间的转换更加简洁和直观。

# @JsonAnyGetter
在 Jackson 中，@JsonAnyGetter 注解用于指示 Jackson 在序列化过程中取得对象动态属性的方法。它的作用是将动态属性以键值对的形式==包含在序列化结果==中。

**@JsonAnyGetter 注解的要求**
使用 @JsonAnyGetter 注解的方法必须满足以下要求：
1. 方法必须是公共的
2. 方法不能有参数
3. 方法的返回类型必须是 Map<String, Object> 或其子类
```java
class User { 
	private String name; 
	private int age; 
	private Map<String, Object> dynamicProps;

	@JsonAnyGetter 
	public Map<String, Object> getDynamicProps() { 
		return dynamicProps; 
	}
}
```

# @JsonAnySetter
@JsonAnySetter用于指示 Jackson 在反序列化过程中将动态属性设置到对象上。它的作用是接收动态属性的键值对，并将其==设置到对象的属性==中。
**@JsonAnyGetter 注解的要求**
1.  方法必须是公共的
2. 方法的参数包括一个 String 类型的键和一个 Object 类型的值
3. 方法不能有返回值
```java
class User { 
	private String name; 
	private int age; 
	private Map<String, Object> dynamicProps;

	@JsonAnySetter 
	public void setDynamicProp(String key, Object value) { 
		dynamicProps.put(key, value); 
	}
}
```

# @JsonInclude
用于控制在序列化过程中如何处理属性值为 null 的情况。它的作用是指定在将对象转换为 JSON 字符串时是否包含属性值为 null 的字段。

@JsonInclude 注解可以应用在类级别或属性级别上。
## 类级别的 @JsonInclude 注解
当应用在类级别上时，@JsonInclude 注解指示了默认的 null 处理策略，该策略会应用到整个类的所有属性上。

通过设置 @JsonInclude 的 value 属性，可以指定序列化过程中的 null 处理策略，常用的取值包括：
**Include.ALWAYS**：始终包含属性值为 null 的字段。
**Include.NON_NULL**：仅包含属性值不为 null 的字段。
**Include.NON_EMPTY**：仅包含属性值不为 null 且不为空（如空字符串、空集合）的字段。
**Include.USE_DEFAULTS**：使用默认的 null 处理策略。

## 属性级别的@JsonInclude注解
当应用在属性级别上时，@JsonInclude 注解可以覆盖类级别的默认 null 处理策略，为该属性指定独立的 null 处理策略。通过设置 @JsonInclude 的 value 属性，可以指定序列化过程中该属性的 null 处理策略，取值与类级别的注解相同。

# @JsonAlias
指定属性的别名，在反序列化时将别名与属性进行映射。
```java
public class MyClass {  
    @JsonAlias({"name", "fullName"})  
    private String property;  
}
```

# @JsonManagedReference 和 @JsonBackReference
用于解决循环引用的问题，即某个对象与其他对象存在相互引用的情况。
```java
public class Parent {  
    private String name;  
    @JsonManagedReference  
    private List<Child> children;  
    // Getter and Setter methods  
}  
  
public class Child {  
    private String name;  
    @JsonBackReference  
    private Parent parent;  
    // Getter and Setter methods  
}
```

@JsonManagedReference 注解用于标注父对象中的子对象集合，而 @JsonBackReference 注解用于标注子对象中的父对象引用。可以防止循环引用导致的无限递归问题。

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

# @JsonTypeInfo
在序列化和反序列化过程中，用于处理多态类型。
```java
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, include = JsonTypeInfo.As.PROPERTY, property = "type")  
@JsonSubTypes({  
    @JsonSubTypes.Type(value = Dog.class, name = "dog"),  
    @JsonSubTypes.Type(value = Cat.class, name = "cat")  
})  
public abstract class Animal {  
    // Common properties and methods  
}  
  
public class Dog extends Animal {  
    // Dog-specific properties and methods  
}  
  
public class Cat extends Animal {  
    // Cat-specific properties and methods  
}
```
@JsonTypeInfo 注解指定了类型信息在序列化和反序列化中的处理方式，@JsonSubTypes 注解标注了派生类与其对应的类型标识。

# @JsonFilter
用于动态过滤在序列化过程中要包含的属性。在运行时动态地指定要序列化的属性，在某些场景下非常有用，比如根据用户权限或者其他条件决定序列化的内容。
使用 @JsonFilter 注解定义过滤器，先要定义一个过滤器，将其应用到需要动态过滤的类上。
```java
@JsonFilter("myFilter")  
public class MyDto {  
    private String name;  
    private int age;  
    private String email;  
}
```

配置 ObjectMapper 使用过滤
```java
ObjectMapper mapper = new ObjectMapper();  
SimpleFilterProvider filterProvider = new SimpleFilterProvider();  
filterProvider.addFilter("myFilter", SimpleBeanPropertyFilter.filterOutAllExcept("name", "age"));  
mapper.setFilterProvider(filterProvider);
```
应用过滤器进行序列化
```java
String json = mapper.writer(filterProvider).writeValueAsString(myDto);
```
序列化过程中，只会包含 "name" 和 "age" 两个属性，因为在过滤器中指定了这两个属性。

使用 @JsonFilter注解可以实现动态过滤要序列化的属性，根据需求灵活地控制序列化结果，对于构建灵活的API或者处理动态权限控制非常有用。

# @JsonAppend
允许用户在序列化时动态地添加属性到 JSON 对象中，这些属性可能源自于 Java 对象的不同字段或方法。


# @JsonIgnoreType
在序列化和反序列化过程中忽略被注解的类型。这意味着被 @JsonIgnoreType 注解的类型将会被完全忽略，不会被包含在最终生成的 JSON 中，也不会被用于反序列化。@JsonIgnoreType 注解会被用于一些辅助性的、不需要被序列化和反序列化的类型，或者是一些与 JSON 交互无关的类型。

```java
@JsonIgnoreType  
class AdditionalInfo {  
    private String info;  
    // 省略构造函数和getter/setter方法  
}
```

# @JsonGetter和@JsonSetter
@JsonGetter 注解
1. 用于指定一个非标准的getter方法作为JSON属性的读取方法。
2. 通过在非标准的getter方法上使用@JsonGetter 注解，可以指定该方法对应的JSON属性的名称。
3. 可将Java对象中的属性映射到不同于属性名的JSON属性

@JsonSetter 注解
1. 用于指定一个非标准的 setter 方法作为 JSON 属性的写入方法。
2. 通过在非标准的 setter 方法上使用 @JsonSetter 注解，可以指定该方法对应的 JSON 属性的名称。
3. 可将JSON中的属性值映射到不同于属性名的 Java 对象属性，更灵活的属性赋值。
```java
public class Main {  
    public static void main(String[] args) throws Exception {  
        ObjectMapper mapper = new ObjectMapper();  
        String json = "{\"first_name\":\"John\",\"last_name\":\"Doe\"}";  
        Person person = mapper.readValue(json, Person.class);  
        System.out.println(person.getFullName()); // 输出：John Doe  
  
        String outputJson = mapper.writeValueAsString(person);  
        System.out.println(outputJson); // 输出：{"first_name":"John","last_name":"Doe"}  
    }  
}  
  
class Person {  
    private String firstName;  
    private String lastName;  
  
    @JsonGetter("full_name")  
    public String getFullName() {  
        return firstName + " " + lastName;  
    }  
  
    @JsonSetter("first_name")  
    public void setFirstName(String firstName) {  
        this.firstName = firstName;  
    }  
  
    @JsonSetter("last_name")  
    public void setLastName(String lastName) {  
        this.lastName = lastName;  
    }  
}
```
getFullName方法使用@JsonGetter("full_name")注解，指定返回的全名属性对应的JSON 属性名称为"full_name"。setFirstName和setLastName方法使用了@JsonSetter注解，指定它们对应的JSON属性名称。

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
```java:Controller.java
@GetMapping("/user")
@JsonView({User.UserSimpleView.class})
public List<User> query(){...}

@GetMapping("/user/{userName}")
@JsonView(User.UserDetailView.class)
public User get(@PathVariable String userName){...}
```


```java:User
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

如果没有在类上使用，仅在字段上使用@JsonView注解，此时如果有字段没有加上`@JsonView`则在所有地方都不显示，这个注解也**可以放在类**上，他代表的含义是：没有加注解的字段默认用类上那个设置，除非特别设定

**注意：使用R类统一封装时，由于R类上没有关于JsonView的注解，并且R类是公用的，这会导致返回空对象**

解决办法：
```java:JacksonConfig.java
@Configuration  
public class JacksonConfig {  
    @Primary  
    @Bean(name = "objectMapper")  
    public ObjectMapper objectMapper(Jackson2ObjectMapperBuilder builder)  {  
        // 使用builder可以保留Springboot中的自动配置  
        ObjectMapper objectMapper = builder  
                .timeZone(TimeZone.getTimeZone(ZoneId.systemDefault()))  
                .build();  
        objectMapper.findAndRegisterModules();  
        SimpleModule module = new SimpleModule();  
        //Long类型的反序列化器，防止丢失精度  
        module.addSerializer(Long.class, ToStringSerializer.instance);  
        module.addSerializer(Double.class, ToStringSerializer.instance);  
        module.addSerializer(BigDecimal.class, ToStringSerializer.instance);  
        module.addSerializer(Double.TYPE, ToStringSerializer.instance);  
        module.addSerializer(LocalTime.class, new LocalTimeSerializer(DateTimeFormatter.ofPattern("HH:mm:ss")));  
        module.addDeserializer(LocalTime.class, new LocalTimeDeserializer(DateTimeFormatter.ofPattern("HH:mm:ss")));  
        module.addSerializer(LocalDate.class, new LocalDateSerializer(DateTimeFormatter.ofPattern("yyyy-MM-dd")));  
        module.addDeserializer(LocalDate.class, new LocalDateDeserializer(DateTimeFormatter.ofPattern("yyyy-MM-dd")));  
        module.addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));

		// 关键在这里，添加R类的自定义序列化器
        module.addSerializer(R.class, new RSerializer(objectMapper));  
        
        // module.addDeserializer(LocalDateTime.class, new MyLocalDateTimeDeserializer());  
        objectMapper.registerModule(module);  
        // 在序列化时，忽略值为 null 的属性  
        // objectMapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);  
        // 在反序列化时，忽略在 json 中存在但 Java 对象不存在的属性。  
        objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);  
        return objectMapper;  
    }
}
```

R类序列化器如下：
```java
// @JsonComponent  这里绝对不可以使用JsonComponent自动注册序列化器，因为会有循环依赖问题
public class RSerializer extends JsonSerializer<R> {  
    private final ObjectMapper objectMapper;  
    public RSerializer(ObjectMapper objectMapper) {  
        this.objectMapper = objectMapper;  
    }  
    @Override  
    public void serialize(R r, JsonGenerator jsonGenerator, SerializerProvider serializerProvider) throws IOException {  
        // 获取当前视图  
        Class<?> activeView = serializerProvider.getActiveView();  
        // 开始序列化  
        jsonGenerator.writeStartObject();  
        // 序列化 code 和 message 字段  
        jsonGenerator.writeNumberField("code", r.getCode());  
        jsonGenerator.writeStringField("message", r.getMessage());  
        // 序列化 data 字段，并根据视图进行过滤  
        jsonGenerator.writeFieldName("data");  
        if (r.getData() != null) {  
            // 使用当前激活的视图来序列化数据  
            this.objectMapper.writerWithView(activeView).writeValue(jsonGenerator, r.getData());  
        } else {  
            jsonGenerator.writeNull();  
        }  
        jsonGenerator.writeEndObject();  
    }  
}
```
这里序列化的内容需要根据自己的R类自行修改