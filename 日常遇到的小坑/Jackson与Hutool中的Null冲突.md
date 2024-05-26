在JSONObject中如果有JSONNull类型（Hutool的）的Null值，在经过Springboot自带的Jackson序列化返回时会出现问题，通过配置自定义序列化器解决

```java
@JsonComponent
public class JsonNullSerializer extends JsonSerializer<JSONNull> {
    @Override
    public void serialize(JSONNull jsonNull, JsonGenerator jsonGenerator, SerializerProvider serializerProvider) throws IOException {
        jsonGenerator.writeNull();
    }
}
```

