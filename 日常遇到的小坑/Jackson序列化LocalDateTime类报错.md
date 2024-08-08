在Config中创建自定义的ObjectMapper Bean

```java
@Configuration  
public class JacksonConfig {  
    @Bean  
	@Primary  
	@ConditionalOnMissingBean(ObjectMapper.class)  
	public ObjectMapper objectMapper(Jackson2ObjectMapperBuilder builder)  {  
	    // 保留Springboot中的自动配置  
	    ObjectMapper objectMapper = builder  
        .timeZone(TimeZone.getTimeZone(ZoneId.systemDefault()))  
        .build();  
	    objectMapper.findAndRegisterModules();  // 核心，下面的按需添加
	      
	    SimpleModule module = new SimpleModule();  
	    //Long类型的反序列化器，防止丢失精度  
	    module.addSerializer(Long.class, ToStringSerializer.instance);  
	    module.addSerializer(Double.class, ToStringSerializer.instance);  
	    module.addSerializer(Double.TYPE, ToStringSerializer.instance);  
	      
	    module.addSerializer(LocalTime.class, new LocalTimeSerializer(DateTimeFormatter.ofPattern("HH:mm:ss")));  
	    module.addDeserializer(LocalTime.class, new LocalTimeDeserializer(DateTimeFormatter.ofPattern("HH:mm:ss")));  
	    module.addSerializer(LocalDate.class, new LocalDateSerializer(DateTimeFormatter.ofPattern("yyyy-MM-dd")));  
	    module.addDeserializer(LocalDate.class, new LocalDateDeserializer(DateTimeFormatter.ofPattern("yyyy-MM-dd")));  
	    module.addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));  
	    module.addDeserializer(LocalDateTime.class, new MyLocalDateTimeDeserializer());
	      
	    objectMapper.registerModule(module);  
	      
	    // 在序列化时，忽略值为 null 的属性  
	    objectMapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);  
	  
	    // 在反序列化时，忽略在 json 中存在但 Java 对象不存在的属性。  
	    objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);  
	      
	    return objectMapper;  
	}
}
```


定义序列化器
```java
public class MyLocalDateTimeDeserializer extends JsonDeserializer<LocalDateTime> {  
      
    private static final DateTimeFormatter FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");  
      
    @Override  
    public LocalDateTime deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {  
        String text = p.getText();  
        try {  
            if (text.matches("^\\d+$")) {  
                // 时间戳转LocalDateTime  
                return LocalDateTime.ofInstant(Instant.ofEpochMilli(Long.parseLong(text)), ZoneId.systemDefault());  
            } else {  
                // 字符串转LocalDateTime  
                return LocalDateTime.parse(text, FORMATTER);  
            }  
        } catch (Exception e) {  
            throw new JsonParseException("Could not parse timestamp '" + text + "'");  
        }  
    }  
}
```

手动使用`objectMapper.readValue`：
```java
@Resource  
ObjectMapper objectMapper;

String json = "{\n" +  
        "  \"create_time\": \"2020-08-06 08:06:06\"\n" +  
        "}";  
  
Person person = objectMapper.readValue(json, Person.class);
```

自动转换不需要做自动注入

字段上不需要加上`@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "GMT+8")`