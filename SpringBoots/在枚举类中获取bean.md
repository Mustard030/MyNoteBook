# 场景

[参考文章](https://www.cnblogs.com/Marydon20170307/p/13970490.html)

有时需要通过入参的值判别需要执行不同的策略，例如同一请求入口，根据不同入参用不同的实体类接收&调用不同接口实现类

例如需要根据入参的`interfaceName`字段进行判别：

```json
{
    "interfaceName": "xxx",
    ...
}
```





# 简单实现(switch)

对外只暴露一个接口：

```java
@PostMapping("/gateway")
public ResponseDto<JSONObject> gateway(@RequestBody @Valid RequestDto params) {
    String response;  //本接口返回数据
    JSONObject result;  //对应实现接口返回数据
    String interfaceMethod = "";  //接口方法

    try {
        //1.用实体类接收请求参数
        log.debug("入参：\n" + params.toString());
        //2.非空校验
        //3.json格式校验
        //即将调用的接口方法
        interfaceMethod = params.getInterfaceMethod();

    } catch (Exception e) {
        log.error(e.toString());
    }

    switch (interfaceMethod) {
        case "getStudentNew":
            result = testMapper.getStudentNew();
            break;
        case "getStudentOld":
            result = testMapper.getStudentOld();
            break;
        default:
            throw new RuntimeException("接口方法不存在 interfaceMethod=" + interfaceMethod);
    }

    ...
}
```











# 利用枚举类实现

文件有点多，目录结构如下

- controller
  - [Controller](#Controller)
- service
  - impl
    - [TypeAImpl](#TypeAImpl)
    - [TypeBImpl](#TypeBImpl)
  - [multiTypeService](#multiTypeService)
- enums
  - [InterfaceEnum](#InterfaceEnum)
- [dto](#Dto)
  - [RequestDto](#RequestDto)
  - [RequestParams](#RequestParams)
  - [TypeAParams](#TypeAParams)
  - [TypeBParams](#TypeBParams)
  - [ResponseDto](#ResponseDto)



###### Controller

```java
@RestController
@RequestMapping
public class Controller {
    @PostMapping("/multiImpl")
    public ResponseDto<JSONObject> getValue(@RequestBody @Valid RequestDto params){
        return params.getInterfaceMethod().getInterfaceImpl().entrance(params);
        //params中的InterfaceMethod字段匹配到
    }
}
```



###### multiTypeService

```java
public interface multiTypeService {
    ResponseDto<JSONObject> entrance(RequestDto params);
}
```



###### TypeAImpl

```java
@Service
@Slf4j
public class TypeAImpl implements multiTypeService {
    @Override
    public ResponseDto<JSONObject> entrance(RequestDto params) {
        log.info("执行TypeAImpl");
        JSONObject res = JSONUtil.createObj();
        ResponseDto<JSONObject> responseDto = new ResponseDto<>();
        responseDto.setData(res);
        return responseDto;
    }
}
```



###### TypeBImpl

类似A，关键是实现了multiTypeService接口



###### InterfaceEnum

枚举类

```java
@Getter
public enum InterfaceEnum {
    TYPE_A("typeA"),
    TYPE_B("typeB");
    
    @JsonValue  //json传参入参用这个来匹配对应枚举值
    private final String interfaceName;
    
    @Setter
    private multiTypeService interfaceImpl;
    
    @Component
    public static class EnumTypeServiceInjector {
        @Resource private multiTypeService typeAImpl;  //注入实现类bean
        @Resource private multiTypeService typeBImpl;  //注入实现类bean
        
        @PostConstruct
        private void postConstruct() {
            TYPE_A.setInterfaceImpl(typeAImpl);
            TYPE_B.setInterfaceImpl(typeBImpl);
        }
    }
    
    
    /**
     * 只有一个参数的构造器，另一个属性通过注入并赋值
     */
    ImplMapEnum(String interfaceName){
        this.interfaceName = interfaceName;
    }
}
```



##### Dto

###### RequestDto

入参，只包含最外层结构

```java
@Data
public class RequestDto {
    //不使用泛型时，可以使用@JsonAlias进行反序列化，映射到指定属性上；但是，用泛型后，只能使用@JsonProperty
    @JsonProperty("InterfaceMethod")
    private ImplMapEnum InterfaceMethod;
    
    //用于类、接口、属性，用来开启多态类型处理
    @JsonTypeInfo(
            //使用哪种类型识别码
            use = JsonTypeInfo.Id.NAME, //指定使用名称作为识别码，与指定类上的@JsonSubTypes.Type中的name属性对应
            include = JsonTypeInfo.As.EXTERNAL_PROPERTY,//作为扩展属性包含进去
            //定制识别码的属性名称
        	//这个property指的是，当InterfaceMethod的值为不同的name时，params的类实际上是哪个JsonSubTypes
            property = "InterfaceMethod"  //当use = JsonTypeInfo.Id.NAME时若不指定，则默认值为type
    )
    @JsonProperty("Params")
    @Valid	//用于传递属性验证
    private RequestParams params;
}

```

例如入参`params`为：

```json    
{
    "InterfaceName": "typeA",
    "Params": {
        ...
    }      
}
```



###### RequestParams

实际上是在定义`InterfaceMethod`的值对应的实体接收类

```java
@Data
@JsonSubTypes({ //所有入参的父类，在这设置多态的子类包括哪些，与RequestDto中的@JsonTypeInfo对应
        @JsonSubTypes.Type(value = TypeAParams.class, name = "typeA"),	//name:识别码名称相对应的值，也可以说是入参中InterfaceMethod的值对应的实体接收类
        @JsonSubTypes.Type(value = TypeBParams.class, name = "typeB")
})
public abstract class RequestParams { }
```



###### TypeAParams

```java
@Data
public class TypeAParams extends RequestParams{	//只需继承RequestParams即可
    @JsonProperty("a_p")        //a的专属入参
    private String a_p;
    
    @JsonProperty("normal")     //公共入参
    private String normal;
}

```



###### TypeBParams

同上





###### ResponseDto

根据业务需求来即可，例如

```java
@Data
public class ResponseDto<T> {
    private Integer code;
    private T data;
}
```

