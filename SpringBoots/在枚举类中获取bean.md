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

首先编写枚举类

```java
@Getter
public enum InterfaceEnum {
    XXX("xxx"),
    YYY("yyy");

    @JsonValue
    private final String interfaceName;

    @Setter
    private InterfaceSth interfaceImpl;

    public static class EnumTypeServiceInjector {
        @Resource private InterfaceSth xxxImpl;  //注入实现类bean
        @Resource private InterfaceSth yyyImpl;  //注入实现类bean

        @PostConstruct
        private void postConstruct() {
            XXX.setInterfaceImpl(xxxImpl);
            YYY.setInterfaceImpl(yyyImpl);
        }
    }


    /**
     * 只有一个参数的构造器，另一个属性通过注入并赋值
     */
    InterfaceEnum(String interfaceName){
        this.interfaceName = interfaceName;
    }
}
```


例如入参`params`为：
```json    
{
    "interface_method": "XXX",
    ...      
}
```

使用时：

`params.getInterfaceMethod().getInterfaceImpl().entrance(params.getParams())`
