在Config中创建自定义的ObjectMapper Bean

```java
@Configuration
public class ObjectMapperConfig {

    /**
     * 初始化单例
     *
     * @return {@link ObjectMapper }
     */
    @Bean("objectMapper")
    @Primary
    public ObjectMapper getInstance(Jackson2ObjectMapperBuilder builder) {
        ObjectMapper instance = builder
                .timeZone(TimeZone.getTimeZone(ZoneId.systemDefault()))
                .build();
        instance.findAndRegisterModules();
        // 把“忽略重复的模块注册”禁用，否则下面的注册不生效
        JsonMapper.builder().disable(MapperFeature.IGNORE_DUPLICATE_MODULE_REGISTRATIONS);
        // 禁用反序列化时对未知属性的失败处理，当JSON中包含Java对象中不存在的字段时，不会抛出异常，而是忽略这些未知字段
        instance.disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
        // 反序列化空数组时将其视为 null
        instance.enable(DeserializationFeature.ACCEPT_EMPTY_ARRAY_AS_NULL_OBJECT);
        // 扩展序列化反序列化方法
        instance.registerModule(configTimeModule());
        instance.registerModule(configNumberModule());
        // 重新设置为生效，避免被其他地方覆盖
        JsonMapper.builder().enable(MapperFeature.IGNORE_DUPLICATE_MODULE_REGISTRATIONS);
        return instance;
    }

    /**
     * 时间对象序列化、反序列化配置
     *
     * @return {@link JavaTimeModule }
     */
    private static JavaTimeModule configTimeModule() {
        JavaTimeModule javaTimeModule = new JavaTimeModule();

        String localDateTimeFormat = "yyyy-MM-dd HH:mm:ss";
        String localDateFormat = "yyyy-MM-dd";
        String localTimeFormat = "HH:mm:ss";
        
        javaTimeModule.addSerializer(LocalTime.class, new LocalTimeSerializer(DateTimeFormatter.ofPattern(localTimeFormat)));
        javaTimeModule.addDeserializer(LocalTime.class, new LocalTimeDeserializer(DateTimeFormatter.ofPattern(localTimeFormat)));
        
        javaTimeModule.addSerializer(LocalDate.class, new LocalDateSerializer(DateTimeFormatter.ofPattern(localDateFormat)));
        javaTimeModule.addDeserializer(LocalDate.class, new LocalDateDeserializer());
        
        javaTimeModule.addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(DateTimeFormatter.ofPattern(localDateTimeFormat)));
        javaTimeModule.addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer());

        return javaTimeModule;
    }
    
    private static SimpleModule configNumberModule() {
        SimpleModule simpleModule = new SimpleModule();
        simpleModule.addSerializer(Long.class, ToStringSerializer.instance);
        simpleModule.addSerializer(Double.class, ToStringSerializer.instance);
        simpleModule.addSerializer(Double.TYPE, ToStringSerializer.instance);
        simpleModule.addSerializer(BigDecimal.class, new BigDecimalSerializer());
        return simpleModule;
    }
}

```


定义序列化器
```java
@JsonComponent
public class MyLocalDateTimeDeserializer extends JsonDeserializer<LocalDateTime> {
    
    private static final DateTimeFormatter DATE_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd");
    private static final DateTimeFormatter DATE_TIME_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
    
    @Override
    public LocalDateTime deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
        String text = p.getText();
        try {
            if (text == null || "null".equals(text) || text.isEmpty() || "undefined".equals(text)) {
                return null;
            }
            else if (text.matches("^-?\\d+$")) {
                // 时间戳转LocalDateTime
                return LocalDateTime.ofInstant(Instant.ofEpochMilli(Long.parseLong(text)), ZoneId.systemDefault());
            } else {
                // 字符串转LocalDateTime
                try {
                    // 尝试按完整的日期时间格式解析
                    return LocalDateTime.parse(text, DATE_TIME_FORMATTER);
                } catch (DateTimeParseException e) {
                    // 尝试按日期格式解析
                    try {
                        LocalDate date = LocalDate.parse(text, DATE_FORMATTER);
                        // 将日期转换为带有时间部分的 LocalDateTime
                        return date.atStartOfDay();
                    } catch (DateTimeParseException ex) {
                        // 如果仍然解析失败，抛出异常
                        throw new IllegalArgumentException("Invalid date or datetime format: " + text, ex);
                    }
                }
            }
        } catch (Exception e) {
            throw new IOException("Could not parse '" + text + "'");
        }
    }
}
```

LocalDate反序列化器

```java
@JsonComponent
public class MyLocalDateDeserializer extends JsonDeserializer<LocalDate> {
    
    private static final DateTimeFormatter DATE_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd");
    private static final DateTimeFormatter DATE_TIME_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
    
    @Override
    public LocalDate deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
        String text = p.getText();
        try {
            if (text == null || "null".equals(text) || text.isEmpty() || "undefined".equals(text)) {
                return null;
            }
            else if (text.matches("^-?\\d+$")) {
                // 时间戳转LocalDateTime
                return LocalDateTime.ofInstant(Instant.ofEpochMilli(Long.parseLong(text)), ZoneId.systemDefault()).toLocalDate();
            } else {
                // 字符串转LocalDateTime
                try {
                    // 尝试按正确的日期格式解析
                    return LocalDate.parse(text, DATE_FORMATTER);
                } catch (DateTimeParseException e) {
                    // 尝试按日期格式解析
                    try {
                        LocalDateTime date = LocalDateTime.parse(text, DATE_TIME_FORMATTER);
                        // 将日期转换为带有时间部分的 LocalDateTime
                        return date.toLocalDate();
                    } catch (DateTimeParseException ex) {
                        // 如果仍然解析失败，抛出异常
                        throw new IllegalArgumentException("Invalid date or datetime format: " + text, ex);
                    }
                }
            }
        } catch (Exception e) {
            throw new IOException("Could not parse '" + text + "'");
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